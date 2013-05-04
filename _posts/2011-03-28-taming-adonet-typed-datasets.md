--- 
layout: post
title: "Taming ADO.NET Typed Datasets"
author: "dan"
comments: true
tags:
- .net
- datasets
---

At my current employer, I get thrown a lot of small “write-it-and-forget-it” apps that often involve querying a bunch of databases and producing some result. Since we’re a fairly large institution, this might involve querying a variety of database servers — not all SQL Server.

For these kinds of projects, I love ADO.NET Typed DataSets. Even though there are many newer data access technologies from Microsoft available now (Entity Framework, for example), typed datasets are still useful when you’re interacting with data from non-MS SQL databases. I can easily throw my database tables onto the designer, let Visual Studio generate my table adapters, and get to coding.

However, I think anyone who’s ever worked with ADO.NET datasets has encountered this horrifying error message:

> Failed to enable constraints. One or more rows contain values
> violating non-null, unique, or foreign-key constraints.

Great. Which constraints? Which records? If my in-memory dataset is holding 20,000+ records, am I really going to know where my problem is? Although you can sometimes figure it out if you’re in the middle of debugging, when this error occurs in production, you have no way of stepping into the problem and figuring out what went wrong.

You may not know it, but datatables actually contain a neat method called `GetErrors()` which returns all rows that encountered an error. In addition, each tablerow contains a method `GetColumnsInError()` which, well, return the columns that encountered the problem. These two methods can go a *long* way towards helping you solve your problem.

After dealing with this problem for the N^th^ time and deciding to do something about it, I wrote an extension method that can be used to fill an arbitrary Table and, if any constraints are violated, throws a *detailed* exception listing the errors:

<pre class="brush: csharp;">
internal static class DataTableExtensionMethods
{
    /// <summary>
    /// This extension method can be used in place of the standard "GetData"
    /// method that is auto-generated in a strongly-typed data dapter.
    /// It internally calls GetData, but checks for constraint errors and,
    /// if they are found, throws a detailed exception
    /// listing all of the errors instead of the default vague error message used
    /// by ConstraintException.
    /// </summary>
    /// <typeparam name="TableType">The type of the table that is expected
    /// to be returned.</typeparam>
    /// <param name="table">A reference to the table that will hold the data.</param>
    /// <param name="adapter">The adapter that will be used to fill the table.</param>
    /// <returns>The same table as was given, but filled with the appropriate data.</returns>
    public static TableType GetDataSafely<TableType>(this TableType table, Component adapter)
        where TableType : DataTable
    {
        if (table.DataSet == null)
        {
            throw new ArgumentException(@"
The table passed to the GetDataSafely method should not have a null DataSet.
When calling GetDataSafely, be sure to pass in a table from a DataSet and
don't use the table's constructor.", "table");
        }
        table.DataSet.EnforceConstraints = false;
        // Unfortunately, we have to use 'Component' as the type for adapter
        // since there is no better 'base type' that adapter inherits from
        // (if someone knows of something else, please let me know!)
        // So, we can use C# 4's dynamic keyword to help us out here
        // (Obviously, this code will crash if there is no Fill() method on adapter)
        dynamic dynAdapter = adapter;
        dynAdapter.Fill(table);
        try
        {
            table.DataSet.EnforceConstraints = true;
        }
        catch (ConstraintException ex)
        {
            var errorQuery = table.GetErrors()
                .SelectMany(x => x.GetColumnsInError().Select(
                    c => x.GetColumnError(c) + " (Value: " + x[c] + ")")
                );
            throw new DetailedConstraintException(
                String.Join(Environment.NewLine, errorQuery), ex);
        }
        return table;
    }
}
internal class DetailedConstraintException : ConstraintException
{
    public DetailedConstraintException(string constraintErrorMessage,
        ConstraintException constraintException)
        : base(constraintErrorMessage, constraintException)
    { }
}
</pre>

Here is an example of calling this method:

<pre class="brush: csharp;">
var someData = new SomeDataSet().SomeTypedTable.GetDataSafely(
new SomeTypedTableAdapter());
</pre>

This would be equivalent to the following, standard approach:

<pre class="brush: csharp;">
var someData = new SomeDataSet().SomeTypedTable;
var adapter = new SomeTypedTableAdapter();
adapter.Fill(someData);
</pre>

And yes, I’m aware that this approach is even simpler, but it doesn’t work with the above safe method:

<pre class="brush: csharp;">
var someData = new SomeTypedTableAdapter().GetData();
</pre>

The reason it doesn’t work is because we need the dataset to be created so that we can disable constraints before filling the datatable by calling:

<pre class="brush: csharp;">
table.DataSet.EnforceConstraints = false;
dynamic dynAdapter = adapter;
dynAdapter.Fill(table);
</pre>

Then we enable constraints after the datatable has been filled:

<pre class="brush: csharp;">
try
{
    table.DataSet.EnforceConstraints = true;
}
</pre>

As soon as we set EnforceConstraints to true, all constraints are checked against every record in the table. If a ConstraintException is thrown, we can:

 1. Check for any error’d rows (table.GetErrors())
 2. For each error’d row, get the violated columns (GetColumnsInError())
 3. Finally, for each error’d row, get the specific error message per column (GetColumnError(column))

We use LINQ here to make the process of iterating over all of these errors very simple; in the end, we’re left with an enumeration of string error messages. We can then pass these detailed error messages to an Exception and our debugging process will be much easier:

<pre class="brush: csharp;">
var errorQuery = table.GetErrors()
    .SelectMany(x => x.GetColumnsInError()
        .Select(c => x.GetColumnError(c) + " (Value: " + x[c] + ")"));
</pre>

Hope this helps someone!