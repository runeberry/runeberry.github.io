---
layout: post
title: "Ecosystem"
---

# Ecosystem #

One of the things I definitely want to incorporate into the game engine is an Entity Component System. I like the idea of a highly modular, queryable database of "stuff" that I can manipulate to make game magic happen.

What's interesting about this approach is that it's so drastically different from all the game-making tutorials I've read over the years. So many examples I've seen use the idea of game-specific objects to illustrate use-cases for inheritance, and probably some other OOP paradigms. But after having spent a couple of years in the enterprise API world, I know this kind of approach (deeply-inherited business objects) is going to lead to problems and tech debt I'll want to clean up immediately.

I went through a few iterations while I was trying to figure out what exactly an ECS was. I started off just knowing that "an Entity is a collection of Components". Great, let's make some of the most common examples from all the ECS tutorials. A `PositionComponent` has an X and a Y value. A `HealthComponent` has some arbitrary HP value and a max HP value.

It makes enough sense that way up to that point, and then you start to have Components that share information. In comes a `RenderComponent`, and I give it a `Render()` method that can be overridden, and some data about which sprite to render. Suddenly, this Component needs to know about the `PositionComponent` on its Entity in order to know where to draw the thing. And the `HealthComponent`, in case that Entity has a health bar that needs to be drawn. So a couple of problems come to light:

* The Component needs to know about its Entity. How? This can be pretty easily solved - add an `AttachedEntity` property to the base `Component` class, and set that whenever the Component is added to an Entity.
* You're going to need a lot of different `RenderComponent` implementations, or one _stupidly complicated_ one, in order to account for all the different ways in which Entities could be drawn. Plus, if you have a lot of implementations of `RenderComponent`, it's going to be way more complicated to figure out which Entities you actually need to draw at runtime. And we definitely don't want excess complexity in the draw loop, if it can be avoided.

So I backed off of that, and considered another approach. What if some of the common, tightly-looped stuff was stored on the Entity instead? I tried putting position and rendering data on the Entity for just that purpose, but quickly came to dislike it. Now my Component-processing code had to be supplemented by this chunk of code that processed not-Component-but-still-important data. Plus, any future data that needed cross-Component would end up the Entity, so... why even have Components at all?

At about this point, I realized I had no idea what the 'S' in ECS was even used for. What was a "System"? How did it fit in alongside Entities and Components? I came across [this gist](https://gist.github.com/LearnCocos2D/77f0ced228292676689f) how the ECS concept has been implemented over the years, and it points out that the industry has swung in the opposite direction from where I had hit my roadblock. In Martin's approach, an Entity should be just an id, or a pointer, to a bunch of Components. And _that's it_. Furthermore, Components are just data, no logic. That means the Systems of the ECS are where all the 'business logic' of your game lives.

But still, I was having trouble conceptualizing how I would move over groups of Entities (or just Components) across several Systems per frame. I was indexing all Entities in the System by the Components they had, stored in a `Dictionary<Type, List<Entity>>`. It sounded good enough to be able to get "all Entities with a PositionComponent" as an O(1) operation. It's good, but then you need to get "all Entities with a PositionComponent _and_ a HealthComponent"... suddenly that's two O(1) lookups and an O(n^2) join for _each system_ in your ECS. Per frame. Not acceptable.

Finally, a came across [this StackExchange answer](https://gamedev.stackexchange.com/questions/31473/what-is-the-role-of-systems-in-a-component-based-entity-architecture/31491#31491), which I had glossed over a couple of times already but never read thoroughly. This is what made it click for me, and judging by the comments I'm not alone. The key/lock analogy is fantastic, and I knew it was time to break out my favorite trick in the book - bitmasks! Michael suggests just that in his explanation, but not wanting to settle for just 64 bits with which to register my Components (and perpetually guilty of ignoring [YAGNI](https://en.wikipedia.org/wiki/You_aren%27t_gonna_need_it)) I ended up with some scalable `int[]` bitmask solution where the mask extends as far as it needs to when new components are added.

That's pretty much what I needed to finally pull together an ECS that does what it's supposed to do. I was able to write a unit test suite and a couple of benchmarks to make sure I wasn't completely off-base in my design. I can draw a thing on a screen with it. At some point I'll push the code up here for reference. Overall, it was a fun exercise, and I'm pretty happy with the resulting API.

Most imporantly, I really like the name that came out of it: **Ecosystem**. Come on, **E**ntity **Co**mponent **System**? How has nobody thought of that? Really, I googled to make sure nobody had thought of that, because I was embarassingly proud of this name. I stopped just short of renaming my Entities to "Organisms" and my Components to "Organs", remembering that there is a good reason I'm not using my Biology degree in my career.

Oh, one more cool thing I came across. I wrote my Entity methods so that they could be used with a fluent syntax, so in one of my benchmarks I had this code (which I generated in Google Sheets, this is a super-terrible idea outside of a unit test):

```
entity
  .AddComponent<BulkComponent0001>()
  .AddComponent<BulkComponent0002>()
  ...
  .AddComponent<BulkComponent0999>()
  .AddComponent<BulkComponent1000>();
```

Wouldn't you know it - this code throws a StackOverflowException. It was easy enough to fix, just rewrite it to (er, change the spreadsheet so it generates) the following:

```
entity.AddComponent<BulkComponent0001>();
entity.AddComponent<BulkComponent0002>();
...
entity.AddComponent<BulkComponent0999>();
entity.AddComponent<BulkComponent1000>();
```
Apparently this is a [known limitation of the compiler](https://github.com/dotnet/roslyn/issues/9795). I really thought the compiler would just rewrite the first example to be like the second example. But the more I think about it, the harder that sounds to solve in a generic, non-hacky way. I'm not holding my breath for a fix for that one. After all, I'm not gonna need it, right?
