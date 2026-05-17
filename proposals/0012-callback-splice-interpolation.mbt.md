# Writer-callback splices in string interpolation

* Proposal: [ME-0012](https://github.com/moonbitlang/moonbit-evolution/blob/0012-callback-splice/proposals/0012-callback-splice-interpolation.mbt.md)
* Author: [Hongbo Zhang](https://github.com/bobzhang)
* Review and discussion: [GitHub issue](https://github.com/moonbitlang/moonbit-evolution/pull/22)

## Introduction

Today, every interpolation hole `\{ expr }` in a MoonBit template is desugared
to `<target>.write(expr)`, which routes the value through `Show::output` into
the surrounding builder / logger. We propose a small, syntactically-gated
extension: when the splice expression is a **lambda** of the form
`pat => body`, treat it as a writer-callback that is invoked with the current
target instead of being routed through `Show`. This gives users a streaming,
allocation-free escape hatch from inside the familiar interpolation syntax,
analogous to OCaml's `%a` format specifier.

## Motivation

`target.write(value)` already streams via `Show::output` — there is no
intermediate `String` allocation for ordinary splices. The actual gap is for
values whose textual form is **custom** (escaped, indented, JSON-quoted,
pretty-printed, etc.) and is exposed as a "write into a logger" callback
rather than via `Show`. Many such APIs already exist in `moonbitlang/core`:

- `StringView::escape_to(self, logger : &Logger, quote? : Bool) -> Unit`
- `Buffer::output`, `StringBuilder::output`, and many internal pretty-printers

Today the only way to splice the output of such an API into a template is to
break out of interpolation entirely. The example from
`moonbitlang/core/builtin/autoloc.mbt` (function `SourceLocRepr::to_json_string`):

```moonbit
fn SourceLocRepr::to_json_string(self : SourceLocRepr) -> String {
  let sb = StringBuilder()
  sb.write_string("{\"filename\":")
  self.filename.escape_to(sb)
  sb.write_string(",\"start_line\":\{self.start_line}")
  sb.write_string(",\"start_column\":\{self.start_column}")
  sb.write_string(",\"end_line\":\{self.end_line}")
  sb.write_string(",\"end_column\":\{self.end_column}}")
  sb.to_string()
}
```

The literal-text fragments switch between `write_string` and interpolated
`write_string(...)`, and `escape_to` lives outside any template. The author
wanted to write something like:

```moonbit
fn SourceLocRepr::to_json_string(self : SourceLocRepr) -> String {
  let sb = StringBuilder()
  sb <+ #|{"filename":\{sb => self.filename.escape_to(sb)},"start_line":\{self.start_line},"start_column":\{self.start_column},"end_line":\{self.end_line},"end_column":\{self.end_column}}
  sb.to_string()
}
```

The same need shows up in JSON encoders, pretty-printers, error formatters,
and any code that wants to splice custom-formatted content into a larger
template without leaving interpolation. The win is primarily **readability**
(one continuous template instead of an interleaved sequence of write calls)
and secondarily allocation avoidance for cases where the user would otherwise
have built an intermediate `String`.

### The target is not always `&Logger`

The `<+` operator and template strings are *target-polymorphic*: they
expand into `write_string` / `write` calls against whatever value is on the
left. That target can be a `StringBuilder`, a `Buffer`, an `&Logger`, or
any user-defined builder. A non-trivial MoonBit codebase typically has
several builders, each with its own surface API:

- A **`StringBuilder`** with builder-only fast paths (`write_substring`,
  `write_view`, code-unit operations) that `&Logger` does not expose.
- An **`HTMLBuilder`** with context-aware methods (`open_tag`, `attr`,
  `text`, `raw`) that enforce escaping rules at the type level.
- A **`JSONBuilder`** with structural methods (`begin_object`, `field`,
  `value`, `array`) that enforce well-formedness.
- A logging sink, a tracing event builder, an ANSI-color formatter, etc.

These are not all reducible to `&Logger`. A writer callback that wants
to consume one of them needs a parameter of *that* concrete type, so
that builder-specific methods are in scope inside the lambda body.

A type-fixed wrapper such as `Fmt((&Logger) -> Unit)` cannot express
splices against any of these builders — it forces the body to talk to an
`&Logger` it does not have. The writer-splice form, by contrast, is
naturally polymorphic in the same way `<+` already is: the lambda
parameter is whatever the surrounding target is, so the same splice
mechanism works inside an `HTMLBuilder` template, a `JSONBuilder`
template, or a plain `StringBuilder` template without any new types or
ceremony. The more diverse a project's builder ecosystem is, the larger
the gap between the writer-splice form and the `Fmt`-newtype workaround.

### Comparison to OCaml `%a`

This is the role played by `%a` in OCaml's `Printf`/`Format` families:

```ocaml
Format.fprintf ppf "filename:%a, line:%d" pp_view view line
```

OCaml's `%a` is **explicit** in the format string and is part of the typed
format-string machinery; takes a `(formatter, 'a) -> unit` printer plus an
`'a` value. MoonBit closures collapse the `(printer, value)` pair, and the
proposal below keeps the splice form **explicit at the source** by requiring
a lambda head.

## Proposed solution

Extend the desugaring of a template splice as follows. Given a template
written into a target value `t : T`:

1. If the splice head is a **function literal** — concretely one of
   - `pat => body`
   - `fn (pat) { body }` (with or without a return-type annotation)
   - a parenthesized or type-ascribed wrapping of either of the above
     (e.g. `(sb => body)`, `(sb => body) : (T) -> Unit`)

   it desugars to `(fn (pat) -> Unit { body })(t)`. The lambda parameter is
   elaborated under the expected type `T` exactly (no implicit coercion for
   the parameter itself); the body is elaborated under the expected type
   `Unit` and may use any effect ordinary statements may use (including
   `raise`). Ordinary argument coercions inside `body` are unchanged — e.g.
   passing `sb : StringBuilder` to a function that takes `&Logger` works as
   it does anywhere else.
2. Otherwise, the splice desugars to `t.write(expr)` (the current behaviour,
   which goes through `Show::output`). A function value spliced as
   `\{ f }` therefore still goes through `Show` and fails to compile because
   functions do not implement `Show`. Users must eta-expand.

Literal fragments continue to desugar to `t.write_string("...")` as today.

The chosen rule reserves *function-literal syntax* inside a splice for the
writer-callback arm. Function *values* spliced via `\{ f }` (an identifier
or any non-literal expression) are unaffected and may continue to participate
in any future `Show` extension.

### Why a syntactic gate rather than a type-directed rule

The syntactic gate — "lambda head means writer-callback" — has three
advantages over dispatching on the type of `expr`:

- **Predictable diagnostics.** A type error inside the lambda body is just a
  type error against `(T) -> Unit`; there is no fallback to `Show` that
  could mask the real error.
- **Strictly additive.** No existing template can be a lambda head, so no
  existing code changes meaning. The change does not require the language to
  reserve function types from `Show`.
- **No interaction with generics or inference.** The expected type for the
  lambda parameter is fully determined by the target type at the splice site.

### Determining the target type `T`

Templates are already desugared into method calls on a target whose type the
compiler knows. The three common forms:

| Form | Target type |
| --- | --- |
| `"..."` used as a `String` (no explicit target) | a fresh `StringBuilder`, then `to_string()` |
| `sb <+ "..."` against a concrete builder | the builder's type |
| `logger <+ "..."` against an abstract `&Logger` | `&Logger` |

The lambda parameter is elaborated under exactly this target type. No
implicit coercion happens at the **parameter binding** itself — if the
target is `StringBuilder`, the parameter is `StringBuilder`. Inside the body
ordinary argument coercion still applies, so passing `sb` to a function that
takes `&Logger` works as it does anywhere else. A writer callback intended
for both concrete builders and `&Logger` should be written against `&Logger`
and used from a target of that type, or wrapped with an inline upcast.

When the target type is a generic type variable `t : T` (e.g. inside a
function generic over its writer), the lambda parameter type is exactly that
type variable. The body's type constraints do not refine or re-infer `T`;
the splice is elaborated in the type environment the caller chose.

### Examples

```moonbit
// Inline custom escaping.
let view = "Hello\n\"World\""[:]
let sb = StringBuilder()
sb <+ "body:\{sb => view.escape_to(sb)}"
// desugars to:
//   sb.write_string("body:")
//   (fn (sb) -> Unit { view.escape_to(sb) })(sb)

// Mixing ordinary and callback splices.
sb <+ "name:\{name} body:\{sb => view.escape_to(sb)}"

// Calling an existing writer function — eta-expansion at the splice site.
sb <+ "\{sb => pretty_print(tree, sb)}"
```

#### A non-`&Logger` builder

Consider an `HTMLBuilder` whose surface API encodes context-aware escaping
at the type level. The body of a writer splice on this target has the
builder's methods (`open_tag`, `attr`, `text`, ...) in scope; an `&Logger`
parameter would have none of them:

```moonbit
let html = HTMLBuilder::new()
html <+ "<p>\{sb => sb..text("Hello ")..open_tag("b")..text(user)..close_tag()}!</p>"
// desugars to:
//   html.write_string("<p>")
//   (fn (sb : HTMLBuilder) -> Unit {
//     sb..text("Hello ")..open_tag("b")..text(user)..close_tag()
//   })(html)
//   html.write_string("!</p>")
```

Crucially, `text(user)` performs HTML-correct escaping at the type level —
that behaviour lives on `HTMLBuilder`, not on `&Logger`. A
`Fmt((&Logger) -> Unit)` wrapper could not host this lambda at all; an
inline lambda whose parameter type is `HTMLBuilder` can. The same shape
applies to a hypothetical `JSONBuilder::field`, `XMLBuilder::cdata`,
`AnsiFormatter::style`, etc.

### What is *not* a writer-callback splice

- `\{ f }` where `f` is a value of function type — desugars via `Show`,
  fails to compile because functions do not implement `Show`. Users must
  eta-expand: `\{ sb => f(sb) }`.
- `\{ if cond { sb => f(sb) } else { sb => g(sb) } }` — the splice head is
  an `if` expression, not a function literal, so this is a regular `Show`
  splice; it fails to compile. To dispatch dynamically, lift to a single
  lambda: `\{ sb => if cond { f(sb) } else { g(sb) } }`.

### Evaluation order

A template is evaluated strictly left to right. For a splice
`\{ pat => body }`, the lambda is constructed and invoked exactly once, in
place, with the current target; `body` is evaluated under the same effect
context as the enclosing template (it may `raise`, capture, or mutate as
ordinary statements may). If `body` raises, the partial output already
written to the target remains.

### Naming

We call this construct a *writer splice*. The lambda parameter is the
*target*. The proposal does **not** introduce a new top-level type, trait,
or method.

## Prior art

The idea of letting a splice be "a callback that writes into the surrounding
stream" is well-established in formatting libraries that aim to avoid
materializing intermediate strings:

- **OCaml `Printf`/`Format` `%a` specifier.** `Format.fprintf ppf "%a" pp_v v`
  takes a printer `(formatter, 'a) -> unit` and a value `v`. The proposal
  here collapses the printer/value pair into a single closure that already
  closes over its value, which is natural in a language with first-class
  closures.
- **F# `printf "%a"`.** Same as OCaml.
- **Common Lisp `format` `~/name/` directive.** Invokes a named function
  with the stream and arguments, e.g. `(format t "~/pprint-fill/" obj)`.
  This is closer in spirit to the sigil-based alternatives (`\@{f}`,
  `\{f(_)}`) than to the lambda form below.
- **Racket's Scribble `@func{...}`.** Explicit `@` sigil that dispatches into
  a function with the surrounding context.
- **Smalltalk's `printOn:`, Ruby's `pp`, Go's `fmt.Formatter`.** Trait-shaped
  variants of the same idea — a type exposes a method that takes a stream
  argument. These are the spiritual ancestors of the `Fmt`/`Show` alternative
  discussed below; they tie writer behaviour to a value's type rather than to
  the splice site, which is exactly what this proposal *does not* do.
- **JavaScript tagged template literals** (`` tag`...${x}...` ``) are
  strictly more general — the tag function receives the literal-array and
  every substitution and can do anything. The proposal here is a much
  smaller knob aimed at one specific need.

What is novel here is the *syntactic gate*: dispatching on whether the
splice head is a function literal rather than on a sigil, a trait, or a
named-function lookup. This keeps the proposal aligned with how MoonBit
templates already work (purely structural expansion of literal-vs-splice
fragments into `write_string` / `write` calls) rather than introducing a
parallel mechanism.

## Use cases

Beyond the `escape_to` motivator, writer splices apply anywhere the
rendering *algorithm* — not the value's `Show` instance — is what the call
site wants, and where allocating the intermediate string is wasteful or the
renderer needs to consult the target. Concrete examples:

- **Streaming a sequence with a separator.** Today users either build an
  intermediate `Array[String]` and call `String::concat`, or interleave
  literal-string writes with manual loops. With a writer splice:

  ```moonbit
  sb <+ "[\{sb => @iter.write_separated(sb, items, sep=", ")}]"
  ```

- **Conditional fragments.** Replace `if cond { sb <+ "..." }` interleaved
  with surrounding template text by a single template:

  ```moonbit
  sb <+ "user=\{name}\{sb => if admin { sb <+ " [admin]" }}"
  ```

- **HTML/XML/SQL escaping.** Splice content that knows the right escaping
  rule for its position (text content vs attribute vs URL component), all
  without materializing.

- **Pretty-printers that consult the target.** A structured pretty-printer
  needs the current column / indent depth, which lives on the builder. A
  writer splice gives it the builder directly:

  ```moonbit
  sb <+ "let \{name} =\n\{sb => pp.indent(sb, 2, () => pp.expr(sb, body))}"
  ```

- **Error messages with source spans.** Render a snippet with carets at the
  right column inside a larger diagnostic template without building the
  snippet as a String first.

- **Hex/byte dumps inside diagnostics.**

  ```moonbit
  sb <+ "unexpected: \{sb => hex_dump(sb, bytes, group=8)}"
  ```

- **i18n with pluralization and runtime values.** Plural/gender selection
  often depends on count/locale and writes a localized fragment directly:

  ```moonbit
  sb <+ "\{sb => i18n_plural(sb, key, count, locale)}"
  ```

- **JSON / s-expression / protobuf encoders.** Encoders today come in two
  flavours — one that materialises and one that streams into a logger;
  writer splices make the streaming form the natural call site, removing
  the need for the materialising variant in most user code.

The common thread is that the value at the splice site is best expressed
*as the act of writing it* rather than as a `Show`-able representation of
itself.

## Alternatives considered

### Newtype wrapper, no language change

Ship a `Fmt((&Logger) -> Unit)` newtype with a `Show` impl:

```moonbit
pub struct Fmt((&Logger) -> Unit)
pub impl Show for Fmt with output(self, logger) {
  let Fmt(f) = self
  f(logger)
}
```

Users would then write `"\{Fmt(sb => view.escape_to(sb))}"`. This works
today with zero compiler change, but the shape is structurally wrong for
how templates already work. Template strings and the `<+` / `..write`
forms are **purely syntactic** today: literal fragments expand to
`write_string` and splices expand to `write`, against *whatever target*
the user wrote on the left. There is no trait constraint on the operator
itself — a `StringBuilder`, a `Buffer`, an `&Logger`, or a user-defined
type with a `write_string` / `write` pair all work the same way.

Routing writer callbacks through `Show` collapses that generality:

- `Show::output` is fixed to `(Self, &Logger) -> Unit`, so `Fmt`'s
  callback type is forced to `(&Logger) -> Unit`. A writer that wants to
  consume a concrete `StringBuilder` (e.g. to use builder-only fast
  paths) cannot be expressed as a `Fmt` without an upcast at the body
  level on every call.
- The wrapper is per-target — a separate `Fmt[Buffer]` newtype would be
  needed to bypass `&Logger`, and each would need its own `Show` story,
  which is itself a contortion since `Show` is `&Logger`-oriented.

  Concretely: a `HtmlFmt((HTMLBuilder) -> Unit)` newtype solves nothing
  at the language level. Its `Show` impl can only write text into an
  `&Logger`, so to splice an `HtmlFmt` into an `HTMLBuilder` template the
  builder would have to invoke the closure via some bespoke method like
  `html..apply(htmlFmt)` — different from `Show::output`, different from
  every other builder's bespoke method, and unable to participate in
  `<+` interpolation at all without exactly the language change this
  proposal makes. Each new builder would reinvent the same dispatch
  mechanism under a different name. That is the language-level cost the
  proposal removes by gating on syntax instead of on a trait.
- It conceptually conflates "writer callback" with "value that knows
  how to render itself," which is a layer mismatch: a writer callback
  is not a value of some type with a representation; it *is* the
  rendering.

The writer-splice form preserves the macro-style expansion: the
parameter type of the lambda is whatever the target on the left is.
Library authors may still ship `Fmt` for cases where they need a
value-shaped, `Show`-routed writer (e.g. stored in a struct, passed
through a generic `Show` API). Both forms can coexist; the writer
splice is just the one that matches the existing `<+` / template
semantics.

### Explicit sigil — `\!{f}`, `\@{f}`, `\{f(_)}`

A sigil makes the writer arm explicit at the splice site without requiring a
lambda head:

```moonbit
sb <+ "body:\@{view.escape_to}"   // borrows OCaml %a
sb <+ "body:\!{view.escape_to}"   // "do this side effect"
sb <+ "body:\{view.escape_to(_)}" // hole-style placeholder
```

The sigil forms read shorter at the call site and avoid eta-expansion. They
also give a clearer "this is special" signal to readers and to LLM-based
tooling. The downsides are: they commit the language to a second splice form
that lives only inside templates; they need an answer for what `\@{e}` does
when `e` is not function-shaped (presumably a type error); and once shipped
they are hard to remove. The writer-splice form is the smallest possible
change — if real usage shows the lambda head is too verbose, a sigil can be
added later as pure surface sugar.

### A `WriteInto[T]` trait

Introduce

```moonbit
pub trait WriteInto[T] {
  write_into(self, target : T) -> Unit
}
```

and dispatch the splice via this trait. This is more general (a type
implements it once and is spliced everywhere) but it locks the language into
a second stringification trait next to `Show`, and the interaction with
`Show` must be specified. The lambda-based design here is a strict subset
and can be migrated to a trait-based design later if the value-shaped
formulation proves necessary.

## Impact

- **Source compatibility.** Strictly additive at the value level: no
  existing template can have a function-literal head inside a splice, so no
  existing code changes meaning. The change does **reserve function-literal
  syntax inside splices** from any future `Show`-based extension — once
  this proposal ships, `\{ x => ... }` is always a writer splice. Function
  *values* (`\{ f }`) are unaffected.
- **Diagnostics.** Errors inside a writer splice are reported as ordinary
  type errors against the expected callback type `(T) -> Unit`. There is no
  fallback path.
- **Performance.** A writer splice compiles to a direct call to the lambda
  with the target argument; modern MoonBit closures inline through this
  trivially. No intermediate `String` is allocated for custom-formatted
  content.
- **Tooling.** Goto-definition, rename, and "find references" on the lambda
  body behave exactly as for any other lambda. The target parameter is in
  scope only inside the lambda body.
- **Migration.** None required. Library authors may opt into writer splices
  where it improves clarity or removes intermediate allocations.

## Future work

- A standard helper trait or wrapper (e.g. `Fmt` in `moonbitlang/core`) to
  let library authors expose writer callbacks as first-class values for use
  outside templates.
- If usage shows the lambda head is consistently boilerplate, a sigil
  (`\@{f}` or `\{f(_)}`) can be layered on top as a pure surface-level sugar
  that desugars to the same lambda form.
- Possible extension to two-step formatting (an explicit `Format` namespace
  with combinators) once the closure-based primitive sees real usage.
