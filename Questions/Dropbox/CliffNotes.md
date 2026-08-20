# DropBox — System Design Cliff Notes

## Data Store Choice
Store file metadata in a database and the file bytes in S3. They solve different problems: S3 is cheap, durable (11 nines), and infinitely scalable object storage, but it is bad at queries — you can't efficiently ask "give me all files in this folder owned by user X, sorted by modified date." A database is where you answer those queries fast, enforce structure, and store the pointer (S3 key) to the actual bytes. So "metadata in a DB, bytes in S3" is the standard pattern. The real question is which database.

### Why SQL Is a Reasonable Default
1. Relationships everywhere — files belong to folders, folders nest, files are shared with users, files have versions. These are joins, and SQL does them well.
2. Transactions — creating the `pending` metadata row, flipping it to `committed` on the S3 event, and updating a user's storage quota benefit from ACID guarantees. Getting quota accounting wrong is a real bug.
3. Strong consistency — after a sync client uploads, other devices should see the new version immediately. SQL gives you read-your-writes without extra work.
4. The schema is well-known and stable, so a rigid schema is an asset, not a constraint.

### The Scale Caveat
The caveat is scale. Metadata rows are small (a few hundred bytes each), but the row count is enormous — a single Postgres instance won't hold hundreds of billions of files. The answer isn't "switch to NoSQL"; it's:
- Shard the SQL DB, typically by `userId`, so all of one user's file metadata lives on one shard. This keeps folder listings and per-user queries on a single shard — no cross-shard joins for the common path.
- Add read replicas for the read-heavy listing traffic.
- Use a managed, horizontally-scalable relational store (Aurora, Spanner, CockroachDB, Vitess) to get SQL semantics without hand-rolling sharding.

### When to Reach for NoSQL Instead
If the access pattern is overwhelmingly key-based lookups (`fileId` → metadata, `userId` → file list) with few ad-hoc joins, a wide-column store like DynamoDB or Cassandra scales horizontally more painlessly and gives predictable latency. The cost is that you denormalize, manage consistency yourself, and cross-entity transactions (quota, sharing) get harder.

### Bottom Line
SQL for metadata + S3 for blobs is a solid choice — just qualify it. At DropBox scale the SQL DB must be sharded by user, and a wide-column NoSQL store is a valid trade if the workload is purely key-based lookups.

## PreSigned URLs
To avoid uploading the same file twice (once from the client to the App Server and a second time from the App Server to Blob Storage) we can use presigned URLs to allow our clients to directly upload or download their files.

You can use presigned URLs to grant time-limited access to objects in Amazon S3 without updating your bucket policy. A presigned URL can be entered in a browser or used by a program to download an object. The credentials used by the presigned URL are those of the AWS Identity and Access Management (IAM) principal who generated the URL.

You can also use presigned URLs to allow someone to upload a specific object to your Amazon S3 bucket. This allows an upload without requiring another party to have AWS security credentials or permissions. If an object with the same key already exists in the bucket as specified in the presigned URL, Amazon S3 replaces the existing object with the uploaded object.

## S3 Events
Amazon S3 automatically generates event notifications whenever an upload successfully completes. You can capture these completion events using either S3 Event Notifications or Amazon EventBridge. 

S3 fires specific events under the `s3:ObjectCreated:*` family depending on how the file was uploaded. For files being deleted, events come under `s3:ObjectRemoved:*`.

S3 can deliver upload completion events to 4 main targets:

1. Amazon EventBridge
2. AWS Lambda
3. Amazon SQS
4. Amazon SNS

You can use either FileSystemWatcher (Windows) or FSEvents (macOS) to monitor file system events. 

## Client-Side Change Detection (FSEvents / Sync)
FSEvents is macOS's kernel-level API for getting notified when the contents of a directory tree change. For a DropBox-style sync client, it's how the local daemon knows "something changed on disk, I need to sync it."

**FSEvents is directory-scoped, not file-scoped.** The OS tells you which directories had changes underneath them — not exactly which file changed or what kind of change it was. The kernel coalesces events per-directory. Your client then has to scan that directory (e.g. via `stat`) and compare file state against its local metadata DB to figure out what actually changed. This is deliberate: watching a huge tree stays cheap because the OS isn't tracking every individual file.

### Why the Client Treats Events as a Hint, Not Truth
- **Coalescing** — multiple changes to the same path within the latency window collapse into one event. A quick create-then-delete may just show up as "this directory changed."
- **No "what changed" detail** — even with file events, you don't get old-vs-new content; a rename may appear as a remove + create.

So the standard pattern is: FSEvents tells you *where* to look, then the client `stat`s the affected paths and compares against its local metadata DB (path, size, mtime, and ideally a content hash) to decide what actually changed. The hash comparison is also what enables block-level/delta sync and avoids re-uploading unchanged content.

### End-to-End Sync Flow
1. FSEvents fires → client learns a watched directory/file changed.
2. Client `stat`s + hashes the path, diffs against its local metadata DB.
3. If genuinely changed, client requests a presigned URL and does the multipart upload (see below).
4. On restart, `sinceWhen` + the persisted event ID replays anything missed offline.

## Updating File Metadata After Upload
How do we update the FileMetaData after upload to S3 is complete? The key is to create the PostgreSQL metadata row before giving the client the pre-signed URL, then encode a stable identifier into the S3 Object Key. S3 would then include the key as part of its event notification, which our Lambda can parse and use to locate the correct database row.

![File Meta Data Update Flow Chart](../assets/FileMetaDataUpdateFlowChart.png)

```
File Metadata table will contain the file ID {Twitter Snowflake or UUIDv7 based ID}
Object Key = uploads/<ID>/<file-name>
Generate a pre-signed URL that permits only PUT operation for that exact key
Return this to the client
```

## Chunking & Multipart Upload
Chunking can help us handle large files efficiently by splitting them into small, manageable pieces. S3 multipart upload splits one object into numbered parts, uploads those parts independently, and then asks S3 to assemble them into the final object. The client performs chunking and uploads the parts. S3 is only responsible for assembling the parts.

1. The backend app server is going to talk to S3 through the SDK to initiate a new multipart file upload by calling `CreateMultipartUpload`
2. S3 returns an opaque upload ID. This ID is not the same as the file upload ID that was generated by our application
3. Our application is also responsible for generating pre-signed URLs for each of the chunks that the client creates from the original file
4. Clients will chunk the file. In JavaScript they can do `file.slice` and in Java they can do `RandomAccessFile`. These methods do not copy the entire file into memory. They read and upload the selected range as needed. Chunking also allows us to upload parts in parallel if there is available bandwidth
5. After every part succeeds, the client sends your backend the collected part numbers and ETags
6. Once all the parts are uploaded, the application can call `CompleteMultipartUpload`. S3 validates the uploaded parts and atomically creates the final object in part-number order.
7. Before we call `CompleteMultipartUpload`, it is better to also check that the parts are not corrupted. Every part uploaded receives an `ETag`. We send the `ETag` to the application for persistence in the DB after marking the chunk upload as complete. The application calls `ListParts` to get all the parts that were uploaded to S3. The response contains the `ETag`, which the application server can use to verify.

### What Is an ETag?
An ETag ("entity tag") is a version fingerprint a server assigns to a resource — an opaque string the client compares for equality (originally an HTTP header used for caching via `If-None-Match` and concurrency via `If-Match`). In S3 its meaning depends on how the object was uploaded:
- **Single PUT** → the ETag is the **MD5 of the object's bytes**, so you can recompute it locally to confirm the upload wasn't corrupted.
- **Multipart upload** → it is **not** a plain MD5 of the whole file. S3 takes the MD5 of each part, concatenates the binary digests, hashes that, and appends `-<part count>` (e.g. `...-42`). So you can't verify a multipart object by MD5-ing the reassembled file — instead verify **each part's** ETag as it uploads (which is what step 7 does). `CompleteMultipartUpload` also *requires* the list of part numbers + ETags so S3 knows which parts, in which order, form the final object.
- **Caveat:** the ETag-equals-MD5 assumption breaks for SSE-KMS / SSE-C encrypted objects. For integrity, AWS now offers explicit checksum options (CRC32, SHA-256) that are cleaner than relying on the ETag.

## Dedup & Block Storage
There are two separate reasons to chunk a file, and they compose differently depending on the storage architecture:
- **Chunking for transfer** — parallel, resumable, retryable uploads. Boundaries are arbitrary byte offsets driven by S3 multipart limits (part ≥ 5 MB, ≤ 10,000 parts) and network throughput.
- **Chunking for dedup/delta sync** — so that editing one paragraph in a 2 GB file only re-uploads the affected block, not the whole file. Here the boundary decision is the whole game.

### The Insertion Problem
Fixed-size chunking (cut every N bytes) is simple but breaks on edits: insert one byte at the front and every downstream boundary shifts, so every chunk hash changes and you dedup nothing. **Content-defined chunking (CDC)** fixes this — slide a rolling hash over the bytes and place a boundary wherever the hash matches a chosen pattern, so cut points depend on content. An insert only disturbs the chunk around it; downstream boundaries re-sync to the same content-defined points. Identical content anywhere (same file, different file, different user) produces identical chunks → identical hashes → stored once. (Interview altitude: name CDC and the insertion problem it solves; the rolling-hash math is optional bonus depth.)

### How the Two Layers Compose
**Model A — one S3 object per file.** The file is reassembled from multipart parts. To also get dedup here you must pack multiple small CDC blocks into one ≥ 5 MB transfer part. But S3 stores the whole object, so you've deduped the *transfer*, not the *storage* — which is why serious dedup systems avoid this model.

**Model B — content-addressable block store (what DropBox does).** Don't store one object per file. Store each unique block as its own object keyed by its hash (`blocks/<sha256>`), and make the file's metadata row an **ordered list of block hashes** (a manifest / "file recipe"). Now:
- Dedup is natural at rest — before uploading a block, ask if the hash already exists; if so, skip the upload and store nothing new.
- The **block is the transfer unit**, so there's no "packing CDC blocks into parts." DropBox uses fixed **4 MB** blocks precisely because 4 MB is both a reasonable dedup granularity and a fine single-PUT size. Multipart only re-enters if an individual block is large.
- **Reassembly moves from S3 to the client** — on download, read the manifest, fetch blocks by hash in parallel, and concatenate.

| | Model A (single object) | Model B (block store) |
|---|---|---|
| Storage unit | 1 S3 object per file | 1 S3 object per unique block |
| Transfer unit | multipart part (≥ 5 MB) | one block |
| Dedup benefit | transfer only | transfer **and** storage |
| CDC ↔ transfer | pack many CDC blocks per part | block ≈ transfer unit, no packing |
| Reassembly | S3 `CompleteMultipartUpload` | client concatenates from manifest |

### The Trade-off Worth Naming
DropBox uses *fixed* 4 MB blocks, not CDC — a deliberate simplification: fixed blocks reassemble trivially and are a clean transfer unit, but they pay the insertion-problem cost. CDC fixes the insertion problem but yields variable-size blocks that are messier as transfer units. Picking fixed blocks is a valid answer as long as you can name that trade. The key architectural idea to articulate: **file = an ordered list of content-addressed blocks**, which flips the block into being the unit of both dedup and transfer.

## Downloads
For downloads, we don't need to keep the data chunks. For very large files, S3 and HTTP natively support Range requests, which let the client download different byte ranges in parallel or resume an interrupted download without starting over. The client doesn't need to know anything about the original chunk boundaries.
