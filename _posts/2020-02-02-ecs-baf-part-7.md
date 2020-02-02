---
layout: post
title: ECS back and forth
subtitle: Part 7 - Shared data
gh-repo: skypjack/entt
gh-badge: [star, follow]
tags: [ecs, entt]
---

This time I want to answer a question that has been asked me several times on
[gitter](https://gitter.im/skypjack/entt) or directly through issues open to
[`EnTT`](https://github.com/skypjack/entt).<br/>
Given that sharing data can be a concise and elegant solution to some problems,
I'm not a fan of the built-in solutions offered sometimes for this purpose. Long
story short: there is not a single model that solves all the problems and often
the predefined solutions lack this or that facet that would make them suitable
for my purposes.<br/>
This is why I decided not to offer a built-in yet incomplete solution in `EnTT`.
The library allows multiple entities to share data but doesn't sponsor a sharing
model in particular by design.<br/>
This doesn't mean that built-in solutions are wrong regardless or that they
shouldn't be used. There are notable cases of
[engines](https://docs.unity3d.com/Packages/com.unity.entities@0.1/manual/shared_component_data.html)
or [libraries](https://github.com/SanderMertens/flecs) that offer out-of-the-box
solutions for a specific problem and that work as long as you don't try to apply
them where you shouldn't.<br/>
In `EnTT` it's rather a design choice. I prefer to implement what I need on top
of what the library offers rather than making it part of it. The reason will
probably be clear later. It's mainly due to the fact that it already supports
some models transparently while it would be a forcing to try to support others
just for the sake of having them.

I'll get the chance to explain what these models are and how it's possible to
create your own solution for shared components in a few lines of code.<br/>
I've also invited a friend of mine to contribute to this post and to describe
the model he implemented because I found it really valuable.

# Introduction

Personally, I find very few use cases for shared data. The first one that comes
to my mind is about resources, since in no case it makes sense to have multiple
copies of them. Other more or less meaningful examples I can make are about high
level systems and are usually very specific or highly related to the application
under development.<br/>
Speaking of games, you can imagine for example a horde of enemies that share the
same goal. How long does this remain shared though? Is there a _main_ entity
making decisions while the others follow their leader? Or can the individual
entities leave the _group_ by picking up a new goal spontaneously?<br/>
If instead we shared a component that describes the _race_ associated with one
or more enemies, this could last over time. However we have to define who owns
the component and eventually what happens if this _entity_ dies and therefore is
recycled to represent something else.

A user of `EnTT` once said:

>I've found myself wanting shared components, only to realise that it
>would have introduced additional problem that would be harder to solve.

This is true in part. Shared data aren't for free and come often with their own
can of problems to solve. However, they may be worth it if used properly.<br/>
Let's try to explore some models that could make sense to an extent and to
understand **when** shared data are the _right_ solution to use.

## Flyweight

I called this model _flyweight_ since the basic idea is the same behind the
well known design pattern.

To explain this, consider the problem of managing resources.<br/>
First of all, one should avoid duplicating them for a lot of reasons such as
memory consumption. On the other side, it doesn't make much sense to store
resources directly within the _registry_ (in the `EnTT` terminology). It's
mainly a matter of separation of concerns from my point of view. Managing
resources means prealoding them, trying to perform prediction, doing a lazy
unload of elements and so on. In short, all operations of which the registry
shouldn't even be aware.<br/>
Imagine now to have a predictor that triggers the preload of a given resource.
If we decided to put resources directly within the registry, we would have to
_bind_ the resource to an entity. However, it isn't _in use_ yet and therefore
we would need also to design a way to _hide_ our entity from the systems that
could intercept it otherwise. Then we need a way to make it alive when needed
other than to _look for_ a specific resource from hidden and visible entities
since resources are _runtime things_ and we want to share them.<br/>
It can quickly become pretty confusing and hard to work with and to maintain.

With a dedicated class (or subsystem) aimed at managing our resources instead,
all we need is a way to _retrieve_ a given element from the components. Let's
call it _handle_. It can be either an index into an array of resources or a more
structured class like a pointer associated to a reference counter, it doesn't
matter here. What is important is that all the components that own a copy of the
handle are also _sharing_ the same resource.<br/>
This helps to reduce the memory consumption and to separate those that _consume_
the resources from who's in charge of _managing_ them, that is definitely a
nice-to-have feature.

### Chunk level reference

A _chunk based design_ can go a bit further on this.<br/>
The idea is that of storing entities and components in chunks rather than in
plain arrays. To put it simply, imagine an array of _pages_ (or _chunks_) each
of which can contain a maximum of N entities and their components. It's slightly
more complicated than that but this is enough for our purpose.<br/>
At the time I'm writing there are two implementations of which I'm aware that
combine archetypes and chunks: the one offered by the Unity engine and the one
implemented in [`decs`](https://github.com/vblanco20-1/decs). In fact, one can
easily combine also sparse sets and chunks if needed, but you've to weigh the
pros and cons before moving in this direction since having the chunks means
giving up other things.

Having said that, here is a quote from the Unity documentation that explains how
a chunk based design can get more in some cases:

>When you add a shared component to an entity, the EntityManager places all
>entities with the same shared data values into the same Chunk. Shared
>components allow your systems to process like entities together. For example,
>[...] When rendering, it is most efficient to process all the 3D objects that
>all have the same value for those fields together.

However, those who wrote the documentation were aware of both the strengths and
weaknesses of this approach:

>Over using shared components can lead to poor Chunk utilization since it
>involves a combinatorial expansion of the number of memory Chunks required
>based on archetype and every unique value of each shared component field.

In other words, the risk is to waste a lot of memory and increase the
fragmentation (or put in the other way around, reduce the performance during
iterations) if the shared components are misused (or _over used_) when it comes
to working with chunks.<br/>
Actually, it can be worse than this if you consider that two entities go in the
same archetype and thus the same chunk only if they have exactly the same
components. In all other cases, it doesn't matter if they _share_ something,
they'll occupy different chunks in different archetypes anyway and this makes
this _feature_ suitable mainly for _static types_.<br/>
Finally, it's easy to imagine that there can exist more entities than those that
fit in a chunk all sharing the same component. Once again, the fragmentation can
reduce a bit the advantage.

With these premises, the chunk-based design can bring some benefits if used
_properly_, especially when combined with a flyweight model.

To be honest, I made an effort to find another use case for this particular
approach and it didn't come to my mind. Or rather, I haven't found any for which
the benefits outweigh the problems.<br/>
This is probably due to the fact that, for various reasons, I prefer to rely on
other designs and I'm used to thinking about different models. I invite anyone
who wants to investigate the matter or present other interesting use cases (with
which to also complete this section if possible) to contact me for a chat. I'd
be glad to discuss them.

## Hierarchy like

We've already encountered a different _sharing model_ in the
[part 4](https://skypjack.github.io/2019-06-25-ecs-baf-part-4/)
of this series and its
[follow-up](https://skypjack.github.io/2019-08-20-ecs-baf-part-4-insights/).
That is how we structured our components in order to define hierarchies.

This time, entities share a parent and from the latter we can easily iterate
through all the children.<br/>
If compared to the previous solutions, it has both pros and cons. An obvious
drawback is the cost in terms of memory consumption, since we have one (or more)
pointers for every entity (the `parent` component) while the chunk-based design
has only an additional pointer per chunk. On the other side, this solution makes
possible to iterate **all** the entities that share the same component at once.
Using chunks is likely to introduce fragmentation instead and therefore to
suffer from the fact that not all entities are at hand all the times.

I won't explore further this model though. I highly recommend reading the posts
above to know how this design works instead.

## Shared data

The models above are based on instances managed externally (resources) or
entities linked together by means of dedicated components (hierarchies).<br/>
It's time to look into some models that allow to share data directly.

### Master and slaves

One of the possible ways of sharing data is that in which they are owned by a
_main_ entity (the _master_) and a bunch of _secondary_ entities (the _slaves_)
want to access them.<br/>
Sharing data through hierarchies is configured in this way in a sense. In fact,
in a multi-level hierarchy, the leaves could get to access data exposed by the
root of the tree, even though the latter isn't a direct parent of the former.

There aren't many ways (or at least, not many that are worth mentioning) to make
this model _flat_.<br/>
The most obvious one is that of using two components: the _data_ and the
_pointer-to-data_. I don't think there is a need to explain who is assigned
which component and how it works. All in all, this is a kind of _flat_
hierarchy.<br/>
A slightly different implementation is that in which everyone is assigned the
same component type. In this case, the component is defined in such a way that
it can assume both the form of the data or of the _pointer-to-data_. Something
like a tagged union or a safer `std::variant` could work in short. If it may not
seem like a good compromise, in fact it is not for this type of design but
something similar could prove very useful for other models, as we will see.

Long story short, personally I find that the hierarchical design is the one that
best suits this model of data ownership. Among other things, it is flexible
enough to be complicated at will and to satisfy every requirement, such as the
need to know who's _looking_ at data of a given entity.<br/>
In its _flat_ form with two components, it's even possible to sort, process,
reason separately on those who manage the data and those who _observe_ them,
which can give us advantages in some cases.

The main problem with this approach at sharing data is what to do with the data
when the owner dies or leaves the scene in some other ways.<br/>
One possible solution is to transfer the data to the first available candidate
but it may not be the desired result. In the hierarchical model, this translates
into choosing a new parent, that's all. A variant is that of designing an
algorithm that makes the orphans _elect_ their new master.<br/>
The group can even be dissolved and whoever was observing will have nothing to
refer to from now on. I can imagine some use cases for this approach in fact, so
it cannot be ruled out.<br/>
Another solution may be to provide all children with a copy of the data and let
them decide for themselves from then on.

In short, as mentioned, often this decision is application specific and in the
same software we may have different data managed differently when the owner
ceases to exist.<br/>
It's difficult to define what to do in a generic way on the subject.

### Ownerless

This model is more common and perhaps more useful than the previous one, at
least from my point of view. In this case, there are a number of entities that
share data but none of them actually _own_ it.<br/>
The _flyweight_ model enters this category. Resource management is demanded to a
dedicated class. Entities are assigned components that _refer_ some resources
but none of them actually owns it. It's not mandatory that shared data are wiped
out immediately when no entity refers them, as it happens for resources that are
lazy destroyed.<br/>
Similarly, the block based reference to external resources falls in this family.

A _standardized_ tool to share data between entities and have them deleted when
unused is a shared pointer, that is nothing more than a type and can be used as
component if needed.<br/>
To be honest, it's pretty annoying to do this with libraries like `EnTT` that
rely heavily on the type system of the C++, mainly because you can no longer
`get<T>(entity)` your instance but you've to `get<std::shared_ptr<T>>(entity)`
it instead. Fortunately, aliases can mitigate the ugliness here.<br/>
The fact that the lifetime of the shared data depends on that of the entities
that _own_ the component can be either an advantage or not. If it's so mostly
depends on the requirements you have time by time. Again.

When it comes to working with `EnTT` or any other ECS library that allows for
multiple containers (or registries in the `EnTT` terminology), there exists also
another viable solution that doesn't require us to design a new class to manage
shared data nor to use a shared pointer.<br/>
Since there is nothing wrong in creating multiple instances of our container, we
can use one of them to host entities and components for the simulation and
another instance to put aside shared data (for example, prototypes). This works
as long as we can easily make entities refer each other across registries, that
is straightforward usually. Unless you can guarantee pointer stability, we'll
refer to shared data by means of entity identifiers though and this may
introduce indirection. On the other side, the lifetime of the data no longer
depends on that of the entities that share the component, that is a nice-to-have
feature in many cases.

### Copy on write

_Ownerless_ sharing policies are probably the most common ones. Sometimes we
want to make entities _exit_ their groups though and therefore we need something
more than a simple _reference_ to shared data.<br/>
This isn't as easy as it seems at a first glance. As always, we must find a
tradeoff between the typical requirements that make a developer happy: easy to
use, fast enough and designed in such a way that it doesn't consume more memory
than what it would have consumed otherwise by assigning an instance of our
component to all entities.

For what it matters, my idea of _easy to use_ is that of a tool that doesn't
require me to guess at runtime and in the codebase whether an entity refers to a
shared instance or its own copy of the components.<br/>
Fast enough doesn't need explanation but the idea is to avoid having an indecent
number of hops to reach the information if possible.

The problem of memory consumption is interesting though. If we had `N` entities
and the size of our component was `S`, we cannot expect to spend more than
`(N+1)*S`, where the extra space is eventually reserved for the shared instance.
However, the optimal solution is that in which we spend `S` when all entities
are assigned the same component and thus share the same instance, while the
consumption grows up to `N*S` when the entities _exit_ the group.<br/>
As far as I know, the actual consumption is strictly related to how easy to use
the final solution is. The more we save, the uglier will be the final code at
the call site.

Despite the efforts, I didn't find an approach to reduce memory consumption to a
minimum that doesn't also require to use two different components, one for when
the entity is assigned the shared component and the other one for when it owns a
private instance.<br/>
Put in these terms, the implementation details are straightforward. The problem
is when we decide to iterate all components, no matter if they are shared or
private instances. We have two different loops for that, one for every type.
Another problem is that we cannot _sort_ entities since they are split in two
different sets. We cannot even easily make cross references to get the component
when needed, since we don't know what type the entity owns and this requires a
branch. And so on...

If we accept to increase the memory consumption a bit, we can design our
components such that they have the actual data and a sort of _pointer-to-data_.
All entities will have at least an instance of the latter. For those that share
the data in an ownerless model, the _pointer-to-data_ points to an instance of
the component stored somewhere else (for example, in a side registry). In all
other cases, the entity is assigned also an instance of the data and the
_pointer-to-data_ points to the entity itself.<br/>
The copy and thus the assignment of an instance of the data to the entity and
the update of the _pointer-to-data_ component happens when the need to write and
use a custom copy of the shared information arises.<br/>
Iterations take place on the _pointer-to-data_ component this time, whatever it
is. We don't have anymore to split the loops in two parts. Moreover, we can now
reason on our entities and even sort them and therefore their _pointer-to-data_
components if needed. Roughly speaking, we have most of the benefit of a single,
tightly packed component although we are using two different types.<br/>
Note that in the best case, that is when all entities share the same piece of
information, memory consumption is pretty low. In the worst case instead, that
is when all entities own a private copy of the data, memory consumption is as
bad as it can be. It's a matter of knowing your application to know if this
approach is worth it though.

A variation of the previous approach exists too. It has some pros in terms of
performance but it also requires _more memory_.<br/>
The idea is that of using a single component that contains both the data and the
_pointer-to-data_. This means that all entities will consume as much memory as
it is required for the component, even if they don't own a private copy.<br/>
During iterations, the entities that _share_ some data will introduce a hop to
reach them while the entities that own a private copy will have it along with
the _pointer-to-data_ and therefore will avoid the indirection.

# A real world example

As I said, I'm not a fan of the built-in solutions but I don't discourage them
or the use of them.<br/>
[Sander Mertens](https://github.com/SanderMertens) explains below how the
solution he implemented in [`flecs`](https://github.com/SanderMertens/flecs)
works, the problems he tried to solve and why it _matters_.

## Prototyping

One application of component sharing is the ability to create entity prototypes
(sometimes called _templates_ or _prefabs_).<br/>
Prototypes are special entities that an application can instantiate, so that
each instance ends up with the same values as the prototype. Prototyping can be
implemented in many ways. We'll discuss how it can be implemented with component
sharing only here.

To create a prototype, we need to create an entity that contains the components
we want to _share_. We can then create an entity that _uses_ the components from
the prototype.<br/>
Lets call the prototype the _base_, and the instantiated entity the _instance_.
Multiple instances can share the same base. This approach is in many ways
similar to master-slave component sharing discussed earlier.<br/>
In pseudo code:

```cpp
entity base;
base.set<Position>({10, 20});

entity instance_1;
instance_1.add_instanceof(base);

entity instance_2;
instance_2.add_instanceof(base);
```

In the above example `instance_1` and `instance_2` share the component
`Position` from `base`. When `Position` changes on the base, it will also change
for `instance_1` and `instance_2` as they point to the same address in memory.
Base _owns_ `Position`, whereas `instance_1` and `instance_2` _share_ it.

There are several features that can make prototyping more expressive. I'll
describe some of them in the following sections.

### Overriding

Overriding is the ability to assign a private value to a shared component by
adding it to the instance. Upon overriding, the value of the base component is
copied to the instance component.<br/>
In pseudo code:

```cpp
entity base;
base.set<Position>({10, 20});

// Create instance that shares Position from base
entity instance;
instance.add_instanceof(base);

// instance owns Position with value {10, 20}
instance.add<Position>();

// instance shares Position from base again
instance.remove<Position>();
```

In `flecs` applications have the ability to _auto-override_ components by
creating a _type_ that both has the instance of relationship on the base as well
as the override component:

```cpp
entity base;
base.set<Position>({10, 20});
base.set<Velocity({1, 1});

type prototype;
prototype.add_instanceof(base);
prototype.add<Position>();

// instance owns Position with {10, 20} and shares Velocity with base
entity instance;
instance.add(prototype);
```

### Variants

With variants applications have the ability to specialize prototypes by creating
a prototype hierarchies. Variants are enabled by the ability to share components
from nested base entities.<br/>
In pseudo code:

```cpp
// a white circle
entity circle;
circle.set<Circle>({10});
circle.set<Color>({255, 255, 255});

// create red circle, override color
entity red_circle;
red_circle.add_instanceof(circle);
red_circle.set<Color>({255, 0, 0});

// Shares Circle from 'circle' and Color from red_circle
entity my_red_circle;
my_red_circle.add_instanceof(red_circle);
```

### Caveats

Prototyping is an useful ability in game development, and when implemented with
component sharing it provides an application with a single consistent way of
defining both prototypes and entities.

There are however some caveats to consider. The most obvious one is that since
prototypes are regular entities, they are also matched with systems which is
usually undesirable. A common way to address this is to introduce a special tag
that is filtered out by systems.<br/>
Another caveat is that prototyping can, depending on the implementation,
introduce fragmentation as entities with different base units are typically
stored in different arrays.

Consider this example:

```cpp
entity base;

entity e1;
e.add<Position>();

entity e2;
e.add_instanceof(base);
e.add<Position>();
```

Here both `e1` and `e2` have exactly the same components, but because `e2` has
the instance of relationship with the base, it will be stored in a different
array.<br/>
This is ok when the number of instances is much larger than the number of base
entities, but if not this can cause performance issues.

Another thing that may initially seem strange is that, since prototypes are
regular entities, they can also be deleted.<br/>
Consider this example:

```cpp
entity base;
e.add<Position>();

entity e1;
e.add_instanceof(base);

base.delete_entity(); // now what?
```

This may seem like it would introduce undefined behavior, but we can solve this
elegantly since in ECS entities are not _objects_ but _identifiers_ (typically
an integer value). Even though we can delete all components from an entity,
the _identifier_ may still be valid.<br/>
In the above example we create an instance of relationship to `base` using its
identifier and that relationship persists even after deleting the base.

# Conclusion

Unfortunately I didn't find the time and space to provide practical examples.
It took me longer than expected to write this post and it contains a lot of
information already.<br/>
Probably I'll write an _insight_ in future to describe how one can implement a
_pointer-to-data_ component or how we can use a side registry to setup shared
data and back links, then walk through the entities that share the same
instance.

For the time being, I hope you enjoyed what you've read so far.<br/>
Shared data are sometimes useful but what the _right_ model is for your case
strictly depends on the requirements. There doesn't exist a _one-catch-all_
solution unfortunately.<br/>
Note also that the shared components on the one hand solve a problem, on the
other they introduce some new design choices to be taken into consideration.
Sometimes these can quickly turn into problems, especially if you don't choose
the _right_ model for your problem.

All in all, however, they are a useful tool to solve some problems. In this
sense, I hope I've given you some useful tips.

## Shout out

Thanks once again to [Sander Mertens](https://gist.github.com/SanderMertens)
for contributing to this post. <br/>
This isn't the first time that we cooperate. We have a rather different approach
to the problem and this helps to put many good ideas on the table.<br/>
I hope you found the different points of view interesting. Hopefully, this won't
be the last chance to work together!

## Let me know that it helped

If you liked this post and want to say thanks, consider to star the
[GitHub project](https://github.com/skypjack/skypjack.github.io) that hosts this
blog. It's the only way you have to let me know that you appreciate my work.

Thanks.
