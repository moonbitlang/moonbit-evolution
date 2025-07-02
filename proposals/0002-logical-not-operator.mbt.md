# Logical not operator

* Proposal: [ME-0002](https://github.com/moonbitlang/moonbit-evolution/blob/0002-orsuccess/proposals/0002-logical-not-operator.mbt.md)
* Author: [Yorkin](https://github.com/Yoorkin)
* Review and discussion: [Github issue](https://github.com/moonbitlang/moonbit-evolution/pull/3)

## Introduction

In MoonBit, the `not` function takes a boolean value and returns its negation, 
as implemented in `moonbitlang/core` . We propose the `!expr` syntax to negate a 
boolean expression, which aligns with the conventions of other programming languages.

## Motivation

Since `not` is a function, boolean expressions must be wrapped in parentheses 
when used. This can be cumbersome and deviates from the conventions of other 
programming languages that use a logical operator for negation. Furthermore, 
we already have `&&` and `||` for logical operations. This inconsistency may 
confuse users from diverse programming backgrounds and lead to misunderstandings 
by LLMs during code generation.

## Solution

We propose introducing a logical operator `!` that can be used in expressions 
to negate a boolean value:

```moonbit
fn f(x : Bool) -> String {
  if !x {
    "false"
  } else {
    "true"
  }
}

let false_value = !true
let true_value = !!true // !(!true)
```

The `not` function will still be available for pipeline syntax and use as a 
higher-order function.
