# JavaScript Interoperability

- Proposal: [ME-0010](https://github.com/illusory0x0/moonbit-evolution/blob/0010-javascript-interoperate/proposals/0010-javascript-interoperate.mbt.md)
- Author: [illusory0x0](https://github.com/illusory0x0)
- Status: Under review
- Review and discussion: [Github issue](https://github.com/moonbitlang/moonbit-evolution/pull/13)

## Introduction

TypeScript leverages union types and function types with rest parameters extensively to describe JavaScript APIs. However, MoonBit's current JavaScript FFI lacks robust support for these essential type patterns, limiting effective interoperability.

## Motivation

### Union Types

The current approach uses phantom types as documentation without type checking capabilities, preventing **AI agents** from detecting errors in generated code through static analysis. Type conversion currently relies solely on `unsafe_coerce`.

```mbt
#external
pub(all) type Union2[_, _]
```

This approach presents several limitations:
- Requires defining multiple `UnionN` types to support different union arities
- Lacks built-in type safety guarantees
- Creates verbose boilerplate code

A built-in `Union` type supporting arbitrary numbers of union members would be more elegant, similar to how tuple types work.

### Function Rest Parameters

While `FuncRef` serves FFI purposes in the native backend, JavaScript FFI lacks an equivalent for function types with rest parameters. The current `Func1Rest[Thenable[String?], String, String]` syntax suffers from poor readability and ergonomics.

Consider this TypeScript function signature:

```typescript 
function showInformationMessage(message : string, ...items : string[]) : Thenable<string | undefined> {
}
```

The proposed `FuncRest[(String, Array[String]) -> Thenable[String?]]` syntax would provide a much cleaner representation.

Current limitations include cumbersome calling conventions and conversion requirements:

```mbt skip 
test {
    let message = vscode.window.showInformationMessage.into()
    // Explicit conversion to MoonBit function type required
    message("Hello moonbit", []) |> ignore
}
```

```mbt
///|
#external
pub(all) type Func1Rest[R, E, A1]

///|
type Thenable[_]

///|
struct Foo {
  showInformationMessage : Func1Rest[Thenable[String?], String, String]
}
```

## Proposed Solution

### Union Type Enhancement

Introduce a built-in `Union` type that accepts arbitrary numbers of type parameters:

```mbt skip 
test {
    let _ : Union[String,Int,Bool] = "hello" // Union type
    let _ : Union[String,Int,Bool] = 42
    let _ : Union[String,Int,Bool] = true
}
```

### FuncRest Type

Implement `FuncRest` as a marker type that generates functions conforming to JavaScript calling conventions:

```mbt skip
test {
    let message : FuncRest[(String, Array[String]) -> Thenable[String?]] = vscode.window.showInformationMessage
    message("Hello moonbit", []) |> ignore
    // FuncRest serves as a marker for JavaScript calling convention generation
}
```

## Concrete Use Cases

[vscode.mbt](https://github.com/moonbit-community/vscode.mbt)

[js-ffi](https://github.com/moonbit-community/js-ffi)