# Indexes : JavaScript Indexes

This feature was created for users who want to create an index and prefer JavaScript over C#.   
JavaScript indexes can be defined by a user with lower permissions than the C# indexes (admin not required).   
All other capabilities and features are the same as C# indexes.   

{INFO:Information}
This is an experimental feature.
You need to enable the featured by adding the following key to your [settings.json](../server/configuration/configuration-options#json) file:

{CODE-BLOCK:json}
{
    ...
    "Features.Availability": "Experimental"
    ...
}
{CODE-BLOCK/}
{INFO/}
## Map index

`Map` indexes, sometimes referred to as simple indexes, contain one (or more) mapping functions that indicate which fields from the documents should be indexed. 
They indicate which documents can be searched by which fields.

{CODE-BLOCK:json}
   map(<collection-name>, function (document){
        return {
            // indexed properties go here e.g:
            // Name: document.Name
        };
    })
{CODE-BLOCK/}

### Example I - Simple map index

{CODE javaScriptindexes_6@Indexes\JavaScript.cs /}

### Example II - Map index with additional sources

{CODE indexes_2@Indexes\JavaScript.cs /}

Read more about map indexes [here](../indexes/map-indexes).

## Multi map index

Multi-Map indexes allow you to index data from multiple collections

### Example

{CODE multi_map_5@Indexes\JavaScript.cs /}

Read more about multi map indexes [here](../indexes/map-reduce-indexes).

## Map-Reduce index
Map-Reduce indexes allow you to perform complex aggregations of data.
The first stage, called the map, runs over documents and extracts portions of data according to the defined mapping function(s).
Upon completion of the first phase, reduction is applied to the map results and the final outcome is produced.

{CODE-BLOCK:json}
   groupBy(x => {map properties})
        .aggregate(y => {
            return {
                // indexed properties go here e.g:
                // Name: y.Name
            };
        })
{CODE-BLOCK/}

### Example I

{CODE map_reduce_0_0@Indexes\JavaScript.cs /}

### Example II

{CODE map_reduce_3_0@Indexes\JavaScript.cs /}

Read more about map reduce indexes [here](../indexes/multi-map-indexes).

{INFO:Information}
Supported JavaScript version : ECMAScript 5.1
{INFO/}

## Related Articles

### Indexes

- [Indexing Related Documents](../indexes/indexing-related-documents)
- [Map Indexes](../indexes/map-indexes)
- [Map-Reduce Indexes](../indexes/map-reduce-indexes)
- [Creating and Deploying Indexes](../indexes/creating-and-deploying)

### Querying
- [Basics](../indexes/querying/basics)