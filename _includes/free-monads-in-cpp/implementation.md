> Functor.h


~~~c++
#pragma once

#include <utility>
~~~
{: .cpp2blog-source}


### Functors ###

A "Functor" is something very precise in category theory, but you can think of
it as something that somehow contains values with some structure and allows
you to transform the contained values in a way that preserves the structure.

In Haskell, it is defined like so:

```haskell
class Functor f where
  fmap :: (a -> b) -> f a -> f b
```

This just says that a type _f_ is a Functor if it supports the `fmap`
function, which takes:

- a function that operates on each value of type _a_ contained in the Functor,
  returning a value of type _b_ and
- a value of type _f a_, being a particular value of our Functor containing
  values of type _a_

and produces a value of type _f b_, being a particular value o four Functor
containing values of type _b_.

In C++, we will define a Functor as any class template with a single type
parameter for which a valid specialization of the `Functor` class template
exists.



~~~c++
namespace Functor {
~~~
{: .cpp2blog-source}


The default specialization of `Functor` is undefined; this will cause a
compile-time error (or substitution failure when used in certain template
contexts) if it is used.

The first template parameter is the class template to be considered a
Functor. The second template parameter is there to enable partial
specializations to use SFINAE-based techniques.


~~~c++
  template <template <typename> class T, typename = void>
  struct Functor;
~~~
{: .cpp2blog-source}


It is very useful to have a concise way to query, at compile-time, whether a
given class template has a valid `Functor` specialization. This is a perfect
application of "Concepts" but as we wish to remain compatible with C++14
compilers we have to do it the old-fashioned way, i.e. using partial
template specialization and SFINAE.


~~~c++
  namespace detail {
~~~
{: .cpp2blog-source}


Note that MS Visual C++ doesn't properly support expression SFINAE as of
VS2015 update 3, so we just assume everything is a Functor there.


~~~c++
#ifdef _MSC_VER
    template <template <typename> class>
    using IsFunctorT = std::true_type;
#else
~~~
{: .cpp2blog-source}


In the usual C++ template metaprogramming fashion, we define a class
template with the default case and then use partial specialization to
change it for specific cases. Another common idiom is the use of an extra,
defaulted template parameter that partial specializations can use to
SFINAE themselves away.

The default case is that any given class template is not a Functor.


~~~c++
    template <template <typename> class, typename = void>
    struct IsFunctorT : std::false_type {};
~~~
{: .cpp2blog-source}


Then we essentially say that any class template `T` is a Functor if a call to
`Functor<T>::fmap` compiles and has the correct result type.


~~~c++
    struct dummy1 {};
    struct dummy2 {};

    template <template <typename> class T>
    struct IsFunctorT<T,
                      std::enable_if_t<std::is_same<
                          T<dummy2>,
                          decltype(Functor<T>::fmap(std::declval<dummy2(dummy1)>(),
                                                    std::declval<T<dummy1>>()))>::value>>
        : std::true_type {};
#endif
  }
~~~
{: .cpp2blog-source}


Using the `IsFunctorT` class template is little bit inconvenient, so we hide
it away in the `detail` namespace and instead expose a `constexpr` variable
template. An example of the use of this is coming up in the definition of `fmap`,
below.


~~~c++
  template <template <typename> class T>
  constexpr bool IsFunctor = detail::IsFunctorT<T>::value;
~~~
{: .cpp2blog-source}


To make using the `fmap` function on a Functor value more convenient, we
define a free function that wraps it. Using this allows the Functor template
argument to be deduced.


~~~c++
  template <template <typename> class F,
            typename A,
            typename Fun,
            typename = std::enable_if_t<IsFunctor<F>>>
  F<std::result_of_t<Fun(A)>> fmap(Fun&& fun, const F<A>& f) {
    return Functor<F>::fmap(std::forward<Fun>(fun), f);
  }
~~~
{: .cpp2blog-source}


It is useful for testing purposes to have a very basic Functor---we call
this `NullFunctor`. It contains exactly zero values, each of the requisite
type.


~~~c++
  namespace Test {
    template <typename A>
    struct NullFunctor {};
  }
  template <>
  struct Functor<Test::NullFunctor> {
    template <typename F, typename A>
    static Test::NullFunctor<std::result_of_t<F(A)>> fmap(F, Test::NullFunctor<A>) {
      return {};
    }
  };
  static_assert(IsFunctor<Test::NullFunctor>, "NullFunctor must be a Functor");
}
~~~
{: .cpp2blog-source}

> Monad.h


~~~c++
#pragma once

#include "Functor.h"
#include "void_t.h"
~~~
{: .cpp2blog-source}


### Monads ###

The concept of a Monad comes from category theory, but the defintion I'm using
is that of its practical application in Haskell.

In Haskell, `Monad` is a typeclass, defined something like so:

```haskell
class (Functor m) => Monad m where
  pure :: a -> m a
  bind :: m a -> (a -> m b) -> m b
```

There are some additional laws that Monads are supposed to obey but for now I'm
just going to focus on the type signatures.

In C++, we can treat the type variable `m` as a class template with a class
template template parameter and define a class template `Monad` that we will
specialize for each class template `M` that we want to treat as a Monad.



~~~c++
namespace Monad {
  template <template <typename> class M, typename Enable = void>
  struct Monad;
~~~
{: .cpp2blog-source}


C++ doesn't provide a neat way to define the members that we expect
specializations of the `Monad` template to have, but we can write a type-level
predicate that uses SFINAE to check.


~~~c++
  namespace detail {
    template <template <typename> class M, typename Enable = void>
    struct IsMonadT : std::false_type {};
~~~
{: .cpp2blog-source}


At the moment we just have a very basic check---is there a specialization
of Monad for the template and is the template a Functor.


~~~c++
    template <template <typename> class M>
    struct IsMonadT<M, void_t<Monad<M>, std::enable_if_t<Functor::IsFunctor<M>>>>
        : std::true_type {};
  }

  template <template <typename> class M>
  constexpr bool IsMonad = detail::IsMonadT<M>::value;
~~~
{: .cpp2blog-source}


Now that we can tell (approximately) whether a class template is a Monad, we
can define the Monad operations as free functions so that the template
arguments can be deduced in at least some cases.


~~~c++
  // pure :: (Monad m) => a -> m a
  template <template <typename> class M,
            typename A,
            typename = std::enable_if_t<IsMonad<M>>>
  M<A> pure(const A& x) {
    return Monad<M>::pure(x);
  }

  // bind :: (Monad m) => m a -> (a -> m b) -> m b
  template <typename F,
            template <typename> class M,
            typename A,
            typename = std::enable_if_t<IsMonad<M>>>
  std::result_of_t<F(A)> bind(const M<A>& m, F&& f) {
    return Monad<M>::bind(m, std::forward<F>(f));
  }
}
~~~
{: .cpp2blog-source}


It is common in Haskell to use the infix (>>=) operator as a synonym for
_bind_; this reduces the need for parentheses. We can do the same thing in
C++, allowing us to write `m >>= [](){}` instead of `bind(m, [](){})`.


~~~c++
template <template <typename> class M,
          typename A,
          typename F,
          typename = std::enable_if_t<Monad::IsMonad<M>>>
auto operator>>=(const M<A>& m, F&& f) {
  return Monad::bind(m, std::forward<F>(f));
}
~~~
{: .cpp2blog-source}


The (>>) operator is also commonly used in Haskell. It just throws away the
result of evaluating the first argument and returns the second argument
instead.

```haskell
(>>) :: (Monad m) => m a -> m () -> m ()
x >> y = x >>= \_ -> y
```


~~~c++
template <template <typename> class M,
          typename A,
          typename B,
          typename = std::enable_if_t<Monad::IsMonad<M>>>
M<B> operator>>(const M<A>& m, const M<B>& v) {
  return Monad::bind(m, [=](auto&&) { return v; });
}
~~~
{: .cpp2blog-source}

> List.h


~~~c++
#pragma once

#include "Functor.h"
#include "Monad.h"

#include <algorithm>
#include <iterator>
#include <ostream>
#include <vector>
~~~
{: .cpp2blog-source}


## The List Monad ##

We introduce a special kind of vector that we call List. We could use vector
directly but this allows us to avoid making every vector a Monad.


~~~c++
template <typename A>
struct List : std::vector<A> {
  using std::vector<A>::vector;
};
~~~
{: .cpp2blog-source}


It is useful to be able to print out Lists for testing and debugging.


~~~c++
template <typename A>
std::ostream& operator<<(std::ostream& os, const List<A>& l) {
  os << "[";
  bool first = true;
  for (const auto& x : l) {
    if (!std::exchange(first, false)) os << ", ";
    os << x;
  }
  os << "]";
  return os;
}
~~~
{: .cpp2blog-source}


### Making List a Functor ###

First, we need to make `List` a Functor by partially specializing `Functor` and
defining `fmap`.


~~~c++
namespace Functor {
  // instance Functor List where
  template <>
  struct Functor<List> {
    // fmap :: (a -> b) -> List a -> List b
    template <typename Fun, typename A>
    static auto fmap(Fun&& f, const List<A>& l) {
      List<std::result_of_t<Fun(A)>> result;
      std::transform(
          l.begin(), l.end(), std::back_inserter(result), std::forward<Fun>(f));
      return result;
    }
  };
  static_assert(IsFunctor<List>, "List should be a Functor");
}
~~~
{: .cpp2blog-source}


### Making List a Monad ###

Then we can make `List` a Monad by partially specializing `Monad` and defining
`pure` and `bind`.


~~~c++
namespace Monad {
  // instance Monad List where
  template <>
  struct Monad<List> {
    // pure :: a -> List a
    template <typename A>
    static List<A> pure(const A& x) {
      return List<A>({x});
    }

    // bind :: List a -> (a -> List b) -> List b
    template <typename F, typename A, typename ListB = std::result_of_t<F(A)>>
    static ListB bind(const List<A>& l, F&& f) {
      ListB result;
      for (const A& x : l) {
        auto subresult = std::forward<F>(f)(x);
        std::copy(subresult.begin(), subresult.end(), std::back_inserter(result));
      }
      return result;
    }
  };
  static_assert(IsMonad<List>, "List should be a Monad");
}
~~~
{: .cpp2blog-source}

> Free.h


~~~c++
#pragma once

#include "Functor.h"
#include "Monad.h"
~~~
{: .cpp2blog-source}


### Free Monads ###

Every type _f_ that is a Functor has a "Free" Monad. A Free Monad is some
category theory gobbledygook but it's basically the simplest possible Monad
that doesn't throw any information away.

In Haskell, this is defined quite simply:

```
data Free f a = Return a | Bind (f (Free f a))

instance (Functor f) => Monad (Free f) where
...
```

This is a bit complicated to express in C++ due to the lack of algebraic data
types, in particular the lack of a sum type (`tuple` and `struct` provide
product types). In C++17 we will get `variant` which plugs the gap acceptably
and in the meantime we can use `boost::variant`.


~~~c++
#include <boost/variant.hpp>
~~~
{: .cpp2blog-source}


To define something like the Haskell `data` with two cases, we define a type
for each case and then combine them into a `variant`. Because the `Bind`
case is recursive we need to forward-declare them and use
`recursive_wrapper` to define the `variant`.


~~~c++
namespace Free {
  template <template <typename> class F, typename A>
  struct Return;
  template <template <typename> class F, typename A>
  struct Bind;
~~~
{: .cpp2blog-source}


We wrap the `variant` in a `struct` so we have somewhere to hang a couple of
type aliases that we will need later to be able to write down the type
signatures for some functions.


~~~c++
  template <template <typename> class F, typename A>
  struct Free {
    using ContainedType = A;
    using ReturnType    = Return<F, A>;
    boost::variant<boost::recursive_wrapper<Return<F, A>>,
                   boost::recursive_wrapper<Bind<F, A>>>
        v;
  };
~~~
{: .cpp2blog-source}


Now we can define the individual cases, quite analogously to the Haskell
definition. The only difference (apart from the syntax) is that we need to
name the members something.


~~~c++
  template <template <typename> class F, typename A>
  struct Return {
    A a;
  };
  template <template <typename> class F, typename A>
  struct Bind {
    F<Free<F, A>> x;
  };
~~~
{: .cpp2blog-source}


For our convenience, we define some helper function templates that handle
wrapping things up in the `Free` struct and allow some of the template
arguments to be deduced.


~~~c++
  // This one is for when you know the Functor template argument - it can deduce
  // the contained type.
  template <template <typename> class F, typename A>
  Free<F, A> make_return(const A& x) {
    return {Return<F, A>{x}};
  }

  // This one is for when you know the resulting Free type but not the template
  // arguments.
  template <typename FA>
  FA make_return(const typename FA::ContainedType& x) {
    return {typename FA::ReturnType{x}};
  }

  template <template <typename> class F, typename A>
  Free<F, A> make_bind(const F<Free<F, A>>& x) {
    return {Bind<F, A>{x}};
  }
~~~
{: .cpp2blog-source}


It can be helpful while testing to be able to print out a value of `Free`,
so we define an overload of the left shift^W^Wstream out operator. This also
demonstrates how to get values out of the `variant` in a type-safe way---by
using a visitor.


~~~c++
  template <template <typename> class F, typename A>
  std::ostream& operator<<(std::ostream& os, const Free<F, A>& free) {
    struct Visitor {
      std::ostream& os;
      std::ostream& operator()(const Return<F, A>& r) {
        return os << "Return{" << r.a << "}";
      }
      std::ostream& operator()(const Bind<F, A>& b) {
        return os << "Bind{" << b.x << "}";
      }
    };
    Visitor v{os};
    return boost::apply_visitor(v, free.v);
  }
~~~
{: .cpp2blog-source}


Now we implement the Functor functions. We don't actually create the
specialization of the Functor template here because that has to be done in
the Functor namespace but it is more convenient to write the functions in
the Free namespace.

The Functor instance for Free is hilariously short and to-the-point in
Haskell:

```haskell
instance Functor f => Functor (Free f) where
  fmap fun (Return x) = Return (fun x)
  fmap fun (Bind x)   = Bind (fmap (fmap fun) x)
```

We assume that any user that wants to use the Functor instance of Free for a
specific Functor F will create a wrapper type like so:

```
template<typename A>
struct Wrapper : Free<F, A> {};
```

This is needed so that the template parameters match up correctly - all the
Functor-related functions expect to deal with a template with a single
parameter, whereas `Free` has two.


~~~c++
  template <template <typename> class Wrapper>
  struct FunctorImpl {

    // The visitor struct can't be defined inside the fmap function because it
    // contains member function templates, which we use to get the compiler to
    // tell us what the template F is in the Free<F, A> inside the wrapper.
    template <typename A, typename Fn>
    struct Visitor {
      Fn& fun;
      template <template <typename> class F>
      auto operator()(const Return<F, A>& r) const {
        return make_return<F>(fun(r.a));
      }
      template <template <typename> class F>
      auto operator()(const Bind<F, A>& b) const {
        using Functor::fmap;
        return make_bind(fmap([&](const auto& f) { return fmap(fun, f); }, b.x));
      }
    };

    // fmap :: (a -> b) -> Free f a -> Free f b
    template <typename A, typename Fn>
    static Wrapper<std::result_of_t<Fn(A)>> fmap(Fn&& fun, const Wrapper<A>& f) {
      return boost::apply_visitor(Visitor<A, Fn>{fun}, f.v);
    }
  };
~~~
{: .cpp2blog-source}


Now that we have a way to make `Wrapper<A>`, aka `Free<F, A>`, a Functor, we
can also make it a Monad. Once again, the instance in Haskell is
embarrassingly short:

```haskell
instance (Functor f) => Monad (Free f) where
  return = Return
  (Bind x)   >>= f = Bind (fmap (>>= f) x)
  (Return r) >>= f = f r
```

Again, we're not actually specializing the `Monad` class template here
because that has to be done in the `Monad` namespace and it's more
convenient to write this stuff in the `Free` namespace.

Also again, we require the user to wrap their Functor to be Freed into
something with a single template parameter, as `Free<F, A>` doesn't work
directly. We make an additional assumption about the wrapper class
template---it must have a type alias member named `WrappedFree` that is an
alias for the `Free<F, A>` type that is being wrapped, e.g.

```
template<typename A>
struct Wrapper : Free<F, A> {
  using WrappedType = Free<F, A>;
};
```


~~~c++
  template <template <typename> class Wrapper>
  struct MonadImpl {
    template <typename A>
    using M = Wrapper<A>;

    // pure :: a -> m a
    // pure :: a -> Free<F>
    template <typename A>
    static M<A> pure(const A& x) {
      return make_return<typename Wrapper<A>::WrappedFree>(x);
    }

    // bind :: Free f a -> (a -> Free f b) -> Free f b
    template <typename A, typename Fn>
    struct BindVisitor {
      using result_type = std::result_of_t<Fn(A)>;
      using B           = typename result_type::ContainedType;
      static_assert(std::is_same<result_type, Wrapper<B>>::value, "");
      Fn&& fun;

      // bind (Bind x) f = Bind (fmap (\m -> bind m f) x)
      template <template <typename> class F>
      result_type operator()(const Bind<F, A>& b) {
        auto& f = this->fun;
        return make_bind(Functor::fmap(
            [f](const Free<F, A>& m) {
              return static_cast<Free<F, B>>(Monad::bind(Wrapper<A>(m), f));
            },
            b.x));
      }

      // bind (Return r) f = f r
      template <template <typename> class F>
      result_type operator()(const Return<F, A>& r) {
        return fun(r.a);
      }
    };

    template <typename A, typename Fn>
    static auto bind(const M<A>& x, Fn&& fun) {
      BindVisitor<A, Fn> v{std::forward<Fn>(fun)};
      return boost::apply_visitor(v, x.v);
    }
  };
~~~
{: .cpp2blog-source}


It is traditional to provide some helpers to make working with free monads
more convenient. The first of these is "liftFree", which is used to "lift" a
functor F into a free monad.

The implementation looks like so in Haskell:

```haskell
liftFree :: (Functor f) => f a -> Free f a
liftFree x = Bind (fmap Return x)
```

In C++ this looks very similar, yay! A few extra type annotations are
required to help the compiler but it's not too bad.


~~~c++
  template <template <typename> class F, typename A>
  Free<F, A> liftFree(const F<A>& x) {
    return make_bind<F>(Functor::fmap(&make_return<F, A>, x));
  }
~~~
{: .cpp2blog-source}


The other helper is really the whole point of the exercise; it allows you to
take a value of `Free<F, A>` and "evaluate" it in some way to yield another
monadic value.

```haskell
foldFree :: Monad m => (forall x . f x -> m x) -> Free f a -> m a
foldFree _ (Return a)  = return a
foldFree f (Bind as)   = f as >>= foldFree f
```

Because we want to be totally generic with regards to the callable object
that is passed in, it's not possible to deduce the resulting Monad template,
so the caller has to specify it. The other template parameters can be
deduced, however.


~~~c++
  template <template <typename> class M,
            template <typename> class F,
            typename Fun,
            typename A>
  M<A> foldFree(Fun fun, const Free<F, A>& free) {
    struct Visitor {
      Fun fun;

      auto operator()(const Return<F, A>& r) { return Monad::pure<M>(r.a); }
      auto operator()(const Bind<F, A>& b) {
        return fun(b.x) >>= [&](const auto& x) { return foldFree<M>(fun, x); };
      }
    };
    Visitor v{fun};
    return boost::apply_visitor(v, free.v);
  }
}
~~~
{: .cpp2blog-source}


All that remains to be done is to actually make `Free<F>` a Functor and a
Monad. Recall that `Free` actually has two template parameters but the
`Functor` and `Monad` class templates expect a template with one parameter as
their template argument. So, we require a wrapper class template that has only
one template parameter.

Given a class template `Wrapper` whose instantiations `Wrapper<A>` have
`Free<F, A>` as a base and a type alias template member `WrappedFree` equal to
`Free<F, A>`, we can create a partial speciailization of both `Functor` and
`Monad`.

The below speciailizations are not quite as constrained as I would like
because they don't actually mention `Free` anywhere, so they apply to any
class template that inherits from the class named by its `WrappedFree` member.
The reason for this is that I haven't been able to figure out how to get the
`F` out in order to write `Free<F, void>`.


~~~c++
namespace Functor {
  template <template <typename> class Wrapper>
  struct Functor<Wrapper,
                 std::enable_if_t<std::is_base_of<typename Wrapper<void>::WrappedFree,
                                                  Wrapper<void>>::value>>
      : Free::FunctorImpl<Wrapper> {};
}

namespace Monad {
  template <template <typename> class Wrapper>
  struct Monad<Wrapper,
               std::enable_if_t<std::is_base_of<typename Wrapper<void>::WrappedFree,
                                                Wrapper<void>>::value>>
      : Free::MonadImpl<Wrapper> {};
}
~~~
{: .cpp2blog-source}


To finish things off, we do some compile-time verification that these partial
specializations work as we expect by wrapping `NullFunctor`.


~~~c++
namespace Free {
  namespace Test {
    template <typename A>
    struct NullFreeWrapper : Free<Functor::Test::NullFunctor, A> {
      using WrappedFree = Free<Functor::Test::NullFunctor, A>;
    };
    static_assert(Functor::IsFunctor<NullFreeWrapper>,
                  "Functor speciailization for Free wrappers works");
    static_assert(Monad::IsMonad<NullFreeWrapper>,
                  "Monad speciailization for Free wrappers works");
  }
}
~~~
{: .cpp2blog-source}

