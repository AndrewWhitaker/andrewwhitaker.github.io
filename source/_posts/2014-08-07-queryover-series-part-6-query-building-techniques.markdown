---
layout: post
title: "QueryOver Series - Part 6: Query Building Techniques"
date: 2014-08-07 16:05:37 -0500
comments: true
categories: [NHibernate, QueryOver, QueryOver Series]
---

In this post, I'm going to explain some more advanced techniques for building queries with QueryOver. Practically, this means adding joins and where clauses dynamically. This is actually one of the most powerful abilities of QueryOver so it's worth understanding.
<!-- more -->

{% render_partial queryover-series/queryover_series_partial.markdown %}

To make this easier to explain, I'm going to use a simple example that I can build on throughout the post.

## The Problem

Imagine your company is building a page that lists all of the company's available products. On the left hand side, there are a few different filters the user can use to narrow his or her search.

You're immediately faced with a problem: How do I conditionally add where clauses and joins depending on the user's filters? If you're using a pure SQL solution, you would most likely need to build up dynamic SQL and then execute that. This will work fine, but you're immediately faced with a few other problems:

* **You're more open to SQL injection**. Instead of using a parameterized query, a programmer could modify your query to concatenate user-inputted data into the query.
* **You have a big maintainability problem**. You're going to have to deal with giant strings of SQL. This isn't fun to read or modify.

Both of these problems apply whether you're building the SQL in a stored procedure or if you're building SQL outside of the database engine (say, in your application layer).

## Solutions with QueryOver

Using QueryOver to dynamically build queries is an attractive solution because it solves both problems:

* NHibernate is building the SQL behind the scenes using parameterized queries, so we don't have to worry about SQL injection.
* Instead of looking at huge amounts of string concatenation, we're looking at more expressive .NET code. As a bonus, this is compiled with our application making it much more maintainable.

Lets take a look at how we can dynamically construct queries with QueryOver. Keeping with our example, we'll start with a base query that retrieves information about all products:

Here's the DTO we're projecting to with our queries:

```csharp
public class ProductDTO
{
    public string Name { get; set; }

    public string Color { get; set; }

    public decimal ListPrice { get; set; }
}
```

And here's the basic query we'll be building on:

```csharp
IList<ProductDTO> products = session.QueryOver<Product>()
    .SelectList(list => list
        .Select(pr => pr.Name).WithAlias(() => result.Name)
        .Select(pr => pr.Color).WithAlias(() => result.Color)
        .Select(pr => pr.ListPrice).WithAlias(() => result.ListPrice))
    .TransformUsing(Transformers.AliasToBean<ProductDTO>())
    .List<ProductDTO>();
```

### Conditional Restrictions

Continuing with our example, imagine we have a "Color" filter where the user can look for products only matching the colors they specify. Since `Product.Color` is a `string`, we'll introduce an `IEnumerable<string>` containing user-specified colors.

Adding this to our query gives us the following:

```csharp
ProductDTO result = null;

var query = session.QueryOver<Product>();

if (colors != null && colors.Any())
{
    query.Where(pr => pr.Color.IsIn(colors.ToArray()));
}

IList<ProductDTO> products = query
    .SelectList(list => list
        .Select(pr => pr.Name).WithAlias(() => result.Name)
        .Select(pr => pr.Color).WithAlias(() => result.Color)
        .Select(pr => pr.ListPrice).WithAlias(() => result.ListPrice))
    .TransformUsing(Transformers.AliasToBean<ProductDTO>())
    .List<ProductDTO>();
```

Our query has gotten a little more complicated, but it's still readable. Later on in the post, I'll show one way you can make this a bit more readable.

### Conditional Joins

Sometimes you'll need to conditionally perform a join. One reason for this would be that a query *without* the join performs much better and you don't want to join unless you have to.

Lets add another filter to our example. Each `Product` in our domain has 0 to many `ProductReview`s. Let's add a filter that allows the user to find only products with a minimum user rating. For example, this would let a user find only products that have at least one rating of 3 stars or higher.

We'll add a `minimumRating` (a `Nullable<int>` where `null` indicates that the filter is not being used) and incorporate that into our query:

```csharp
ProductDTO result = null;

var query = session.QueryOver<Product>();

if (colors != null && colors.Any())
{
    query.Where(pr => pr.Color.IsIn(colors.ToArray()));
}

if (minimumRating.HasValue)
{
    ProductReview reviewAlias = null;
    query.JoinAlias(pr => pr.Reviews, () => reviewAlias)
        .Where(() => reviewAlias.Rating >= minimumRating.Value);
}

IList<ProductDTO> products = query
    .SelectList(list => list
        .Select(pr => pr.Name).WithAlias(() => result.Name)
        .Select(pr => pr.Color).WithAlias(() => result.Color)
        .Select(pr => pr.ListPrice).WithAlias(() => result.ListPrice))
    .TransformUsing(Transformers.AliasToBean<ProductDTO>())
    .List<ProductDTO>();
```

This is still more readable than dynamic SQL, but it's starting to look hairy. If we were to add a few more filters we'd have a mess on our hands.

### Refactoring into Extension Methods

One way to apply our filters conditionally is to use [extension methods](http://msdn.microsoft.com/en-us/library/bb383977.aspx). This will allow us to retain the flow of the QueryOver query so that it's a little easier to read.

Here's a static class containing our extension methods:

```csharp
public static class ProductQueryExtensions
{
    public static IQueryOver<Product, Product> ApplyColorFilter(
        this IQueryOver<Product, Product> query,
        IEnumerable<string> colors)
    {
        if (colors != null && colors.Any())
        {
            query.Where(pr => pr.Color.IsIn(colors.ToArray()));
        }

        return query;
    }

    public static IQueryOver<Product, Product> ApplyRatingFilter(
        this IQueryOver<Product, Product> query,
        int? minimumRating)
    {
        if (minimumRating.HasValue)
        {
            ProductReview reviewAlias = null;

            query.JoinAlias(pr => pr.Reviews, () => reviewAlias)
                .Where(() => reviewAlias.Rating >= minimumRating.Value);
        }

        return query;
    }
}
```

And here's our updated query using those extension methods:

```csharp
IList<ProductDTO> products = session.QueryOver<Product>()
    .ApplyColorFilter(colors)
    .ApplyRatingFilter(minimumRating)
    .SelectList(list => list
        .Select(pr => pr.Name).WithAlias(() => result.Name)
        .Select(pr => pr.Color).WithAlias(() => result.Color)
        .Select(pr => pr.ListPrice).WithAlias(() => result.ListPrice))
    .TransformUsing(Transformers.AliasToBean<ProductDTO>())
    .List<ProductDTO>();

return products;
```

This is *much* easier to read and our filtering logic is in it's own class. As a bonus, it's reusable: we can reuse those same extension methods in another area of our application if we need to.

There are a few problems with this approach though--so let's address those.

#### The extension methods only work on an `IQueryOver<Product, Product>`

This may not seem like a problem at first, but imagine we changed our query to start at another table. Say, for example, we wanted to start at `ProductReview` and *join* to `Product` instead of *starting* at `Product`. In that case, our extension methods would be useless since we aren't working with an `IQueryOver<Product, Product>` anymore, we're working with an `IQueryOver<ProductReview, Product>`.

The solution to this problem is to slightly change our extension methods to take advantage of the fact that when we're filtering we only care about `TSubType` (see [part 2, Basics and Joining](../../../../2014/03/16/queryover-series-part-2-basics/) if you need a refresher on `TRoot` and `TSubType`):

```csharp
public static class ProductQueryExtensions
{
    public static IQueryOver<TRoot, Product> ApplyColorFilter<TRoot>(
        this IQueryOver<TRoot, Product> query,
        IEnumerable<string> colors)
    {
        if (colors != null && colors.Any())
        {
            query.Where(pr => pr.Color.IsIn(colors.ToArray()));
        }

        return query;
    }

    public static IQueryOver<TRoot, Product> ApplyRatingFilter<TRoot>(
        this IQueryOver<TRoot, Product> query,
        int? minimumRating)
    {
        if (minimumRating.HasValue)
        {
            ProductReview reviewAlias = null;

            query.JoinAlias(pr => pr.Reviews, () => reviewAlias)
                .Where(() => reviewAlias.Rating >= minimumRating.Value);
        }

        return query;
    }
}
```

Now our extension methods will work on any QueryOver query whose `TSubType` is `Product`.

#### Aliases may need to be passed around

This is a more subtle problem and requires some understanding of how NHibernate generates SQL from QueryOver code.

Remember that QueryOver is built on top of the Criteria API for querying, and is really just a strongly-typed wrapper for that API using [expression trees](http://msdn.microsoft.com/en-us/library/bb397951.aspx).

What this means for aliases is that NHibernate is using the name of the variable you're using as an alias--parsing the expression you pass `.JoinAlias` or `.JoinQueryOver` into a `string` that's used with the QueryOver query's underlying criteria query.

This is easiest to see with an example.

Here's an example QueryOver query and its underlying Criteria query, as well as the SQL that's ultimately generated:

**QueryOver**:

```csharp
session.QueryOver<Product>()
    .JoinQueryOver(pr => pr.Reviews, () => reviewAlias)
    .SelectList(list => list
        .SelectGroup(pr => pr.Id)
        .SelectMax(() => reviewAlias.Rating))
    .List<object[]>();
```

**Criteria**:

```csharp
session.CreateCriteria(typeof(Product))
    .CreateCriteria("Reviews", "reviewAlias")
    .SetProjection(Projections.ProjectionList()
        .Add(Projections.GroupProperty("Id"))
        .Add(Projections.Max("reviewAlias.Rating")))
    .List<object[]>();
```

These generate *identical* SQL:

```sql
SELECT 
	this_.ProductID AS y0_, 
	MAX (reviewalia1_.Rating) AS y1_ 
FROM 
	Production.Product this_ 
	INNER JOIN Production.ProductReview reviewalia1_ ON 
        this_.ProductID = reviewalia1_.ProductID 
GROUP BY 
	this_.ProductID 
```

The main thing to notice here is that the *name* of the alias (`reviewAlias`) we used in the QueryOver query is turned into a `string` which is ultimately used in the SQL query (`reviewalia1_`).

What this means is that you **cannot** write code like this:

```csharp
var query = session.QueryOver<Product>()
    .JoinAlias(pr => pr.Reviews, () => reviewAlias);

FilterQueryByRating(query, reviewAlias);

query
    .SelectList(list => list
        .SelectGroup(pr => pr.Id)
        .SelectMax(() => reviewAlias.Rating))
    .List<object[]>();
    
// Include only products with a rating > 2   
public static void FilterQueryByRating(
    IQueryOver<Product, Product> query, 
    ProductReview reviewAlias)
{
    query.Where(Restrictions.Gt(Projections.Property(() => reviewAlias.Rating), 2));
}
```

Do you see the problem? If we were to rename either `FilterQueryByRating`'s `reviewAlias` parameter *or* the `reviewAlias` that the query is using, our query would not work. In other words, this will not work:

```csharp
// Will explode:
public static void FilterQueryByRating(
    IQueryOver<Product, Product> query, 
    ProductReview productReviewAlias)
{
    query.Where(Restrictions.Gt(
        Projections.Property(() => productReviewAlias.Rating), 2));
}
```

You'll get an error stating "could not resolve property: productReviewAlias...". This is because the alias we're using in the `Where` clause does not match the one we created when we joined to `ProductReview`.

This may not seem like a big deal, but you really don't want simply renaming an alias to cause queries to blow up. This is especially true if you're trying to reuse query building methods and you can't guarantee what variable name the user of your method will choose.

To solve this problem, we can create a helper method that will do something similar to what NHibernate is doing under the hood for us--combine expressions and create a projection from the resulting `string`:

```csharp
public static PropertyProjection BuildProjection<T>(
    Expression<Func<object>> aliasExpression, 
    Expression<Func<T, object>> propertyExpression)
{
    string alias = ExpressionProcessor.FindMemberExpression(aliasExpression.Body);
    string property = ExpressionProcessor.FindMemberExpression(propertyExpression.Body);

    return Projections.Property(string.Format("{0}.{1}", alias, property));
}
```

(`ExpressionProcessor` is a class under the `NHibernate.Impl` namespace)

We'll then update our filtering function to use it:

```csharp
public static void FilterQueryByRating(
    IQueryOver<Product, Product> query, 
    Expression<Func<object>> productReviewAlias)
{
    PropertyProjection prop = BuildProjection<ProductReview>(
        productReviewAlias, pr => pr.Rating);

    query.Where(Restrictions.Gt(prop, 2));
}
```

Finally, we need to make a small tweak to our main query:

```csharp
ProductReview reviewAlias = null;

var query = session.QueryOver<Product>()
    .JoinAlias(pr => pr.Reviews, () => reviewAlias);

FilterQueryByRating(query, () => reviewAlias);

query
    .SelectList(list => list
        .SelectGroup(pr => pr.Id)
        .SelectMax(() => reviewAlias.Rating))
    .List<object[]>();
```

This is much more robust: our filtering method doesn't even know what alias the query is using, and if that alias changes or someone decides to use our method in the future, everything will work fine.

## Summary

I covered a lot in this post, but hopefully it will help you take advantage of QueryOver's most powerful features--building queries dynamically.

* Building queries with dynamic SQL can be a pain, but using QueryOver to dynamically build queries can be much easier and more maintainable.
* Refactoring conditional restrictions and joins into extension methods can keep queries readable and refactor logic into reusable pieces.
* To make those extension methods as robust as possible, we can make the methods generic and therefore more flexible.
* Passing around aliases between methods when building QueryOver queries has some pitfalls and needs some special attention.
