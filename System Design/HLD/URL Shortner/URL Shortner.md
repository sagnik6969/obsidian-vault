Designing a URL shortener like Bitly or TinyURL is a classic system design exercise. It seems simple on the surface, but scaling it to handle millions of requests requires some strategic choices.

Here is a comprehensive High-Level Design (HLD) to help you understand the core architecture and trade-offs.

## **1. Requirements Clarification**

Before jumping into the architecture, we need to define what the system actually does.

**Functional Requirements:**
- **Shortening:** Given a long URL, the system generates a unique, short alias (e.g., `short.ly/xyz123`).
- **Redirection:** When a user clicks the short link, they are redirected to the original long URL.
- **Custom URLs (Optional but common):** Users can pick their own custom short alias.

**Non-Functional Requirements:**
- **High Availability:** The redirection service must have near 100% uptime. If it goes down, every link breaks.
- **Low Latency:** Redirection should happen in milliseconds.
- **Scale:** The system will be heavily read-biased (the industry standard assumption is a 10:1 to 100:1 read-to-write ratio).

## **2. Back-of-the-Envelope Estimation**

Let's assume the system generates **100 million new URLs per month** and has a **10:1 read/write ratio**.

- **Writes:** 100 million URLs / month ≈ ~40 writes per second.
- **Reads:** 1 billion redirects / month ≈ ~400 reads per second.
- **Storage:** If we keep links for 10 years, that’s 12 billion records. If one record (Long URL, Short URL, Created_At) is roughly 500 bytes, we need about **6 Terabytes** of storage.

_Note: These numbers are relatively low for modern databases, meaning storage isn't our biggest bottleneck—read throughput is._

---

## **3. API Design**

We need two primary REST API endpoints:

1. **Create Short URL:**
    
    `POST /api/v1/urls`
    
    - **Payload:** `{ "long_url": "https://www.example.com/very/long/path" }`
        
    - **Response:** `201 Created` with payload `{ "short_url": "https://short.ly/ab38Xy" }`
        
2. **Redirect:**
    
    `GET /{short_url_alias}`
    
    - **Response:** HTTP `301` or `302` redirect to the long URL.
        
        - _Design Note:_ **301 (Permanent Redirect)** means the browser caches the result, reducing load on your servers. **302 (Temporary Redirect)** forces the browser to hit your server every time, which is necessary if you want to track analytics (click rates).
            

## **4. Core Logic: The Shortening Algorithm**

How do we generate a unique 6-to-8 character string? We generally use **Base62 Encoding** (A-Z, a-z, 0-9 = 62 characters). A 7-character Base62 string gives us 62^7 (~3.5 trillion) possible URLs, which is more than enough.

**The preferred approach: Unique ID Generator + Base62**

Instead of hashing the long URL (which can cause collisions), we assign a globally unique integer ID to every new request, and then convert that base-10 integer into base-62.

- **Step 1:** Request comes in.
    
- **Step 2:** A distributed ID generator (like Twitter Snowflake or a standalone database counter) gives this request a unique ID (e.g., `2009215674938`).
    
- **Step 3:** Convert `2009215674938` to Base62 -> `zn9edcu`.
    
- **Step 4:** Store the mapping: `zn9edcu` -> `https://longurl...`
    

## **5. High-Level Architecture**

Here is how the request flows through the system:

**For Creating a URL (Write Path):**

1. The client sends a POST request to the API.
    
2. The Load Balancer routes the request to an available Web Server.
    
3. The Web Server talks to the **Unique ID Generator** to get a new integer.
    
4. The server converts the integer to a Base62 short string.
    
5. The server saves the mapping (Short String <-> Long URL) in the Database.
    

**For Redirection (Read Path):**

1. The client clicks the short link (e.g., `short.ly/zn9edcu`).
    
2. The Load Balancer routes the GET request to a Web Server.
    
3. The Web Server checks the **Cache** (e.g., Redis).
    
    - If the mapping is in the cache (Cache Hit), it instantly returns the Long URL.
        
    - If not (Cache Miss), it queries the Database, updates the cache, and returns the Long URL.
        
4. The server sends an HTTP 302 response to the client with the Long URL.
    

---

## **6. Database & Scaling Strategies**

- **The Database:** Because the data is highly relational (just key-value pairs of short-to-long URLs) and we need massive read/write scalability, a **NoSQL Database** (like DynamoDB or Cassandra) is usually the best choice. It scales horizontally much easier than a traditional SQL database.
    
- **The Cache:** The Pareto Principle (80/20 rule) applies here—80% of clicks will go to 20% of the links (viral tweets, trending news). A caching layer like **Redis** or **Memcached** is critical. It stores the most frequently accessed URLs in memory, drastically reducing database load and speeding up redirect times.
    
- **Rate Limiting:** To prevent malicious users from spamming the system and exhausting the ID pool, you would implement an API Gateway or rate limiter based on IP addresses or API keys.
