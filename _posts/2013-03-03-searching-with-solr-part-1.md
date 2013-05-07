--- 
layout: post
title: "Searching with SOLR - Part 1"
author: "dan"
comments: true
tags:
- solr
- lucene
- search
---

In this series of blog posts, I'm going to walk through the process of getting up and running with using Apache Solr in ASP.NET MVC. This post, the first in the series, will offer background information and show how to get Solr installed.

- Part 1 (Background, installation)
- [Part 2 (Custom schema, data import)](/2013/03/28/searching-with-solr-part-2.html)

## Some Background

Lately, the projects I've worked on have all required fast and accurate search capabilities. When you start really getting into implementing search in an application, it quickly becomes apparent that a `WHERE MyField LIKE '%abc'` isn't going to cut it. Let's say your app is an online computer hardware store and a user is searching for a 500 GB SATA hard drive. If the user types in '500 gb sata', you might treat that as:

<pre class="brush: sql;">
    SELECT * FROM Products
    WHERE Name LIKE '%500 gb sata%'
</pre>

But this is far too restrictive. What if the title of the product is '500 GB AwesomeSauce SATA Hard Drive'? In that case, the product wouldn't be returned. OK, let's separate the search terms by splitting on whitespace:

<pre class="brush: sql;">
    SELECT * FROM Products
    WHERE Name LIKE '%500%' OR Name LIKE '%gb%' OR Name LIKE '%sata%'
</pre>

OK, but what if the search terms aren't in the **Name** field? What if the name is 'AwesomeSauce SATA Hard Drive', and the fact that it's 500 GB is somewhere in the **Description** field?

<pre class="brush: sql;">
    SELECT * FROM Products
    WHERE Name LIKE '%500%' OR Name LIKE '%gb%' OR Name LIKE '%sata%'
    OR Description LIKE '%500%' OR Description LIKE '%gb%' OR Description LIKE '%sata%'
</pre>

But now we're now making our search so loose that, if we have a large product catalog, the user will get so many results that they might not even be able to find the item they're searching for.

Obviously, we need something better; something that will eliminate common stop words and figure out which results are truly **relevant** to the results. Oh, and it should rank them so that the most relevant results appear on top. And it should be fast -- **really fast**.

The [Apache Lucene](http://lucene.apache.org/core/) project is a full-fledged indexing/searching library that does all of the above things, and more. I've been using the .NET port of it, [Lucene.NET](http://lucenenet.apache.org/), for a while now at my job, and it is ridiculously powerful. Almost linear with its power, however, is its complexity. Getting up and running with Lucene can be a little daunting, and I strongly recommend a book like [Lucene in Action](http://www.manning.com/hatcher2/). Furthermore, at the time you decide to use Lucene, you have to make several administrative decisions: where will the Lucene index be stored? How will we restrict access to it? Which applications will be writing to the Lucene index, and which will be read-only? Do we have to worry about concurrency issues?

You can certainly make all of these decisions yourself (we did), but once you get more applications using Lucene and more than one index running, it might be useful to have a tool to help you out.

That tool, and the real subject of this blog post, is [Apache Solr](http://lucene.apache.org/solr/). In a nutshell, Solr is a search engine application that uses Lucene to search and index documents. Solr is a Java application that runs in any Java servlet container (like Tomcat or Jetty), but it is basically platform agnostic because your apps will interact with it using "simple REST-like services based on proven standards of XML, JSON, and HTTP." (Solr in Action) Plus, the devs working on Solr have made it ridiculously simple to install -- in fact, there really isn't any installation needed, because it ships with a Jetty container built-in. This means that once you unzip the files, all you do is start up the app and you're good to go.

## Installing Solr

Now that we've established reasoning behind why we'd want to use Solr, let's install it! The latest version can be downloaded from the [Apache Solr website](http://lucene.apache.org/solr/downloads.html), which at the time of this post is version 4.1.

**Prerequisite:** You must have the Java Runtime Environment (JRE) installed. [Get it here](http://www.oracle.com/technetwork/java/javase/downloads/index.html)

1. Download the zip file and extract everything inside it to somewhere on your computer like **c:\solr\**.
2. Open a command prompt.
3. `CD c:\solr\example\`
4. `java -jar start.jar`
5. Open up a web browser
6. Navigate to <http://localhost:8983/solr>
7. Disco!

![Screenshot of the Solr dashboard][1]

## What Just Happened?

Yeah, it was that easy. OK, you haven't really done anything yet, but Solr is running, and you have a slick dashboard.

That **example** folder is an example of a Solr instance. The example is extremely flexible and has a lot of things already configured, like a document schema and some indexing options. In reality you'd want to copy this example into a new folder so that you can delete the stuff you don't need (we'll do that in a later blog post), but for this tutorial it's fine.

Now that Solr is running, let's index some data and try out a query! The example instance comes with one index named **collection1**. It also comes with some example data that is ready to be indexed.

<pre class="brush: ps;">
    CD C:\solr\example\exampledocs
    java -jar post.jar *.xml
</pre>

You'll see the following output:

<pre class="brush: ps;">
    SimplePostTool version 1.5
    Posting files to base url http://localhost:8983/solr/update using content-type a
    pplication/xml..
    POSTing file gb18030-example.xml
    POSTing file hd.xml
    POSTing file ipod_other.xml
    POSTing file ipod_video.xml
    POSTing file manufacturers.xml
    POSTing file mem.xml
    POSTing file money.xml
    POSTing file monitor.xml
    POSTing file monitor2.xml
    POSTing file mp500.xml
    POSTing file sd500.xml
    POSTing file solr.xml
    POSTing file utf8-example.xml
    POSTing file vidcard.xml
    14 files indexed.
    COMMITting Solr index changes to http://localhost:8983/solr/update..
</pre>

Looks like some files were indexed! To check them out, direct your web browser to <http://localhost:8983/solr/#/collection1/query> and click **Execute Query**. You'll see a response, in XML, that includes a bunch of information, including some documents! For example, here's a 500GB SATA hard drive:

<pre class="brush: xml;">
    &lt;doc>
        &lt;str name="id">6H500F0&lt;/str>
        &lt;str name="name">Maxtor DiamondMax 11 - hard drive - 500 GB - SATA-300&lt;/str>
        &lt;str name="manu">Maxtor Corp.&lt;/str>
        &lt;str name="manu_id_s">maxtor&lt;/str>
        &lt;arr name="cat">
          &lt;str>electronics&lt;/str>
          &lt;str>hard drive&lt;/str>
        &lt;/arr>
        &lt;arr name="features">
          &lt;str>SATA 3.0Gb/s, NCQ&lt;/str>
          &lt;str>8.5ms seek&lt;/str>
          &lt;str>16MB cache&lt;/str>
        &lt;/arr>
        &lt;float name="price">350.0&lt;/float>
        &lt;str name="price_c">350,USD&lt;/str>
        &lt;int name="popularity">6&lt;/int>
        &lt;bool name="inStock">true&lt;/bool>
        &lt;str name="store">45.17614,-93.87341&lt;/str>
        &lt;date name="manufacturedate_dt">2006-02-13T15:26:37Z&lt;/date>
        &lt;long name="_version_">1428538688513507328&lt;/long>
    &lt;/doc>
</pre>

As you can see from the sample doc, Solr supports many different field types, such as strings, floats, and ints. Let's try a query similar to the one we discussed at the beginning of this blog post. On the query page, in the field that is labeled **q**, type '500 gb sata' and hit 'Execute Query':

![sample solr query for 500 gb sata][2]

It works! So now we have an example solr instance running with example data, and we can indeed perform queries on it. In fact, you don't even need to use the UI to perform these queries. If you examine the web request, all it is doing is making a GET request to <http://localhost:8983/solr/collection1/select?q=500+gb+sata&wt=xml&indent=true>. Don't want to return xml? Change the `wt=xml` to `wt=json` and execute the request:

<pre class="brush: javascript;">
    {
      "responseHeader":{
        "status":0,
        "QTime":0,
        "params":{
          "indent":"true",
          "q":"500 gb sata",
          "wt":"json"}},
      "response":{"numFound":3,"start":0,"docs":[
          {
            "id":"6H500F0",
            "name":"Maxtor DiamondMax 11 - hard drive - 500 GB - SATA-300",
            "manu":"Maxtor Corp.",
            "manu_id_s":"maxtor",
            "cat":["electronics",
              "hard drive"],
              // .
              // snip
              // .
    }}
</pre>

That means that if you know how to programmatically make web requests, you can now programmatically query against Solr.

In the next post, I'm going to walk through messing around with the schema, ditching the example docs, and loading some of our own data into Solr. Stay tuned!

  [1]: /img/blog/solr-dashboard.png
  [2]: /img/blog/solr-query-1.png