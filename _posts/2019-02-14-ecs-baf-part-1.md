---
layout: post
title: ECS back and forth
subtitle: Part 1 - Introduction
gh-repo: skypjack/entt
gh-badge: [star, follow]
tags: [ecs]
---

The first time I heard of the entity component system architectural pattern, I
searched on the web for more information. Sadly, I didn't find many details on
the topic nor a single source where the different approaches were described
along with their pros and cons. Almost every article, every post, every comment
was about a (nounce of a) specific implementation and only barely referred to
the others.

In this post I'll try to introduce the topic and to explore briefly a couple of
introductory models I've seen so far. Later on I'll dive into some others so as
to give you more details and some tips for the development of your own ECS.

### Why should I use an ECS?

Don't be fooled by what is said around. Unless you are working on AAA games of a
certain level, the main reason why you should use such a tool is the code
organization and not (only) the performance. Of course, performance matter, but
a well organized codebase is invaluable sometimes and for a lot of games you
won't have performance problem even with OOP or suboptimal implementations of
component based models.<br/>
In fact, component based programming is an incredibly powerful tool to make code
flexible and to speed up iterations during development. This must be your first
goal, nothing else.

Then there are also the performances, of course. However, here we play in
another league and hardly a series of introductory posts will give you enough
details, although some of the models I'll explore are much performance oriented.

## Introduction

First of all, let's make a small introduction to explain what it means to go
from the OOP world to a component based model.

Roughly speaking, it's a matter of _partitioning_ things. Imagine to put all the
instances of your _concepts_ or domain objects on a plane. There is a dwarf, an
elf, a playing character, a rock, and so on, all next to each other. There are
two ways you can look at them or literally _cut_ the whole thing now:
horizontally and vertically.<br/>
If you cut it vertically, what you obtain are the _concepts_ themselves. In
other worlds, the typical OOP classes, nothing more, nothing less.<br/>
If you cut it horizontally, you get a bunch of meshes, some transforms, a few
finite state machines and so on. In other worlds, the _parts_ that compose your
original _concepts_, a row per type.

Where does the logic go then?

Classical OOP says that you've to iterate all the objects (instances of your
_concepts_) and rely on a sort of `update` method that _moves on_ the objects
once at a time. Following our previous example, you can walk through all the
instances and update them, as if they were columns and you work on a column at a
time.<br/>
A component based approach flips the problem on its head and iterates _parts_,
one _type_ at a time. This is what systems do at the end of the day. The
movement system takes all the instances of a transform and do its job with them,
the rendering system uses all the meshes and the transforms and draw things, and
so on. Following our previous example, it's as if you iterate things one row at
a time instead of a column at a time.

This is an extremely simplified description, but hopefully it's enough to give
you a grasp of the differences between classical OOP and component based
programming.

## ECS back and forth

Let's explore some known ways I've seen so far to implement an ECS.<br/>
In this first part of the series, I'll describe the one that is probably less
appealing but represents also a first step towards full component based models.
A slightly different implementation of this approach can be find in the great
book [Game Programing Patterns](http://gameprogrammingpatterns.com/) by Robert
Nystrom. [Here](http://gameprogrammingpatterns.com/component.html) is the link
to the web version, that is _free for you, absolutely zero cost_.<br/>
Later on, I'll try to investigate the model behind a well known ECS library that
is also largely used in many real world software.

First of all, let's dive into some implementation details that will ease the
description of the different models now.

### Unique identifiers

Almost all the possible approaches to component based programming require some
common tools and techniques. Intuitively, they also present similar problems to
some extent and these can be solved more or less in the same way in all cases.
Obviously, there doesn't exist a single way to _do things_. For simplicity, I'll
try to use some common patterns throughout all the models for those parts that
are similar, so that you can also spot easily the differences between the
different approaches.

One of the most common requirement is the necessity to have unique runtime
identifiers for components. We will see later what problem they solve.<br/>
There are plenty of techniques to generate unique identifiers. Some are well
suited to use with shared libraries, some others are not. In any case, one can
mix two or more techniques to target both cases.

A straightforward approach is to give names to components, then hash them to get
unique identifiers that can be used as keys for maps. A discussion about hashing
functions is beyond the purposes of this post, but you can find a lot of
information on the web if you're interested. This technique works fine if it
comes to working with shared libraries, even though maps are not the best tool
when performance matter. Unfortunately, identifiers aren't sequentially
generated in this case and thus they cannot be used to index arrays or vectors.

A common approach to the problem of generating sequential identifiers is the so
called _family generator_. It's a mix of templates and static stuff that gets
the job done in a few lines of code. Examples of free software that use this
technique are [`entityx`](https://github.com/alecthomas/entityx) and
[`EnTT`](https://github.com/skypjack/entt) among the others.<br/>
Here is a possible implementation of a family generator:

```cpp
class family {
    static std::size_t identifier() noexcept {
        static std::size_t value = 0;
        return value++;
    }

public:
    template<typename>
    static std::size_t type() noexcept {
        static const std::size_t value = identifier();
        return value;
    }
};
```

This is the code required to generate the identifier for a given type when
needed:

```cpp
const auto id = family::type<my_type>();
```

The drawback of this implementation is that it makes use of static local
variables and static functions. Therefore, it doesn't work well across
boundaries on some platforms. On the other side it's straightforward and quite
easy to use and to understand.

I'll use the family generator from time to time in the discussions that will
follow. If you have any doubt, take you time to go once more through its
implementation and get all the details before to continue.

### From the ground up: maps and hierarchies.

The most obvious implementation of a component based model is the one that
involves _maps_ (or sort of) and objects taken directly from the typical OOP
world. This is usually the first attempt to develop a home-made ECS and I
consider it half-way between a pure OOP approach and full component based
models.

Simply put, game objects still exist, but their types are somehow _erased_ and
contained in the sets of components that compose them. Not that it matters,
since in component based programming the types of objects are not even taken
into consideration, nor deduced in some ways and for some specific purposes.

How does it works exactly?

A trivial approach that _works_ and gets the job done is to reduce your game
object to a map of components. Systems will iterate all the objects and test
them to check if they have the set of components required before to do their
work. A bitset or similar can also be used to speed up the lookup phase
sometimes.<br/>
If you prefer not to use maps, components can be put in a vector. With a family
generator, the unique identifier of a type represents also its position within
the vector itself. No need to search things each time then. On the other side,
if you use maps the unique identifier can be used to generate keys for the
types to store.<br/>
A graceful API is probably all what you need to get it up and running now. If
you imagine to put the game objects in a vector, a system might iterate all of
them and ask to each object if it owns the desired components before to proceed.
In other terms:

```cpp
void my_nth_system(std::vector<game_objects> &objects) {
    for(auto &&object: objects) {
        if(object.has<position, velocity>()) {
            auto &pos = object.get<position>();
            auto &vel = object.get<velocity>();

            // ...
        }
    }
}
```

The pros of this implementation are few and quite obvious: straightforward to
implement and to maintain, it's easy to understand for those coming from the OOP
world. Moreover, it's quite close to the typical list of game objects we have
with OOP programming.<br/>
And that's all. In a sense, this can be even worse in terms of performance if
compared to classical OOP. However, take it for what it represents: a step
towards other models, not something to use in real world code.

This approach has also a lot of cons.<br/>
First of all, components are scattered around in memory and you've multiple
jumps each and every time you access them. However, the biggest issue is the
fact that you don't know at any time what are the game objects that match a
given query and thus you must iterate all of them in each system. It goes
without saying that this is far from being welcome in terms of performance. In
fact, it could quickly ruin performance for medium sized games, when the number
of objects grows up.

Something that I didn't mention and that is important instead is that _concepts_
are no longer part of our codebase. Did you spot it? There doesn't exist anymore
an _elf_ class or an instance of a _playing character_. We have only plain game
objects filled with components of different types.<br/>
Sounds interesting, doesn't it? Even though this solution is half-way between
OOP and fully component based models, we can already appreciate some of the
benefits of using components.<br/>
Keep reading to find out what are the others.

### Towards an objects-free solution: entities are indexes

The next step towards a full component based model is intuitively to get rid of
game objects.<br/>
In the previous example, game objects were nothing more than containers for
components. Our game object was defined as a class with a thin interface to
easily manipulate its parts. Components were stored in maps by game objects and
every game object had its own set of components. It should be quite easy to get
rid of these wrappers and change a bit the layout, so that components of a same
type are stored together. This is also the next step along the path.

First of all, we need to give a _name_ to our game objects and we know that
naming things is one of the hardest jobs in computer programming. Because we put
game objects in a vector previously, we could use the index of an object as its
name. Therefore the first object is named 0, the second one is named 1, and so
on. Let's call them _entities_ instead of _names_ and we have that the first
group of components is assigned to entity 0 while the second set of components
is  assigned to entity 1.<br/>
Did you spot the pattern in it? If we flip the problem on its head and put
components in packed arrays, one per type, we have that at position 0 of each
array lay the components for entity 0, at position 1 of each array lay the
components for entity 1, and so on. More in general, given an entity N and M
separated arrays of components, we can use the entity itself as an index to
access its data in the different vectors.

We don't have anymore game objects. Instead, we have entities as numbers and
instances of components laid out each in its own array. Entity identifiers are
also the indexes to use to retrieve instances of components. This is more or
less the idea behind [`entityx`](https://github.com/alecthomas/entityx), a quite
known ECS library in C++.

Usually, entities in this model are assigned also a bitset to keep track of what
components are attached to them. This speeds up the lookup and acts as a
workaround for another problem. Consider the case in which component `C` is
assigned to entity `N` > 0. In this case, the vector for `C` is resized so as to
accommodate at least `N` components. However, we never assigned `C` to entity
`0`, even though an instance exists in memory now.<br/>
How do we know then if the component is valid or not? Bitsets seem to be the
common answer of those that like to use this approach.

One of the pros of this model is that iterations are pretty fast, even more if
bitsets are used to speed up lookups. It's also quite intuively and
straightforward to implement and works pretty well for small games. Entity
identifiers can be easily recycled to reduce memory consumptions in case.

Unfortunately, this approach has also some cons.<br/>
The most evident is the fact that we waste **a lot** of memory when arrays are
sparse and there is no easy way to avoid it apparently. This has also a less
evident drawback, that is the high number of cache misses due to the fact that
we pull from the memory also the _holes_ that live side by side with valid
components. Another cons is that we still don't know what entities own what
components and therefore we have to iterate all of them in each system to test
if they match the requirements. The use of bitsets can mitigate this aspect, but
it won't eliminate the problem completely.

## What's next?

I hope this brief introduction helped to start a journey from the OOP world
towards component based models. What we have seen so far is well suited for
small- up to medium-sized games, but it doesn't fit well when the number of
entities and components grow up and performance really matter.

In the next chapters of the series I'll try to explore other approaches more
performance oriented. In particular, a model based on sparse sets and another
one that relies on grouping things as much as possible.

## Let me know that it helped

I hope you enjoyed what you've read so far.

If you liked this post and want to say thanks, consider to star the
[GitHub project](https://github.com/skypjack/skypjack.github.io) that hosts this
blog. It's the only way you have to let me know that you appreciate my work.

Thanks.
