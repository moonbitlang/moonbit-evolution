# Typed View Tag Syntax

* Proposal: [ME-0009](https://github.com/moonbitlang/moonbit-evolution/blob/main/proposals/0009-typed-view-tag-syntax.mbt.md)
* Author: [ubugeeei](https://github.com/ubugeeei)
* Status: Proposed
* Review and discussion: [GitHub issue #19](https://github.com/moonbitlang/moonbit-evolution/issues/19)

## Introduction

This proposal refines and extends [issue #19](https://github.com/moonbitlang/moonbit-evolution/issues/19).
It keeps the strongest parts of that design:

1. Minimal tag-oriented surface syntax
2. No HTML-style whitespace semantics
3. Explicit comma-separated children
4. Straightforward desugaring to MoonBit UI builders

At the same time, it addresses four gaps that become important once the syntax is used at scale:

1. Views should be explicit in the language, not just inferred from ordinary functions
2. Props and slots should both be part of the view interface
3. Generic views and higher-order wrappers should remain fully type-safe
4. The parser should preserve a fast path for ordinary `.mbt` files instead of forcing every file into a JSX-aware mode

The core idea is to introduce a dedicated `view` keyword together with:

1. Tag syntax for view invocation
2. First-class `view(...) renders ...` types
3. Typed named slots
4. Exact view-interface preservation for higher-order wrappers

This proposal intentionally does not introduce hook syntax.

## Motivation

### Preserve the good parts of issue #19

Issue #19 is directionally right because it avoids the two biggest sources of JSX complexity:

1. Implicit whitespace-sensitive text nodes
2. A large HTML-oriented semantic model that does not match MoonBit's expression-oriented design

Its explicit child list is especially attractive for MoonBit because it is formatter-friendly, LSP-friendly, and predictable for both humans and tools.

### Why `view` is the right name

If we care about conceptual integrity, the feature should be named after what it semantically denotes.
This construct is not a generic framework object and it is not a mutable runtime instance.
It is a typed UI-producing value.

`view` is therefore a better fit than `component`:

1. It describes the semantic role directly
2. It pairs naturally with `renders`
3. It avoids importing React-style baggage into the language design
4. It scales cleanly to higher-order wrappers and generic view families

By contrast:

1. `component` is familiar, but framework-loaded
2. `widget` is precise but more object-like than necessary
3. `element` names the produced node, not the producer
4. `template` suggests expansion rather than typed values

To reduce keyword pressure, `view` can be introduced as a contextual keyword in declaration and type positions rather than as a globally reserved word.

### Make views explicit

If MoonBit adopts tag syntax only as sugar over ordinary functions, the language loses too much information:

1. Whether a value is intended to be used as a view
2. Which arguments are props
3. Which arguments are slots
4. What the view is guaranteed to render
5. What interface a higher-order wrapper preserves or transforms

Using an explicit `view` keyword makes these properties visible to both programmers and the type checker.

### Avoid the usual generic view trap

In TypeScript, generic components and higher-order components often interact poorly because components are modeled as ordinary callable values plus framework-specific helper types. In practice, wrappers can easily erase generic parameters, widen children too aggressively, or fall back to imprecise `React.Node`-style escape hatches.

MoonBit should avoid that entire class of problems by treating view interfaces as first-class language concepts.

### Keep compile performance predictable

A practical concern with JSX-like syntax is parser mode selection.
Real-world toolchains often keep JSX and non-JSX files separate because that preserves a fast extension-based dispatch and reduces parser ambiguity around `<`.

This proposal therefore recommends an explicit opt-in parse mode for tag syntax, preferably by file extension, while still allowing the feature to compile down to ordinary MoonBit code after type checking.

## Goals

1. Preserve issue #19's explicit child-list design
2. Introduce an explicit `view` declaration form
3. Make view types easy to declare and easy to manipulate
4. Distinguish props from slots in the type system
5. Support generic views without losing precision
6. Make higher-order wrappers preserve exact interfaces by default
7. Keep parser complexity and compile-performance overhead low

## Non-goals

1. Reproducing HTML's whitespace semantics
2. Introducing raw text nodes
3. Reproducing React's hook model in the language
4. Making every ordinary function callable as a tag
5. Solving all UI-library runtime details in v1

## Proposed Solution

### `view` declarations

This proposal introduces a dedicated declaration form:

```moonbit
view Button(
  label~ : String,
  on_click~ : () -> Unit,
) renders Ui::Button {
  <Ui::button label=label on_click=on_click />
}
```

A view declaration has three parts:

1. Props
2. Slots
3. A render contract introduced by `renders`

Slots are declared explicitly:

```moonbit
view Layout(
  title~ : String,
  slot header : renders Ui::Header,
  slot footer : renders? Ui::Footer = None,
  children : renders* Ui::Section = [],
) renders Ui::Page {
  <Ui::page title=title>
    slot header = header,
    ..children,
    slot footer = footer,
  </Ui::page>
}
```

Design notes:

1. `children` is the default slot
2. `slot name` declares a named slot
3. `renders T` means exactly one renderable child of contract `T`
4. `renders? T` means zero or one child of contract `T`
5. `renders* T` means zero or more children of contract `T`

### View type syntax

Views need a source-level type form, not just inference.

This proposal introduces:

```moonbit
type LayoutType = view(
  title~ : String,
  slot header : renders Ui::Header,
  slot footer : renders? Ui::Footer,
  children : renders* Ui::Section,
) renders Ui::Page
```

For higher-order wrapper work, the compiler also exposes a canonical structural view:

```moonbit
type LayoutType = view(
  props {
    title~ : String,
  },
  slots {
    header : renders Ui::Header,
    footer : renders? Ui::Footer,
    children : renders* Ui::Section,
  },
) renders Ui::Page
```

The first form is the ergonomic source syntax.
The second form is the canonical type-level model used for interface extraction and wrapper typing.

## Tag Syntax

### Tag forms

```moonbit
<tag attr1=expr1 attr2=expr2>
  child_item_1,
  child_item_2,
</tag>
```

and

```moonbit
<tag attr1=expr1 attr2=expr2 />
```

The surface syntax remains close to issue #19, but tag expressions are type-checked as view invocations before any lowering step.

### Child item grammar

```text
ChildItem ::= Expr
            | ".." Expr
            | "slot" Ident "=" Expr
            | "slot" Ident "=" ".." Expr
```

Interpretation:

1. `expr` contributes one child to the default slot
2. `..expr` contributes an array of children to the default slot
3. `slot name = expr` contributes one child to a named slot
4. `slot name = ..expr` contributes an array of children to a repeated named slot

### String literals

String literals are explicit expressions:

```moonbit
<Ui::div>
  "hello",
  "world",
</Ui::div>
```

No raw text node syntax is introduced.

### Nested tags

Nested tags remain ordinary expressions:

```moonbit
<Layout title=title>
  slot header = <Header title=title />,
  <Ui::section>"body"</Ui::section>,
</Layout>
```

## Type System

### Views are a distinct kind

This proposal treats `view` as a distinct type-level concept.
A view may compile down to ordinary runtime functions or closures, but the type checker should not model it as "just another function type".

That distinction is what preserves precision for:

1. Named slots
2. Render contracts
3. Exact prop checking
4. Higher-order wrapper transformations
5. Generic view preservation

### Exact prop and slot checking

View invocations are checked against a closed interface by default.

That means:

1. Extra props are rejected
2. Missing required props are rejected
3. Undeclared slots are rejected
4. A `renders` slot must be supplied exactly once unless it has a default
5. A `renders?` slot may be supplied at most once
6. A `renders*` slot may be supplied zero or more times

This default should be strict.
Open-ended "accept anything" behavior should require an explicit type annotation.

### Render contracts

`renders T` is a contract on rendered structure, not just a synonym for `Ui::Node`.

This makes APIs such as slots significantly safer:

```moonbit
view Dialog(
  slot trigger : renders Ui::ButtonLike,
  children : renders* Ui::DialogBody,
) renders Ui::Dialog {
  ...
}
```

Now the compiler can reject invalid trigger content at the call site instead of accepting any node and failing later in the UI layer.

The exact representation of render contracts is left to the implementation, but the type checker must be able to distinguish at least:

1. An unconstrained top type such as `Ui::Node`
2. Nominal or trait-like render contracts such as `Ui::ButtonLike`
3. Transparent render contracts preserved across wrappers

### Slots are not props

Slots should be part of the view interface, but they should not be collapsed into ordinary props.

Treating slots separately gives better diagnostics and better wrapper support:

1. Slot multiplicity can be checked structurally
2. Default slot handling is explicit
3. Named-slot errors are more precise
4. Wrappers can preserve slot interfaces independently of props

## Generic Views

Generic views are a first-class use case:

```moonbit
view[T] ForEach(
  items~ : Array[T],
  render_item~ : (T) -> Ui::Node,
  slot empty : renders? Ui::EmptyState = None,
) renders Ui::Fragment {
  if items.is_empty() {
    match empty {
      Some(view) => view
      None => <Ui::Fragment />
    }
  } else {
    <Ui::Fragment>
      ..items.map(render_item),
    </Ui::Fragment>
  }
}
```

Its type should be expressible directly:

```moonbit
type ForEach[T] = view(
  props {
    items~ : Array[T],
    render_item~ : (T) -> Ui::Node,
  },
  slots {
    empty : renders? Ui::EmptyState,
  },
) renders Ui::Fragment
```

This is intentionally stronger than a framework helper such as `React.FC`.
The generic parameter belongs to the view type itself and must not be erased when the view is wrapped.

## Higher-order Wrappers

Higher-order wrappers are only pleasant to use when wrappers preserve exact interfaces.
This proposal includes that requirement from the beginning.

### Interface extraction

The compiler should expose the following built-in type operators:

1. `ViewOf[T]` for the exact view interface of `T`
2. `PropsOf[T]`
3. `SlotsOf[T]`
4. `RendersOf[T]`

These operators must preserve generic parameters.

### Exact-preserving wrappers

Wrappers such as `memo`, `trace`, or `suspend_boundary` should be able to preserve a view exactly:

```moonbit
fn memo[V](inner : V) -> ViewOf[V] {
  ...
}
```

If `inner` is generic, the result stays generic.
If `inner` has named slots, the result keeps the same named slots.
If `inner` has a precise render contract, the result keeps that contract.

### Interface-transforming wrappers

When a wrapper intentionally changes the public interface, it should say so directly in the type:

```moonbit
fn with_theme[P, S, R](
  theme : Theme,
  inner : view(
    props { theme~ : Theme, ..P },
    slots S,
  ) renders R,
) -> view(
  props P,
  slots S,
) renders R {
  ...
}
```

This example removes `theme` from the public props because the wrapper injects it internally.

Likewise, a wrapper that adds a slot can say so explicitly:

```moonbit
fn with_fallback_slot[P, S, R](
  inner : view(
    props P,
    slots S,
  ) renders R,
) -> view(
  props P,
  slots { fallback : renders? Ui::Fallback, ..S },
) renders R {
  ...
}
```

This is significantly safer than treating wrappers as ordinary function transforms because prop rows, slot rows, and render contracts remain visible.

## Desugaring and Lowering

Issue #19 describes tags as sugar over function calls with labeled arguments and children arrays.
This proposal keeps that spirit, but refines the compilation pipeline:

1. Parse a tag expression in view-syntax mode
2. Resolve the tag target to a view or intrinsic UI constructor
3. Type-check attributes, default-slot children, and named slots against the view interface
4. Lower the checked form to the runtime representation chosen by the implementation

Conceptually:

```moonbit
<Layout title=title>
  slot header = <Header title=title />,
  <Ui::section>"body"</Ui::section>,
</Layout>
```

can lower to something equivalent to:

```moonbit
Layout(
  title=title,
  header=<Header title=title />,
  children=[
    <Ui::section>"body"</Ui::section>,
  ],
)
```

However, the exact runtime encoding is intentionally not standardized here.
The important property is that typing happens at the view level before this representation is erased.

## Parsing Mode and Compile Performance

### Why explicit opt-in matters

The parser cost of JSX-like syntax is not just a grammar bikeshed.
Toolchains often rely on extension-based or loader-based dispatch to avoid paying extra parsing cost on files that do not contain tag syntax.

This is the same general pressure visible in:

1. TypeScript requiring `.tsx` for JSX-enabled files
2. Bun and other build tools selecting loaders from file type or extension
3. Vite discussions around the drawbacks of treating ordinary `.js` files as JSX

MoonBit should learn from that experience instead of forcing all `.mbt` files through a tag-aware parser path.

### Recommended strategy

The reference design should prefer one of the following:

1. A dedicated source extension such as `.mbx`
2. A per-file opt-in mode that enables tag parsing only in files that declare view syntax

If the community strongly prefers a single file extension, the parser should still keep tag parsing gated behind clearly delimited `view` contexts rather than making all expression parsing globally tag-aware.

### Rationale

This preserves:

1. Fast parser dispatch for ordinary files
2. Lower ambiguity around `<`
3. Simpler tooling integration
4. Better incremental compilation characteristics

## Compatibility with Issue #19

This proposal keeps the key decisions from issue #19:

1. Explicit comma-separated child items
2. No raw text nodes
3. `..expr` for repeated children
4. Fragments are optional future work
5. Tag syntax remains an expression-oriented tree-building form

It changes or extends issue #19 in the following ways:

1. Introduces a dedicated `view` keyword
2. Separates props from slots
3. Introduces `renders`, `renders?`, and `renders*`
4. Adds source-level view types
5. Treats views as a distinct kind for typing and wrappers
6. Recommends explicit parse-mode opt-in for performance reasons

## Examples

### Basic view

```moonbit
view Avatar(
  name~ : String,
  size~ : Int = 32,
) renders Ui::Avatar {
  <Ui::avatar name=name size=size />
}
```

### View with named slots

```moonbit
view Card(
  title~ : String,
  slot actions : renders* Ui::ButtonLike = [],
  children : renders* Ui::Section,
) renders Ui::Card {
  <Ui::card title=title>
    ..children,
    slot actions = ..actions,
  </Ui::card>
}
```

Usage:

```moonbit
<Card title="Settings">
  <Ui::section>"Body"</Ui::section>,
  slot actions = <Button label="Save" />,
  slot actions = <Button label="Reset" />,
</Card>
```

### Generic view

```moonbit
view[T] Select(
  value~ : T,
  options~ : Array[T],
  render_option~ : (T) -> Ui::Node,
) renders Ui::Field {
  ...
}
```

### Exact-preserving higher-order wrapper

```moonbit
fn trace[V](name : String, inner : V) -> ViewOf[V] {
  ...
}
```

### Interface-transforming higher-order wrapper

```moonbit
fn with_logger[P, S, R](
  logger : Logger,
  inner : view(
    props { logger~ : Logger, ..P },
    slots S,
  ) renders R,
) -> view(
  props P,
  slots S,
) renders R {
  ...
}
```

## Open Questions

1. Should the opt-in source extension be `.mbx`, `.mbtx`, or another spelling?
2. Should the canonical type form use `props {}` and `slots {}` exactly as written here, or a lighter-weight row syntax?
3. Should a future proposal add fragment sugar such as `<> ... </>`?
4. Should separators inside tag bodies remain `,`, or should MoonBit consider `;` in a separate ASI-focused proposal?

These questions are orthogonal to the main idea and can be resolved independently.

## Conclusion

MoonBit should adopt tag syntax as part of a typed view model, not as a loose parser trick.

Introducing `view` declarations, source-level view types, typed slots, render contracts, and exact wrapper preservation gives MoonBit a path that is:

1. Simpler than JSX in semantics
2. Safer than React-style helper typing
3. Better for generic views
4. Better for higher-order wrappers
5. More respectful of parser and build performance

In short, the language should keep issue #19's explicit child-list design, but place it inside a stronger and more explicit view type system.

## References

1. [MoonBit JSX-like Tag Syntax Proposal, issue #19](https://github.com/moonbitlang/moonbit-evolution/issues/19)
2. [Flow Component Syntax](https://flow.org/en/docs/react/component-syntax/)
3. [Flow Component Types](https://flow.org/en/docs/react/component-types/)
4. [Flow Render Types](https://flow.org/en/docs/react/render-types/)
5. [TypeScript JSX documentation](https://www.typescriptlang.org/docs/handbook/jsx.html)
6. [Vite discussion about using JSX in ordinary `.js` files](https://github.com/vitejs/vite/discussions/3448)
