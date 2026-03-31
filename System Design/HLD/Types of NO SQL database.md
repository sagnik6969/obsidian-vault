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

Unlike relational databases that store data in rows, wide-column stores organize data into flexible columns. Each row can have a entirely different set of columns. They are designed to distribute massive amounts of data across multiple servers (partitions) without a single point of failure, making them incredibly resilient and fast for write-heavy workloads.

- **Popular Examples:** Apache Cassandra, ScyllaDB, Apache HBase.
- **When to choose it:**
    - **Time-Series Data:** Storing massive amounts of sequential data like IoT sensor readings, stock market ticks, or server metrics.
    - **Event Logging:** Ingesting millions of application logs or user activity events per second.
    - **High Availability at Massive Scale:** When you need an "active-active" setup across multiple geographical data centers and cannot afford any downtime.

### Difference between Wide column and DocumentDB

Here is why a Document Database (like MongoDB) and a Wide-Column Store (like Cassandra) are actually distinct tools for very different jobs.

#### 1. Data Shape: Deeply Nested vs. Incredibly Flat
- **Document Databases are for Trees:** They natively understand deep, complex, multi-level hierarchies. You can have an array of objects, inside a dictionary, inside another array. The database engine can parse this entire tree, build indexes on a field 4 levels deep, and update a single nested value.
- **Wide-Column Stores are for Flat Maps:** They do not understand nested JSON. A wide-column store is essentially a massive, two-dimensional Key-Value map. A single row can have millions of columns, but those columns are flat. If you put JSON into a Wide-Column store, it just treats it as a dumb string.
#### 2. The Architecture: Primary-Replica vs. Masterless Ring
This is the biggest architectural difference and the main reason you choose one over the other in system design.
- **Document DBs (Usually Primary-Replica):** In databases like MongoDB, you typically have one "Primary" node that handles all the _Writes_. The other nodes are "Replicas" that handle the _Reads_.
    - _The Bottleneck:_ If you try to write 5 million records per second, that single Primary node will melt down. It scales well for general applications, but has a hard ceiling for write-throughput.
- **Wide-Column Stores (Masterless Ring):** Databases like Cassandra use a Peer-to-Peer Ring architecture. There is no "Primary" node. **Every single server in the cluster can accept writes simultaneously.** * _The Superpower:_ If you need to handle an insane avalanche of data (like Netflix tracking every single pause, rewind, and click for 200 million users in real-time), a Wide-Column store absorbs those writes effortlessly by distributing them across hundreds of equal nodes.
#### 3. How Data is Stored on Disk
- **Document DBs store the whole object together:** When you fetch a document, the database grabs the entire BSON/JSON blob from the disk. This is highly optimized for "Give me everything about User 123."
- **Wide-Column DBs store columns together:** They physically group data on the hard drive by column families. If you have a table with 100 columns, but your query only asks for 2 specific columns (e.g., `cpu_usage` and `ram_usage` across a thousand servers), the database only reads those specific chunks of the hard drive. It ignores the rest of the row entirely, making massive analytical reads incredibly fast.
#### 4. Query Flexibility
- **Document DBs are flexible:** You can search, filter, and sort by almost any field if you build a secondary index for it. It feels very close to querying a traditional SQL database.
- **Wide-Column DBs are extremely rigid:** You _must_ query exactly how you designed the table (using the Partition Key and Clustering Key). If you try to search for a value in a random column that isn't part of your primary key, the database will likely reject the query or perform a catastrophic full-table scan. You design the database around the exact query you intend to run.

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
