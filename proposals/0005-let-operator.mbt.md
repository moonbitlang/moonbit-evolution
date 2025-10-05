# Let Operator

* Proposal: [ME-0005](https://github.com/illusory0x0/moonbit-evolution/blob/0005-let-operator/proposals/0005-let-operator.mbt.md)
* Author: [illusory0x0](https://github.com/illusory0x0)
* Status: Under review
* Review and discussion: [Github issue](https://github.com/moonbitlang/moonbit-evolution/pull/6)

## Introduction

The introduction of binding operators can significantly reduce the depth of nesting when writing Monadic functions.

[quickcheck](https://github.com/moonbitlang/quickcheck/blob/main/src/gen.mbt#L141) and [simple_parserc](https://github.com/moonbit-community/simple_parserc/blob/master/src/monadic.mbt#L2)
rely heavily on Monadic functions, especially the quickcheck library.


## Motivation

Writing Monadic-related functions now would result in over-nested.

for example in [Parser Combinator](https://github.com/moonbit-community/elaboration_zoo.mbt/blob/master/src/03-holes/parser.mbt#L55-L139)

```moonbit 
and let_ = fn(input : Input) -> (Raw, Input) raise ParseError {
  let parser = symbol("let")
    .discard_left(
      name.bind(fn(x) {
        symbol(":")
        .discard_left(expr)
        .bind(fn(a) {
          symbol("=")
          .discard_left(expr)
          .bind(fn(t) {
            symbol(";").discard_left(expr).map(fn(u) { RLet(x, a, t, u) })
          })
        })
      }),
    )
    .label("let")
  parser(input)
}
```

```moonbit 
pub fn[A, B, C, D, E, F, G] liftA6(
  ff : (A, B, C, D, E, F) -> G,
  v : Gen[A],
  w : Gen[B],
  x : Gen[C],
  y : Gen[D],
  z : Gen[E],
  u : Gen[F]
) -> Gen[G] {
  v.bind(fn(a) {
    w.bind(fn(b) {
      x.bind(fn(c) {
        y.bind(fn(d) {
          z.bind(fn(e) { u.bind(fn(f) { pure(ff(a, b, c, d, e, f)) }) })
        })
      })
    })
  })
}
```

## Proposed solution

User define `T::op_bind` then we can use `let*` to introduce **binder**.


```moonbit 
// define binding operator
pub fn[A, B] Parser::op_bind(x : Parser[A], f : (A) -> Parser[B]) -> Parser[B] {
  fn(input) {
    let (v, rest) = x(input)
    f(v)(rest)
  }
}

// using `let*` make the code more clear and simple!
and let_ = fn(input : Input) -> (Raw, Input) raise ParseError {
  let parser = {
     symbol("let") |> ignore 
     let* x = name 
     symbol(":") |> ignore 
     let* a = expr 
     symbol("=") |> ignore 
     let* t = expr 
     symbol(";") |> ignore 
     let* u = expr 
     pure(RLet(x,a,t,u))

  }.label("let")
  parser(input)
}
```