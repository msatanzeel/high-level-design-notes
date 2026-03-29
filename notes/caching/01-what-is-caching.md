# What is Caching?

## Why This Matters
Every time you read from a database, make an API call, or compute something expensive, you're spending time and resources. Caching is the most effective way to reduce latency and scale systems—it's the difference between a 5ms response and a 500ms response.

---

## Core Concept

**Caching** = Storing a copy of frequently accessed data in a faster storage layer (closer to the application) so you don't have to fetch it from the slower source every time.

**The fundamental trade-off**: Speed vs Freshness
- Faster access = Less fresh data (might be stale)
- Always fresh data = Slower access (hit the source every time)

---

## How It Works - High Level

Think of cache as a **high-speed short-term memory** sitting between your application and your data source:

```
Without Cache:
App → Database (100ms latency)
App → Database (100ms latency)
App → Database (100ms latency)

With Cache:
App → Cache (2ms) → Cache Miss → Database (100ms) → Store in Cache
App → Cache (2ms) → Cache Hit ✓
App → Cache (2ms) → Cache Hit ✓
```

**Key Terms**:
- **Cache Hit**: Data found in cache (fast!)
- **Cache Miss**: Data NOT in cache, must fetch from source (slower)
- **Hit Ratio**: % of requests served from cache (higher is better)
- **TTL (Time To Live)**: How long data stays in cache before expiring

---

## Visual Flow

![Caching Flow](./diagrams/basic-caching-flow.svg)

**Typical Request Flow**:
1. Application receives request
2. Check cache first (in-memory, Redis, etc.)
3. If **cache hit**: Return data immediately (2-5ms)
4. If **cache miss**:
   - Fetch from database/API (50-500ms)
   - Store in cache for next time
   - Return data to user

---

## Real-World Analogy

**Your brain = cache**
- You memorize your friend's phone number (cache)
- You don't look it up in your contacts every time (database)
- But if you haven't called them in 6 months, you forget it (eviction)
- You have limited memory capacity (cache size limit)

---

## Where Caching Happens (Everywhere!)

| Layer | Example | Speed | Typical TTL |
|-------|---------|-------|-------------|
| **Browser** | Static assets, API responses | ~0ms (local) | Days/Weeks |
| **CDN** | Images, videos, HTML | 10-50ms | Hours/Days |
| **Application** | In-memory cache (local) | 1-5ms | Minutes/Hours |
| **Distributed Cache** | Redis, Memcached | 2-10ms | Minutes/Hours |
| **Database** | Query result cache, buffer pool | 5-20ms | Dynamic |
| **CPU** | L1/L2/L3 cache | Nanoseconds | N/A |

---

## Key Takeaways

- ✅ Cache is a **fast, temporary copy** of frequently accessed data
- ✅ Always sits **between app and data source**
- ✅ Trade-off is **speed vs freshness** (you can't have both perfectly)
- ✅ Measured by **hit ratio** (aim for >80% in most systems)
- ✅ Exists at **every layer** of modern systems

---

## When to Use Caching

**Use when**:
- ✅ Data is **read frequently** (high read:write ratio, e.g., 10:1 or higher)
- ✅ Data is **expensive to compute or fetch** (complex queries, external APIs)
- ✅ You can **tolerate some staleness** (eventually consistent is okay)
- ✅ Same data requested by **multiple users** (session data, product catalogs)

**Avoid when**:
- ❌ Data **changes constantly** (real-time stock prices, live sports scores)
- ❌ Data is **unique per request** (no reuse benefit)
- ❌ Consistency is **critical** (financial transactions, inventory counts)
- ❌ Memory cost > **compute/fetch cost** (huge blobs, rarely accessed data)

---

## Common Use Cases

1. **API Response Caching** - Store expensive API call results (weather data, exchange rates)
2. **Database Query Results** - Cache frequent SELECT queries
3. **Computed Results** - Store expensive calculations (recommendation scores, analytics)
4. **Session Data** - User authentication tokens, preferences
5. **Static Assets** - Images, CSS, JS files (CDN caching)
6. **Configuration** - Feature flags, app settings

---

## FAQ / Common Misconceptions

### ❓ "Is caching only for database results?"

**No! You can cache ANY expensive operation**, not just database queries.

**What can be cached:**

| Data Source | Example | Why Cache? | Typical Latency |
|-------------|---------|------------|-----------------|
| **Database** | User profile query | Reduce DB load | 50-500ms |
| **External APIs** | Weather API, Payment gateway | Avoid rate limits, reduce cost | 100-2000ms |
| **Microservice Calls** | Auth service, Product service | Reduce network hops | 20-200ms |
| **Expensive Computations** | Recommendation algorithm, ML inference | Avoid re-computing | 100-5000ms |
| **File System** | Config files, templates | Avoid disk I/O | 10-100ms |
| **DNS Lookups** | Domain → IP resolution | Network latency | 20-100ms |
| **Static Assets** | Images, CSS, JS | Bandwidth savings | Varies |

**Real-world examples:**
- **External API**: Cache currency exchange rates for 1 hour (API has rate limits)
- **Computation**: Cache Netflix recommendations (algorithm takes 2 seconds to run)
- **Microservice**: Cache user data in Order service (avoid repeated calls to User service)
- **Configuration**: Cache feature flags in-memory (checked on every request)

**The universal pattern:**
```
If it's SLOW and ACCESSED FREQUENTLY → Cache it!
```

Database is just the most common example, but the principle applies to **any expensive operation**.

---

## The Two Hard Problems in Computer Science

> "There are only two hard problems in Computer Science: cache invalidation and naming things."
> — Phil Karlton

**Why cache invalidation is hard**:
- How do you know when cached data is stale?
- How do you update/remove it across distributed systems?
- We'll cover this in Topic #4 (Invalidation Strategies)

---

## Interview Talking Points

When discussing caching in interviews:
1. **Always mention the trade-off**: Speed vs freshness
2. **Ask about read/write patterns**: "What's the read:write ratio?"
3. **Consider consistency requirements**: "Can we tolerate stale data?"
4. **Think about scale**: "Is this data shared across users?"
5. **Discuss hit ratio goals**: "We'd aim for 80%+ hit ratio"

---

## Related Concepts
- **[Next: 02-benefits-and-tradeoffs.md](02-benefits-and-tradeoffs.md)** - Deep dive into why caching matters
- **[Topic 04: Invalidation Strategies](04-invalidation-strategies.md)** - The hardest problem
- **[Topic 09: Cache Topologies](09-cache-topologies.md)** - Where to place your cache

---

*Last updated: 2026-03-29*
*Status: ✅ Complete*
