# Tuple struct

* Proposal: [ME-0008](https://github.com/moonbitlang/moonbit-evolution/blob/0008-tuple-struct/proposals/0008-tuple-struct.mbt.md)
* Author: [Yu-zh](https://github.com/Yu-zh)
* Status: Implemented
* Review and discussion: [GitHub issue](https://github.com/moonbitlang/moonbit-evolution/pull/11)

## Introduction

Currently, MoonBit uses the `type` syntax for implementing the newtype idiom, which creates a new type that wraps an existing type. However, the current syntax has several issues, particularly when dealing with tuple types. We propose to replace the `type` syntax with a more intuitive `tuple struct` syntax that provides clearer semantics and better ergonomics while maintaining the newtype pattern.

## Motivation

### Problems with current type syntax

The current `type` syntax for implementing the newtype idiom with tuple types creates ambiguity and inconsistency:

```moonbit
type Newtype Int
type NewtypeTuple (Int, Int)
```

For tuple types, the semantics are unclear and there are multiple ways to construct and access elements:

#### Construction ambiguity
```moonbit
fn make_newtype_tuple1(x: Int, y: Int) -> NewtypeTuple {
  (x, y)  // Direct tuple conversion
}

fn make_newtype_tuple2(x: Int, y: Int) -> NewtypeTuple {
  NewtypeTuple((x, y))  // Constructor with extra parentheses
}
```

#### Access ambiguity
```moonbit
fn get_x1(t: NewtypeTuple) -> Int {
  t.0  // Direct field access
}

fn get_x2(t: NewtypeTuple) -> Int {
  t.inner().0  // Using .inner() method then field access
}

fn get_x3(t: NewtypeTuple) -> Int {
  match t {
    NewtypeTuple((x, _)) => x  // Pattern matching
  }
}
```

These inconsistencies become even more problematic when dealing with value types and create cognitive overhead for developers. The newtype pattern should provide clear, unambiguous semantics.

## Proposed solution

We propose introducing `tuple struct` syntax as a replacement for the `type` syntax when implementing the newtype idiom:

```moonbit
struct Single(Int)
struct Multiple(Int, String, Char)
```

### Benefits of tuple struct syntax

1. **Clearer semantics**: Tuple structs have unambiguous construction and access patterns
2. **Consistent access**: Use `.0`, `.1`, etc. for all tuple structs
3. **Simpler construction**: Constructor takes the same number of arguments as the struct definition
4. **Better value type support**: Easier to implement and reason about
5. **Maintains newtype pattern**: Still provides type safety and distinct semantics from the underlying type

### Construction and access

```moonbit
struct Multiple(Int, String, Char)

fn make_multiple(a: Int, b: String, c: Char) -> Multiple {
  Multiple(a, b, c)  // Clear constructor syntax
}

fn use_multiple(x: Multiple) -> Unit {
  println(x.0)  // Access first element
  println(x.1)  // Access second element
  println(x.2)  // Access third element
}
```

## Special cases and features

### Single-element tuple structs

Single-element tuple structs are guaranteed to be unboxed at runtime and provide several convenient shorthands, maintaining the efficiency benefits of the newtype pattern:

```moonbit
struct S {
  a: Int
}

struct R(S)

fn get_a(r: R) -> Int {
  r.a  // Automatic field forwarding
}
```

### Function type support

When the element type is a function, tuple structs provide seamless function application:

```moonbit
struct F((Int) -> Int)

fn apply(f: F, x: Int) -> Int {
  f(x)  // No need for .0 when applying
}
```

## Migration strategy

### Single-element types

For single-element types where the underlying type is not a tuple, the formatter automatically migrates to the new syntax:

```moonbit
// Old syntax
type A1 Int
fn A1::get(a: A1) -> Int {
  a.inner()
}

// New syntax (automatically migrated)
struct A2(Int)
fn A2::get(a: A2) -> Int {
  a.0
}
```

To facilitate migration, single-element tuple structs temporarily provide a `.inner()` method, which will be deprecated and removed in future versions.

### Multi-element types

Multi-element tuple structs differ from the old tuple type syntax in important ways:

1. **No direct tuple construction**: Tuple structs cannot be constructed directly from tuples
2. **No .inner() method**: Tuple structs do not provide access to the underlying tuple

### Compatibility with tuple conversion

If you need a tuple struct that can be directly converted to and from tuples while maintaining the newtype pattern, you can wrap the tuple type:

```moonbit
struct T((Int, Int))

fn make_t(x: Int, y: Int) -> T {
  (x, y)  // Direct tuple construction
}

fn use_t(t: T) -> (Int, Int) {
  t.0  // Access the wrapped tuple
}

// Access individual elements requires nested access
fn get_x(t: T) -> Int {
  t.0.0  // t.0 gets the tuple, .0 gets the first element
}
```

## Conclusion

The tuple struct syntax provides a cleaner, more intuitive alternative to the current `type` syntax while maintaining the newtype idiom's benefits. It eliminates ambiguity in construction and access patterns while preserving type safety and distinct semantics from the underlying types. The migration strategy ensures backward compatibility while encouraging adoption of the new syntax.

This proposal aligns with MoonBit's goal of providing a clear, consistent language design that reduces cognitive overhead for developers while maintaining the powerful newtype pattern for type safety and domain modeling.
