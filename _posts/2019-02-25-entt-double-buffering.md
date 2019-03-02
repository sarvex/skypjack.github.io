---
layout: post
title: EnTT and double buffering
gh-repo: skypjack/entt
tags: [entt]
---

More than once I've been asked to put built-in support for double buffering or
N-buffering in general within [EnTT](https://github.com/skypjack/entt). A
[patch](https://github.com/skypjack/entt/pull/171) has also been submitted but
it had some inherent problems and could not be merged.<br/>
Fortunately double buffering can also be easily implemented on top of a registry
and it seems this is already a good way to make it both safe and
feature-complete at the same time.

Let's discuss what are the problem with making it built-in in the registry and
how we can develop an external hand-crafted tool in a few steps.

## Introduction

EnTT stores all the components in separate arrays and makes always available a
couple `(T *, size)` for each type of component. Moreover, it doesn't support
multiple pools for the same type.<br/>
The registry heavily relies on type erasure techniques to store everything and
to support adding different types of components dinamically at runtime. Because
of that, it uses a few methods (let's say a single one for simplicity) to give
unique integer identifiers to types. Imagine as if it stored pools for all the
types in an array and knew the index to use to access the right pool when
needed. This has a lot of pros (it's easy to guarantee type safety to begin
with) but makes almost impossible to literally _change_ the pool for a given
type, that is all what you need for double buffering at the end of the day.

So, what can we do to work around this limitation?

## A failed attempt

The proposed patch and some other proposals discussed in the
[gitter channel](https://gitter.im/skypjack/entt) were quite smart and _work_ to
an extent, for some definitions of _work_. The main problem is that more or less
all of them try to exploit an UB that is unlikely to change, but it's still an
UB.

The basic idea is almost always to define two components as derived from a same
base class, the type for which we would like to allow double buffering:

```cpp
struct position { int x; int y; };
struct position_1: position {};
struct position_2: position {};
```

This way `position_1` and `position_2` are actually different types (at least
from the point of view of the type system) having the same memory layout.<br/>
Consider now the following snippet (it isn't the real code, but it should be far
easier to understand):

```cpp
template<typename T>
struct wrapper { T instance; };

// ...

wrapper<position_1> pos1;
wrapper<position_2> pos2;

void *ptr1 = &pos1;
void *ptr2 = &pos2;

// ...

std::swap(ptr1, ptr2);
auto &ohoh = *static_cast<wrapper<position_1> *>(ptr1);
ohoh.instance.x = 3;
```

Oh oh. If you compile this code, it _works_. However, you cannot convert `ptr1`
to `wrapper<position_1> *` after the swap, because it points to an instance of
`wrapper<position_2>` and the two instances have nothing in common. In other
terms, it's technically UB.<br/>
Why does it work then? Roughly speaking, there is a lot of code out there that
relies on the assumption that this kind of cast made between types with the same
memory layout just _works_. No matter if we like it or not. This doesn't turn it
in a _valid_ piece of code anyway and I preferred to avoid offering features
based on risky things like this one. In fact, the proposed patch was based on
the same assumption, even though the code involved is much more complex than
this. The proposal required to change slightly the API to allow swapping
silently two pools under the hood, so that a registry would have `static_cast`ed
an erased pointer originally created for a pool for `position_1`s to a pool for
`position_2`s.

### Groups and built-in solutions

Another problem to face with a built-in solution are groups, in particular
owning groups.

When a group owns a type of component, it owns its pool and is allowed to
rearrange instances in the pool so as to iterate them as fast as possible. To do
that and to reduce to a minimum the number of operations made on a pool when
needed, a group stores aside some information to use later. These information
are strictly related to the given pool and to the way things are laid out within
it (I see it's not that clear, but an in-depth explanation of how groups work is
beyond the purpose of this post). It goes without saying that swapping pools
will break the groups that own them, if any.<br/>
To be honest, there are some precautions that can be taken to avoid this
problem, but they are in charge to the users that should set up specific
listeners for the components involved by double buffering. If I was one of those
users, I'm pretty sure I would forget it sooner or later and this would lead to
subtle bugs hard to track down at runtime.<br/>
Definitely error-prone and quite annoying a constraint.

## A trivial but working solution

So, how do we achieve the goal in a type-safe manner?<br/>
There doesn't exist a single answer to the question. I'm going to show a
possible solution that is built on top of a registry and doesn't require to
change its API. Of course, I won't describe a full-featured implementation.
Instead, I'll give only some hints to allow those interested in the topic to
develop their own solution.

A possible use case is the following one:

```cpp
buffering<entt::type_list<position_1, position_2>, my_type> executor;

const auto entity = registry.create();
registry.assign<position_1>(entity, ...);
registry.assign<position_2>(entity, ...);
registry.assign<my_type>(entity, ...);

executor.run(registry, [i](position &pos, my_type &instance) { /* ... */ });
executor.swap();
executor.run(registry, [i](position &pos, my_type &instance) { /* ... */ });
```

As above, `position_1` and `position_2` both derive from `position` and the
latter is the type we want to receive in our lambda. `swap` will literally
_change_ the actual type to get from the registry.<br/>
Because it's not a built-in feature offered by the registry, a sort of
_accessor_ helps to make double buffering transparent for the user. We don't
want to have to deal with pointers to members or anything else waiting for us.

The following implementation uses an `entt::view` under the hood for the sake
of simplicity. Probably a good choice is to use groups where components for
which we want to set up double buffering (eg `position_1` and `position_2`)
are owned types and `my_type` is a free type. I won't go in depth on this,
mainly because it's up to you to decide what is worth in your real code.

First of all, we need to separate the types that are involved by the double
buffering and those that are not. This is quite easy with a bit of template
machinery:

```cpp
template<typename...>
class buffering;

template<typename... Common, typename... Types>
class buffering<entt::type_list<Common...>, Types...> {
    using ident = entt::identifier<Common...>;

    // ...

private:
    std::size_t next{};
};
```

`Common` is a type list we want to _iterate_ somehow. To do that, we need to
give sequential identifiers to types. Fortunately, `EnTT` helps us with its
`entt::identifier` tool. See
[here](https://github.com/skypjack/entt/wiki/Crash-Course:-core-functionalities#compile-time-identifiers)
for all the details.<br/>
`Types` are the types we want to get other than the one returned because of the
double buffering and `next` is the _current index_ we are working with.

The `swap` function is straightforward to implement:

```cpp
void swap() {
    next = (next + 1) % sizeof...(Common);
}
```

All what it does is to increment the current index and ensure it wraps around
when it reaches `sizeof...(Common)`. We assume in this case that we want to
iterate types as if they were in a loop array.

So far, so good. We have still to implement the `run` function we saw a few
lines above. The purpose of this function is to get the _right_ pool (whatever
_right_ means in this case), then provide the lambda with a reference to a
`position` and a reference to an instance of `my_type`.<br/>
How to do that? Well, we have an index and a bunch of numeric identifiers for
our types. The function implementation should be obvious at this point:

```cpp
    template<typename Type, typename... Other, typename Func>
    void execute(entt::registry<> &registry, Func &&func) {
        if(next == ident::template type<Type>) {
            registry.view<Type, Types...>().each(std::forward<Func>(func));
        } else {
            if constexpr(sizeof...(Other) == 0) {
                assert(false);
            } else {
                execute<Other...>(registry, std::forward<Func>(func));
            }
        }
    }

public:
    template<typename Func>
    void run(entt::registry<> &registry, Func &&func) {
        execute<Common...>(registry, std::forward<Func>(func));
    }
```

`run` just forwards the lambda to an internal function that accepts all the
types as template arguments and _iterates_ them one at a time to see if they
match with the current index. Pretty simple to be honest and nothing of which
to worry about.<br/>
The `execute` function is instead the core of the `buffering` class and it's
simple as well, that is a good news for us. In practice:

* If the numeric identifier of the _i-th_ type is equal to the current index, we
  use that type as the first argument of the template parameter list for
  `registry.view` and we forward the function directly to `view.each`.

* If we didn't reach the current index yet, we invoke `execute` recursively. The
  `assert(false)` for the last case is just a placeholder for the sake of
  curiosity, but we can get rid of it because of how `swap` is defined and
  everything will continue to work correctly. In fact, we are guaranteed that we
  won't reach the `assert` in any case.

The way `execute` iterates types is by _consuming_ them one at a time. As you
can see from the recursion step, `Type` isn't propagated and only the  parameter
pack `Other` is unpacked in the template parameter list instead.

That's all. The double buffering is up and running. Type safe, transparent, easy
to use and the class is so small and poor of features that one can easily extend
it to add all what is needed without the risk of breaking things.<br/>
The sole objection I can see is that we are _iterating_ types each and every
time we invoke `run`. However, consider that `sizeof...(Common)` is 2 for double
buffering and hardly it will get greater than 3 or 4 any time soon. If you're
going to iterate 150k/200k entities, you won't even notice the impact of `run`
on you loop. If you're not iterating 150k/200k entities, you shouldn't even care
of this.

### Side notes

First of all, note that groups aren't affected by this solution. They work like
a charm and aren't even aware of double buffering. In fact, we are no longer
_switching pools_ here, so private data for groups will remain consistent all
the time for obvious reasons.<br/>
Views also won't have any problem with this approach and they are no longer
required to do risky casts to invalid types, thus exploiting UB under the hood.

Despite the proposed solution apparently _works_ already without errors, the
result could not be the expected one when users forget to attach to an entity
all the components for which they want to set up N-buffering (`position_1` and
`position_2` in our example). Similarly, if one of the components is removed,
all the others should be removed as well. Otherwise, the set of entities
returned after a swap could differ and contain too much components or few than
expected.<br/>
Fortunately, the registry allows to attach listeners on pools to be notified on
construction and destruction of components. To set up an automatism and forget
about the dependency problem, you can safely use this technique and do the
_dirty work_ inside your listeners. The way to go is straightforward indeed:

* When one of the components is constructed, `assign_or_replace` to the same
  entity all the others.

* When one of the components is removed, `reset` all the others for the same
  entity.

See `registry::construction` and `registry::destruction` for that.

### A refined solution

A limit of the solution presented above is that the signature of the view is
part of the type itself. It would be great if `buffering` accepted only the
types to switch on and the others were passed when calling `run`.<br/>
The revised use case is this:

```cpp
buffering<position_1, position_2> executor;

const auto entity = registry.create();
registry.assign<position_1>(entity, ...);
registry.assign<position_2>(entity, ...);
registry.assign<my_type>(entity, ...);

executor.run<my_type>(registry, [i](position &pos, my_type &instance) { /* ... */ });
executor.swap();
executor.run<my_type>(registry, [i](position &pos, my_type &instance) { /* ... */ });
```

If it seems that anything changed, consider that you can use your `executor` for
all the types of views now:

```cpp
executor.run<my_type>(registry, [i](position &pos, my_type &instance) { /* ... */ });
// ...
executor.run<another_type>(registry, [i](position &pos, another_type &instance) { /* ... */ });
```

With the previous implementation, two instances of `buffering` were required.
The problem was that all the instances must be kept in sync probably and this
might not be that easy to do.<br/>
This revised version gets rid of the problem:

```cpp
template<typename... Types>
class buffering {
    using ident = entt::identifier<Types...>;

    template<typename Type, typename... Tail, typename... Other, typename Func>
    void execute(entt::type_list<Other...>, entt::registry<> &registry, Func &&func) {
        if(curr == ident::template type<Type>) {
            registry.view<Type, Other...>().each(std::forward<Func>(func));
        } else {
            if constexpr(sizeof...(Tail) == 0) {
                assert(false);
            } else {
                execute<Tail...>(entt::type_list<Other...>{}, registry, std::forward<Func>(func));
            }
        }
    }

public:
    template<typename... Other, typename Func>
    void run(entt::registry<> &registry, Func &&func) {
        execute<Types...>(entt::type_list<Other...>{}, registry, std::forward<Func>(func));
    }

    void next() {
        curr = (curr + 1) % sizeof...(Types);
    }

private:
    std::size_t curr{};
};
```

No need to walk through the whole class again. As you can see, I just moved
parameters from the template parameter list of the class to the one of the `run`
function.

## Conclusion

This is a _quick & dirty_ solution to show you an idea I had for the problem of
double buffering when it comes to speaking of `EnTT`. You can also consider this
post as a reference to be used for the next time someone will ask me about the
topic.<br/>
To be honest, we are still discussing in the
[gitter channel](https://gitter.im/skypjack/entt) a few other approaches to use
to offer built-in support for N-buffering directly from the registry. However,
none of the solutions proposed so far has proved worthy of being integrated.
Probably we will find the _right one_ sooner or later, but until then you've to
develop your own solution on top of the registry and this can be a good starting
point for that.

In all cases, if you ever had the problem of doing double buffering with `EnTT`
or any other ECS library, this post can help you.

## Let me know that it helped

If you liked this post and want to say thanks, consider to star the
[GitHub project](https://github.com/skypjack/skypjack.github.io) that hosts this
blog. It's the only way you have to let me know.

Thanks.
