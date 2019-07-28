---
layout: post
title: ECS back and forth
subtitle: Part 3 - Why you don't need to store deleted entities
tags: [ecs, entt]
---

When it comes to working with the ECS architectural pattern, it's common to
destroy entities sooner or later. In this case, it's likely that entity
identifiers are recycled for several reasons, then returned the next time a new
entity is created.<br/>
Unfortunately, most of the time this part is poorly designed and results in a
waste of memory and unwanted allocations.

I want to explain why you don't need to store aside deleted entities and how we
can avoid to consume memory for them. This is also the same technique I've used
in [`EnTT`](https://github.com/skypjack/entt).

If you haven't done it yet, read the
[previous parts](https://skypjack.github.io/) of the series before to continue.
It will help to fully understand this one.

## Introduction

No matter what's your implementation for component based design, recycling
entities is quite useful in all cases. Sometimes it's even required to avoid
problems. At least it seems to be so with the most known models I ran across in
my experience.

### Recycling is useful

Recycling entities is useful because it gives us a way to find if a known
instance is still alive, though it isn't the only way to do that. There exist
implementations where an entity is considered _deleted_ or _dead_ when it has no
more components assigned. However, as we will see in the next section, this can
quickly lead to problems that are instead avoided if the entity is recycled.

How do we recycle an entity then? Quite easy.<br/>
Let's consider 32bit identifiers for entities. We can split them in two parts
(for the sake of simplicity 16bit each), then assign the first part to the
actual entity and the second part to the _version_ or _generation_ or whatever
is the name you want to use for that:

![identifier](https://user-images.githubusercontent.com/1812216/57165441-65931b00-6df7-11e9-91b6-e9ebdad11c0d.png)

Identifiers become thus opaque values to return to the users and from which we
can extract both the parts when received.

When a new entity is created, we just look if there exists an identifier that
has been previously destroyed to return. Otherwise, we create a new identifier
with the right value for the entity and with the version set to 0.<br/>
On the other hand, when an entity is destroyed we can increase its version and
keep track of it for later uses.

This has the benefit that users can easily store aside identifiers and know at
any time if they are still valid or if the entities to which they refer had been
deleted in the meantime. If the version of the stored identifier differs from
the current one, it means that the entity has been recycled.

### Recycling is required

The major implementations of an ECS of which I'm aware are:

* Component matrix (as an example
  [`entityx`](https://github.com/alecthomas/entityx)): here entities are used to
  index the matrix itself. Not recycling them would be a huge waste of memory.

* Sparse sets (as as an example [`EnTT`](https://github.com/skypjack/entt)):
  here entities are put in sparse arrays to allow fast random access aside
  perfect SoA. Not recycling them would be a huge waste of memory (this can be
  mitigated by paginating the sparse array but still it's not that convenient).

* Archetypes (as an example [`decs`](https://github.com/vblanco20-1/decs)): here
  entities are stored aside with a direct pointer to the archetype to which they
  belong. Recycling them permits to reduce the memory usage for the vector
  of entities.

Moreover, if we split our identifiers in two parts, the number of entities we
can generate is limited. If we consider 32bit identifiers and use only 16bit for
the entity, it's likely that we'll go out of identifiers after a while if we
don't recycle them. We can mitigate the problem by using 20bit for the entity,
but  still it's a matter of time.<br/>
Another option is that of using 64bit identifiers but they occupy twice the
space of their counterparts for no real benefits but for the fact that probably
we have no longer to recycle them. We can also combine bigger identifiers with a
version-less model and decide that an entity is destroyed when it is assigned no
components. Now we have to find a way to make it explicit and it's a bit
counterintuitive to me to be honest. It sounds odd and has its own can of
problems anyway.

From my point of view, smaller identifiers are worth it if all what I've to do
is to find a smart way to recycle them. Memory occupancy isn't my favorite pet
and the drawbacks of not recycling entities are annoying, so why shouldn't I
care of it?

## The entity and the version

_So, I've decided to recycle my identifiers, you convinced me. What should I do
now?_

The first approach is the most obvious one. I've used it for long time in
[`EnTT`](https://github.com/skypjack/entt) and I must say that it worked pretty
well.<br/>
Just use two vectors and that's all:

```
std::vector<std::uint32_t> entities;
std::vector<std::uint32_t> destroyed;
```

Its problem is that it occupies (not so much) memory for literally nothing and
pulls in extra allocations we can avoid. In other terms, if we generate N
entities, then destroy all the identifiers, it occupies twice the space required
for them because of how an `std::vector` works (and because it's likely we are
going to reuse them soon, so we don't want to shrink the `entities`
vector).<br/>
Moreover, it could incurr in extra cache misses during some operations. However,
it's biggest problem is that you cannot say at a first glance if an entity has
been destroyed and not recycled yet. You've either to keep the two instances of
an identifier in sync or to search through the `destroyed` vector.<br/>
On the other hand, it's straightforward to implement and to work with, that are
usually good features.

As a side note, [`EnTT`](https://github.com/skypjack/entt) used to keep in sync
the instances of the indentifiers in the two vectors. It's not that difficult,
but it has a cost that is better to avoid.

A variant would be that of using something like this:

```
std::vector<std::pair<std::uint32_t, bool>> entities;
```

It consumes less memory and we know immediately if an entity is still alive or
not. In this case, the problem is that we have no clue about what's the _next_
entity to recycle and we have to search it somehow. A side data structure to
keep track of deleted entities would mitigate the problem, but it defeats a bit
the purpose of the `bool` value and duplicates the information in different
places.

_So far, so good. What's the alternative then?_

The other way around is to get rid of the second vector and to construct an
implicit list of destroyed entities directly within `entities`. This way we can
know immediately if an entity has been destroyed (because it belongs to the
list, but keep calm and continue to read to know why this is true), we avoid
allocations and reduce memory usage, plus another set of benefits I'll mention
below.<br/>
It sounds pretty good. This is what we need for that:

```
std::vector<std::uint32_t> entities;
std::size_t available{};
std::uint32_t next{};
```

Where:

* `entities` is the list of entities, both alive and dead ones.
* `available` indicates how many entities have been destroyed so far and not
  recycled yet.
* `next` tells us what's the _next_ entity to recycle.

If you take a closer look at it, you can spot a common pattern. I've named it
`next` for the sake of clarity, but it's nothing more than the head of the
implicit list we are going to construct. If you have ever worked with linked
lists in C/C++, you know almost certainly what I'm speaking about.

## From explicit to implicit

Defining an implicit list is quite simple indeed. It's just a matter of
_shifting_ the list of deleted entities back of a position.

![basic](https://user-images.githubusercontent.com/1812216/57167272-7c3c7080-6dfd-11e9-8f8a-c1f7737063d1.png)

The `next` variable contains the first identifier to recycle. We exploit the
fact that the entity (that is, the identifier masked to get rid of the version)
represents also its position within the vector of entities. At the position
pointed out by the identifier stored in `next` we can put then the _next_ entity
to recycle and so on, until the end of the implicit list. The last element of
the list can contain whatever you want, its value isn't going to be used any
time soon (thanks to `available`).

In case an entity is destroyed, we increase its version and swap the identifier
with `next`. This way it's automatically added to the list of deleted entities:

![destroy](https://user-images.githubusercontent.com/1812216/57167633-bc502300-6dfe-11e9-975a-cd03afda6d55.png)

Do not forget to increase `available` too, otherwise you've to walk through the
whole list of deleted entities to know how many identifiers have been deleted so
far.<br/>
This is particularly useful to check if there are entities still in use. When
`available` is equal to the size of `entities`, it means that the container is
empty.

To recycle an entity, we can swap `next` with the value contained at the
position pointed by the identifier stored in `next`, then return the latter.
Also in this case, do not forget to decrease `available` to avoid problems.

Increase the version and swap to destroy, swap and return to recycle. This is
trivial and pretty cheap. Definitely something that is worth it if you consider
the benefits. However, unfortunately this solution has also a **huge** problem
that isn't obvious at a first glance.<br/>
As I mentioned above, we can use the version to know if an entity has been
destroyed. However, in this case, the position associated with an identifier no
longer contains it if it's destroyed and has not been recycled yet. Even worse,
if we consider how the implicit list works, we know that the identifier is
stored in the _previous node_ of the list that isn't a double linked one.
Therefore, we have no way to retrieve it easily.

Fortunately, there exists a solution also for this problem.

### Split entity and version

So far, to make an identifier part of the implicit list of destroyed entities
or to recycle it, we simply updated its version and swapped it with the value
contained in `next`.<br/>
However, what is really needed to create the list isn't the whole identifier.
In fact, the entity part is more than enough because that's what gives us the
position of the identifier itself within `entities`. Therefore, we can leave the
version in its original position, where we need to have it at the end of the
day. In other terms, we can exploit the fact that the version isn't required to
create the implicit list and shift **only** the entity part of the identifiers,
**not** the whole value.

![version](https://user-images.githubusercontent.com/1812216/57168765-8bbeb800-6e03-11e9-9060-9b5fa5b567b9.png)

Note that, in the image above, the version is that associated with the entity if
not explicitly specified. As an example, `E0` can also be read as `E0|V0`.<br/>
This requires you to work with bitwise operations, do some shifts and make your
hands dirty with a couple of bitmasks, but solves the problem of having always
the version where we search it.

What we obtain is:

* If an entity is still alive, the identifier stored in the position pointed by
  the entity is the _right one_, that is a 32bit value composed by the entity
  itself and its version.
* If an entity has been destroyed, the identifier stored in the position pointed
  by the entity is a composed value that contains the version associated with
  the entity itself and the value of the next entity in the implicit list of
  destroyed entities.

If it seems complicated, I can assure you that it is not (but for the first time
you run across it probably). On the other side, it's cheap and all the
operations are trivial.

For the sake of curiosity, this is more or less how
[`EnTT`](https://github.com/skypjack/entt) works under the hood. The `registry`
class doesn't contain a list of destroyed entities. Instead, it constructs an
implicit list directly within the list of entities. It makes also use of these
_mixed identifiers_ generated as a result of shifting only the entity part of
the identifiers.<br/>
You can have a look at the code of the `registry` class to see an actual
implementation of this concepts.

## Let me know that it helped

I hope you enjoyed what you've read so far.

If you liked this post and want to say thanks, consider to star the
[GitHub project](https://github.com/skypjack/skypjack.github.io) that hosts this
blog. It's the only way you have to let me know.

Thanks.
