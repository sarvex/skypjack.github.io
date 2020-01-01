---
layout: post
title: ECS back and forth
subtitle: Part 4 - Hierarchies
gh-repo: skypjack/entt
gh-badge: [star, follow]
tags: [ecs, entt]
---

Hierarchies, both a blessing and a curse for the entity-component-system
architectural pattern.<br/>
One of the first questions that anyone makes when starting to work with an ECS
is how to represent hierarchies in this model without ruining the performance.
There are several approaches to the problem and what's the best one depends
mainly on the real problem we are facing.

The first thing to solve is usually how to express a hierarchy, however complex,
in terms of components. The second is how to go through it in a way that is
efficient and that doesn't require us to jump around in memory or to increase
the fragmentation, that is something that defeats a bit the purpose of using an
ECS.

If you haven't done it yet, read the
[previous parts](https://skypjack.github.io/) of the series before to continue.
It will help to fully understand this one.

## Introduction

In this case more than in others, a solution as generic as perfect doesn't
exist. Or at least, there is none that is both flexible and capable of not
affecting the performance of our software to an extent.<br/>
In particular, it's always better to follow the _less is better_ principle to
avoid problems and therefore to implement only what's necessary for our case.

In the following sections I'll explore some common approaches to the problem. In
a future post I'll try to describe some techniques to keep our pools
almost-always almost-sorted, something aimed to reduce even further the jumps
when walking through a hierarchy.

For the sake of simplicity, I'll consider only the case of pure trees. However,
it's quite simple to extend the same concepts to the case of a graph, if
needed.<br/>
Moreover, to be able to present the solutions with some code that is as close as
possible to a real world case, I'll use
[`EnTT`](https://github.com/skypjack/entt) as an ECS and I'll propose snippets
that will use the terminology of this library. It shouldn't be complicated to
bring these examples back to any other library.

## Hierarchies to the rescue

If all you want is to move from an entity to its parent, it would be a waste to
implement support for exploring children as well. Similarly, if you know a given
entity won't have more than N children any time soon, it wouldn't make sense to
implement a dynamic model where you can add a theoretically unlimited amount of
children. What if you know nothing about the maximum number of children instead?

Sometimes things just work out-of-the-box with a single component, some others
we get straight to the point but we can do better in terms of performance with
multiple components.<br/>
Let's see what are the types that can help in these cases and what we can do to
get the best from them.

### Where is my parent

The most basic problem is that in which one needs only to visit a hierarchy from
the leaves towards the root and therefore from the children towards their
parents.<br/>
A trivial example is that in which we want to adjust the position of an entity
based on that of the parent in a 2D game.

In this case, the solution is straightforward and not particularly exciting. To
do that, we can keep track of the relationship with a component assigned to the
child entity.<br/>
In fact, a type in all respects similar to the following is enough:

```
struct relationship {
    entt::entity parent{entt::null};
    // ... other data members ...
};
```

It's not necessary to initialize the component with the _null entity_,
especially if your framework doesn't offer it. However, it can be useful for
deferred assignments and in few other cases.<br/>
Whenever you want to go back to the parent from a given entity, you can query
the above component:

```
if(auto *comp = registry.try_get<relationship>(entity); relationship && relationship->parent != entt::null) {
    // ...
}
```

Obviously, this is the let's say _secure_ version of the query. In the vast
majority of cases, it will even be possible to get rid of the `if` (for example,
by using a
[group](https://github.com/skypjack/entt/wiki/Crash-Course:-entity-component-system#groups)
in `EnTT` and therefore by processing the entities that have a parent separately
from the others), thus greatly increasing performance.

### N-children policy

Things get more interesting when we need to move through the hierarchy the other
way around, that is from a parent to its children. However, there is a special
case that still allows to solve the problem in a very simple way.<br/>
This happens when the maximum number of children an entity can ever have is
known in advance.

If you think that this is rather uncommon, remember that your playing character
most likely has two hands and therefore two weapons in use, plus perhaps another
pair in the holsters, for a total of four weapons available.<br/>
There is little to no difference if your hero also has a bag with him, since
this too will probably contain a finite and known number of objects anyway.

Also in this case, the solution is straightforward. It will be enough to update
the `relationship` component as follows:

```
struct relationship {
    std::size_t size;
    std::array<entt::entity, N> children;
    entt::entity parent{entt::null};
    // ... other data members ...
};
```

We can even split it in two components if required:

```
struct parent {
    entt::entity entity{entt::null};
    // ... other data members ...
};

struct children {
std::size_t size{};
    std::array<entt::entity, N> entity{};
    // ... other data members ...
};
```

The latter is probably better when all you want is to keep track of the weapons
assigned to a character, but it isn't necessarily the best choice in every
case.<br/>
However, if the number of accesses to children is much less than the number of
times we want to follow the hierarchy towards the root, dividing the whole thing
into two components can bring benefits. It's a sort of hot vs cold data pattern
after all. If it's worth it mainly depends on the use case.

In both cases, it's important to update the `size` data member of the parent
whenever a child is created or removed. This shouldn't be much difficult anyway,
because we know always who's the parent when we are to construct a child. It
wouldn't be possible to initialize the `parent` data member (or struct)
otherwise and therefore we couldn't go back from a leaf to its root.

To iterate the hierarchy we follow the exact same pattern shown above. On the
other hand, iterating all the children is a matter of using a `for` loop:

```
auto &comp = registry.get<children>(parent);

for(std::size_t i{}; i < comp.size; ++i) {
    const auto entity = comp.entity[i];
    // ...
}
```

Pretty simple indeed but quite limited a solution. What if the number of
children isn't known in advance instead?

### Unconstrained model

The _unconstrained model_ requires a slightly more complex type to work
properly:

```
struct relationship {
    std::size_t children{};
    entt::entity first{entt::null};
    entt::entity prev{entt::null};
    entt::entity next{entt::null};
    entt::entity parent{entt::null};
    // ... other data members ...
};
```

Again, nothing prevents you from splitting the type in multiple classes if you
find that it fits better your real world case.<br/>
Moreover, the component above is good enough for most of the possible cases.
However, it doesn't mean that we cannot solve our specific issue with something
smaller than this. This component is just fine to show what's needed in the most
complex case.

Let's see what are its fields for:

* `children`: the number of children for the given entity.
* `first`: the entity identifier of the first child, if any.
* `prev`: the previous sibling in the list of children for the parent.
* `next`: the next sibling in the list of children for the parent.
* `parent`: the entity identifier of the parent, if any.

Note that the `children` data member isn't strictly necessary, but knowing the
number of them can be useful sometimes and helps also to get rid of some `if`s
when we iterate the list of children. In fact, children can be found starting
from the `first` entity and then traveled by visiting either `prev` or `next`
from the `relationship` component of it.<br/>
The two data members `prev` and `next` are used to create an implicit double
linked list of entities. Both the previous and the next sibling are required to
be able to easily remove elements in the middle of the list. In case this isn't
required, the sole `next` data member is enough (but I don't know of an use case
where removing entities from the middle isn't an option).<br/>
As usual, `parent` is the hook to the parent of an entity and is used to walk
through a hierarchy from the leaves to the root.<br/>
In order to avoid problems when visiting the children of an entity starting from
an element in the center of the list, the implicit list must be a circular one.
However, this isn't required in all the other cases.

One of the benefits of this approach is that we have no dynamic allocations nor
the need for reallocation to store the children of an entity or to add new
elements to the implicit list. However, there is also no guarantee that all the
children are tightly packed in memory, unless actions are taken in this regard.

To iterate all the children of an entity, this is the way to go:

```
auto &comp = registry.get<relationship>(parent);
auto curr = comp.first_child;

for(std::size_t i{}; i < comp.children; ++i) {
    // ...
    curr = registry.get<relationship>(curr).next;
}
```

Actually we can do better in terms of performance by exploting some nice tricks
of `EnTT`, but this isn't the goal of the post and I won't go in details on
this.<br/>
As you can see, all we need is to get the first child of an entity and start
visiting the list of children (the size of which is known) one sibling at a
time. In case we don't want to use and keep up-to-date the `children` data
member, it's still possible to iterate the same entities as:

```
auto &comp = registry.get<relationship>(parent);
auto curr = comp.first_child;

while(curr != entt::null) {
    // ...
    curr = registry.get<relationship>(curr).next;
}
```

Note that in the snippet above I also made the assumption that almost all the
entities (at least the ones interested by the hierarchy) have the `relationship`
component. If this isn't the case, the `try_get` member function of the
`registry` class represents a valid alternative to probe and get the component
at once if it exists.

## What's next?

The full-featured component shown above is pretty good in most cases, if not all
of them. However, it has an intrinsic problem: there is no guarantee that
children of an entity are close to each other in memory and therefore iterating
them could result in some extra jumps to pay for. Something similar applies also
to the way parents are stored.

In the second part of this post I'll try to explore some ideas to reduce or even
eliminate the impact due to the way things are laid out when it comes to working
with hierarchies.<br/>
In fact, there are some techniques that can be very useful for the purpose. From
the most obvious, that is a full sorting, to the almost-always almost-ordered
approach, up to the use of `EnTT` groups to partition our pools by construction
so as to get the best out of them.

## Let me know that it helped

I hope you enjoyed what you've read so far.

If you liked this post and want to say thanks, consider to star the
[GitHub project](https://github.com/skypjack/skypjack.github.io) that hosts this
blog. It's the only way you have to let me know that you appreciate my work.

Thanks.
