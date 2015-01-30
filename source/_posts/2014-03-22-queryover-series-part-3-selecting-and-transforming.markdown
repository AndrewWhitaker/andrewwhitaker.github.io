---
layout: post
title: "QueryOver Series - Part 3: Selecting"
date: 2014-03-22 11:01:46 -0400
comments: true
categories: [NHibernate, QueryOver, QueryOver Series]
---

In this post I'll go over building the `SELECT` statement with NHibernate QueryOver. I'll also cover the different ways you can actually get a result back from your query.
<!-- more -->

{% render_partial queryover-series/queryover_series_partial.markdown %}

### Selecting a single property

In the simplest case, you'll want to select a single column from a single row. For example, if you wanted to retrieve a single `Product`'s `Name`:

``` csharp
string name = session.QueryOver<Product>()
    .Where(p => p.Id == 1)
    .Select(p => p.Name)
    .SingleOrDefault<string>();

```

Which yields the following SQL:

``` sql
SELECT
    this_.Name as y0_
FROM
    Production.Product this_
WHERE
    this_.ProductID = 1;
```

Note that if your query actually returns more than one result, NHibernate will throw an exception, letting you know that the query did not return a unique result.

Similarly, if you want to select a list of single properties, say the `Name` of every `Product`:

``` csharp
IList<string> names = session.QueryOver<Product>()
    .Select(p => p.Name)
    .List<string>();
```

Generates:

``` sql
SELECT
    this_.Name as y0_
FROM
    Production.Product this_
```

### Selecting multiple properties

Most of the time you won't want to select just one column, you'll want to build a whole result set. You have a few options in this area:

#### Using `SelectList`

`SelectList` is one way to specify a list of properties you'd like to select. Here's a simple example:

``` csharp
IList<object[]> productInformation = session.QueryOver<Product>()
    .SelectList(list => list
        .Select(p => p.Id)
        .Select(p => p.Name)
        .Select(p => p.StandardCost)
    )
    .List<object[]>();
```

This generates the SQL you'd expect:

``` sql
SELECT
    this_.ProductID as y0_,
    this_.Name as y1_,
    this_.StandardCost as y2_
FROM
    Production.Product this_
```

Those are the basics of using `SelectList`. There are some cool things you can do with `SelectList` to build `SELECT` clauses dynamically.

`SelectList` accepts a `QueryOverProjectionBuilder<TRoot>`. We can take advantage of QueryOver's dynamic nature to dynamically build a select list. 

One way to do this is to create a method that accepts a `QueryOverProjectionBuilder<TRoot>` and has the same return type. To expand on the `Product` example above:

``` csharp
static QueryOverProjectionBuilder<Product> BuildSelectList(
    QueryOverProjectionBuilder<Product> list)
{
    bool getName = /* some condition */;

    if (getName)
    {
        list.Select(p => p.Name);
    }

    list
        .Select(p => p.Id)
        .Select(p => p.StandardCost);

    return list;
}
```

We can then call the method directly from `SelectList`:

``` csharp
IList<object[]> names = session.QueryOver<Product>()
    .SelectList(BuildSelectList)
    .List<object[]>();
```
#### Using `Projections.ProjectionList()`

Another way to build a `SELECT` clause is using `Projections.ProjectionList()`. You can pass a `ProjectionList` to the `.Select` method:

``` csharp
Product productAlias = null;

session.QueryOver<Product>(() => productAlias)
    .Select(Projections.ProjectionList()
        .Add(Projections.Property(() => productAlias.Id))
        .Add(Projections.Property(() => productAlias.Name))
    )
    .List<object[]>();
```

This generates the following SQL:

``` sql
SELECT
    this_.ProductID as y0_,
    this_.Name as y1_
FROM
    Production.Product this_
```

It's also easy to generate dynamic `SELECT` clauses with `ProjectionList`:

``` csharp
Product productAlias = null;

ProjectionList projectionList = Projections.ProjectionList()
    .Add(Projections.Property(() => productAlias.Id))
    .Add(Projections.Property(() => productAlias.StandardCost));

bool getName = true;

if (getName)
{
    projectionList.Add(Projections.Property(() => productAlias.Name));
}

session.QueryOver<Product>(() => productAlias)
    .Select(projectionList)
    .List<object[]>();
```

I think if you're dynamically building the `SELECT` clause, `Projections.ProjectionList` is actually cleaner, due to the way you can easily build it outside of the query itself.

### Aggregates

So far I've looked at building simple `SELECT`s. Now I'll look at using aggregate functions.

In the simplest cases, using `SelectList` along with `SelectGroup` and the aggregate function you want will get the job done.

For example:

``` csharp
session.QueryOver<Product>()
    .JoinQueryOver(pr => pr.TransactionHistory, () => transactionHistoryAlias)
    .SelectList(list => list
        .SelectGroup(pr => pr.Id)
        .SelectCount(() => transactionHistoryAlias.Id)
    )
    .List<object[]>();
```

Will generate:

``` sql
SELECT
    this_.ProductID as y0_,
    count(transactio1_.TransactionID) as y1_
FROM
    Production.Product this_
inner join
    Production.TransactionHistory transactio1_
        on this_.ProductID=transactio1_.ProductID
GROUP BY
    this_.ProductID
```

You can call `SelectGroup` multiple times to add more columns to group on. You'll notice that `.SelectGroup` adds a column to the `GROUP BY` clause as well as the `SELECT` clause.

You can also add a `HAVING` clause, although it is *not* intuitive at all:

``` csharp
var results = session.QueryOver<Product>()
    .JoinQueryOver(pr => pr.TransactionHistory, () => transactionHistoryAlias)
    .SelectList(list => list
        .SelectGroup(pr => pr.Id)
        .SelectGroup(pr => pr.Name)
        .SelectCount(() => transactionHistoryAlias.Id)
    )
    /* Generates a HAVING clause: */
    .Where(Restrictions.Gt(
        Projections.Count(
            Projections.Property(() => transactionHistoryAlias.Id)), 5))
    .List<object[]>();
```

This generates the following SQL:

``` sql
SELECT
    this_.ProductID as y0_,
    this_.Name as y1_,
    count(transactio1_.TransactionID) as y2_
FROM
    Production.Product this_
inner join
    Production.TransactionHistory transactio1_
        on this_.ProductID=transactio1_.ProductID
GROUP BY
    this_.ProductID,
    this_.Name
HAVING
    count(transactio1_.TransactionID) > 5;
```

### Subqueries

There are several ways to create subqueries. You can create a correlated subquery by creating an alias in the outer query and referencing it in the other query. Here's an example using `SelectList` and `SelectSubQuery`:

``` csharp
var results = session.QueryOver<Product>(() => productAlias)
    .SelectList(list => list
        .Select(pr => pr.Id)
        .SelectSubQuery(
            QueryOver.Of<TransactionHistory>()
                // Creates a correlated subquery
                .Where(tx => tx.Product.Id == productAlias.Id)
                .OrderBy(tx => tx.TransactionDate).Asc
                .Select(tx => tx.TransactionDate)
                .Take(1)
            )
    )
    .List<object[]>();
```

Which generates:

``` sql
SELECT
   this_.ProductID as y0_,
   (SELECT
       TOP (1)  this_0_.TransactionDate as y0_
   FROM
       Production.TransactionHistory this_0_
   WHERE
       this_0_.ProductID = this_.ProductID
   ORDER BY
       this_0_.TransactionDate asc) as y1_
FROM
   Production.Product this_;
```

In general, if you can't find a method on `QueryOverProjectionBuilder<TRoot>` using `.SelectList`, you can drop back into criteria methods on the `Projections` class. For example, say you want to use a case statement in your `SELECT` clause. You can use `Projections.Conditional` for that:

``` csharp
var results = session.QueryOver<Product>(() => productAlias)
    .JoinQueryOver(pr => pr.TransactionHistory, () => transactionHistoryAlias)
    .SelectList(list => list
        .Select(pr => pr.Id)
        .Select(Projections.Conditional(
            Restrictions.Gt(
                Projections.Property(() => transactionHistoryAlias.Quantity), 5),
            Projections.Constant(true),
            Projections.Constant(false)
    )))
    .List<object[]>();
```

Which generates:

``` sql
SELECT
    this_.ProductID as y0_,
    (case
        when transactio1_.Quantity > 5 then 'True'
        else 'False'
    end) as y1_
FROM
    Production.Product this_
inner join
    Production.TransactionHistory transactio1_
        on this_.ProductID=transactio1_.ProductID;
```

### Summary

This post covered a lot, but that's because there are many ways to build a `SELECT` clause with QueryOver. In summary:

* `Select` can be used to build a `SELECT` clause with single columns
* `SelectList` and `Projections.ProjectionList` can be used to create more complex `SELECT` clauses.
* When aggregating values, use `SelectGroup` (or `Projections.GroupProperty`).
* For more complex scenarios, you can drop back in to Criteria methods on the `Projections` class. These support lambda expressions and can be used with QueryOver.