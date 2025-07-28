# Introduce `defer` syntax for resource cleanup

* Proposal: [ME-0006](https://github.com/moonbitlang/moonbit-evolution/blob/0006-defer/proposals/0006-defer.md)
* Author: [Guest0x0](https://github.com/Guest0x0)
* Revew and discussion: [GitHub issue](https://github.com/moonbitlang/moonbit-evolution/pull/9)

## Introduction
Currently, MoonBit lacks a way to reliably cleanup resource.
This document proposes introducing a `defer` syntax, as found in Go/Swift/Zig,
to ease resource cleanup in MoonBit

## Motivation
Consider a piece of program with error,
it is very common that we want to perform some resource cleanup when the program exits,
no matter normally or due to an error.
Currently, this can only be achieved by writing an error handler with `try`,
and duplicate the cleanup code in the `catch` part and `noraise` part of `try`.
This way of resource cleanup is verbose and error-prone.
If the program contains early exit, such as `return` or `break`,
it would be even more difficult and cumbersome to properly perform resource cleanup.

## Proposed solution
We propose introducing the `defer` syntax, as found in Go/Swift/Zig,
to perform reliable resource cleanup.
The syntax is as follows:

```moonbit
defer expr
body
```

`defer` is a statement-like construct, just like `let`.
The semantic is:
`expr` will be executed immediately after `body` exits,
regardless of how `body` terminates (normally, via error or via early exit constructs such as `return`).

There are several implications of the above semantic:

- the proposed semantic here is **lexically-scoped** `defer`, similar to Swift/Zig and unlike Go.
- consecutive `defer` blocks are executed in reverse order. For example, in the following:

    ```moonbit
    defer expr1
    defer expr2
    body
    ```

    After `body` exits, `expr2` will be executed first.

All control flow consrtucts, such as `return`/`break`/`continue`,
are disallowed in the right hand side of `defer` (`expr` above).
Raising error or performing `async` operations are also disallowed, as in Swift.
But the restriction on error/`async` may be lifted in the future,
it practical need arises.

## Possible alternatives

### `try .. catch .. finally`
`try .. catch .. finally` is an extension to current `try .. catch` syntax.
In `try A catch B noraise C finally D`:

- if `A` fail with error, `B` and `D` will be executed
- if `A` succeed without error, `C` and `D` will be executed
- if `B` or `C` raises error, `D` will still get executed

`try .. catch .. finally` can provide reliable resource cleanup for program with error,
but it cannot handle early exit constructs such as `return`.
`try .. catch .. finally` is also more verbose than `defer`,
and requires an extra layer of indentation for the expression after `try`.
Thus, we consider `defer` more superior compared to `try .. catch .. finally`.

### `with`/`using`
`with` or `using` is also a syntax for reliable resource cleanup.
Python and C# are examples of languages that use this syntax.
The syntax is as follows (take `using` as example):

```moonbit
using expr as resource
body
```

Compared to `defer`, `using` combines resource creation and cleanup:
the expression after `using` creates a resource that can be used in `body`,
and after `body` exits, the cleanup method of the resource object will be invoked.
`using` can be simulated by `defer` as follows:

```moonbit
let resource = expr
defer resource.clenaup()
body
```

Compared to `defer`, `using` is more concise,
as it combines resource creation and cleanup into one construct,
and avoid the need to explicitly write down cleanup code.
But `defer` has its own advantage too:

- `defer` is a pure control-flow construct,
  while `using` requires a pre-defined protocol for cleanup,
  such as a trait or a magic method.
  So `defer` is more modular and explicit
- it is easier to have custom cleanup logic in `defer`.
  In `using`, custom cleanup logic must be implemented by creating a new type,
  and attach custom logic to the type.
  This is more verbose and non-local (the cleanup logic need to be placed elsewhere)
- it is easier to extend `defer` to support raising error and `async`,
  while in `using`, support cleanup with error/`async` requires
  changing or even duplicating the cleanup protocol

Hence we favor `defer` for its simplicity and extensibility.
