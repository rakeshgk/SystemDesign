# URL Shortener — Design Notes

## Core Entities

When it comes to Core Entities, we can state that the URL is a core entity, and within it there are two types:
- Original URL / Long URL
- Short URL

## Creating a Short URL & De-duplication

When we receive a POST request to create a short URL, we can talk about de-duplication: check whether this exact long URL was already shortened and return the existing short code. That said, most URL shorteners allow multiple short codes for the same long URL, since different users may want separate expiration dates, independent analytics, or different custom aliases.

## Redirects & HTTP Status Codes

Important HTTP status codes worth remembering:
- 301 Moved Permanently: the requested resource has been definitively and permanently relocated to a new URL
- 302 Found: the requested resource has been temporarily relocated to a new URL
- 410 Gone: the target resource has been permanently deleted

For a URL shortener, it is better to return 302 Found, as this allows subsequent requests to keep coming to our server. This also helps with click analytics. The trade-off is that a 302 is not cached by browsers or intermediaries, so every click incurs a round trip to our server, whereas a 301 lets clients cache the redirect and reduces load at the cost of losing per-click visibility.

## Base62 Encoding
```
Let's convert the number 12345 to base62 encoding
1. First division: 12,345 ÷ 62 = 199 with a remainder of 17 (17 maps to H)
2. Second division: 199 ÷ 62 = 3 with a remainder of 13 (13 maps to N)
3. Third division: 3 ÷ 62 = 0 with a remainder of 3 (3 maps to 3)

Reading the remainders from last to first gives 3NH. This is the base62 encoding for the number 12345.

Let's convert 3NH back to base 10
1. Find the decimal values of the characters using the standard Base62 index (0-9 = 0-9, A-Z = 10-35, a-z = 36-61):
- 3 = value 3
- N = value 23 (since A=10, B=11... N=23)
- H = value 17 (since A=10, B=11... H=17)

Now, multiply each value by its positional weight and add them together:
- Position 2 (Left): 3 × 62² = 3 × 3,844 = 11,532
- Position 1 (Middle): 23 × 62¹ = 23 × 62 = 1,426
- Position 0 (Right): 17 × 62⁰ = 17 × 1 = 17
Total = 11,532 + 1,426 + 17 = 12,345
```

We go with base62 representation for this question because it is a compact way to represent numbers using 62 characters. Compared to base64, we exclude `+` and `/`: the slash is a path separator in URLs, and the plus sign can be interpreted as a space in query strings.

## Unique ID Generation

When it comes to unique ID generation, we can suggest either an approach that involves a database (for example, an auto-incrementing counter or a range/ticket server that hands out blocks of IDs) or Twitter Snowflake. In the Twitter Snowflake model, we use 41 bits for the timestamp, 5 bits for the data center ID, 5 bits for the machine ID, and 12 bits for the sequence number. Besides avoiding a DB round-trip, Snowflake also ensures that the generated IDs are not easily guessable or strictly sequential, which makes short codes harder to enumerate.

### Why Avoid Purely Sequential IDs?
- **Enumeration / privacy:** sequential IDs produce sequential base62 codes, so an attacker can just increment the code and scrape every URL ever shortened — bad for "unlisted" links (private docs, payment links) meant to be shared, not discovered.
- **Information leakage:** because codes are monotonic, `code(today) − code(yesterday)` reveals how many URLs are created per day, i.e. our growth rate.
- **Write hotspotting:** a monotonically increasing key makes every insert land at the same end of the index (or the same shard), concentrating write load on one node instead of spreading it. A shortener only does point lookups by code, so it gets this downside with none of the range-scan upside.
- **Counter contention:** a single global auto-increment counter serializes every write. The range/ticket-server pattern (each app server grabs a block of IDs and serves them locally) relieves this.

Trade-off: Snowflake fixes hotspotting and contention (generated locally, timestamp-prefixed) but is only *weakly* non-enumerable — the timestamp bits stay monotonic, so codes remain roughly time-ordered. If unguessability is a hard requirement, use random IDs with a collision check, or run a dense internal ID through a keyed permutation so the stored code is scrambled.

## Short Code Length & Capacity

The maximum number of characters needed to encode a 32-bit integer in base62 representation is 6 characters, and for a 64-bit integer it is 11 characters. With 6 characters, we can encode up to 62⁶ ≈ 56 billion URLs; with 7 characters, we can encode up to 62⁷ ≈ 3.5 trillion URLs.
