# Sharding Essentials

Sharding allows you to split your data across multiple machines. It becomes a necessity at scale. 

## Partitioning vs Sharding

People often use the words "partitioning" and "sharding" to mean the same thing. Technically they are slightly different. Partitioning usually refers to splitting data within a single database instance, often by table ranges or hash partitions. Sharding means splitting data across multiple machines. In practice most engineers use the terms loosely, so do not get hung up on the wording. Just be clear about whether your data lives on one machine or many.

### Partitioning

Partitioning means splitting a large table into smaller pieces inside a single database instance. It does not add more machines. Instead it organizes data so the database can work more efficiently. For example - A query for last month’s orders only scans the relevant partition instead of the full table.

There are two common types of partitioning

1. Horizontal partitioning - Split rows across partitions. For example, one partition per year of orders. Same columns, fewer rows per partition.
2. Vertical partitioning - Split columns across partitions. For example, keep frequently accessed columns in one partition and large or rarely used columns in another. Same rows, fewer columns per partition.

### Sharding 

Sharding is horizontal partitioning across multiple machines. Each shard holds a subset of the data, and together the shards make up the full dataset. Unlike partitioning, which stays within a single database instance, sharding spreads the load across many independent databases.

Each shard is a standalone database with its own CPU, memory, storage, and connection pool. No single machine holds all the data or handles all the traffic, which allows both storage capacity and read/write throughput to scale as you add more shards.

Sharding solves scaling but introduces new problems. You now have to choose a shard key, route queries to the right shard, avoid hotspots, and rebalance data as shards grow. We will cover how to handle these next.

## How to shard your data?

### Choosing your shard key

In an interview, a common statement is "I'm going to shard by field X". The key is knowing what field to use as your shard key and why. Bad shard key leads to uneven data distribution, hot spots where one shard gets pounded while others sit idle, and queries that have to hit every shard to find what they need. A good shard key distributes data evenly, aligns with your query patterns, and scales as your system grows.

Here is what makes a good shard key: 

1. High Cardinality - The key should have many unique values. Sharding by a boolean field (true/false) means you can only have two shards max, which defeats the purpose. Sharding by user ID when you have millions of users gives you plenty of room to distribute data.
2. Even distribution - Values should spread evenly across shards. If you shard by country and 90% of your users are in the US, that shard will be massively larger than the others. User ID usually distributes well. 
3. Aligns with queries - Your most common queries should ideally hit just one shard. If you shard users by `user_id`, queries like "get user profile" or "get user's orders" hit a single shard. Queries that span all shards become expensive.

### Sharding Stratgies 

Once you know your shard key, you need to decide how to distribute that data across shards. There are three main strategies, each with different trade-offs.

#### Range Based Sharding

Range sharding is the most straightforward. It just groups records by a continuous range of values. You pick a shard key like `user_id` then assign value ranges to shards. For example, we might assign the first 1 million users to shard 1, the next 1 million users to shard 2, and so on. 

The main advantage of range-based sharding is simplicity and support for efficient range scans. If you need all orders between user IDs 500K and 600K, you only hit one shard. Range-based sharding works best when different users naturally query different ranges. Multi-tenant systems, for example, are a good fit.

#### Hash Based Sharding (Default)

Hash sharding uses a hash function to evenly distribute records across shards. Instead of assigning ranges, you take a shard key like `user_id`, hash it, and use the result to pick a shard. This is the default and the most common sharding strategy to use in System Design Interviews. 

The big advantage of hash-based sharding is even distribution. Since the hash function scrambles the input values, new users get distributed evenly across all shards.

The downside shows up when you need to add or remove shards. With naive sharding a resize implies almost every record maps to a different shard. You have to move massive amounts of data around. This is where consistent hashing comes in. Instead of simple modulo, consistent hashing minimizes data movement when you add or remove shards.

#### Directory Based Sharding 

Directory sharding uses a lookup table to decide where each record lives. Instead of using a formula, you store shard assignments in a mapping table or service. 

The power of directory-based sharding is flexibility. If a particular user generates tons of traffic, you can move them to a dedicated shard. If you need to rebalance load, you just update the mapping table. 

The downside is that every single request requires a lookup. Before you can query user data, you have to ask the directory service which shard that user lives on. This adds latency to every request and makes the directory service a critical dependency. If the directory goes down, your entire system stops working even if all the data shards are healthy.

Directory-based sharding makes sense when you need maximum flexibility and can afford the extra lookup cost. 

## Challenges of Sharding

### Hot Spots and Load Imbalance

Even with a good shard key, some shards can end up handling way more traffic than others. This is called a hot spot, and it negates the main benefit of sharding because one overloaded shard becomes your bottleneck. You can detect hot spots by monitoring shard metrics like query latency, CPU usage, and request volume. When one shard consistently shows higher metrics than others, you have a hot spot problem.

Here is how to handle Hot Spots: 

1. Isolate Hot Keys to Dedicated Shards - If Taylor Swift's account generates too much traffic, move it to a dedicated shard that only handles celebrity accounts. This is why directory-based sharding can be useful for specific cases
2. Use Compound Shard Keys - Instead of sharding just by `user_id`, combine it with another dimension like `hash(user_id + date)`. This spreads a single user's data across multiple shards over time, which helps if the hot spot is both high volume and spans time periods.
3. Dynamic shard splitting: Some databases support automatically splitting a shard when it gets too large or too hot. This happens in MongoDB.

### Cross-Shard Operations

When your data lives on multiple machines, any query that needs data from more than one shard becomes expensive. Instead of querying one database, you have to query multiple shards, wait for all of them to respond, and aggregate the results yourself. You can't eliminate cross-shard queries entirely, but you can minimize them:

1. Cache the results - If "top 10 most popular posts" requires hitting all shards, cache the result for 5 minutes. The first query is expensive, but the next thousand requests hit the cache instead of querying all 64 shards. This works especially well for queries that don't need real-time accuracy (ie. eventual consistency is acceptable). Leaderboards, trending content, and aggregate stats are perfect candidates.
2. Denormalize to keep related data together: If you frequently need to query posts along with user data, store some post information directly on the user's shard. This can lead to duplicate data and make updates difficult but it lets you query everything from one shard.
3. Accept the hit for rare queries: Sometimes a query genuinely needs to hit all shards and that's okay as long as it's infrequent. An admin dashboard that shows "total users across all shards" can afford to be slow if it's only loaded a few times a day.

### Maintaining Consistency 

Once you shard your data, it now lives in multiple databases. A single DB can no longer handle consistency guarantees if the data is split across DBs. 

The textbook solution when multiple databases are involved is to use Two-Phase-Commit (2PC). The coordinator asks all shards to prepare the transaction, waits for everyone to confirm they're ready, then tells everyone to commit. This guarantees consistency but is slow and fragile. If any shard or the coordinator fails mid-transaction, the whole system can get stuck. Most production systems avoid 2PC because the performance and reliability trade-offs aren't worth it.

Instead of 2PC, you can instead do the following

1. Design to avoid cross-shard transactions:
2. Use sagas for multi-shard operations: When you absolutely need to coordinate across shards, use the saga pattern. Break the operation into a sequence of independent steps, each with a compensating action. If step 3 fails, you run compensating actions for steps 2 and 1 to undo the work. This gives you eventual consistency without the fragility of 2PC.
3. Accept eventual consistency: For many operations, strict consistency isn't required. If you're updating a user's follower count and that count is denormalized across multiple shards for fast profile lookups, it's fine if some shards show different counts for a few seconds. Eventually all shards will converge to the correct number. This is much simpler than coordinating a distributed transaction, and for most applications, a brief period of inconsistency is acceptable.

## Sharding in System Design Interviews

Sharding comes up just about anytime you are discussing scaling. The key is knowing when to bring it up, what to say, and what mistakes to avoid.

### When to Mention Sharding?

Do not make the mistake of prematurely sharding. You need to establish why a single database won't work first.
Bring up sharding when you're discussing capacity planning and hit one of these limits:

1. Storage: "We have 500M users with 5KB of data each, that's 2.5TB. A single Postgres instance can handle that, but if we grow 10x we'll need to shard."
2. Write throughput: "We're expecting 50K writes per second during peak. A single database will struggle with that write load, so we should shard."
3. Read throughput: "Even with read replicas, if we're serving 100M daily active users making multiple queries each, we'll need to distribute the read load across shards."

The formula is simple:

1. Identify the bottleneck
2. Explain why single database won't scale
3. Propose sharding

### What to Say during interviews

1. Propose a Shard Key based on your access pattern
2. Choose your Distribution Strategy (Default to Hash based sharding with consistent hashing)
3. Call out the tradeoffs (Global queries become expensive)
4. Address how you will handle growth (Start with x number of shards, consistent hashing makes it easier to add/remove shards later with minimal data movement)