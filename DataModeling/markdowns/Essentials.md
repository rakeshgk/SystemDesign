# Data Modeling Essentials

Data modeling is the process of defining how your application's data is structured, stored, and related.

## Database Model Options

Before you can design a schema, you need to pick what type of database you're working with. Most of the time, the right answer is a relational database. It's the default unless your requirements clearly signal a specialized model. Stick with PostgreSQL.

### Relational Databases (SQL)

Relational databases organize data into tables with fixed schemas, where rows represent entities and columns represent attributes. They enforce relationships through foreign keys and provide ACID guarantees for transactions.

Example Technologies: PostgreSQL, MySQL, SQLite

### Document Databases

Document databases store data as JSON-like documents with flexible schemas, making them good for rapidly evolving applications where you don't know all your data fields upfront. Your data modeling becomes more about nesting and embedding related information within documents rather than normalizing across tables.

Use document databases over SQL when your schema changes frequently, when you have deeply nested data that would require many joins in SQL, or when different records have vastly different structures. A user profile system where some users have extensive work histories while others have minimal data fits this pattern.

Example Technologies: MongoDB, Firestore and CouchDB

### Key-Value Store

Key-value stores provide simple lookups where you fetch values by exact key match. They're extremely fast but offer limited query capabilities beyond that basic operation.

Use a Key-Value Store over SQL for caching, session storage, feature flags, or any scenario where you only need to look up data by a single identifier. They're also good for high-write scenarios where you need maximum performance and don't need complex queries.

Example Technologies: Redis, Memcached and DynamoDB

> Note: DynamoDB is really a wide-column/document hybrid, not a pure key-value store. You can use it as a simple KV store, but its real power is single-table design with composite keys and secondary indexes (covered under NoSQL modeling below). Don't undersell it as "just a KV store" in an interview.

### Wide Column Databases

Wide-column databases organize data into column families where rows can have different sets of columns. They're optimized for massive write-heavy workloads and time-series data. Data in these tables is grouped by a partition key. For example - All data related to one user lives together. 

Use Wide Column Databases over SQL when you have enormous write volumes, time-series data, or analytics workloads where you primarily append data and run aggregations. Think telemetry, event logging, or IoT sensor data.

Example Technologies: Cassandra and HBase

### Graph Databases

Graph databases store data as nodes and edges, optimizing for traversing relationships between entities. They almost never get used in interviews. Some companies do model social networks and recommendation engines using graph databases.

Example Technologies: Neo4J and Amazon Neptune

## Schema Design Fundamentals

Once you've picked your database type, you need to design a schema that supports your system's requirements.

### Start with the requirements

Everything flows from three key factors that require careful consideration and were likely already determined during the requirements gathering and API design phases.

1. **Data Volume**
   - Determines where your data can physically live.
   - A social media app with millions of users might need data spread across multiple data stores, which drives schema design choices.
   - If user data and post data need to live on separate systems for performance or organizational reasons, they necessarily need distinct schemas with careful consideration of how they reference each other.
2. **Access Patterns**
   - How will your data be queried?
   - This comes naturally from your APIs. Just ask: what queries will I need to support each endpoint?
   - A news feed that loads "recent posts by followed users" suggests you'll want denormalized data or carefully designed indexes. An analytics dashboard that aggregates data across time periods might need different table structures entirely.
3. **Consistency Requirements**
   - Determine how tightly coupled your data can be.
   - Financial transactions need strong consistency (no partial charges), which often means keeping related data in the same database with ACID guarantees.
   - A user's activity feed, for instance, can handle eventual consistency. It's okay if a post's likes appear a few seconds later.

### Entities, Keys and Relationships

Once you've identified your core entities, the next step is to map them into tables (or collections) with clear identifiers and relationships.

For a social media app, you might have `users`, `posts`, `comments`, and `likes`. Each entity needs a primary key to identify individual records. Use system-generated IDs like `user_id` or `post_id` rather than business data like email addresses. System-generated keys stay stable even when business rules change.

Once the entities are defined, you connect them through relationships.

1. **One-to-many (1:N):** a user has many posts, a post has many comments.
2. **Many-to-many (N:M):** users like many posts, posts are liked by many users. These need a join table (e.g. `likes` with `user_id` and `post_id`).
3. **One-to-one (1:1):** less common, and sometimes a sign that two tables should be merged — but it's a legitimate modeling choice when you deliberately want to split hot columns from cold ones, isolate a large blob (like a profile bio or image), or put PII behind a separate security boundary. Treat it as a trade-off, not automatically a smell.

These relationships are enforced through foreign keys in SQL (e.g., `posts.user_id → users.id`) or by application logic in NoSQL. Foreign keys help ensure referential integrity — meaning they prevent orphaned records like a post referencing a user that doesn't exist, or comments pointing to deleted posts. However, they come at a cost because the database has to validate each insert/update. At very large scale, some companies drop them for write performance and enforce integrity at the application level. In an interview, mentioning them shows you understand the trade-off.

### Choosing Column Data Types

The types you pick for your columns are part of the schema design, not an afterthought — the wrong choice causes correctness bugs that are painful to fix once data exists.

- **Money:** never store as a float. Use an integer number of cents (or a fixed-precision `DECIMAL`/`NUMERIC`). Floating point rounding errors on currency are a classic production bug.
- **Primary keys — UUID vs bigint:** auto-incrementing `bigint` keys are compact and index-friendly but leak volume and require a round-trip to the DB to generate. UUIDs (or ULIDs) can be generated client-side and don't leak counts, but they're larger and, if random (UUIDv4), hurt index locality on inserts. ULIDs / UUIDv7 are time-ordered and avoid that penalty — a good default when you need client-generated IDs.
- **Timestamps:** store in UTC, use a timezone-aware type (`TIMESTAMPTZ` in Postgres), and keep display-timezone conversion in the application layer.
- **Enums / status fields:** prefer a constrained set (a native enum, a `CHECK` constraint, or a lookup table) over free-form strings so invalid states can't be written.

### Indexing for Access Patterns

Indexes are data structures that help the database find records quickly without scanning every row. Think of them like the index in a book — instead of reading every page to find "normalization," you look it up in the index and jump directly to page 149. While data modeling in an interview, you'll typically want to call out which columns are indexed and why. A few things worth mentioning to show depth:

- **Composite indexes and column order.** An index on `(user_id, created_at)` efficiently serves "posts by user, most recent first" but does *not* help a query that filters on `created_at` alone. Left-to-right prefix ordering matters — put the equality-filter column first, the range/sort column second.
- **Covering indexes.** If an index contains every column a query needs, the database can answer straight from the index without touching the table (an "index-only scan"). Useful for hot read paths.
- **Selectivity.** Indexes pay off on high-cardinality columns (e.g. `email`). An index on a low-cardinality column like a boolean `is_active` rarely helps.
- **Indexes are not free.** Every index has to be updated on each insert/update/delete and consumes storage. On write-heavy tables, over-indexing is a real cost — index for the reads you actually run, not speculatively.

### Normalization vs Denormalization

Normalization means storing each piece of information in exactly one place. User data lives only in the users table, not duplicated across other tables. This prevents data anomalies where updates happen in one place but not another, leaving your system with inconsistent state.

In system design interviews, start with a clean normalized model and denormalize only when needed. Avoid repeating data in your schema design. Repeating data is wasteful and creates consistency problems that are much harder to solve than the performance problems you're trying to avoid.

There are a few key exceptions where denormalization might make sense:

1. Analytics and reporting systems where you're aggregating data that changes infrequently.
2. Event logs and audit trails where you're capturing a snapshot of data at a point in time.
3. Heavily read-optimized systems like search engines where consistency is less critical than speed.

That said, even if you need denormalized quick access for performance, you can just put a cache in front that has a denormalized representation of the data. Your source of truth stays clean and normalized, but your cache can have pre-computed joins, aggregations, or whatever structure makes reads fast.

**Common denormalization patterns worth naming:**

- **Precomputed counters and aggregates.** Storing `like_count` on a `posts` row instead of `COUNT(*)`-ing the likes table on every read. Fast to read, but now you have to keep the counter in sync.
- **Materialized views.** A database-maintained, physically stored result of a query that you refresh on a schedule or on write. A clean middle ground between recomputing every time and hand-rolled denormalization.
- **Fan-out-on-write vs fan-out-on-read.** This is the core decision behind a news feed. *Fan-out-on-write* pushes each new post into every follower's precomputed feed at write time — reads are cheap, but writes are expensive and celebrity accounts with millions of followers cause write storms. *Fan-out-on-read* assembles the feed by querying followed users at read time — writes are cheap, reads are expensive. Most real systems use a hybrid: fan-out-on-write for normal users, fan-out-on-read for celebrities.

### Concurrency and Optimistic Locking

When two requests read the same row and both write it back, one update can silently overwrite the other (the lost-update problem). The common modeling fix is an **optimistic lock**: add a `version` (or `updated_at`) column, and on update require that the version still matches what you read (`UPDATE ... WHERE id = ? AND version = ?`). If it doesn't match, the write failed because someone else got there first, and the client retries. The mention here is that this is a *schema* decision — you need to design the version column in — even though the locking mechanics themselves live in the database deep-dive.

### Schema Evolution and Migrations

A schema is never done. Staff-level data modeling is as much about how the schema *changes over time* as how it starts — you rarely get to design greenfield and walk away. The hard requirement is that migrations happen with **zero downtime** while old and new versions of the application are running simultaneously (during a rolling deploy).

The key technique is the **expand–contract (parallel change)** pattern, run in phases:

1. **Expand.** Make an additive, backward-compatible change — add the new nullable column or new table. Old code ignores it; new code can start using it. Never rename or drop in this phase.
2. **Migrate / backfill.** Populate the new structure for existing rows. Do this in batches, not one giant transaction, so you don't lock the table or blow up replication lag. Have the application dual-write (write both old and new) while the backfill runs.
3. **Contract.** Once all code reads from the new structure and the backfill is verified complete, drop the old column/table.

Other things worth calling out:

- **Additive changes are safe; destructive changes are not.** Adding a nullable column is cheap. Renaming or dropping a column, or adding a `NOT NULL` column without a default, can break the currently-running old code — split those into expand/contract steps.
- **Online DDL.** On large tables, naive `ALTER TABLE` can lock the table for the duration. Use the database's online/concurrent DDL (e.g. `CREATE INDEX CONCURRENTLY` in Postgres) or a tool like gh-ost / pt-online-schema-change.
- **Migrations are versioned and forward-only.** Treat schema changes like code — checked in, ordered, and reviewed. Prefer a forward "fix" migration over a rollback in production.

### Scaling: Partitioning vs Sharding

These are related but distinct, and it's worth keeping them straight.

**Partitioning** splits one logical table into multiple physical pieces *within a single database*. It's transparent to the application and the database handles routing. Common strategies:

- **Range partitioning** — e.g. by date, so old months can be dropped or archived cheaply. Great for time-series data.
- **List partitioning** — by a discrete value like region.
- **Hash partitioning** — to spread rows evenly when there's no natural range.

Partitioning also comes in two flavors: **horizontal** (splitting rows across partitions, as above) and **vertical** (splitting columns — e.g. moving rarely-accessed or large columns into a separate table, which is really the 1:1 hot/cold split mentioned earlier).

**Sharding** takes it a step further: it spreads data across *multiple separate database machines*. This is what you reach for when the data (or the write load) is too large for one box. The application (or a routing layer) now has to know which shard a piece of data lives on.

The key with sharding is choosing a **shard key** that matches your primary access pattern. If you mostly query "posts by user," shard by `user_id` — this keeps a user's posts on the same machine and avoids expensive cross-shard queries.

Sharding introduces hard problems you should acknowledge:

- **Hot / celebrity shards.** If you shard by `user_id` and one user is wildly more active than everyone else, their shard becomes a bottleneck. May need special handling for those keys.
- **Cross-shard queries and joins** become expensive or impossible — you often have to scatter-gather across shards and merge in the application.
- **Resharding** (adding machines later) is painful if you used naive modulo hashing, because it remaps almost every key. **Consistent hashing** minimizes how much data moves when the shard count changes.
- **Distributed ID generation.** Auto-increment doesn't work across shards. Use a scheme like Snowflake IDs or ULIDs that generate globally-unique, roughly-ordered IDs without a central counter.

### Replication and Its Modeling Impact

Databases are usually replicated — one primary takes writes, one or more replicas serve reads — for availability and read scaling. This is mostly an operational concern, but it leaks into data modeling and API design in one important way: **replication lag**. A read that hits a replica may not yet reflect a write that just succeeded on the primary.

The classic symptom is **read-your-own-writes**: a user posts a comment, the page reloads from a replica, and their comment isn't there yet. You handle this at design time — route reads that must be fresh to the primary, pin a user to the primary for a short window after they write, or serve the just-written value from the client/cache. Worth a sentence in any design that offloads reads to replicas.

### Multi-Tenancy

If your system serves multiple customers/organizations, how you isolate their data is a foundational modeling decision that ripples into keys, indexes, and sharding. Three common models, from least to most isolated:

1. **Shared table with a `tenant_id` column.** All tenants share tables; every row carries a `tenant_id` and every query filters by it. Cheapest and simplest to operate, scales to many small tenants — but a missing `tenant_id` filter is a serious data-leak bug, and `tenant_id` naturally becomes part of your composite indexes and shard key.
2. **Schema-per-tenant.** Each tenant gets its own set of tables within a shared database. Stronger isolation, easier per-tenant backup/export, but harder to run cross-tenant queries and doesn't scale to huge tenant counts.
3. **Database-per-tenant.** Full isolation, easiest to meet strict compliance or per-tenant performance guarantees, but the most operationally expensive (migrations must run across every database).

The usual answer is shared-table-with-`tenant_id` for many small tenants, escalating to dedicated schemas or databases for large or compliance-sensitive customers.

### Data Lifecycle, PII, and Compliance

Real systems have to answer "how long do we keep this, and how do we get rid of it," and that shapes the schema.

- **Soft delete vs hard delete.** A soft delete (`deleted_at` / `is_deleted` flag) keeps the row for audit/recovery and preserves referential integrity, but every query now has to filter it out, and it doesn't satisfy "delete my data" requests. A hard delete actually removes the row. Many systems soft-delete first, then hard-delete on a retention schedule.
- **Retention and TTL.** Decide up front how long data lives. Some stores (DynamoDB, Cassandra, Redis) support native TTL that expires rows automatically — good for sessions, logs, and ephemeral data.
- **PII handling.** Personally identifiable information deserves deliberate modeling — isolate it (often in its own table behind a security boundary), encrypt sensitive columns at rest, and know which columns are PII so you can honor deletion/export requests.
- **Compliance (GDPR / CCPA).** "Right to be forgotten" means you must be able to actually delete a user's data, which is at odds with denormalized copies scattered across the system and with soft-delete-only schemes. Design for it rather than bolting it on later.

### Dimensional Modeling (OLTP vs OLAP)

The schema you'd design so far is optimized for **OLTP** — Online Transaction Processing: many small, concurrent reads and writes, normalized, serving the live application. **OLAP** — Online Analytical Processing — is the opposite workload: fewer, huge, read-heavy aggregation queries for analytics and reporting. You don't run heavy analytics against your normalized production database; you model separately for it.

The canonical OLAP model is the **star schema**: a central **fact table** (one row per event/measurement — e.g. a sale, with quantities and amounts) surrounded by **dimension tables** (the descriptive context — customer, product, date, store). It's deliberately denormalized so analysts can slice and aggregate with simple joins. A **snowflake schema** is the same idea with the dimensions further normalized.

Data typically flows from the OLTP system into the OLAP warehouse via **CDC (Change Data Capture)** or batch ETL. Related is **temporal / audit modeling** — capturing history rather than just current state:

- **Slowly Changing Dimensions (SCD)** — techniques for tracking how a dimension's attributes change over time (e.g. a customer's address history) instead of overwriting them.
- **Event sourcing** — storing the immutable log of events that happened rather than only the current derived state, so you can reconstruct any past state.

## Out of Scope: Covered in the Database Deep-Dive

The topics below are deliberately *not* covered here. They're about how the database engine works and operates, not how you model your data — so they belong in the database deep-dive. They're listed so a reader knows they were scoped out, not forgotten. Where a topic leaves a modeling footprint, that footprint is already captured in the sections above.

- **Consistency mechanics** — isolation levels (read committed, repeatable read, serializable), transaction boundaries, idempotency, the dual-write / outbox problem, and the saga pattern for cross-service consistency. (Modeling footprint: choosing strong vs eventual consistency, covered under *Start with the requirements*.)
- **Locking and concurrency internals** — optimistic vs pessimistic locking mechanics, row vs table locks, deadlock detection, MVCC. (Modeling footprint: the version column, covered under *Concurrency and Optimistic Locking*.)
- **Index internals** — B-tree vs hash vs GIN/GiST/BRIN, how each is physically structured, partial indexes, index maintenance. (Modeling footprint: which columns to index and composite/covering choices, covered under *Indexing for Access Patterns*.)
- **Partitioning and sharding mechanics** — the routing layer, rebalancing operations, query planning across partitions. (Modeling footprint: choosing the partition/shard key, covered under *Scaling: Partitioning vs Sharding*.)
- **Replication mechanics** — sync vs async replication, quorum reads/writes, failover and leader election, multi-primary setups. (Modeling footprint: replication lag and read-your-own-writes, covered under *Replication and Its Modeling Impact*.)
- **Caching internals** — eviction policies, write-through vs write-back, cache invalidation strategies. (Modeling footprint: keeping a denormalized cache over a normalized source of truth, covered under *Normalization vs Denormalization*.)
- **Search and vector stores** — full-text search engines, inverted indexes, embedding/vector databases for similarity search.
- **Backups, disaster recovery, and observability** — snapshotting, point-in-time recovery, RPO/RTO, query performance monitoring.

## How to approach Data Modeling in System Design interviews?

Data modeling is a core part of system design interviews, but it's not the focus. Your goal is to show that you can design a reasonable schema that supports your system's requirements, then move on.

Start by outlining your core entities early in the interview. Then, when introducing a database component during the high-level design:

1. Determine the type of database you'll use.
2. List the columns needed to fulfill the functional requirements for each entity, and pick sensible data types.
3. Specify primary and foreign keys for each relationship.
4. Determine which columns need indexes (and call out composite/covering index choices where they matter).
5. Determine whether you need to denormalize for performance (and, if so, which pattern — counters, materialized views, fan-out).
6. Consider whether sharding is necessary. If yes, choose a shard key that matches your main access pattern.

The deeper topics — schema evolution and zero-downtime migrations, concurrency control, replication lag, multi-tenancy, data lifecycle/PII, and OLTP-vs-OLAP modeling — are what separate a Staff-level answer. You won't get to all of them in every interview, but reaching for the one or two that matter for the system at hand is a strong signal.
