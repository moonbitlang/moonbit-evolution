# Generics Traits

* Proposal: [ME-0004](https://github.com/illusory0x0/moonbit-evolution/blob/0004-generics-traits/proposals/0004-generics-traits.mbt.md)
* Author: [illusory0x0](https://github.com/illusory0x0), [Erchiusx](https://github.com/Erchiusx)
* Status: Under review
* Review and discussion: [Github issue](https://github.com/moonbitlang/moonbit-evolution/pull/5)

## Introduction

We use type to constrain the value, and trait to constrain the type, but now supports generic data structures, but not generic traits which leads to a lot of API consistency issues when using generic data structures.

On the other hand, JavaScript has many APIs that require generic traits, the underlying DOM, and other web front-end frameworks make heavy use of generic classes, such as `react.js`, `vue.js`.

Although using `cast` function can solve some problems, it is not good practice to use `cast`  function heavily.


## Motivation

### Check API consistency

Right now Moonbit's trait doesn't support generic parameters, 
which can lead to a lot of API inconsistencies when using generic data structures.

for example, 
[Moonbit core library](https://github.com/moonbitlang/core/issues?q=is%3Aissue%20label%3A%22consistency%20review%22) has 
**29** consistency review issues, if we have **Generics Traits**, the compiler can check API consistency without human to code review.

### Improving JavaScript API Interoperability

Many JavaScript APIs use [Generic Classes](https://www.typescriptlang.org/docs/handbook/2/generics.html#generic-classes) 
or [Generic Interfaces](https://www.typescriptlang.org/docs/handbook/2/generics.html#generic-types).

Using the most commonly used API `querySelector` as an example. We have to write `DOM_Element::downcast` to cast.

[rescript webapi](https://github.com/TheSpyder/rescript-webapi/) using sophisticated techniques to emulate subtying and inheritance.

[purescript-dom-classy](https://pursuit.purescript.org/packages/purescript-dom-classy/) using typeclass to make DOM API binding more ergonomic.

[jsoo dom](https://github.com/ocsigen/js_of_ocaml/blob/master/lib/js_of_ocaml/dom.ml) using OCaml object inheritance 
to implement DOM API binding whose experience is closest to JavaScript.

Moonbit's type system is much closer to PureScript, 
and the experience with traits/typeclass is still quite good, 
so it was necessary to enhance the ability of the trait.

### Improve performance via avoiding dynamic dispatch 

Now that the `Show` trait calls the `&Logger` trait object using dynamic dispatch can affect performance.

```moonbit skip
pub(open) trait Show {
  output(Self, &Logger) -> Unit
  to_string(Self) -> String = _
}
```


```moonbit skip
///|
#external
type Element

///|
trait DOM_Element {
  querySelector(Self, String) -> Element = _
  downcast(Element) -> Self = _
  to_element(Self) -> Element
}

///|
extern "js" fn Element::querySelector(
  self : Element,
  selectors : String
) -> Element = "(self,selector) => self.querySelector(selector)"

///|
fn[A, B] coerce(x : A) -> B = "%identity"

///|
impl DOM_Element with querySelector(self, selectors) {
  Element::querySelector(coerce(self), selectors)
}

///|
impl DOM_Element with downcast(self) = "%identity"

///|
#external
type HTMLCanvasElement

///|
#external
type HTMLDivElement

///|
#external
type HTMLAnchorElement

///|
#external
type HTMLImageElement

///|
impl DOM_Element for HTMLCanvasElement with to_element(self) = "%identity"

///|
impl DOM_Element for HTMLDivElement with to_element(self) = "%identity"

///|
impl DOM_Element for HTMLAnchorElement with to_element(self) = "%identity"

///|
impl DOM_Element for HTMLImageElement with to_element(self) = "%identity"

///|
test {
  let x : HTMLDivElement = {
    ...
  }
  let e : HTMLImageElement = x.querySelector("x") |> DOM_Element::downcast

}
```




## Proposed solution


### Functor Example 


This makes it easy to use the compiler to check for API consistency.

Actually this `Functor` more similar to [traverse](https://hackage.haskell.org/package/base-4.21.0.0/docs/Prelude.html#v:traverse) when passing callback function which raise error.

```moonbit skip
trait Functor[A] {
  fn[B] map(self : Self[A], f : (A) -> B raise?) -> Self[B] raise?
}

trait Monad[A] : Functor[A] {
  fn[B] bind(self : Self[A], f : (A) -> Self[B] raise?) -> Self[B] raise?
  pure(x : A) -> Self[A]
}
```

without type annotation impl.

```moonbit skip
impl Functor for FixedArray with map(self, f) { ... }
impl Monad for FixedArray with bind(self, f) { ... }
impl Monad for FixedArray with pure(a) { ... }

impl Functor for Result with map(self, f) { ... }
impl Monad for Result with bind(self, f) { ... }
impl Monad for Result with pure(a) { ... }
```

with type annotation impl.

```moonbit skip
impl Functor for FixedArray with[A,B] map(self : Self[A], f : (A) -> B raise?) -> Self[B] raise? { ... }
impl Monad for FixedArray with[A,B] bind(self : Self[A], f : (A) -> Self[B] raise?) -> Self[B] raise? { ... }
impl Monad for FixedArray with[A] pure(x : A) -> Self[A] { ... }

impl Functor for Result with[A,B] map(self : Self[A], f : (A) -> B raise?) -> Self[B] raise? { ... }
impl Monad for Result with[A,B] bind(self : Self[A], f : (A) -> Self[B] raise?) -> Self[B] raise? { ... }
impl Monad for Result with[A] pure(x : A) -> Self[A] { ... }
```


with generic traits, we can list the list of such traits that types are meant to belong, and thus improve API consistency.

### Iterable Example 

We can define `Iterable` trait to `derive` something like `C#` [IEnumerable<T>](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.ienumerable-1?view=net-9.0) or `Haskell` [Foldable](https://hackage.haskell.org/package/base-4.21.0.0/docs/Prelude.html#t:Foldable) reduce many boilerplates and keep API consistency.

Use traits as API design guidelines, just as we did with traits that weren't generalized before.

C++ using [Range Concepts](https://en.cppreference.com/w/cpp/header/ranges.html#Concepts) and [Iterator Concepts](https://en.cppreference.com/w/cpp/iterator.html) as API design guidelines, If someone implement this convention, then they enjoy the benefits of [algorithm](https://en.cppreference.com/w/cpp/algorithm.html) which is common data structures operation.

```moonbit skip

trait Iterable[A] {
  iter(Self[A]) -> Iter[A]
  each(Self[A], f : (A) -> Unit raise?) -> Unit raise? = _ 
  eachi(Self[A], f : (Int,A) -> Unit raise?) -> Unit raise? = _
  fn[S] fold(Self[A], init~ : S, f : (S,A) -> S raise?) -> S raise? = _
  find_first(Self[A], f : (A) -> Bool) -> T? = _
  contains(Self[A],value : A) -> Bool = _ 
}

trait KnownSizedIterable[A] : Iterable[A] {
  size(Self[A]) -> Int 
  collect(Self[A]) -> Array[A] = _ 
  // This performance is better than `Iter::collect`
  to_array(Self[A]) -> Array[A] = _ 
  to_fixedarray(Self[A]) -> FixedArray[A] = _ 

}

```

we also can improve performance for `Array::push_iter`.

```moonbit skip 
fn[A,I : KnownSizedIterable] Array::push_iter(self : Self[A], xs : I[A]) -> Unit {
  self.reserve_capacity(self.length() + xs.size())
  for x in xs {
    self.push(x)
  }
}
```

### querySelector Example

support generics method in traits can reduce many `downcast` call.

```moonbit skip 
///|
#external
type Element

///|
trait DOM_Element {
  fn[E : DOM_Element] querySelector(Self, String) -> E = _
  to_element(Self) -> Element
}

///|
extern "js" fn Element::querySelector(
  self : Element,
  selectors : String
) -> Element = "(self,selector) => self.querySelector(selector)"

///|
fn[A, B] coerce(x : A) -> B = "%identity"

///|
impl DOM_Element with querySelector(self, selectors) {
  coerce(Element::querySelector(coerce(self), selectors))
}

///|
#external
type HTMLCanvasElement

///|
#external
type HTMLDivElement

///|
#external
type HTMLAnchorElement

///|
#external
type HTMLImageElement

///|
impl DOM_Element for HTMLCanvasElement with to_element(self) = "%identity"

///|
impl DOM_Element for HTMLDivElement with to_element(self) = "%identity"

///|
impl DOM_Element for HTMLAnchorElement with to_element(self) = "%identity"

///|
impl DOM_Element for HTMLImageElement with to_element(self) = "%identity"

///|
test {
  let x : HTMLDivElement = {
    ...
  }
  let e : HTMLImageElement = x.querySelector("x") 

}
```

### Show trait

```moonbit skip 
pub(open) trait Show {
  fn[L : Logger] output(Self,L) -> Unit
  to_string(Self) -> String = _
}
```

## Possible alternatives

use [virtual packages](https://docs.moonbitlang.com/en/latest/language/packages.html#virtual-packages) to check API consistency, 
like OCaml [module signature](https://ocaml.org/manual/5.3/moduleexamples.html#s:signature), but module signature only can check module, can not check for type.
