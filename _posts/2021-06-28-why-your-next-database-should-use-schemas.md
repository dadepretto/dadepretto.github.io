---
layout: post
title: "Why your next database should use schemas"
date: 2021-06-28 22:54 +0200
category: design
tags: SQL Database Design Schemas
---

If you're using SQL Server, the chances are that you have all your objects inside the `dbo` schema. In this post, we will discover why you should leverage the expressive power of schemas in your next database project, plus I'll show you my method to create a meaningful categorization with them.

## TL;DR

Putting all your stuff inside `dbo` is most probably not your best option. Instead, go to [https://github.com/dadepretto/demo-sample-database](https://github.com/dadepretto/demo-sample-database) and take a look at my way of structuring databases using schemas to avoid this anti-pattern in your next project.

## Is `dbo` evil?

As you already probably know, the `dbo` schema is the default one in SQL Server, and it's often the only schema used to store all user-defined objects.

As long as you're creating a sample database for tests and demos, or you're working with Entity Framework's code-first approach for small databases, that's completely fine. But as soon as you're using a database-first method, and maybe you're defining lots and lots of user-defined objects, this approach could not be the best one.

That's because, in a typical business database, you can find more than the mere data tables for your business entities. For example, there could be data-access and data-manipulation stored procedures used by one or more applications, landing tables for incoming and outgoing data, the logic for ETL processes, and finally, your maintenance/utility scripts.

By keeping all of these objects in the same schema, you usually end up with a big mess or - at best - find yourself using a funky naming convention that makes your objects' names cryptic and eventually unreadable: a perfect recipe for creating legacy code.

Moreover, keeping track of permissions on objects to technical users (e.g., your applications' one) is pretty tricky, so people usually tend to grant permissions on the entire database. Of course, unless they put all users directly inside the `db_owner` group...

## The benefits of schemas

Rational usage of schemas could solve all these problems and more just by maintaining your database structure neat and clean.

Firstly, you'll be able to quickly and easily grant the right level of permissions to groups/roles based on their function, improving security and reducing the risk of data breaches.

Also, by enforcing a clear and explicit structure, you are auto-documenting your database's data and control flows, making it more straightforward for you to reason about in the future and for your colleagues to understand and follow. All of this, without the hassle of maintaining extensive documentation.

Finally, your Azure Data Studio's Database Projects or Visual Studio's SSDT projects will look like and feel a lot more organized and much easier to navigate.

## Ok, but how?

At this point, you're maybe thinking that, after all, using schema can be a best practice, and perhaps you're wondering about how a database could end up looking like with this approach or how you can implement it inside your project. But, as always, there are lots of options and opinions out there.

Microsoft, for example, gives a great demo of this approach with their WorldWideImporters sample database, whose structure is explained thoroughly in [this article](https://docs.microsoft.com/en-us/sql/samples/wide-world-importers-oltp-database-catalog?view=sql-server-ver15).

I think it's a good starting point for sure, and it's great that Microsoft itself promotes such an organized approach to database development. Still, you may find their proposal a little bit overkill for your use case or, in general, not so simple to remember and use.

That's why I'll dedicate the following sections to describe my solution for schema organization that I developed and refined throughout several projects, which I think represents a good balance between completeness and easiness of implementation and usage.

## Introducing, *DePretto's schema-based structure* for SQL Server databases (v1.0)

This solution builds on top of three main pillars concepts that drive the entire architecture, which are:

- External applications and processes must not access core data tables directly but only through views, tabular functions, or stored procedures
- These data-access objects should have their dedicated schema(s), although they can share the same one if it's semantically feasible
- You should separate your non-business-related tools from the other components

As a results of these rules, I've identified some standard schemas you can use to organize your database:

- Core business data
  - `Data`
  - `History`/`Trash`
- External applications
  - `App`/`App*`
- ETL Processes
  - `Inbound`/`Inbound*`
  - `Outbound`/`Outbound*`
  - `Processing`/`Processing*`
- Non-business tools
  - `Utility`

Now, let's go through each one of them to deep dive into their definition, their rules, and their exceptions, alongside concrete examples of their content in a fictional ERP database.

### `Data`

The `Data` schema is where all your core business data tables live. You can also put other objects here, like views, tabular functions, and stored procedures, but their only purpose is to embed business logic. The fundamental rule is that external applications must never access any of these objects directly.

In our fictional ERP database, the content could be:

- Tables
  - `Employee`
  - `Customer`
  - `Product`
  - `Order`
  - `OrderDetail`
- Views
  - `Order_Extended`

### `History`/`Trash`

If you're using temporal tables, you should put your history tables here using the same names as the original ones. This schema must not contain any other object.

It would be best to choose the schema's name based on why you want to keep the data in the first place. For example, if you do it to keep the history of your data, then go with `History.` If it's more like a backup, go ahead and use the `Trash` name.

In our fictional ERP database, the content could be:

- Tables
  - `Employee`
  - `Customer`
  - `Product`
  - `Order`
  - `OrderDetail`

### `App`/`App*`

Here is where you have all the views, tabular functions, and stored procedures used by your application(s) as a data-access layer. It must be the only surface through which your application(s) access the database, becoming effectively your applications' database interface.

The objects inside here usually refer to those inside the `Data` or `History`/`Trash` schemas. However, you can provide stripped-down versions of them, for example, by hiding some system columns or filtering some rows based on static system rules.

An important exception to the general rule is that you can have some accessed tables directly by your application(s), but only if they contain non-business-related data like configurations or logs. Of course, I suggest putting that information in another (probably better-suited) data store (like a JSON file or a non-relational database), but you can put them here if that is not an option.

Lastly, if you have more than one application and want to provide each one with a completely different set of objects, you can create multiple schemas with the `App` prefix and then the application's name or function. (e.g., `AppAdminConsole` for your internal management application and `AppCustomerPortal` for your customer-facing portal).

For our ERP database example, let's imagine a multi-application scenario with the following structure.

- `AppAdminConsole`
  - Stored Procedures
    - `Employee_Get`
    - `Employee_List`
    - `Employee_Insert`
    - `Employee_Update`
    - `Employee_Delete`
    - `Customer_Get`
    - `Customer_List`
    - `Customer_Insert`
    - `Customer_Update`
    - `Customer_Delete`
    - `Product_Get`
    - `Product_List`
    - `Product_Insert`
    - `Product_Update`
    - `Product_Delete`
    - `Order_Get`
    - `Order_List`
    - `Order_Insert`
    - `Order_Update`
    - `Order_Delete`
    - `OrderDetail_Get`
    - `OrderDetail_List`
    - `OrderDetail_Insert`
    - `OrderDetail_Update`
    - `OrderDetail_Delete`
- `AppCustomerProtal`
  - Stored Procedures
    - `Customer_Get`
    - `Product_Get`
    - `Product_List`
    - `Order_Get`
    - `Order_List`
    - `Order_Insert`
    - `Order_Delete`
    - `OrderDetail_Get`
    - `OrderDetail_List`
    - `OrderDetail_Insert`
    - `OrderDetail_Update`
    - `OrderDetail_Delete`

Homonym stored procedures that are on different schemas can differ in functionalities and code flow. For example, each procedure inside the `AppCustomerProtal` could require a mandatory `CustomerId` parameter to ensure that every dataset is filtered by `CustomerId`. This parameter alone won't prevent data leakage, of course. Still, once your application has the right `CustomerId`, you're sure that the datasets themselves contain only pieces of information about that specific customer.

### `Inbound`/`Inbound*`

This schema(s) is where all your data landing tables are supposed to be, and typically external agents (e.g., Azure Data Factory) have access to it to perform data loading.

If you have more than one flow of incoming data and wish to divide them into separate spaces (e.g., because they import the same data), you can have two or more of these schemas using the same `Inbound` prefix then the name of the process. You can also have only one `Inbound` schema shared between two or more agents, as long as they import different business entities, and it's not a problem that agents could access tables outside their specific scope.

Although it's pretty untypical, you can also have other types of objects here, like stored procedures, table functions, or views. The only important thing is that these objects must never access data outside the `Inbound`/`Inbound*` schema(s). They should only contain data-access functionalities for the schemas' tables or, at most, some logic to transform the data from the source to the landing table (e.g., an upsert procedure for rows of a landing table). Finally, in a multi-`Inbound` scenario, objects inside one `Inbound*` schema must not refer to any objects from other `Inbound*` schemas outside itself.

Generally speaking, one set of `Inbound` tables is more than enough for most cases. Still, our fictional ERP database will have two different sources for customer definitions for completeness of examples.

- `InboundCRM`
  - Tables
    - `Customer`
- `InboundPublicWebsite`
  - Tables
    - `Customer`

### `Outbound`/`Outbound*`

This schema(s) works the same as their `Inbound`/`Inbound*` counterpart, except that their purpose is to export data rather than import data, of course. All the other considerations and rules are the same.

In our example, we only want to export order data without any other information, so we'll use only one `Outbound` schema.

- `Outbound`
  - Tables
    - `Order`
    - `OrderDetail`

### `Processing`/`Processing*`

Here is where all your ETL logic will reside, so you'll have stored procedures, table functions, and views to accomplish data transformation and data load to and from your `Data` schema.

Although having many configuration tables is a little bit of a code smell for me, you should put them inside here if necessary. As for other schema types, you can choose to create a separate schema using the standard `Processing` prefix to separate different processes in different categories if you feel the need to do so.

A critical aspect of this schema is objects' naming convention. Given the complexity of this topic, I'll leave it for another future post. For now, let's say that you may want to find a rigid convention and stick to it, and also clearly identify the main objects that the external agents should access to operate.

In our database, we choose to have one schema for each source/destination.

- `ProcessingCRM`
  - Stored Procedures
    - `Main_Import`
    - `Customer_Import`
- `ProcessingPublicWebsite`
  - Stored Procedures
    - `Main_Import`
    - `Customer_Import`
- `ProcessingOutbound`
  - Stored Procedures
    - `Main_Export`
    - `Order_Export`
    - `OrderDetail_Export`

### `Utility`

That's probably the most straightforward schema, and - as the name suggests - it should be your toolbelt of SQL stored procedures and other objects to perform your day-to-day work or any general SQL-related stuff.

I always put here Itzik Ben-Gan's [`GetNums`](https://tsql.solidq.com/SourceCodes/GetNums.txt) procedure and Adam Machanic's [`sp_WhoIsActive`](http://whoisactive.com), alongside some utilities like a procedure to swap all the objects between two schemas, a procedure to truncate all the tables inside one schema. So, as you can see, they are tools you can deploy in any database despite having different business logic.

In our ERP database, we'll put just the basics.

- `Utility`
  - Functions
    - `GetNums`
  - Stored Procedures
    - `WhoIsActive`

## Security by design

As said earlier, if you organize your code like this, you'll get more than just code organization.

Now you can create custom roles associated with your schemas. For example, you can create an `App` role that has various grants on the `App` schema itself or an `ETL` role that has CRUD grants for the tables inside the `Inbound` schema, and `execute` grants for the `Processing` stored procedure(s) that it needs to call to bootstrap the ETL process.

Of course, if you have more than one `Application*` or `Inbound*`/`Outbound*`/`Processing*` schemas, you can (and should) create one role for each one so that you are adopting the Principle of Least Privilege (PoLP) best practice.

You can define all of these roles and associated permissions at the project level inside your `.sqlproj` file, and then later, you can add users to those roles after the deployment. Please note that if you go this route, you'll need to set the right publishing profile to prevent group memberships from being deleted, but it's a matter of clicks. If you'd like a post about this last topic, please let me know in the comments.

## Final remarks

What we've seen so far is a template that served me well during recent years, went through several refinement cycles, and I don't exclude that it could have other refinements in the future. But, still, I think it's a solid baseline, and I invite you to try it if you think you could use it inside your next database project.

Also, although the database example presented here is an OLTP one, the whole concept can be easily adapted to OLAP applications, such as data warehouses.

And to answer that question that you've been thinking about since the beginning: yes. Of course, you will need to prepend all objects' names with their schema, but you should be doing it also with `dbo`, so I see no points here. I'll talk about writing T-SQL correctly in a future post, so stay tuned if you want to learn more or have a fight over it! 🙃

Finally, if you want to discuss what I've presented here, please contact me anywhere, and I'll be more than pleased to hear from you.
