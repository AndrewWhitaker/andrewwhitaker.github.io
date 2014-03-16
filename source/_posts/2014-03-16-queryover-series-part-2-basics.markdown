---
layout: post
title: "QueryOver Series - Part 2: Basics and Joining"
date: 2014-03-16 12:05:33 -0400
comments: true
categories: 
---

In this post, I'll outline some basics on QueryOver, including the NHibernate types involved and basic query structure. I'll also talk about joining using `JoinAlias` and `JoinQueryOver`

#### `IQueryOver<TRoot, TSubType>`

If you look closely at the types involved when writing QueryOver queries, you'll notice that there are two generic type parameters: `TRoot` and `TSubType`. Why would the API need two type parameters?

When you create a QueryOver object using `session.QueryOver<TRoot>`, `TRoot` and `TSubType` are the same. `TRoot` stays the same as you build the query, and `TSubType` changes as you use `JoinQueryOver` to join to other tables. This is worth mentioning before we go into more depth on building queries.

In general:

* Operations *except* for `.Select` use `TSubType` as the type parameter for lambda expressions you pass to QueryOver methods
* `TRoot` is the type parameter for lambda expressions you use in the `.Select` step of the query.

Here's an example:

``` csharp
session.QueryOver<Person>()
// TRoot and TSubType are Person                         
    .JoinQueryOver(p => p.Addresses)
    // TRoot is Person, TSubtype is Address
        .Where(a => a.ModifiedDate == DateTime.Now)
        // Where accepts an Expression<Func<TSubtype, bool>>
        .Select(p => p.Id);
        // Select accepts an Expression<Func<TRoot, object>>
```

#### `JoinAlias` and `JoinQueryOver`

Since we're on the subject of `TRoot` and `TSubType`, now is a good opportunity to talk about `JoinAlias` and `JoinQueryOver`

##### `JoinAlias`

`JoinAlias` adds a join to your query without changing `TSubType`. This is useful if you're joining to more than one table from a source table:

``` csharp
BusinessEntityAddress addressAlias = null;
BusinessEntityContact contactAlias = null;

session.QueryOver<Person>()
    .JoinAlias(p => p.Addresses, () => addressAlias)
    .JoinAlias(p => p.Contacts, () => contactAlias)
    .Where(p => p.FirstName == "Andrew")
    /* etc */
```

In this example, since we want to join to `Address` and `Contact`, we can use `JoinAlias` twice, since `TSubType` doesn't change.

The second parameter (in this particular overload) to `.JoinAlias` is a lambda expression that creates an alias. You can use the alias throughout the query in other lambda expressions when you don't want to use either `TRoot` or `TSubType`.

##### `JoinQueryOver`

`JoinQueryOver` adds a join to your query and *changes* `TSubType`. You can also use an alias with `JoinQueryOver`, but with simpler queries it's often not necessary:

``` csharp
session.QueryOver<Person>()
    .JoinQueryOver(p => p.Addresses)
        .Where(a => a.ModifiedDate == DateTime.Now)
    .Select(p => p.FirstName)
```

Notice that the lambda expression passed to the `.Where` method has the signature `Expression<Func<Address, bool>>`. The call to `JoinQueryOver` changed `TSubType`. This can make for more concise, easier to read code, since you don't need to declare aliases for every join you do.

`JoinAlias` and `JoinQueryOver` are interchangeable, as far as I know. You can write the same query using either one, it's just that the number of aliases you declare changes based on which you use. Typically I choose whichever one avoids creating and managing more aliases. This means using `JoinQueryOver` when possible, but that's personal preference.

#### Summary:

* `IQueryOver` is a generic type with two type parameters `TRoot` and `TSubType`
* `.Select` operates on `TRoot` while other QueryOver methods operate on `TSubType`.
* `TRoot` stays the same as you're building a query, but `TSubType` changes when you join using `JoinQueryOver`
* `JoinQueryOver` and `JoinAlias` add joins to your query. `JoinAlias` doesn't change `TSubType`, but `JoinQueryOver` does.
* You can use aliases when building a query to refer to properties that don't belong to `TRoot` or `TSubType`
