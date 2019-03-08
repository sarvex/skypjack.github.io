---
layout: post
title: ECS back and forth
subtitle: Part 2 - Where are my entities?
tags: [ecs, entt]
---

Second part of the _ECS back and forth_ series. After a half-way solution
between OOP and component-based models and an approach where entities were also
indexes, I want to introduce another couple of techniques that try to get rid of
some common problems seen so far. In particular, we have still to find a way
that gives us **all** the entities and **only** the entities that have a given
set of components.

Later on in the series, I'll go probably in depth into one of the models
presented, the one I know best and on which I can give more implementation
details.<br/>
Fortunately, more or less all the solutions share many aspects with each other.
It will therefore be possible to reuse most of the techniques and suggestions
while trying to implement your own tool.

If you haven't done it yet, read the
[first part](https://skypjack.github.io/2019-02-14-ecs-baf-part-1/) of the
series before to continue.

## Introduction

In our journey from OOP towards component-based models we left out a problem
that is still unsolved. In particular, both the implementations discussed in the
previous part require systems to iterate all the entities to know what are the
ones with the desired set of components. There is apparently no way to avoid
it.<br/>
However, as it happens ofter in computer science, an extra level of indirection
can solve the problem. This is due mainly to the fact that it helps to decouple
the client code from the data structures where entities and components are
stored and this allows to lay out the objects to ease the search or to fill the
_holes_, if any.

As a side note, the extra level of indirection is more powerful than this. In
fact, we can do much more than rearranging components and fill holes. I'll try
to give more details in a future post, but this is beyond the purposes of this
one. Because of that, in the next sections I'll concentrate only on how to get
packed arrays for components and entities and to leave holes out of the way.

## Archetypes

A model very popular lately, at least since Unity has declared to have
implemented a version of it, is that of **archetypes**. They were previously
known also as **prototypes**, but it seems that _archetypes_ is now the common
term used for this approach.

The basic concept is rather simple, even if implementing an efficient version of
it isn't trivial at all.<br/>
The idea behind this approach can be summarized as follows: if an entity has a
particular set of components, take the pool (also known as _archetype_) for the
entities that have that same set (if it doesn't already exist, create it) and
assign the entity and all its components to that pool. Whenever you add/remove
a component to/from an entity, pick up everything again and move the entity and
all its components from a pool to the other, from an archetype to the other.

The implementation details for this aren't that easy, but there exists a good
project on GitHub called [`decs`](https://github.com/vblanco20-1/decs) that
offers an experimental version of an archetype based model in C++.<br/>
Another implementation of this design I'm aware of is
[`reflecs`](https://github.com/SanderMertens/reflecs), although this time it's
about a C project and not C++. I'm not a fan of _C-ish_ interfaces (I wrote
[`uvw`](https://github.com/skypjack/uvw) to avoid working directly with `libuv`
after all). However, this project is particularly interesting, since the author
has tried to go beyond the standard features usually offered by component based
models and added things such as the ability to make entities literally
_children_ of other entities as if they were components. I can't say if features
like these can be useful in everyday use, personally I've never felt the need,
but they are certainly interesting and make `reflecs` more intriguing than other
projects.

So, how does archetypes solve the problem of finding all the entities that have
certain components?<br/>
An archetype based solution no longer requires you to iterate all the entities
and test them to know if they have the desired components. Instead, this time
we iterate all the archetypes (that are likely to be much less than the
entities), then we return all the entities from the archetypes that are built
for a set of components that contains **at least** the desired ones (but not
necessarily **only** the desired ones). Most likely there will be more than one
archetype to visit, but you are guaranteed that all the entities in an archetype
have at least the components of interest if the archetype matches the query and
therefore you don't have to test the entities as well. This results in a huge
speed up at the end of the day.<br/>
Another benefit is that the archetype contains both the entities and the
components and the latter will most likely be organized in packed arrays all
ordered in the same way. Therefore, visiting an archetype and returning what it
contains results in an ordered visit of some arrays.

One of the advantages of this approach that isn't obvious at all initially is
that it fits particularly well with **multithreading**. In general, component
based models are often designed to exploit also this aspect. However, archetypes
allows to do it in an even easier way, if possibile.<br/>
If you consider that the entities that have at least some components are likely
to be scattered around in different archetypes, you can imagine to elaborate the
archetypes separately. If you then put objects in different blocks of the same
size within an archetype, you can have even a better balance by directly
distributing those blocks to the various threads. Consider that archetypes could
be highly unbalanced, therefore the first solution could result in suboptimal
performance and defeat a bit the purpose of using multiple threads.<br/>
In both cases, multithreading becomes trivial to implement and to reason of,
what to elaborate and how to distribute the load derives directly from how the
data are laid out under the hood. Moreover, the way instances are stored can
easily reduce to a minimum false sharing and improve overall performance.

Unfortunately, also archetypes have their own drawbacks as with any other
solution.

Intuitively, the first _problem_ to face derives from the way they work. Every
time a component is added or removed, an entity and all its components are moved
from an archetype to another one. This affects to an extent the construction and
destruction of components. The problem can be mitigated with a bulk add, but one
cannot really get rid of it if instances of components are assigned and removed
on the fly.<br/>
Because of that, this model works like a charm when the sets of components
assigned to the entities don't change much during the runtime. This is for
example true in low level systems, why it isn't necessarily the case with high
level systems. A possible solution to reduce the impact of this problem is to
use this model to manage almost-fixed types of components (eg it's unlikely that
instances of the `physics` component are removed from their entities once
assigned and this makes `physics` a good candidate among the others) while
relying on a secondary data structure to deal with those components that are
frequently assigned or removed during the runtime.

However, the main issue you can encounter with archetypes is the
**fragmentation**.<br/>
Shortly, an aspect that is less evident but still present is the fact that
archetypes aren't primarily designed around usage patterns. This technique
simply tries to optimize everything as much as possible, since it is known that
within the _everything_ there are also our usage patterns. However, this has a
drawback, that is that you lose the chance to get even more on the archetypes
(let me say) _of interest_ and sacrifice something to optimize things for
iterations you won't ever perform. Even worse, the number of archetypes could
explode if you have a high number of possible combinations of components
assigned to different entities at runtime and this will definitely affect the
iterations to an extent by adding more and more jumps to find all the entities.

Finally, there is something that can and cannot be a problem. If it is depends
mainly on your point of view, but it's worth mentioning.<br/>
In particular, entities and components are all stored in separate blocks and
it's not possible anymore to get a couple `(T *, size)` for a given type to
iterate or to `memcpy` everything at once. For the same reasons, using custom
allocators could give us slightly lower benefits than with other solutions that
combine all components into a single array.

That being said, archetypes play in another league when compared to the
solutions seen so far and can be safely considered one of the best possible
approaches to the problem.<br/>
Do not forget that the perfect solution doesn't exist and each has pros and
cons. In this case, if we consider it as a whole, it's definitely worth it.

## Sparse sets

Sparse sets flip the problem on its head and approach it in a way that is
completely different from that of archetypes. I've not seen many implementations
of component based models that are based on this data structure, probably
because it can be used in so many different ways that is difficult to find a
sort of _guideline_. Therefore newcomers are probably discouraged.

Before to describe how sparse sets can be used to address the problem of knowing
what are **all** the entities and **only** the entities that own a specific set
of components, let's see what the sparse sets are and how they work.

### They called them packed arrays

A **sparse set** is such that it's:

>[...] a clever data structure for storing sparse sets of integers on the range
>`0 .. u−1` and performing initialization, lookup, and insertion is time `O(1)`
>and iteration in `O(n)`, where `n` is the number of elements in the set.

At least according with
[this](https://programmingpraxis.com/2012/03/09/sparse-sets/) interesting
article that explains clearly how they work.<br/>
You can find blog posts and examples of use of sparse sets/packed arrays all
around the web if you know what to search. As an example, the BitSquid blog
contains [one post](http://bitsquid.blogspot.com/2011/09/managing-decoupling-part-4-id-lookup.html)
about it. Similarly, Molecular Matters tries to give a few details on the topic
in [a post](https://blog.molecular-matters.com/2013/07/24/adventures-in-data-oriented-design-part-3c-external-references/)
of its _adventures in data oriented design_ series.

Personally, I don't want to go in depth of the argument this time. I kindly
suggest to read the articles above to have a grasp of how sparse sets work
before to continue.<br/>
In short, this data structure contains two arrays: the former is a **sparse
array** (also known as _external_ or _reverse_ array) while the latter is a
**packed array** (also known as _internal_ or _direct_ array), hence the name
often used to refer to the sparse sets. To know if the sparse set contains a
value, as an example an entity idenfier, you do the following test:

```cpp
const bool match = (packed[sparse[entity]] == entity);
```

If `match` is true, than the sparse set contains the given value.<br/>
Fortunately, the indirection isn't required when you want to iterate all the
values contained by the sparse set. It's suffice to walk through the packed
array for that, from the first element to the last one. This dual access mode is
what makes them both fast and flexible.

In general, all the implementations I've seen so far differ slightly from the
theoretical version. Sparse sets allow for several optimizations if you know
your data. Therefore, you can easily get rid of indirections most of the time or
store _augmented values_ in one or two of the arrays. As an example of this, the
[implementation](https://github.com/skypjack/entt/blob/master/src/entt/entity/sparse_set.hpp)
proposed by [EnTT](https://github.com/skypjack/entt) is quite different from the
classical one and hardly anyone would say it's a sparse set without knowing the
details and being able to understand the differences.

### Systems own sparse sets

A possible use of sparse sets is to match entities against systems. An open
source project in C++ from GitHub that uses sparse sets for that is called
[`ecst`](https://github.com/SuperV1234/ecst).

The basic idea is that a sparse set is instantiated for each and every system.
The purpose is to make it contain all and only the entities to which the system
itself is interested and these depend strictly on the components it wants to
iterate at runtime. Systems are then registered with a sort of central manager
that executes them and manages entities and components. During the registration
phase, a system _declares_ what are the components of interest. Its sparse set
will be kept up-to-date after each iteration.<br/>
When a system wants to iterate its entities, it reads its packed array and use
the entities thus obtained to get the components from the manager. The way the
manager actually stores the components is out of scope in this case.

As you can guess, iterating entities is incredibly fast. They are all contained
in a packed array and the number of cache misses is reduced to a minimum.
Similarly, keeping a system up-to-date has also a very low cost, thanks to the
properties of the sparse set.<br/>
On the other side, getting components in this model could incur in a performance
penalty. If it's so mostly depends on how things are laid out by the central
manager. This is a double-edged sword: a clever organization can really increase
performance, while a less clever approach can definitely ruin everything.
However, unlike other solutions, data can be organized on a per-component basis,
choosing the best from time to time according to the usage patterns.

A variation on the topic exists and relies on a different ownership model to
speed up the whole thing. I've read more than once of this, so it's worth
mentioning, even though I must admit that the first time I've read of it I felt
like it doesn't fit well with real world cases. Nowadays I've changed my mind
and I firmly believe that not only can it work but it's also an excellent
approach, if you can develop it correctly.<br/>
The idea is quite simple in fact: define your systems so that most if not all of
them _own_ one or more types of components and thus their instances. Owned types
don't necessarily have to be all those iterated by a system, of course. One can
easily understand that a type can be owned at most by a system and referred to
by the others in some way. However, it should be easy to define which system
intuitively owns which component: as a rule of thumb, the one that wants to
_write_ it is the best candidate. As you can imagine, a central manager for
components is no longer required, because their instances are managed directly
by systems.

How does this approach improve the one described above?

Consider that a sparse set now contains not only entities in a packed array, but
also instances of components. If a system wants to iterate the components it
owns, it does it by iterating one or more packed arrays all sorted the same.
Again, cache misses are reduced to a minimum, this time not for entities only
but for entities and components.<br/>
The _problem_ (that isn't a problem at all in fact) arises when a system wants
to iterate also the components owned by one or more of the other systems. In
this case, the entity identifiers can be used to access those items through
indirection in the other systems, as it happens usually with sparse sets. In
other terms, the entity identifiers are used to access the external arrays of
the other systems and to retrieve the indexes where the instances of the
components are stored in the internal arrays.<br/>
As long as the number of not owned components is kept low, the advantages due to
the use of full packed arrays for owned components should overcome the time
spent on indirections and the overall performance should be incredibly good.
Moreover, it has the advantage that data can be organized on a per-component or
per-system basis to exploit usage patterns if possible, that is something you
cannot really achieve with other models like archetypes.

Something similar to the last approach described above is used also by
[`EnTT`](https://github.com/skypjack/entt) for its grouping functionality. Of
course, the implementation differs quite a lot from what I wrote, mainly because
systems don't own components in its case. However, the principle on which groups
rely is similar and they exploit a combination of packed arrays and optionally
indirection to improve performance when needed.<br/>
The way groups work is beyond the purposes of this section. Some additional
details can be found in the next section, but I'll leave a thorough analysis of
the topic for a future post.

### Independent sparse sets

The other way around to use sparse sets to implement a component based model is
to rely on them as the key data structure for pools of components. This time,
the systems aren't involved and the optimizations find room elsewhere.<br/>
Now it should be clear how the sparse sets work, so I'll give you perhaps less
details and go straight to the point where possible.

To use a sparse set as a pool is straightforward, at least if you don't aim to
optimize it. The basic implementation already works fine for that. You can just
add a third array ordered as the internal one and add or remove items from it
every time you do that with the packed array.<br/>
This gives us a very simple way to know if an entity has a certain component or
not: just use the entity to index the sparse array and the position thus
obtained will also be that of the instance in the packed array of components if
a link exists. Moreover, iterating all the instances of one type of component is
the fastest thing you can imagine, because all your components are tightly
packed in a single array, with no holes in it. If you use entity identifiers
as indexes for the sparse array and because of how sparse sets work, the packed
array of indices will contain in fact identifiers and therefore also iterating
the entities assigned to the given component should be fast as hell.<br/>
Note that entities and components will be ordered in the same way in the last
case, so iterations are reduced to the ordered visit of a pair of arrays.

The problems arise when you want to iterate multiple components, mainly because
pools are completely independent from each other. There exist several approaches
to the problem, either trivial and slightly slower or more complex but faster:

* The most obvious one is probably more than enough for most of the games and
  applications out there.<br/>
  To iterate components `A` and `B`, pick both the pools required, then find
  what's the _shortest one_, that is the one that contains fewer entities. It's
  likely that it contains by far less entities than the other pool and this will
  save **a lot** of time during iterations. Once the pool to be iterated is
  decided, begin to visit the entities one at a time and check if the other pool
  contains them. If so, the entity can be returned to the caller with its
  components, otherwise skip it and check the next one.<br/>
  It goes without saying that with this method we don't know a priori all the
  entities that have the desired components. Moreover, the indirection required
  to verify if the entity has all the components slightly degrades the
  performance, which however remain very good and definitely sufficient for the
  vast majority of cases.

* A less obvious and more tricky solution is that of creating an additional
  sparse set to track the entities that have a given set of components. Whenever
  the component set of an entity changes so that it owns at least the components
  of interest, the entity is added to this additional sparse set. When one of
  the components is deleted, the entity that owns it is removed from the pool if
  present. In general, at runtime, the sparse set will contain **all** and
  **only** the entities that have at least the required components.<br/>
  This approach obviously increases performance in a lot of cases. However, it
  also suffers from the indirection required to access the components when the
  entities contained in the sparse set are iterated. The performance hit due to
  this aspect can be mitigated in some ways but, from personal experience, I
  find that none of the possible methods to do it is actually worth it. On the
  other hand, when you don't want to access all the required components despite
  being part of the query, this approach gives you the freedom to choose where
  to spend your cpu cycles.

* The third and last alternative is perhaps the least obvious, but also the most
  appealing and is in theory the best in terms of performance, although it has
  some limitations. That's also the one implemented to an extent by the grouping
  functionality of [`EnTT`](https://github.com/). I'll try to describe it
  briefly, without going into too many details<br/>
  Consider you want to iterate components `position` and `velocity`. Because of
  the way sparse sets work, you know components are all tightly packed in two
  arrays. Both of them contain some entities that have both the components and
  some others that have only one of the components. If you can arrange things
  so that all the entities that have both the components are at the top of the
  arrays while all the others are at the end, as a result the components will
  also be arranged accordingly. Iterations will benefit indecently from how
  things are laid out in this case, because all the entities that have both the
  components and the components themselves are tightly packed and sort in the
  same way at the beginning of their arrays. Therefore, during the iterations no
  test is required, nor any indirection.<br/>
  **Performance are theoretically the best** you can get, because the system
  knows now exactly what are the entities and the components to return and have
  them all tightly packed in the same order. It's like having archetypes, but
  without the _jumps_ between different blocks to pay for. The downside is that
  only one order can be imposed within an array. Therefore, if a type of
  component is affected by multiple iterations, you'll have to choose for which
  you want maximum performance and for which you are willing to pay the
  indirection.

There are probably other solutions that I'm not yet aware of, but these are the
ones I've seen so far and/or tried for my own and on which I can guarantee both
in terms of performance and functionalities.<br/>
Reach me if you know any other variation on the topic that can be interesting.

The variety of available solutions gives us a perhaps not obvious
advantage.<br/>
Consider that a software is usually composed of critical and non-critical
paths. In this case, each of the approaches has advantages and disadvantages and
increases the performance on one or the other feature (eg iterations or
component construction/destruction). Therefore, one could choose to use
different methods for different paths, so as to maximize the benefits depending
on the case.<br/>
Obviously this requires that you know very well the software you are working on,
but this is another story.

### Final notes on sparse sets

What can be seen as the main problem with sparse sets, at least if compared to
the previous solution, is that it is not _automatic_. While archetypes are
implicitly generated for the users, this time we need to explicitly decide what
to optimize. On the other hand, however, it's true that you've the possibility
to choose and shape things on your **usage patterns**, that could be seen as an
advantage instead.<br/>
Another consequence of this is that one can decide where to spend cpu cycles,
opting for solutions with slightly lower performance during iterations but with
less impact on other functionalities when needed.

One aspect that may seem interesting about this approach is the fact that it is
definitely easier to set **per-component allocators**. Not that it isn't a thing
for everyone, nor a widespread desire, but it is something to keep in mind if it
were among the requirements.<br/>
Of course, even with archetypes you can manage to do it, but it is slightly more
inconvenient to do and results probably in a suboptimal solution due to the
fragmentation.

**Multithreading** is easy to do even in this case, although less trivial than
with archetypes. Intuitively, it will be enough to break the arrays to be
iterated in several parts, as many as the processes to run, then assign a
portion of data to each of the threads.

## So, archetypes or sparse sets?

There is **no answer** to this question, at least from my point of view. These
two solutions, for my experience, **play in the same league** and allow to reach
performance well above expectations if implemented properly, both with their
pros and their cons.<br/>
There are advantages and disadvantages in both cases and each solution offers
features that are simply impossible to obtain for the other. Therefore, probably
the choice should be more due to a matter of taste than to the performance of
the individual approach.

I personally believe that, at these levels, any performance problems will be
difficult to ascribe to the time spent on iterations in both cases. Most likely,
**bottlenecks will be elsewhere**.<br/>
Therefore, I don't suggest either one or the other solution. I obviously made my
choice, but I don't think this is better or worse than the other. I think rather
that the features offered by one of the two approaches are closer to my way of
thinking and writing code and, therefore, I prefer that solution.

**It's a matter of taste**, in fact.

## Let me know that it helped

I hope you enjoyed what you've read so far.

If you liked this post and want to say thanks, consider to star the
[GitHub project](https://github.com/skypjack/skypjack.github.io) that hosts this
blog. It's the only way you have to let me know.

Thanks.
