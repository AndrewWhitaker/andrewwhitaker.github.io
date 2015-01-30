---
layout: post
title: "QueryOver Series - Part 4: Transforming"
date: 2014-06-19 17:51:00 -0500
comments: true
categories: [NHibernate, QueryOver, QueryOver Series]
---

You might have noticed that the last post in the series always projects each result row into an `object[]`. This might have made you wonder if there's a better way to get results from a QueryOver query. Well there is! It's called transforming.

In the context of an NHibernate query, a *transformer* is simply a class that transforms each row from a query into an instance of an object. NHibernate comes with several and allows you to easily create a custom transformer if you'd like.
<!-- more -->

{% render_partial queryover-series/queryover_series_partial.markdown %}

Transformers are supplied to the `TransformUsing` function on an instance of `IQueryOver<TRoot, TSubtype>`. For example, here's how you would use `Transformers.DistinctRootEntity` (which I'll go into more detail later about):

```csharp
var results = 
    session.QueryOver<Product>()
        .TransformUsing(Transformers.DistinctRootEntity)
        .List<Product>();
```

### Using the built-in transformers

NHibernate supplies several built-in transformers in the `NHibernate.Transform` namespace. These may be all you need in your application since they cover most use cases. I'll go over each built-in transformer and how to use it.

#### `DistinctRootEntity`

This transformer works the way you'd think it would: it transforms the query results into a list of *distinct* entities of the *root* type. What's the *root* type? Well if you read [part 1](../../../../2014/03/12/queryover-series-part-1-why-queryover/), you'll remember that a QueryOver query deals with two types, `TRoot` and `TSubType`. the root type is simply `TRoot`.

For example, here's a query that returns a list of all `Product`s:

```csharp
// TRoot is Product
IList<Product> results = session.QueryOver<Product>()
    .TransformUsing(Transformers.DistinctRootEntity)
    .List<Product>();
```
        
As you can see, using `DistinctRootEntity` allows us to get a list of entities easily. This example doesn't address the *distinct* part of `DistinctRootEntity`. Here's another, more interesting example:

```csharp
IList<Product> results = session.QueryOver<Product>()
    .JoinQueryOver(pr => pr.TransactionHistory)
        .Where(th => th.ActualCost > 2.0M)
    .TransformUsing(Transformers.DistinctRootEntity)
    .List<Product>();
```
        
This is more interesting because a `Product` might have many related rows in `TransactionHistory`. The join would cause each `Product` to appear as many times as it has `TransactionHistory` records, which we probably don't want if we're just trying to find all `Product`s that were ever priced over $2.00.

Here's the SQL the above query generates:

```sql
SELECT
    this_.ProductID as ProductID7_1_,
    -- All product columns
    transactio1_.TransactionID as Transact1_13_0_,
    -- All TransactionHistory columns
FROM
    Production.Product this_
inner join
    Production.TransactionHistory transactio1_
        on this_.ProductID=transactio1_.ProductID
WHERE
    transactio1_.ActualCost > 2;
```
        

The result we  get back is a list of distinct `Product`s.

`DistinctRootEntity` is most useful if you have a simple query in which you need instances of an entity and may or may not want to do filtering on some related entities.

#### `AliasToEntityMap`

This transformer allows you to transform each row of the result set into an `IDictionary` (hash table). Unfortunately it's not a generic `IDictionary`. The keys are strings containing the aliases you defined in the query, and the values are entities. This is best explained with an example:

``` csharp
TransactionHistory historyAlias = null;
Product productAlias = null;

IList<IDictionary> results = session.QueryOver<Product>(() => productAlias)
    .JoinQueryOver(pr => pr.TransactionHistory, () => historyAlias)
        .Where(th => th.ActualCost > 2.0M)
    .TransformUsing(Transformers.AliasToEntityMap)
    .Take(10)
    .List<IDictionary>();
```

Each item in `results` is an `IDictionary`. This `IDictionary`'s keys are the *aliases* we assigned while building our query. For example, if you wanted to get the first row's `TransactionHistory` entity, you would write:

```csharp
TransactionHistory history = (TransactionHistory)results[0]["historyAlias"];
```

This might seem a bit odd at first, but using `AliasToEntityMap` can prove useful if you need to retrieve multiple entities in a single query.

#### `PassThrough`

This transformer appears to be quite similar to `AliasToEntityMap` in that it generates a collection of entities for each row in the resultset. I say "appears" because I haven't had much experience with it and I cannot find much about it online. I'll add to this post if I come across anything interesting.

Anyway for a simple example it seems to place an instance of an entity from the query in a slot in an `object` array in *reverse* order from when it was added to the query. For example:

```csharp
IList<object[]> results = session.QueryOver<Product>()
    .JoinAlias(pr => pr.Reviews, () => reviewAlias)
    .JoinQueryOver(pr => pr.TransactionHistory)
        .Where(th => th.ActualCost > 2.0M)
    .TransformUsing(Transformers.PassThrough)
    .Take(10)
    .List<object[]>();


foreach (object[] result in results)
{
    ProductReview review = (ProductReview)result[0];
    TransactionHistory t = (TransactionHistory)result[1];
    Product p = (Product)result[2];
}
```

As you can see, `result[0]` is a `ProductReview`, `result[1]` is a `TransactionHistory` and `result[2]` is the `Product` itself.

#### `RootEntity`

`RootEntity` is similar to `DistinctRootEntity` in that it projects a list of `TRoot`. The difference is that the results are *not* distinct. Therefore if you join on a related table that multiplies the root entity, you'll get back that entity many times for each related row. Here's the example from `DistinctRootEntity` again, except using `RootEntity`:

```csharp
IList<Product> results = session.QueryOver<Product>()
    .JoinQueryOver(pr => pr.TransactionHistory)
        .Where(th => th.ActualCost > 2.0M)
    .TransformUsing(Transformers.RootEntity)
    .List<Product>();
```

This will return any `Products` with a `TransactionHistory` that has an `ActualCost` over $2.00, but will not remove duplicate `Product` records.

#### `ToList`

This transformer works very similarly to not specifying a transformer at all and getting back an `IList<object[]>`. The difference here is that you'll get back an `IList<IList>` instead.

For example:

```csharp
Product productAlias = null;

IList<IList> results = session.QueryOver<Product>(() => productAlias)
    .JoinQueryOver(pr => pr.TransactionHistory)
        .Where(th => th.ActualCost > 2.0M)
    .TransformUsing(Transformers.ToList)
    .SelectList(list => list
        .Select(() => productAlias.Id)
        .Select(() => productAlias.Name)
    )
    .List<IList>();

Console.WriteLine(results[0][0]); // product Id
Console.WriteLine(results[0][1]); // product Name
```

#### `AliasToBean`

In my experience, this transformer is by far the most useful. It allows you to transform each row into an instance of a type you specify. You can project columns from different entities into properties on each instance.

Lets use `AliasToBean` to get a list of `HighestProductReviewDTO`s. Here's the definition for `HighestProductReviewDTO`:

```csharp
public class HighestProductReviewDTO
{
    public int ProductID { get; set; }

    public string ProductName { get; set; }

    public int Rating { get; set; }

    public string Comments { get; set; }
}
```

NHibernate requires that the DTO have a parameterless constructor so that it can create an instance of your class for each row it retrieves.

We're going to get a list of `Product`s that have reviews, followed by some information from that `Product`'s highest review. Here's what our query looks like:

```csharp
IList<HighestProductReviewDTO> highestReviews =
    session.QueryOver<Product>(() => productAlias)
        .JoinQueryOver(pr => pr.Reviews, () => productReviewAlias)
            .WithSubquery.Where(pr => pr.Id == QueryOver.Of<ProductReview>()
                .Where(rev => rev.Product.Id == productAlias.Id)
                .OrderBy(rev => rev.Rating).Desc()
                .Select(rev => rev.Id)
                .Take(1)
                .As<int>())
        .SelectList(list => list
            .Select(() => productAlias.Id).WithAlias(() => result.ProductID)
            .Select(() => productAlias.Name).WithAlias(() => result.ProductName)
            .Select(() => productReviewAlias.Rating).WithAlias(() => result.Rating)
            .Select(() => productReviewAlias.Comments).WithAlias(() => result.Comments)
        )
        .TransformUsing(Transformers.AliasToBean<HighestProductReviewDTO>())
        .List<HighestProductReviewDTO>();
```

Pay particular attention to the `.WithAlias` calls at the end of the `.Select` calls inside of `SelectList`. These are what tell NHibernate to associate particular column values in each row retrieved with the correct property in our DTO class.

In case you're curious, here's the SQL that NHibernate generated:

```sql
SELECT
    this_.ProductID as y0_,
    this_.Name as y1_,
    productrev1_.Rating as y2_,
    productrev1_.Comments as y3_
FROM
    Production.Product this_
inner join
    Production.ProductReview productrev1_
        on this_.ProductID=productrev1_.ProductID
WHERE
    productrev1_.ProductReviewID = (
        SELECT
            TOP (1)  this_0_.ProductReviewID as y0_
        FROM
            Production.ProductReview this_0_
        WHERE
            this_0_.ProductID = this_.ProductID
        ORDER BY
            this_0_.Rating desc
```

`AliasToBean` is extremely useful. It allows us to specify exactly what columns we need and transform the resulting rows into instances of simple types. However it does have some limitations:

* The class you're projecting to must have a parameterless constructor
* You cannot populate collections (e.g., if you had a class with `ProductID` and a collection of `ProductReviews` you could not do that in one step using `AliasToBean`)
* You cannot populate full entities (e.g., `.Select(() => productReview.Product).WithAlias(() => result.Product)`)

While the second limitation is unfortunate, you *can* specify a collection type in your result class and write a separate query to populate it. You can possibly even do this in one database round trip using the `.Future` method, which I'll talk about in a later post.

#### `AliasToBeanConstructor`

`AliasToBeanConstructor` is similar to `AliasToBean`, except that it uses a result type's constructor to create new objects from result rows. Here's our example from above slightly modified to use `AliasToBeanConstructor` instead.

Here's our modified result class:

```csharp
public class HighestProductReviewDTO
{
    public HighestProductReviewDTO(
        int productId, string productName, int rating, string comments)
    {
        this.ProductID = productId;
        this.ProductName = productName;
        this.Rating = rating;
        this.Comments = comments;
    }

    public int ProductID { get; private set; }

    public string ProductName { get; private set; }

    public int Rating { get; private set; }

    public string Comments { get; private set; }
}
```

And here's our new QueryOver query:

```csharp
IList<HighestProductReviewDTO> highestReviews =
    session.QueryOver<Product>(() => productAlias)
        .JoinQueryOver(pr => pr.Reviews, () => productReviewAlias)
            .WithSubquery.Where(pr => pr.Id == QueryOver.Of<ProductReview>()
                .Where(rev => rev.Product.Id == productAlias.Id)
                .OrderBy(rev => rev.Rating).Desc()
                .Select(rev => rev.Id)
                .Take(1)
                .As<int>())
        .SelectList(list => list
            .Select(() => productAlias.Id)
            .Select(() => productAlias.Name)
            .Select(() => productReviewAlias.Rating)
            .Select(() => productReviewAlias.Comments)
        )
        .TransformUsing(Transformers.AliasToBeanConstructor(
            typeof(HighestProductReviewDTO).GetConstructors().First()))
        .List<HighestProductReviewDTO>();
```

We're passing [`ConstructorInfo`][1] to `AliasToBeanConstructor` which we get using `GetConstructors`. NHibernate calls our constructor with the column values we're retrieving with our `SelectList`. Note that *all* items in the `SelectList` are passed to the constructor in the order you add them.

### Creating your own transformer

The built in transformers are great, but if you need your own result transformer, that's possible too.

For example, let's say we want to call a callback function every time a row is transformed. We could also iterate over our results after retrieving them, but this gives us a way to apply any modifications we might want while we're transforming the row. Here's our new transformer class:

```csharp
/// <summary>
/// A result transformer that calls a callback after successfully transforming a result row 
/// into an instance of T
/// </summary>
/// <typeparam name="T">The result type</typeparam>
public class AliasToBeanWithCallbackTransformer<T> : IResultTransformer
{
    private readonly AliasToBeanResultTransformer aliasToBeanTransformer;
    private readonly Action<T> callback;

    public AliasToBeanWithCallbackTransformer(Action<T> callback)
    {
        this.aliasToBeanTransformer = new AliasToBeanResultTransformer(typeof(T));
        this.callback = callback;
    }

    public IList TransformList(IList collection)
    {
        return this.aliasToBeanTransformer.TransformList(collection);
    }

    public object TransformTuple(object[] tuple, string[] aliases)
    {
        object result = this.aliasToBeanTransformer.TransformTuple(tuple, aliases);
        
        // Call the callback before returning the result.
        callback((T)result);

        return result;
    }
}
```

In this example, all I've done is wrap `AliasToBeanResultTransformer` in a class that calls the callback the user specifies after calling `AliasToBeanResultTransformer`'s `TransformTuple` method. I'll use this transformer in an example that retrieves product review information but with an added property, `DateRetrieved`:

```csharp
public class ProductReviewDTO
{
    public int ProductReviewID { get; set; }

    public int Rating { get; set; }

    public string Comments { get; set; }

    public DateTime DateRetrieved { get; set; }
}
```

We can use the transformer to assign `DateRetrieved` after creating a new `ProductReviewDTO`:

```csharp
DateTime dateRetrieved = DateTime.Now;

IList<ProductReviewDTO> highestReviews =
    session.QueryOver<ProductReview>()
        .SelectList(list => list
            .Select(pr => pr.Comments).WithAlias(() => result.Comments)
            .Select(pr => pr.Id).WithAlias(() => result.ProductReviewID)
            .Select(pr => pr.Rating).WithAlias(() => result.Rating)
        )
        // Assign "DateRetrieved correctly:
        .TransformUsing(new AliasToBeanWithCallbackTransformer<ProductReviewDTO>(
            hp => hp.DateRetrieved = dateRetrieved))
        .Take(10)
        .List<ProductReviewDTO>();
```

This is a simple example, but it should demonstrate how easy it is to extend the built in transformers. It would be nice if we could subclass the built in transformers, but unfortunately the methods we would need to override are not marked `virtual`.

A good place to look for how to write a transformer is the [NHibernate source code itself](https://github.com/nhibernate/nhibernate-core/tree/master/src/NHibernate/Transform).
        

### Summary

I covered a lot in this post, but I was aiming to be comprehensive with each transformer type. This should enable you to effectively use the built in transformers and create your own if you need to.

* There are several built in result transformers in the `NHibernate.Transform` namespace.
* `DistinctRootEntity` and `RootEntity` retrieve a list of the "root" of the QueryOver query
* `AliasToEntityMap` and `PassThrough` retrieve the entities present in the QueryOver query in an `IDictionary` and `object[]`, respectively.
* `AliasToBean` and `AliasToBeanConstructor` are powerful transformers that allow you to create a list of instances of a type you specify.
* You can create your own result transformer pretty easily to suit your needs.
        
        
[1]:http://msdn.microsoft.com/en-us/library/system.reflection.constructorinfo(v=vs.110).aspx
