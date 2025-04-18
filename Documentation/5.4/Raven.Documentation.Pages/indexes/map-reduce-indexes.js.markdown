# Indexes: Map-Reduce Indexes

Map-Reduce indexes allow you to perform complex aggregations of data. The first stage, called the map, runs over documents and extracts portions of data according to the defined mapping function(s).
Upon completion of the first phase, reduction is applied to the map results and the final outcome is produced.

The idea behind map-reduce indexing is that aggregation queries using such indexes are very cheap. The aggregation is performed only once and the results are stored inside the index.
Once new data comes into the database or existing documents are modified, the map-reduce index will keep the aggregation results up-to-date. The aggregations are never done during
querying to avoid expensive calculations that could result in severe performance degradation. When you make the query, RavenDB immediately returns the matching results directly from the index.

For a more in-depth look at how map reduce works, you can read this post: [RavenDB 4.0 Unsung Heroes: Map/reduce](https://ayende.com/blog/179938/ravendb-4-0-unsung-heroes-map-reduce).

{PANEL:Creating}

When it comes to index creation, the only difference between simple indexes and the map-reduce ones is an additional reduce function defined in index definition. 
To deploy an index we need to create a definition and deploy it using one of the ways described in the [creating and deploying](../indexes/creating-and-deploying) article.

### Example I - Count

Let's assume that we want to count the number of products for each category. To do it, we can create the following index using `LoadDocument` inside:

{CODE:nodejs map_reduce_0_0@indexes\mapReduceIndexes.js /}

and issue the query:

{CODE-TABS}
{CODE-TAB:nodejs:Query map_reduce_0_1@indexes\mapReduceIndexes.js /}
{CODE-TAB-BLOCK:sql:RQL}
from 'Products/ByCategory'
where Category == 'Seafood'
{CODE-TAB-BLOCK/}
{CODE-TABS/}

The above query will return one result for _Seafood_ with the appropriate number of products from that category.

### Example II - Average

In this example, we will count an average product price for each category. The index definition:

{CODE:nodejs map_reduce_1_0@indexes\mapReduceIndexes.js /}

and the query:

{CODE-TABS}
{CODE-TAB:nodejs:Query map_reduce_1_1@indexes\mapReduceIndexes.js /}
{CODE-TAB-BLOCK:sql:RQL}
from 'Products/Average/ByCategory'
where Category == 'Seafood'
{CODE-TAB-BLOCK/}
{CODE-TABS/}

### Example III - Calculations

This example illustrates how we can put some calculations inside an index using on one of the indexes available in the sample database (`Product/Sales`).

We want to know how many times each product was ordered and how much we earned for it. In order to extract that information, we need to define the following index:

{CODE:nodejs map_reduce_2_0@indexes\mapReduceIndexes.js /}

and send the query:

{CODE-TABS}
{CODE-TAB:nodejs:Query map_reduce_2_1@indexes\mapReduceIndexes.js /}
{CODE-TAB-BLOCK:sql:RQL}
from 'Product/Sales'
{CODE-TAB-BLOCK/}
{CODE-TABS/}

{PANEL/}

{PANEL:Reduce Results as Artificial Documents}

In addition to storing the aggregation results in the index, the map-reduce index can also output 
those reduce results as documents to a specified collection. In order to create these documents, 
called _"artificial",_ you need to define the target collection using the `outputReduceToCollection` 
property in the index definition.  

Writing map-reduce outputs into documents allows you to define additional indexes on top of them 
that give you the option to create recursive map-reduce operations. This makes it cheap and easy 
to, for example, recursively create daily, monthly, and yearly summaries on the same data.  

In addition, you can also apply the usual operations on artificial documents (e.g. data 
subscriptions or ETL).  

If the aggregation value for a given reduce key changes, we overwrite the output document. If the 
given reduce key no longer has a result, the output document will be removed.  

#### Reference Documents

To help organize these output documents, the map-reduce index can also create an additional 
collection of artificial _reference documents_. These documents aggregate the output documents 
and store their document IDs in an array field `ReduceOutputs`.  

The document IDs of reference documents are customized to follow some pattern. The format you 
give to their document ID also determines how the output documents are grouped.  

Because reference documents have well known, predictable IDs, they are easier to plug into 
indexes and other operations, and can serve as an intermediary for the output documents whose 
IDs are less predictable. This allows you to chain map-reduce indexes in a recursive fashion, 
see [Example II](../indexes/map-reduce-indexes#example-ii).  

Learn more about how to configure output and reference documents in the 
[Studio: Create Map-Reduce Index](../studio/database/indexes/create-map-reduce-index).  

### Artificial Document IDs

The identifiers of artificial documents are generated as:

- `<OutputCollectionName>/<hash-of-reduce-key>`

For the above sample index, the document ID can be:

- `MonthlyProductSales/13770576973199715021`

The numeric part is the hash of the reduce key values, in this case: `hash(Product, Month)`.

If the aggregation value for a given reduce key changes then we overwrite the artificial document. It will get removed once there is no result for a given reduce key.
    
### Artificial Document Flags

Documents generated by map-reduce indexes get the following `@flags` metadata:

{CODE-BLOCK:json}
"@flags": "Artificial, FromIndex"
{CODE-BLOCK/}

Those flags are used internally by the database to filter out artificial documents during replication.

### Usage

The map-reduce output documents are configured with these properties of 
`IndexDefinition`:  

{CODE:nodejs syntax_0@indexes\mapReduceIndexes.js /}

| Parameters | Type | Description |
| - | - | - |
| **outputReduceToCollection** | `string` | Collection name for the output documents. |
| **patternReferencesCollectionName** | `string` | Optional collection name for the reference documents - by default it is `<outputReduceToCollection>/References`. |
| **patternForOutputReduceToCollectionReferences** | `string` | Document ID format for reference documents. This ID references the fields of the reduce function output, which determines how the output documents are aggregated. The type of this parameter is different depending on if the index is created using [IndexDefinition](../client-api/operations/maintenance/indexes/put-indexes#put-indexes-operation) or [AbstractJavaScriptIndexCreationTask](../indexes/creating-and-deploying#define-a-static-index-using-a-custom-class). |

### Examples

{CODE:nodejs map_reduce_3_0@indexes\mapReduceIndexes.js /}

## Important Comments

#### Saving documents
[Artificial documents](../indexes/map-reduce-indexes#reduce-results-as-artificial-documents) 
are stored immediately after the indexing transaction completes.  

#### Recursive indexing loop
It is **forbidden** to output reduce results to collections such as the following:  

- A collection that the current index is already working on.  
  E.g., an index on a `DailyInvoices` collection outputs to `DailyInvoices`.  
- A collection that the current index is loading a document from.  
  E.g., an index with `LoadDocument(id, "Invoices")` outputs to `Invoices`.  
- Two collections, each processed by a map-reduce indexes,  
  when each index outputs to the second collection.  
  E.g.,  
  An index on the `Invoices` collection outputs to the `DailyInvoices` collection,  
  while an index on `DailyInvoices` outputs to `Invoices`.  

When an attempt to create such an infinite indexing loop is 
detected a detailed error is generated.  

#### Output to an Existing collection
Creating a map-reduce index which defines an output collection that already 
exists and contains documents, will result in an error.  
Delete all documents from the target collection before creating the index, 
or output results to a different collection.  

#### Modification of Artificial Documents
Artificial documents can be loaded and queried just like regular documents.  
However, it is **not** recommended to edit artificial documents manually since 
any index results update would overwrite all manual modifications made in them.  

{PANEL/}

## Related Articles

### Indexes

- [Indexing Related Documents](../indexes/indexing-related-documents)
- [Creating and Deploying Indexes](../indexes/creating-and-deploying)

### Querying

- [Query Overview](../client-api/session/querying/how-to-query)

### Studio

- [Create Map-Reduce Index](../studio/database/indexes/create-map-reduce-index)

