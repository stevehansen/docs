# FAQ: Transaction Support in RavenDB

---

{NOTE: }  

* In this page:
  * [ACID storage](../../client-api/faq/transaction-support#acid-storage)  
  * [What is and what isn't a transaction](../../client-api/faq/transaction-support#what-is-and-what-isn)  
  * [Working with transactions in RavenDB ](../../client-api/faq/transaction-support#working-with-transactions-in-ravendb)
     * [Single-node model](../../client-api/faq/transaction-support#single-node-model)
     * [Multi-master model](../../client-api/faq/transaction-support#multi-master-model)
     * [Cluster-wide transactions](../../client-api/faq/transaction-support#cluster-wide-transactions)
  * [ACID for document operations](../../client-api/faq/transaction-support#acid-for-document-operations)
  * [BASE for query operations](../../client-api/faq/transaction-support#base-for-query-operations)

{NOTE/}  

---

{PANEL: ACID storage}

All storage operations performed in RavenDB are fully ACID-compliant (Atomicity, Consistency, Isolation, Durability),  
this is because internally RavenDB uses a storage engine called [Voron](../../server/storage/storage-engine), built specifically for RavenDB's usage,
which guarantees all ACID properties, whether executed on document, index or local cluster data.

{PANEL/}

{PANEL: What is and what isn't a transaction}

* A transaction represents a set of operations executed against a database as a single, atomic, and isolated unit.

* In RavenDB, a transaction (read or write) is limited to the scope of a __single__ HTTP request.

* The terms "ACID transaction" or "transaction" refer to the storage engine transactions. 
  Whenever a database receives an operation or batch of operations in a request, it will wrap it in a "storage transaction",  
  execute the operations and commit the transaction.

* RavenDB ensures that for a single HTTP request, all the operations in that request are transactional.  
  It employs _Serializable_ isolation for write operations and _Snapshot_ isolation for read operations.

* RavenDB doesn't support a transaction spanning __multiple__ HTTP requests. Interactive transactions are not implemented by RavenDB 
  (see [below](../../client-api/faq/transaction-support#no-support-for-interactive-transactions) for the reasoning behind this decision). 
  RavenDB offers [optimistic concurrency](../../client-api/session/configuration/how-to-enable-optimistic-concurrency) feature to achieve similar behavior.  

* The [Client API Session](../../client-api/session/what-is-a-session-and-how-does-it-work) is a pure Client API object and does not represent a transaction, 
  thus it is not meant to provide interactive transaction semantics. 
  It is entirely managed on the client side without maintaining a corresponding session state on the server. 
  The server does not reference or keep track of the session context.

{PANEL/}

{PANEL: Working with transactions in RavenDB}

### Single-node model

Transactional behavior with RavenDB is divided into two modes:

* __Single requests__:  
In this mode, a user can perform all requested operations (read and/or write) in a single request.  

  * __Multiple writes__:  
  A batch of multiple write operations will be executed atomically in a single transaction when calling [SaveChanges()](../../client-api/session/saving-changes).
  Multiple operations can also be executed in a single transaction using the low-level [SingleNodeBatchCommand](../../client-api/commands/batches/how-to-send-multiple-commands-using-a-batch).  
  In both cases, a single HTTP request is sent to the database.

  * __Multiple reads & writes__:  
  Performing interleaving reads and writes or conditional execution can be achieved by [running a patching script](../../client-api/operations/patching/single-document).  
  In the script you can read documents, make decisions based on their content and update or put document(s) within the scope of a single transaction.  
  If you only need to modify a document in a transaction, [JSON Patch syntax](../../client-api/operations/patching/json-patch-syntax) allows you to do that.

* __Multiple requests__:  
  RavenDB does not support a single transaction that spans all requested operations within multiple requests.  
  Instead, users are expected to utilize [optimistic concurrency](../../client-api/session/configuration/how-to-enable-optimistic-concurrency) to achieve similar behavior.  
  Your changes will get committed only if no one else has changed the data you are modifying in the meantime. 

#### No support for interactive transactions

RavenDB client uses HTTP to communicate with the RavenDB server. 
It means that RavenDB doesn't allow you to open a transaction on the server side, make multiple operations over a network connection, and then commit or roll it back.  
This model, known as the interactive transactions model, is incredibly costly. Both in terms of engine complexity and the impact on the overall performance of the system.

{INFO: }

In [one study](http://nms.csail.mit.edu/~stavros/pubs/OLTP_sigmod08.pdf) the cost of managing the transaction state across multiple network operations was measured at over 40% of the total system performance. 
This is because the server needs to maintain locks and state across potentially very large time frames.

{INFO/}

RavenDB's approach differs from the classical SQL model, which relies on interactive transactions. Instead, RavenDB uses the batch transaction model. It allows us to provide the same capabilities as interactive transactions in 
conjunction with [optimistic concurrency](../../client-api/session/configuration/how-to-enable-optimistic-concurrency), with much better performance.

Key to that design decision is our ability to provide similar guarantees about the state of your data without experiencing the overhead of interactive transactions.  

#### Batch transaction model

RavenDB uses the batch transaction model, where a RavenDB client submits all the operations to be run in a single transaction in one network call. 
This allows the storage engine inside RavenDB to avoid holding locks for an extended period of time and gives plenty of room to optimize the performance.

This decision is based on the typical interaction pattern by which RavenDB is used.  
RavenDB serves as a transactional system of record for business applications, where the common workflow involves presenting data to users, 
allowing them to make modifications, and subsequently save these changes.  
A single request loads the data which is then presented to the user. 
After a period of contemplation or "think time," the user submits a set of updates, which are then saved to the database.  
This model fits the batch transaction model a lot more closely than the interactive one, as there's no necessity to keep a transaction open during the user's "think time."

All changes that are sent via _SaveChanges_ are persisted in a single unit.  
If you modify documents concurrently and you want to assure they won't by affected by the lost update problem,  
then you must enable [optimistic concurrency](../../client-api/session/configuration/how-to-enable-optimistic-concurrency) (turned off by default) across all sessions that modify those documents.

<hr/>

### Multi-master model

RavenDB employs the multi-master model, allowing writes to be made to any node in the cluster.  
These writes are then propagated asynchronously to the other nodes via [replication](../../server/clustering/replication/replication). 

The interaction of transactions and distributed work is anything but trivial. Let's start from the obvious problem:

* RavenDB allows you to perform concurrent write operations on multiple nodes.  
* RavenDB explicitly allows you to write to a node that was partitioned from the rest of the network.

Taken together, this violates the [CAP theorem](https://en.wikipedia.org/wiki/CAP_theorem) 
which states that a system can only provide 2 out of 3 guarantees around consistency, availability, and partition tolerance.

RavenDB's answer to distributed transactional work is nuanced and was designed to give you as the user the choice  
so you can utilize RavenDB for each of your scenarios:  

* Single-node operations are available and partition tolerant (AP) but cannot meet the consistency guarantee.
* If you need to guarantee uniqueness or replicate the data for redundancy across more than one node,  
  you can choose to have higher consistency at the cost of availability (CP).

When running in a multi-node setup, RavenDB still uses transactions. However, they are single-node transactions.  
That means that the set of changes that you write in a transaction is committed only to the node you are writing to.  
It will then asynchronously replicate to the other nodes.  
To achieve consistency across the entire cluster please refer to the [Cluster-wide transactions](../../client-api/faq/transaction-support#cluster-wide-transactions) section below.  

#### Replication conflicts

This is an important observation because you can get into situations where two clients wrote (even with [optimistic concurrency](../../client-api/session/configuration/how-to-enable-optimistic-concurrency) turned on)
to the same document and both of them committed successfully (each one to a separate node). 
RavenDB attempts to minimize this situation by designating a [preferred node](../../client-api/configuration/load-balance/overview#the-preferred-node) for writes for each database,  
but since writing to the preferred node isn't guaranteed, this might not alleviate the issue.

In such a case, the data will replicate across the cluster, and RavenDB will detect that there were [conflicting](../../server/clustering/replication/replication-conflicts) modifications to the document. 
It will then apply the [conflict resolution](../../studio/database/settings/conflict-resolution) strategy that you choose.  
That can include selecting a manual resolution, running a [resolution script](../../server/clustering/replication/replication-conflicts#conflict-resolution-script) to reconcile the conflicting versions,  
or simply selecting the latest version. You are in control of this behavior. 

This behavior was influenced by the [Dynamo paper](https://dl.acm.org/doi/10.1145/1323293.1294281) which emphasizes the importance of writes.  
The assumption is that if you are writing data to the database, you expect it to be persisted.

RavenDB will do its utmost to provide that to you, allowing you to write to the database even in the case of partitions or partial failure states. 
However, handling replication conflicts is a consideration you have to take into account when using single-node transactions in RavenDB (see below for running a [cluster-wide transaction](../../client-api/faq/transaction-support#cluster-wide-transactions)).

{WARNING: Lost updates}

If no conflict resolution script is defined for a collection, then by default RavenDB resolves the conflict using the latest version based on the `@last-modified` property of conflicted versions of the document.  
That might result in the lost update anomaly.

If you care about avoiding lost updates, you need to ensure you have the conflict resolution script defined accordingly or use a [cluster-wide transaction](../../client-api/faq/transaction-support#cluster-wide-transactions).

{WARNING/}

#### Replication & transaction boundary

The following is an important aspect to RavenDB's transactional behavior with regards to asynchronous replication.  

When replicating modifications to another server, RavenDB will ensure that the [transaction boundaries](../../server/clustering/replication/replication#replication-&-transaction-boundary) are maintained.  
If there are several document modifications in the same transaction they will be sent in the same replication batch, keeping the transaction boundary on the destination as well.  

However, a special attention is needed when a document is modified in two separate transactions but the replication of the first transaction has not occurred yet. 
Read more about that in [How revisions replication help data consistency](../../server/clustering/replication/replication#how-revisions-replication-help-data-consistency).

<hr/>

### Cluster-wide transactions

RavenDB also supports [cluster-wide transactions](../../client-api/session/cluster-transaction/overview).  
This feature modifies the way RavenDB commits a transaction, and it is meant to address scenarios where you prefer to get a failure if the transaction cannot be persisted to a majority of the nodes in the cluster.  
In other words, this feature is for scenarios where you want to favor consistency over availability.

For cluster-wide transactions, RavenDB uses the [Raft](../../server/clustering/rachis/what-is-rachis#what-is-raft-?) protocol. 
This protocol ensures that the transaction is acknowledged by a majority of the nodes in the cluster and once committed, the changes will be visible on any node that you'll use henceforth.  

Similar to single-node transactions, RavenDB requires that you submit the cluster-wide transaction as a single request of all the changes you want to commit to the database. 

Cluster-wide transactions have the notion of [atomic guards](../../client-api/session/cluster-transaction/atomic-guards) to prevent an overwrite of a document modified in a cluster transaction by a change made in another cluster transaction.

{INFO: }

The usage of atomic guards makes cluster-wide transactions conflict-free.  
There is no way to make a conflict between two versions of the same document.  
If a document got updated meanwhile by someone else then a `ConcurrencyException` will be thrown.

{INFO/}

{PANEL/}

{PANEL: ACID for document operations}

In RavenDB all actions performed on documents are fully ACID. 
Each document operation or a batch of operations applied to a set of documents sent in a single HTTP request will execute in a single transaction.  
The ACID properties of RavenDB are:  

* __Atomicity__  
  All operations are atomic. Either they fully succeed or fail without any partial execution. 
  In particular, operations on multiple documents will be carried out atomically, meaning they are either completed entirely or not executed at all.

* __Consistency and Isolation / Consistency of Scans__  
  Within a single _read_ transaction, all operations are performed under _Snapshot_ isolation. 
  This ensures that even if you access multiple documents, you'll get all of their state exactly as it was at the beginning of the request.

* __Visibility__  
  All changes to the database are immediately made available upon commit.  
  Therefore, if a transaction updates two documents and is committed, you will always see the updates to both documents at the same time.
  That is, you either see the updates to both, or you don't see the update to either one.

* __Durability__   
  If an operation has been completed successfully, it is fsync'ed to disk.  
  Reads will never return any data that has not been flushed to disk.

All of these constraints are guaranteed for each individual request made to the database when using a [Session](../../client-api/session/what-is-a-session-and-how-does-it-work).  
In particular, every `Load` call is a separate transaction, and the [`SaveChanges`](../../client-api/session/saving-changes) 
call will encapsulate all documents created, deleted, or modified within the session into a single transaction.

{PANEL/}

{PANEL: BASE for query operations}

The transaction model is different when indexes are involved, because indexes are BASE (Basically Available, Soft state, Eventual consistency), not ACID. 
The indexing in RavenDB will always happen in the background. When you write a new document or update an existing one, RavenDB doesn't wait to update all the indexes before it completes the write operation.
Instead, it writes the document data and completes the write operation as soon as the transaction is written to disk, scheduling any index updates to occur in an async manner.

There are several reasons for this behavior:

* Writes are faster because they aren't going to be held up by the indexes.
* Indexes running in an async manner allow to handle updates in batches instead of having to update all the indexes on every write.
* Indexes are operating independently, so a single slow or expensive index isn't going to impact any other indexes or the overall write performance in the system.
* Indexes can be added dynamically and on the fly to busy production systems.
* Indexes can be updated in a [side-by-side manner](../../indexes/creating-and-deploying).

The BASE model means that the following constraints are applied to query operations:

* __Basically Available__  
  Index query results will be always available but they might be stale.

* __Soft state__  
  The state of the system could change over time because some amount of time is needed to perform the indexing. 
  This is an incremental operation; the fewer documents remain to index, the more accurate index results we have.

* __Eventual consistency__  
  The database will eventually become consistent once it stops receiving new documents and the indexing operation finishes.

The async nature of RavenDB indexes means that you need to be aware that, by default, writes will complete without waiting for indexes.
Although there are ways to wait for the indexes to complete as part of the write or even during the read (although that is not recommended). 
Please read a dedicated article about the [stale indexes](../../indexes/stale-indexes).

{PANEL/}

## Related Articles

### Server

- [Storage Engine](../../server/storage/storage-engine)
- [What is a Session and How Does it Work](../../client-api/session/what-is-a-session-and-how-does-it-work)
- [Optimistic concurrency](../../client-api/session/configuration/how-to-enable-optimistic-concurrency)
