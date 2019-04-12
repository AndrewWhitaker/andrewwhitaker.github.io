---
layout: post
title: "Getting NHibernate Up and Running Quickly"
date: 2014-08-28 10:44:06 -0500
comments: true
categories: nhibernate
---

NHibernate can seem like a daunting library to set up. The configuration can get quite complicated--XML mappings, code mappings, mapping conventions, dialects, logging, etc. Sometimes you just want to get something up and running to test out a query or play around with a database other than your primary one. In this post, I'll show you how to get up and running with NHibernate in about 5 minutes and in around 50 lines of code in [LINQPad](http://www.linqpad.net).

This assumes you already have a database configured and ready to create new tables and connect to with NHibernate.
<!-- more -->

## 1. Create your database table

I'll use MySQL as an example, but first you'll need to create a database table (or a few tables) that you want to play around with. I'll create a `person` table:

```sql
CREATE TABLE `person` (
  `Id` int(11) NOT NULL AUTO_INCREMENT,
  `FirstName` varchar(200) NOT NULL,
  `BirthDate` date NOT NULL,
  `LastName` varchar(200) NOT NULL,
  PRIMARY KEY (`Id`)
)
```

## 2. Prepare your LINQPad Query

After creating a new query in LINQPad, set the **Language** to **C# Program**, then press F4 to open the **Query Properties** window. From here you can browse to the "NHibernate.dll" file that you'd like to use. I'll also add MySql.Data.dll since I need to interact with a MySql database:

![linqpad](/assets/images/posts/2014-08-28-getting-nhibernate-up-and-running-quickly-linqpad-query-properties/linqpad-query-properties.png)

Next, go over to the **Additional Namespace Imports** tab and add the following namespaces:

```
NHibernate
NHibernate.Cfg
NHibernate.Cfg.MappingSchema
NHibernate.Dialect
NHibernate.Mapping.ByCode
NHibernate.Mapping.ByCode.Conformist
```

Okay, that's all we should need to get started with actually writing the query.

## 3. Add some configuration code

Next we'll actually configure NHibernate to connect to the database correctly. Normally you would configure NHibernate via your project's *.config file, but since we're using LINQPad, a configuration file isn't really feasible. Luckily, we can configure NHibernate in C# using the `.DatabaseIntegration` extension method:

```csharp
void Main()
{
    Configuration cfg = new Configuration()
        .DataBaseIntegration(db =>
        {
            db.ConnectionString = "Server=127.0.0.1;Database=test;Uid=nhibernate;Pwd=nhibernate;";
            db.Dialect<MySQLDialect>();
        });
}
```

Here I'm just specifying the dialect I'd like to use (`MySQLDialect`) and setting my connection string.

## 4. Add an entity and mapping

Since I created a `person` table earlier, I'm going to go ahead and create a `Person` class and `PersonMap` to map that class using NHibernate's mapping-by-code. This code goes just under the `Main` method we filled in earlier. Here's the entity:

```csharp
public class Person
{
    public virtual int Id { get; set; }
    
    public virtual string FirstName { get; set; }
    
    public virtual string LastName { get; set; }
    
    public virtual DateTime BirthDate { get; set; }
}
```

Here's the mapping:

```csharp
public class PersonMap : ClassMapping<Person>
{
    public PersonMap()
    {
        this.Table("person");
        this.Id(p => p.Id);
        this.Property(p => p.FirstName);
        this.Property(p => p.LastName);        
        this.Property(p => p.BirthDate);
    }
}
```

## 5. Add code to process the mapping we added

The last configuration step is to modify our `Main` method to incorporate our mappings into the configuration. We'll modify our `Main` method as follows:

```csharp
void Main()
{
    Configuration cfg = new Configuration()
        .DataBaseIntegration(db =>
        {
            db.ConnectionString = "Server=127.0.0.1;Database=test;Uid=nhibernate;Pwd=nhibernate;";
            db.Dialect<MySQLDialect>();
        });
 
    /* Add the mapping we defined: */
    var mapper = new ModelMapper();
    mapper.AddMappings(Assembly.GetExecutingAssembly().GetExportedTypes());
  
    HbmMapping mapping = mapper.CompileMappingForAllExplicitlyAddedEntities();
    cfg.AddMapping(mapping);
}
```

## 6. Create an `ISessionFactory`, `ISession` and `ITransaction` and write a query

Ok the last step is to build an `ISessionFactory` from our configuration. From there we can get an `ISession` and an `ITransaction` to actually work with the entities we've created and mapped. We'll modify the `Main` method again as follows:

```csharp
void Main()
{
    Configuration cfg = new Configuration()
        .DataBaseIntegration(db =>
        {
            db.ConnectionString = "Server=127.0.0.1;Database=test;Uid=nhibernate;Pwd=nhibernate;";
            db.Dialect<MySQLDialect>();
        });
 
    /* Add the mapping we defined: */
    var mapper = new ModelMapper();
    mapper.AddMappings(Assembly.GetExecutingAssembly().GetExportedTypes());
    
    HbmMapping mapping = mapper.CompileMappingForAllExplicitlyAddedEntities();
        
    cfg.AddMapping(mapping);   
    
    /* Create a session and execute a query: */
    using (ISessionFactory factory = cfg.BuildSessionFactory())
    using (ISession session = factory.OpenSession())
    using (ITransaction tx = session.BeginTransaction())
    {
        session.Get<Person>(1).Dump();
                
        tx.Commit();
    }
}
```

...And that's really it. If you hit **F5** or the green play button in LINQPad, your query should run, assuming you have everything configured correctly. Here's the whole listing below, just in case:

```csharp
void Main()
{
    Configuration cfg = new Configuration()
        .DataBaseIntegration(db =>
        {
            db.ConnectionString = "Server=127.0.0.1;Database=test;Uid=nhibernate;Pwd=nhibernate;";
            db.Dialect<MySQLDialect>();
        });
 
    /* Add the mapping we defined: */
    var mapper = new ModelMapper();
    mapper.AddMappings(Assembly.GetExecutingAssembly().GetExportedTypes());
    
    HbmMapping mapping = mapper.CompileMappingForAllExplicitlyAddedEntities();
        
    cfg.AddMapping(mapping);
       
    /* Create a session and execute a query: */
    using (ISessionFactory factory = cfg.BuildSessionFactory())
    using (ISession session = factory.OpenSession())
    using (ITransaction tx = session.BeginTransaction())
    {
        session.Get<Person>(1).Dump();
                
        tx.Commit();
    }
}

public class PersonMap : ClassMapping<Person>
{
    public PersonMap()
    {
        this.Table("person");
        this.Id(p => p.Id);
        this.Property(p => p.FirstName);
        this.Property(p => p.LastName);        
        this.Property(p => p.BirthDate);
    }
}

public class Person
{
    public virtual int Id { get; set; }
    
    public virtual string FirstName { get; set; }
    
    public virtual string LastName { get; set; }
    
    public virtual DateTime BirthDate { get; set; }
}
```

## Summary

Hopefully this post will come in handy if you're looking to quickly get NHibernate up and running for some throwaway or experimental code. When you boil the setup down to the very basics it's actually not that bad at all. From here, you can make some configuration changes or enhancements. Two useful configuration options to turn on inside of the `.DatabaseIntegration` method are the `LogFormattedSql` property and the `LogSqlInConsole` property. These will allow you to actually see the SQL that NHibernate is generating.
