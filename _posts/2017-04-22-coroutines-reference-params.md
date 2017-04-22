---
layout: post
title: Coroutines and Reference Parameters
---

Traditionally, C++ functions that take parameters by reference have been able to
rely on the objects bound to such references outliving the execution of the
function. However, things are not so simple when your function is a coroutine;
such functions can be suspended and resumed, returning to their caller before
they have finished executing. In this article I show how this violation of
long-standing expectations can lead to subtle bugs in coroutine code and
describe some approaches for avoiding them.

## Background

### Coroutines

[C++ Extensions for Coroutines] is a Proposed Draft Technical Specification (TS)
that describes changes to the C++ language to allow functions to be suspended
and resumed. The TS provides a very low-level interface to the coroutine
machinery; it is intended to be used by higher-level abstractions.

[C++ Extensions for Coroutines]: http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/n4649.pdf

The implementation of coroutines described in the current draft of the TS is
"stackless", which means that the entire stack is not saved when the coroutine
is suspended; instead, only local variables are saved. This is different to
"stackful" coroutines in which the entire call stack is saved and restored. The
advantage of the "stackless" approach in the TS is that coroutines can be very
light-weight, only needing storage for any local variables.

If you like watching videos, a really great introduction to coroutines was given
by James McNellis at CppCon
2016: [Introduction to C++ Coroutines][jmcncppcon2016].

[jmcncppcon2016]: https://www.youtube.com/watch?v=ZTqHjjm86Bw

### Forwarding references

In order to understand the solutions presented later in this article, it will be
helpful to understand how forwarding references work. When writing a function
template, if we write what looks like an rvalue reference for a parameter then
we end up with a forwarding reference, which means that it will be the same kind
of reference as the caller provided. Let's take a very simple example.

```c++
template <typename T>
void f(T&& x);

f(3); // T is int; x is int&&
int i = 3;
f(i); // T is int&; x is int&
const int j = 3;
f(j); // T is int const&; x is int const&
```

This falls out from the rules of [template argument deduction] and [reference
collapsing]. The important thing to remember is when an `int&&` is passed, the
template parameter is just `int`, but when an `int&` is passed, the template
parameter is `int&`.

[template argument deduction]: http://en.cppreference.com/w/cpp/language/template_argument_deduction
[reference collapsing]: http://en.cppreference.com/w/cpp/language/reference#Reference_collapsing

## The problem

So, let's imagine we're writing a really simple coroutine that just yields
values from a `vector` (or any other Range). This will use a result type called
`generator`, which is an example used in the Coroutines TS and is also available
in Visual Studio (in `<experimental/generator>`).

```c++
template<typename T>
using range_value_t = decltype(*std::declval<T>().begin());

template<typename T, typename F>
generator<std::result_of_t<F(range_value_t<T>)>>
map(const T& v, F f) {
  for (auto&& element : v) {
    co_yield f(element);
  }
}
```

This takes a reference to a range and a callable object and returns a
`generator` that yields the results of calling the callable with the elements of
the range. The `generator` type provides `begin()` and `end()` members that
return iterators. To use it, you'd write something like this:

```c++
std::vector<int> v{1, 2, 3, 4};
auto f = [](int i) { return i * 2; };
for (auto&& element : map(v, f)) {
  process(element);
}
```

Then you notice that you don't need that `v` variable lying around:

```c++
auto f = [](int i) { return i * 2; };
for (auto&& element : map(std::vector<int>{1, 2, 3, 4}, f)) {
  process(element);
}
```

This compiles but exhibits Undefined Behaviour (UB) when run. Probably the
program will crash, but technically anything could happen. The `vector` that we
pass into the `map` function is a temporary, a prvalue. This binds to the
`const` reference parameter no problem, but the object only lives until the end
of the containing full-expression. If we de-sugar the range-based `for` loop, we
get this:

```c++
{
  auto&& __range = map(std::vector<int>{1, 2, 3, 4}, f);
  for (auto __begin = __range.begin(), __end = __range.end();
      __begin != __end;
      ++__begin) {
    auto&& element = *__begin;
    process(elements);
  }
}
```

We can now see that the full-expression containing the construction of the
temporary `vector` ends before the loop even starts. This leaves the coroutine
with a reference to an object whose lifetime has ended, leading to UB when it is
accessed.

> In a coroutine, you can't rely on references passed as arguments remaining
> valid for the life of the coroutine. You need to think of them like captures
> in a lambda.

As an aside, it's interesting to consider that this problem doesn't arise if we
use a `for_each` function that accepts a range instead of the `for` loop.

```c++
template<typename R, typename F>
auto for_each(R&& r, F f) {
  return std::for_each(r.begin(), r.end(), f);
}

for_each(
  map(std::vector<int>{1, 2, 3, 4}, f),
  [&](auto&& element) {
    process(element);
  });

```

However, we want to make our `map` function useable in any way that would seem
reasonable, not only within one full-expression.

So, what we really want is to take a copy of the range if it's a temporary.
In fact, because it's an rvalue, we will actually move it rather than copying
it.

### Solution Zero: Just Copy Already

It's easy enough to make `map` always copy/move its argument---simply take it by
value instead of by reference. However, if we wish to use a non-temporary range,
it would be nice to not have to make a copy of it. After all, one of the reasons
to use a coroutine is so that we can lazily perform the transformation only on
those elements that we access.

If the things that you're iterating over are cheaply copied, however, this is by
far the easiest option to understand and should really be your first choice.

> Passing and returning objects by value makes your code much easier to reason
> about; do it whenever you can.

### Solution One: Clever Forwarding

With thanks to Vaughn Cato on the cpplang slack, I now present a neat technique
that causes the argument to be passed by reference for lvalues and by value for
rvalues.

First, we take our existing `map` function and rename it to `map_impl` (or
`detail::map` or whatever), changing the parameter to be by-value.

```c++
template<typename T, typename F>
generator<std::result_of_t<F(range_value_t<T>)>>
map_impl(T v, F f) {
  for (auto&& element : v) {
    co_yield f(element);
  }
}
```

Then we define a wrapper around this, called `map` that forwards the parameter on.

```c++
template<typename T, typename F>
auto map(T&& v, F f) {
  return map_impl<T>(std::forward<T>(v), std::forward<F>(f));
}
```

This is slightly different to the usual `forward` idiom in that we are
explicitly passing `T` as the template argument for `map_impl` rather than
letting it be deduced. The result is that when `map` is passed an `int&&` it
calls `map_impl<int>` and when it is passed an lvalue it calls `map_impl<int&>`.

We can still get into trouble with the new definition, because it's pretty easy
to create a coroutine that outlives the object it refers to. The following
function will return a generator that has a reference to a local variable of the
function and any caller that uses the generator will invoke UB.

```c++
auto mapped_vector() {
  auto f = [](int i) { return i * 2; };
  vector v{1, 2, 3, 4};
  return map(v, f);
}
```

However, this is no different to code that returns an iterator or a reference to
an element of a `vector`, so you may be OK with that.

### Solution Two: Reference Wrapper

If we decide that we don't like that the previous solution quietly stores a
reference to what we pass to it in some cases but not others, there are a couple
of alternatives to consider.

What we want is for the caller to be explicit about whether we should copy the
argument or whether we can store a reference. One way to be explicit about this
is for the caller to create a `reference_wrapper` using `std::ref`.

```c++
vector v{1, 2, 3, 4};
return map(v, f); // copies v
return map(std::ref(v), f); // stores a reference to v
```

For this to work, we need `map` to know about `reference_wrapper`:

```c++
template<typename T>
using unwrap_reference_t = decltype(std::ref(std::declval<T>()).get());

template<typename T, typename F>
generator<std::result_of_t<F(range_value_t<unwrap_reference_t<T>>)>>
map(T v, F f) {
  for (auto&& element : std::ref(v).get()) {
    co_yield f(element);
  }
}
```

This exploits the fact that `std::ref` when called with a `reference_wrapper`
just returns that wrapper without wrapping it again.

### Solution Three: Different Functions

Another way of making it explicit whether we're storing a copy or a reference is
to just have two different functions with different names. I know that's a weird
concept but bear with me.

We will reuse `map_impl` from Solution One and create two wrappers that forward
to it in subtly different ways.

```c++
template<typename T, typename F>
auto map_ref(T&& v, F f) {
  return map_impl<T&&>(std::forward<T>(v), std::forward<F>(f));
}

template<typename T, typename F>
auto map_val(T&& v, F f) {
  return map_impl(std::forward<T>(v), std::forward<F>(f));
}
```

Here, `map_ref` exploits the same reference collapsing rules that allow
forwarding references to work in order to make the `T` template argument the
same kind of reference as was passed to `map_ref`. In contrast, `map_val` uses
the usual forwarding idiom to cause a copy or move, as appropriate, to be done
into `map_impl`'s argument.

## Summary

1. Be very careful accepting references as parameters to a coroutine.
1. Reference collapsing is weird but pretty useful to understand.
1. Don't forget that the range-expression in a range-based for loop is evaluated
   as part of a full-expression that finishes before the loop starts.
