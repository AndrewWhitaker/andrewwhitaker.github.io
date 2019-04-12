---
layout: post
title: "QueryOver Series - Part 9: Extending QueryOver to Use Custom Methods and Properties"
date: 2015-01-28 19:05:39 -0600
comments: true
categories: [NHibernate, QueryOver, QueryOver Series]
---

A basic tenet of QueryOver queries is that you can't query against unmapped properties. While this is generally true, in this post I'll outline some strategies you can use to register properties and functions with QueryOver so that they generate meaningful SQL.

<!-- more -->

{% include queryover-series/queryover_series_partial.markdown %}

## Basics

Just as a refresher, consider the following class:

```csharp
public class Rectangle
{
    public Rectangle(double width, double height)
    {
        this.Width = width;
        this.Height = height;
    }

    protected Rectangle()
    {
    }

    public virtual int Id { get; set; }

    public virtual double Width { get; protected set; }

    public virtual double Height { get; protected set; }

    public virtual double Area
    {
        get { return this.Width * this.Height; }
    }
}
```

Nothing too complicated right? This class maps to the following database table:

```sql
create table [Rectangle]
(
	[Id] int identity(1,1) primary key clustered,
	[Width] float not null,
	[Height] float not null
)
```

Notice that there's no `Area` column in the database. This property is computed by C# code. When you attempt to query against it using QueryOver:

```csharp
session.QueryOver<Rectangle>()
    .Where(r => r.Area > 4.0)
    .List<Rectangle>();
```

You'll get the following exception:

```
NHibernate.QueryException was unhandled
  HResult=-2146232832
  Message=could not resolve property: Area of: Rectangle.Rectangle
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
       ...
```

The gist is that since `Area` is an unmapped property, NHibernate doesn't know what to do with it. Lets see if we can change that.

## Registering an Unmapped Property with QueryOver

To get the gist of what we're going to do, it might be helpful to take a look at the [ProjectionExtensions.cs](https://github.com/nhibernate/nhibernate-core/blob/master/src/NHibernate/Criterion/ProjectionsExtensions.cs) and [ExpressionProcessor.cs](https://github.com/nhibernate/nhibernate-core/blob/master/src/NHibernate/Impl/ExpressionProcessor.cs) files in the NHibernate source code. There are only two things that we need to do:

1. Write a function that can translate an `Expression` and turns it into a `Projection` that QueryOver can use.
2. Register the function from #1 with the `ExpressionProcessor` class referenced above.

### Translating the `Area` property into a `Projection`

If you look at the `ExpressionProcessor` class, you'll see a bunch of `RegisterCustomProjection` calls. A few of them are used to let you use `DateTime` properties like `Month` or `Hour` directly in a QueryOver query. Since this is really close to what we want to do with the `Area` property, lets follow those methods as an example.

Taking a closer look at `RegisterCustomProjection`, you'll notice that the function takes two arguments:

1. The property or method we want to register, and
2. A `Func<MemberExpression, IProjection`. In other words, a lambda that takes a `MemberExpression` and returns an `IProjection`.

Let's implement #2 first.

#### Implementing the `ProcessArea` function

The hardest part of this process is going to be the `ProcessArea` function that will take an expression and somehow return a projection that computes the length. Here's what that looks like:

```csharp
public static class RectangleExtensions
{
    /// <summary>
    /// Helper function that takes an "alias" and property name and combines them into 
    /// property access that can then be used in a projection.
    /// </summary>
    /// <param name="alias">The alias.</param>
    /// <param name="property">The property.</param>
    /// <returns>A string representing full property access.</returns>
    private static string BuildPropertyName(string alias, string property)
    {
        if (!string.IsNullOrEmpty(alias))
        {
            return string.Format("{0}.{1}", alias, property);
        }

        return property;
    }

    /// <summary>
    /// Processes the "Area" property access, an unmapped property,
    /// and turns it into a computation on the SQL side.
    /// </summary>
    /// <param name="expression">The expression to process.</param>
    /// <returns>The resulting projection.</returns>
    public static IProjection ProcessArea(System.Linq.Expressions.Expression expression)
    {
        /* Expressions from which we can get "Width" and "Height" property names to 
         * build a projection */
        Expression<Func<Rectangle, double>> w = r => r.Width;
        Expression<Func<Rectangle, double>> h = r => r.Height;

        /* The name of the alias used in the query, if any */
        string aliasName = ExpressionProcessor.FindMemberExpression(expression);

        /* Retrieves the strings "Width" and "Height" from the expressions above */
        string widthName = ExpressionProcessor.FindMemberExpression(w.Body);
        string heightName = ExpressionProcessor.FindMemberExpression(h.Body);

        /* Combines the "Width" and "Height" strings with the alias name to 
         * build a projection: */
        PropertyProjection widthProjection = 
            Projections.Property(BuildPropertyName(aliasName, widthName));

        PropertyProjection hProjection =
            Projections.Property(BuildPropertyName(aliasName, heightName));

        /* Finally, return a SQL function that computes the product of */
         * Width and Height */
        ISQLFunction multiplication =
            new VarArgsSQLFunction(NHibernateUtil.Double, "(", "*", ")");

        return Projections.SqlFunction(
            multiplication, NHibernateUtil.Double, widthProjection, hProjection);
    }
}
```

There's a lot going on there. Basically we need to get the `Width` and `Height` properties from the `Rectangle` class so that we can use them in a projection. This would be simple, but since the query could be using an alias, we need to combine the property names with the alias (which can be `string.Empty`) to create the proper property access.

Grabbing the property names as strings from `Expression`s prevents magic strings, which QueryOver is designed to prevent anyway.

Next, we need to build actual `Projection`s from the strings we've made, which is easy using the overload of `Projection.Property` that takes a `string`.

Finally, we'll register our function with the `ExpressionProcessor`.

#### Registering the `Area` property and `ProcessArea` function with the `ExpressionProcessor`.

I don't think it matters exactly *when* you register your custom function, but it obviously has to be before you use the property or method in a query.

Here's what the code looks like:

```csharp
ExpressionProcessor.RegisterCustomProjection(
    () => default(Rectangle).Area,
    expr => RectangleExtensions.ProcessArea(expr.Expression));
```

...And that's basically it. We're telling NHibernate to call the `ProcessArea` function when the `Rectangle.Area` property is used.

Now our original query is valid and should generate the correct SQL:

```sql
SELECT
    this_.Id as Id0_0_,
    this_.Width as Width0_0_,
    this_.Height as Height0_0_
FROM
    Rectangle this_
WHERE
    (
        this_.Width*this_.Height
    ) > 4
```

### Using a user-defined function to compute `Area`

Now, lets take the concept a slightly different direction. Lets assume that the area function is defined in a user-defined function:

```sql
create function UFN_CalculateArea
(
	@Width float,
	@Height float
)
returns float
as
begin
	declare @Area float;

	select @Area = @Width * @Height;

	return @Area;
end
```

Now our QueryOver code becomes:

```csharp
/// <summary>
/// Processes the "Area" property access, an unmapped property,
/// and turns it into a computation on the SQL side.
/// </summary>
/// <param name="expression">The expression to process.</param>
/// <returns>The resulting projection.</returns>
public static IProjection ProcessArea(System.Linq.Expressions.Expression expression)
{
    /* Expressions from which we can get "Width" and "Height" property names to 
     * build a projection */
    Expression<Func<Rectangle, double>> w = r => r.Width;
    Expression<Func<Rectangle, double>> h = r => r.Height;

    /* The name of the alias used in the query, if any */
    string aliasName = ExpressionProcessor.FindMemberExpression(expression);

    /* Retrieves the strings "Width" and "Height" from the expressions above */
    string widthName = ExpressionProcessor.FindMemberExpression(w.Body);
    string heightName = ExpressionProcessor.FindMemberExpression(h.Body);

    /* Combines the "Width" and "Height" strings with the alias name to 
     * build a projection: */
    PropertyProjection widthProjection =
        Projections.Property(BuildPropertyName(aliasName, widthName));

    PropertyProjection hProjection =
        Projections.Property(BuildPropertyName(aliasName, heightName));

    /* Finally, return ISQLFunction that calls our user-defined function:  */
    ISQLFunction multiplication =
        new SQLFunctionTemplate(NHibernateUtil.Double, "dbo.UFN_CalculateArea(?1, ?2)");

    return Projections.SqlFunction(
        multiplication, NHibernateUtil.Double, widthProjection, hProjection);
}
```

And the generated SQL is what we expect:

```sql
SELECT
    this_.Id as Id0_0_,
    this_.Width as Width0_0_,
    this_.Height as Height0_0_
FROM
    Rectangle this_
WHERE
    dbo.UFN_CalculateArea(this_.Width, this_.Height) > 4;
```

## Other Tips

There are a few things worth noting here:

<h3>Registered methods need not be implemented in C#</h3>

We could actually decide not to provide an implementation for our `Area` property:

```csharp
public class Rectangle
{
    public Rectangle(double width, double height)
    {
        this.Width = width;
        this.Height = height;
    }

    protected Rectangle()
    {
    }

    public virtual int Id { get; set; }

    public virtual double Width { get; protected set; }

    public virtual double Height { get; protected set; }

    public virtual double Area
    {
        get
        {
            throw new NotImplementedException("Only available inside a QueryOver query"); 
        }
    }
}
```

NHibernate will happily transform the `Area` property into the correct SQL as before--after all, it isn't actually *invoking* the property, it's evaluating it inside of an expression tree.

Another example of where this might be useful is for functionality that's actually only available in SQL. Let's pick on the `checksum` function. If we wanted to use that inside of a QueryOver query, one way to do that would be to create a `CheckSum` extension method and register it (check [Using SQL Functions](../blog/2014/08/15/queryover-series-part-7-using-sql-functions) for how to add the `checksum` function with a custom dialect):

Here's our `CheckSum` method:

```csharp
public static class QueryOverExtensions
{
    public static int CheckSum(this object o)
    {
        throw new NotImplementedException("Must be used inside of a QueryOver query.");
    }
}
```

Here's the `ProcessCheckSum` method that processes the `CheckSum` method call into an `IProjection`:

```csharp
public static class Extensions
{
    public static IProjection ProcessCheckSum(MethodCallExpression methodCallExpression)
    {
        IProjection property =
            ExpressionProcessor.FindMemberProjection(
                methodCallExpression.Arguments[0]).AsProjection();

        return Projections.SqlFunction("checksum", NHibernateUtil.Int32, property);
    }
}
```

Finally, here's registering it with the `ExpressionProcessor`:

```csharp
ExpressionProcessor.RegisterCustomProjection(
    () => default(object).CheckSum(), Extensions.ProcessCheckSum);
```

Here's an example of getting a `checksum` of `Rectangle.Height`:

```csharp
int checksum = session.QueryOver<Rectangle>()
    .Select(rct => rct.Height.CheckSum())
    .Take(1)
    .SingleOrDefault<int>();
```

### Using multiple properties/columns in a function call

There isn't an example of this (that I can find) in the NHibernate code base, so I figured I'd go ahead and provide one.

Continuing with the `checksum` example above, what if we wanted to supply some additional columns to the `checksum` function? At this point it might be wiser to use the strategy outlined in Part 7, but lets expand the `CheckSum` extension method for the sake of an example.

Here's how the extension method itself needs to change:

```csharp
public static class QueryOverExtensions
{
    public static int CheckSum(this object o, params object[] additionalProperties)
    {
        throw new NotImplementedException("Must be used inside of a QueryOver query.");
    }
}
```

Note the use of `params`--we've made it possible for the user of our method to supply as many additional properties as they'd like.

Here's the new implementation of `ProcessCheckSum`:

```csharp
public static IProjection ProcessCheckSum(MethodCallExpression methodCallExpression)
{
    /* Retrieve the property the extension method was called on as a projection */
    IProjection property =
        ExpressionProcessor.FindMemberProjection(methodCallExpression.Arguments[0])
            .AsProjection();

    var projections = new List<IProjection> { property };

    /* Process the array that's supplied as the second argument in the expression. */
    var additionalProperties = (NewArrayExpression)methodCallExpression.Arguments[1];

    /* Convert each item in the array into a projection */
    IEnumerable<IProjection> additionalProjections =
        additionalProperties.Expressions
            .Select(expr => 
                ExpressionProcessor.FindMemberProjection(expr).AsProjection());

    /* Combine the first projection and the additional ones */
    projections.AddRange(additionalProjections);

    return Projections.SqlFunction(
        "checksum", NHibernateUtil.Int32, projections.ToArray());
}
```

The key here is to notice that `Arguments[1]` is of type `NewArrayExpression` (because of the second argument being and array). We need to take out the expressions *within* that array (that's what the LINQ block does) and then supply those projections to `Projections.SqlFunction`.

Now we can call our extension method with more properties:

```csharp
int checksum = session.QueryOver<Rectangle>(() => rectAlias)
    .Select(rct => rct.Height.CheckSum(rct.Width))
    .Take(1)
    .SingleOrDefault<int>();
```

Note that `rct.Width` could easily be a property referenced with an alias (e.g. `rectAlias.Width`), even one from another class.

## Summary

Hopefully this post has been helpful in providing some useful strategies on how to get NHibernate to generate meaningful SQL from a QueryOver query, even when using unmapped properties or methods. Doing this could be hugely helpful when you have existing user-defined functions you need to call, or you'd like to use a computed property inside of a query, rather than pull back every item in your table and then filter it.

To summarize:

* Normally, unmapped properties are unavailable for querying inside of a QueryOver query, since NHibernate does not know how to translate the property into the correct SQL.
* You can tell NHibernate what to do with a method or property using the static `ExpressionProcessor` class and a method that returns an `IProjection` given an expression containing the method call.
* As long as you can represent the SQL you want to generate as some kind of `IProjection`, you can register any property or method with the `ExpressionProcessor`.
* You can create extension methods or properties that are only available for use inside of a QueryOver query.
* Methods can contain a provision for processing an arbitrary number of properties for inclusion in the SQL that's ultimately generated.
