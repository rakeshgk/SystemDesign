# Data Modeling Essentials 

Data modeling is the process of defining how your application’s data is structured, stored, and related. 

## Database Model Options 

Before you can design a schema, you need to pick what type of database you're working with. Most of the time, the right answer is a relational database. It's the default unless your requirements clearly signal a specialized model. Stick with PostgreSQL. 

### Relational Databases (SQL)

Relational databases organize data into tables with fixed schemas, where rows represent entities and columns represent attributes. They enforce relationships through foreign keys and provide ACID guarantees for transactions.

Example Technologies : PostgreSQL, MySQL, SQLite

### Document Databases 

Document databases store data as JSON-like documents with flexible schemas, making them good for rapidly evolving applications where you don't know all your data fields upfront. Your data modeling becomes more about nesting and embedding related information within documents rather than normalizing across tables.

Use document databases over SQL when your schema changes frequently, when you have deeply nested data that would require many joins in SQL, or when different records have vastly different structures. A user profile system where some users have extensive work histories while others have minimal data fits this pattern.

Example Technologies : MongoDB, Firestore and CouchDB

### Key-Value Store

Key-value stores provide simple lookups where you fetch values by exact key match. They're extremely fast but offer limited query capabilities beyond that basic operation.

Use Key-Value Store over SQL for caching, session storage, feature flags, or any scenario where you only need to look up data by a single identifier. They're also good for high-write scenarios where you need maximum performance and don't need complex queries.

Example Technologies : Redis, Memcached and DynamoDB

### Wide Column Databases

Wide-column databases organize data into column families where rows can have different sets of columns. They're optimized for massive write-heavy workloads and time-series data.

Use Wide Column Databases over SQL when you have enormous write volumes, time-series data, or analytics workloads where you primarily append data and run aggregations. Think telemetry, event logging, or IoT sensor data.

Example Technologies : Cassandra and HBase

### Graph Databases

Graph databases store data as nodes and edges, optimizing for traversing relationships between entities. Almost always never gets used in interviews. Some companies do model social networks and recommendation engines using Graph Databases. 

Example Technologies : Neo4J and Amazon Neptune

## Schema Design Fundamentals

Once you've picked your database type, you need to design a schema that supports your system's requirements.

### Start with the requirements

Everything flows from three key factors that require careful consideration and were likely already determined during the requirements gathering and api design phases.

1. Data Volume
- Determines where your data can physically live
- A social media app with millions of users might need data spread across multiple data stores, which drives schema design choices.
- If user data and post data need to live on separate systems for performance or organizational reasons, they necessarily need distinct schemas with careful consideration of how they reference each other.
2. Access Patterns
- How will your data be queried?
- This comes naturally from your APIs. Just ask what queries will I need to support each endpoint?
- A news feed that loads "recent posts by followed users" suggests you'll want denormalized data or carefully designed indexes. An analytics dashboard that aggregates data across time periods might need different table structures entirely. 
3. Consistency Requirements
- Determine how tightly coupled your data can be
- Financial Transactions need strong consistency (no partial charges) which often means keeping related data in the same Database with ACID guarantees
- User's activity feed for instance can handle eventual consistency. It is okay if a post's likes appears a few seconds later

### Entities, Keys and Relationships

Once you’ve identified your core entities, the next step is to map them into tables (or collections) with clear identifiers and relationships.

For a social media app, you might have `users`, `posts`, `comments`, and `likes`. Each entity needs a primary key to identify individual records. Use system-generated IDs like `user_id` or `post_id` rather than business data like email addresses. System-generated keys stay stable even when business rules change.

Once the entities are defined, you connect them through relationships.

1. One-to-many (1:N): a user has many posts, a post has many comments.
2. Many-to-many (N:M): users like many posts, posts are liked by many users.
3. One-to-one (1:1): rare in practice, often a sign that two tables should just be merged.

These relationships are enforced through foreign keys in SQL (e.g., `posts.user_id → users.id`) or by application logic in NoSQL. Foreign keys help ensure referential integrity - meaning they prevent orphaned records like a post referencing a user that doesn't exist, or comments pointing to deleted posts. However, they come at a cost because the database has to validate each insert/update. At very large scale, some companies drop them for write performance and enforce integrity at the application level. In an interview, mentioning them shows you understand the trade-off.

### Indexing for Access Patterns

Indexes are data structures that help the database find records quickly without scanning every row. Think of them like the index in a book - instead of reading every page to find "normalization," you look it up in the index and jump directly to page 149. While data modeling in an interview, you'll typically want to callout which columns are indexed and why.

### Normalization vs Denormalization

Normalization means storing each piece of information in exactly one place. User data lives only in the users table, not duplicated across other tables. This prevents data anomalies where updates happen in one place but not another, leaving your system with inconsistent state.

In system design interviews, start with a clean normalized model and denormalize only when needed. Avoid repeating data in your schema design. Repeating data is wasteful and creates consistency problems that are much harder to solve than the performance problems you're trying to avoid.

There are a few key exceptions where denormalization might make sense:

1. Analytics and reporting systems where you're aggregating data that changes infrequently
2. Event logs and audit trails where you're capturing a snapshot of data at a point in time
3. Heavily read-optimized systems like search engines where consistency is less critical than speed

That said, even if you need denormalized quick access for performance, you can just put a cache in front that has a denormalized representation of the data. Your source of truth stays clean and normalized, but your cache can have pre-computed joins, aggregations, or whatever structure makes reads fast.

### Scaling and Sharding 

When your data gets too large for a single database, you need to shard it across multiple machines. The key is choosing a partition strategy that keeps related data together.

Shard by the primary access pattern. If you mostly query "posts by user," shard by user_id. This keeps a user's posts on the same database, avoiding expensive cross-shard queries.

## How to approach Data Modeling in System Design interviews?

Data modeling is a core part of system design interviews, but it's not the focus. Your goal is to show that you can design a reasonable schema that supports your system's requirements, then move on.

Start by outlining your core entities early in the interview. Then, when introducing a database component during the high-level design:

1. Determine the type of database you'll use
2. List the columns needed to fulfill the functional requirements for each entity
3. Specify primary and foreign keys for each relationship
4. Determine which columns need indexes (if any)
5. Determine whether you need to denormalize for performance
6. Consider whether sharding is necessary. If yes, choose a shard key that matches your main access pattern.
