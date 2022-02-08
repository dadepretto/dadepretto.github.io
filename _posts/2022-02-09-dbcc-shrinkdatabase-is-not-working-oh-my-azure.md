---
layout: post
title: "DBCC SHRINKDATABASE is not working. Oh my Azure!"
date: 2022-02-09 00:28:00 +0200
category: troubleshooting
tags: Azure SQL Database Administration DBCC Shrink
---

Long time no see, huh? I have to admit that I've vastly underestimated the power of my perfectionism when starting this blog, which led me to this stall. Could an exciting database story and a random peak of creativity get me out of this pit?

Talking about the actual business, though, here is today's story!

## TL;DR

Try rebuilding your indexes before (and after!) your shrink, but watch out for tables that take too long to do so...

## Introduction

Nowadays, the SQL Server community almost unanimously considers [shrinking database files a bad practice](https://www.brentozar.com/archive/2017/12/whats-bad-shrinking-databases-dbcc-shrinkdatabase/). Like, [really bad](https://www.brentozar.com/archive/2009/08/stop-shrinking-your-database-files-seriously-now/). The main idea is that allocated space will expand and reduce organically during normal database usage. If you see some free space, it's just a matter of time until it will fill again: continuously shrinking and reallocating space would be just a waste.

Of course, like any other guideline, also this one comes with its exceptions. For example, it might be sensible and accepted to shrink after deleting a significant portion of your data, especially if you don't plan to reach the old max size anytime soon. And still, the general suggested pattern is to [avoid shrinking live files directly and instead move all your indexes to a new, empty data file and shrink the old file after that](https://www.brentozar.com/archive/2020/07/what-if-you-really-do-need-to-shrink-a-database/).

This last one is an effective technique, but unfortunately, as we will see, it can't be applied if you don't have access to the data files, namely, on Azure SQL Database.

Now that we both are on the same page for what concerns industry best practices about this topic, let me introduce you to a weird problem I encountered these days, including its solution that - while effective - opens more questions than it answers.

## The case

Everything started when my team moved a significant amount of data, mostly JSON files, from the relational database to a more appropriate storage solution, leaving it with more than 50% free space. In numerical terms, the database was initially more than 860GB and fell to around 390GB after the operation, leaving more than 470GB of free but allocated space on the database. My trusted database file query reported:

| Type       | FileName   | TotalSpaceInGB | UsedSpaceInGB | FreeSpaceInGB |
| ---------- | ---------- | -------------- | ------------- | ------------- |
| ROWS       | data_0     | 509.47         | 231.58        | 277.89        |
| ROWS       | dfa_data_3 | 359.17         | 159.27        | 199.91        |

If we were on-premises, we could have used the files technique or maybe just left the database as it was, especially if we had already allocated the storage budget without any obvious repurpose plan. Sadly, but predictably enough, you don't have this luxury in the cloud. In the P1 Azure SQL Database tier we were using, only 500GB are included in the base price. Every other increment in storage size is billed separately every month.

The idea of spending money without providing any real business value seemed just too wasteful, so we concluded that - despite all the guidelines - a shrink was strictly necessary in our case. And even worse, since - as mentioned before - Azure SQL Database doesn't let you manage your data files, the file trick wasn't a feasible solution either: we needed to perform the plain old shrink on live data files.

Interestingly enough, this is also the approach Microsoft itself suggests when you try to reduce the storage tier to a value above the consumed and below the allocated, yielding the following message:

> The storage size of your database cannot be smaller than the currently allocated size. To reduce the database size, the database first needs to reclaim unused space by running `DBCC SHRINKDATABASE (<db_id>)`. Note that this operation can impact performance while it is running and may take several hours to complete.

## A perfect execution

Since I was already breaking one community best practice with this, I decided not to try my luck any further and proceeded with a diligent test of the whole process in a clone environment before applying it to production.

And in hindsight, this has been one of the best choices I've made in this project because, after I launched the usual `DBCC SHRINKDATABASE (<db_id>);`, nothing changed! File usage statistics were precisely identical as before shrinking.

"Okay, maybe I did something wrong" - I thought - "let me try again". So I relaunched the procedure, and... still nothing.

## What is going on here?

At this point, I was pretty puzzled since I did never seen this behavior before, so I started frantically trying out queries and techniques to try to get something out of that free space. I questioned my basic knowledge of `DBCC SHRINKDATABASE` trying out various parameters, used `DBCC SHRINKFILE` to tackle the problem one file per time, and even tried to rebuild some of the indexes in the hope that something would happen.

After a couple of days of this desperate trial and error, I felt like I wasn't making any progress: I didn't find an answer, article, or blog post on the subject, and I was only reading "don't shrink your database" over and over, so I opened a [question on dba.stackexchange.com](https://dba.stackexchange.com/questions/306663/why-dbccs-shrinkdatabase-and-shrinkfile-arent-working). If you're not familiar with this website, managed under the [stackoverflow.com](https://stackoverflow.com) umbrella, it's an incredible community of people where I always get clever answers to my bizarre technical questions. Highly suggested!

On that thread, some people suggested some up-and-coming ideas that I promptly tried, like `DBCC UPDATEUSAGE`, which I didn't know existed, and the `WITH (LOB_COMPACTION = ON)` clause on index reorganize, which also I didn't know about. But, aside from the educational aspect, none of them worked, sadly.

At this point, it was close to the weekend, and I was frustrated, so I decided to push that giant red "NUKE" button and proceeded to a complete rebuild of all the indexes, followed by the infamous shrink. For future reference, I used this (hopefully elegant) script:

```sql
create or alter procedure [#ForEachTable](@inputStmt nvarchar(max))
as
begin
    set nocount, xact_abort on;
    drop table if exists [#Tables];

    select concat(quotename([S].[name]), N'.', quotename([T].[name])) as [table]
    into [#Tables]
    from [sys].[schemas] as [S]
        inner join [sys].[tables] as [T]
            on [S].[schema_id] = [T].[schema_id]
    where [T].[is_ms_shipped] = 0;

    declare tables cursor local fast_forward for select [table] from [#Tables];
    open tables;

    declare @table nvarchar(max);
    fetch next from tables into @table;

    declare @total integer = (select count(*) from [#Tables]);
    declare @space integer = len(cast(@total as nvarchar(max)));
    declare @current integer = 1;
    while @@fetch_status = 0
    begin
        declare @stmt nvarchar(max) = replace(@inputStmt, N'?', @table);
        
        declare @msg nvarchar(max) = concat(
            sysutcdatetime(), N' - ',
            N'[', right(concat(N'000', @current), @space), N'/', @total, N']: ',
            N'Executing command: "', @stmt, N'".'
        );
        raiserror(@msg, 10, 1) with nowait;

        execute [sys].[sp_executesql] @stmt = @stmt;
    
        fetch next from tables into @table;
        set @current += 1;
    end;

    close tables;
    deallocate tables;
end;
go

-- Step 1: Rebuild all the indexes
raiserror(N'First rebuild...', 10, 1) with nowait;
execute [#ForEachTable] N'alter index all on ? rebuild with (online = on);';
go

-- Step 2: Shrink the database
raiserror(N'Shrink...', 10, 1) with nowait;
declare @stmt nvarchar(max) = concat(N'dbcc shrinkdatabase (', db_id(), N')');
execute [sys].[sp_executesql] @stmt = @stmt;
go

-- Step 3: Rebuild all the indexes
raiserror(N'Final rebuild...', 10, 1) with nowait;
execute [#ForEachTable] N'alter index all on ? rebuild with (online = on);';
```

Of course, this operation would take quite some time, so I launched it and called it for the day.

I admit that the hope was high, at least to close this week-long project, and the following morning I even opened my laptop before my morning coffee to see if we had progressed. And indeed, the database shrunk, reaching the desirable size of 470GB of allocated space, just enough to allow me to scale down to P1 500GB plan. Hurray! 🥳

At this point, given the number of tests I performed, I wasn't sure if the success was attributed to that last rebuild or some combination of the previous steps. But a "quick" test in another fresh copy of the database dispelled all doubts, so I proceeded to run the script in production, check the space gains, and closed the ticket.

## Remarks

The story would end here if it weren't for something quite odd that I noticed during the production deployment and I thought worthy of sharing, which made me think about this post in the first place.

In the first version of the above script, the `[#ForEachTable]` procedure didn't have any timestamp log. I added it before deploying to production because I wanted to monitor the situation better, given the time-sensitive environment.

But while I was reviewing the logs, after the fact, I noticed that in the first round of index rebuilds, a particular table took 89% of the total time, even if it consumes only 15% of the whole storage space, all index included.

This table is known for having a high LOB insert/delete usage pattern, so I initially came up with the theory that, by its nature, it was going to take more time to reorganize all that mess. But this idea didn't last long because then I realized that also in the second rebuild - after the shrink - took once again more than 80% of the total time of execution.

Right now, I don't have the instruments, the infrastructure, and, most importantly, the time to dig deeper on this topic to discover more, but my gut feeling is that something inside that table could be the culprit of the problem in the first place.

If you have any idea about this subject, feel free to drop me a line or reply directly to the question on dba.stackexchange.com and I'll do my best to try it out. I will let you know in another post if I will eventually find the root cause of the problem, so consider subscribing to be notified!

Until then, bye! :)
