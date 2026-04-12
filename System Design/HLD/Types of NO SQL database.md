NoSQL is actually an umbrella term that covers several entirely different types of database architectures. In System Design, lumping them all together is a common trap. You choose a specific NoSQL database based entirely on your **access patterns**—how your application writes and reads data.

Here are the four primary categories of NoSQL databases, how they work, and when to use them.

## 1. Key-Value Stores

This is the simplest type of NoSQL database. It functions like a massive, highly distributed hash map or dictionary. Every piece of data (the value) is stored and retrieved using a unique identifier (the key). The database doesn't care what is inside the "value" (it could be a string, JSON, or a blob); it only cares about matching it to the key.

- **Popular Examples:** Redis, Amazon DynamoDB, Memcached.

- **When to choose it:**
	- **Caching:** Storing frequently accessed data to reduce load on your primary database (e.g., Redis).
    - **Session Management:** Storing user session states for web applications.
    - **Shopping Carts:** Temporarily holding user data before a transaction is finalized.
    - **Leaderboards & Counters:** When you need ultra-low latency (sub-millisecond) reads and writes by a known primary key.

## 2. Document Databases

Document databases extend the key-value concept. Instead of an opaque "value," the data is stored in a structured, semi-structured, or nested format—typically JSON, BSON, or XML. The database _understands_ this structure, allowing you to query against specific fields inside the document, rather than just fetching the whole thing by a single key.

- **Popular Examples:** MongoDB, Couchbase, Google Cloud Firestore.
    
- **When to choose it:**
    - **Rapid Prototyping & Agile Development:** When your schema is constantly evolving and you don't want to run expensive database migrations every time you add a feature.
    - **Content Management Systems (CMS):** Storing articles, blogs, or user-generated content where each item might have different attributes.
    - **Product Catalogs:** Where an electronics product might have completely different specifications (voltage, screen size) than a clothing product (size, fabric), but both need to be stored as "Products.
### Advantage of Document database over key value DB

The fundamental difference comes down to one concept: **Opaque vs. Transparent data.**
### 1. The "Opaque" Key-Value Store
To a Key-Value database, the value is **opaque**. It does not know, or care, what is inside the string you saved. It just sees a blob of bytes.
Imagine you store this JSON string under the key `user:101`:

JSON

```
{
  "name": "Sagnik Jana",
  "age": 24,
  "city": "Kolkata",
  "motorcycle": "TVS Raider 125",
  "skills": ["Python", "FastAPI", "React.js"]
}
```

**The Problem:** What if you want to find all users who live in "Kolkata" or all users who know "FastAPI"?
In a Key-Value store, **you cannot do this natively.** The database cannot look _inside_ the string. To answer that query, your application would have to fetch every single key in the database, parse the JSON in your application's memory, and filter the results manually. This is a massive performance bottleneck.

### 2. The "Transparent" Document Database

To a Document database (like MongoDB or Couchbase), the data is **transparent**. The database engine actually parses and understands the JSON (or BSON) tree structure as it saves it.
Because it understands the internal structure, it provides three massive advantages:
#### Advantage A: Deep Querying and Filtering

You don't have to fetch the whole document just to check a field. You can ask the database engine to do the heavy lifting.

You can run a query directly on the database level:
`db.users.find({ "city": "Kolkata", "skills": "FastAPI" })`
The database will only return the documents that match, saving enormous amounts of network bandwidth and application memory.

#### Advantage B: Secondary Indexes
This is the true superpower of Document DBs. Because the DB understands the JSON fields, you can tell it to build an index on specific nested data.
If you frequently search for users by the motorcycle they ride, you can create a secondary index on the `"motorcycle"` field. The Document DB will maintain a highly optimized lookup table for that specific JSON key, making those queries nearly instant, even across millions of records. A Key-Value store only ever indexes the primary Key.

#### Advantage C: Partial Document Updates

Imagine you have a birthday and need to update your age to 25.

- **In a Key-Value Store:** You have to read the _entire_ JSON string into your app, change the age to 25, serialize it back to a string, and overwrite the _entire_ value in the database. If two servers try to do this at the same time, you get race conditions and data loss.
- **In a Document DB:** You simply send a command: `db.users.updateOne({ "name": "Sagnik Jana" }, { $set: { "age": 25 } })`. The database goes precisely to that field inside the JSON and changes just that number. It is faster, atomic, and uses less bandwidth.

## Summary

Store JSON in a **Key-Value database** when you only ever need to fetch the _entire_ object using its exact ID (like loading a user's session cache).

Store JSON in a **Document database** when you need to search, filter, index, or update the specific data _inside_ that JSON structure.
## 3. Wide-Column (Column-Family) Stores

Unlike relational databases that store data in rows, wide-column stores organize data into flexible columns. Each row can have a entirely different set of columns. They are designed to distribute massive amounts of data across multiple servers (shards) without a single point of failure, making them incredibly resilient and fast for write-heavy workloads.

- **Popular Examples:** Apache Cassandra, ScyllaDB, Apache HBase.
- **When to choose it:**
    - **Time-Series Data:** Storing massive amounts of sequential data like IoT sensor readings, stock market ticks, or server metrics.
    - **Event Logging:** Ingesting millions of application logs or user activity events per second.
    - **High Availability at Massive Scale:** When you need an "active-active" setup across multiple geographical data centers and cannot afford any downtime.

### Why it is called "Wide column":
The name **"wide-column"** comes from the fact that a single row in this database can dynamically hold a massive, theoretically limitless number of columns, making the rows horizontally very "wide."

###  Apache Cassandra strongly enforces a schema So it has fixed number of columns, then Why it is called "Wide column":
The main reason Apache Cassandra is classified as a wide-column database boils down to one specific architectural feature: **how it physically groups data on the hard drive using Partitions.** While the modern query language (CQL) makes Cassandra look like a standard relational database with rigid rows and columns, the storage engine underneath ignores that illusion.

Here is the core mechanism that makes it "wide."

#### The Magic of the Partition Key

In Cassandra, when you design a table, your Primary Key is divided into two parts:

1. **The Partition Key:** This tells Cassandra _which physical server_ the data should live on.
    
2. **The Clustering Key:** This tells Cassandra how to sort the data _inside_ that server.
    

The defining characteristic of a wide-column store is that **all data sharing the exact same Partition Key is stored as a single, contiguous, massive row on the disk.**

#### How a Row Becomes "Wide"

Let's imagine a messaging app where you are storing chat messages.

**Your Logical View (How you see it in CQL):**

You create a table where `chat_room_id` is the Partition Key, and `message_timestamp` is the Clustering Key.

To you, it looks like this table has thousands of rows:

|**chat_room_id**|**message_timestamp**|**sender**|**message**|
|---|---|---|---|
|Room_Alpha|10:00:01|Alice|"Hi"|
|Room_Alpha|10:00:05|Bob|"Hello"|
|Room_Alpha|10:01:00|Alice|"How are you?"|

**Cassandra's Physical View (How the hard drive sees it):**

Cassandra takes everything with the partition key `Room_Alpha` and flattens it into one single row. It uses the Clustering Key (`message_timestamp`) to dynamically create new columns as data flows in.

**Row Key: `Room_Alpha`**

- **Column 1:** `[10:00:01_sender]: Alice`
    
- **Column 2:** `[10:00:01_message]: "Hi"`
    
- **Column 3:** `[10:00:05_sender]: Bob`
    
- **Column 4:** `[10:00:05_message]: "Hello"`
    
- **Column 5:** `[10:01:00_sender]: Alice`
    
- **Column 6:** `[10:01:00_message]: "How are you?"`
    

If a million messages are sent in `Room_Alpha`, you do not have a million rows. You have **one single row with two million columns.** That is why it is "wide."

#### Why did they build it this way?

This specific physical layout is the secret to Cassandra's legendary performance:

1. **Blazing Fast Writes:** Because data is appended to the end of a wide row as new columns, the hard drive's disk head doesn't have to seek around to find an empty slot. It just writes sequentially, which is incredibly fast.
    
2. **Lightning Fast Slices:** If you want to read all the messages in `Room_Alpha` from the last hour, Cassandra goes to exactly one place on the hard drive, reads that single row, grabs the specific chunk of columns you asked for, and returns it.

### Difference between Wide column and DocumentDB
To understand the difference between Document Databases and Wide-Column Stores, it helps to look at how they structure data, how they are queried, and what they are built to achieve. Both are NoSQL databases, meaning they avoid rigid relational schemas, but they solve entirely different engineering problems.

Here is a breakdown of the core differences.
#### 1. The Data Model

The most fundamental difference is how they visualize and store a single record.

- **Document Databases (e.g., MongoDB, Couchbase):** Think of these as **Hierarchical Trees**. Data is stored natively as JSON or BSON documents, allowing for deep, complex nesting—arrays within objects within arrays. While you can technically store nested JSON in a wide-column store, Document databases have a massive advantage here: **the database engine actually understands the tree.** Because it parses the BSON natively, you can easily create secondary indexes on a specific field buried five levels deep, or update a single leaf node in an array without having to fetch, parse, and rewrite the entire document.

- **Wide-Column Stores (e.g., Cassandra, HBase):** Think of these as a **Flat, Two-Dimensional Map**. Data is stored in rows, and each row contains dynamic columns. While you can store collections (like a list or a map) inside a column, deep nesting is generally an anti-pattern. The structure is meant to be relatively flat but incredibly wide.
#### 2. Query Flexibility vs. Predictability

How you ask the database for information differs wildly between the two.

- **Document DBs offer flexible querying (with caveats at scale):** You can typically query a Document DB by almost any field within the JSON document. If you have a user profile, you can easily search for "all users living in London who are over 30." In a standard, unsharded deployment, the database uses a central secondary index to find this data instantly, making it very flexible for general-purpose applications. **However, this flexibility comes with a performance penalty when the database is horizontally scaled.** If you shard your MongoDB database (e.g., by `user_id`) and run that exact same query without including the `user_id`, the database query router is forced to perform a "scatter-gather" operation. It must broadcast your query across the network to _every single shard_, wait for all of them to check their local secondary indexes, and then merge the results. While MongoDB can typically survive this across a small number of large shards, querying without the shard key creates a severe network bottleneck at massive scale.

- **Wide-Column DBs require strict access patterns:** You generally cannot query a wide-column store by random fields. You must query by the exact Partition Key (and optionally the Clustering Key). If your partition key is `User_ID`, you cannot ask the database "find all users in London." **While wide-column databases like Cassandra technically support secondary indexes for non-primary columns, using them for arbitrary queries is considered a dangerous anti-pattern.** Because Cassandra is designed to distribute data across hundreds or thousands of nodes, a secondary index query triggers a massive, cluster-wide scatter-gather operation that will spike network latency and can easily bring the system down. Instead, you must design your database specifically around the exact queries you intend to run. Because of this, it is highly recommended (and standard practice) to use **Query-Driven Modeling**—creating a separate, duplicated table for each distinct query pattern. For example, if you need to look up users by ID and also by City, you would maintain two entirely separate tables (e.g., `users_by_id` and `users_by_city`) and write the data to both simultaneously.
#### Side-by-Side Comparison

| **Feature**            | **Document Database (e.g., MongoDB)**                                     | **Wide-Column Store (e.g., Cassandra)**                                                 |
| ---------------------- | ------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| **Data Format**        | JSON / BSON Documents                                                     | Rows with dynamic, sparse columns                                                       |
| **Structure**          | Deeply nested, hierarchical                                               | Flat, tabular, but horizontally expansive                                               |
| **Query Flexibility**  | High. Can query on almost any nested field using indexes.                 | Low. Must query strictly by Primary/Partition Key.                                      |
| **Read/Write Focus**   | Balanced. Optimized for fetching whole, complex objects quickly.          | Write-heavy. Optimized for millions of writes per second with zero downtime.            |
| **Schema enforcement** | Usually completely schema-less (though validation can be added).          | Modern wide-column (CQL) enforces a table schema, though columns remain sparse on disk. |
| **Ideal Use Cases**    | Content management, user profiles, product catalogs, mobile app backends. | IoT telemetry, real-time logging, time-series data, massive-scale messaging apps.       |

#### The Bottom Line

Choose a **Document Database** if your data is complex, hierarchical, and you need the flexibility to query it in various ways (like a standard web or mobile application).

Choose a **Wide-Column Store** if your application needs to ingest a firehose of data (millions of events per second), requires 100% uptime across multiple geographic data centers, and you know your exact query patterns ahead of time.

## 4. Graph Databases

Graph databases are specifically built to handle complex relationships. Instead of tables or documents, data is stored as **Nodes** (entities like people, places, or objects) and **Edges** (the relationships connecting them). They allow you to traverse deep, complex webs of connections in milliseconds—a task that would require agonizingly slow, multi-level `JOIN` operations in a standard SQL database.

- **Popular Examples:** Neo4j, Amazon Neptune, ArangoDB.
- **When to choose it:**
    - **Social Networks:** Querying "friends of friends who also like this specific movie."
    - **Recommendation Engines:** Connecting users to products based on complex purchase histories and shared affinities.
    - **Fraud Detection:** Tracking complex money flows between disparate bank accounts to identify synthetic identity rings.
    - **Knowledge Graphs & Mapping:** Routing algorithms or mapping out IT network infrastructure.

## Summary Rule of Thumb

- Need sub-millisecond lookups by an ID? **Key-Value**.
- Need flexible schemas for dynamic objects? **Document**.
- Need to write millions of logs/metrics per second? **Wide-Column**.
- Need to analyze complex connections and networks? **Graph**.

# Note for dynamoDB
- its not strictly a key value store. its a **Document / Key-Value hybrid** .

DynamoDB natively supports JSON (it calls them `Map` and `List` data types). You actually _can_ filter query results based on nested JSON fields using something called `FilterExpressions`.

**The catch:** You cannot build a Secondary Index (GSI) on a deeply nested JSON field. So, while DynamoDB can look inside the JSON to filter results before sending them to your application, it still has to read the whole item first. If you want blazing-fast indexed searches on a specific field, you have to pull that field out to the top level of the table.

# What is partition key sort key and primary key in DynamoDB

The easiest way to understand them is to realize that a **Primary Key** isn't a third, separate thing. The Primary Key is the overarching term for _how you uniquely identify a row_.

Here is how they all fit together.

## 1. The Primary Key (The Unique Identifier)

In DynamoDB, every single item (row) must have a **Primary Key** to uniquely identify it. No two items can have the exact same Primary Key.

DynamoDB gives you two ways to build this Primary Key:

- **Simple Primary Key:** Consists of _only_ a Partition Key.
- **Composite Primary Key:** Consists of a Partition Key _plus_ a Sort Key.

## 2. The Partition Key (The "Where")

Also known as the Hash Key. This is the first (and sometimes only) part of your Primary Key.

When you write data to DynamoDB, it takes the value of your Partition Key, runs it through an internal hash function, and uses the output to determine exactly which physical server (partition) will store your data.
- **Its Job:** To distribute your data evenly across Amazon's massive fleet of servers.
- **The Rule:** If you only use a Partition Key (Simple Primary Key), every Partition Key value must be 100% unique.
- **Example:** In a `Users` table, the Partition Key would be `UserId`. You provide the `UserId`, and DynamoDB instantly knows exactly which server holds that user's data (O(1) lookup speed).

## 3. The Sort Key (The "How it's Organized")

Also known as the Range Key. This is the optional second half of a Composite Primary Key.

If you use a Sort Key, multiple items _can_ have the exact same Partition Key, as long as their Sort Keys are different.

- **Its Job:** It takes all the items that share the same Partition Key (meaning they live on the exact same physical server) and physically sorts them on the hard drive based on the Sort Key's value.
- **The Superpower:** It allows you to perform highly efficient "Range Queries". You can ask the database for items where the Sort Key is `>`, `<`, `BETWEEN`, or `begins_with` a certain value.
- **Example:** In a chat history table, The Partition Key is `SessionId` and the Sort Key is `Timestamp`. All messages for one session go to the same server, and they are neatly sorted by time so you can quickly grab "the last 50 messages".
