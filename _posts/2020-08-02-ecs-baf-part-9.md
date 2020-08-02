---
layout: post
title: ECS back and forth
subtitle: Part 9 - Sparse sets and EnTT
g5-repo: skypjack/entt
tags: [ecs, entt, cpp]
---

It's been a while since my last post. You know: a global pandemic, a son that is
now 4 years old, the time spent in researching for something that I've been
chasing for some time in the ECS field, and so on.<br/>
In the end, here we are again, with another post from the _ECS back and forth_
series.

This time I want to give an answer to all those who have asked what the sentence
below means:

> As an example of this, the implementation proposed by EnTT is quite different
> from the classical one and hardly anyone would say it's a sparse set without
> knowing the details and being able to understand the differences.

This is a mention from
[another post](https://skypjack.github.io/2019-03-07-ecs-baf-part-2/) of mine
and apparently it left some doubts to those who read it.<br/>
Let's dig a little into this data structure to understand how it works and what
changes in [`EnTT`](https://github.com/skypjack/entt).

## Introduction

The sparse set is a structure that many developers are not familiar with or have
never encountered, for reasons completely unknown to me.<br/>
In fact, it's widely used in one variant or the other but many developers cannot
say what it is, despite its simplicity.

I've already talked a little about this data structure and its pros and cons
when used for the implementation of a component model. I also mentioned its
flexibility and how it can be used in different ways. For example:

* As a brick for component storage, that is, as a basis for object pools.
* To keep track of relevant entities for a system, that is, as a basis for
  defining the systems themselves.

While `EnTT` implements the first model, a good example of a project that uses
sparse sets for its systems is [`ecst`](https://github.com/SuperV1234/ecst), a
library that inspired me quite a lot in the past.<br/>
There are also many other uses for sparse sets. For example, in an archetype and
eventually chunk-based model they could be used to track alive entities to
quickly retrieve the pool and/or chunk that contains them.

In short, `EnTT` didn't invent anything special but only exploited an already
known data structure, bending it to its own needs.<br/>
In the following sections we will see how a sparse set works in its basic
implementation and how `EnTT` has made it more performing.

## Sparse set

So, what's so special about sparse sets?

Long story short, they make us pay a little more memory and give back better
overall performance.<br/>
How much better? I'd say definitely, in many ways, even though I'm probably
biased. To sum up:

* They make it possible to iterate all contained elements as a tightly packed
  and contiguous array and so it's O(N).
* Adding and removing elements is O(1) as well as lookup operation.
* Sorting is possible and the cost mostly depends on the chosen approach (I've
  already written a
  [post](https://skypjack.github.io/2019-09-25-ecs-baf-part-5/) on this, so I
  won't repeat myself).

How is this possible? The sparse set is represented by two vectors: a _sparse_
one and a _dense_ one.<br/>
Its intended purpose it that of mapping a potentially large set of integers (or
elements that are at least convertible to an integral type) to a smaller one.
The sparse array is indexed by the integer itself while the dense array contains
all elements in random order (unless you sort it, of course).

![sparse set](https://user-images.githubusercontent.com/1812216/89120776-31565080-d4b9-11ea-905c-df3df0d5e7d6.png)

As you can see, the sparse array contains the position of the element within the
dense array. On the other hand, the dense array contains the integer used to
_index_ the other array, that is, the element itself.<br/>
To know if an element is _contained_ by a sparse set in its basic implementation
you've to do this:

```
bool contained = (dense[sparse[element]] == element);
```

As easy as it can be and definitely fast, even though we can make it even
_faster_ (more on this later).

To add an element to a sparse set, we push it back in the dense array and we set
the position in the sparse array:

![sparse set: add](https://user-images.githubusercontent.com/1812216/89120814-65317600-d4b9-11ea-8bbe-a802179d3d3e.png)

The (pseudo)code for this is the following:

```
const auto pos = dense.size();
dense.push_back(element);
sparse[element] = pos;
```

Things get a little more complex when it comes to removing elements. This is due
to the fact that we want to keep the dense array tightly packed.<br/>
To do this, before removing an element we swap it with the last one in the dense
array and update their values in the sparse array to reflect the new positions:

![sparse set: remove 1](https://user-images.githubusercontent.com/1812216/89120831-8eea9d00-d4b9-11ea-86fb-a08621cfabc0.png)<br/><br/>
![sparse set: remove 2](https://user-images.githubusercontent.com/1812216/89120832-8f833380-d4b9-11ea-9d5f-c798c89c64c7.png)

If it sounds difficult, it is not in fact:

```
const auto last = dense.back();
swap(dense.back(), dense[sparse[element]]);
swap(sparse[last], sparse[element]);
dense.pop_back();
```

Why don't we also _reset_ the position in the sparse array for the removed
element? Remember how we lookup elements:

```
dense[sparse[element]] == element
```

In this case, `dense[sparse[element]]` doesn't contain anymore `element` because
we just put there another value. So, there is no need to _reset_ the position in
the sparse set. The check will fail anyway.

Note that I've deliberately omitted all controls on the size of the two arrays
so far to make the presentation _easier_. Obviously, this isn't something to
overlook in a real world implementation.<br/>
For example, a lookup test is more like the following than what we've seen
above:

```
const bool contained = (element < sparse.size() && sparse[element] < dense.size() && dense[sparse[element]] == element);
```

After this very brief introduction, let's see what the potential _problems_ of
this data structure are and how we can mitigate them.<br/>
Yeah, I know, as the author of a library that uses this data structure all over
the codebase, I shouldn't admit that there are _problems_ with it but ... hey!
I'm not that kind of author and so there are already enough out there to make me
laugh. So, let's be realistic, let's admit that the perfect solution doesn't
exist and let's see together how to get the best out of what we have.

### Sparse array

The first point that literally everyone debates sooner or later is the size of
the sparse array.<br/>
It's easy to understand that this can cost a little memory for few benefits in
the worst case. For example, what if I had 200k entities and I assigned 50
single instance components to the last one?<br/>

Fortunately, these extreme cases are rare as heck. It's also true that `EnTT`
and thus this data structure is used in `Minecraft`, not exactly a small game,
and of all the feedback I've received so far the memory usage of sparse arrays
has never been mentioned.<br/>
The truth is that today memory is almost no longer a problem in the vast
majority of cases but it's still worth knowing that it can become one and
therefore how to improve the situation. Moreover, we can still have less dense
pools, so it may be useful to save some memory in these cases.<br/>
Note that `EnTT` is today in the midst of a transition from predefined pools to
fully customizable pools. In order not to anticipate too much and to make things
simple, I'll focus only on the approach used in the first case.

Ok, having said all this, I'll make it very short: _pagination_.<br/>
This is how `EnTT` _solved_ the problem (although this is about to change but I
have to keep aside some material for the next posts!).

What does it cost and in what measure the problem is _solved_?<br/>
The cost for this is that of an indirection to reach the page when the sparse
array is accessed. This is also one of the reasons for which I'm about to change
it. From my experience in the field, I can say that the cost is irrelevant
during iterations since the external array is very rarely accessed (for example,
it's completely ignored during iterations on groups and single component views).
On the other hand, adding and removing elements use the sparse array and are
thus affected to an extent. This cost has never shown up so far in profiling
bottlenecks, so it doesn't worry me much, but it's there and it's worth
mentioning.<br/>
Similarly, the problem isn't completely solved. For example, if you add one
element per page to the sparse set, you have the same memory usage as before.
However, this is far from being the usual access pattern I've seen in real world
applications and it's a _problem_ only on paper (mainly becuase similar entities
are often create in bursts). Instead, tuning the right size for your pages can
matter here and make you save more memory in the best case. Again, if the memory
usage is your problem. Otherwise, my advice is to move on and let it be. In my
humble opinion, this is the right price to pay nowadays for all the pros of a
sparse set.

### Lookup

The other aspect that often makes the nose turn up is the cost of a
_lookup_.<br/>
Do I have to access two arrays to find out if my element is in there? How much
is it? Not that much but we can reduce it further if you like.

To do that, you've to introduce a _tombstone_ element in your codebase, that is,
something you recognize immeditely as invalid in the sparse array.<br/>
`EnTT` uses its `entt::null` object for the purpose. All elements in the sparse
array are initialized with this value. Similarly, when you remove an element
from a given set, we also update the sparse array and set `entt::null` as the
_position_ for the entity.

How does it work exactly? The way `EnTT` does it is slightly different from what
I'm going to describe but the following should be enough to get the full
picture.<br/>
Let's imagine our system supports a maximum of 1M entities. In this case, we can
sacrifice an entity and make it the _null_ one. I know, you're tempted to
sacrifice entity 0 because... oh, c'mon, because 0 converts to false, it's
fantastic! Well, no, it is not. If you do that, you're literally sacrifying
position 0 in the dense array. This means that either you waste the first
element for every type or you have to adjust the position all the times. So, the
choice is between wasting some memory or some cpu cycles. It's not that
attractive apparently. The other way around is as simple as it can be: if you
can't reserve the _first_ entity, then use the _last_ one.<br/>
With this in mind, we can now use value `(1M-1)` as a tombstone in the sparse
array and our lookup test becomes:

```
sparse[element] != (1M-1)
```

If you remember it, it was like this instead:

```
dense[sparse[element]] == element
```

As you can see, we no longer access the dense array. The tombstone tells us
already what we need to know.<br/>
Looks good? It is, but we only moved the cost around somewhat and reduced it a
little. This is how this kind of optimizations work after all.

First of all, consider adding and removing entities to a sparse set.<br/>
In the first case, nothing changed. Adding entities works exactly as before. In
the second case, we put now a tombstone in the sparse array while we didn't even
update it before. Does it worth paying this cost to improve the lookup? Let's
dissect the two operations.<br/>
The lookup now doesn't reach the dense array anymore, so we definetely saved a
lot here. On the other hand, removing an entity updates also the sparse array.
Are we accessing something we didn't before? If you give a look at the
implementation, we were already reading the sparse array to know the positions
of the elements in the dense array. Therefore, we haven't added the cost of
pulling it from the memory in this case and in fact we are going to update the
same element we want to read. We pay the cost of flushing it though.<br/>
So, all in all, we moved the cost somewhere else and reduced it to a minimum.

If we gained so much on this front, where did we lose then? Because that's how
computer science works, it's a matter of compromises.<br/>
Well, in this case, we have _lost_ an entity. This is the real price paid. We
will no longer be able to create 1M entities but _only_ `(1M-1)`. I don't want
to exaggerate but I think it's a more than fair price to pay for what we bought.
Plus, we can always use a 64-bit integer if we want even more entities!

### Iterations

This is the last thing in which the sparse set implementation of `EnTT` differs
from the classic one.<br/>
Before discussing it though, it's worth mentioning the problem it tries to
solve.

I use the library mainly for code organization and rely on its fast paths
(groups and such) to get around bottlenecks. The last one is also a point on
which I've been asked several times to be honest. How do I manage to design
reusable systems with groups? How can I avoid group conflicts without
problems?<br/>
I find it amusing to see how those who have never used this model find reasons
to _criticize_ it in their _incompetence_ (intended as _incompetence_ on the
model itself, I'm incompetent enough in many other ways to pretend to feel
superior to anyone in this field). I think I've better arguments than those who
have never wanted to understand it for some reasons though. Because of this,
I'll try to answer these questions with one of the next posts, so as to dispel
any doubts and make it clear where this model originally came from. Apparently
an approach unknown to most but that I found valuable.

So, back to the topic, the first thing I tried to address is the possibility to
add and remove elements easily and in the cheapest possible way during
iterations.<br/>
There are many approaches for that. Command queues for delayed operations,
staging areas with sync/merge points, and so on. What people usually don't tell
you is that all these solutions have an hidden cost. Roughly speaking, you're
doing the same operations twice (for some definitions of _twice_) in all cases.
You can reduce the cost but you can't avoid to duplicate it.<br/>
Consider now the differences between inner and outer parallelism. If you split
correctly your computation, in fact you **don't** need any cumbersome costly
workaround in the vast majority of cases, no matter how transparent it is.<br/>
So far, so good. Can we avoid these costs whenever we don't need to pay them?
Unfortunately, no. Whether or not this is possible depends on the model in use.
For example, with an archetype (eventually chunk based) solution you can't
really do it and for many reasons. Is this a problem? Again, no. For example,
well known engines offer the possibility to get around the limitation offering a
method to _put aside_ the operations for a sync point. There is nothing wrong
with that, it costs a little more but it also gets the job done. The good news
is that a sparse set based model gives you the possibility to completely
eliminate these costs in many cases.<br/>
You know, I'm a _make me decide what to pay_ freak after all.

Doing this is pretty easy. Imagine to iterate the elements from position 0 to
N-1 in the dense array. If you remove one of them or add another one, things
break quickly and iterators get probably invalidated in a blink.<br/>
Try to flip the problem on its head now. If you iterate things backward, that
is, from N-1 to 0, this is what you get:

* You can add elements, since adding means pushing them to the end of the dense
  array. They won't enter the iteration and they won't invalidate iterators (if
  properly defined, of course).

* You can remove the element currently iterated. In fact, it's swapped with the
  last element in the dense array, that is, one we already visited during our
  iteration. Therefore, there is no risk we will visit them twice after a `++`.

Of course, this doesn't make it possible to get other solutions out of the way
in all cases, but it does so in a lot of situations. Definitely a nice-to-have
alternative to combine with those approaches that offer different possibilities
and higher costs by their nature.<br/>
The question to ask is: is the cost the same as a _normal_ iteration? Long story
short: yes, if your compiler is smart enough. You can look at the assembly
generated for `EnTT` to realize that a good compiler can vectorize iterations
pretty easily. However, iterators are a little more _complex_ in this case, at
least if you don't want to get them invalidated. That's also why `EnTT` offers
the possibility to iterate elements in both directions and therefore to get even
more from a sparse set. It's up to you to decide: if you know that you won't add
nor remove elements, you can freely ignore all this stuff and go straight to the
point with an _unsafe_ iteration.

## Conclusion

At the end of this digression, we saw how the sparse set is a truly flexible
data structure, but also prone to various types of improvements.<br/>
I tried to get the best out of this object a bit at a time, squeezing every
possible CPU cycle from it. I probably missed something, maybe I did something
else wrong, but I certainly did my best so far.<br/>
My hope is that this post will be a starting point for those who intend to use
the same data structure for their purposes.

## Let me know that it helped

I hope you enjoyed what you've read so far.

If you liked this post and want to say thanks, consider to star the
[GitHub project](https://github.com/skypjack/skypjack.github.io) that hosts this
blog. It's the only way you have to let me know that you appreciate my work.

Thanks.
