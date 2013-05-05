--- 
layout: post
title: "Searching with SOLR - Part 2"
author: "dan"
comments: true
tags:
- solr
- lucene
- search
---

In the [last post](/blog/searching-with-solr-part-1) I introduced Solr, explained why it was helpful over straight Lucene, and showed how to get it up and running with some example data.

- [Part 1 (Background, installation)](http://danpincas.azurewebsites.net/blog/searching-with-solr-part-1)
- Part 2 (Custom schema, data import)

In this post, I'd like to show how we can quickly get Solr loaded with our own data.

## Picking a Data Set

For the purposes of this tutorial, let's use a sample data set everyone's familiar with: [AdventureWorks](http://msftdbprodsamples.codeplex.com/releases/view/93587)! (I used the 2008R2 version) This is a great sample data set to use because you can download it, attach the MDF to your local SQL Server Express instance, and be ready to go in minutes.

The AdventureWorks database has a huge variety of tables to pick from, and since we're testing a search engine, we want some data with a fair amount of text. I would have loved to use the *Production.ProductReview* table but it only had four rows, which isn't very fun. Instead, let's look at Products, Models, and Product Descriptions:

<pre class="brush: sql;">
SELECT      p.ProductID, p.Name, p.ProductNumber, pm.Name AS ModelName,
            pd.[Description], pc.Name AS CategoryName, ps.Name AS SubcategoryName
FROM        Production.Product p
INNER JOIN  Production.ProductSubcategory ps
ON          p.ProductSubcategoryID = ps.ProductSubcategoryID
INNER JOIN  Production.ProductCategory pc
ON          ps.ProductCategoryID = pc.ProductCategoryID
INNER JOIN  Production.ProductModel pm
ON          pm.ProductModelID = p.ProductModelID
INNER JOIN  Production.ProductModelProductDescriptionCulture pmpdc
ON          pm.ProductModelID = pmpdc.ProductModelID
INNER JOIN  Production.Culture c
ON          pmpdc.CultureID = c.CultureID
INNER JOIN  Production.ProductDescription pd
ON          pmpdc.ProductDescriptionID = pd.ProductDescriptionID
WHERE       c.Name = 'English'
</pre>

Note that in the above query we're limiting the result data set to English. Multilingual queries are definitely possible in Solr but are outside the scope of this post. Keep in mind that if you're going to be indexing data in multiple languages, you'll [probably want to run multiple Solr cores](http://stackoverflow.com/questions/6439019/single-or-multi-core-solr), one for each language that you're indexing. I'm not much more familiar with it than that, so let me know if you have any suggestions and I'll edit this post.

The above query will return 294 products. Not too bad, but definitely just scratching the surface of what Solr can do in terms of indexing performance.

![example result of running above SQL query against AdventureWorks][1]

Note that although all of the information is about products, we'll probably want to index these fields in different ways. For example, when a user searches by product number, will we want to return them similar or partial matches, or only an exact match? If a user searches the description, on the other hand, we'll probably want a fairly loose search, instructing Solr to analyze the text and break it apart.

## Preparing Solr for our Data

In the previous post we just extracted Solr and ran the example instance. We're going to need to make some changes to that if we want it to work properly with our data.

> Before you proceed, make sure your Solr instance is not running

First, let's copy the *example* folder and rename it. This way, if we ever mess up we can go back to the example folder that was already working.

<pre class="brush: ps">
CD c:\solr
MKDIR adventureworks
XCOPY example\* adventureworks /E
CD adventureworks
RMDIR /S /Q exampledocs
RMDIR /S /Q example-DIH
RMDIR /s /Q multicore
</pre>

That should leave you with a clean copy of a Solr instance, without a lot of the example stuff. Now let's make a few more changes:

1. Navigate to the **adventureworks\solr** folder. This is where your Solr instance keeps track of all of its cores.
2. You should still see the **collection1** core that was used in the previous post. Let's rename it.
3. Rename **collection1** to **products**.
4. Open **solr.xml** and replace everything that says **collection1** with **products**.
5. Open up the **products** folder. You'll see two folders, **conf** and **data**. **conf** is, as you might have guessed, where all of the configuration files for the **products** core are located. **data** is where the actual indexed information is stored on the file system.
6. Since this **data** folder is left over from the example core, let's delete it.
7. In the **conf** folder, open up the **schema.xml** file. This is where each core specifies what kind of data (what fields) the index can handle, and how to index those fields. The default schema you get out of the box is very flexible, but let's custom tailor it to our data.

## Schema Configuration

First, rename the schema. (This is only for display purposes, but hey, let's be neat ;-)

<pre class="brush: xml">
&lt;schema name="adventureworks_products" version="1.5">
</pre>

If you scroll down, the next thing you'll notice is all of the fields. By default, you'll see fields like id, sku, name, manu (manufacturer), cat (category), and more. All of these fields have different types. All of these types actually roll up to Java classes and/or primitives, such as `String`, `int` and `boolean`. A `string` type is plain text, great for text that should be indexed as-is and left alone. This is especially useful if the user won't be 'searching' on that field, but instead selecting it from a list (e.g. product categories). The `text_general`, `text_en`, `text_en_splitting`, and `text_en_splitting_tight` field types are all great for text that will be searched, but with varying options depending on the kind of text. For example, the **schema.xml** comments describe `text_en` as:

> A text field with defaults appropriate for English:
> it tokenizes with StandardTokenizer, removes English stop words
> (lang/stopwords_en.txt), down cases, protects words from protwords.txt,
> and finally applies Porter's stemming.  The query time analyzer
> also applies synonyms from synonyms.txt.

This sounds great for our Product Description field! But what about Product Numbers? We probably shouldn't treat a Product Number as normal English text. The comments describe the `text_en_splitting_tight` as:

> Less flexible matching, but less false matches.  Probably not ideal for product names,
> but may be good for SKUs.  Can insert dashes in the wrong place and still match.

This sounds ideal for the Product Number. You can learn about all of the field types by reading the comments in the `<types>` section of **schema.xml**. Note that there's nothing special about these fields, per se -- they all roll up to the Java class `solr.TextField`. However, it's the different tokenizers and filters applied to them that makes the difference. For example, do we convert the string to lowercase before indexing it? Do we 'stem' words so that words like 'housing' and 'purchasing' become 'house' and 'purchase'? How do we know when to split a string into 'words', anyway? (What counts as a delimiter) It's worth spending some time in **schema.xml** and examining the different field types and their properties.

Let's use the following field definitions for our products data set:

<pre class="brush: xml">
&lt;field name="id" type="string" indexed="true" stored="true" required="true" /&gt;
&lt;field name="product_number" type="text_en_splitting_tight" indexed="true" stored="true" omitNorms="true" /&gt;
&lt;field name="model_name" type="text_en" indexed="true" stored="true" omitNorms="true" /&gt;
&lt;field name="name" type="text_en" indexed="true" stored="true" /&gt;
&lt;field name="product_description" type="text_en" indexed="true" stored="true" /&gt;
&lt;field name="product_category" type="string" indexed="true" stored="true" /&gt;
&lt;field name="product_subcategory" type="string" indexed="true" stored="true" /&gt;
</pre>

Leave all of the other fields (after the first batch of fields) alone (e.g. the common metadata fields).

Now scroll down to where you see several `<copyField>` tags. Copyfield commands are useful when you want to simplify your search query by only searching one field for a variety of different types of data. It's also useful when you want to index one field multiple ways. Replace the first batch of copy fields with the following:

<pre class="brush: xml">
&lt;copyField source="name" dest="text" /&gt;
&lt;copyField source="product_number" dest="text" /&gt;
&lt;copyField source="product_description" dest="text" /&gt;
&lt;copyField source="model_name" dest="text" /&gt;
</pre>

And that's it for modifying the schema! Now, let's tell Solr how to query AdventureWorks and index our data.

## Data Import Handler

Solr uses something called a Data Import Handler (DIH) to run batch imports from an external data source. To get started configuring one, you need a data configuration file. Go to **c:\solr\adventureworks\solr\products\conf** and create a file named **data-config.xml**. Put this in the file:

<pre class="brush: xml">
&lt;dataConfig&gt;  
  &lt;dataSource type="JdbcDataSource" name="ds1"
        driver="com.microsoft.sqlserver.jdbc.SQLServerDriver"
        url="jdbc:sqlserver://localhost;databaseName=AdventureWorks2008R2"
        user="SQL USERNAME"
        password="SQL PASSWORD" /&gt;
  &lt;document&gt;
    &lt;entity name="data" dataSource="ds1" pk="key"
    query="SELECT p.ProductID, p.Name, p.ProductNumber, pm.Name AS ModelName, pd.[Description], pc.Name AS CategoryName, ps.Name AS SubcategoryName FROM Production.Product p INNER JOIN Production.ProductSubcategory ps ON p.ProductSubcategoryID = ps.ProductSubcategoryID INNER JOIN Production.ProductCategory pc ON ps.ProductCategoryID = pc.ProductCategoryID INNER JOIN Production.ProductModel pm ON pm.ProductModelID = p.ProductModelID INNER JOIN Production.ProductModelProductDescriptionCulture pmpdc ON pm.ProductModelID = pmpdc.ProductModelID INNER JOIN Production.Culture c ON pmpdc.CultureID = c.CultureID INNER JOIN Production.ProductDescription pd ON pmpdc.ProductDescriptionID = pd.ProductDescriptionID WHERE c.Name = 'English'"&gt;
      &lt;field column="ProductID" name="id" /&gt;
      &lt;field column="ProductNumber" name="product_number" /&gt;
      &lt;field column="ModelName" name="model_name" /&gt;
      &lt;field column="Description" name="product_description" /&gt;
      &lt;field column="CategoryName" name="product_category" /&gt;
      &lt;field column="SubcategoryName" name="product_subcategory" /&gt;
    &lt;/entity&gt;
  &lt;/document&gt;
&lt;/dataConfig&gt;
</pre>

The contents of this file are fairly self-explanatory: we first describe a data source, specifying its driver, url, username, password, and then describe the query that is associated with that data source. A query is represented by a `<document>` tag, meaning each result from this query will go into one Lucene document. Inside the query we specify the fields that we care about and associate them to the Solr fields in our **schema.xml** file.

Now, we have to tell solr that we want to use a data import handler with this configuration file. Open up your **solrconfig.xml** file (should be located in **c:\solr\adventureworks\solr\products\conf\solrconfig.xml**) and, anywhere in the request handlers section, add the following:

<pre class="brush: xml">
&lt;requestHandler name="/dataimport" class="org.apache.solr.handler.dataimport.DataImportHandler"&gt;
  &lt;lst name="defaults"&gt;
    &lt;str name="config"&gt;data-config.xml&lt;/str&gt;
  &lt;/lst&gt;
&lt;/requestHandler&gt;
</pre>

Then, near the top of this file, tell Solr to load the libraries that deal with data import:

<pre class="brush: xml">
&lt;lib dir="../../../dist/" regex="solr-dataimporthandler-.*\.jar"&gt;&lt;/lib&gt;
</pre>

Almost done! We just need to tell Solr about our driver, which in this case is the Microsoft-provided JDBC-->SQL Server driver. To have Solr load this driver, [download it](http://msdn.microsoft.com/en-us/sqlserver/aa937724.aspx) and, after installation, copy the **sqljdbc4.jar** file to **c:\solr\adventureworks\solr\products\lib\**. (If no **lib** folder exists, create one)

Now, start Solr:

<pre class="brush: ps">
java -jar start.jar
</pre>

Solr should start up without any errors.

**Note:** Depending on how much you've modified your schema, you may see the following error on startup:

> Caused by: java.lang.NumberFormatException: For input string: "MA147LL/A"

If you see this error, it's related to the **elevate.xml** file. Delete everything inside the `<elevate>` tag and the error should go away. For more information, see here: <http://wiki.apache.org/solr/QueryElevationComponent#elevate.xml>

When you open up your Solr instance in your web browser, you should see the **products** core and when you expand it, you should be able to click on the **Data Import** button and see a screen like this:

![solr data import handler screen][2]

Let's run a full import! Make sure you have the **commit** checkbox checked, and then click the **Execute** button. In the console window for Solr you should see something like:

<pre class="brush: ps">
INFO: [products] webapp=/solr path=/dataimport params={optimize=false&clean=fals
e&indent=true&commit=true&verbose=false&command=full-import&debug=false&wt=json}
 status=0 QTime=27 {add=[680 (1430726226597642240), 706 (1430726226680479744), 7
07 (1430726226682576896), 708 (1430726226684674048), 709 (1430726226685722624),
710 (1430726226686771200), 711 (1430726226688868352), 712 (1430726226689916928),
 713 (1430726226690965504), 714 (1430726226692014080), ... (294 adds)],commit=}
</pre>

Also, if you click the *Refresh Status* button you should see a success message:

![solr data import complete][3]

294 adds! Perfect! If we do a query in Solr, we certainly see that there are 294 documents.

Now let's try querying on a term like 'aluminum': `http://localhost:8983/solr/products/select?q=aluminum&wt=xml&indent=true`

Running the above query, you should get back 105 documents. OK, but that's nothing special.

What if I want to search for a product by a product number? Recall that we set up the schema to use `text_en_splitting_tight` for the product numbers, so we should be able to get the product whose number is FR-R92B-58 by searching for FRR92B58. Running the query `http://localhost:8983/solr/products/select?q=product_number%3AFRR92B58&wt=xml&indent=true` returns the matching result. Success!

  [1]: /img/blog/adventureworks_query_result.png
  [2]: /img/blog/solr_data_import_handler.png
  [3]: /img/blog/solr_data_import_complete.png