
Designing a scalable chat application like WhatsApp, Discord, or Facebook Messenger is a classic system design challenge. It requires balancing low latency for real-time communication with massive storage capabilities for message history.

Here is a breakdown of the high-level architecture and the core components required to make it work.

### 1. Core Requirements

Before designing, we establish what the system must do:

- **Functional:** 1-on-1 messaging, group chats, online presence (status), and message history.
    
- **Non-Functional:** Low latency (real-time delivery), high availability, and scalability to handle millions of concurrent users.
    

### 2. The High-Level Architecture

To handle these requirements efficiently, the architecture is typically split into two main paths: **Stateless Services** (for traditional requests) and **Stateful Services** (for real-time messaging).

#### A. The Edge / Entry Point

- **Clients:** The mobile or web applications.
    
- **Load Balancer:** Distributes incoming traffic across multiple servers to prevent any single server from becoming a bottleneck.
    
- **API Gateway:** Acts as the single entry point for all API calls, handling authentication, rate limiting, and routing.
    

#### B. Stateless Services (Traditional API)

These servers handle requests that do not require a persistent connection.

- **User Service:** Manages user profiles, settings, and authentication.
    
- **Friend/Contact Service:** Manages connections and friend lists.
    
- **Database (Relational/SQL):** A database like PostgreSQL or MySQL is typically used here to guarantee ACID compliance for user data and relationships.
    

#### C. Stateful Services (Real-Time Messaging)

This is the heart of the chat app. Because HTTP is inefficient for real-time, bi-directional communication, we use **WebSockets**.

- **Chat Servers:** These maintain millions of open, persistent WebSocket connections with active clients. When a user opens the app, they connect to a specific Chat Server and stay connected.
    
- **Session/Service Discovery (e.g., ZooKeeper):** Since there are many Chat Servers, the system needs a way to know _which_ Chat Server User A is currently connected to.
    

#### D. Routing and Message Delivery

What happens when User A (on Server 1) sends a message to User B (on Server 2)?

- **Pub/Sub System (e.g., Redis Pub/Sub or Kafka):** The Chat Servers act as publishers and subscribers. Server 1 publishes the message to a channel for User B. Server 2, which is subscribed to User B's channel, receives the message and pushes it through its open WebSocket to User B.
    
- **Push Notification Service:** If User B is entirely disconnected (app is closed), the message is routed to Apple Push Notification Service (APNs) or Firebase Cloud Messaging (FCM) to wake up their phone.
    

#### E. Storage and Presence

- **Message Database (NoSQL):** Chat apps generate massive amounts of append-only data. A Key-Value or Wide-Column store like **Cassandra** or **DynamoDB** is ideal because it allows for rapid writes and efficient reads of message sequences.
    
- **Presence Servers:** A dedicated service that tracks who is online, offline, or typing. It broadcasts status changes to a user's active friends.
    

---

### Step-by-Step Data Flow: Sending a 1-on-1 Message

1. **User A** types a message and sends it via their active WebSocket connection to **Chat Server 1**.
    
2. **Chat Server 1** generates a unique ID for the message and immediately saves it to the **Message Database** (or places it in a message queue for asynchronous saving).
    
3. **Chat Server 1** queries the **Session Service** to find out where **User B** is connected.
    
4. If **User B** is online and connected to **Chat Server 2**, the message is sent to Server 2 via the **Pub/Sub system**.
    
5. **Chat Server 2** pushes the message down its WebSocket connection to **User B**.
    
6. If **User B** is offline, the message is routed to the **Push Notification Service**.
    

To help you visualize how these components interact and route data, I have generated an interactive architecture explorer below. You can click on the different nodes to explore their responsibilities and simulate the flow of a message through the system.

Show me the visualisation