﻿# Index Cleanup
---

{NOTE: }

* Indexes are essential for accelerating data retrieval and query performance.
 
* However, over time, an accumulation of indexes may lead to redundant, unused, or overlapping indexes  
  that can consume resources and slow down the system since every time data is inserted, updated, or deleted, the corresponding indexes must be updated.

* To address this, RavenDB offers a set of **index cleanup options** that can help you maintain a lean and efficient indexing setup, ensuring that only valuable indexes are retained.
  Reducing the number of indexes the database needs to update each time data is modified can improve indexing performance and reduce resource usage.
 
    {WARNING: }
    Note that when merging or removing any indexes via Index Cleanup,  
    you will need to update the index references in your application's client code.
    {WARNING/}

* In this page:  
  * [Merge indexes](../../../studio/database/indexes/index-cleanup#merge-indexes)  
  * [Remove sub-indexes](../../../studio/database/indexes/index-cleanup#remove-sub-indexes)  
  * [Remove unused indexes](../../../studio/database/indexes/index-cleanup#remove-unused-indexes)  
  * [Indexes that will not be merged](../../../studio/database/indexes/index-cleanup#indexes-that-will-not-be-merged)  
  * [Merge suggestions errors](../../../studio/database/indexes/index-cleanup#merge-suggestions-errors)  

{NOTE/}

---

{PANEL: Merge indexes}

The merge indexes option allows you to combine multiple indexes that serve similar purposes into a single, unified index.
Note that the server only suggests merging indexes that index data from the same collection.

Once a new merged index definition is created, the original indexes can be removed.

---

For example, consider the following 2 indexes.  
Both indexes index data from the _Employees_ collection and share the common index-field `Title`.

![Index cleanup - Index 1](images/index-cleanup-1.png "Index 1: EmployeesByFirstNameAndTitle")

![Index cleanup - Index 2](images/index-cleanup-2.png "Index 2: EmployeesByLastNameAndTitle")

---

Open the Index Cleanup view in the Studio (**Indexes > Index Cleanup view**).  
The server will suggest merging those two indexes:

![Index cleanup - Merge indexes](images/index-cleanup-3.png "Merge indexes")

1. Go to Indexes > Index Cleanup.
2. Open the **Merge indexes** tab.
3. The suggested indexes that can be merged.
4. Click to review the suggested new index.

![Index cleanup - Merge indexes](images/index-cleanup-4.png "Review suggested merge")

1. Provide a name for the new index.
2. This is the suggested map function that combines the index-fields from the original two indexes.  
   In this example the new index contains all 3 fields: `FirstName`, `LastName`, and `Title`.
3. Before saving this index:  
   * Review the suggested index-fields.     
   * In addition, you can configure any index-field or customize the indexing configuration for this new index as needed.

---

After saving the new index, the next dialog will ask you whether you wish to delete the original two indexes.

![Index cleanup - Delete original indexes](images/index-cleanup-5.png "Delete original indexes")

{PANEL/}

{PANEL: Remove sub-indexes}

If an index is completely covered by another index (i.e., all its fields are present in a larger index),  
maintaining it adds unnecessary overhead without providing additional value.

You can remove the sub-index without losing any query optimization benefits.

---

For example, consider the following 2 indexes.  
All index-fields of the `ProductsByNameAndSupplier` index are contained within the `ProductsByNameSupplierAndCategory` index.

![Index cleanup - Index 3](images/index-cleanup-6.png "Index 1: ProductsByNameAndSupplier")

![Index cleanup - Index 4](images/index-cleanup-7.png "Index 2: ProductsByNameSupplierAndCategory")

---

The server will suggest removing the sub-index `ProductsByNameAndSupplier`:

![Index cleanup - Remove sub-index](images/index-cleanup-8.png "Remove sub-index")

1. Go to Indexes > Index Cleanup and open the **Remove sub-indexes** tab.
2. The sub-indexes that can be safely deleted are listed here.
3. Click to delete the selected sub-indexes.

{PANEL/}

{PANEL: Remove unused indexes}

Indexes that haven’t been queried for a period of time still consume resources,  
such as storage space and processing power needed to keep indexed data up-to-date.

The **Remove unused indexes** tab lists indexes that have Not been queried for over a week.  
Review the list and consider deleting any unnecessary indexes.

![Index cleanup - Remove unused indexes](images/index-cleanup-9.png "Remove unused indexes")

1. Go to Indexes > Index Cleanup and open the **Remove unused indexes** tab.
2. The indexes that have not been queried for over a week are listed here.  
   Select the indexes you wish to delete.
3. Click to delete the selected indexes.

{PANEL/}

{PANEL: Indexes that will not be merged}

* **Static-indexes** that belong to the following categories will Not be considered by the server for merging or for removing a sub-index.
  These indexes will be listed under the **Unmergeable indexes** tab, with a specific reason provided for each index.
   * [Map-reduce indexes](../../../indexes/map-reduce-indexes)
   * [Multi map indexes](../../../indexes/multi-map-indexes)
   * [Fanout indexes](../../../indexes/indexing-nested-data#fanout-index---multiple-index-entries-per-document)
   * [Indexes that use a `where` clause](../../../indexes/map-indexes#filtering-data-within-fields) in their index definition
   * [Indexes that load a related document](../../../indexes/indexing-related-documents) in their index definition
   * Indexes that use `let` in their LINQ index definition - only when deployed via the [PutIndexesOperation](../../../client-api/operations/maintenance/indexes/put-indexes#put-indexes-operation-with-indexdefinition) syntax
   * Indexes for which the server did not find any other index to merge with

* [JavaScript indexes](../../../indexes/javascript-indexes) are also excluded from merging but are Not listed in this view.

* **Auto-indexes** are automatically merged by the server and will Not be listed in this view.  

---

![Index cleanup - Unmergeable indexes](images/index-cleanup-10.png "Unmergeable indexes")

1. Go to Indexes > Index Cleanup and open the **Unmergeable indexes** tab.
2. The indexes that are not mergeable are listed here.

{PANEL/}

{PANEL: Merge suggestions errors}

The **Merge suggestions errors** tab lists errors that were encountered while trying to create index merge suggestions.
This tab will only appear if the server has encountered any such errors.

![Index cleanup - Merge suggestions errors](images/index-cleanup-11.png "Merge suggestions errors")

1. The **Merge suggestions errors** tab will appear only if the server has encountered any such errors.
2. Indexes that caused the errors are listed here.
3. Click _Show details_ to view the full error information.

{PANEL/}

## Related Articles

### Indexes

- [Map indexes](../../../indexes/map-indexes)
- [Multi-Map indexes](../../../indexes/multi-map-indexes)
- [Map-Reduce indexes](../../../indexes/map-reduce-indexes)
- [Fanout indexes](../../../indexes/indexing-nested-data#fanout-index---multiple-index-entries-per-document)

### Studio

- [Indexes: Overview](../../../studio/database/indexes/indexes-overview)
- [Index List View](../../../studio/database/indexes/indexes-list-view)
- [Create Multi-Map Index](../../../studio/database/indexes/create-multi-map-index)
- [Create Map-Reduce Index](../../../studio/database/indexes/create-map-reduce-index)
- [Index History](../../../studio/database/indexes/index-history)
