---
layout: post
title: ECS back and forth
subtitle: Part 5 - Sparse sets and sorting
gh-repo: skypjack/entt
gh-badge: [star, follow]
tags: [ecs, entt]
---

As we have seen with
[this post](https://skypjack.github.io/2019-03-07-ecs-baf-part-2/), the sparse
set is a great tool for decoupling component pools and creating a much more
flexible model.<br/>
Another of the big advantages of using sparse sets, although less obvious, is
that of being able to easily sort an entire pool of components according to our
needs. How easy is it to induce an order in such a complex data structure
efficiently?

With this post I'll try to describe some solutions with which I've experimented,
emphasizing their pros and cons.

If you haven't done it yet, read the
[previous parts](https://skypjack.github.io/tags/#ecs) of the series before to
continue. It will help to fully understand this one.

## Introduction

One of the first problems encountered when we want to sort a sparse set is the
fact that it's composed by two vectors, the sparse array and the dense array.
Consider `std::sort` and how we would use it. There is no way to sort two
vectors at once directly with it. This gets even _worse_ when the sparse set is
used as a base for the pools of components because we have three vectors in this
case (at least with [`EnTT`](https://github.com/skypjack/entt)).<br/>
However, a sorting algorithm like `std::sort` would be perfect for in-place
sorting. It can sort pools using no auxiliary data structures nor extra memory
but for the few variables used to control the algorithm itself.

On the other side, keep in mind that in-place sorting requires to swap elements
and therefore to move them here and there. This has a cost and it's amplified
when we've to do it on many different vectors.<br/>
So, what about an alternative approach that **does** consume memory but that has
also better overall performance? Permutations seems to be the way to go here.

Maybe there is also another solution, though. A method that combines the best of
the two worlds. How far can we go with it?

### Mr Obvious

The basic case of a sparse set is trivial to sort. If we don't use it as a base
for our pools or similar and therefore we have only two arrays (the dense and
the sparse ones), sorting the sparse set means sorting only the dense array,
then update the indexes in the sparse array:

```
std::sort(dense.begin(), dense.end(), compare);

for(auto pos = 0; pos < dense.size(); pos++) {
    sparse[dense[pos]] = pos;
}
```

Nothing easier. This works also when we combine entities and components in a
single object in the dense array, because we still have only two arrays, that is
the basic implementation of the sparse set.<br/>
However, this doesn't work that well when we have one or more extra dense arrays
like it happens in `EnTT`. If you are wondering why it's not enough to merge the
dense arrays and return to the basic case, the answer is because I want to have
always two separate pointers to the lists of entities and components, without
having to load them all in memory during the iterations and then filter what
isn't necessary.

In the sections that follow I'll refer only to the solutions that are useful in
cases in which there are two or more dense arrays associated with a single
sparse array.

### In-place sorting

In-place sorting is the best approach when we want to keep low memory usage.
These algorithms get the job done a swap at a time until the container is fully
sorted.

Sorting in-place a sparse set is a bit tricky and cannot be done with the tool
offered by the C++ standard library, namely `std::sort`. This function accepts
two iterators to the range to sort and starts to swap elements within the
container directly. Unfortunately, we don't have a single container here.
Instead we have two (or even more) of them and therefore this approach isn't
viable.<br/>
One of the problems is also the fact that `std::sort` requires random iterators
and they are such that they return actual references rather than proxy objects.
Otherwise the problem could be solved with a dedicated class that swaps what's
required in all the arrays when moved around.

The solution is pretty simple: roll your own sorting algorithm and take control
of the section where elements are moved. This means writing a very specific
function that works only with sparse sets and the performance of which are most
likely lower than that of `std::sort`.<br/>
No matter how good programmers we feel: `std::sort` is developed by people who
are probably smarter than us, it's widely tested and highly optimized for
various corner cases. We will hardly do better, but at least we'll have
something that works even with a sparse set!

How does this work?

I won't give you magic advices on how to implement a sorting algorithm. Whether
you're a young programmer or an experienced programmer, you've probably already
heard about sorting algorithms an infinite number of times. So, choose the one
you prefer and start to sort the dense array of a sparse set. When we get to
the point where two elements need swapping, this is where things change and
these are the steps to do:

* Swap the elements in the sparse array using the _positions_ expressed by the
  dense array.

* Swap the elements in the dense array. Repeat this step for **all** the dense
  arrays.

If you remember how a sparse set works, the dense array is a kind of backward
link to the sparse array used for validity checks. Therefore, as the elements in
the sparse array literally _points_ to the cells in the dense array, the
opposite is true as well. This makes the whole thing straightforward to
implement.<br/>
The first step makes our elements point to each other values. The second step
just swaps the actual values and rebuild the links.

As you can see, ordering a sparse set in-place is pretty trivial. However, when
two elements need to be swapped, the number of operations is much higher than
usual. The more dense arrays we have, the more this cost increases with the
number of swaps, with the risk that it may become _too_ high.<br/>
If we combine the cost of the _swaps_ with the fact that our sorting algorithm
will probably not be as efficient as for example `std::sort`, it's clear why
this solution is not always the best one.

### Permutation

If you can afford an out-of-place sorting, permutations are the way to go to
exploit the performance of `std::sort` and drastically reduce the number of
swaps.<br/>
The basic idea is simple: first of all, compute the permutation in an external
array. Then apply it to the sparse set. The cost of the first step is that of
the sorting algorithm but this time we're sorting a vector of integers. On the
other hand, applying the permutation to a sparse set is linear with the size of
the container and this should help to speed up everything quite a lot.

To get our permutation, first of all we need a temporary array the size of which
is the the same of the dense array of our sparse set:

```
std::vector<std::size_t> copy(dense.size());
std::iota(copy.begin(), copy.end(), std::size_t{});
```

We initialize the array with the numbers from 0 to N-1, where N is the size of
our dense array. It will give us at any time the information about what position
a given element should have in the sorted sparse set. It goes without saying
that the _i-th_ element occupies the _i-th_ cell before to start and that's why
we fill it this way.

To _sort_ our support vector, we use one of the dense arrays (most likely the
one that containse the components) and the positions returned by `std::sort` as:

```
algo(copy.begin(), copy.end(), [this, compare = std::move(compare)](const auto lhs, const auto rhs) {
    return compare(dense[lhs], dense[rhs]);
});
```

Where `compare` is the comparison function provided by the user. From the
`dense` array we get the elements that occupy the given positions and return
them instead of the numbers used to fill `copy`, because the latter have no
sense for the users. However, note that those numbers are _stable_ even if they
are moved around within `copy`. Therefore they can be safely used to retrieve
the _right_ elements step by step.<br/>
If we look at a real world example like
[`EnTT`](https://github.com/skypjack/entt) (until version 3.1 at least), it's
really close to the example above. Because users are sorting components and not
entities, we use the positions to retrieve the actual instances of the
components and return them directly. The logic is exactly the same, in fact we
have two dense arrays that are kept in sync and it's a matter of using _the
right one_ to get what we need.

When `std::sort` returns, we have the permutation to use to sort our sparse set
within `copy`.<br/>
To _apply_ the permutation to our sparse set, the basic idea is that we get the
_i-th_ element and move it around, one swap at the time, until it reaches its
destination. This has the side effect that also all the elements encountered
along the path are placed in their final positions.<br/>
The way we must move the element is _encoded_ directly within `copy`:

```
for(size_type pos{}, length = copy.size(); pos < length; ++pos) {
    auto curr = pos;
    auto next = copy[curr];

    while(curr != next) {
        swap(copy[curr], copy[next]);
        copy[curr] = curr;
        curr = next;
        next = copy[curr];
    }
}
```

As described above, `swap` is a function that is in charge of swapping the items
both in the sparse array and in the dense array (be aware that we aren't
invoking `std::swap`, we don't want to swap the elements within `copy`).<br/>
If it seems complicated and it's not immediate why the cost is O(N), let's
dissect the snippet and see what's happening under the hood.

If we get a closer look at the permutation to apply to our vector, we note that
we can start from any index and jump to the element that should occupy that
position, one element at a time and until the first item reaches its position
(this happens when we get to the point where we should jump back to the starting
point).<br/>
In other terms, we can spot one or more _cycles_ in the support vector. Every
element _points_ to the item that reclaims its position and we have a closed
path if we put these steps in a row.

Because of this, every time we make a jump, it will be enough to move back the
element we find in the new cell and take the initial one with us until we jump
to its final position. When the next jump takes us to the starting point, we
stop and release the object in the current cell. The snippet above exploits this
fact and nothing more. It applies the permutation _one thread at a time_.

The cost is linear despite the internal loop becasue the `while` is entered only
for the elements that haven't been moved already to the _right position_. To
skip the elements that were part of the _cycles_ we walked through so far, we
use the `copy` array and update it every time we make a jump. We know that the
position we left behind after a jump is occupied now by the _right element_ and
therefore we can set `copy[i] = i`, that is what satisfies the guard of the
internal loop.<br/>
This way, the loop serves only the purpose of iterating a cycle, it doesn't make
the overall cost grow up fortunately. It also happens that this approach is
quite efficient, even though it requires auxiliary memory in order to save our
permutation.

### Mixing in-place sorting and permutations

Fortunately sparse sets have some feature we can exploit to get the best of the
two worlds, that is a kind of _permutation based in-place sorting_. This will
give us all the benefits of in-place sorting and get rid of the extra costs of
swapping.<br/>
Sounds interesting, doesn't it?

Permutations use a support vector, that is also why they consume memory. Sorting
in-place work with the dense array of the sparse set. The sparse array is what
we needed to be able to sort in-place the elements inside the dense array, thus
obtaining a permutation, then apply the latter to all the other arrays.<br/>
Another of the advantages of this solution is that we don't have to apply the
permutation also to the dense array used as a base to sort the set, which makes
us save further.

A possible implementation (quite close to that of `EnTT`) is this:

```
template<typename Compare>
void sort(Compare compare) {
    std::sort(dense.begin(), dense.end(), std::move(compare));

    for(std::size_t pos{}, end = dense.size(); pos < end; ++pos) {
        auto curr = pos;
        auto next = sparse[dense[curr]];

        while(curr != next) {
            adjust(dense[curr], dense[next]);
            sparse[dense[curr]] = curr;
            curr = next;
            next = sparse[dense[curr]];
        }
    }
}
```

It's not very different from what we've already seen, except for a few
details.<br/>
Basically, the `copy` support array is no longer necessary. In its place
`sparse` is used to literally keep track of the fact that we've already visited
some elements. These are then excluded when tested again in the internal loop,
keeping the algorithm linear in the length of `dense`.

Furthermore, the `swap` function mentioned above is no longer in use. This is
because this time we don't have anymore to swap elements within `dense`.<br/>
However, we introduce thn new function `adjust` that is invoked when we are
going to swap two entities within `sparse`. By construction, we sorted one of
the dense arrays but we have still to update all the other dense arrays as well
as the sparse one. Fortunately we can still get the original indexes of the
given entities from the sparse array and use them to swap the elements in their
data structures, just before we update on of those indexes.<br/>
We can define the `adjust` function as something along this line, repeated for
every dense array that is still to be sorted:

```
std::swap(another_dense[sparse[lhs]], another_dense[sparse[rhs]]);
```

If it seems complicated, well, I must admit that this time it's slighlty
trickier than what we've seen in the previous sections. However, the same
algorithm and ideas have been used here, so it's only a matter of putting the
pieces together to understand how the sparse array takes part into the
game.<br/>
If you're interested, you can look into the implementation provided with
[`EnTT`](https://github.com/skypjack/entt) and follow it step by step to see
what's happening under the hood.

## Conclusion

Sorting a sparse set can be simple but also very complicated, depending on the
use that is made of it and what constraints are placed on this part. The basic
implementation is trivial to sort but things tend to get more complex when we
_extend_ it with multiple dense arrays.<br/>
With this article I hope to have given you some inspiration to take the first
steps in this direction and, why not, to improve my solution further and propose
your modifications to the implementation provided with
[`EnTT`](https://github.com/skypjack/entt). It took me some time and several
iterations to get to what's there today, starting with a simple but less
powerful version, up to one that I can easily use on a large number of entities
without too many worries.

The best part of all this is that maybe I didn't even get close to the best and
I made errors that I can't notice because I'm so ignorant, so any feedback is
welcome!<br/>
Every time I look at this algorithm I have the feeling that something is wrong,
but maybe it's just my amazement that it really works.

## Let me know that it helped

I hope you enjoyed what you've read so far.

If you liked this post and want to say thanks, consider to star the
[GitHub project](https://github.com/skypjack/skypjack.github.io) that hosts this
blog. It's the only way you have to let me know that you appreciate my work.

Thanks.
