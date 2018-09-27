---
layout: blog-post
title: Entity Framework
---
I haven't used Entity Framework much, despite working primarily with C# and .NET for the past few years. A majority of my database experience has been with document-based databases, such as [MongoDB](https://www.mongodb.com/) and its corresponding [C# driver](http://mongodb.github.io/mongo-csharp-driver/?jmp=docs). Weird, right?

I had my chance to get up to speed with EntityFrameworkCore recently when working with [veekun's Pokemon database](https://veekun.com/), which I was turning into a containerized Web API that I could query for this project I'm working on. I just needed some queryable sample data, so I figured I'd have fun with it. And even though my RDBMS is a little rusty, I figured I should be able to pick up EF in no time. But there is a _lot_ of subtlety and magic going on behind the scenes in EF that I was not aware of. Here's a brief story of how that cost me a few hours.

The API that I was building seemed pretty simple. I wanted to be able to query a list of Pokemon by their _Types_ and the _Pokedex_ in which they originated, along with some basic paging with _Skip_ and _Take_. For example, give me 10 "Fire, Water, or Grass" pokemon that appeared in the "Kanto" Pokedex. It didn't need to be terribly accurate, just complex enough to give us a couple of parameters to filter on.

Here's the model I wanted to produce as JSON (nested with a `List` property of a parent model):

```csharp
public class PokemonJsonResponseItem
{
    public string Name { get; set; }

    public string Genus { get; set; }

    public Dictionary<string, int> PokedexNumbers { get; set; }

    public List<string> Types { get; set; }
}
```

The property `PokedexNumbers` would be key-value pairs of "Pokedex name" to "number within that Pokedex", such as `{ "national": 1, "kanto": 1, ... }` for Bulbasaur. And `Types` would simply be the names of the Pokemon's types. As far as I know, Pokemon can only have up to two types. But I haven't played a mainline game since [Gen 4's Diamond Version](https://bulbapedia.bulbagarden.net/wiki/Generation_IV), so don't take me as the authority on this kind of knowledge.

Since I was simply running the `.sqlite` database file on disk (it's a read-only resource, no need for persistence), it was a piece of cake to get my Entities and Context scaffolded in EFCore. Just one line:

```
dotnet ef dbcontext scaffold "Filename=./veekun-pokedex.sqlite" Microsoft.EntityFrameworkCore.Sqlite -d -o Data/Entities
```

Now all that's left to do is query the thing. Here's roughly what I started out with:

```csharp
var results = this._pokedex.Pokemon
    .Where(p => p.IsDefault == TRUE)
    .Where(p => p.Species.PokemonDexNumbers.Any(pdn => query.Pokedexes.Contains(pdn.PokedexId)))
    .Where(p => p.PokemonTypes.Any(pt => query.Types.Contains(pt.TypeId)))
    .OrderBy(p => p.Order)
    .Skip(query.Skip)
    .Take(query.Take)
    .Select(p => new PokemonJsonResponseItem
    {
        Name = p.Species.PokemonSpeciesNames.Single(x => x.LocalLanguageId == Lang.EN).Name,
        Genus = p.Species.PokemonSpeciesNames.Single(x => x.LocalLanguageId == Lang.EN).Genus,
        PokedexNumbers = p.Species.PokemonDexNumbers.ToDictionary(x => x.Pokedex.Identifier, x => (int)x.PokedexNumber),
        Types = p.PokemonTypes.Select(x => x.Type.TypeNames.Single(y => y.LocalLanguageId == Lang.EN).Name).ToList()
    });
```

I wasn't aiming for performance, but I'm totally open to criticism in that regard. Still, it looks functional to me. I let the `scaffold` command take care of setting up all the relationships and navigation properties, so from then on it seemed no different than querying any sufficiently complex object collection with LINQ. But what happened when I hit the API with my first query? NullReferenceException, because of... well, something in this mess.

I started throwing null checks all over the place to try and isolate what was going wrong, and eventually found that something in this line had to be going wrong:
```csharp
PokedexNumbers = p.Species.PokemonDexNumbers.ToDictionary(x => x.Pokedex.Identifier, x => (int)x.PokedexNumber),
```
But how? Even if I added to my query `.Where(p => p.Species != null && p.Species.PokemonDexNumbers != null && p.Species.PokemonDexNumbers.All(x => x.Pokedex != null))` (because expression trees still [don't support the null-coalescing operator](https://github.com/dotnet/roslyn/issues/2060) in 2018), I was still getting errors. And even breaking the Select statement away from the query so that I could run it through the debugger, I was finding that all navigation collections were _empty_. I knew this wasn't true just by running the raw SQL equivalent myself against the database. What gives?

I tried rewriting the whole thing with method syntax and query syntax, thinking maybe `scaffold` didn't set up my models correctly - or, rather, didn't set them up in a way that I understood. Let's be honest, I'm not better at figuring this stuff out than the toolset will be. But I figured maybe I could just manually join the data and get _something_ working. Unfortunately, the same problem resurfaced with every attempt, but at least I got to learn about [GroupJoin](https://stackoverflow.com/a/15599143) along the way.

It was about this time I started exploring how "lazy loading" works in Entity Framework, and learned that it works a little differently in EFCore. I had so far assumed that navigation properties (basically every class and `ICollection` property on a Pokemon entity) were getting auto-populated because the `scaffold` tool took care of all that work for me. But there is [more to this topic](https://docs.microsoft.com/en-us/ef/core/querying/related-data) than I realized, and you have to do a little more work to tell EF exactly how and when to load your related data. Although I was using EFCore 2.1, I opted instead for "eager loading", since it looked like the simplest solution for my case.

```csharp
var results = this._pokedex.Pokemon
    .Include(p => p.Species.PokemonDexNumbers)
        .ThenInclude(pdn => pdn.Pokedex)
    .Where(p => p.IsDefault == TRUE)
    ...
```

What still didn't quite click with me was this: howcome my first-level navigation properties seemed to be loading correctly? I never got a NullReferenceException for `p.Species` or `p.PokemonTypes`, for example. Anyway, I was hoping this would be the panacea to all my problems. It was not.

Eventually, I got [another lead](http://code-ninja.org/blog/2014/07/24/entity-framework-never-call-groupby-todictionary/). Most importantly,

> ToDictionary() is an extension method on the Enumerable class in System.Linq. It doesn’t know anything about Entity Framework – all it can do is enumerate an IEnumerable.

Suddenly, I knew what I had to try - remove ToDictionary(). I refactored my response model to this:

```csharp
public class PokemonJsonResponseItem
{
    public string Name { get; set; }

    public string Genus { get; set; }

    public List<PokedexNumberWithPokedex> PokedexNumbers { get; set; }

    public List<string> Types { get; set; }
}

public class PokedexNumberWithPokedex
{
    public string Pokedex { get; set; }

    public int Number { get; set; }
}
```

And my Select statement to this:

```csharp
PokedexNumbers = p.Species.PokemonDexNumbers.Select(x => new PokedexNumberWithPokedex
{
    Pokedex = x.Pokedex.Identifier,
    Number = (int)x.PokedexNumber
}).ToList(),
```

And wouldn't you know it, _it worked_. My understanding of the problem is that since ToDictionary is not implemented by IQueryable, it was not triggering "whatever EF magic" causes those collection properties to get loaded, even if you specify them for eager loading in your query. So the query was, in fact, returning me elements where `p.Species.PokemonDexNumbers != null` in literal database terms, but that only holds true as long as you're still operating within IQueryable. I bet this could also have been solved by using explicit loading instead of eager loading, but frankly I'm _done_ with this and I don't want to touch it for awhile.

I'm sure I'm not the first person to run into this problem, but with any luck this might help somebody, somewhere avoid a few hours of fruitless debugging and query-mangling down the road. StackOverflow, feel free to use that as your mission statement.