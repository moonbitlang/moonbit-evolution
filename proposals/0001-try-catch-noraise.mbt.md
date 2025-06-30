# Change `try-catch-else` to `try-catch-noraise`

* Proposal: [ME-0001](https://github.com/moonbitlang/moonbit-evolution/blob/0001-orsuccess/proposals/0001-try-catch-noraise.mbt.md)
* Author: [Yu-zh](https://github.com/Yu-zh)
* Status: Under review
* Review and discussion: [Github issue](https://github.com/moonbitlang/moonbit-evolution/pull/2)

## Introduction

The usage of `else` keyword in the `try` expression is not consistent with its
usage in the `guard` and `if` expressions. This inconsistency is confusing and
adds cognitive load for users. We propose to change the keyword `else` to
`noraise` in the `try` expression for more clarity.

## Motivation

In MoonBit, we can use `else` to continue the computation after the expressions
in the `try` block when no error is raised. The `else` block contains match
cases for the result of the `try` block. For example:
```moonbit
fn f() -> Int raise { ... }
fn init {
    try {
        f()
    } catch {
        e => println(e)
    } else {
        i => println(i)
    }
}
```

With this design, the code in
the `try` block can be pinpointed to the code that potentially raises an error,
and the rest of the code is left in the `else` block. So the code for error
handling is more structured and easier to follow.

The `else` in the `try` expression is followed by a sequence of cases to
immediately match the result of the `try` block. This is consistent with the
`catch` cases which match the error raised in the `try` block. However, the
`else` in the `if` and `guard` expressions are followed by an expression to be
executed when the condition is not met. Therefore, it becomes a mental burden
for users to remember the different usages of `else` in different situations.

To make things worse, pattern cases like `i => println(i)` is a valid arrow
function expression, which makes the `else` in the `try` expression more
confusing.

## Proposed solution

We propose to change the keyword `else` to `noraise` in the `try` expression
for more clarity.

```moonbit
fn init {
    try {
        f()
    } catch {
        e => println(e)
    } noraise {
        i => println(i)
    }
}
```

The keyword `noraise` solves the aforementioned problem, and clearly indicates
it runs only if the `try` block succeeds and the `catch` block is skipped. 

This change is also minimal because this is only a substitution of the keyword,
and the formatter can help with the migration.

## Possible alternatives

### Remove `else` keyword

In most cases, the expression
```
try { expr } catch { err_cases } else { success_cases }
```
can be transformed to
```
try { match expr { success_cases }} catch { err_cases }
```

However, this transformation will make the error raised in the `success_cases`
be unexpectedly caught by the `catch` block. For example,

```moonbit
fn g(x: Int) -> Unit raise { ... }
fn h() -> Unit raise {
    try {
        f()
    } catch {
        e => println(e)
    } else {
        i => g(i) // this cannot be moved into the `try` block
    }
}
```

Therefore, removing `else` in the `try` expression will result in a loss of
expressiveness.

### Combine the patterns in the `catch` block and the `else` block

This is similar to the OCaml's approach to match the expression with possible
exception. For example, the alternative syntax would be:
```moonbit
fn h1() -> Unit raise {
    try { ... } catch {
        Err1 => ...
        Err2 => ...
        _ => ...
        noraise i => ...
    }
}
```

There are several potential problems with this approach: 
* It is weird that there are cases following the `_` case. 
* The keyword `noraise` is awkward because another way to understand it is that
  the branch following `noraise` does not raise an error.
* This will introduce a complicated change to the current codebase.

### Add payload to the `catch` and `else` block

For example, the alternative solution would be:

```moonbit
fn h2() -> Unit raise {
    try { ... } catch (e) {
        match e {
            Err1 => ...
            Err2 => ...
            _ => ...
        }
    } else (i) {
        println(i)
    }
}
```

Several problems with this approach:
* It adds another level of indentation when match on the error compared to the
  current syntax.
* We have two kinds of blocks now: the one with match cases and the one with
  expression. This syntax will add another kind of block, which potentially complicates the syntax overall.
* This will also introduce a complicated change to the current codebase.