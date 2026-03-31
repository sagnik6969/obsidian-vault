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

For a URL shortener, the **Range-Based Generation** or **Snowflake** are usually the best choices.

Would you like to explore how we would design the caching layer to handle viral links that suddenly get millions of clicks per minute?


# Distributed Id generation Using zoo keeper step by step

To understand exactly how the Distributed ID Generator fits into the larger architecture, let's zoom in on the **Range-Based (ZooKeeper/etcd)** approach. This is often the most complex to visualize because it requires the application servers to manage their own local state.

Let's walk through the exact lifecycle of a system, from the moment a server boots up to the moment it successfully stores a new short URL.

## 1. Server Initialization (The Boot-Up Phase)

Imagine you are spinning up a new web server running a fast, asynchronous framework like FastAPI.

Before this server is allowed to accept any incoming HTTP requests from the Load Balancer, it must initialize its local ID generator.

1. **The Request:** The FastAPI server reaches out to ZooKeeper over the network. It essentially asks, _"I am Server Node #5. Give me a fresh block of IDs."_
2. **The Allocation:** ZooKeeper checks its internal registry, sees that ranges 1 through 5,000,000 are already taken, and assigns the next block: `5,000,001` to `6,000,000`.
3. **Local Memory:** The FastAPI server stores these two numbers in its local application memory (RAM).
    - `current_id = 5,000,001`
    - `max_id = 6,000,000`

The server is now healthy and ready to accept traffic.

## 2. The Write Path (Processing a Request)

Now, a user submits a POST request to shorten `https://www.example.com/very-long-article-name`.

1. **API Endpoint Hit:** The request hits your `POST /api/v1/urls` endpoint.
2. **In-Memory ID Generation:** The application needs a unique integer. Instead of querying a database, it simply looks at its local RAM.
    
    - It grabs `current_id` (`5,000,001`).
    - It immediately increments `current_id` in memory to `5,000,002`.
    - _Note: This operation takes mere nanoseconds because it requires absolutely no network I/O._
        
3. **Base62 Conversion:** The server runs a local function to convert the integer `5,000,001` into a Base62 string. Let's say it becomes `g8Kx`.
4. **Database Write:** The server now has the Short URL (`g8Kx`) and the Long URL. It constructs a query to write this directly to your NoSQL database (e.g., DynamoDB or Cassandra), using `g8Kx` as the Partition Key.
5. **Response:** Once the database confirms the write, the server returns the `201 Created` response to the client.

Here is a conceptual look at how that state is managed within the application code:

Python

```
class LocalIDManager:
    def __init__(self):
        self.current_id = 0
        self.max_id = 0
        self.lock = asyncio.Lock() # To prevent race conditions if using multiple threads/workers

    async def get_next_id(self):
        async with self.lock:
            # If we run out of IDs in our current block, fetch more
            if self.current_id >= self.max_id:
                await self._fetch_new_range_from_zookeeper()
            
            # Grab the ID and increment
            allocated_id = self.current_id
            self.current_id += 1
            
            return allocated_id

    async def _fetch_new_range_from_zookeeper(self):
        # Network call to ZooKeeper to get a new block of 1,000,000 IDs
        new_range = await zookeeper_client.allocate_range("url_shortener")
        self.current_id = new_range.start
        self.max_id = new_range.end
```

## 3. Handling Edge Cases

Integrating stateful logic into stateless web servers introduces a few architectural challenges that you have to design around:

- **What happens when the server runs out of IDs?** When `current_id` hits `6,000,000`, the `if` condition in the code above triggers. The server pauses processing that specific request for a few milliseconds, makes a synchronous network call to ZooKeeper for the next block (e.g., `12,000,001` to `13,000,000`), updates its local variables, and then resumes.
    
- **What happens if the FastAPI server crashes or restarts?** If the server crashes while `current_id` is at `5,500,000`, the remaining 500,000 IDs in that block are lost forever. When the server restarts, it will ask ZooKeeper for a _brand new_ block. Because we don't care if short URLs are perfectly sequential (we only care that they are unique), losing IDs is a perfectly acceptable trade-off for the massive performance gains this architecture provides.
    
- **What if two servers ask ZooKeeper for a range at the exact same time?** ZooKeeper ensures atomic operations. It will queue the requests and guarantee that Server A gets block #5 and Server B gets block #6, preventing any overlapping ranges.
    

---

By decoupling the ID generation entirely from the database writes, your application servers can operate at blazing speeds, and your NoSQL database only has to worry about what it does best: storing and retrieving key-value pairs at scale.



# Problem with above approach: If we want to implement auto scaling too much ids will be lost when number of servers are scaled down on low traffic?

## 1. Does Wasting IDs Actually Matter?

In most large-scale systems, the short answer is **no**.

If you are using a 7-character Base62 string, your total namespace is $62^7$, which gives you roughly **3.5 trillion** unique combinations.

Even if you auto-scale aggressively and waste 500,000 IDs every time a server spins down, you would have to destroy millions of servers before making a dent in a 3.5 trillion ID pool. Storage is cheap, and integer space is vast. Wasting IDs is generally considered an acceptable tax for gaining blazing-fast, lock-free performance on your write path.

However, if your product requirement dictates keeping the URLs as short as possible for as long as possible (e.g., exactly 5 characters, which only yields ~916 million combinations), wasting large blocks becomes a massive problem.

## 2. How to Mitigate ID Loss in an Auto-Scaled Environment

If you want to minimize the bleeding of IDs without sacrificing the decoupled architecture, you have two primary strategies.

#### Strategy A: Tune the Block Size (The Pragmatic Approach)

You don't have to allocate a million IDs at once. You tune the block size to match your expected throughput and pod lifecycle.

If a single server generates roughly 100 URLs per second, allocating a block of **5,000 IDs** means the server only hits ZooKeeper/etcd once every 50 seconds.

- **Pros:** When the server scales down, you lose a maximum of 4,999 IDs—a negligible amount.
    
- **Cons:** It slightly increases the read load on your coordination service, but tools like ZooKeeper are explicitly built to handle tens of thousands of requests per second, so fetching a block every minute is barely a blip.
    

#### Strategy B: Graceful Shutdowns (The Engineering Approach)

When your infrastructure (like Kubernetes) decides to scale down a server, it doesn't just instantly kill the process. It sends a `SIGTERM` signal, giving the application a brief window to clean up before the forceful `SIGKILL`.

You can use application-level lifecycle events to catch this signal and write the remaining unused range back to ZooKeeper so the next server can claim it.

Here is how that conceptually looks using modern async Python frameworks like FastAPI:

Python

```
from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    # --- STARTUP ---
    # Fetch initial block from ZooKeeper when the pod spins up
    await id_manager.fetch_new_range()
    
    yield # Application runs and serves requests here
    
    # --- SHUTDOWN ---
    # Kubernetes sent SIGTERM. Pod is scaling down.
    # Return the unused IDs back to the ZooKeeper pool.
    unused_start = id_manager.current_id
    unused_end = id_manager.max_id
    await zookeeper_client.return_range(unused_start, unused_end)

app = FastAPI(lifespan=lifespan)
```

- **Pros:** Zero wasted IDs. Maximum efficiency of your namespace.
    
- **Cons:** It introduces complexity. If the server crashes abruptly (OOM kill, hardware failure) instead of shutting down gracefully, those IDs are still lost.
    

## The Verdict

For a URL shortener, the standard industry practice is to simply **tune the block size down** to a reasonable number (e.g., 1,000 to 10,000) and accept the minor losses during scale-down events. It keeps the system highly available and the codebase significantly simpler.


