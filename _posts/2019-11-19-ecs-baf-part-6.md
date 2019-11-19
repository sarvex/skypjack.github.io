---
layout: post
title: ECS back and forth
subtitle: Part 6 - Nested groups and perfect SoA to the rescue
g5-repo: skypjack/entt
tags: [ecs, entt]
---

With [this](https://skypjack.github.io/2019-03-21-ecs-baf-part-2-insights/) post
I described what is probably one of the most interesting features of the sparse
set based model.<br/>
The grouping functionality makes possible what is called _perfect SoA_, that is
something we can achieve **only** with this approach among the ones described by
the series. As we have seen, that's theoretically the best we can obtain in
terms of performance during iterations because we have all the entities and
their components tightly packed and ordered the same. No jumps, no branches,
nothing on which to waste our cpu cycles.

If it sounds good, you've probably also heard me say that great powers come with
some limitations. In particular, the fact that a type can belong only to one
group and therefore we must define carefully our groups to get the best out from
them.<br/>
This contraint can be relaxed in fact and we can have perfect SoA in many more
cases than expected.

If you haven't done it yet, read the
[previous parts](https://skypjack.github.io/tags/#ecs) of the series before to
continue. It will help to fully understand this one.

## Introduction

Long story short, _groups_ are a way to organize the elements within the packed
array(s) of two or more sparse sets so that the items that belong to all sets
and match some given requirements (the most common of which is that they are
assigned to the same entity) are tightly packed and ordered the same at the
beginning of their respective arrays.<br/>
This is extremely powerful because iterations happen on a fixed number of items
with no branches, no jumps, nothing at all. They are just plain iterations from
0 to N and they are performed on thightly packed _pieces of memory_. We know for
sure that the i-th element of each array is assigned to the same entity, no
matter what.

However, it goes without saying that we cannot force more than one order within
a given array. Therefore groups are _limited_. A given type of component can
belong only to one group and therefore can only be optimized for that group. In
all other cases we'll access it through indirection because of how sparse sets
work.<br/>
It would be great if we could create both groups `<sprite, renderable>` and
`<sprite, renderable, position>`. They are _almost identical_, it's a shame we
cannot reuse the first group to somehow _construct_ the second.

What if I said you that a given type of component **could** belong to more than
one group then? There are some rules to follow but it may work.<br/>
This would mean more and more usage patterns matched and therefore maximum
performance on more and more paths, not only the most critical ones.

## A quick recap

From the
[insights](https://skypjack.github.io/2019-03-21-ecs-baf-part-2-insights/) on
the grouping functionality of the sparse sets, we know how groups work. They are
a tool that trades more performance during iterations with the invocation of a
callback or such during the creation and destruction of components.<br/>
Granted, you've not to implement it necessarily by means of some callbacks but
this is how it works in [`EnTT`](https://github.com/skypjack/entt). You know:
_don't call us, we will call you_. That's it. Here is our trade-off for the
grouping functionality. It's not something one can easily observe in a real
world case and it's still faster than many of the other solutions when it comes
to adding or removing components but it's something to take in consideration
anyway.

A _running_ group is like the following:

```
A packed    : [e4|e7|e3|e8|e6]
A component : [A4|A1|A0|A2|A3]
     group _________^

B packed    : [e4|e7|e5]
B component : [B0|B2|B1]
     group _________^
```

In other terms, it behaves like a _pointer_ to the first element after the group
itself. By construction (read the article linked above for the details) we know
then that all elements are tightly packed and ordered the same.

When a new entity enters the group, all the components assigned to it are moved
to the position pointed by the group and the latter is then _incremented_. When
an entity leaves the group, it's swapped with the last item present in the group
itself and the latter is then _decremented_.

So far, so good. In a sense, the grouping functionality induces an _order_ in
the packed arrays of the sparse sets for the types of components it _owns_.
Moreover, it's known that we cannot sort an array in two different ways at the
same time.<br/>
Here we are thus with our limitations. One component, one group. Nothing more
and nothing less.

## Nested groups

The trick to overcome the _one-component-belongs-to-one-group_ limit is in the
fact that we aren't inducing an _order_ at all. Quite the opposite actually. The
order is a nice-to-have consequence of the way we construct and maintain a
group. However, roughly speaking, ordered items aren't a requirement to be able
to properly define a group.<br/>
Of course, I'm not saying we could renounce to the order because it would make
groups pointless. However, for the purpose of this post, let's ignore it for a
while and look at what remains to us: a separation in two parts of the original
array. This is what defines a group. After all, it's only a pointer to the
element immediately after the ones in the group itsef, right?

If you're wondering how this can help, look at the two partitions as if they
were two arrays. Nothing prevents us from splitting them in two parts and these
parts in two parts and so on until necessary.

Consider now the example above. It describes a group `<A, B>` that contains
already entities `e4` and `e7`:

```
A packed    : [e4|e7|e3|e8|e6]
A component : [A4|A1|A0|A2|A3]
     group _________^

B packed    : [e4|e7|e5]
B component : [B0|B2|B1]
     group _________^
```

The _parts_ left out don't have much to share with each other, so we cannot
really _divide_ them for our purposes. However, they are also the entities in
which we weren't interested initially, so who cares anyway.<br/>
On the other side, the parts of the arrays that _contain_ the groups are
identical in terms of entities: `e4` occupies position 0 and `e7` occupies
position 1 in both the arrays. We know also that all the elements that belong to
an entity will move together here and there when needed by construction.

Imagine to add another group `<A, B, C>` that subsumes the one already created.
It's _strictier_ in a sense, it sets more constraints to the entities that want
to enter it. Furthermore, we can say already that `<A, B>` will contain at least
the elements of `<A, B, C>`. So, `<A, B, C>` defines two partitions within
`<A, B>`: the entities that have also `C` and the ones that have not it, if any:

```
A packed    : [e4|e7|e3|e8|e6]
A component : [A4|A1|A0|A2|A3]
  <A, B, C> _____^
  <A, B> ___________^

B packed    : [e4|e7|e5]
B component : [B0|B2|B1]
  <A, B, C> _____^
  <A, B> ___________^

C packed    : [e4|e8|e5]
C component : [C2|C0|C1]
  <A, B, C> _____^
```

The example is trivial but it should give you a grasp of what I mean. We can
define as many nested groups as we want. The new condition is that two nested
groups, that is two groups that _own_ the same type(s) of component(s), must be
such that one literally _contains_ the other.<br/>
_Contains_ here means that the list of types of components of one group
_extends_ the list of the other. The largest group (as in the group that owns
    the highest number of types of components) also contains at least all the
elements of the smallest.

Once defined, nested groups are then identical to _normal_ groups. In case we
wanted to iterate `<A, B>`, we could get the packed arrays for the components
`A` and `B` and iterate them from 0 to _size of `<A, B>`_ (whatever it is). To
iterate `<A, B, C>` instead, we just add a third packed array to the list and we
use _size of `<A, B, C>`_ as our `N` in this case. We don't have to know
anything about `<A, B>`.

An example of two conflicting groups that should be rejected is `<A, B>` and
`<A, C>`. In this case, none of them _extends_ the other. Therefore, it will be
impossible to create multiple nested partitions within the arrays of entities,
components or whatever your arrays contain.

### The order matters

I bet it's not immediately evident what the _algorithm_ to make an entity enter
a deep nested group is however. I made this error and what I'm to say is also
the reason for which order of execution on callbacks is guaranteed now in
`EnTT`.<br/>
Unfortunately, we cannot just say - _if the entity has `A`, `B` and `C`, let's
swap it with the element pointed by this group_.

Consider the following example:

```
A packed    : [e4|e7|e3|e8|e6]
A component : [A4|A1|A0|A2|A3]
  <A, B, C> __^
  <A, B> ___________^

B packed    : [e4|e7|e5]
B component : [B0|B2|B1]
  <A, B, C> __^
  <A, B> ___________^

C packed    : [e6|e8|e5]
C component : [C2|C0|C1]
  <A, B, C> __^
```

The entity `e8` has both `A` and `C`, so it's not part of any of our groups.
However, it's only far a `B` from both of them. Let's add this component and
_just swap `e8` with the element pointed to by `<A, B, C>`_:

```
A packed    : [e8|e7|e3|e4|e6]
A component : [A2|A1|A0|A4|A3]
  <A, B, C> _____^
  <A, B> ___________^

B packed    : [e8|e7|e5|e4]
B component : [B3|B2|B1|B0]
  <A, B, C> _____^
  <A, B> ___________^

C packed    : [e8|e6|e5]
C component : [C0|C2|C1]
  <A, B, C> _____^
```

No good: `e8` entered both groups as expected but `e4` exited `<A, B>` even
though it has both `A` and `B`. Our groups are broken now and there is no way we
can easily recover.

This happened because _order of execution_ matters in this case. Rules are
pretty simple but we cannot escape them:

* When we add a component to an entity, _test-and-swap_ groups from the largest
  to the smallest.
* When we remove a component from an entity, _test-and-swap_ groups from the
  smallest to the largest.

In the example above, it would mean to make `e8` enter firstly `<A, B>` and only
**then** `<A, B, C>`.

Why should this solve the issue though?

In **all** cases, the element pointed by `<A, B, C>` is either the same pointed
by `<A, B>` or one within this group. Because of this, swapping the element
pointed by `<A, B, C>` with the one we want to move into the group means
swapping two elements within `<A, B>` if it already contained the entity and
thus no element will unexpectedly leave `<A, B>`.<br/>
The same is true in case of destruction of components but the order to follow is
the opposite this time, for similar reasons.

### Perfect SoA to the rescue... what else?

As it was for the basic grouping functionality, nested groups come with a price.
The good news are that it's exactly the same as before. The more groups _own_ a
given type of component, the more this will affect construction and destruction
of its instances.<br/>
On the other side, the more you are willing to pay for a given type of
component, the higher the number of critical paths on which it will have the
best performance ever.

Note that I use the term _pay_ to make it clear that nothing is for free in this
field but I strongly suggest you to measure, measure, measure to have an exact
idea of what the price is at the end of the day.<br/>
It's unlikely you'll ever note it unless you're writing some crappy benchmarks
that creates and destroys 1M entities at once or so repeatedly.

Furthermore, fortunately the benefits of the groups aren't limited only to the
fact that they are _faster_.<br/>
Also in this case you'll observe the difference only when you decide to push
your software beyond its limits. I even suggest users not to use groups with
`EnTT` during the prototypying phase and for large part of the development,
until they know what are their most critical paths and where it makes sense to
use groups or not.<br/>
However, groups provide the users with a `(T*/size)` couple that fits with many
uses other than iterations. Serialization? Networking? Raw copies? Just use your
imagination and I'm pretty sure you'll find many other examples. Because of
this, having the possibility to define _more groups_ as nested groups is a great
plus

## Let me know that it helped

I hope you enjoyed what you've read so far.

If you liked this post and want to say thanks, consider to star the
[GitHub project](https://github.com/skypjack/skypjack.github.io) that hosts this
blog. It's the only way you have to let me know.

Thanks.
