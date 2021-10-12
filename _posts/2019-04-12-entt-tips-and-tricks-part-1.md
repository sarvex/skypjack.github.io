---
layout: post
title: EnTT tips & tricks
subtitle: Groups on sale
gh-repo: skypjack/entt
gh-badge: [star, follow]
tags: [entt]
---

Groups in [`EnTT`](https://github.com/skypjack/entt) are an incredibly powerful
tool that allows for perfect SoA under certain circumnstances. However, they've
also some limitations.<br/>
How far can we go to squeeze the best from them?

With this post, I'll describe some tricks to get something more from a group.
In fact, there are some groups that are _on sale_ as a result of the definition
of other groups. However, it's not that obvious sometimes. At least, it is not
if one doesn't know
[how they work](https://skypjack.github.io/2019-03-21-ecs-baf-part-2-insights/)
under the hood.<br/>
Because I've found this trick useful more than once, it's worth sharing it.

## Groups on sale

Suppose we have two components we want to group and to iterate sooner or later.
They are named `A` and `B` (abounding with imagination). We can combine them in
different ways to form a group (mainly changing the ownership model):

* As a full-owning group: `registry.group<A, B>()`.
* As a partial-owning group: `registry.group<A>(entt::get<B>)`.
* As a non-owning group: `registry.group<>(entt::get<A, B>)`.

Unfortunately, the last case is also the only one that doesn't leave much room
for optimizations and improvements, since the group doesn't own any of the
components. However, the other two cases give us some implicit groups that
aren't immediately obvious.<br/>
This is due to the fact that to own a component for a group means also to be
allowed to reorder its elements in the underlying pool. Even though the way
things are arranged depends mainly on some internals of the library, we can
easily deduce it from the exposed API and exploit the way elements are laid out
for our purposes.

### Partial-owning groups

Let's consider the following partial-owning group:

```
auto group = registry.group<A>(entt::get<B>);
```

We don't know much about the free type `B`. The group relies on indirection to
access it and the order for entities and components is imposed by another group,
if any.<br/>
On the other hand, we know what is the _size_ of the group. This tells us also
how many instances of `A` are tightly packed at the top of the internal arrays.
These same entities are those that are part of the group, that is the ones that
have both `A` and `B`.

Interesting enough. If you get an array and move to one side all the elements
that respect a given requirement, it means that on the other side there are all
the elements that don't respect the same requirement. In this case, it means
that the packed arrays of entities and components for `A` are split in such a
way that on one side there are the entities that have both `A` and `B` while on
the other side there are the entities that have `A` only.

How can we exploit it?

Quite simple indeed, at least with the default pools. Remember that this type of
pools allow to get a couple `(T*, size)` both for the entities and for the
components for a given type `T`. The arrays thus obtained contain all the
elements of the pool, not only those that are part of the group. Therefore, it's
a matter of knowing how to skip the second set.

First of all, we know that the following ranges are guaranteed to be valid and
such that they contain the elements that are part of the group itself if the
group owns the given component:

* `[group.data<A>(), group.data<A>() + group.size()]`
* `[group.raw<A>(), group.raw<A>() + group.size()]`

Moreover, the following ranges are guaranteed to contain all the elements that
are part of the underlying pool, not only the ones that are owned by the group:

* `[group.data<A>(), group.data<A>() + group.size<A>()]`
* `[group.raw<A>(), group.raw<A>() + group.size<A>()]`

This is enough for our purposes. The elements for the component `A` are `size()`
positions away from `data<A>()` and `raw<A>()`. In other terms, these are the
ranges that contain them:

* `[group.data<A>() + group.size(), group.data<A>() + group.size<A>()]`
* `[group.raw<A>() + group.size(), group.raw<A>() + group.size<A>()]`

To iterate all the entities that have both `A` and `B` before those that have
only `A`, this is therefore the resulting code:

```
std::for_each(group.data<A>(), group.data<A>() + group.size(), [raw = group.raw<A>()](auto entity) mutable {
    auto &component = *(raw++);
    // ...
});

std::for_each(group.data<A>() + group.size(), group.data<A>() + group.size<A>(), [raw = group.raw<A>() + group.size()](auto entity) mutable {
    auto &component = *(raw++);
    // ...
});
```

Note that all the arrays we are visiting are tightly packed and iterated in
order. Therefore cache misses are reduced to a minimum. It's more or less like
iterating a single component view, but getting more out of it.<br/>
Of course, this example is certainly extreme, since nothing prevents us from
using `each` or the built-in iterators to iterate the group. However, it gives
an idea of how the data are actually organized behind the scenes and how we can
get what remains if needed once we have done with the group.

### Full-owning groups

What happens if the previous group is a full-owning one instead of a
partial-owning group?

```
auto group = registry.group<A, B>();
```

In fact, all what have been said so far is still valid. The only difference is
that the same reasoning applies also to the component `B`.<br/>
In other terms, this time all of these are for free with a sole group:

* The entities that have both `A` and `B` and their components.
* The entities that have `A` only and their components.
* The entities that have `B` only and their components.

Nothing more, nothing less. We have three contiguous arrays of entities (and
components) to iterate when needed. With the right code we can get the best from
the group and iterate it as if we were iterating a plain single component view:

```
group.each([](auto &component_a, auto &component_b) {
    // ...
});

std::for_each(group.raw<A>() + group.size(), group.raw<A>() + group.size<A>(), [](auto &component) {
    // ...
});

std::for_each(group.raw<B>() + group.size(), group.raw<B>() + group.size<B>(), [](auto &component) {
    // ...
});
```

I've used a slightly different snippet this time to show yet another alternative
approach. In this case, the components in the group are iterated directly by
means of the `each` member function. On the other hand, what remains from the
pools for `A` and `B` is iterated using the raw pointers only. This is an even
faster approach to iterate only the components when we aren't interested in the
entity identifiers.<br/>
Remember that what remains from the pool for a given component are the elements
that aren't part of the group. As an example, when we iterate the elements from
the pool of `A`, we are iterating the components assigned to entities that have
not `B`.

### What about larger groups?

Unfortunately, this trick doesn't extend to larger groups. Or rather, it applies
but only within certain limits. Consider as an example the following group:

```
auto group = registry.group<A, B, C>();
```

Obviously, the entities and components for the triplet `A`, `B` and `C` are
still available, as a consequence of the definition of the group itself.<br/>
However, what remains from `A` are no longer the elements that have an instance
of `A` only. Instead, they are a mix of the following:

* The elements that have `A` only.
* The elements that have both `A` and `B`.
* The elements that have both `A` and `C`.

The only thing that is guaranteed is that there are no entities that have `A`,
`B` and `C` at the same time in the range:

```
[group.data<A>() + group.size(), group.data<A>() + group.size<A>()]
```

This is due to the fact that all those elements are already tightly packed and
part of the group. Because we are skipping all what is contained in the group,
there is no chance we are going to iterate an entity that has the three
components assigned.

I don't know if this type of subgroups can be useful or not. It mainly depends
on the type of application being developed. I can imagine some cases in which
they might be useful, but they aren't strictly necessary most of the times nor
easy to use and therefore it's likely that you won't use them in any case.<br/>
However, it's always good to know that something is possible, in case you find
yourself having to use it sooner or later.

## A real world example

Imagine we've reproduced via ECS a scenegraph with the most classic of the
`parent` component associated with the` transform` component to handle
parent-child relationships.<br/>
At first glance, the best bet is to have a group leaded by the `parent`
component, mainly because that is the type that _makes the difference_:

```
auto group = registry.group<parent>(entt::get<transform>);
```

However, the following group already provides us with an implicit list of
components that don't have a parent and therefore it allows to have the best
performance overall:

```
auto group = registry.group<transform, parent>();
```

In fact, the group works with a _perfect SoA_ when iterating `transform` and
`parent`:

```
group.each([](auto &transform, auto &parent) {
   // ...
});
```

While something like the following allows us to quickly iterate what remains
of the list of the `transform` component:

```
const auto end = group.raw<transform>() + group.size<transform>();
const auto begin = group.raw<transform>() + group.size();

std::for_each(begin, end, [](auto &transform) {
    // ...
});
```

Just in case it's not clear, in all cases we are iterating packed arrays, one
element at a time and without jumps or indirection. Most likely we are close to
the best we can get when it comes to iterating entities and components.<br/>
In this way, we can update all the transforms giving priority to those that have
or don't have a parent, to then iterate the others without having to sacrifice
performance in any case.

Being able to order a group (as it happens in
[`EnTT`](https://github.com/skypjack/entt)) or the ability to cleverly insert
elements into a group can also allow us to iterate the transforms that have a
parent by minimizing the jumps and the cache misses, but this is beyond the
scope of this post and I won't go into further details on the subject.

## Let me know that it helped

I hope you enjoyed what you've read so far.

If you liked this post and want to say thanks, consider to star the
[GitHub project](https://github.com/skypjack/skypjack.github.io) that hosts this
blog. It's the only way you have to let me know that you appreciate my work.

Thanks.
