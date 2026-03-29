## 1. Single server: 
DB and Backend are in same server

## 2. DB server seperation
- Backend Application and DB will be in different server

## 3. Load balancer and multiple app server
- Add more than 1 application server and add a load balancer on top of it.

## 4. Database Replication
- 1 muster db (used for write operations)
- Multiple Slave DB (Used for read)
- If master db fails then 1 of the slave db s become master. 

## 5. Cacheing
- self explanatory

## 6. CDN (Content delivery Network)
- CDN also does cacheing. But all those who does cacheing is not CDN.
- Used to cache static contents at the edge locations (close to users)

## 7. Data Centres
- Add multiple data centre at multiple geo locations so that if one data centre goes down the application still stays up.

## 8. Message Queues
- Make tasks asynchronous. 
- Example: To send notifications


## 9. Database scaling
#### 1. Vertical:
- increase cpu & ram
#### 2. Horizontal
- add new db nodes. 
- Known as sharding
#### Types of sharding
##### 1. horizontal
- keep some rows in node 1 some in node 2 and so on.
- each node is known as shard. 
- There will be a logic by which we will determine data should be in which shard.
- For example lets say if name starts with a-m then go to shard 1 else share 2.
##### 2. vertical
- Keep some columns in shard 1 some in shard 2 and so on.
- 






