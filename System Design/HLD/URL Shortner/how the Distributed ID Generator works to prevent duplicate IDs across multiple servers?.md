Generating unique IDs in a distributed system is one of the most critical challenges in system design. If you have ten different application servers handling requests simultaneously, relying on a standard database `AUTO_INCREMENT` becomes a massive bottleneck and a single point of failure.

Here are the three most common strategies used in the industry to solve this, ranging from simple to highly scalable.

## 1. Multi-Master Database with Offsets (The Simple Approach)

Instead of a single database, you run multiple database servers. To prevent them from generating the same ID, you configure each server to use a specific starting offset and increment by the total number of database servers.

- **Server A:** Generates 1, 4, 7, 10...
    
- **Server B:** Generates 2, 5, 8, 11...
    
- **Server C:** Generates 3, 6, 9, 12...
    

**Pros:** Relatively easy to set up with standard relational databases.

**Cons:** It's very rigid. If you need to add or remove a database server to handle more traffic, reconfiguring the offsets across the cluster without causing collisions is incredibly difficult.

## 2. Twitter Snowflake (The Industry Standard)

Twitter created Snowflake to generate unique IDs for tweets at a massive scale without needing nodes to coordinate with each other. It generates a 64-bit integer that is mostly time-based, making the IDs naturally sortable by creation time.

Here is how the 64 bits are typically divided:

- **Sign Bit (1 bit):** Always 0 (keeps the number positive).
    
- **Timestamp (41 bits):** Milliseconds since a custom epoch (e.g., the moment your system goes live). 41 bits gives you about 69 years of IDs before rolling over.
    
- **Datacenter/Machine ID (10 bits):** Identifies the specific machine generating the ID. This allows for up to 1,024 unique machines.
    
- **Sequence Number (12 bits):** A local counter on that specific machine. It starts at 0 every millisecond and increments for every ID generated. It allows a single machine to generate up to 4,096 IDs per millisecond.
    

**Pros:** Highly scalable, zero network coordination required during generation, and highly available.

**Cons:** It requires managing machine IDs (often done using a configuration tool or coordination service) and is vulnerable to clock synchronization issues (NTP drift) on the servers.

## 3. Range-Based Generation using a Coordination Service (The High-Performance Approach)

In modern cloud-native architectures—especially those utilizing orchestration tools like Kubernetes—you often have access to a distributed consensus store like **ZooKeeper** or **etcd**. You can use this to manage ID generation brilliantly.

Instead of the application servers asking a database for a new ID on _every_ request, they ask the coordination service for a **batch** or **range** of IDs.

1. **Server A** requests a range from ZooKeeper. It gets assigned `1` to `100,000`.
    
2. **Server B** requests a range. It gets assigned `100,001` to `200,000`.
    
3. Server A now generates IDs locally in its own memory until it hits `100,000`. This is blazingly fast because there is no network call required to generate the ID.
    
4. Once Server A exhausts its range, it goes back to ZooKeeper to claim the next available block (e.g., `200,001` to `300,000`).
    

**Pros:** Incredibly fast (in-memory generation), and resilient. Even if ZooKeeper goes down temporarily, the application servers can continue generating short URLs until their current range runs out.

**Cons:** If a server crashes before using its entire block, those remaining IDs are lost forever. For a URL shortener, skipping some IDs is perfectly fine, making this an excellent choice.

---

For a URL shortener, the **Range-Based Generation** or **Snowflake** are usually the best choices.

Would you like to explore how we would design the caching layer to handle viral links that suddenly get millions of clicks per minute?