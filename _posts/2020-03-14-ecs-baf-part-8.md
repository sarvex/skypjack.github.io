---
layout: post
title: ECS back and forth
subtitle: Part 8 - Type Id
g5-repo: skypjack/entt
tags: [ecs, entt, cpp]
---

When I started writing this series, the approach used to make
[`EnTT`](https://github.com/skypjack/entt) work across boundaries was
questionable. It worked but in a cumbersome manner that was difficult to work
with, sometimes unclear or even dangerously close to being unstable.<br/>
I tried to give some insights of this approach with my
[first post](https://skypjack.github.io/2019-02-14-ecs-baf-part-1/). Long story
short, that was a fully runtime solution used to generate sequential identifiers
that worked just fine for a self-contained application. However, one could get
in troubles across boundaries because of them.

Nowadays, `EnTT` relies on a completely different approach. It mixes standard
and non-standard features in a single solution that is also sfinae-friendly and
therefore fully customizable in case users already have their own type id
generation system.<br/>
The current generator is `constexpr` in the best case but it has also a runtime
fallback if needed.

If you haven't done it yet, read the
[previous parts](https://skypjack.github.io/tags/#ecs) of the series before to
continue. They will probably help to fully understand this one, even though they
aren't strictly necessary.

## Introduction

In C++, there isn't a reliable way to associate unique identifiers to our types.
One could argue the there exists an `std::type_info` class capable of returning
information for a given type but:

* These information are implementation defined.
* There is no guarantee that they are stable across different runs.
* It's recommended to avoid collisions but it may still happen and there is no
  way to get around this in case.

Not exactly the right tool for our purposes most of the times. So, all in all,
we cannot really differentiate among types in a safe manner out of the box.

A quick search online gives us many appealing solutions though. Some are based
on a cast of a function pointer _returned_ by the specialization of a function
template, some others exploit a slightly different feature of the language to
turn this in a sequential identifier.<br/>
Unfortunately, none of them is reliable, stable across different runs and usable
across boundaries at the same time.

So am I telling you that there is no solution? Not quite.<br/>
We can approach it from multiple sides and try to get the best out of what we
have.

### Don't panic

The following class is a refined version of my first attempt:

```cpp
class generator {
    inline static std::size_t counter{};

public:
    template<typename Type>
    inline static const std::size_t type = counter++;
};
```

This works like a charm in a standalone application and can generate sequential
identifiers that aren't necessarily stable across different runs, nor across
boundaries.<br/>
We can use it like this:

```cpp
const auto type = generator::type<my_class>;
```

The basic idea is quite simple. We have a shared counter that is incremented
every time we generate an identifier for a new type. Since `type` is a variable
template, it forces an increment of `counter` with every specialization.<br/>
In case `type<T>` has been already initialized, its value is returned without
affecting `counter` anymore.

This works because of how inline variables work. A thin layer of template
machinery makes the magic happen.<br/>
However, these identifiers are generated at runtime and aren't therefore that
_stable_ unless you force them to be so somehow.

Consider the following snippet:

```cpp
if(condition) {
    const auto it = generator::type<int>;
    const auto ct = generator::type<char>;

    // ...
} else {
    const auto ct = generator::type<char>;
    const auto it = generator::type<int>;

    // ...
}
```

In one case `int` is assigned the identifier N and `char` is assigned the
identifier N+1. In the other case, the two types have opposite identifiers
instead. What will happen strictly depends on `condition` the first time we
encounter this statement.<br/>
This is a trivial example that is unlikely to happen but in a complex codebase
something like this could be lying everywhere, deeply buried under a chain of
function calls or whatever.

Another problem of this solution is that it doesn't work across boundaries in
all cases.<br/>
On linux, with default visibility, it just works because everything is literally
_public_ and thus _exposed_ to the linker. On Windows, it can easily break and
we can have different identifiers assigned to the same type from different sides
of a boundary.<br/>
With a bunch of `dllimport`, `dllexport` and
`__attribute__((visibility("default")))` we can get around it though but there
are still problems with plugins:

```cpp
struct GENERATOR_API generator {
    static std::size_t next() {
        static std::size_t value{};
        return value++;
    }
};

template<typename Type>
struct GENERATOR_API type {
    static std::size_t id() {
        static const std::size_t value = generator::next();
        return value;
    }
};
```

The slightly different form is due to some idiosyncrasies of MSVC that has some
serious limitations when it comes to working across boundaries.<br/>
On the other hand, `GENERATOR_API` is the _well known_ macro we find everywhere
for this kind of things:

```cpp
#if defined _WIN32 || defined __CYGWIN__ || defined _MSC_VER
#    define GENERATOR_EXPORT __declspec(dllexport)
#    define GENERATOR_IMPORT __declspec(dllimport)
#elif defined __GNUC__ && __GNUC__ >= 4
#    define GENERATOR_EXPORT __attribute__((visibility("default")))
#    define GENERATOR_IMPORT __attribute__((visibility("default")))
#else /* Unsupported compiler */
#    define GENERATOR_EXPORT
#    define GENERATOR_IMPORT
#endif


#ifndef GENERATOR_API
#   if defined GENERATOR_API_EXPORT
#       define GENERATOR_API GENERATOR_EXPORT
#   elif defined GENERATOR_API_IMPORT
#       define GENERATOR_API GENERATOR_IMPORT
#   else /* No API */
#       define GENERATOR_API
#   endif
#endif
```

This is a reduced version for the sake of brevity but I'm sure you know what I'm
talking about at this point.

## May the standard C++ be with you

The standard is our source of truth. It's also very clear when it comes to
dispelling doubts about what it means to work across boundaries.<br/>
We can sum up all it spends on this topic with the following:

> ...

Absolutely nothing. Working across boundaries isn't something that the standard
takes or should even take in consideration and, in fact, it doesn't help much
here.<br/>
This means that there isn't a _standard way_ to do that. This cannot mean
anything good though.

However, the standard may perhaps help us with our goal. Most likely there is
something that can help us define stable type identifiers that work well also
across boundaries.<br/>
We played with function templates so far. Let's see if we can get _more_ out of
them.

### Also the standard disappoints

Apparently, `__func__` seems to be a good candidate for extrapolating some
useful information from a function. At least that was what I thought the first
time I saw it. Well, it is not.<br/>
The uselessness of `__func__` in C++ cannot be easily described in my opinion.

First of all, what's that? It's a predefined variable that is defined within the
function body of every function. So far, so good. Sounds interesting.<br/>
The actual definition already raises some doubts though:

```cpp
static const char __func__[] = "function-name";
```

In other terms, this variable _contains_ `foo` within the following function:

```cpp
void foo();
```

What if `foo` is a function template instead? It doesn't matter. `__func__`
still _contains_ `foo` and the template parameter used to specialize it is lost.
It's not part of this variable, that's all.

### Standardish alternatives

Unfortunately, there isn't a refined counterpart of `__func__` that works also
with function templates in the standard. However, the major compilers introduced
some _standardish_ solutions to get around this limitation.<br/>
In particular, MSVC offers the `__FUNCSIG__` macro while both clang and GCC
make available the `__PRETTY_FUNCTION_` identifier. With the most recent
versions of these compilers, these literals are also constant expressions and
therefore they can be used at compile-time.

Suppose now we have a compile-time hashing function (yes, it's possible). With
these things at hand, we can easily compute an identifier for a type during
compilation and use it everywhere with no costs at runtime:

```cpp
template<typename Type>
struct generator {
    static constexpr std::size_t id() {
        constexpr auto value = hash_fn(GENERATOR_PRETTY_FUNCTION);
        return value;
    }
};
```

Here `GENERATOR_PRETTY_FUNCTION` is an opaque identifier used to abstract the
actual name on the different systems. Something along this line is enough in
most of the cases:

```cpp
#if defined _MSC_VER
#   define GENERATOR_PRETTY_FUNCTION __FUNCSIG__
#elif defined __clang__ || (defined __GNUC__)
#   define GENERATOR_PRETTY_FUNCTION __PRETTY_FUNCTION__
#endif
```

This solution isn't perfect and doesn't work with less recent compilers but it's
enough for our purposes. For the sake of brevity, I won't spend more time on
discussing how to refine it.<br/>
What makes it attractive instead is the fact that even though these identifiers
are no longer sequential, they are at least _stable_ across boundaries (well,
under certain circumnstances but let's assume this is always the case for a
moment) and between different runs.

### A refined solution

Let's put everything together finally. This is what we obtain:

```cpp
struct GENERATOR_API generator {
    static std::size_t next() {
        static std::size_t value{};
        return value++;
    }
};

template<typename Type>
struct GENERATOR_API type {
#if defined GENERATOR_PRETTY_FUNCTION
    static constexpr std::size_t id() {
        constexpr auto value = hash_fn(GENERATOR_PRETTY_FUNCTION);
        return value;
    }
#else
    static std::size_t id() {
        static const std::size_t value = generator::next();
        return value;
    }
#endif
};
```

So far, so good. We have a generator that works at compile-time by relying on
some non-standard solutions. Morever, there exists a runtime, fully standard
fallback just in case.<br/>
We can even get around conflicts by means of a class specialization. In fact,
since we are hashing _strings_ on which we have no control, we cannot just
ignore the problem this time.

The only thing we can't do (yet) is to provide a generation function for a
_family_ of types, for example on a per-traits basis. This is where SFINAE can
enter the game to add that bit of salt that makes our solution suitable for all
purposes.

### Welcome SFINAE

It's not that hard to get the job done actually:

```cpp
template<typename Type, typename = void>
struct GENERATOR_API type {
    // ...
};
```

By adding a second template parameter, we make it possible to specialize this
class on a per-traits basis.<br/>
As an example, in case we want to use a built-in generation system for a bunch
of types we receive from a third-party library:

```cpp
template<typename Type>
struct GENERATOR_API type<Type, std::void_t<decltype(Type::id())>> {
    static constexpr std::size_t id() {
        return Type::id();
    }
};
```

Of course, I admit that it may not be the most attractive of the features and
that we can live quite well without it but while I was there, why not...

## Conclusion

The solution proposed in [EnTT](https://github.com/skypjack/entt) is slightly
more complex than the one described but nothing particularly striking.<br/>
What left me stunned is that on the one hand there is no reliable way to
uniquely identify a type in a stable manner across boundaries and between
different runs, on the other there is no tool that can be used for the purpose.

Unfortunately, `<typeinfo>` doesn't have much to offer in this sense, at least
as I see it. It seems more like an attempt made without trying too hard for some
reasons unknown to me.<br/>
To be honest, I don't think the solution described above is the only one or the
best to get around the lack of an appropriate tool but it _gets the job done_ in
a straightforward manner and this is what matters most of the times.

As usual, for any comment or suggestion, ping me. Especially this time!<br/>
This is the best that I've managed to do so far but I wouldn't be disappointed
to discover that there are alternative solutions that I have not yet considered.

## Let me know that it helped

If you liked this post and want to say thanks, consider to star the
[GitHub project](https://github.com/skypjack/skypjack.github.io) that hosts this
blog. It's the only way you have to let me know that you appreciate my work.

Thanks.
