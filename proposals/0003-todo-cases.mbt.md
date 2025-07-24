# Support `...` in match cases

- Proposal: [ME-0003](https://github.com/moonbitlang/moonbit-evolution/blob/0003-todo-cases/proposals/0003-todo-cases.mbt.md)
- Author: [Yu-zh](https://github.com/Yu-zh)
- Review and discussion: [GitHub issue](https://github.com/moonbitlang/moonbit-evolution/pull/4)

## Introduction

In MoonBit, we can use `...` as a placeholder for unimplemented code. For example:

```moonbit
fn init {
  fn unimplemented() {
    ...
  }
}
```

The compiler emits a warning when the `...` is used, but the code will be
further compiled. If `...` is reached at runtime, the program will panic.

## Allow `...` in match cases

Currently, `...` is only allowed when a statement is expected. Therefore, if we
want leave the code in match cases unimplemented, we have to write `_ => ...`:

```moonbit
pub fn f1(x : Int) -> Unit {
  match x {
    0 => ()
    ...
  }
}
```

We propose to extend the support to match cases so that we can write `...`
directly:

```moonbit
pub fn f2(x : Int) -> Unit {
  match x {
    0 => ()
    ...
  }
}
```

Note this means `...` can be used in any action case, including such cases:

```moonbit
pub fn f3() -> Unit {
  ...
  try {
    ...
  } catch {
    ...
  } noraise {
    ...
  }
}
```
