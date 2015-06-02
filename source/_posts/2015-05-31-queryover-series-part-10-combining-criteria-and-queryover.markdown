---
layout: post
title: "QueryOver Series - Part 10: Combining Criteria and QueryOver"
date: 2015-05-31 19:43:08 -0500
comments: true
categories: [NHibernate, QueryOver, QueryOver Series]
---

As I mentioned in the first post in this series, QueryOver is built on top of NHibernate's Criteria API. In this post, I'll explore how to mix QueryOver and Criteria functionality to enable powerful query building techniques.

<!-- more -->

{% render_partial queryover-series/queryover_series_partial.markdown %}

## Building truly dynamic queries

If you've read the article in this series about dynamically building queries, then you already know that with QueryOver you can conditionally add `WHERE` clauses and even `JOIN`s to a query without messy string manipulation. This functionality is very powerful and might be all you ever need. However, you can write even *more* flexible queries by combining QueryOver with Criteria.

For example, lets say you're building a product searching application and you want to let users search on any attribute of the product they'd like. You could start with a QueryOver query that looks like this:

```csharp
public static IList<Product> QueryProducts(ISession session, string column, string value)
{
    Expression<Func<Product, object>> propertyExpression = null;

    if (column == "Name")
    {
        propertyExpression = pr => pr.Name;
    }
    else if (column == "Color")
    {
        propertyExpression = pr => pr.Color;
    }

    return session.QueryOver<Product>()
        .Where(Restrictions.Eq(Projections.Property<Product>(propertyExpression), value))
        .List<Product>();
}
```

You've probably noticed the problem. Since QueryOver works with `Expression`s, we have to figure out what the user typed, then build the appropriate expression. This is boring and ugly. Lets see if we can use Criteria's functionality to simplify things.

### Simplifying things using Criteria

First it might be helpful to step back and remember what a Criteria query looks like. If we were writing the `QueryProducts` method with a Criteria query, we could write it like this:

```csharp
public static IList<Product> QueryProducts(ISession session, string column, string value)
{
    return session.CreateCriteria<Product>()
        .Add(Restrictions.Eq(column, value))
        .List<Product>();
}
```

Here, we're passing the user's input directly to `Restrictions.Eq`.

Before we go any further, I should mention that this does *not* open you to SQL injection. If you attempt to supply a value to `column` that is not a mapped column for the `Product` class, NHibernate throws an exception.

That said, throwing an exception isn't the most user-friendly thing in the world. We'll look at some ways to help with that later on.

### Combining QueryOver and Criteria

A combination of the two versions of `QueryProducts` gives us this:

```csharp
public static IList<Product> QueryProducts(ISession session, string column, string value)
{
    return session.QueryOver<Product>()
        .Where(Restrictions.Eq(column, value))
        .List<Product>();
}
```

That was pretty easy! But before you go committing that and introducing bugs you'll discover later on, there are two major pitfalls I need to address. Both problems are solved the same way, so I'll introduce the problems first.

#### Joining

So imagine `QueryProducts` has been running smoothly for a few months. Now, however, you'd like to add to the query. You need to `JOIN` to the `TransactionHistory` table and only retrieve products that have been modified in the last 7 days (you don't want customers seeing old stock). No problem-- you make the following modification:

```csharp
public static IList<Product> QueryProducts(ISession session, string column, string value)
{
    return session.QueryOver<Product>()
        .JoinQueryOver(pr => pr.TransactionHistory)
            .Where(tx => tx.ModifiedDate > DateTime.Now.AddDays(-7))
        .Where(Restrictions.Eq(column, value))
        .List<Product>();
}
```

That was easy right? Well, if you were to run this, NHibernate would throw an exception:

```
  Message=could not resolve property: Color of: AdventureWorks.Entities.Production.TransactionHistory
  Source=NHibernate
  StackTrace:
       at NHibernate.Persister.Entity.AbstractPropertyMapping.ToType(String propertyName)
       at NHibernate.Persister.Entity.AbstractEntityPersister.GetSubclassPropertyTableNumber(String propertyPath)
       at NHibernate.Persister.Entity.BasicEntityPropertyMapping.ToColumns(String alias, String propertyName)
       at NHibernate.Persister.Entity.AbstractEntityPersister.ToColumns(String alias, String propertyName)
       at NHibernate.Loader.Criteria.CriteriaQueryTranslator.GetColumns(ICriteria subcriteria, String propertyName)
       at NHibernate.Loader.Criteria.CriteriaQueryTranslator.GetColumnsUsingProjection(ICriteria subcriteria, String propertyName)
       at NHibernate.Criterion.CriterionUtil.GetColumnNamesUsingPropertyName(ICriteriaQuery criteriaQuery, ICriteria criteria, String propertyName, Object value, ICriterion critertion)
       at NHibernate.Criterion.CriterionUtil.GetColumnNamesForSimpleExpression(String propertyName, IProjection projection, ICriteriaQuery criteriaQuery, ICriteria criteria, IDictionary`2 enabledFilters, ICriterion criterion, Object value)
       /* etc */
```

What happened here? To understand we need to go back to the [Basics and Joining](../blog/2014/03/16/queryover-series-part-2-basics/) article. When you use `JoinQueryOver`, you're changing the context of any `.Where` calls that follow to the table you've joined on. After calling `JoinQueryOver`, NHibernate thinks that we're trying to access the `Color` property of `TransactionHistory`.

You could just move the `.Where` call *before* the `JoinQueryOver` call, but this can be hard to keep track of with a big query. There's a more robust solution.

#### Aliases

Our simple `Product` query doesn't have this problem yet, but consider another query, this one starting at `TransactionHistory` and joining to `Product`:

```csharp
Product productAlias = null;

session.QueryOver<TransactionHistory>()
    .JoinAlias(tx => tx.Product, () => productAlias)
    .Where(Restrictions.Eq(column, value))
    .List();
```

This query throws the same exception that the above query throws. You need some way to combine the supplied column name with the alias that you're supplying NHibernate. How can we solve these problems?

### Incorporating aliases

The key to solving the problems is using aliases along with our dynamic column name. Again, this is easier using Criteria directly:

```csharp
session.CreateCriteria<TransactionHistory>()
    .CreateAlias("Product", "productAlias")
    .Add(Restrictions.Eq("productAlias." + column, value))
    .List<TransactionHistory>();
```

Here, when we join to the `Product` table, we're assigning an alias (`"productAlias"`), that we're using later to create a property access expression (in the form of a `string`) that NHibernate understands.

We can use the same strategy with QueryOver, but it takes a few steps.

#### 1. Determine the name of the alias we're using

Under the hood, NHibernate is going to turn the `Expression` we supply to `JoinAlias` into a `string` that gets used the same way that the alias name supplied to `CreateAlias` is used. If we can determine the name of the alias, we can prepend that to the property we're trying to access.

To do this, we can use the `NHibernate.Impl.ExpressionProcessor` class:

```csharp
Product productAlias = null;

Expression<Func<Product>> productAliasExpression = () => productAlias;

string productAliasName =
    ExpressionProcessor.FindMemberExpression(productAliasExpression.Body);
```

Now, we'll have the name of the alias in the form of a `string`. Note that we could just use the string `"productAlias"` here, but that doesn't stand up well to refactoring.

#### 2. Combine the alias with the supplied property

The next step is easy: just combine the alias name with the property we're trying to query on:

```csharp
Product productAlias = null;

Expression<Func<Product>> productAliasExpr = () => productAlias;

string productAliasName = ExpressionProcessor.FindMemberExpression(productAliasExpr.Body);

session.QueryOver<TransactionHistory>()
    .JoinAlias(tx => tx.Product, productAliasExpr)
    .Where(Restrictions.Eq(string.Format("{0}.{1}", productAliasName, column), value))
    .List();
```

This is a little messy, so lets refactor the property access building code into a helper function:

```csharp
public static string BuildPropertyAccess<T>(Expression<Func<T>> alias, string propertyName)
{
    string aliasName = ExpressionProcessor.FindMemberExpression(alias.Body);

    return string.Format("{0}.{1}", aliasName, propertyName);
}
```

Now our query looks like this:

```csharp
Product productAlias = null;

Expression<Func<Product>> productAliasExpr = () => productAlias;

session.QueryOver<TransactionHistory>()
    .JoinAlias(tx => tx.Product, productAliasExpr)
    .Where(Restrictions.Eq(BuildPropertyAccess(productAliasExpr, column), value))
    .List();
```

This also fixes the problem with the query performing the `JOIN` from earlier:

```csharp
public static IList<Product> QueryProducts(ISession session, string column, string value)
{
    Product productAlias = null;

    Expression<Func<Product>> productAliasExpr = () => productAlias;

    return session.QueryOver<Product>(productAliasExpr)
        .JoinQueryOver(pr => pr.TransactionHistory)
            .Where(tx => tx.ModifiedDate > DateTime.Now.AddDays(-7))
        .Where(Restrictions.Eq(BuildPropertyAccess(productAliasExpr, column), value))
        .List<Product>();
}
```

The call to `session.QueryOver` also assigns an alias that we can use later to build the `Restriction` correctly.

### Validating supplied property names

The last piece of this is to nicely be able to let the user know that they've supplied an invalid property name.

Using [this answer](http://stackoverflow.com/a/856791/497356) as a guide, we can use the following method to determine if a given property name is mapped and queryable:

```csharp
public static bool IsMappedProperty<T>(string property, ISessionFactory sessionFactory)
{
    IClassMetadata metadata = sessionFactory.GetClassMetadata(typeof(T));

    return metadata.PropertyNames.Contains(property);
}
```

Now we have a nice way to validate the user's input before attempting to query using it.

With all that in mind, here's a complete(ish) demonstration:

```csharp
public static void Query(ISession session)
{
    Console.Write("Please enter a column to query on: ");
    string col = Console.ReadLine();

    Console.Write("Please enter a value to query for: ");
    string value = Console.ReadLine();

    if (IsMappedProperty<Product>(col, session.SessionFactory))
    {
        QueryProducts(session, col, value);
    }
    else
    {
        Console.WriteLine("Invalid property name.");
    }
}

public static bool IsMappedProperty<T>(string property, ISessionFactory sessionFactory)
{
    IClassMetadata metadata = sessionFactory.GetClassMetadata(typeof(T));

    return metadata.PropertyNames.Contains(property);
}

public static IList<Product> QueryProducts(ISession session, string column, string value)
{
    Product productAlias = null;

    Expression<Func<Product>> productAliasExpr = () => productAlias;

    return session.QueryOver<Product>(productAliasExpr)
        .JoinQueryOver(pr => pr.TransactionHistory)
            .Where(tx => tx.ModifiedDate > DateTime.Now.AddDays(-7))
        .Where(Restrictions.Eq(BuildPropertyAccess(productAliasExpr, column), value))
        .List<Product>();
}

public static string BuildPropertyAccess<T>(Expression<Func<T>> alias, string propertyName)
{
    string aliasName = ExpressionProcessor.FindMemberExpression(alias.Body);

    return string.Format("{0}.{1}", aliasName, propertyName);
}
```

And that's it! It took awhile to get here, but hopefully you see some potential in this technique. Keep in mind that you're not limited to `Restrictions.Eq`. You can use this strategy with anything that works with an `IProjection`, as long as you build the projection properly. This includes building dynamic `SELECT` clauses, which would be an interesting application.

## Summary

* With a normal QueryOver query, it's hard to dynamically choose property names because QueryOver works with strongly-typed `Expression`s.
* We can mix QueryOver with Criteria to dynamically supply column names...
* ... but we need to assign aliases to entities involved in the queries in order to generate the correct SQL.
* We can check for valid property names using the `GetClassMetadata` function on `ISessionFactory`.