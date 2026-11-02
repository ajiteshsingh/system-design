# URL Shortener

System design notes for a URL shortener.

## Requirements

## Design

## Hash Function

Two approaches for generating the short code.

### Approach 1: SHA-256 + Base62

Hash the long URL with SHA-256, then encode the digest in base62 (`[0-9a-zA-Z]`) and take the first N characters (e.g. 7) as the short code.

- **Pros:** Stateless, no coordination needed. The same URL always gives the same code, so duplicates are easy to detect.
- **Cons:** Truncating the digest can cause collisions, so you need a uniqueness check and a retry (e.g. rehash with a salt) on conflict. Every write requires a DB lookup.

### Approach 2: Unique Counter in Redis + Base62

Use an atomic Redis counter (`INCR`) to get a unique ID for each new URL, then encode that ID in base62 to get the short code.

This works because Redis executes commands on a single thread. `INCR` is therefore atomic: concurrent clients are handled one at a time, so no two requests can ever receive the same ID.

The write service doesn't have to fetch just one counter at a time. It can request a batch (e.g. 1000 with `INCRBY 1000`) and hand out IDs from that range locally until it runs out, then request another batch. This cuts the number of Redis calls by roughly 1000x and means a short Redis outage doesn't block writes until the batch is used up.

- **Pros:** No collisions by construction. Codes are short and predictable in length (7 base62 chars covers ~3.5 trillion IDs).
- **Cons:** Redis is a central dependency and a possible bottleneck or single point of failure. It needs persistence and replication, or range allocation per app server to scale. Sequential codes are guessable unless the ID is shuffled or obfuscated, so a hacker can brute force all the URLs by iterating through the codes.

## Trade-offs
