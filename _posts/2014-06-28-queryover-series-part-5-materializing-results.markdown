---
layout: post
title: "QueryOver Series - Part 5: Materializing Results"
date: 2014-06-28 10:47:22 -0400
comments: true
categories: [NHibernate, QueryOver, QueryOver Series]
---

So far I've been using the `.List` or `.SingleOrDefault` methods to actually get results from a query. In this post I'll go into a little more detail about those methods. I'll also cover other ways you can materialize a query's results.
<!-- more -->

{% include queryover-series/queryover_series_partial.markdown %}

### `SingleOrDefault<T>` and `List<T>`

These two are pretty self-explanatory. Calling `.List<T>` at the end of your query will immediately yield an `IList<T>`. For example:

```csharp
IList<Employee> empl = session.QueryOver<Employee>()
    .List<Employee>();
```

Similarly, `SingleOrDefault<T>` will immediately give you a single item of type `T`, provided the query only returns one row. Otherwise, NHibernate will throw an exception telling you that the query did not return a unique result.

### `Future<T>` and `FutureValue<T>`

These two are more interesting. [Ayende has discussed these in detail](http://ayende.com/blog/3979/nhibernate-futures) with the Criteria API, but I don't think a blog series on QueryOver is really complete without mentioning them again. In NHibernate using these creates a "MultiQuery" or "MultiCriteria", which just means that NHibernate is batching your queries into one round trip and deferring populating the results.

#### `Future<T>`

Calling `.Future<T>` on your query will return an `IEnumerable<T>` that will not be populated with results until one of the results or resultsets is needed. NHibernate will build up queries using `.Future<T>` or `.FutureValue<T>` until one of the "future" results are accessed. 

When the results are processed, NHibernate will send one request to the database containing the SQL for all of the "future" queries and populate the results accordingly. For example, lets say we were trying to display all `Employee`s and all `Product`s on the same page. We could just write two separate queries:

```csharp
// Get the list of employees first
IList<Employee> employees = session.QueryOver<Employee>()
    .List<Employee>();

// Then the list of products
IList<Product> products = session.QueryOver<Product>()
    .List<Product>();
```

Which yields the following SQL:

```sql
    SELECT
        this_.BusinessEntityID as Business1_0_1_,
        -- More employee fields
    FROM
        HumanResources.Employee this_
```
... And then a second query for the `Product`s:
```sql
    SELECT
        this_.ProductID as ProductID7_0_
        -- More product fields
    FROM
        Production.Product this_
```
This would work fine, but NHibernate is sending two separate queries to the database when we can accomplish getting those results with only one using `.Future<T>`:

```csharp
// Batch the two queries together
IEnumerable<Employee> employees = session.QueryOver<Employee>()
    .Future<Employee>();

IEnumerable<Product> products = session.QueryOver<Product>()
    .Future<Product>();

// A single round trip to the database is made here containing both queries
int numProducts = products.Count();
```

This generates a single database round trip with both queries when the `products.Count()` call forces NHibernate to process the `Product`s query:

```sql
-- First query:
SELECT
    this_.BusinessEntityID as Business1_0_1_,
    -- More Employee columns
FROM
    HumanResources.Employee this_;
-- Second query:
SELECT
    this_.ProductID as ProductID7_0_,
    -- More product columns
FROM
    Production.Product this_;
```

This can be hugely helpful when querying from multiple sources that don't depend on each other.

#### `FutureValue<T>`

The same applies for `.FutureValue`, which is a deferred version of `.SingleOrDefault`. 

Lets say we wanted a list of all employees and the total number of products. Again, we could write two separate queries:

```csharp
IList<Employee> employees = session.QueryOver<Employee>()
    .List<Employee>();

int productsCount = session.QueryOver<Product>()
    .SelectList(list => list
        .SelectCount(pr => pr.Id))
    .SingleOrDefault<int>();
```

As you'd expect, this will create two queries and two round-trips to the database:

```sql
SELECT
    this_.BusinessEntityID as Business1_0_1_,
    -- More Employee columns
FROM
    HumanResources.Employee this_
```
... And then the `Product`s count query:
```sql
    SELECT
        count(this_.ProductID) as y0_
    FROM
        Production.Product this_
```

We can turn this into one query by using `Future` and `FutureValue`:

```csharp
// Batch the two queries together
IEnumerable<Employee> employees = session.QueryOver<Employee>()
    .Future<Employee>();

IFutureValue<int> productsCount = session.QueryOver<Product>()
    .SelectList(list => list
        .SelectCount(pr => pr.Id))
    .FutureValue<int>();

// Access the "Value" property of IFutureValue, which will execute both queries in one round-trip
Console.WriteLine(productsCount.Value);
```

Like our previous example with `.Future`, this will generate one round-trip to the database with two queries:

```sql
-- First query:
SELECT
    this_.BusinessEntityID as Business1_0_1_,
    -- More Employee Columns
FROM
    HumanResources.Employee this_;
-- Second query:
SELECT
    count(this_.ProductID) as y0_
FROM
    Production.Product this_;
```

Note the return type of the count query. `IFutureValue<T>` is simply a type that allows NHibernate to give you a deferred `SingleOrDefault` result. Accessing `.Value` (just like causing the `IEnumerable<T>` returned by `.Future`) will cause the batched queries to execute.

I'd highly recommend using `.Future` and `.FutureValue` where possible when you need to execute multiple queries at once. You'll save round trips to the database and therefore get results to your users faster.

### Summary
This post covered different ways to materialize resultsets with NHibernate QueryOver.

* `.SingleOrDefault` and `.List` immediately give you the results
* `.FutureValue` and `.Future` batch queries and defer execution until one of the results in the batch is needed
* `.FutureValue` and `.Future` use one round trip to the database instead of several, which is more efficient.
