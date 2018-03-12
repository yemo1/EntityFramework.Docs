---
title: What is new in EF Core 2.1 - EF Core
author: divega
ms.author: divega

ms.date: 2/20/2018

ms.assetid: 585F90A3-4D5A-4DD1-92D8-5243B14E0FEC
ms.technology: entity-framework-core

uid: core/what-is-new/ef-core-2.1
---

# New features in EF Core 2.1
> [!NOTE]  
> This release is still in preview.

Besides numerous small improvements and more than a hundred product bug fixes, EF Core 2.1 includes several new features:

## Lazy loading
EF Core now contains the necessary building blocks for anyone to author entity classes that can load their navigation properties on demand. We have also created a new package, Microsoft.EntityFrameworkCore.Proxies, that leverages those building blocks to produce lazy loading proxy classes based on minimally modified entity classes (e.g. classes with virtual navigation properties).

Read the [section on lazy loading](xref:core/querying/related-data#lazy-loading) for more information about this topic.

## Parameters in entity constructors
As one of the required building blocks for lazy loading, we enabled the creation of entities that take parameters in their constructors. You can use parameters to inject property values, lazy loading delegates, and services.

Read the [section on entity constructor with parameters](xref:core/modeling/constructors) for more information about this topic.

## Value conversions
Until now, EF Core could only map properties of types natively supported by the underlying database provider. Values were copied back and forth between columns and properties without any transformation. Starting with EF Core 2.1, value conversions can be applied to transform the values obtained from columns before they are applied to properties, and vice versa. We have a number of conversions that can be applied by convention as necessary, as well as an explicit configuration API that allows registering custom conversions between columns and properties. Some of the application of this feature are:

- Storing enums as strings
- Mapping unsigned integers with SQL Server
- Automatic encryption and decryption of property values

Read the [section on value conversions](xref:core/modeling/value-conversions) for more information about this topic.  

## LINQ GroupBy translation
Before version 2.1, in EF Core the GroupBy LINQ operator was always be evaluated in memory. We now support translating it to the SQL GROUP BY clause in most common cases.

This example shows a query with GroupBy used to compute various aggregate functions:

``` csharp
var query = context.Orders
    .GroupBy(o => new { o.CustomerId, o.EmployeeId })
    .Select(g => new
        {
          g.Key.CustomerId,
          g.Key.EmployeeId,
          Sum = g.Sum(o => o.Amount),
          Min = g.Min(o => o.Amount),
          Max = g.Max(o => o.Amount),
          Avg = g.Average(o => Amount)
        });
```

The corresponding SQL translation looks like this:

``` SQL
SELECT [o].[CustomerId], [o].[EmployeeId],
    SUM([o].[Amount]), MIN([o].[Amount]), MAX([o].[Amount]), AVG([o].[Amount])
FROM [Orders] AS [o]
GROUP BY [o].[CustomerId], [o].[EmployeeId];
```

## Data Seeding
With the new release it will be possible to provide initial data to populate a database. Unlike in EF6, seeding data is associated to an entity type as part of the model configuration. Then EF Core migrations can automatically compute what insert, update or delete operations need to be applied when upgrading the database to a new version of the model.

As an example, you can use this to configure seed data for a Post in `OnModelCreating`:

``` csharp
modelBuilder.Entity<Post>().SeedData(new Post{ Id = 1, Text = "Hello World!" });
```

Read the [section on data seeding](xref:core/modeling/data-seeding) for more information about this topic.  

## Query types
An EF Core model can now include query types. Unlike entity types, query types do not have keys defined on them and cannot be inserted, deleted or updated (i.e. they are read-only), but they can be returned directly by queries. Some of the usage scenarios for query types are:

- Mapping to views without primary keys
- Mapping to tables without primary keys
- Mapping to queries defined in the model
- Serving as the return type for `FromSql()` queries

Read the [section on query types](xref:core/modeling/query-types) for more information about this topic.

## Include for derived types
It will be now possible to specify navigation properties only defined on derived types when writing expressions for the `Include` method. For the strongly typed version of `Include`, we support using either an explicit cast or the `as` operator. We also now support referencing the names of navigation property defined on derived types in the string version of `Include`:

``` csharp
var option1 = context.People.Include(p => ((Student)p).School);
var option2 = context.People.Include(p => (p as Student).School);
var option3 = context.People.Include("School");
```

Read the [section on Include with derived types](xref:core/querying/related-data#include-on-derived-types) for more information about this topic.

## System.Transactions support
We have added the ability to work with System.Transactions features such as TransactionScope. This will work on both .NET Framework and .NET Core when using database providers that support it.

Read the [section on System.Transactions](xref:core/saving/transactions#using-systemtransactions) for more information about this topic.

## Better column ordering in initial migration
Based on customer feedback, we have updated migrations to initially generate columns for tables in the same order as properties are declared in classes. Note that EF Core cannot change order when new members are added after the initial table creation.

## Optimization of correlated subqueries
We have improved our query translation to avoid executing "N + 1" SQL queries in many common scenarios in which the usage of a navigation property in the projection leads to joining data from the root query with data from a correlated subquery. The optimization requires buffering the results from the subquery, and we require that you modify the query to opt-in the new behavior.

As an example, the following query normally gets translated into one query for Customers, plus N (where "N" is the number of customers returned) separate queries for Orders:

``` csharp
var query = context.Customers.Select(
    c => c.Orders.Where(o => o.Amount  > 100).Select(o => o.Amount));
```

By including `ToList()` in the right place, you indicate that buffering is appropriate for the Orders, which enable the optimization:

``` csharp
var query = context.Customers.Select(
    c => c.Orders.Where(o => o.Amount  > 100).Select(o => o.Amount).ToList());
```

Note that this query will be translated to only two SQL queries: One for Customers and the next one for Orders.

### OwnedAttribute

It is now possible to configure [owned entity types](xref:core/modeling/owned-entities) by simply annotating the type with `[Owned]` and then making sure the owner entity is added to the model:

``` csharp
[Owned]
public class StreetAddress
{
    public string Street { get; set; }
    public string City { get; set; }
}

public class Order
{
    public int Id { get; set; }
    public StreetAddress ShippingAddress { get; set; }
}
```

## Database provider compatibility

EF Core 2.1 was designed to be compatible with database providers created for EF Core 2.0. While some of the features described above (e.g. value conversions) require an updated provider others (e.g. lazy loading) will light up with existing providers.

> [!TIP]
> If you find any unexpected incompatibility or any issue in the new features, or if you have feedback on them, please report it using [our issue tracker](https://github.com/aspnet/EntityFrameworkCore/issues/new).
