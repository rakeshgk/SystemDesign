When it comes to Core Entities, we can state that URL is a core entity and within the URL there are two types
- Original URL
- Short URL

When we receive a POST request to create a short URL, we can talk about de-duplication. Check if this exact long URL was already shortened and return the existing short code. Most URL shorteners allow multiple short codes for the same long URL since different users may want separate expiration dates, independent analytics, or different custom aliases.

Important HTTP Status codes worth remembering
- 301 Permanent Redirect : Requested resource has been definitively and permanently relocated to a new URL
- 302 Found : Requested resource has been temporarily relocated to a new URL
- 410 Gone : Target resource has been permanently deleted

For Url Shortener, it is better to return 302 Found as this allows subsequent requests to come to our server. This also helps with click analytics. 

Base 62 Encoding
```
Lets convert the number 12345 to base62 encoding
1. First division: 12,345 ÷ 62 = 199 with a remainder of 17 (maps to H)
2. Second division: 199 ÷ 62 = 3 with a remainder of 13 (maps to N)
3. Third division: 3 ÷ 62 = 0 with a remainder of 3 (maps to 3)

Reading the remainders from last to first gives 3NH. This is the base 62 encoding for the number 12345. 

Lets convert 3NH back to base 10
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

We go with Base62 representation for this question as it is a compact representation of numbers that uses 62 characters. We exclude `+` and `/`. The slash is a path separator in URLs and the plus sign can be interpreted as a space in query strings.

When it comes to unique ID generation we can suggest either the one where a Database is involved or we can go with Twitter Snowflake. In the Twitter Snowflake model, we use 41 bits for the timestamp, 5 bits for the data center ID, 5 bits for the machine ID and 12 bits for the sequence number. Along with avoiding a DB round trip time, we can also ensure that the IDs generated are not sequential in nature if we go with Twitter snowflake. 

The maximum number of characters needed to encode a 32 bit integer in base 62 representation is 6 characters. The maximum number of characters needed to encode a 64 bit integer in base 62 representation is 11 characters. With 6 characters in base 62 representation, we can encode upto 62⁶ or 56 billion URLs. With 7 characters in base 62 representation, we can encode upto 62⁷ or 3.5 trillion URLs. 