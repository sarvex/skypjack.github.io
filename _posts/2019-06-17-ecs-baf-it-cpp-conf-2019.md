---
layout: post
title: ECS back and forth
subtitle: Italian C++ Conference 2019
tags: [talk, ecs, entt, cpp]
---

[These]({{ site.baseurl }}/pdf/ecs_back_and_forth.pdf) are the slides of a talk
I gave at the
[Italian C++ Conference 2019](https://italiancpp.org/itcppcon19) in Milan. The
conference was organized by the
[Italian C++ Community](https://www.italiancpp.org/).<br/>
The goal of the talk was to present the three best known and apparently most
used models for the implementation of a software (either a game or whatever)
based on the entity-component-system architectural pattern.

In particular:

* The Big Array: used by [entityx](https://github.com/alecthomas/entityx) among
  the others, probably one of the first ECS libraries in C++.
* Archetypes: used in the Unity engine, but also by
  [decs](https://github.com/vblanco20-1/decs) if you're looking for an
  open-source implementation in C++.
* Sparse Set: used by [EnTT](https://github.com/skypjack/entt), an ECS library
  in modern C++ and of which I'm also the author.

In just 50 minutes I tried to condense the pros and cons of these models
(probably failing in the attempt, but I did my best), hoping to give a grasp of
the topic to those interested.

The talk was recorded and should be made available soon as far as I know.
Therefore, I'll update this post as soon as I've a link to a video in which
these slides are commented and completed as they deserve.

## Let me know that it helped

I hope you enjoyed what you've read so far.

If you liked this post and want to say thanks, consider to star the
[GitHub project](https://github.com/skypjack/skypjack.github.io) that hosts this
blog. It's the only way you have to let me know.

Thanks.
