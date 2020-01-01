---
layout: post
title: Generic, type-safe delegates in C++ (revised)
gh-repo: skypjack/entt
gh-badge: [star, follow]
tags: [entt, cpp, signal]
---

Back in 2011, an
[interesting post](https://blog.molecular-matters.com/2011/09/19/generic-type-safe-delegates-and-events-in-c/)
about _generic, type-safe delegates and events in C++_ appeared on the web.
Most of the implementations of a delegate class you can find around on GitHub
are largely inspired by this article, sometimes with a few changes. Sadly, most
of these implementations also stick to the _pre-C++11-ish_ approach of the
original version and only few tried to add _something more_ to what is described
in the article.

[`EnTT`](https://github.com/skypjack/entt) comes with
[its own implementation](https://github.com/skypjack/entt/blob/master/src/entt/signal/delegate.hpp)
of a delegate class, initially inspired by the post above and lately revised to
be more _modern_ (as in _modern C++_). For a while, it also supported _curried
functions_, data members and lambdas, that are something not even mentioned by
the original author. However, they proved to be not that useful in fact. On the
other side, they came with some limitations due to the rules of the language
that required to pay something in terms of performance to work around their
problems. Because of this and some other reasons, I finally decided to simplify
down the implementation to what I really needed on a daily basis.<br/>

This post still describes the _full featured_ implementation of a delegate. I
added a some notes in the conclusion paragraph to mention its limits and to give
some hints to deal with them.<br/>
Let's give it a look more in details.

## Desiderata

First of all, what are the use cases for such a class? A trivial example is:

```cpp
void my_func(int) { /* ... */ }

struct my_class {
    void my_member(int) { /* ... */ }
};

delegate<void(int)> my_delegate;
my_delegate.connect<&my_func>();
my_delegate(42);

my_class instance;
my_delegate.connect<&my_class::my_member>(&instance);
my_delegate(42);
```

So far, so good. Since C++17, `noexcept` became part of the function definition
as well as `const`. Therefore, we need to find a way to accept all the possible
functions, no matter if they are const, non-const or have a `noexcept`
specifier.<br/>
Moreover, it would be great to have at least a limited support for _curried
functions_:

```cpp
void my_func(char, int) { /* ... */ }

delegate<void(int)> my_delegate;
my_delegate.connect<&my_func>('c');
my_delegate(42);
```

Looks good, doesn't it? The delegate appears as if its function type was
`void(int)`, but I can connect functions with different definitions under
certain constraints.<br/>
Is it that useful? Indeed. Have you ever used the capture list of a lambda?
This is a zero allocation abstraction (much more constrained) of the same
concept and it's useful in a lot of cases in fact.

Finally, I'd like to have also support for data members and lambdas, at least to
a certain extent. All of this is possible thanks to a mix of `auto` template
parameters, placement new and type erasure techniques.

## Aligned storage is the new void *

If you have ever read the post linked above and you remember how it works, a
delegate for a given function type `Ret(Args...)` had two data members like
these:

```cpp
using proto_fn_type = Ret(void *, Args...);
proto_fn_type *wrapper;
void *instance;
```

Where `wrapper` is a pointer to a function used to invoke the original function
connected to the delegate and `instance` contains an erased pointer to the
instance on which to invoke the member function if that's the case.<br/>
In order to support curried functions, lambdas and all the others, this is no
longer the way to go. An opaque storage area of the right size does the job
instead:

```cpp
using storage_type = std::aligned_storage_t<sizeof(void *), alignof(void *)>;
using proto_fn_type = Ret(storage_type &, Args...);
mutable storage_type storage;
proto_fn_type *fn;
```

This is necessary because we want to use the space reserved for the pointer to
store optional values in case the linked function is a free one. After all, the
pointer isn't used in this case and we can freely reuse the storage area as long
as the type `T` of the value to store is such that
`sizeof(T) <= sizeof(void *)`. This way we can accept members along with
instances on which to _invoke_ them as well as free functions with optional
values to provide silently when invoked. Support for lambdas also finds its way
in the delegate because of this and it goes without saying that it's still quite
easy to work with free functions that doesn't have _attached_ parameters.

Fortunately, this doesn't even defeat the purpose of being type-safe.<br/>
In fact, as we will see in the following sections, the erased functions always
know how to deal with the storage area and they can correctly cast things back
and forth this bunch of bytes without risks.

## Member functions and curried functions

In order to connect both a member function and a curried function to a delegate
class, a bit of templates are necessary. We cannot just use a simple pointer to
function or to member function. The reasons behind this are correctly explained
in the original article and it doesn't worth it to repeat them in details.<br/>
To sum up, definitions for pointers to functions differ from those of pointers
to member functions. Moreover, we cannot _erase_ them and put everything in an
old fashioned `void *`, because pointers to free functions and member functions
aren't necessarily guaranteed to fit.

For these and some other reasons, what we need at the end of the day is
something like the following:

```cpp
template<auto Candidate, typename Type>
void connect(Type value_or_instance) {
    static_assert(sizeof(Type) <= sizeof(void *));
    static_assert(std::is_trivially_copyable_v<Type> and std::is_trivially_destructible_v<Type>);
    static_assert(std::is_invocable_r_v<Ret, decltype(Candidate), Type &, Args...>);

    new (&storage) Type{value_or_instance};

    fn = [](storage_type &storage, Args... args) -> Ret {
        Type value_or_instance = *reinterpret_cast<Type *>(&storage);
        return std::invoke(Candidate, value_or_instance, args...);
    };
}
```

In this case, `Candidate` is either a member function or a free function. On the
other side, `Type` is either the type of the class to which the member function
belongs or the type of the value we want to use to create a curried function.
The bunch of `static_assert`s give us enough guarantees on the nature of the
function and that of the parameter.<br/>
For the sake of curiosity, we can easily achieve the same result using two
functions and a bit of SFINAE, but I don't think it's worth it in this
case.<br/>
Note also that we don't even care much of the fact that we are dealing with a
member function or a curried function because of how `std::invoke` works. In
both cases, this form is just fine for our purposes.

The way it works is indeed straightforward. Put aside the `static_assert`s, the
first line of code copies the value received as an argument into the storage
area by means of a placement new. Then, it assigns to `fn` (the type of which is
`Ret(*)(storage_type &, Args...)`) a lambda that decays to a pointer to function
according with the rules of the language. The lambda _sees_ the template
parameters list and is in charge of converting back to its original type the
value previously put in the storage area so as to literally _invoke_ (as in
`std::invoke`) the `Candidate` function with the given parameters.<br/>
Thanks to `std::invoke`, this is enough to support both member functions and
curried functions. In fact:

* If `Candidate` is a member function, `value_or_instance` is guaranteed to be
  a pointer to an instance and `std::invoke` will call it with the given
  arguments `args...`.
* If `Candidate` is a free function, `value_or_instance` is guaranteed to be at
  least convertible to the type of its first argument and `std::invoke` will
  call it while appending `args...` to the first parameter.

If you aren't confident with lambdas and the rules of the language, consider
that it's equivalent to the following snippet:

```cpp
template<auto Candidate, typename Type>
static Ret proto(storage_type &storage, Args... args) {
    Type value_or_instance = *reinterpret_cast<Type *>(&storage);
    return std::invoke(Candidate, value_or_instance, args...);
}

template<auto Candidate, typename Type>
void connect(Type value_or_instance) {
    static_assert(sizeof(Type) <= sizeof(void *));
    static_assert(std::is_trivially_copyable_v<Type> and std::is_trivially_destructible_v<Type>);
    static_assert(std::is_invocable_r_v<Ret, decltype(Candidate), Type &, Args...>);

    new (&storage) Type{value_or_instance};
    fn = &proto<Candidate, Type>;
}
```

Where `proto` is a static private function of the `delegate` class that can be
directly converted to a pointer to function.

One of the most interesting aspects of this implementation is that it uses
`auto` as a non-type template parameter to capture the function to invoke. This
form is rather convenient because it allows to deal with all the types of
functions, no matter if they are const, non-const or has a `noexcept`
specifier. This was exactly our goal.<br/>
Without `auto`, up to four (!!) overloads of `connect` are required to obtain
the same result in C++17 and only for member functions:

```cpp
template<typename Instance, Ret(Instance:: *Member)(Args...)>
void connect(Instance *instance);

template<typename Instance, Ret(Instance:: *Member)(Args...) const>
void connect(Instance *instance);

template<typename Instance, Ret(Instance:: *Member)(Args...) noexcept>
void connect(Instance *instance);

template<typename Instance, Ret(Instance:: *Member)(Args...) const noexcept>
void connect(Instance *instance);
```

Not to mention that the syntax at the call site becomes uglier:

```cpp
my_class instance;
my_delegate.connect<my_class, &my_class::my_member>(&instance);
```

Even more overloads are necessary to deal also with free functions and curried
functions.<br/>
C++17 definetely saved us from writing a lot of redundant code.

## Data members

Another nice to have feature that works out of the box with the implementation
described in the previous section is the support for const and non-const data
members. In fact, you can freely connect them to a delegate if required:

```cpp
struct my_class {
    const int value = 42;
};

delegate<int()> my_delegate;
my_class instance;

my_delegate.connect<&my_class::value>(&instance);
int value = my_delegate();
```

In this case, the function type of the delegate must be such that the parameter
list is empty and the the value of the data member is at least convertible to
the return type. Once connected, you can invoke the delegate itself to read the
value contained in the data member.

## Free functions

What we didn't manage with the definitions we've seen so far are the free
functions to which we don't want to _attach_ parameters.<br/>
To do that, we need to define another overload for `connect`:

```cpp
template<auto Function>
void connect() {
    static_assert(std::is_invocable_r_v<Ret, decltype(Function), Args...>);

    new (&storage) void *{nullptr};

    fn = [](storage_type &, Args... args) -> Ret {
        return std::invoke(Function, args...);
    };
}
```

It works more or less as its counterpart, but for the fact that the storage area
is ignored. In order to use the same pointer to function (namely `fn`) in both
cases, the function type must be the same and that's why we keep passing the
storage area around even if it's unused.

## Lambdas and functors

We said before that it would be really great if we can make the delegate class
work also with lambdas. However, we want to avoid allocations if possible and
use only the space reserved for the storage area. This is due to the fact that
we don't have instances connected to the delegate in this case.<br/>
What is allowed within these constraints is to accept lambdas (and functors in
general, so even user defined classes) the size of which fits the one of a
`void *`. In other terms, non-capturing lambdas and lambdas that capture
primitive types or pointers should work just fine.

To force users to pass free functions as template parameters, we can use a mix
of `static_assert`s and `std::is_class_v` trait. Moreover, since we don't have
the chance to explicitly invoke the destructor for these objects (it would
require another pointer to function for that, if someone is interested), they
must be also trivially destructible.<br/>
Here it is an overload of the `connect` member function that gets the job done:

```
template<typename Invokable>
void connect(Invokable invokable) {
    static_assert(sizeof(Invokable) < sizeof(void *));
    static_assert(std::is_class_v<Invokable>);
    static_assert(std::is_trivially_destructible_v<Invokable>);
    static_assert(std::is_invocable_r_v<Ret, Invokable, Args...>);

    new (&storage) Invokable{std::move(invokable)};

    fn = [](storage_type &storage, Args... args) -> Ret {
        Invokable &invokable = *reinterpret_cast<Invokable *>(&storage);
        return std::invoke(invokable, args...);
    };
}
```

It doesn't differ much from what we've seen so far and I won't go in details.
What is important is that we can also do this from now on:

```cpp
delegate<void(int)> my_delegate;
my_delegate.connect([value = my_int_variable](int v) { return v * value; });
```

It's not as flexible as an `std::function`, mainly because of the limitations
put on the capture list. However it's more than enough in a lot of cases and
it's surely another useful tool in our pocket.

## A function call operator to rule them all

A delegate is an opaque wrapper for free functions, member and so on. To invoke
them, we can introduce a function call operator that accepts the parameters used
for the function type of the delegate:

```cpp
Ret operator()(Args... args) const {
    return fn(storage, args...);
}
```

Type erasure is such that there is no way to literally _extract_ the type(s)
once erased. All what you can do is to push things down along the chain and
expect someone to know how to treat them.<br/>
This is exactly the case. The delegate class knows what are the parameters it
accepts but it doesn't know if it has to invoke a free function, a member
function or whatever, nor how to treat the storage area. On the other side, the
function we assigned to `fn` has all the required information. Because of that,
a delegate has only to forward the storage area and the parameters directly to
the erased function stored by `fn` and return its value, if any.

## Constructors and deduction guidelines

Let's take a closer look at the initial example to see how we defined a delegate
and connected a function to it:

```cpp
delegate<void(int)> my_delegate;
my_delegate.connect<&my_func>();
```

It would nice to do it all at once with a parameter passed directly to the
constructor of the delegate. Something like this, where we get rid also of the
function type of the delegate and let the class deduce it for itself:

```cpp
delegate my_delegate{connect_arg_t<&my_func>};
```

The _problem_ is that we used the `auto` template parameter to capture all the
possible function types. Because of that, we need to exploit some tricks to know
what's the return type and what are the arguments of the function we are going
to connect. Otherwise, it would be impossible to deduce the function type used
to specialize the delegate itself.

This is quite easy indeed. What we need is to write a function declaration (no
definition required) like this:

```cpp
template<typename Ret, typename... Args>
auto to_function_pointer(Ret(*)(Args...)) -> Ret(*)(Args...);
```

As you can see, it returns a pointer to a function having type `Ret(Args...)`.
Unfortunately we cannot return directly a function type, but we can still use
`std::remove_pointer_t` on the return type to know it:

```cpp
std::remove_pointer_t<decltype(to_function_pointer(Function))>;
```

If you're trying to figure out why `noexcept` doesn't compare in the code above,
that's because `Ret(*)(Args...) noexcept` (let me say) _converts_ to
`Ret(*)(Args...)`, while the other way around doesn't work. Therefore we don't
care about `noexcept` and this is enough to deal with all the possible function
types.

However, this isn't enough yet to reach our goal. In fact, we need also to write
a deduction guideline to get rid of the function type for the delegate:

```cpp
template<auto Function>
delegate(connect_arg_t<Function>)
-> delegate<std::remove_pointer_t<decltype(to_function_pointer(Function))>>;
```

Now we can easily define a constructor that accepts a function to connect:

```cpp
template<auto Function>
delegate(connect_arg_t<Function>)
    : delegate{}
{
    connect<Function>();
}
```

Then have delegates deduce their own function types:

```cpp
delegate my_delegate{connect_arg_t<&my_func>};
```

And that's it. Obviously, something similar can also be done for members,
curried functions and lambdas. For the sake of brevity, I won't repeat here the
whole process for all of them.

## Conclusion

The `delegate` class offered more functionalities than what we discussed here.
However, these are the basic concepts behind its implementation and they show
how modern C++ helps us to write less and get more from our code.

This class isn't intended as a drop-in replacement for an `std::function`, it's
by far less flexible than a lambda and shares with them some problems (as an
example, users must guarantee that the lifetime of an object connected to a
delegate overcomes the one of the delegate itself).<br/>
However, it has also some advantages over an `std::function` or a lambda. This
is mainly due to the fact that it's a zero allocation abstraction that has a
known type and it makes possible to use a delegate as a data member or as a
parameter of a non-template function.

The implementation shown above has also some limits you cannot and should not
ignore. In particular, whenever you copy or move a delegate and use the new
instance, the  behavior is technically undefined. This is due to the fact that
an object hasn't been explicitly constructed in the storage area of the second
instance, even if the bytes it contains are exactly the same of the source.
Because of that, the `reinterpret_cast` of the storage area to a specific type
doesn't magically create a type for you and what you obtain isn't properly an
object, at least according to the rules of the standard. Ironically, it behaves
as if it's a valid object, but it is not for the language.<br/>
Roughly speaking, it works with all the major compilers, but use it at your own
risk. Unlikely it will stop working, because there is a lot of code out there
that relies on this assumption and to change this behavior would mean to break a
lot of  working programs. I don't think a compiler will ever dare so much, but
who knows?<br/>
In any case, it's worth mentioning that the magic is based on a risky
assumption. For all those interested in the _standardese_,
[here](https://stackoverflow.com/questions/54512451/about-aligned-storage-and-trivially-copyable-destructible-types)
is also an interesting Q/A on SO that goes a bit deeper into the problem.

If you're interested in the topic, take a look at the full implementation of the
[delegate](https://github.com/skypjack/entt/blob/master/src/entt/signal/delegate.hpp)
class that is part of [`EnTT`](https://github.com/skypjack/entt) or visit the
[wiki page](https://github.com/skypjack/entt/wiki/Crash-Course:-events,-signals-and-everything-in-between#delegate)
to know how to use it.

## Let me know that it helped

If you liked this post and want to say thanks, consider to star the
[GitHub project](https://github.com/skypjack/skypjack.github.io) that hosts this
blog. It's the only way you have to let me know that you appreciate my work.

Thanks.
