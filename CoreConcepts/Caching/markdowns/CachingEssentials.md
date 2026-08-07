# Caching Essentials

In system design interviews, caching comes up almost every time you need to handle high read traffic. Your database becomes the bottleneck, latency starts creeping up, and the interviewer is waiting for you to say the word: cache. Caches are essential for scalable systems. They reduce load on the database and cut latency dramatically. But they also create new challenges around invalidation and failure handling.

## Where to Cache?

When most engineers hear caching, they immediately think of Redis or Memcached sitting between the application and the database. It is the most common type of cache and the one interviewers care about the most. But caching shows up in multiple layers of a system. Browsers cache. CDNs cache. Applications cache. Even databases have built-in caching layers. Let's look at the main places you can cache data and when it makes sense to use it. 

### External Caching 

An external cache is a standalone cache service that your application talks to over the network. You store frequently accessed data in something like Redis or Memcached so you do not have to hit the database every time. External caches scale well because every application server can share the same cache. They also support eviction policies like LRU and expiration via TTL so your memory footprint stays controlled.

![External Caching](../assets/ExternalCaching.png)

In system design interviews, external caching with Redis is the default answer when discussing caching strategies. Interviewers expect you to mention it for any high-traffic system. Start here, then layer on other caching types such as CDN or client-side caching only if the problem calls for them.

### CDN (Content Delivery Network)

A CDN is a geographically distributed network of servers that caches content close to users. Instead of every request traveling to your origin server, a CDN stores copies of your content at edge servers around the world.

Modern CDNs like Cloudflare, Fastly, and Akamai can cache much more than static files. They can also cache public API responses, HTML pages, and even run edge logic to personalize content or enforce security rules before requests reach your servers. But the most common and most impactful use of a CDN is still media delivery. Talk about CDNs in System Design interviews if you are asked to serve static media at scale. 

### Client-side Caching

Client-side caching stores data close to the requester to avoid unnecessary network calls. This usually means the user's device. For user-facing caching, you have limited control from the backend. Data can go stale and invalidation is harder.

![Client-side Caching](../assets/ClientSideCaching.png)

### In-process Caching

If your service keeps requesting the same small pieces of data again and again, store them in a local cache inside the process. Reads from local memory are even faster than reads from Redis because they avoid any network call. This light-weight form of caching makes sense for small pieces of data that are requested frequently like:

1. Configuration values
2. Feature flags
3. Small reference datasets
4. Hot keys
5. Rate limiting counters
6. Precomputed values

![In-process Caching](../assets/InProcessCaching.png)

In-process caching is blazing fast, but it comes with obvious limitations. Each instance of your application has its own cache, so cached data is not shared across servers. If one instance updates or invalidates a cached value, the others will not know.

## Cache Architectures

There are 4 core cache patterns you should know for System Design Interviews

### Cache Aside (Lazy Loading)

This is the most common caching pattern and the one you should default to in interviews.

1. Application checks the cache.
2. If the data is there, return it.
3. If not, fetch from the database, store it in the cache, and return it.

![Cache Aside](../assets/CacheAside.png)

Cache-aside only caches data when needed, which keeps the cache lean. The downside is that a cache miss causes extra latency. If you only remember one caching pattern for interviews, make it cache-aside.

### Write Through Cache

With write-through caching, the application writes only to the cache. The cache then synchronously writes to the database before returning to the application. The write operation does not complete until both the cache and database are updated. 

This requires a cache implementation that supports write-through, like a caching library. When you write to the cache, the library handles calling your database write logic before acknowledging the write.

![Write Through Caching](../assets/WriteThroughCaching.png)

The tradeoff is slower writes because the application must wait for both the cache update and the database write to complete. Write-through can also pollute the cache with data that may never be read again. Write-through also suffers from the dual-write problem. If the cache update succeeds but the database write fails, or vice versa, the systems can end up inconsistent. You need retry logic, error handling, or eventually accept that perfect consistency is difficult without distributed transactions.

Use this pattern when reads must always return fresh data and your system can tolerate slightly slower writes.

### Write Behind (Write Back) Caching 

The application writes only to the cache. The cache batches and writes the data to the database asynchronously in the background.

![Write Behind Caching](../assets/WriteBehindCaching.png)

This makes writes very fast, but introduces risk. If the cache crashes before flushing, you can lose data. This is best for workloads where occasional data loss is acceptable. Use this when you need high write throughput and eventual consistency is acceptable. Common in analytics and metrics pipelines.

### Read Through Caching

With read-through caching, the cache acts as a smart proxy. Your application never talks to the database directly. On a cache miss, the cache itself fetches from the database, stores the data, and returns it. This is the read equivalent of write-through. In both patterns, the cache acts as an intermediary that handles database operations.

![Read Through Caching](../assets/ReadThroughCaching.png)

CDNs are a form of read-through cache. When a CDN gets a cache miss, it fetches from your origin server, caches the result, and returns it. But for application-level caching with Redis, cache-aside is far more common.

## Cache Eviction Policies

Caches have limited memory, so they need a strategy for deciding which entries to remove when full. These strategies are called eviction policies.

1. LRU (Least Recently Used)
- LRU evicts the item that has not been accessed for the longest time. It tracks access order using a linked list or ring buffer so the least recently used item can be removed in constant time.
- Adapts well to most workloads where recently used data is likely to be used again.
2. LFU (Least Frequently Used)
- LFU evicts the item that has been accessed the least. It maintains a counter for each key and removes the one with the lowest frequency. Some implementations use approximate LFU to avoid the cost of precise frequency tracking.
- Used when you want to cache consistenly popular keys like Trending Videos or Top Playlists
3. FIFO (First In First Out)
- FIFO evicts the oldest item in the cache based only on insertion time. It can be implemented with a simple queue, but it ignores usage patterns.
- Because it may evict items that are still hot, it is rarely used in real systems beyond simple caching layers.
4. TTL (Time to live)
- TTL is not an eviction policy by itself. Instead, it sets an expiration time for each key and removes entries that are too old. It is often combined with LRU or LFU to balance freshness and memory usage.
5. Random
- Evicts a randomly chosen key. It sounds crude, but it needs almost no bookkeeping and performs surprisingly well when access is fairly uniform. Redis actually uses approximated, sampled variants of LRU and LFU (it checks a small random sample rather than tracking every key precisely) for exactly this reason: precise tracking is expensive at scale.

Modern caching libraries increasingly use hybrid policies like W-TinyLFU (used by Caffeine), which combines a frequency sketch with a recency window to get the best of LRU and LFU. For interviews, LRU is the safe default answer and TTL is essential for freshness, but it is worth knowing that pure LRU/LFU is rarely what production systems run.

## Cache Key Design

Keys are the interface to your cache, and how you design them decides whether the cache is easy to reason about, easy to invalidate, and safe in a shared environment. It is easy to overlook, but poor key design is a common source of subtle cache bugs.

### Naming and namespacing

Use structured, hierarchical keys with a consistent separator, typically a colon: `user:123:profile`, `post:456:comments`. A few rules that pay off:

1. Namespace by entity and id. `user:123:profile` tells you at a glance what the key holds and makes it easy to find related keys. Avoid opaque keys like a bare hash with no context.
2. Include everything the value depends on. If a cached response varies by locale or API version, that has to be in the key: `product:789:en-US`. A key that omits a dimension the value depends on will serve the wrong data to someone.
3. Keep keys short but readable. Keys live in memory too, and millions of long keys add up. Balance readability against size.
4. Watch for collisions. Two different concepts that hash or serialize to the same key string will silently overwrite each other. A disciplined namespace prevents this.

### Versioning keys

Versioning is a powerful alternative to delete-on-write for invalidation, and it sidesteps the read-populates-stale race described in the consistency section. Instead of deleting an entry when data changes, you change the key so the old entry is simply never looked up again and ages out on its own via TTL.

There are two common flavors:

1. Schema or deploy versioning: Prefix keys with a global version, for example `v2:user:123:profile`. When you change the shape of what you cache, or ship a deploy that would make old cached values invalid, bump the version. Every reader instantly starts on fresh keys and the entire old generation is abandoned at once, no scan or bulk delete required. This is the cleanest way to invalidate a whole category of keys.
2. Entity version tags: Store a version number or timestamp for an entity and fold it into the key, for example `user:123:profile:v8` where `v8` comes from the user's `updated_at` or a version counter. On write you bump the entity's version; the next read composes a new key and misses cleanly, while the stale entry expires on its own.

The tradeoff is that abandoned entries linger until their TTL expires rather than being reclaimed immediately, so versioning leans on eviction and TTL to clean up. That is usually a fine price for avoiding both the dual-write problem and the stale-repopulation race. It is why versioning is a favorite in systems that cannot tolerate the consistency window that delete-on-write leaves open.

Two things to keep in mind whichever scheme you use: always attach a TTL (versioning relies on old generations aging out, so an entry with no TTL leaks forever), and remember that a version bump that touches many keys at once is itself an avalanche, so the earlier avalanche defenses still apply.

## Common Caching Problems

### Cache Stampede (Thundering Herd)

A cache stampede happens when a popular cache entry expires and many requests try to rebuild it at the same time. There is a brief window, even if only a second, where every request misses the cache and goes straight to the database. Instead of one query, you suddenly have hundreds or thousands, which can overload the database.

How to handle it?

1. Request coalescing (single flight): Allow only one request to rebuild the cache while others wait for the result. This is the most effective solution.
2. Cache warming: Refresh popular keys proactively before they expire. This only helps when using TTL-based expiration. If you invalidate cache on writes instead, warming does not prevent stampedes.
3. Probabilistic early expiration: Have requests randomly refresh a key slightly before its TTL expires, with the probability rising as expiry approaches. One unlucky request rebuilds the value while everyone else keeps serving the still-valid cached copy, so the expiry never lands on all requests at once.

### Cache Avalanche

A stampede is many requests missing on a single key. An avalanche is the same problem across many keys at once. It happens when a large number of entries share the same expiration time and expire together, or when the cache node restarts cold. Suddenly a huge fraction of traffic misses simultaneously and floods the database.

How to handle it?

1. TTL jitter: Add a small random offset to each key's TTL (for example, 60 seconds plus or minus 10) so keys expire spread out over time instead of all at once. This is the single most important fix and costs almost nothing.
2. Cache warming on startup: Preload hot keys before sending traffic to a freshly restarted cache node so it does not start cold.
3. Layered fallback: A local in-process cache in front of Redis absorbs some of the load if the shared cache expires en masse.

### Cache Penetration

Stampede and avalanche are about entries that exist and expire. Penetration is the opposite: requests for keys that do not exist in the cache or the database. Because there is nothing to cache, every one of these requests falls through to the database. This is often accidental (a bad client looping over missing IDs) but can also be a deliberate attack that requests random non-existent keys to bypass the cache entirely.

How to handle it?

1. Negative caching: Cache the "not found" result too, with a short TTL. A repeated lookup for a missing key then hits the cache instead of the database. Keep the TTL short so a key that later gets created is not masked for long.
2. Bloom filter: Keep a Bloom filter of keys that are known to exist. Check it before touching the cache or database. If the filter says a key definitely does not exist, reject the request immediately. Bloom filters can have false positives but never false negatives, which is exactly the guarantee you want here.
3. Input validation: Reject malformed or out-of-range keys at the edge before they ever reach the cache.

### Cache Consistency 

This happens when the cache and DB return different values for the same key. This is common because most systems read from the cache but write to the database first. That creates a window where the cache still holds stale data.

How to handle it?

1. Cache invalidation on writes: Delete the cache entry after updating the database so it gets repopulated with fresh data.
2. Short TTLs for stale tolerance: Let slightly stale data live temporarily if eventual consistency is acceptable.
3. Accept eventual consistency: For feeds, metrics, and analytics, a short delay is usually fine.

#### Delete, don't update

When a write happens, prefer deleting the cache entry over writing the new value into it. Two reasons. First, updating the cache re-runs the dual-write problem: if the DB write succeeds but the cache write fails, you are left with stale data and no easy way to know. Deleting is simpler and self-healing, since the next read repopulates from the source of truth. Second, writing the value eagerly pollutes the cache with data that may never be read again. Delete-on-write keeps the cache lean, exactly like cache-aside.

#### The read-populates-stale race

Even delete-on-write has a subtle race. Consider this interleaving:

1. Request A reads, misses the cache, and fetches the old value from the DB.
2. Request B writes the new value to the DB and deletes the cache entry.
3. Request A now writes the old value it read in step 1 back into the cache.

The cache is left holding stale data with no pending invalidation to fix it. It stays wrong until the TTL expires. In practice a short TTL bounds the damage, which is one more reason to always set a TTL even on entries you invalidate explicitly. Systems that cannot tolerate this window use delayed double-delete (delete, then delete again after a short delay to catch late repopulations) or bind the cache to the database's change stream (CDC) so invalidation is driven by the commit log rather than application code.

#### Ordering matters

Always write to the database first, then invalidate the cache. If you invalidate first and then write, a concurrent read can repopulate the cache from the old DB value before your write commits, and you are stale again. The database is the source of truth, so it commits first.

### Hot Keys

A hot key is a cache entry that receives a huge amount of traffic compared to everything else. Even if the cache hit rate is high, a single hot key can overload one cache node or one Redis shard and become a bottleneck.

How to handle it?

1. Replicate hot keys: Store the same value on multiple cache nodes and load balance reads across them.
2. Add a local fallback cache: Keep extremely hot values in-process to avoid pounding Redis.
3. Apply rate limiting: Slow down abusive traffic patterns on specific keys.

## Multi-Tier Caching

Multi-tier caching stacks a small, ultra-fast local cache in front of a larger shared one, so most reads never leave the process. It is the natural composition of the two patterns from "Where to Cache?", in-process caching and external caching, working together.

- L1, in-process (for example Caffeine or a local map). Nanosecond reads with no network hop. Tiny and per-instance.
- L2, external and shared (for example Redis). Bigger, shared across all app servers, and survives instance restarts.
- The database is the implicit final tier and the source of truth.

Read path: check L1, on a miss check L2, on a miss hit the database, then populate L2 and L1 on the way back. Each layer catches what the one above it missed.

This directly softens two problems covered earlier. An extremely hot key served from L1 never touches Redis, so it cannot overwhelm a single shard. And if L2 expires en masse or goes down, L1 absorbs part of the load instead of everything falling straight through to the database.

### The tradeoff: L1 coherence

This is the reason multi-tier caching takes judgment rather than being a free win. Each instance has its own L1, so when data changes there is no clean way to invalidate every instance's copy. If one instance invalidates and another does not know, the second serves stale data. Ways to manage it:

1. Short L1 TTLs: Keep L1 entries short-lived, on the order of seconds, so staleness is bounded. The simplest and most common approach.
2. Pub/sub invalidation: Broadcast invalidation events, for example over Redis pub/sub, so every instance drops its L1 copy. Redis client-side caching with tracking automates this.
3. Only cache stable data in L1: Configuration, feature flags, and reference data, where brief staleness is acceptable.

### When to use it

Use it for read-heavy workloads with a small hot set that is read far more than it is written and tolerates brief staleness. Skip it when data must be strongly consistent across instances, or when the working set is too large to get a meaningful L1 hit ratio, since a low-hit-ratio L1 just adds coherence complexity for little gain.

## Distributed Caching

A single cache node eventually runs out of memory or throughput. To scale past one machine, you spread data across many nodes. That raises two questions: how do you decide which node holds a given key, and what happens when the set of nodes changes.

### Sharding and Consistent Hashing

The naive approach is `node = hash(key) % N`, where N is the number of nodes. It works until N changes. Add or remove a single node and almost every key remaps to a different node, so nearly the entire cache misses at once. That is an avalanche triggered by a routine scaling event.

Consistent hashing solves this. Nodes and keys are both hashed onto the same circular keyspace (a ring), and each key is owned by the next node clockwise. When a node is added or removed, only the keys in its immediate arc move; the rest stay put. Adding one node to a cluster of N reshuffles roughly 1/N of the keys instead of all of them. Real implementations add virtual nodes, mapping each physical node to many points on the ring, so load stays balanced and a departing node's keys spread across many survivors rather than dumping onto one neighbor.

### Replication

Sharding spreads data for capacity; replication copies data for availability and read throughput. A primary handles writes and one or more replicas serve reads. If the primary fails, a replica is promoted. The tradeoff is replication lag: a read served by a replica may return a value slightly behind the primary, so replication trades a little consistency for availability and read scale. This is also a tool for hot keys, as noted earlier, since replicas let you spread the reads on a single hot key across nodes.

### Redis Cluster vs. Sentinel

For Redis specifically, there are two common topologies. Sentinel gives you high availability without sharding: one primary, several replicas, and Sentinel processes that watch the primary and automate failover. The whole dataset must fit on one node. Redis Cluster adds sharding, partitioning the keyspace into 16384 hash slots spread across primaries, each with its own replicas. Reach for Sentinel when the data fits on one node and you only need failover; reach for Cluster when the dataset or write throughput outgrows a single machine.

## Operating a Cache

A cache you cannot observe is a liability. These are the things worth measuring and planning for once a cache is in production.

### Metrics that matter

1. Hit ratio: The single most important signal. It is hits divided by total lookups. A low or falling hit ratio means the cache is doing little useful work and you are paying its cost for nothing. Investigate whether the working set has outgrown memory, the TTLs are too short, or you are caching the wrong things.
2. Eviction rate: How often entries are pushed out to make room. A high eviction rate alongside a falling hit ratio is the classic signal that the cache is undersized for its working set.
3. Latency: Track p99, not just the average. A cache with a great average but a bad tail can still miss latency SLAs.
4. Memory and key count: Watch usage against capacity so you can size ahead of demand rather than reacting to evictions.

### Sizing

Cache the working set, the data actually accessed in a given window, not the entire dataset. If the working set fits in memory the hit ratio stays high; once it exceeds memory, evictions climb and the hit ratio collapses. Size from real access patterns and leave headroom, since a cache running at the edge of its memory evicts aggressively and behaves unpredictably.

### When the cache goes down

Plan for the cache being unavailable, because eventually it will be. The danger is a cascading failure: the cache dies, every request falls through to a database that was never provisioned for full traffic, the database is overwhelmed, and the outage spreads. Defenses:

1. Graceful degradation: On a cache error, fall back to the database and keep serving. The cache is an optimization, not a hard dependency.
2. Protect the database: Combine the fallback with request coalescing, timeouts, and circuit breakers so a cold or dead cache cannot translate directly into a database meltdown.
3. Avoid cold starts: Warm hot keys before a restarted node takes traffic, so it does not come up empty and immediately trigger an avalanche.

### Security and multi-tenancy (reference notes)

- Be deliberate about caching sensitive data. Anything cached lives in memory and possibly on disk (persistence), often with weaker access controls than the database. Decide consciously what is safe to cache.
- Isolate tenants in the key. In a multi-tenant system, put the tenant id in the key (`tenant:42:user:123`) so one tenant can never read another's cached data through a key collision. This is a real data-leak vector, not a theoretical one.
- Watch for cache poisoning at the edge. At the CDN or shared-cache layer, an attacker who influences what gets cached (for example via unkeyed request headers) can serve malicious or wrong content to other users. Cache only on inputs you control and trust.

### What not to cache (reference notes)

Caching adds complexity and a second source of truth, so it is not always worth it. Skip or reconsider caching when:

- Hit ratio would be low. Data accessed once or rarely gains nothing from caching and just wastes memory.
- Data changes constantly. If it is invalidated almost as fast as it is cached, you pay the cost without the benefit.
- Strong consistency is required. If readers cannot tolerate any staleness, a cache fights you rather than helps. Go to the source of truth.

## Caching in System Design Interviews

### When to bring up caching?

Don't jump straight to caching. You need to establish why it's necessary first. Bring up caching when you identify one of these problems.

1. Read-heavy workload - 10 million users, 20 read requests per day for a total of 200 million read requests. Hitting the DB takes 20 - 50 milliseconds per query. We can serve the same content using a cache in under 2 ms
2. Expensive Queries - Computing User's personalized feed is an expensive operation that requires joining multiple tables. Cache the computed feed with a TTL of 60 seconds and serve it from cache
3. Latency Requirements - We need sub-10ms response times for the API. Database queries are taking 30-50ms. We have to cache

### How to introduce caching?

Follow these steps to walk through your caching stategy systematically

1. Identify the bottleneck
- Start by pointing to the specific problem caching will solve. Is it database load? Query latency? Expensive computations? Be specific about what's slow and why.
2. Decide what to cache
- Not everything should be cached. Focus on data that is read frequently, doesn't change often, and is expensive to fetch or compute.
3. Choose your cache architecture
- Pick a caching pattern that matches your consistency requirements. Write-through makes sense when you need strong consistency. Write-behind works for high-volume writes where you can tolerate some risk.
- If you're dealing with static content like images or videos, mention CDN caching. 
- If you have extremely hot keys that get hammered, mention in-process caching as an optimization layer.
4. Set an eviction policy
- Explain how you'll manage cache size. LRU is the safe default answer. TTL is essential for preventing stale data.
5. Address any possible downsides
- Caching introduces complexity. Show you've thought about the trade-offs.
