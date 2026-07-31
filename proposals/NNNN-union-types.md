# Union types sketch / pre pitch musing

dont read this! i just wanted to get it off my disk. it isn't even done enough to consider a draft.

or, some ways i think we could achieve (limited?) anonymous type uniony things (or maybe its an anonymous sum type, but able to be commutative since we use an existential representation)

## Union? Sum type?

I've realized I was confused about the difference before I started writing this nerd snipe (ty for clarifying Andrew!) so here goes;

A union doesn't carry a tag / discriminator, so you need to narrow the type with checks or only perform operations that are safe on any of the types it could be. A union of `A` and `B` will have the exact same memory as either `A` or `B`. Typescript is like this. This enables `A | B = B | A`, and `Bool | Bool = Bool`; order doesn't matter because there is no bookkeeping, and things which "are" the same collapse away because they have indistinguishable representation. There is no way to tell $Bool_1$ and $Bool_2$ apart without a tag!

A sum does carry a tag, so you can distinguish empty cases, cases with the same contents, and so on. Our enums and Rust's enums are sum types. You lay out your memory like `<tag> <data>`, you may try to stack allocate it, and you may try to reclaim spare tag bits under nesting or uninhabitable cases for efficiency (Rust does niche optimization and we do spare bits). However, order matters, nesting matters, and so on; they have different tags, so different memory representations, converting between `A | B` and `B | A` is a retag even if you didn't name the cases.

## Motivation

Typed throws; specifically, cases where you want to call multiple functions which have typed throws you want to propagate. Also functions that want to call a provided function which throws and propagate that, or throw error related to the HOF. Since you can only throw one type, this isn't possible today; you can nest errors into one type, but the ergonomics of catching and switching are *poor*.

## Desirable properties (in my opinion)

TODO: make correct mathjax

1. $A | Never = A$ / $A| Any = Any$ (identity)
  - top and bottom of the lattice
2. $A | A = A$ (idempotence)
3. $A | B = B | A$ (commutative)
4. $A | (B | C) = (A | B) | C$ (associative)
5. $A | B <: A | B | C$ (width subtyping)
  - you can pass a narrow union to a wider one
6. $A <: B \implies A | C <: B | C$ (monotonicity)
7. $A <: B \iff A | B = B$ (not sure what to call this)
  - a union of two types where one is a subtype of the other collapses to the supertype
  - im not 100% sure this is good, but maybe we can't avoid it? its kind of bad for exhaustive matching

Some of these could really slow down type checking! (and this is not a minimal set of properties, but i liked spelling them out to think about it)

Andrew Wagner pointed out that what I describe aren't really unions (since we always have a tag / discrim) but sum types have an order (not commutative). I *think* Swift programmers would expect commutation, since the place programmers have usually seen a feature like this is Typescript (commutative, real union types). I think what I'm describing is a union, in the sense that we don't have a 

## What if we just use switch and catch sugar over enums with generic parameters?

One thing I think may be worth exploring is if sufficient sugar could be introduced for matching nested enums (instead of adding anonymous sum or union types). We could represent errors that can include an impl defined error with a case that stores a generic error, and just make it easy to match these cases and perform exhaustiveness checking.

```swift
enum ThingErr<T: Error> {
  case A
  case B
  @wrappedError
  case Impl(T)
}

func doThing<In, Out, Err>(_ fn: (_: In) -> Out throws(Err)) -> (_: In) -> Out throws(ThingErr<Err>)

enum Other {
  case Good(amount: Int)
  case Bad
}

func catchExample() {
  try {
    doThing ...
  // .Impl is annotated with @wrappedError, so the error in it can be caught as though it was directly thrown
  } catch Other {
    
  } catch ThingErr {
    // Ideally, we could allow omitting .Impl if you switch over ThingErr in here
  }
}

func switchExample() {
  let val: ThingErr<Other> = ...

  switch (val) {
  case .A: {}
  case .B: {}
  case .Impl(.Good(let amount)): {}
  // err: forgot .Impl(.Bad) !
  }

  // or maybe

  switch(val) {
  case .A: {}
  case .B: {}
  case .Impl: {
    .Good(let amount): {
    .Bad: {}
  }
  }
}
```

This is not a type union, but it might be enough for typed throws. It has some serious advantages over an `A | B` like syntax! It makes no allusions to properties like commutation (`A<B>` is clearly not `B<A>`), does not introduce join types to the type system, and adds switch sugar that is useful in other cases (we could do without the switch sugar fwiw). The function passed to the HOF does not need any awareness of this system and can throw any error type it chooses (including a non enum error). The memory layout and performance implications of nesting enums like this are clear (no expectation that the discriminators can collapse), and do not ask any heroics of our compiler and our type system. This system composes (although not associatively), since `A<B<C>>` is a natural error.

This presents an idempotence problem; `ThingErr<ThingErr<Never>>`. It is possible we can forbid generics being fixed to the same enum, when the generic is being stored in an annotated case? To spell it out, what does this do?

```swift
try { /* throw ThingErr<ThingErr<Never>> */ }
catch ThingErr { ... }
catch ThingErr { ... }
```

Possibly this is ok, since the only way to have 'two' `ThingErr` is if the outer one is `.Impl`, so maybe we reject this code and have you write a single `catch ThingErr`? It is strange that `.Impl(.A)` is the same as `.A`... I think allowing the nesting or forbidding it would both be ok, I lean towards allowing the nesting and 'unwrapping' the inner case.

We would probably want to (at least consider if we need to) reject any definitions where `@wrappedError` is applied to multiple cases which store a value of the same generic;

```swift
enum Glass<T: Error> {
  @wrappedError
  case Half(T)
  @wrappedError
  case Full(T)
}
```

Or maybe we are ok with `Half` and `Full` being indistinguishable to catch... I'd argue that definition is malformed, since an enum is only one case...

```swift
enum Glass<U: Error, V: Error> {
  @wrappedError
  case Half(U)
  @wrappedError
  case Full(V)
}
```

Clients can't distinguish `Glass<A, B>` from `Glass<B, A>` while catching (though this is *not* commutativity since one `Glass` cannot be substituted for the other) but it is useful to model an 'either' type, I think all of these *could* be fine to allow, maybe we warn when multiple cases contain the same generic (though it could still be useful to switch them differently in some case I can't think of). It is *strange*, but not intolerable, that this leads to a situation where catching feels like a union (no tag, can't distinguish the same type in `Half` vs `Full`) but matching feels like a sum (tagged, and `Half` vs `Full` can be treated differently).

Implementation side, we would need a request to compute uninhabitability, and to consider that `V = Never` means only `Glass.Half` can be constructed, `Glass<Never, Never>` is as uninhabitable as `Never`, and to reduce or eliminate the need to exhaustively catch / match based on this.

What about bubbling errors from invocation of multiple throwing functions? I think we would want an either type + allowing nested wrapped error catching, i think the codegen can be done efficiently? It might require thinking deeply about how we lay out the order of the emitted code / requiring catches to have a certain order to reduce work, which is not ideal (the idea being, catch in the order with the fewest checks possible).

```swift
enum Either<A: Error, B: Error> {
  @wrappedError
  case Left(A)
  @wrappedError
  case Right(B)
}

typealias One = Either<A, Either<B, C>>
typealias Two = Either<Either<A, B>, Either<C, D>>
// what order should A, B, C, D be checked? to minimize branches? codesize?
// would be nice if we could use only one tag, and not two. we do spare bits optimization for Optional...

typealias Three = Either<Either<Either<A, B>, Never>, Never>
// It would be nice to be able to collapse all the uninhabitable arms within the tagging scheme...
```

We could entertain the discussion of inference rules for its errors:

```swift
func throwA() throws(A) {}
func throwB() throws(B) {}
func throwC() throws(C) {}

func foo() {
  let fn = { in // needs to infer OneOf<A, B, C> / Either<A, Either<B, C>> ?
    if (true) {
      throwA()
    } else {
      throwB()
    }

    throwC()
  }
}
```

This starts to look like a really ugly join type if we do inference (we don't need to! but closures exist).

(I think) `Either` is an "anonymous" sum type, and we could even treat `A | B` / `A + B` / `A / B` as sugar for `Either<A, B>`... we could even add it later? Depending on why `Optional` is a builtin, `Either` will want to be a builtin for the same reason.

I'll make a bunch of arguments that true union types are regrettable below though; so if we don't add any inference rules, I think `@wrappedError` and smarter computation of uninhabitability could be enough for *our* uses of typed throws, and we could consider switch sugar as well.

## Union memory layout

I think we can represent union types as either an existential (but with more type information!) OR as a compiler generated enum. Compiler can find the discriminant for each case by walking the disconnected graph of types that share a type union, assigning incrementing indexes, and considering that an enum (with cases narrowed by type information). This enables `A | B` to be substituted for `A | B | C` since `A` and `B` would have the same discriminant in either union if it is possible for one to be used as the other in the call graph. We could only perform this optimization within WMO, static linking, embedded, and only in cases where the union isn't exposed outside in a way that needs to evolve. In addition to a consistent discriminator scheme, we need a shared amount of padding, to be able to pass `A | B` as `A | B | C`. When `A <: B`, I think you want the same discriminant for `A` and `B` so that `A | C` can be substituted for `B | C` without retag, which should be fine if we have property 7 (you won't encounter a union where a type and its subtype are present).

We could consider doing this graph walk optimization later (somehow?) when reinterpreting SIL, to try to get it to work across modules? We could also consider accepting a retag and pad between module boundaries.

I was considering how we could use the table pointer of each type as a discriminant, but that is an existential type + dynamic casting. We support `any Error` in embedded today, so this isn't untenable. When we aren't able to optimize it, we still preserve exhaustive switching, catching, and the type information required to optimize it further when needed. We can even emit slightly better codegen, since if you are matching `A | B | C` and have failed to cast to `A` and `B`, you can cast like `as! C` (without even the potential to throw?).

## Union ergonomics

How much of a problem this is for the type system, I'm not sure! I'll include some cases that seem problematic to me below. We could consider narrowing, but it should be fine to add more narrowing rules later, since the narrowed union is a subtype of the unnarrowed union. Conversion from `A` to `A | B` would require wrapping though...

We can treat cross module unions like enums, and require `@unknown default` unless `@frozen (A | B)` (that syntax is confusing...)

In ambiguous cases, we would need to diagnose the ambiguity and require specificity, which is concerning (since dot shorthand is already problematic, and detecting ambiguity can get difficult);

```swift
enum A {
  case left, right
}
enum B {
  case wrong, right
}

switch (val) { // A | B
case .left: {} // fine
case .right: {} // ambiguous, need to write:
case A.right: {} // ok
case B.right: {} // ok
}
```

It poses concerns for type inference if we infer these:

```swift
var n = 5 // n: Int | String
if (cond) { n = "hi" }
```

there are places the join could silently fire where today you get a good error:

```swift
func pair<T>(_ x: T, _ y: T) -> (T, T)
pair(1, "x") // T = Int | String

let z = cond ? 1 : "x" // Int | String

let xs = [1, "two", 3.0] // [Int | String | Double]
```

we are put in a weird spot with overload resolution...

```swift
func f(_ x: Int) {}
func f(_ x: String) {}
let val: Int | String = 3
f(val) // do we do dynamic dispatch? reject?
```

## Limiting scope

### Restricting to the typed throws position

What if we only allow unions as typed throws? Can't store or pass, you can only catch them. I'm not convinced that limits scope sufficiently, since you still have subtyping relationships; you need to do some nontrivial work to see that this code can be allowed...


```swift
class Parent {}
class Child: Parent {}
class Other {}

func f() throws(Parent | Other) {}
func g() throws(Child | Other) {}
func h(_ fn: () throws(Parent | Other) -> ()) {}

h(g) // is this ok?
```

TODO: check if this works today, but with Either style type in another type position?

I don't think the lattice is concerning on its own, but in the presence of multiple types with ExpressibleBy protocols, you are still in trouble. Let `A` and `B` be ExpressibleByStringLiteral;

```swift
func foo() throws(A | B) {
  throw "boom"
}

try {
  foo()
} catch A {
  // do i run?
} catch B {
  // or do i?
}
```

We could ban ExpressibleBy into a type within the throws union, but we will still need to be able to diagnose this as ambiguous:

```swift
enum A {
  case left, right
}
enum B {
  case wrong, right
}

func foo() throws(A | B) {
  throw .right  // nope!
  throw A.right // ok
}
```
