<properties 
	pageTitle="Sorting DocumentDB data using Order By | Azure" 
	description="Learn how to use ORDER BY in DocumentDB queries in LINQ and SQL, and how to specify an indexing policy for ORDER BY queries." 
	services="documentdb" 
	authors="arramac" 
	manager="jhubbard" 
	editor="cgronlun" 
	documentationCenter=""/>

<tags 
	ms.service="documentdb" 
	ms.workload="data-services" 
	ms.tgt_pltfrm="na" 
	ms.devlang="na" 
	ms.topic="article" 
	ms.date="06/04/2015" 
	ms.author="arramac"/>

# Sorting DocumentDB data using Order By
Microsoft Azure DocumentDB supports querying documents using SQL over JSON documents. Query results can be ordered using the ORDER BY clause in SQL query statements.

After reading this article, you'll be able to answer the following questions: 

- How do I query with Order By?
- How do I configure an indexing policy for Order By?
- What's coming next?

[Samples](#samples) and an [FAQ](#faq) are also provided.

For a complete reference on SQL querying, see the [DocumentDB Query tutorial](documentdb-sql-query.md).

## How to Query with Order By
Like in ANSI-SQL, you can now include an optional Order By clause in SQL statements when querying DocumentDB. The clause can include an optional ASC/DESC argument to specify the order in which results must be retrieved. 

### Ordering using SQL
For example here's a query to retrieve books in descending order of PublishTimestamp. 

    SELECT * 
    FROM Books 
    ORDER BY Books.PublishTimestamp DESC

### Ordering using SQL with Filtering
You can order using any nested property within documents like Books.ShippingDetails.Weight, and you can specify additional filters in the WHERE clause in combination with Order By like in this example:

    SELECT * 
    FROM Books 
	WHERE Books.SalePrice > 4000
    ORDER BY Books.ShippingDetails.Weight

### Ordering using the LINQ Provider for .NET
Using the .NET SDK version 1.2.0 and higher, you can also use the OrderBy() or OrderByDescending() clause within LINQ queries like in this example:

    foreach (Book book in client.CreateDocumentQuery<Book>(booksCollection.SelfLink)
        .OrderBy(b => b.PublishTimestamp)) 
    {
        // Iterate through books
    }

### Ordering with paging using the .NET SDK
Using the native paging support within the DocumentDB SDKs, you can retrieve results one page at a time like in the following .NET code snippet. Here we fetch results up to 10 at a time using the FeedOptions.MaxItemCount and the IDocumentQuery interface.

    var booksQuery = client.CreateDocumentQuery<Book>(
        booksCollection.SelfLink,
        "SELECT * FROM Books ORDER BY Books.PublishTimestamp DESC"
        new FeedOptions { MaxItemCount = 10 })
      .AsDocumentQuery();
            
    while (booksQuery.HasMoreResults) 
    {
        foreach(Book book in await booksQuery.ExecuteNextAsync<Book>())
        {
            // Iterate through books
        }
    }

DocumentDB supports ordering with a single numeric, string or Boolean property per query, with additional query types coming soon. Please see [What's coming next](#Whats_coming_next) for more details.

## Configure an indexing policy for Order By
In order to execute Order By queries, you must create a collection with a custom index policy for Order By. The most common indexing policies are the following:

<table border="0" cellspacing="0" cellpadding="0">
    <tbody>
        <tr>
            <td valign="top">
                <p>
                    <strong>Indexing Policy</strong>
                </p>
            </td>
            <td valign="top">
                <p>
                    <strong>Support for Order By</strong>
                </p>
            </td>
        </tr>
        <tr>
            <td valign="top">
                <p>
                    All Hash
                </p>
            </td>
            <td valign="top">
                <p>
                    All string and numeric properties use Hash indexing. Order By is NOT supported. Has the lowest indexing storage overhead.
                </p>
            </td>
        </tr>
        <tr>
            <td valign="top">
                <p>
                    All Range
                </p>
            </td>
            <td valign="top">
                <p>
                    All string and numeric properties use Range indexing with maximum precision. Order By is supported with both numbers and strings. Has a higher index storage overhead.
                </p>
            </td>            
        </tr>
        <tr>
            <td valign="top">
                <p>
                    Default
                </p>
            </td>
            <td valign="top">
                <p>
                    All string properties use Hash indexing, and numeric properties use Range indexing with maximum precision. Order By is supported against numbers, but not strings. Has lowe indexing storage overhead.
                </p>
            </td>
        </tr>        
        <tr>
            <td valign="top">
                <p>
                    Your policy (custom)
                </p>
            </td>
            <td valign="top">
                <p>
                    Using the SDKs, you can configure specific paths for range indexing and for string/range. Order By is supported for just these properties. Has a low indexing overhead.
                </p>
            </td>            
        </tr>        
    </tbody>
</table>        
Maximum precision (represented as precision of -1 in JSON config) utilizes a variable number of bytes depending on the value that's being indexed. Therefore:

- Properties with larger number values e.g., epoch timestamps, the max precision will have a high index overhead. 
- Properties with smaller number values (enumerations, zeroes, zip codes, ages, etc.) will have a low index overhead.

For more details see [DocumentDB indexing policies](documentdb-indexing-policies.md).

### Indexing for Order By against all numeric properties
Here's how you can create a collection with indexing for Order By against all numeric or string properties that appear within JSON documents within it.
                   
   booksCollection.IndexingPolicy.IncludedPaths.Add(
    	new IncludedPath { Path = "/*", Indexes = new Collection<Index> { 
                new RangeIndex(DataType.String) { Precision = -1 }, 
                new RangeIndex(DataType.Number) { Precision = -1 } } 
    });

    await client.CreateDocumentCollectionAsync(databaseLink, 
        booksCollection);  

### Indexing for Order By for a single property
Here's how you can create a collection with indexing for Order By against just the PublishTimestamp property.                                                       
    booksCollection.IndexingPolicy.IncludedPaths.Add(
    	new IncludedPath { 
    		Path = "/shippedTimestamp/?", 
    		Indexes = new Collection<Index> { 
    			new RangeIndex(DataType.Number) { Precision = -1 } } 
    		});

    booksCollection.IndexingPolicy.IncludedPaths.Add(
    	new IncludedPath { 
    		Path = "/*" 
    	});

## Samples
Take a look at this [Github samples project](https://github.com/Azure/azure-documentdb-net/tree/master/samples/orderby) that demonstrates how to use Order By, including creating indexing policies and paging using Order By. The samples are open source and we encourage you to submit pull requests with contributions that could benefit other DocumentDB developers. Please refer to the [Contribution guidelines](https://github.com/Azure/azure-documentdb-net/blob/master/Contributing.md) for guidance on how to contribute.  

## What's coming next?

Future service updates will expand on the Order By support introduced here. We are working on the following additions and will prioritize the release of these improvements based on your feedback:

- Dynamic Indexing Policies: Support to modify indexing policy after collection creation and in the Azure Portal
- Support for Compound Indexes for more efficient Order By and Order By on multiple properties.

## FAQ

**Which platforms/versions of the SDK support ordering?**

In order to create collections with the indexing policy required for Order By, you must download the latest drop of the SDK (1.2.0 for .NET and 1.1.0 for Node.js, JavaScript, Python and Java). The .NET SDK 1.2.0 is also required to use OrderBy() and OrderByDescending() within LINQ expressions. 


**What is the expected Request Unit (RU) consumption of Order By queries?**

Since Order By utilizes the DocumentDB index for lookups, the number of request units consumed by Order By queries will be similar to the equivalent queries without Order By. Like any other operation on DocumentDB, the number of request units depends on the sizes/shapes of documents as well as the complexity of the query. 


**What is the expected indexing overhead for Order By?**

The indexing storage overhead will be proportionate to the number of properties. In the worst case scenario, the index overhead will be 100% of the data. There is no difference in throughput (Request Units) overhead between Range/Order By indexing and the default Hash indexing.

**How do I query my existing data in DocumentDB using Order By?**

This will be supported with the availability of the  Dynamic Indexing Policies improvement mentioned in the [What's Coming Next](what's-coming-next) section. In order to do this today, you have to export your data and re-import into a new DocumentDB collection created with a Range/Order By Index. The DocumentDB Import Tool can be used to migrate your data between collections. 

**What are the current limitations of Order By?**

Order By can be specified only against a numeric, string or Boolean property when it is range indexed with Max Precision (-1) indexing.

You cannot perform the following:
 
- Order By with internal string properties like id, _rid, and _self (coming soon).
- Order By with properties derived from the result of an intra-document join (coming soon).
- Order By multiple properties (coming soon).
- Order By with queries on databases, collections, users, permissions or attachments (coming soon).
- Order By with computed properties e.g. the result of an expression or a UDF/built-in function.

## Next steps

Fork the [Github samples project](https://github.com/Azure/azure-documentdb-net/tree/master/samples/orderby) and start ordering your data! 

## References
* [DocumentDB Query Reference](documentdb-sql-query.md)
* [DocumentDB Indexing Policy Reference](documentdb-indexing-policies.md)
* [DocumentDB SQL Reference](https://msdn.microsoft.com/library/azure/dn782250.aspx)
* [DocumentDB Order By Samples](https://github.com/Azure/azure-documentdb-net/tree/master/samples/orderby)
 
