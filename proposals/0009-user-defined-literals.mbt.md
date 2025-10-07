# User defined literals

* Proposal: [ME-0009](https://github.com/illusory0x0/moonbit-evolution/blob/0009-user-defined-literals/proposals/0009-user-defined-literals.mbt.md)
* Author: [illusory0x0](https://github.com/illusory0x0)
* Status: Under review
* Review and discussion: [Github issue](https://github.com/moonbitlang/moonbit-evolution/pull/12)

## Introduction

Let's look at this problem: to handle data in different number bases, we have to define many helper functions. However, using `Int` directly can be easily confused, and we need to provide more type-safe methods, such as using `tuple struct` syntax to define newtypes.

```mbt  
fn multiply_with_kb(times : Int, kb : Int) -> Int {
    times * (kb * 1024)
}

fn multiply_with_mb(times : Int, mb : Int) -> Int {
    times * (mb * 1024 * 1024)
}
```

After wrapping with `tuple struct` syntax, `Kb(1)` becomes syntactically ugly. If MoonBit introduces user-defined literals, writing such code would have better readability.

For example, directly writing `8kb`, `512mb` would be more readable than `Kb(8)` and `Mb(512)`. Kotlin supports extension methods and getter/setter property syntax, allowing you to write `8.kb` and `512.mb` with good readability as well.

```mbt
struct Kb(Int)
struct Mb(Int)

trait Multiply {
    multiply(Self, Int) -> Self 
}

pub fn[A : Multiply] multiply(times : Int, base : A) -> A {
    A::multiply(base, times)
}

impl Multiply for Kb with multiply(base,times) {
    times * base.0 
}

impl Multiply for Mb with multiply(base,times) {
    times * base.0 
}

test {
    let _ = multiply(2, Kb(1)) // 2048
    let _ = multiply(2, Mb(1)) // 2097152
}
```

## Motivation

Supporting user-defined literals would make code more readable in many scenarios.

For example, web frontend development has many units like `px`, `em`, `rem`, etc. If we could directly use syntax like `8px`, `1.5em`, `100rem`, it would be more intuitive and readable than using `Px(8)`, `Em(1.5)`, `Rem(100)`.

## Proposed solution

```mbt skip 

// Here we introduce a new namespace to define user-defined literals

#user_defined_literal(kb)
fn kilobyte(x : Int) -> Kb {
    Kb(x)
}

test {
    let kb = 0 // The kb here doesn't conflict with the user_defined_literal above, 
              // the kb below will be looked up from the user_defined_literal namespace 
              // to complete name resolution.
    let val = 8kb // This will be transformed by the compiler to kilobyte(8)
}

```