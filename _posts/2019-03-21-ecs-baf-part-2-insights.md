---
layout: post
title: ECS back and forth
subtitle: Part 2, insights - Sparse sets and grouping functionalities
gh-repo: skypjack/entt
gh-badge: [star, follow]
tags: [ecs, entt]
---

After publishing [part 2](https://skypjack.github.io/2019-03-07-ecs-baf-part-2/)
of the **ECS back and forth** series (if you haven't read it yet, do it now), I
received several requests for further information on sparse sets. In particular,
requests for more details on the **grouping functionalities** and on the support
they can offer to different access patterns (sequential, random, ...).

The grouping functionalities are a topic that can often be found online in the
lines of posts or comments. To be honest, I've struggled to put the pieces
together in the past and this is why I think it's useful to collect all the
information I gathered so far in this post.<br/>
I'm pretty sure that out there, somewhere, there's another me going down the
same road and I hope this can help him.

## Introduction

As a wise boy once said in a chat where I was by chance:

>50% of time you loop components, 50% you access them random, 50% is other
>access pattern.

Therefore, supporting a single access pattern would be limiting and that's why
there are techniques for sparse sets to allow users to decide where and how they
want to access things.<br/>
Be aware that this doesn't mean sparse sets are a one-fits-all solution to use
for all your needs in terms of access patterns. However, as long as they support
it elegantly, there is no reason not to use them and to squeeze the best from a
sparse set.

Ironically, when you take grouping functionalities to the extreme (that is what
[`EnTT`](https://github.com/skypjack/entt) does with its _full-owning groups_),
this technique offers also a way to do **perfect SoA** (citing the words of
another smart person who has contributed largely to this idea).<br/>
In the following I will try to explain what `EnTT` offers with its
[groups](https://github.com/skypjack/entt/wiki/Crash-Course:-entity-component-system#groups),
how they are implemented and why these are available in different flavors (that
is _full-_, _partial-_ and _non-owning_ groups), each with different features.

## A quick recap

From [part 2](https://skypjack.github.io/2019-03-07-ecs-baf-part-2/) we know how
a sparse set works. The most trivial implementation when used as a pool for
components is such that it is composed by three vectors:

* The **sparse array**, also known as reverse array.
* The **packed array**, also known as dense or direct array.
* A **component array** sorted the same of the packed array.

The implementation will be in charge to keep the packed array and the component
array in sync (eg whenever it swaps two elements in the packed array, it
**must** swap the elements at the same positions also in the component array).

So far, so good. Sequential access during iterations is blazing fast for obvious
reasons. Random access is fast as well thanks to the properties of the sparse
set.<br/>
This gives use the best solution ever to iterate a single component and a very
good approach for random access, insertion and deletion in general.

When it comes to iterate multiple components at once, that is multiple sparse
sets, a pretty fast approach is that of getting the _shortest one_ (the sparse
set whose packed array is shorter), then probe the entities before to return
them to the caller to test if they have assigned all the components.<br/>
The two main problems are that:

* The two sets aren't necessarily sorted the same. In fact, they could be
  ordered completely differently from each other. Sparse sets allow users to
  sort the items they contain and sort them in the order dictated by another
  sparse set is pretty fast if you know _the trick_ to use to do that. However,
  this doesn't give us the necessary guarantees anyway, although it helps to
  increase the performance reducing the cache misses.

* We don't know how many entities have both the components. What we know is that
  they cannot be more than the size of the shortest packed array, that's all.
  In most cases it's already enough to improve indecently the performance
  (imagine you've 200k entities still alive but only 100 of them have assigned
  one of the two components), but the feel is that we can do **more**.

This isn't as good as it could be at a first glance, right? To be honest, it's
good enough for the majority of cases, I wouldn't care much of it. This approach
is already by far faster than most of the implementations out there and has some
nice-to-have features that make it a really good solution.<br/>
However, what if we have a **very** critical path and we **do** want **more** on
sequential access and iterations in general? Is it possible?<br/>
The answer is obviously **yes**. That's what I've called **groups** (just
because I've no idea if a name already exists and this seemed appropriate).

## Groups

The idea behind the groups is straightforward. Imagine you have two sparse sets,
one for the component `A` and one for the component `B`. They already contain
some entities, none of which have both the components. A group for `(A, B)` is
empty at this point:

```
A packed    : [e3|e7|e8|e6]
A component : [A0|A1|A2|A3]
     group ___^

B packed    : [e4|e5]
B component : [B0|B1]
     group ___^
```

I omitted the sparse array, because it's pointless for the purpose of the
discussion.<br/>
As you can see, the group _points_ (whatever it means to _point_ for a group) to
the first element of the array. We can look at it as if the group pointed past
the last element contained, that is no elements at all in this case. Another way
of looking at it is as if it were pointing to the first available position to
use to add more elements to the group.

Let's add `B` to entity `e7` and see how it works. When you do that, you're
guaranteed that `e7` isn't part of the group yet, mainly because it didn't have
`B` before.<br/>
Immediately after adding `B` to entity `e7`, the sparse sets will appear as
follows:

```
A packed    : [e3|e7|e8|e6]
A component : [A0|A1|A2|A3]
     group ___^

B packed    : [e4|e5|e7]
B component : [B0|B1|B2]
     group ___^      ^___ assign B to e7
```

All what is needed now is to literally _move_ the entity `e7` within the
group.<br/>
To do that, we can _swap_ the entity after the one pointed by the group with the
one to which we've just attached the component, then advance the group. This
must be done for both the sparse sets:

```
A packed    : [e7|e3|e8|e6] <-- swap e7 with e3
A component : [A1|A0|A2|A3]
     group ______^

B packed    : [e7|e5|e4] <-- swap e7 with e4
B component : [B2|B1|B0]
     group ______^
```

What if we add `A` to the entity `e4` then? That's easy, same operations and the
group keeps growing up:

```
A packed    : [e7|e4|e8|e6|e3]
A component : [A1|A4|A2|A3|A0]
     group _________^

B packed    : [e7|e4|e5]
B component : [B2|B0|B1]
     group _________^
```

By construction, the sparse sets contain the entities that are part of the group
at the top of their packed arrays, all ordered the same. We will see shortly
what are the benefits due to this aspect.<br/>
However, it would be nice if this continued to be true even after removing items
from a group. Fortunately, it's easy to achieve. Whenever we remove a component
from an entity that belongs to a group, we know that it's _before_ the one
pointed out by the group itself. This is by construction. To update our implicit
list of entities and components of interest, it's enough to swap the entity that
is exiting the group with the last one contained in it, then move the group
(that is, it's _pointer_, whatever it is) back of one step.<br/>
As an example, let's imagine we want to remove component `B` from entity `e7`.
Before to do that, we swap it with entity `e4`:

```
A packed    : [e4|e7|e3|e8|e6] <-- swap e7 with e4
A component : [A4|A1|A0|A2|A3]
     group _________^

B packed    : [e4|e7|e5]  <-- swap e7 with e4
B component : [B0|B2|B1]
     group _________^
```

Then we update the group so that it points to entity `e7` (remember, a group
_points_ to the element past the last one that belongs to the group itself):

```
A packed    : [e4|e7|e3|e8|e6] <-- reduce the size of the group
A component : [A4|A1|A0|A2|A3]
     group ______^

B packed    : [e4|e7|e5] <-- reduce the size of the group
B component : [B0|B2|B1]
     group ______^
```

Finally, we can easily remove `B` from entity `e7`. It means usually that we
want to swap the entity with the last one in the sparse set before to pop out
it. This is the best way to keep the internal arrays packed. The result is:

```
A packed    : [e4|e7|e3|e8|e6]
A component : [A4|A1|A0|A2|A3]
     group ______^

B packed    : [e4|e5] <-- swap e7 with e5, then pop out e7 and its component
B component : [B0|B1]
     group ______^
```

That's all. As we assign a component to an entity or remove it, it potentially
becomes part of a group or exits it. Consequently the tail of the group is
lengthened and is shortened depending on the case.<br/>
Within our sparse set we'll still have all the entities and components tightly
packed for blazing fast iterations. Furthermore, the sparse sets involved by the
group will be such that the first N elements of their packed arrays also contain
the entities and components of interest, ordered the same. This is kind of
**the best** we can obtain in terms of performance, because we have in fact N
arrays of the same size and all ordered the same. All what we have to do is to
iterate them from the first to the last element and return everything as-is to
the caller. No jumps, no indirection, no test of validity, just pick the
elements and return them. That's all.

Obviously, there is a price to pay for this, as for everything else. In this
case, the creation and destruction of components is affected by a group to an
extent. Nothing that you'll really notice in real world applications probably,
mainly because construction and destruction aren't usually done along critical
paths and it's unlikely you're to construct or destroy 1M entities or components
every tick. Note also that this price is paid **only** when entities enter or
exit a group, not all the times they are added or removed a component, like it
happens with some other solutions (you've read
[part 2](https://skypjack.github.io/2019-03-07-ecs-baf-part-2/) of this series,
right?). Therefore, the overall cost will be in general low in all cases. On the
other side, as long as the choice of where to pay the price for what we want
remains ours, it is up to us to decide and there is nothing better.

Only one thing remains to be clarified: why does
[`EnTT`](https://github.com/skypjack/entt) then offer different
[types of groups](https://github.com/skypjack/entt/wiki/Crash-Course:-entity-component-system#groups)?<br/>
It's a matter of **ownership**. As some will have guessed, a type of component
cannot belong to more than one group, because it's not possible to induce more
than one sub-set in an array (this isn't properly _true_, but let's say that
it's a good approximation if you don't want to ruin the performance and make
things more complex than what is needed). However, the possible optimizations
with this tool don't stop here and this is why `EnTT` offers more than one type
of groups.

### Full-owning groups

Full-owning groups are such that they literally _own_ all the pools of their
types of components. Therefore, they can create subsets within the packed arrays
as described before.<br/>
This is the fastest _group_ (as in fast to iterate entities and components) you
can construct. For N types of components, we have N packed arrays, each of which
we know to contain M elements at the top all ordered in the same way. All what
is needed to iterate them is to start from the first element and return
instances to the caller, a tuple at a time for M times. Just advance a bunch of
pointers and return what you obtain. Cache misses are intuitively reduced to a
minimum.

### Partial-owning groups

What if I want to define two groups, one for components `A` and `B`, the other
one for components `B` and `C`?<br/>
Ownership of `B` cannot be _shared_ between two groups, so are we hopeless here?
Not yet in fact, we can still give a performance boost to our iterations.

Let our first group take ownership of components `A` and `B` and therefore be a
full-owning group. The second group will take then the ownership of `C` and will
use `B` as a **free-type**, that is a type of component that takes part in the
definition of the group but in whose packed array we cannot create a
subset.

How does this work? It's simple. Whenever an entity enters or exits the group,
we arrange things as shown previously, this time only in the sparse set for
`C`.<br/>
Iterations will benefit quite a lot from this. First of all, the extent of the
group tells us the exact number of entities to return. Even more, we know
exactly what these entities are, because the sparse set of `C` has them at the
top of its array, ordered the same of their components. This is already a huge
boost in terms of performance. To get `B`, instead, we will have to pay the
price of the indirection, fortunately very low due to the properties of the
sparse sets. However, we don't have to test anymore entities to know if they
have `B` because we **know** that they have it, being them in the group. We can
then go directly to the component and return it.

### Non-owning groups

The logical consequence is the case in which we have three groups: one for
components `A` and `B`, one for components `B` and `C` and one for components
`A` and `C`.<br/>
Oh oh. the first one is a full-owning group. The second one is a partial-owning
group. Right. What about the third one instead?

This is less frequent a case, but still possible. This time, we _track_ only the
entities that deserve to belong to the group, even though we cannot create
subsets in already existing arrays. Instead, we use an additional sparse set to
keep the identifiers packed in an array to iterate later.<br/>
We don't have the benefits of having all or some of the components ordered the
same, but at least we know how many entities are in a group and we don't have
to test them before to get the components. Therefore, we can go directly to the
instances. We have to pay the price of indirection for that, of course, but the
properties of the sparse sets help to keep this price low.

Some further optimizations exist for this case tha can be applied also to the
partial-owning groups. As an example, one can cache the indexes used during the
first iteration to avoid the indirection later on in absence of changes.<br/>
In my experience, since we have full freedom of decision, one tends to use
full-owning groups where the performance matter and non-owning groups (if
necessary) along less critical paths or where the iterations aren't the
bottleneck. Therefore, these optimizations tend only to complicate the whole
thing and to increase memory usage without real benefits.

## Free as in free to choose

Sparse sets allow for different solutions to improve sequential access and
therefore iterations, from the blazing fast full-owning and partial-owning
groups to the slightly slower non-owning groups. This without considering that
the model without groups already offers more than sufficient performance for the
vast majority of cases.<br/>
A **full-owning** group is probably **the best** we can obtain in terms of
performance with a bunch of sparse sets used as pools: packed arrays all ordered
the same and of which the size is known. On the other hand, **indirection**
pulls in a price to pay for **partial-owning** and **non-owning** groups. It's
due mainly to the fact that shared ownership isn't possible with groups.

One of the most interesting aspects is that groups can be created on the basis
of our **usage patterns** and the ownership model can be chosen depending on
whether a query is made on a critical path or not. In fact, paths often exist
along which bottlenecks aren't in iterations on entities and components and, in
these cases, one can freely decide not to use a group.<br/>
This leaves us **full control** unlike other more automated solutions, but also
shifts the **responsibility** to make the _right choice_ on the users, which is
not something for everyone.

## Let me know that it helped

I hope you enjoyed what you've read so far.

If you liked this post and want to say thanks, consider to star the
[GitHub project](https://github.com/skypjack/skypjack.github.io) that hosts this
blog. It's the only way you have to let me know that you appreciate my work.

Thanks.
