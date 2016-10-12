---
layout: post
title: Free Monads in C++
---

I present a way to use modern C++ metaprogramming techniques to represent
Functors and Monads in a way that is analogous to the way it is done in Haskell.
I show how a generic free monad can be written such that any Functor has a
corresponding free monad.

Practical applications of this technique are left as an exercise for the reader.

## Introduction ##

Haskell's powerful type system and syntactic sugar for composing monadic values
allow for some impressive abstraction. Inspired by a [talk] given by Chris
Barrett at the [Functional Programming Auckland][fpa] meetup on 2016-09-13, in
which he presented the use of a Free Monad in the construction of a
domain-specific language embedded in Haskell, I decided to see if the type
system of C++ could be used to allow a similar abstraction to be built.

[talk]: https://github.com/chrisbarrett/free-monads-talk
[fpa]: https://www.meetup.com/Functional-Programming-Auckland/

The bulk of this article is taken directly from the source files in the
accompanying [library].

[library]: https://github.com/toby-allsopp/free-monads-in-cpp

## Implementation

{% include free-monads-in-cpp/implementation.md %}

## Usage ##

{% include free-monads-in-cpp/usage.md %}

## Conclusion ##

I have shown a method for implementing something analogous to Haskell's Monad
type class in C++ using a class template with a template template parameter. In
addition, I have show that a generic Free Monad for any Functor can be defined
and used in a similar manner to how it is done in Haskell.

The use of this implementation is not very ergonomic due to the very limited
type inference provided by C++ and the lack of any syntactic sugar for composing
monadic values.

## Related Work ##

* <https://bartoszmilewski.com/2011/07/11/monads-in-c/>
* <http://stackoverflow.com/a/2565191>
