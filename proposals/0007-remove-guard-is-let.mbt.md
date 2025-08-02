# Remove `guard`, `is`, `let` keyword


## Introduction

In the Moonbit language, the functionality of the `let` and `is` keyword overlaps. We can change `=` to match operator. With this, the `guard` keyword becomes unnecessary.

To avoid confusion between assignment and matching, the `:` is required whether the type is specified or not. We use good old `<~` for assignment.


## match operator

```moonbit
let a = 1234
let b: Int = 1234
guard x is Some(v)
while m is Some(v) {}
guard index >= 0 && index < array.length() else { None }
```

changes to

```moonbit
a := 1234
b : Int = 1234
Some(v) := x
while Some(v) := m {}
if not(index >= 0 && index < array.length()) { return None }
```

## assignment operator

```moonbit
x = x + 1
```

changes to

```moonbit
x <~ x + 1
```
