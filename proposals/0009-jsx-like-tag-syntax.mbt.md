# MoonBit JSX-like Tag Syntax

* Proposal: [ME-0009](https://github.com/moonbitlang/moonbit-evolution/blob/0009-jsx-like-tag-syntax/proposals/0009-jsx-like-tag-syntax.mbt.md)
* Author: TBD
* Status: Under review
* Review and discussion: TBD

## Introduction

This proposal formalizes a JSX-like tag syntax for MoonBit, designed specifically for UI-oriented tree construction while remaining consistent with MoonBit's expression-oriented semantics. The syntax introduces `<tag ...> ... </tag>` as declarative sugar over existing function calls with labeled arguments and children arrays.

## Motivation

Building UI trees in MoonBit currently requires verbose function call syntax. A declarative JSX-like syntax would:

- Make UI code more readable and maintainable
- Align with developer familiarity from React/JSX ecosystems
- Enable better tooling support for UI development

The goals of this proposal are:

- Minimal grammar extension
- No HTML-style whitespace semantics
- Explicit, unambiguous child node specification
- Easy mapping to MoonBit's strongly typed UI DSL

## Proposed Solution

### 1. Core Syntax

#### 1.1. Tag Form

```
<tag attr1=expr1 attr2=expr2>
    child_item_1,
    child_item_2,
    ...,
    child_item_N
</tag>
```

Desugars to:

```moonbit
tag(
  attr1 = expr1,
  attr2 = expr2,
  children = build_children(
    child_item_1,
    child_item_2,
    ...,
    child_item_N,
  )
)
```

#### 1.2. Self-closing Form

```
<tag attr1=expr1 attr2=expr2 />
```

Desugars to:

```moonbit
tag(attr1=expr1, attr2=expr2, children=[])
```

### 2. Children Rules

Children inside a tag are an explicit **comma-separated list** of *child items*.

This avoids all HTML/JSX whitespace ambiguity.

#### 2.1. ChildItem Grammar

```
ChildItem ::= Expr
            | ".." Expr     // spread
```

#### 2.2. Interpretation

- `expr`  
  Must evaluate to a **single child node** (e.g. `Html[Msg]`).

- `..expr`  
  Must evaluate to `Array[Html[Msg]]` (or more generally: an array of the node type).  
  Its elements are appended into the children list.

### 3. Semantics of `build_children`

This helper is conceptual; the compiler may inline or optimize it.

Given:

```
<parent>
    a,
    ..bs,
    c,
</parent>
```

The effective children array is:

```moonbit
[a] + bs + [c]
```

#### 3.1. Type-checking Rules

- For plain child `e`:  
  `e : Child` (typically `Html[Msg]`).

- For spread child `..e`:  
  `e : Array[Child]`.

Errors should point directly at the mismatched child item.

### 4. String Literals

Inside tag bodies, string literals are explicit:

```
<span>
    text("Hello"),
</span>
```

No raw-text node syntax is introduced.  
This avoids HTML-like whitespace normalization rules and prevents ambiguous parsing.

### 5. Nested Tags

Children may themselves be tags:

```
<div>
    <span>
        text("Hello"),
    </span>,
</div>
```

Desugars naturally:

```moonbit
div(
  children = build_children(
    span(
      children = build_children(
        text("Hello"),
      )
    ),
  )
)
```

### 6. Spread Examples

#### Dynamic List Generation

```
<ul>
    ..items.map(fn (item) { <li> text(item) </li> }),
</ul>
```

#### Mixed Static + Dynamic

```
<ul>
    <li> text("First") </li>,
    ..middle_items.map(fn (item) { <li> text(item) </li> }),
    <li> text("Last") </li>,
</ul>
```

### 7. Fragments (Optional Future Extension)

A fragment form may be introduced later:

```
<>
    <A />,
    <B />,
</>
```

Desugaring options:

- As a raw children array, or
- As a call to a standard `fragment(...)`.

Not required for v1.

### 8. Self-closing Tags

```
<input type="text" />
```

Always desugars to:

```moonbit
input(type="text", children=[])
```

## Advantages

### Explicit Commas

- Simplifies grammar
- No whitespace-based text nodes
- Formatter/LSP friendly
- Predictable for humans and AI tools

### Spread Syntax (`..expr`)

- Enables natural `map`-based list expansion
- Matches JS/TS spread mental model
- Easy to type-check

### Strongly Typed Children

- MoonBit's type checker ensures UI trees are valid at compile time

### Minimal Compiler Complexity

- No HTML-like whitespace rules
- No multi-mode tokenizer
- Children list is just expression list parsing

## Open Questions (Future Extensions)

- Do we want `<></>` fragments in v1?
- Should `children=` be renamed (e.g., `kids`, `body`)?  
  (The desugaring will standardize it internally anyway.)
- Should the node type be abstracted behind a trait (`IntoChild`)?

These can be deferred until UI DSL stabilization.

## Note on Separators

We may change `,` to `;` to take advantage of ASI (Automatic Semicolon Insertion).

## Conclusion

This design:

- Covers the full needs of MoonBit's UI tree building
- Avoids the complexity issues of real JSX/HTML
- Aligns with MoonBit's expression-oriented design
- Keeps the syntax minimal, orthogonal, and future-proof

It is ready for prototyping and implementation.
