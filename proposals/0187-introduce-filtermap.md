# Introduce Sequence.filterMap(_:)

* Proposal: [SE-0187](0187-introduce-filtermap.md)
* Authors: [Max Moiseev](https://github.com/moiseev)
* Review Manager: [John McCall](https://github.com/rjmccall)
* Status: **Active review  (November 15...20, 2017)**
* Swift-evolution discussion: https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20171023/040609.html
* Implementation: [apple/swift#12819](https://github.com/apple/swift/pull/12819)
* Previous Review: [Discussion](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20171106/040999.html) [Decision Notes](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20171113/041342.html)

<!--
* During the review process, add the following fields as needed:*
* Decision Notes: [Rationale](https://lists.swift.org/pipermail/swift-evolution/), [Additional Commentary](https://lists.swift.org/pipermail/swift-evolution/)
* Bugs: [SR-NNNN](https://bugs.swift.org/browse/SR-NNNN), [SR-MMMM](https://bugs.swift.org/browse/SR-MMMM)

* Previous Revision: [1](https://github.com/apple/swift-evolution/blob/...commit-ID.../proposals/NNNN-filename.md)
* Previous Proposal: [SE-XXXX](XXXX-filename.md)

-->


## Introduction

We propose to deprecate the controversial version of a `Sequence.flatMap` method
and provide the same functionality under a different, and potentially more
descriptive, name.

## Motivation

The Swift standard library currently defines 3 distinct overloads for `flatMap`:

~~~swift
Sequence.flatMap<S>(_: (Element) -> S) -> [S.Element]
    where S : Sequence
Optional.flatMap<U>(_: (Wrapped) -> U?) -> U?
Sequence.flatMap<U>(_: (Element) -> U?) -> [U]
~~~

The last one, despite being useful in certain situations, can be (and often is)
misused. Consider the following snippet:

~~~swift
struct Person {
  var age: Int
  var name: String
}

func getAges(people: [Person]) -> [Int] {
  return people.flatMap { $0.age }
}
~~~

What happens inside `getAges` is: thanks to the implicit promotion to
`Optional`, the result of the closure gets wrapped into a `.some`, then
immediately unwrapped by the implementation of `flatMap`, and appended to the
result array. All this unnecessary wrapping and unwrapping can be easily avoided
by just using `map` instead.

~~~swift
func getAges(people: [Person]) -> [Int] {
  return people.map { $0.age }
}
~~~

It gets even worse when we consider future code modifications, like the one
where Swift 4 introduced a `String` conformance to the `Collection` protocol.
The following code used to compile (due to the `flatMap` overload in question).

~~~swift
func getNames(people: [Person]) -> [String] {
  return people.flatMap { $0.name }
}
~~~

But it no longer does, because now there is a better overload that does not
involve implicit promotion. In this particular case, the compiler error would be
obvious, as it would point at the same line where `flatMap` is used. Imagine
however if it was just a `let names = people.flatMap { $0.name }` statement, and
the `names` variable were used elsewhere. The compiler error would be
misleading.

## Proposed solution

We propose to deprecate the controversial overload of `flatMap` and re-introduce
the same functionality under a new name. The name being `filterMap(_:)` as we
believe it best describes the intent of this function.

For reference, here are the alternative names from other languages:
- Haskell, Idris
  ` mapMaybe :: (a -> Maybe b) -> [a] -> [b]`
- Ocaml (Core and Batteries) 
  `filter_map : 'a t -> f:('a -> 'b option) -> 'b t`
- F# 
  `List.choose : ('T -> 'U option) -> 'T list -> 'U list`
- Rust 
  `fn filter_map<B, F>(self, f: F) -> FilterMap<Self, F>   where F: FnMut(Self::Item) -> Option<B>`
- Scala
  ` def collect[B](pf: PartialFunction[A, B]): List[B]`


## Source compatibility

Since the old function will still be available (although deprecated) all
the existing code will compile, producing a deprecation warning and a fix-it.

## Effect on ABI stability

This is an additive API change, and does not affect ABI stability.

## Effect on API resilience

Ideally, the deprecated `flatMap` overload would not exist at the time when ABI
stability is declared, but in the worst case, it will be available in a
deprecated form from a library post-ABI stability.

## Alternatives considered

It was attempted in the past to warn about this kind of misuse and do the right
thing instead by means of a deprecated overload with a non-optional-returning
closure. The attempt failed due to another implicit promotion (this time to
`Any`).

The following alternative names for this function were considered:
- `mapNonNil(_:) `
  Does not communicate what happens to nil’s
- `mapSome(_:) `
  Reads more like «map some elements of the sequence, but not the others»
  rather than «process only the ones that produce an Optional.some»
