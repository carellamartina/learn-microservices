# Distributed caching

Distributed caching has emerged as a pivotal solution for modern applications that demand high performance, scalability, and real-time data access. By storing frequently accessed data across multiple servers, distributed caching reduces the strain on primary data sources, ensuring rapid data retrieval and enhanced user experiences.



## Local vs Distributed caching

Caching can be broadly categorized into two types: local caching and distributed caching.

* **Local caching** refers to storing data on a single machine or within a single application. It's commonly used in scenarios where data retrieval is limited to one machine or where the volume of data is relatively small. **Local caching, while effective for single-machine applications, faces limitations in distributed systems**. As applications scale and serve users from various locations, relying solely on local caching can lead to data inconsistencies, increased latency, and potential bottlenecks. 

* **Distributed caching** involves storing data across multiple machines or nodes, often in a network. This type of caching is essential for applications that need to scale across multiple servers or are distributed geographically. **Distributed caching addresses the limitations of local caching by storing data across multiple machines or nodes in a network** allowing: **Scalability**, **Fault tolerance**, **Performance** (in exchange for increased complexity).

![](images/cache-architecture.avif)

## Key components of distributed caching

**Cache Servers**: Cache servers are the primary components in a distributed caching system. They store temporary data across multiple machines or nodes, ensuring that the data is available close to where it's needed. Each cache server can operate independently, and in case of a server failure, the system can reroute requests to another server, ensuring high availability and fault tolerance.

**Data Partitioning**: In distributed caching, data is partitioned across multiple cache servers to ensure efficient data distribution and retrieval. There are several strategies for data partitioning:

-   **Consistent hashing**: This method ensures that data is evenly distributed across cache servers and minimizes data movement when new servers are added or existing ones are removed.
-   **Virtual nodes**: Virtual nodes are used to handle scenarios where cache servers have varying capacities. They ensure that data distribution remains balanced even if some servers have higher storage capacities than others.

**Data Replication**: Replication is another crucial aspect of distributed caching. By replicating data across multiple cache servers, the system ensures data availability even if a server fails. Common replication strategies include master-slave replication, where one server acts as the master and others as replicas, and peer-to-peer replication, where each server acts both as a primary store and a replica for different data items.

## Distributed caching patterns

### Cache-Aside

This is perhaps the most commonly used caching approach, at least in the projects that I worked on. The cache sits on the _side_ and the application **directly** talks to both the cache and the database. There is no connection between the cache and the primary database. All operations to cache and the database are handled by the application. This is shown in the figure below.

![cache-aside](https://codeahoy.com/img/cache-aside.png)

Here’s what’s happening:

1.  The application first checks the cache.
2.  If the data is found in cache, we’ve _cache hit_. The data is read and returned to the client.
3.  If the data is **not found** in cache, we’ve _cache miss_. The application has to do some **extra work**. It queries the database to read the data, returns it to the client and **stores** the data in cache so the subsequent reads for the same data results in a cache hit.

#### Use Cases, Pros and Cons

Cache-aside caches are usually general purpose and work best for **read-heavy workloads**. _Memcached_ and _Redis_ are widely used. Systems using cache-aside are **resilient to cache failures**. If the cache cluster goes down, the system can still operate by going directly to the database. (Although, it doesn’t help much if cache goes down during peak load. Response times can become terrible and in worst case, the database can stop working.)

Another benefit is that the data model in cache can be different than the data model in database. E.g. the response generated as a result of multiple queries can be stored against some request id.

When cache-aside is used, the most common write strategy is to write data to the database directly. When this happens, cache may become inconsistent with the database. To deal with this, developers generally use time to live (TTL) and continue serving stale data until TTL expires. If data freshness must be guaranteed, developers either **[invalidate the cache entry](https://codeahoy.com/2022/04/03/cache-invalidation/)** or use an appropriate write strategy, as we’ll explore later.

### Read-Through Cache

Read-through cache sits in-line with the database. When there is a cache miss, it loads missing data from database, populates the cache and returns it to the application.

![read-through](https://codeahoy.com/img/read-through.png)

Both cache-aside and read-through strategies load data **lazily**, that is, only when it is first read.

#### Use Cases, Pros and Cons

While read-through and cache-aside are very similar, there are at least two key differences:

1.  In cache-aside, the application is responsible for fetching data from the database and populating the cache. In read-through, this logic is usually supported by the library or stand-alone cache provider.
2.  Unlike cache-aside, the data model in read-through cache cannot be different than that of the database.

Read-through caches work best for **read-heavy** workloads when the same data is requested many times. For example, a news story. The disadvantage is that when the data is requested the first time, it always results in cache miss and incurs the extra penalty of loading data to the cache. Developers deal with this by ‘_warming_’ or ‘pre-heating’ the cache by issuing queries manually. Just like cache-aside, it is also possible for data to become inconsistent between cache and the database, and solution lies in the write strategy, as we’ll see next.

### Write-Through Cache

In this write strategy, data is first written to the cache and then to the database. The cache sits in-line with the database and writes always go _through_ the cache to the main database. This helps cache maintain consistency with the main database.

![write-through](https://codeahoy.com/img/write-through.png)

Here’s what happens when an application wants to write data or update a value:

1.  The application writes the data directly to the cache.
2.  The cache updates the data in the main database. When the write is complete, both the cache and the database have the same value and the cache always remains consistent.

#### Use Cases, Pros and Cons

On its own, write-through caches don’t seem to do much, in fact, they introduce extra write **latency** because data is written to the cache first and then to the main database (two write operations.) But when paired with read-through caches, we get all the benefits of read-through and we also get data **consistency** guarantee, freeing us from using cache invalidation (assuming ALL writes to the database go through the cache.)

[DynamoDB Accelerator (DAX)](https://aws.amazon.com/dynamodb/dax/) is a good example of read-through / write-through cache. It sits inline with DynamoDB and your application. Reads and writes to DynamoDB can be done through DAX. (Side note: If you are planning to use DAX, please make sure you familiarize yourself with [its data consistency model](http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DAX.consistency.html) and how it interplays with DynamoDB.)

### Write-Around

Here, data is written directly to the database and only the data that is read makes it way into the cache.

#### Use Cases, Pros and Cons

Write-around can be combine with read-through and provides good performance in situations where data is written once and read less frequently or never. For example, real-time logs or chatroom messages. Likewise, this pattern can be combined with cache-aside as well.

### Write-Back or Write-Behind

Here, the application writes data to the cache which stores the data and acknowledges to the application _immediately_. Then later, the cache writes the data _back_ to the database.

This is very similar to to Write-Through but there’s one crucial difference: In Write-Through, the data written to the cache is **synchronously** updated in the main database. In Write-Back, the data written to the cache is **asynchronously** updated in the main database. From the application perspective, writes to Write-Back caches are _faster_ because only the cache needed to be updated before returning a response.

![write-back](https://codeahoy.com/img/write-back.png)

This is sometimes called write-behind as well.

#### Use Cases, Pros and Cons

Write back caches improve the write performance and are good for **write-heavy** workloads. When combined with read-through, it works good for mixed workloads, where the most recently updated and accessed data is always available in cache.

It’s resilient to database failures and can tolerate some database downtime. If batching or coalescing is supported, it can reduce overall writes to the database, which decreases the load and **reduces costs**, if the database provider charges by number of requests e.g. DynamoDB. Keep in mind that **DAX is write-through** so you won’t see any reductions in costs if your application is write heavy. (When I first heard of DAX, this was my first question - DynamoDB can be very expensive, but damn you Amazon.)

Some developers use Redis for both cache-aside and write-back to better absorb spikes during peak load. The main disadvantage is that if there’s a cache failure, the data may be permanently lost.

Most relational databases storage engines (i.e. InnoDB) have write-back cache enabled by default in their internals. Queries are first written to memory and eventually flushed to the disk.


## Distributed caching solutions

Distributed caching solutions have evolved over the years to cater to the growing demands of scalable and high-performance applications:

**Redis** is an open-source, in-memory data structure store that can be used as a cache, [database](https://redis.io/blog/redis-cache-vs-redis-primary-database-in-90-seconds/), and [message broker](https://redis.io/solutions/messaging/). It supports various data structures such as strings, hashes, lists, and sets. Redis is known for its high performance, scalability, and support for data replication and persistence.

**Memcached** is a general-purpose distributed memory caching system. It is designed to speed up dynamic web applications by reducing database load. Memcached is simple yet powerful, supporting a large number of simultaneous connections and offering a straightforward key-value storage mechanism.

**Hazelcast** is an in-memory data grid that offers distributed caching, messaging, and computing. It provides features like data replication, partitioning, and native memory storage. Hazelcast is designed for cloud-native architectures and can be easily integrated with popular [cloud](https://redis.io/redis-enterprise/cloud/) platforms.

**Apache Ignite** is an in-memory computing platform that provides distributed caching, data processing, and [ACID-compliant transactions](https://redis.io/glossary/acid-transactions/). It can be used as a distributed cache, database, and message broker. Apache Ignite supports data replication, persistence, and querying capabilities.

## Resources
