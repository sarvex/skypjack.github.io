---
layout: post
title: ECS back and forth
subtitle: Part 4, insights - Hierarchies and beyond
gh-repo: skypjack/entt
gh-badge: [star, follow]
tags: [ecs, entt]
---

Hierarchies aren't a nightmare anymore. No matter if all you want is to go from
the leaves to the root of a tree, to visit a fixed number of children or to
support an unconstrained model where all the entities can have an unlimited
amount of children. There exists a solution for all the cases.<br/>
The bad news are that these solutions don't give guarantees on how the children
of an entity are laid out and therefore to visit them we can incur in a few
extra jumps we didn't expect.

With this post I'll try to explore a few alternatives to face somehow this
problem and to reduce or even eliminate the jumps around in memory.

If you haven't done it yet, read the
[previous part](https://skypjack.github.io/2019-06-25-ecs-baf-part-4/) about
hierarchies before to continue. It will help to fully understand this one.

## Introduction

In the previous post, we've seen that there exists a component type that can be
used to fully support both trees and graphs and therefore hierarchies in
general.

One of the most interesting aspects of this type is that it allows to construct
an implicit list in the set of components. Because of this, we don't need to set
up and to maintain a secondary data structure to keep track of our hierarchies
nor to make use of dynamic allocations to host more and more children as the
time passes. This is particularly useful when you want to use per-type pools
because it's transparent from the point of view of the data structure.<br/>
On the other hand, there is also no guarantee that all the children are tightly
packed in memory, unless actions are taken in this regard. It can happen in fact
that nodes with different parents are interleaved.

First and foremost, note that this may **not** even be a problem. So before you
go crazy trying to order things just to save a few nanoseconds, think carefully
about whether a sorted pool is really something you need. Otherwise, move
on.<br/>
Too often too many people focus too much on non-existent problems. If you are
not among them and ordering a pool is really what you need, read on in the hope
of finding some interesting ideas.

To be able to discuss a few points with some code that is as close as possible
to a real world case, I'll use a couple of libraries available from
[`GitHub`](https://github.com/): [`EnTT`](https://github.com/skypjack/entt) and
[`Flecs`](https://github.com/SanderMertens/flecs).<br/>
The snippets in the following sections will use the terminology of these
libraries. It shouldn't be complicated to bring the examples back to any other
library that is at least designed around the same principles.

## Go further, go faster

So far, so good. We have our hierarchy and it's constructed directly within the
pool of a given component. It means that there is nothing faster than this when
we decide to iterate all the instances of that component, because they are all
tightly packed in a single array (at least with `EnTT`).<br/>
Furthermore, we have a solution that adds nothing in terms of complexity to our
container, which remains completely transparent and unaware of our work.

What's the problem then? Intuitively, the jumps aren't reduced to a minimum when
we decide to visit all the children of a given entity. This is due to the fact
that these entities (and their components) aren't necessarily close to each
other by construction.<br/>
Even worse, when we visit the entities and their components there is no
certainty that the parents are returned before their children and that a tree is
therefore visited starting from the root and down to the leaves, one level at a
time in a breadth first approach (that is what we usually want).

Fortunately, if your ECS allows sorting (and `EnTT` does it because of how
sparse sets work), we can go a bit further on this aspect and sort things in
such a way that our entities and their components are laid out in a manner that
is at least _convenient_.<br/>
There is even more though. In many cases, we can take advantage of the features
of the software we're working on and the library in use for the ECS part (if
any) to save cpu cycles and get the same result in an easier way.

### Sorting, what else?

The most obvious thing to do is to sort our components according to the `parent`
data member. Of course, you got it right, _sort_ as in _sort a pool as a
whole_.<br/>
Something along this line is already an improvement:

```
registry.sort<relationship>([&registry](const entt::entity lhs, const entt::entity rhs) {
    const auto &clhs = registry.get<relationship>(lhs);
    const auto &crhs = registry.get<relationship>(rhs);
    return crhs.parent == lhs || clhs.next == rhs
        || (!(clhs.parent == rhs || crhs.next == lhs) && (clhs.parent < crhs.parent || (clhs.parent == crhs.parent && &clhs < &crhs)));
});
```

>Discalimer: to be honest, I haven't tested this code and I wrote it straight
>away during my holidays, so I don't guarantee that it works! :)

The idea is that we are sorting things in such a way that parents and children
are both grouped and have the same order. Moreover, parents come always before
their children and therefore we don't incur in the risk of updating for example
the transform of an object the parent of which isn't updated yet.<br/>
It can seems complicated at a first glance, but it is not and gets the job done
at least.

A question arises anyway: is it really necessary to sort a whole pool?<br/>
The answer is: it depends, fortunately most of the times it is not.

Consider the case of the transform above mentioned, that is nothing more than a
way to express a scene graph in terms of components.<br/>
What we really need isn't to sort everything in order to keep the transform
components up-to-date. In fact, all we want is to update the global transforms
of the entities for which we updated the local transform. All the other entities
can remain untouched for what is worth. This can drastically reduce the number
of instances to update and the amount of work to do. Even more: this can
reduce by far the cost of sorting our elements.<br/>
In fact, this time we can also ignore the way in which things are ordered in the
pool of transforms and even be willing to pay the price of some jumps if needed.

Let's introduce another component, an empty one called `dirty`. It's a kind of
boolean value in the ECS terminology.<br/>
Every time we update a transform, we can add this component to the entity that
owns it. With `EnTT` it's a matter of attaching a couple of listeners to the
signals emitted by the registry, then use `registry::replace` to update the
transform instead of editing the components directly:

```
entt::connect<dirty>(registry.on_replace<transform>());
entt::connect<dirty>(registry.on_construct<transform>());
```

Once done and in the best case, the amount of entities we touch per frame is
only a small part of the ones still alive. What we obtain is a packed array that
contains all the entities to update. However we cannot compute it as-is, because
parent and children could have both entered this array and be positioned in such
a way that there is the risk to update a child before its parent.<br/>
So, what? So, sort. This time with an N (as in NlogN) that is by far smaller
than in the previous case. In `EnTT` terminology, we can both sort the pool of
`dirty`, then iterate it and rely on the indirection from the entity identifiers
to get the transforms:

```
registry.sort<dirty>([](auto &&...) { /* ... */ });
registry.view<dirty>().each([&registry](const auto entity) {
    const auto &instance = registry.get<transform>(entity);
    /* ... */
});
```

Or we can create a group with `dirty` and the transform component, so that we
can directly iterate the latter once the group is sorted:

```
auto group = registry.group<dirty, transform>();
group.sort([](auto &&...) { /* ... */ });
group.each([](auto &&...) { /* ... */ });
```

In both cases, when we are done we can invoke `registry.reset<dirty>()` to clear
the pool of the component we used to track _dirty entities_ and we are ready for
the next frame.

### Hierarchies and archetypes

In an archetype-based implementation components can be stored in multiple
arrays, which makes sorting much harder to do efficiently.<br/>
Fortunately this model offers other tools which allow for efficient creation of
hierarchies. One such tool is normally considered a _bad thing_, but comes in
handy here, which is _fragmentation_.<br/>
I suggest you to read
[this post](https://skypjack.github.io/2019-03-07-ecs-baf-part-2/) if it's not
clear what I'm talking about.

Long story short, fragmentation is the measure that indicates the number of
arrays a particular component is stored in. The higher the fragmentation, the
more arrays to iterate and the higher the potential for cache misses.<br/>
Fragmentation should therefore be kept as low as possible. Archetype-based
models have to deal with fragmentation, as each combination of components
results in a separate set of arrays for the components in that type.

So how does fragmentation help hierarchies?

The trick is that we can intelligently fragment to create archetypes that
exactly match the sets of entities we want to iterate as we descend the
hierarchy breadth-first. This allows us to create easily a hierarchy between
archetypes and not entities, that is something generic enough to solve many of
the most common problems anyway. We can then walk these sets in the right order
and get the job done.<br/>
This is also the approach that is taken by `Flecs`. Once again, this should
demonstrate how sometimes the _problems_ of a model aren't necessarily such, but
can be treated as features if kept under control.

Consider this snippet, which constructs a simple hierarchy:

```
ecs_entity_t parent_1 = ecs_new(world, Position);
ecs_entity_t parent_2 = ecs_new(world, Position);

ecs_entity_t child_1 = ecs_new_child(world, parent_1, Position);
ecs_entity_t child_2 = ecs_new_child(world, parent_2, Position);
ecs_entity_t grandchild = ecs_new_child(world, child_2, Position);
```

This code creates four archetypes:

```
[Position]
[Position, CHILDOF | parent_1]
[Position, CHILDOF | parent_2]
[Position, CHILDOF | child_2]
```

Note how the parents are encoded into their types. We can now create a system
that iterates all entities with `Position` like this:

```
ECS_SYSTEM(world, Transform, EcsOnUpdate, Position);
```

This system matches all the archetypes the example created, but it has one
problem: it doesn't iterate the entities in the right order. In other words,
parents and children could be interleaved and this is usually to be
avoided.<br/>
To fix that, we can change our system definition into this:

```
ECS_SYSTEM(world, Transform, EcsOnUpdate, Position, CASCADE.Position);
```

The additional `CASCADE` column will determine its depth by looking only at
parents that have a `Position` component for each matched archetype. This
results in the following depth annotations:

```
[Position] => 0
[Position, CHILDOF | parent_1] => 1
[Position, CHILDOF | parent_2] => 1
[Position, CHILDOF | child_2] => 2
```

This depth is then used to order the archetypes from low to high so entities
will be evaluated in the right order.<br/>
A nice property of this approach is that as long as no new archetypes are
created (or destroyed, if you ever decide to get rid of empty archetypes),
adding or removing entities doesn't require resorting, as the arrays from which
they are added or removed are already matched with the system.<br/>
In addition, the application still iterates a packed array for each archetype,
which is great.

This works quite well but there is one problem: in the example, we have _two_
archetypes for depth 1, where one would have been sufficient.<br/>
In an example with lots of parents the number of archetypes would go through the
roof, resulting in a very fragmented data space. To address this, `Flecs` is
implementing a new kind of storage where entities with the same components
**and** depth are combined in the same archetype.<br/>
It would take another blog post to explain exactly how it works but, seen from
above, we can summarize by saying that it reduces the fragmentation and has
therefore better performance during iterations at the price of a slightly higher
cost on adding and removing components to entities with a parent.

As we can see, archetype-based implementations need to approach hierarchies in a
very different way when compared to sparse sets, with different characteristics
and trade-offs.<br/>
Even between archetype-implementations approaches may vary a lot. It's therefore
important to understand the features and limitations of the ECS framework to
implement hierarchies in an efficient way.<br/>
Finally, note that in this case the tool is aware of the existence of
hierarchies, which are therefore no longer transparent but, on the other hand,
can be shaped on the specific problem by exploiting the underlying model.

### Almost-always almost-sorted pools

Not bad. We managed to arrange our objects in a way that can apparently favor
the subsequent iterations. We also found that sometimes we don't even need to
sort a whole pool and therefore we can highly reduce the number of operations to
get the job done.

However, when sorting a pool or an archetype as a whole is required, the cost of
this operation is all concentrated in a single frame and could give rise to
peaks.<br/>
We can use a different algorithm if we have more information about our data, as
an example the _insertion sort_, so as to exploit its features where it makes
sense. However, we still have the full cost of a sorting step and this could be
a problem in some cases.

Is it really necessary or can we divide the cost more cleverly over multiple
frames?<br/>
Once again: it depends. Fortunately, most of the times we don't care much if
things are sorted exactly or just _almost sorted_.<br/>
As an example, if all we want is to arrange things so that we can further reduce
jumps, it doesn't matter if at any point in time there is an element that is
only close to its final position but not quite there yet.

Consider the most trivial of the sorting algorithms: the bubble sort.<br/>
It's defined by an iterative approach that goes on and on until no more swaps
take place in a cycle. However, all the cycles affect to an extent the order of
the array and move things closer to their final positions. In other terms,
elements are _more sorted_ (if it made sense at all as a concept) then the cycle
before.<br/>
If you take a closer look at your favorite algorithm, it's likely that you can
spot the same pattern. This is how many sorting algorithms work after all.

Long story short, instead of a full sorting pass per frame, we could just do a
single step (or a few steps) and keep our arrays almost sorted if not fully
sorted for most of the time.<br/>
In this regard, some algorithms are better than others and the overall
complexity isn't much important because we are never going to sort the pool as a
whole. Ironically, the faster it is the single step, the better it is for our
purposes. Whatever _faster_ means in this case, of course.

### Render once and use it while it lasts

Another trick for when hierarchies are used to sort things before they are
rendered (for example in case of 2D games that work in painter mode) is that of
using _layers_.

Think for a moment to how many games are structured: they have a background and
we can easily spot different _layers_ where things are logically laid out that
are drawn one on top of the other. These layers form a hierarchy between them
and the entities within the layers form separate hierarchies in turn.<br/>
In this case, it's likely that the number of _logical_ layers is fixed and we
can define as many components as there are layers. If an entity belongs to a
layer, it has the component that _describes_ it as a parent.

First of all, this time we don't need to induce a global order. Two entities
belonging to different levels are already implicitly ordered with respect to
each other. Overall we reduce by far the cost of sorting elements and we can
split the work on multiple threads.<br/>
It could also be the case where you don't need to sort the entities that are
part of the same layer, but only between different layers. In this way, a
few additional components completely relieves us from the problem of
sorting.<br/>
That's not all, though. Often, in this type of simulation, some levels are very
_dynamic_ while others are almost static, although they may be detailed. To
further reduce the computation, we can keep track of the fact that a level has
changed, or that something inside it has moved. When this happens, we can draw
the entire layer on a backing memory and reuse the latter, with its contents,
for all subsequent frames, until it changes again. Taking this approach to the
extreme, you can even split layers into zones and apply the same technique to
different area.

In doing so and depending on the application we are working on, we may find
ourselves in the situation where, for most of the time, rendering is reduced to
drawing static images in transparency one above the other, then refining them
with what remains in the higher layer.<br/>
In any case, excluding the frames where all the layers have to be redesigned,
the workload is fairly low and we can use those cycles for something else.

## Conclusion

To sum up, there doesn't exist a one-fits-all solution when it comes to working
with hierarchies. Of course, we can try to put our problem in a solution that we
already have (because of laziness, because someone sold it to us as the solution
to all problems, or just because nothing better came to mind), but not
everything is a nail just because we have a hammer, right?<br/>
Depending on the application we are developing and the type of hierarchy we are
dealing with, we can apply one or the other optimization, reducing the workload
or moving it here and there as appropriate.<br/>
Obviously, there are many other cases and many other solutions out there, but it
wasn't the purpose of this post to mention them all and provide a user manual
for any eventuality.

Each hierarchy must be taken for what it is, separately from the others. It
should therefore be studied well and understood in detail to find its holds and
exploit its weaknesses.<br/>
Sometimes we'll need a global ordering, other times sorting a subset will be
more than enough. Some hierarchies can also be exploited in such a way that we
can find cycles not in a smaller number of jumps but in a better approach to the
problem.<br/>
And so on. Every problem has its solution.

## Thanks to

This time I want to say **thanks** to
[Sander Mertens](https://gist.github.com/SanderMertens) for contributing to this
post with the section _Hierarchies and archetypes_.<br/>
This series is in a sense the home of anyone who is trying to work on these
topics by offering new ideas and tools. I am therefore very happy and grateful
for this collaboration which I hope will not end here!

## Credits

Part of this and the previous post have been inspired by
[this](http://bitsquid.blogspot.com/2014/10/building-data-oriented-entity-system.html)
one from the [Bitsquid blog](http://bitsquid.blogspot.com/).<br/>
The aim was to bring some of the concepts to `EnTT`, then go further and try to
add some more information and ideas, so as to make it clear why the problem of
hierarchies cannot be reduced to a ready-to-eat solution in any case.

## Let me know that it helped

I hope you enjoyed what you've read so far.

If you liked this post and want to say thanks, consider to star the
[GitHub project](https://github.com/skypjack/skypjack.github.io) that hosts this
blog. It's the only way you have to let me know that you appreciate my work.

Thanks.
