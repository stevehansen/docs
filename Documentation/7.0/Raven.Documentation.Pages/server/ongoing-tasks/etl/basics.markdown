﻿# Ongoing Tasks: ETL Basics
---

{NOTE: }

* **ETL (Extract, Transform & Load)** is a three-stage RavenDB process that transfers data from a RavenDB database to an external target. 
  The data can be filtered and transformed along the way.  

* The external target can be:  
  * Another RavenDB database instance (outside of the [Database Group](../../../studio/database/settings/manage-database-group))
  * A relational database
  * Elasticsearch
  * OLAP (Online Analytical Processing)
  * A message broker such as Apache Kafka, RabbitMQ, or Azure Queue Storage

* ETL can be used on [sharded](../../../sharding/etl) and non-sharded databases alike.  
  Learn more about how ETL works on a sharded database [here](../../../sharding/etl).

* In this page:  
  * [Why use ETL](../../../server/ongoing-tasks/etl/basics#why-use-etl)  
  * [Defining ETL Tasks](../../../server/ongoing-tasks/etl/basics#defining-etl-tasks)  
  * [ETL Stages:](../../../server/ongoing-tasks/etl/basics#etl-stages)  
      * [Extract](../../../server/ongoing-tasks/etl/basics#extract)  
      * [Transform](../../../server/ongoing-tasks/etl/basics#transform)  
      * [Load](../../../server/ongoing-tasks/etl/basics#load)  
  * [Troubleshooting](../../../server/ongoing-tasks/etl/basics#troubleshooting)  
{NOTE/}

---

{PANEL: Why use ETL}

* **Share relevant data**  
  Send data in a well-defined format to match specific requirements, ensuring only relevant data is transmitted  
  (e.g., sending data to an existing reporting solution).

* **Protect your data - Share partial data**  
  Limit access to sensitive data. Details that should remain private can be filtered out as you can share partial data.  

* **Reduce system calls**  
  Distribute data across related services within your system architecture, allowing each service to access its _own copy_ of the data without cross-service calls 
  (e.g., sharing a product catalog among multiple stores).

* **Transform the data**  
  * Modify content sent as needed with JavaScript code.  
  * Multiple documents can be sent from a single source document.  
  * Data can be transformed to match the target destination's model.

* **Aggregate your data**  
  Data sent from multiple locations can be aggregated in a central server  
  (e.g., aggregating sales data from point of sales systems for centralized calculations).

{PANEL/}

{PANEL: Defining ETL Tasks}

* The following ETL tasks can be defined:  
  * [RavenDB ETL](../../../server/ongoing-tasks/etl/raven) - send data to another _RavenDB database_  
  * [SQL ETL](../../../server/ongoing-tasks/etl/sql) - send data to an _SQL database_  
  * [Snowflake ETL](../../../server/ongoing-tasks/etl/snowflake) - send data to a _Snowflake warehouse_  
  * [OLAP ETL](../../../server/ongoing-tasks/etl/OLAP) - send data to an _OLAP destination_  
  * [Elasticsearch ETL](../../../server/ongoing-tasks/etl/elasticsearch) - send data to an _Elasticsearch destination_  
  * [Kafka ETL](../../../server/ongoing-tasks/etl/queue-etl/kafka) - send data to a _Kafka message broker_  
  * [RabbitMQ ETL](../../../server/ongoing-tasks/etl/queue-etl/rabbit-mq) - send data to an _RabbitMQ exchange_  
  * [Azure Queue Storage ETL](../../../server/ongoing-tasks/etl/queue-etl/azure-queue) - send data to an _Azure Queue Storage message queue_  
  * [Amazon SQS ETL](../../../server/ongoing-tasks/etl/queue-etl/amazon-sqs) - send data to an _Amazon SQS message queue_  

* All ETL tasks can be defined from the Client API or from the [Studio](../../../studio/database/tasks/ongoing-tasks/general-info).

* The destination address and access options are set using a pre-defined **connection string**, simplifying deployment across different environments. 
  For example, with RavenDB ETL, multiple URLs can be configured in the connection string since the target database can reside on multiple nodes within the Database Group of the destination cluster. 
  If one of the destination nodes is unavailable, RavenDB automatically executes the ETL process against another node specified in the connection string. 
  Learn more in the [Connection Strings](../../../client-api/operations/maintenance/connection-strings/add-connection-string) article.  

{PANEL/}

{PANEL: ETL Stages}

ETL's three stages are:  

* [Extract](../../../server/ongoing-tasks/etl/basics#extract) - Extract the documents from the database  
* [Transform](../../../server/ongoing-tasks/etl/basics#transform) - Transform & filter the documents data according to the supplied script (optional)  
* [Load](../../../server/ongoing-tasks/etl/basics#load) - Load (write) the transformed data into the target destination  

---

### Extract

The ETL process starts with retrieving the documents from the database.  
You can choose which documents will be processed by the next two stages (Transform and Load).  

The possible options are:  

* Documents from a single collection  
* Documents from multiple collections  
* All documents

---

### Transform

* This stage transforms and filters the extracted documents according to a provided script.  
  Any transformation can be done so that only relevant data is shared.  
  The script is written in JavaScript and its input is a document.  

* A task can be provided with multiple transformation scripts.  
  Different scripts run in separate processes, allowing multiple scripts to run in parallel.  

* You can do any transformation and send only data you are interested in sharing.  
  The following is an example of RavenDB ETL script processing documents from the "Employees" collection:

    {CODE-BLOCK:javascript}

var managerName = null;

if (this.ReportsTo !== null)
{
    var manager = load(this.ReportsTo);
    managerName = manager.FirstName + " " + manager.LastName;
}

// Load the object to a target destination by the name of "EmployeesWithManager"
loadToEmployeesWithManager({
    Name: this.FirstName + " " + this.LastName,
    Title: this.Title ,
    BornOn: new Date(this.Birthday).getFullYear(),
    Manager: managerName
});
    {CODE-BLOCK/}

{NOTE: }

#### Syntax

In addition to the ECMAScript 5.1 API,  
RavenDB introduces the following functions and members that can be used in the transformation script:  

|---|---|---|
| `this` | object | The current document (with metadata) |
| `id(document)` | function | Returns the document ID |
| `load(id)` | function | Load another document.<br>This will increase the maximum number of allowed steps in a script.<br>**Note**:<br>Changes made to the other _loaded_ document will Not trigger the ETL process.|

Specific ETL functions:  

|---|---|---|
| `loadTo` | function | Load an object to the specified target.<br>This command has several syntax options,<br>see details [below](../../../server/ongoing-tasks/etl/basics#themethod).<br>**Note:**<br>An object will only be sent to the destination if the `loadTo` method is called. |
| Attachments: |||
| `loadAttachment(name)` | function | Load an attachment of the current document. |
| `hasAttachment(name)` | function | Check if an attachment with a given name exists for the current document. |
| `getAttachments()` | function | Get a collection of attachments details for the current document. Each item has the following properties:<br> `Name`, `Hash`, `ContentType`, `Size`. |
| `<doc>.addAttachment([name,] attachmentRef)` | function | Add an attachment to a transformed document that will be sent to a target (`<doc>`).<br>For details specific to Raven ETL, refer to this [section](../../../server/ongoing-tasks/etl/raven#attachments). |

{NOTE/}

{NOTE: }

#### The&nbsp;`loadTo`&nbsp;method

{INFO: }
An object will only be sent to the destination if the `loadTo` method is called.
{INFO /}

To specify which target to load the data into, use either of the following overloads in your script.  
The two methods are equivalent, offering alternative syntax.

* **`loadTo<TargetName>(obj, {attributes})`**  
  * Here the target is specified as part of the function name.  
  * The _&lt;TargetName&gt;_ in this syntax is Not a variable and cannot be used as one,  
    it is simply a string literal of the target's name.

* **`loadTo('TargetName', obj, {attributes})`**  
  * Here the target is passed as an argument to the method.  
  * Separating the target name from the `loadTo` function name makes it possible to include symbols like `'-'` and `'.'` in target names. 
    This is not possible when the `loadTo<TargetName>` syntax is used because including special characters in the name of a JavaScript function makes it invalid.
  * This syntax may vary for some ETL types.  
    Find the accurate syntax for each ETL type in the type's specific documentation.

--- 

For each ETL type, the target must be:   

  * RavenDB ETL: a _collection_ name
  * SQL ETL: a _table_ name
  * OLAP ETL: a _folder_ name
  * Elasticsearch ETL: an _index_ name
  * Kafka ETL: a _topic_ name
  * RabbitMQ ETL: an _exchange_ name
  * Azure Queue Storage ETL: a _queue_ name

{NOTE/}

{INFO: }

#### Batch processing

Documents are extracted and transformed by the ETL process in a batch manner.  
The number of documents processed depends on the following configuration limits:

* [`ETL.ExtractAndTransformTimeoutInSec`](../../../server/configuration/etl-configuration#etl.extractandtransformtimeoutinsec) (default: 30 sec)  
  Time-frame for the extraction and transformation stages (in seconds), after which the loading stage will start.

* [`ETL.MaxNumberOfExtractedDocuments`](../../../server/configuration/etl-configuration#etl.maxnumberofextracteddocuments) (default: 8192)  
  Maximum number of extracted documents in an ETL batch.

* [`ETL.MaxNumberOfExtractedItems`](../../../server/configuration/etl-configuration#etl.maxnumberofextracteditems) (default: 8192)  
  Maximum number of extracted items (documents, counters) in an ETL batch.

* [`ETL.MaxBatchSizeInMb`](../../../server/configuration/etl-configuration#etl.maxbatchsizeinmb) (default: 64 MB)  
  Maximum size of an ETL batch in MB.

{INFO/}

---

### Load

* Loading the results to the target destination is the last stage.

* In contrast to [Replication](../../../server/clustering/replication/replication), 
  ETL is a push-only process that _writes_ data to the destination whenever documents from the relevant collections are changed. **Existing entries on the target will always be overwritten**.

* Updates are implemented by executing consecutive DELETEs and INSERTs.  
  When a document is modified, the delete command is sent before the new data is inserted and both are processed under the same transaction on the destination side. 
  This applies to all ETL types with two exceptions:
    * In RavenDB ETL, when documents are loaded to **the same** collection there is no need to send DELETE because the document on the other side has the same identifier and will just update it.
    * in SQL ETL you can configure to use inserts only, which is a viable option for append-only systems.

{NOTE: }

**Securing ETL Processes for Encrypted Databases**:  

If your RavenDB database is encrypted, then you must not send data in an ETL process using a non-encrypted channel by default.
It means that the connection to the target must be secured:

- In RavenDB ETL, a URL of a destination server has to use HTTPS  
  (a server certificate of the source server needs to be registered as a client certificate on the destination server).
- in SQL ETL, a connection string to an SQL database must specify encrypted connection (specific per SQL engine provided).

This validation can be turned off by selecting the _Allow ETL on a non-encrypted communication channel_ option in the Studio, 
or setting `AllowEtlOnNonEncryptedChannel` if the task is defined using the Client API.  
Please note that in such cases, your data encrypted at rest _won't_ be protected in transit.

{NOTE/}

{PANEL/}

{PANEL: Troubleshooting}

ETL errors and warnings are [logged to files](../../../server/troubleshooting/logging) and displayed in the notification center panel.  
You will be notified if any of the following events happen:

- Connection error to the target
- JS script is invalid
- Transformation error
- Load error
- Slow SQL was detected


**Fallback Mode**:  
If the ETL cannot proceed the load stage (e.g. it can't connect to the destination) then it enters the fallback mode.  
The fallback mode means suspending the process and retrying it periodically.  
The fallback time starts from 5 seconds and it's doubled on every consecutive error according to the time passed since the last error,
but it never crosses [`ETL.MaxFallbackTimeInSec`](../../../server/configuration/etl-configuration#etl.maxfallbacktimeinsec) configuration (default: 900 sec).  

Once the process is in the fallback mode, then the _Reconnect_ state is shown in the Studio.

{PANEL/}

## Related Articles

### ETL
- [RavenDB ETL Task](../../../server/ongoing-tasks/etl/raven)
- [SQL ETL Task](../../../server/ongoing-tasks/etl/sql)
- [OLAP ETL Task](../../../server/ongoing-tasks/etl/olap)
- [Elasticsearch ETL Task](../../../server/ongoing-tasks/etl/elasticsearch)
- [Kafka ETL Task](../../../server/ongoing-tasks/etl/queue-etl/kafka)
- [RabbitMQ ETL Task](../../../server/ongoing-tasks/etl/queue-etl/rabbit-mq)
- [Azure Queue Storage ETL Task](../../../server/ongoing-tasks/etl/queue-etl/azure-queue)

### Studio
- [Define RavenDB ETL Task in Studio](../../../studio/database/tasks/ongoing-tasks/ravendb-etl-task)  

### Sharding
- [Sharding Overview](../../../sharding/overview)  
- [Sharding: ETL](../../../sharding/etl)  
