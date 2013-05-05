--- 
layout: post
title: "Entity Framework - Dealing with large 'WHERE IN' statements"
author: "dan"
comments: true
tags:
- entityframework
- c#
- sql
- batches
---

I just wanted to quickly blog about a technique that I found really useful recently. I was in a situation where I was passing a large (30,000+) number of ids from a web app (let's call it System A) to a console app (System B) using Message Queuing.

When System B received the message, it needed to run a query against the database and load information related to each matching record. An easy way to do this in Entity Framework is something like:

<pre class="brush: csharp;">
public void DoStuffWithIds(List&lt;int> ids)
{
    var context = new MyDatabaseContext();
    // Get the matching records, pull them into a list
    var records = new List&lt;Record>();
    records = context.Records.Where(x => ids.Contains(x.Id)).ToList();
    // ...do other stuff
}
</pre>

Simple, right? OK, but as you may imagine, this translates into a query like the following:

<pre class="brush: sql;">
SELECT *
FROM Records r
WHERE r.Id IN (123, 124, 125, ... 1999, 2000, 2001, ... )
</pre>

So what? Well, apparently there's a limit on how long those WHERE ... IN statements can be. Eventually you'll hit a wall and receive an error message like:

> (put error message here)

When you get to this point in your code, you have to get slightly creative. Marc Gravell has [an excellent post on Stack Overflow](http://stackoverflow.com/a/5052187/2273) where he outlines a basic solution:

> For data volumes like 300k rows, I would forget EF. I would do this by having a table such as:
> BatchId  RowId
> Where RowId is the PK of the row we want to update, and BatchId just refers to this "run" of 300k rows (to allow multiple at once etc).
> I would generate a new BatchId (this could be anything unique -Guid leaps to mind), and use SqlBulkCopy to insert te records onto this table.

In my case, my range of ids wasn't going to approach anywhere near 300k, so I didn't have to use SqlBulkCopy, but I did like the idea of a temporary batch table. After all, in my experience the performance of a JOIN is loads better than a WHERE ... IN clause.

A sample solution using this method:

<pre class="brush: csharp;">
public IEnumerable&lt;Record> PerformBatchJoinWithIds(IEnumerable&lt;int> ids)
{
    var context = GetContext&lt;MyDatabaseContext>();
    // Disable auto detection of changes; much faster for batch edits/inserts
    context.Configuration.AutoDetectChangesEnabled = false;
    // A GUID will keep track of this batch operation
    var uniqueId = Guid.NewGuid();
    // Insert the batchquery objects for each id
    foreach (var id in ids)
    {
        context.BatchQueries.Add(new BatchQuery { Id = uniqueId, IdToQuery = id });
    }
    // Detect all changes in one shot and then save them
    context.ChangeTracker.DetectChanges();
    context.SaveChanges();
    // Now we can re-enable auto detection of changes (in case we use this context elsewhere)
    context.Configuration.AutoDetectChangesEnabled = true;
    // Join the batch queries table with the records we're trying to get
    var entities = context.Records.Join(context.BatchQueries, x => x.Id, y => y.IdToQuery, (x, y) => x)
        .ToList();
    // Finally, we can delete all of the BatchQuery records matching the GUID
    context.Database.ExecuteSqlCommand("DELETE FROM BatchQueries WHERE ID = {0}", uniqueId);
    return entities;
}
</pre>