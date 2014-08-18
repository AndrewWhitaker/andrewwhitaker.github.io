---
layout: post
title: "QueryOver Series - Part 7: Using SQL Functions"
date: 2014-08-15 14:46:35 -0500
comments: true
categories: nhibernate
---

In this post, I'll go over how to use functions built into the database engine. This can be useful when you want to do some work inside of your SQL query rather than do post-processing on the result set you get back.
<!-- more -->

## Dialects in NHibernate

To understand how to use and later build SQL functions, it's helpful to understand how the default SQL functions are registered with NHibernate to begin with.

NHibernate has the concept of a SQL *dialect*, a vendor-specific flavor of SQL. As you probably know, [many dialects are supported out of the box](http://www.nhforge.org/doc/nh/en/#configuration-optional-dialects). NHibernate represents dialects with a class per supported dialect. 

The `Dialect` base class registers required functions for a dialect using ANSI-92 standards. If a dialect implements a function differently, that dialect must overwrite the base class' implementation with its own.

For example, SQL Server doesn't implement the ANSI-92 `TRIM` function, so the `MsSql2000` dialect class uses a different implementation than the base class (which ultimately calls `rtrim` and `ltrim` to simulate the ANSI standard).

It's worth looking over the `Dialect` base class and possibly the dialect class for the database engine you're using to see what functions are already available to you.

## Calling functions from your queries

There are two ways to actually use SQL functions inside of your queries.

### Using `Projections.SqlFunction`

Using already registered SQL functions is fairly simple, using `Projections.SqlFunction`. For example, here's a query that gets every `Person`'s middle name, or "Not Applicable" if `MiddleName` is `null`, using the `COALESCE` function:

**QueryOver**:
```csharp
IList<string> middleNames = session.QueryOver<Person>()
    .Select(
        Projections.SqlFunction("coalesce", NHibernateUtil.String,
            Projections.Property<Person>(p => p.MiddleName),
            Projections.Constant("Not Applicable"))
    )
    .List<string>();
```

**SQL**:
```sql
SELECT 
	COALESCE (this_.MiddleName, 'Not Applicable') AS y0_ 
FROM 
	Person.Person this_ 
```

This same pattern applies to all SQL functions that you'd like to call using `Projections.SqlFunction`.

### Using `ProjectionsExtensions`

Inside of a QueryOver query, there's actually a better way to call many of the most common SQL functions. The `ProjectionsExtensions` class inside of the `NHibernate.Criterion` namespace contains extension methods that are parsed into SQL function calls.

For example, here's a query using the `.Upper` extension method. Note that these extension methods are actually on the object's properties:

**QueryOver**:
```csharp
IList<string> middleNames = session.QueryOver<Person>()
    .Select(p => p.FirstName.Upper())
    .List<string>();
```

**SQL**:
```sql
SELECT 
	UPPER (this_.FirstName) AS y0_ 
FROM 
	Person.Person this_ 
```

This is much cleaner than the alternative using `Projections.SqlFunction`:

```csharp
IList<string> names = session.QueryOver<Person>()
    .Select(
        Projections.SqlFunction(
            "upper",
            NHibernateUtil.String,
            Projections.Property<Person>(p => p.FirstName)))
    .List<string>();
```

## Using your own functions

In most cases, functions you want to use will already be registered in the dialect you're using. In some cases, however, you'll want to add a function that's not been registered. In this section of the post, I'll go over how to add the `checksum` function in SQL Server. There are a few steps involved in using your own function, I'll go over each one in detail.

There are actually two ways to invoke a custom SQL function from your queries. You can either add the function "statically" to a custom dialect, or invoke a brand new function "dynamically" at runtime that's not registered with the dialect.

### Adding your own dialect

As I discussed earlier, functions are registered in the dialect class representing the database flavor you're using. Since we can't modify those classes directly to register our function, we'll create a new dialect that's a subclass of the one we're using.

Since I'm using SQL Server in this example, I'll create a custom dialect that's a subclass of `MsSql2012Dialect`.

```csharp
using NHibernate;
using NHibernate.Dialect;
using NHibernate.Dialect.Function;

public class AdventureWorksDialect : MsSql2012Dialect
{
    public AdventureWorksDialect()
    {
        this.RegisterFunction("checksum", new StandardSQLFunction("checksum", NHibernateUtil.Int32));
    }
}
```

Then, we need to make sure our application is using the new dialect. We can do this either in our configuration code:

```csharp
var cfg = new Configuration()
    .Configure()
    .DataBaseIntegration(db =>
    {
        db.Dialect<AdventureWorksDialect>();
    });
```

Or, in our config file:

```xml
<hibernate-configuration xmlns="urn:nhibernate-configuration-2.2">
  <session-factory>
    <property name="show_sql">false</property>
    <property name="connection.driver_class">NHibernate.Driver.Sql2008ClientDriver</property>
    <property name="dialect">AdventureWorks.Database.AdventureWorksDialect</property>
    <property name="connection.connection_string_name">AdventureWorks</property>
  </session-factory>
</hibernate-configuration>
```

#### Calling the function using `Projections.SqlFunction`

If all you want to do is call a function using `Projections.SqlFunction`, you're basically done. All you need to do is call the function:

```csharp
IList<int> checksums = session.QueryOver<Product>()
    .Select(Projections.SqlFunction(
        "checksum",
        NHibernateUtil.Int32,
        Projections.Property<Product>(p => p.Id),
        Projections.Property<Product>(p => p.Name)))
    .List<int>();
```

This will yield the following SQL:

```sql
SELECT 
	CHECKSUM (this_.ProductID, this_.Name) AS y0_ 
FROM 
	Production.Product this_ 
```    

#### Creating a custom projections class

Using `Projections.SqlFunction` isn't quite satisfactory, especially after seeing the built-in `ProjectionExtensions`. We can easily create a `CustomProjections` class that provides some syntactic sugar for calling our custom function:

```csharp
public static class CustomProjections
{
    public static IProjection Checksum(params Expression<Func<object>>[] properties)
    {
        return Checksum(properties.Select(Projections.Property).ToArray());
    }

    public static IProjection Checksum(params IProjection[] projections)
    {
        return Projections.SqlFunction("checksum", NHibernateUtil.Int32, projections);
    }
}
```

Notice that we have two overloads of `Checksum`, one that takes an array of `Expression<Func<object>>`s and another that takes an array of `IProjection`s.

Using the `Expression<Func<object>>[]` overload is convenient when we don't need to combine the use of `checksum` with other functions, for example:

```csharp
IList<int> checksums = session.QueryOver<Product>(() => productAlias)
    .Select(CustomProjections.Checksum(
        () => productAlias.Name,
        () => productAlias.Id))
    .List<int>();
```

Using the `IProjection[]` overload is useful when we need to supply `checksum` with the result of calling *another* function, say `avg`:

```csharp
// Get the `checksum` of the average price for each sell start date.
IList<object[]> checksums = session.QueryOver<Product>(() => productAlias)
    .SelectList(list => list
        .SelectGroup(p => p.SellStartDate)
        .Select(
            CustomProjections.Checksum(
                Projections.Avg(
                    Projections.Property(() => productAlias.ListPrice)))
        ))
    .List<object[]>();
```

#### `StandardSQLFunction` and `SQLFunctionTemplate`

If you look through [NHibernate's implementations of various SQL functions](https://github.com/nhibernate/nhibernate-core/tree/master/src/NHibernate/Dialect/Function), you might notice that many use `StandardSQLFunction` and `SQLFunctionTemplate`. These should take care of most of your custom function needs. If not, you can always implement `ISQLFunction` and create your own implementation.

##### `StandardSQLFunction`

We used `StandardSQLFunction` to implement our `checksum` example. Basically, `StandardSQLFunction` allows you to implement a SQL function that takes an arbitrary number of arguments and returns a scalar value.

##### `SQLFunctionTemplate`

`SQLFunctionTemplate` is a bit more sophisticated, and you can use it to implement SQL functions with a *template*, like the name implies. This is typically useful when you want to require a function to have a specific number of arguments.

An example of this would be the `stuff` [function in SQL Server](http://msdn.microsoft.com/en-us/library/ms188043.aspx). This function inserts one string into another string, deleting the specified number of characters from the first string at a start index, then inserts the second string.

For example, here's how you could use `stuff` to replace "C++" with "C#":

```csharp
select stuff('C++', 2, 2, '#')
```

Since `stuff` has a fixed number of parameters, it's a good candidate for `SQLFunctionTemplate`. All we have to do to register it in our dialect is add the following line:

```csharp
this.RegisterFunction("stuff", new SQLFunctionTemplate(NHibernateUtil.Int32, "stuff(?1, ?2, ?3, ?4)"));
```

Here, we're basically just saying that `stuff` is a function whose syntax is invoking the `stuff` function with exactly four parameters.

We'll add a few more static methods to our `CustomProjections` class, since there are several ways we might want to call this function, we'll provide several overloads:

```csharp
// Usage: CustomProjections.Stuff(() => alias.Property, 1, 2, () => alias.OtherProperty)
public static IProjection Stuff(Expression<Func<object>> characterExpression, int start, int length, Expression<Func<object>> replaceWithExpression)
{
    return Stuff(Projections.Property(characterExpression), start, length, Projections.Property(replaceWithExpression));
}

// Usage: CustomProjections.Stuff(Projections.Property(..), 1, 2, Projections.Constant(...))
public static IProjection Stuff(IProjection characterExpression, int start, int length, IProjection replaceWithExpression)
{
    return Projections.SqlFunction("stuff", NHibernateUtil.String, characterExpression, Projections.Constant(start), Projections.Constant(length), replaceWithExpression);
}

// Usage: CustomProjections.Stuff(() => alias.Property, 1, 2, "Replacement")
public static IProjection Stuff(Expression<Func<object>> characterExpression, int start, int length, string replaceWithExpression)
{
    return Stuff(Projections.Property(characterExpression), start, length, Projections.Constant(replaceWithExpression));
}
```

Here's an example of how it would be used:

```csharp
IList<string> stuffResults = session.QueryOver<Product>(() => productAlias)
    .SelectList(list => list
        .Select(
            CustomProjections.Stuff(() => productAlias.Name, 0, 2, "PR")
        ))
    .List<string>();
```

### Invoking new functions at runtime

If, for some reason, you don't want to create a custom dialect and register functions there, you can still invoke an unregistered SQL function. There's an overload of `Projections.SqlFunction` that takes an `ISQLFunction` that you can define at runtime. For example, if we had not registered our `checksum` function, you could call it dynamically like this:

```csharp
Projections.SqlFunction(
    new StandardSQLFunction("checksum"),
    NHibernateUtil.Int32,
    Projections.Property(() => productAlias.Name),
    Projections.Property(() => productAlias.Id)
```

Here, we're defining and using the `checksum` function in one shot.

There *is* a disadvantage to using this method. When you register a function with the dialect instead, NHibernate adds the function to an internal cache and reuses the function definition whenever you access it by name.

Creating a new `checksum` function every time we needed to call the SQL Server `checksum` function would be wasteful--it would be better to define the function once and have NHibernate cache and reuse it.

However, we may want to leverage invoking a function dynamically to take care of special SQL functions, like SQL Server's `datediff` function.

#### Implementing SQL Server's `datediff` function.

SQL Server has a [function](http://msdn.microsoft.com/en-us/library/ms189794%28SQL.90%29.aspx) called `datediff` that returns the number of "date parts" between a given start and end date.

At first glance, it seems like we could register `datediff` using `SQLFunctionTemplate`:

```csharp
new SQLFunctionTemplate("datediff(?1, ?2, ?3)")
```

The problem here is that `datediff`'s first parameter is a SQL server keyword and *cannot* be supplied as a variable. According to MSDN:

> These dateparts and abbreviations cannot be supplied as a user-declared variable.

So that means we can't call `datediff` and supply the `datepart` dynamically. We could register a function for every possible version of `datediff` and name them all slightly differently:

```csharp
RegisterFunction("datediff-yr", new SQLFunctionTemplate("datediff(yy, ?1, ?2)"));
RegisterFunction("datediff-dd", new SQLFunctionTemplate("datediff(dd, ?1, ?2)"));
/* etc, for each valid datepart */
```

I'm not sure about you but this makes me cringe. Luckily there's a better solution. We can use NHibernate's ability to run an arbitrary, unregistered SQL function to dynamically create and execute the various versions of `datediff`. Here's the code:

```csharp
public static class DateProjections
{
    private const string DateDiffFormat = "datediff({0}, ?1, ?2)";

    public static IProjection DateDiff(
        string datepart, 
        Expression<Func<object>> startDate, 
        Expression<Func<object>> endDate)
    {
        // Build the function template based on the date part.
        string functionTemplate = string.Format(DateDiffFormat, datepart);

        return Projections.SqlFunction(
            new SQLFunctionTemplate(NHibernateUtil.Int32, functionTemplate),
            NHibernateUtil.Int32,
            Projections.Property(startDate),
            Projections.Property(endDate));
    }
}
```

Now, we're able to write queries using any date part we want without having to register a separate function for each date part. For example, here's a query that gets the `datediff` in days, quarters, and months:

```csharp
IList<object[]> checksums = session.QueryOver<Product>(() => productAlias)
    .SelectList(list => list
        .Select(DateProjections.DateDiff("dd", () => productAlias.SellStartDate, () => productAlias.SellEndDate))
        .Select(DateProjections.DateDiff("qq", () => productAlias.SellStartDate, () => productAlias.SellEndDate))
        .Select(DateProjections.DateDiff("mm", () => productAlias.SellStartDate, () => productAlias.SellEndDate)))
    .List<object[]>();
```

This still isn't perfect. You might have realized that we're still at a disadvantage since we're not using cached versions of our function definitions. One good solution to this is to use our own cache for the various `datediff` flavors. Here's what our class looks like with that modification:

```csharp
public static class DateProjections
{
    private const string DateDiffFormat = "datediff({0}, ?1, ?2)";

    // Maps datepart to an ISQLFunction
    private static Dictionary<string, ISQLFunction> DateDiffFunctionCache = 
        new Dictionary<string, ISQLFunction>();

    public static IProjection DateDiff(
        string datepart, 
        Expression<Func<object>> startDate, 
        Expression<Func<object>> endDate)
    {
        ISQLFunction sqlFunction = GetDateDiffFunction(datepart);

        return Projections.SqlFunction(
            sqlFunction,
            NHibernateUtil.Int32,
            Projections.Property(startDate),
            Projections.Property(endDate));
    }

    private static ISQLFunction GetDateDiffFunction(string datepart)
    {
        ISQLFunction sqlFunction;

        if (!DateDiffFunctionCache.TryGetValue(datepart, out sqlFunction))
        {
            string functionTemplate = string.Format(DateDiffFormat, datepart);
            sqlFunction = new SQLFunctionTemplate(NHibernateUtil.Int32, functionTemplate);

            DateDiffFunctionCache[datepart] = sqlFunction;
        }

        return sqlFunction;
    }
}
```

Now we're caching our function definitions so that we're not redefining versions of `datediff` unnecessarily.

Another enhancement that probably should be made is to make the `datepart` argument of `DateProjections.DateDiff` strongly typed. A good solution there would be to use an `enum` defining the possible `datepart` values. Then you could use a `Dictionary<DatePart, string>` to map from `enum` values to strings.

## Summary

Calling built-in SQL functions from NHibernate queries has been written about many times before, but hopefully I was able to shed some light on how those functions are registered and invoked. In summary:

* You can either register a function by using a custom dialect and invoke it by name later, or define and invoke the function in one step.
* Registering a function with a custom dialect is often the best option since the function definition is cached and reused automatically by NHibernate.
* `StandardSQLFunction` and `SQLFunctionTemplate` are implementations of `ISQLFunction` that enable easily defining SQL functions.
* Using a custom projections class is a useful abstraction to lay on top of `Projections.SqlFunction` to make code easier to read and more robust.
* You can use NHibernate's ability to call SQL functions at runtime to implement the `datediff` function in a clean way.
