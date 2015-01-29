---
layout: post
title: "Named Queries and Unmapped Types"
date: 2014-10-26 16:38:35 -0500
comments: true
categories: nhibernate
---

*This post was inspired by [this StackOverflow question](http://stackoverflow.com/q/26512930/497356)*.

Named queries are extremely useful in a variety of situations when you're using NHibernate. If you want to use dialect-specific features that aren't supported in NHibernate, you're practically forced to go this route.

Another (possibly less common scenario) is this: you're working with a database and want to use a stored procedure. For some reason you cannot change the stored procedure.

<!-- more -->

On the surface, this doesn't sound problematic: Just map the stored procedure to a named query and execute it. What if the stored procedure returns a resultset whose column names are less than ideal?

To make this problem a bit more concrete, here's a simple stored procedure written against the AdventureWorks database that I'll work with for the rest of the post:

```sql
create procedure [Products_GetAll]
as begin
	select
		[Production].[Product].[ProductID] as [prod_id],
		[Production].[Product].[Name] as [prod_name],
		[Production].[Product].[Color] as [prod_col]
	from
		[Production].[Product];
end
```

Our result type, `ProductDTO` has property names we'd expect:

```csharp
public class ProductDTO
{
    public int Id { get; set; }

    public string Name { get; set; }

    public string Color { get; set; }
}
```

With these pieces in mind, lets go over some possible solutions to our problem. I'm assuming here that you don't want to just rename the properties on `ProductDTO`. You could certainly do that--however you'd end up with some ugly property names, which is what we're trying to avoid.

So basically the rules are:

* We can't change the stored procedure
* We want sane names for the properties in `ProductDTO`

## Map `ProductDTO`

Named queries can have a `<return>` element with a `class` attribute. If we went this route, our `.hbm.xml` file containing the named query might look like this:

```xml
<hibernate-mapping xmlns="urn:nhibernate-mapping-2.2">
  <sql-query name="Products_GetAll">
    <return class="AdventureWorks.ProductDTO">
      <return-property name="ProductId" column="prod_id" />
      <return-property name="ProductName" column="prod_name" />
      <return-property name="ProductColor" column="prod_color" />
    </return>
    exec [Products_GetAll];
  </sql-query>
</hibernate-mapping>
```

This looks like it would work right? The problem is that if you go this route, `ProductDTO` must be mapped. If you try using this mapping with an unmapped class, NHibernate will throw an exception stating that it doesn't know what `ProductDTO` is.

This might actually make sense depending on your architecture, but if the stored procedure we're executing is purpose-built for a particular area of our application, it probably doesn't make sense to do that. I'm thinking specifically of a domain-driven architecture where domain entities are mapped to tables in a database. If that's the architecture we're using then adding a `GetAllProducts` entity doesn't really make sense.

Furthermore, this stored procedure is simply a *query*. It doesn't really make sense to map something that's just used to query the same way we'd map a table to a class that can be inserted/updated.

There's also the limitation that the poster of the StackOverflow question above had--his or her query returned a resultset that did not have a column/columns that could be used as an identifier. NHibernate requires that mapped classes have a column or combination of columns that can uniquely identify the row.

## Use LINQ to transform the result of executing the named query

This solution involves using `<return-scalar>` in our named query. In other words, an `.hbm.xml` file that looks like this:

```xml
<hibernate-mapping xmlns="urn:nhibernate-mapping-2.2">
  <sql-query name="Products_GetAll">
    <return-scalar column="prod_id" type="integer"/>
    <return-scalar column="prod_name" type="string"/>
    <return-scalar column="prod_col" type="string"/>

    exec [Products_GetAll];
  </sql-query>
</hibernate-mapping>
```

And our C# to get the `Product`s list would look like this:

```csharp
var products = session.GetNamedQuery("Products_GetAll")
    .List<object[]>()
    .Select(obj => new ProductDTO
    {
        Id = (int)obj[0],
        Name = (string)obj[1],
        Color = (string)obj[2]
    })
    .ToList();
```

This will work fine. The problem is that it can become very unmanageable with more than 10 properties. NHibernate is supposed to get rid of code like this anyway. It feels like there should be a better solution.

## Use a Custom ResultTransformer

The reason we can't use a built-in result transformer (i.e. one in the `NHibernate.Transform` namespace) is because the one that would be most useful, `AliasToBeanTransformer`, requires that we map aliases from the query to properties on the class we want to project into.

Unfortunately NHibernate doesn't allow us to do this mapping with a named query. We can, however, create a new result transformer that allows us to do that. We're using the same XML mapping for our named query as before:

```xml
<hibernate-mapping xmlns="urn:nhibernate-mapping-2.2">
  <sql-query name="Products_GetAll">
    <return-scalar column="prod_id" type="integer"/>
    <return-scalar column="prod_name" type="string"/>
    <return-scalar column="prod_col" type="string"/>

    exec [Products_GetAll];
  </sql-query>
</hibernate-mapping>
```

Here are the steps you would take to map `ProductDTO` to the results of the query.

**1. Create an attribute that allows us to map column names to properties:**
```csharp
using System;

[AttributeUsage(AttributeTargets.Property | AttributeTargets.Field, AllowMultiple = false)]
public class NHibernateQueryColumnAttribute : Attribute
{
    public NHibernateQueryColumnAttribute(string columnName)
    {
        this.ColumnName = columnName;
    }

    public string ColumnName { get; private set; }
}
```

**2. Apply the attribute to our result class:**
```csharp
public class ProductDTO
{
    [NHibernateQueryColumn("prod_id")]
    public int Id { get; set; }

    [NHibernateQueryColumn("prod_name")]
    public string Name { get; set; }

    [NHibernateQueryColumn("prod_col")]
    public string Color { get; set; }
}
```

**3. Create a result transformer that can leverage that attribute (based on [`AliasToBeanResultTransformer`](https://github.com/nhibernate/nhibernate-core/blob/master/src/NHibernate/Transform/AliasToBeanResultTransformer.cs):**
```csharp
using System;
using System.Collections;
using System.Collections.Generic;
using System.Linq;
using System.Reflection;
using NHibernate;
using NHibernate.Transform;

public class QueryColumnAttributeTransformer<TResultType> : AliasedTupleSubsetResultTransformer
{
    private readonly ConstructorInfo constructor;
    private readonly Dictionary<string, MemberInfo> memberColumnMap;
    private readonly Type resultType = typeof(TResultType);

    public QueryColumnAttributeTransformer()
    {
        this.constructor = this.resultType.GetConstructor(
            BindingFlags.Public | BindingFlags.NonPublic | BindingFlags.Instance,
            null,
            Type.EmptyTypes,
            null);

        if (this.constructor == null && this.resultType.IsClass)
        {
            throw new ArgumentException(
                "The target class of a QueryColumnAttributeTransformer needs a parameterless constructor");
        }

        const BindingFlags flags = BindingFlags.Public | BindingFlags.Instance;

        this.memberColumnMap = new Dictionary<string, MemberInfo>();

        MemberInfo[] members =
            this.resultType.GetProperties(flags)
                .Cast<MemberInfo>()
                .Concat(this.resultType.GetFields(flags))
                .ToArray();

        foreach (MemberInfo member in members)
        {
            var attr = member.GetCustomAttribute<NHibernateQueryColumnAttribute>();

            this.memberColumnMap.Add(attr == null ? member.Name : attr.ColumnName, member);
        }
    }

    public override bool IsTransformedValueATupleElement(string[] aliases, int tupleLength)
    {
        return false;
    }

    public override object TransformTuple(object[] tuple, string[] aliases)
    {
        if (aliases == null)
        {
            throw new ArgumentNullException("aliases");
        }

        object result;

        try
        {
            result = this.resultType.IsClass
                ? this.constructor.Invoke(null)
                : NHibernate.Cfg.Environment.BytecodeProvider.ObjectsFactory.CreateInstance(this.resultType, true);

            for (int i = 0; i < aliases.Length; i++)
            {
                string alias = aliases[i];

                MemberInfo member;

                if (this.memberColumnMap.TryGetValue(alias, out member))
                {
                    if (member.MemberType == MemberTypes.Property)
                    {
                        ((PropertyInfo)member).SetValue(result, tuple[i]);
                    }
                    else if (member.MemberType == MemberTypes.Field)
                    {
                        ((FieldInfo)member).SetValue(result, tuple[i]);
                    }
                }
                else
                {
                    throw new ArgumentException(
                        string.Format(
                            "{0} has no field or property mapped to column '{1}'", this.resultType, alias));
                }
            }
        }
        catch (MemberAccessException e)
        {
            throw new HibernateException("Could not instantiate result class: " + this.resultType, e);
        }

        return result;
    }

    public override IList TransformList(IList collection)
    {
        return collection;
    }
}
```

**4. Use the result transformer in the query:**
```csharp
var products = session.GetNamedQuery("Products_GetAll")
    .SetResultTransformer(new QueryColumnAttributeTransformer<ProductDTO>())
    .List<ProductDTO>();
```

If you don't like the idea of modifying your result class to accommodate the query, you could make a few changes to the transformer that would allow it to use a `Dictionary` mapping that you pass in instead.

Hope that helps someone out there. And if there's an easier way to accomplish this, please let me know.