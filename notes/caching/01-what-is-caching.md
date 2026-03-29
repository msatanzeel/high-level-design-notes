# What is Caching?

## What It Is
- Storing a copy of frequently accessed data in a **faster storage layer** (closer to the application)
- Avoids fetching from the slower source every time
- Acts as **high-speed short-term memory** between app and data source

## The Core Trade-Off
**Speed vs Freshness**
- Faster access = potentially stale data (might be outdated)
- Always fresh data = slower access (hit the source every time)
- You can't have both perfectly - must find acceptable balance

## Key Terms
- **Cache Hit**: Data found in cache → fast (2-5ms)
- **Cache Miss**: Data NOT in cache → must fetch from source (50-500ms)
- **Hit Ratio**: % of requests served from cache (aim for >80%)
- **TTL (Time To Live)**: How long data stays in cache before expiring

## How It Works

**Without Cache:**
```
Request 1 → Database (100ms)
Request 2 → Database (100ms)
Request 3 → Database (100ms)
Total: 300ms, 3 DB queries
```

**With Cache:**
```
Request 1 → Cache Miss → Database (100ms) → Store in Cache
Request 2 → Cache Hit (2ms)
Request 3 → Cache Hit (2ms)
Total: 104ms (71% faster!), 1 DB query (67% reduction!)
```

## Visual Flow

![Caching Flow](./diagrams/basic-caching-flow.svg)

**Request Flow:**
1. App receives request
2. Check cache first (Redis, in-memory, etc.)
3. **Cache Hit**: Return immediately (2-5ms)
4. **Cache Miss**: Fetch from source → Store in cache → Return

## Where Caching Happens

| Layer | Example | Speed | Typical TTL |
|-------|---------|-------|-------------|
| Browser | Static assets, API responses | ~0ms | Days/Weeks |
| CDN | Images, videos, HTML | 10-50ms | Hours/Days |
| Application | In-memory (HashMap, map) | 1-5ms | Minutes/Hours |
| Distributed | Redis, Memcached | 2-10ms | Minutes/Hours |
| Database | Query cache, buffer pool | 5-20ms | Dynamic |
| CPU | L1/L2/L3 cache | Nanoseconds | Automatic |

## When To Use Caching

**✅ Use When:**
- Read-heavy workload (10:1 read:write ratio or higher)
- Data is expensive to fetch/compute (>50ms)
- Can tolerate staleness (eventually consistent is OK)
- Same data requested by multiple users (shared data)

**❌ Avoid When:**
- Write-heavy workload (data changes constantly)
- Unique data per request (no reuse benefit)
- Strict consistency required (financial transactions, inventory)
- Memory cost > compute/fetch cost (huge blobs, rarely accessed)

## What Can Be Cached

**Any expensive operation**, not just database queries:

| Data Source | Example | Why Cache? | Latency Saved |
|-------------|---------|------------|---------------|
| Database | User profile query | Reduce DB load | 50-500ms → 2ms |
| External APIs | Weather API, payment gateway | Rate limits, cost | 100-2000ms → 2ms |
| Microservices | Auth service, product service | Network hops | 20-200ms → 2ms |
| Computations | ML inference, recommendations | Avoid re-computing | 100-5000ms → 2ms |
| File System | Config files, templates | Disk I/O | 10-100ms → 2ms |
| DNS Lookups | Domain → IP | Network latency | 20-100ms → 2ms |
| Static Assets | Images, CSS, JS | Bandwidth | Varies → instant |

**Pattern**: If it's SLOW and ACCESSED FREQUENTLY → Cache it

## Common Use Cases
1. **API Response Caching** - External APIs (weather, currency, payment gateway)
2. **Database Query Results** - Frequent SELECT queries
3. **Computed Results** - Recommendation algorithms, ML inference, analytics
4. **Session Data** - Auth tokens, user preferences, shopping cart
5. **Static Assets** - Images, CSS, JS (CDN caching)
6. **Configuration** - Feature flags, app settings

## The Famous Problem

> "There are only two hard problems in Computer Science: cache invalidation and naming things."
> — Phil Karlton

**Why cache invalidation is hard:**
- How do you know when cached data is stale?
- How do you update/remove it across distributed systems?
- What if invalidation logic has bugs?

(Covered in Topic #4)

## Interview Talking Points

**When asked about caching:**
1. **Mention the trade-off**: Speed vs freshness
2. **Ask about patterns**: "What's the read:write ratio?"
3. **Consider consistency**: "Can we tolerate stale data?"
4. **Think about scale**: "Is this data shared across users?"
5. **Discuss hit ratio**: "We'd aim for 80%+ hit ratio"

**Example response:**
> "I'd introduce a caching layer like Redis between app and database. This gives us two wins: reduced latency (100ms → 2ms) and reduced DB load (90% hit ratio means DB only handles 10% of traffic). The trade-off is staleness - cached data might be 5 minutes old. I'd need to confirm the business can tolerate that."

## Related Concepts
- **[Next: 02-benefits-and-tradeoffs.md](02-benefits-and-tradeoffs.md)** - When caching is worth the complexity
- **[Topic 04: Invalidation Strategies](04-invalidation-strategies.md)** - The hardest problem
- **[Topic 09: Cache Topologies](09-cache-topologies.md)** - Where to place your cache

---

*Last updated: 2026-03-30*
*Status: ✅ Complete (Concise Reference)*
