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

Each shard is a standalone database with its own CPU, memory, storage, and connection pool. No single machine holds all the data or handles all the traffic, which allows both storage capacity and read/write throughput to scale as you add more shards. Sharding solves scaling but introduces new problems. You now have to choose a shard key, route queries to the right shard, avoid hotspots, and rebalance data as shards grow. 

## How to shard your data?

### Choosing your shard key

In an interview, a common statement is "I'm going to shard by field X". The key is knowing what field to use as your shard key and why. Bad shard key leads to uneven data distribution, hot spots where one shard gets pounded while others sit idle, and queries that have to hit every shard to find what they need. A good shard key distributes data evenly, aligns with your query patterns, and scales as your system grows.

Here is what makes a good shard key: 

1. High Cardinality - The key should have many unique values. Sharding by a boolean field (true/false) means you can only have two shards max, which defeats the purpose. Sharding by user ID when you have millions of users gives you plenty of room to distribute data.
2. Even distribution - Values should spread evenly across shards. If you shard by country and 90% of your users are in the US, that shard will be massively larger than the others. User ID usually distributes well. 
3. Aligns with queries - Your most common queries should ideally hit just one shard. If you shard users by `user_id`, queries like "get user profile" or "get user's orders" hit a single shard. Queries that span all shards become expensive.

### Sharding Strategies 

Once you know your shard key, you need to decide how to distribute that data across shards. There are three main strategies, each with different trade-offs.

#### Range Based Sharding

Range sharding is the most straightforward. It just groups records by a continuous range of values. You pick a shard key like `user_id` then assign value ranges to shards. For example, we might assign the first 1 million users to shard 1, the next 1 million users to shard 2, and so on. 

The main advantage of range-based sharding is simplicity and support for efficient range scans. If you need all orders between user IDs 500K and 600K, you only hit one shard. Range-based sharding works best when different users naturally query different ranges. Multi-tenant systems, for example, are a good fit.

#### Hash Based Sharding (Default)

Hash sharding uses a hash function to evenly distribute records across shards. Instead of assigning ranges, you take a shard key like `user_id`, hash it, and use the result to pick a shard. This is the default and the most common sharding strategy to use in System Design Interviews. 

The big advantage of hash-based sharding is even distribution. Since the hash function scrambles the input values, new users get distributed evenly across all shards.

The flip side of that scrambling is that you lose all locality. Keys that were adjacent in value (user IDs 500K–600K, or a time range of orders) land on different shards, so range scans and ordered reads that were cheap under range-based sharding now become scatter-gather queries across every shard. Hash sharding also makes the "query by a non-shard-key field" problem worse, since there's no ordering to exploit. Pick hash when your access pattern is point lookups by the shard key; pick range when you genuinely need range scans.

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
3. Dynamic shard splitting: Many managed and distributed databases automatically split a shard (or partition) when it gets too large or too hot, then rebalance the pieces across nodes. MongoDB does this with chunk splitting and its balancer; DynamoDB splits partitions under load; and Vitess, CockroachDB, and Spanner all split ranges automatically. This is one of the biggest reasons to reach for a system with built-in resharding rather than rolling your own.

### Cross-Shard Operations

When your data lives on multiple machines, any query that needs data from more than one shard becomes expensive. Instead of querying one database, you have to query multiple shards, wait for all of them to respond, and aggregate the results yourself. You can't eliminate cross-shard queries entirely, but you can minimize them:

1. Cache the results - If "top 10 most popular posts" requires hitting all shards, cache the result for 5 minutes. The first query is expensive, but the next thousand requests hit the cache instead of querying all 64 shards. This works especially well for queries that don't need real-time accuracy (ie. eventual consistency is acceptable). Leaderboards, trending content, and aggregate stats are perfect candidates.
2. Denormalize to keep related data together: If you frequently need to query posts along with user data, store some post information directly on the user's shard. This can lead to duplicate data and make updates difficult but it lets you query everything from one shard.
3. Accept the hit for rare queries: Sometimes a query genuinely needs to hit all shards and that's okay as long as it's infrequent. An admin dashboard that shows "total users across all shards" can afford to be slow if it's only loaded a few times a day.

### Querying by a Non-Shard-Key Field

Sharding optimizes for queries that include the shard key. But real systems query by other fields too. If you shard users by `user_id` but need to look someone up by `email`, you don't know which shard holds them, so you're forced into a scatter-gather across every shard. This is one of the most common real-world sharding problems and it's easy to overlook when you only think about the happy-path query.

Ways to handle it:

1. Global secondary index table - Maintain a separate lookup table that maps the secondary key to the shard (or primary key), e.g. `email -> user_id`. This index can itself be sharded, by the secondary key. The lookup becomes two hops (index shard, then data shard) instead of a scatter-gather. The cost is keeping the index in sync with the base data, usually asynchronously, which means it can lag.
2. Duplicate the data under a second shard key - Store the same record twice, sharded two different ways (once by `user_id`, once by `email`). Fast reads on both access patterns, at the cost of double writes and keeping the copies consistent.
3. Scatter-gather and cache - For infrequent secondary lookups, just query all shards in parallel and merge. Acceptable when the query is rare; back it with a cache if it isn't.

The general rule: every access pattern you care about needs *some* structure that lets you route it to one shard. If you have many such patterns, that's a signal you may want a search index (Elasticsearch) or a separate read model rather than bending your shards to serve all of them.

### Distributed ID Generation

Once data spans multiple databases, per-shard auto-increment IDs collide, two shards will both hand out ID 1, 2, 3. You need IDs that are unique across the whole system and, ideally, that don't require a round trip to a central coordinator on every insert.

Common approaches:

1. UUIDs - Simple and fully decentralized, but random UUIDs are large (128-bit) and destroy locality in indexes. UUIDv7 (time-ordered) mitigates the locality problem.
2. Snowflake-style IDs - A 64-bit ID composed of a timestamp, a machine/shard ID, and a per-node sequence number. Roughly time-ordered, compact, and generated locally without coordination. This is the common production choice (Twitter Snowflake, Instagram's variant, Sonyflake).
3. Central ticket server / ranges - A dedicated service hands out ID blocks to each node; nodes allocate from their block locally and only call back when the block is exhausted. Adds a dependency but keeps IDs dense.

Note that if the ID itself embeds the shard (as Instagram's scheme does), the ID *becomes* your routing key, you can locate a record from its ID alone without a directory lookup.

### Referential Integrity Across Shards

A single database enforces foreign keys for you. Once related rows live on different shards, the database can no longer enforce that a referenced row exists, foreign keys simply don't span shards. This has two consequences a Staff Engineer should call out:

1. The application owns integrity now - Cascading deletes, orphan cleanup, and "does this parent exist" checks move into application code or background reconciliation jobs. Plan for orphaned rows and build sweepers that detect and clean them.
2. It reinforces shard-key design - This is another argument for co-locating related data on the same shard. Data you'd want a foreign key between is data you probably want on the same shard, where local constraints still work.

### Maintaining Consistency 

Once you shard your data, it now lives in multiple databases. A single DB can no longer handle consistency guarantees if the data is split across DBs. 

The textbook solution when multiple databases are involved is to use Two-Phase-Commit (2PC). The coordinator asks all shards to prepare the transaction, waits for everyone to confirm they're ready, then tells everyone to commit. This guarantees consistency but is slow and fragile. If any shard or the coordinator fails mid-transaction, the whole system can get stuck. Most production systems avoid 2PC because the performance and reliability trade-offs aren't worth it.

Instead of 2PC, you can instead do the following

1. Design to avoid cross-shard transactions: The best distributed transaction is the one you never make. Choose a shard key that co-locates data that gets written together. If a user and their orders live on the same shard, "place an order" stays a single-shard transaction with normal ACID guarantees. Most consistency pain comes from a shard key that splits data across boundaries that your writes need to cross.
2. Use sagas for multi-shard operations: When you absolutely need to coordinate across shards, use the saga pattern. Break the operation into a sequence of independent steps, each with a compensating action. If step 3 fails, you run compensating actions for steps 2 and 1 to undo the work. This gives you eventual consistency without the fragility of 2PC.
3. Accept eventual consistency: For many operations, strict consistency isn't required. If you're updating a user's follower count and that count is denormalized across multiple shards for fast profile lookups, it's fine if some shards show different counts for a few seconds. Eventually all shards will converge to the correct number. This is much simpler than coordinating a distributed transaction, and for most applications, a brief period of inconsistency is acceptable.

## Operating Sharded Systems

The sections above cover the theory. This section covers what it actually takes to run a sharded system in production, the parts that separate "understands sharding" from "has operated it."

### The Routing Layer

Something has to translate a query into "which shard does this go to." Where that logic lives is an architectural decision with real trade-offs.

1. Client-side routing - The application (or a shared client library) knows the shard map and connects directly to the right shard. Lowest latency, no extra hop, no extra service to run. The downside is that every client must have the shard map and agree on it; rolling out a topology change means redeploying or reconfiguring every client, and a buggy client can talk to the wrong shard.
2. Proxy / router tier - A dedicated tier (Vitess `vtgate`, ProxySQL, a Redis Cluster proxy) sits between the app and the shards. Clients talk to the proxy as if it were a single database; the proxy owns routing, connection pooling, and often cross-shard query fan-out. This centralizes topology changes (update the proxy, not every client) at the cost of an extra network hop and another tier to run and scale.
3. Coordinator service - A lookup/coordinator service (this is what directory-based sharding leans on) answers "where does key X live." Maximum flexibility, but it's on the critical path for every request and becomes a dependency you must make highly available and cache aggressively.

In practice most large systems converge on a proxy tier, because centralizing the shard map is what makes safe resharding possible.

### Shard Map / Topology Management

The mapping of key ranges (or hash slots) to physical shards is itself critical state. Getting it wrong causes silent correctness bugs, not just outages: a client with a stale map writes to the old shard while reads go to the new one.

Key properties a shard map needs:

1. A single source of truth - Store the topology in a consistent store (ZooKeeper, etcd, Consul, or the database's own metadata layer). Everyone reads from, or is pushed updates from, that authority.
2. Versioning - Stamp the map with a version. Requests and clients can carry the version they used, so the system can detect and reject actions taken against a stale map instead of silently corrupting data.
3. Consistent distribution during change - The dangerous window is a migration, when the map is mid-change. During cutover you need every reader and writer to agree on where a key lives at any given instant. This is exactly why a centralized routing/proxy tier is valuable: you flip the map in one place rather than racing many clients.

### Replication and Sharding Together

The earlier sections describe a shard as "a single machine." In production a shard is almost never one machine, it's a replicated cluster: a primary plus one or more replicas. Sharding and replication are orthogonal and you use both at once: sharding scales writes and total capacity, replication gives each shard high availability and read scaling.

This layering means each shard has its own:

1. Failover - If a shard's primary dies, that shard runs a leader election / promotes a replica. Only that shard's keyspace is affected; the rest of the system keeps serving. Your routing layer has to notice the new primary and redirect writes.
2. Read-replica routing - Reads can be served from replicas within the shard to scale read throughput, at the cost of replication lag (a read may not see the latest write). The router decides primary-vs-replica per query based on the consistency the query needs.
3. Failure-domain awareness - Spread a shard's replicas across availability zones so a single zone loss doesn't take the whole shard down.

The practical takeaway: "add a shard" really means "stand up a new replicated cluster," and your capacity math must account for the replica multiplier (N shards × R replicas).

### Resharding and Rebalancing

This is the hardest part of operating shards, and the part most often glossed over. As shards grow or get hot, you need to split them or add new ones and move data, ideally without downtime and without losing writes.

The mechanics of a live shard split / migration:

1. Provision the target - Stand up the new shard(s) that will receive the data.
2. Backfill - Bulk-copy the existing data for the keys being moved from the source shard to the target, while the source keeps serving traffic. This is a snapshot copy that runs in the background.
3. Dual-write / catch-up - Because the source is still taking writes during the backfill, you must capture changes made after the snapshot. Either dual-write to both source and target from the moment backfill starts, or tail the source's replication log / change stream to apply the delta to the target until it catches up.
4. Verify - Compare source and target for the migrated keys (row counts, checksums) to confirm they're in sync before you trust the target.
5. Cutover - Flip the shard map so reads and writes for the moved keys now go to the target. This is the atomic, version-gated moment; a brief freeze or write-lock on just the affected keys is common to guarantee no lost writes.
6. Clean up - Once you're confident, delete the migrated data from the source and stop dual-writing.

Two things make this dramatically easier:

- Consistent hashing to minimize how much data moves when you add or remove a shard. (Covered in a separate document.)
- A centralized routing/proxy tier so the cutover is a single map flip rather than a coordinated change across every client.

Always have a rollback path. Until you delete the source data (step 6), the source is still authoritative and you can flip the map back. Test resharding before you need it, doing it for the first time during an incident is how you lose data.

### Operational Concerns

The remaining realities of running N databases instead of one:

1. Schema migrations fan out - A migration now has to run across every shard. You must handle partial failure (migration succeeds on 40 of 64 shards) and version skew (application code must work against both old and new schema while the rollout is in flight). Make migrations backward-compatible and roll them out in the expand/contract pattern.
2. Backups and recovery are per-shard - Every shard needs its own backups and point-in-time recovery, and restoring to a globally consistent moment across shards is genuinely hard (there's no single WAL). For most systems, per-shard PITR to "close enough" plus application-level reconciliation is the pragmatic answer.
3. Connection pool exhaustion - With M app servers each holding a pool to N shards, connection count is M × N and grows with both. This can overwhelm the databases well before CPU or storage does. A proxy tier that multiplexes connections is a common fix.
4. Observability per shard - Aggregate dashboards hide skew, the whole point of watching for hot spots is per-shard visibility. Track latency, CPU, QPS, storage, and connection count per shard, and alert on divergence between shards, not just absolute thresholds.
5. Capacity planning and headroom - Decide the initial shard count deliberately: too few and you're resharding again immediately, too many and you carry needless operational overhead. A common heuristic is to over-provision logical shards (e.g. 1024) and map many logical shards onto each physical node, so growth is "move logical shards to new nodes" rather than "split," which is far cheaper. Leave headroom so a traffic spike doesn't force an emergency reshard.

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
