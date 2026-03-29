# Cache Eviction Policies

## What It Is
- Algorithm that decides **which cached item to remove** when cache is full
- Required because caches have limited memory
- Wrong policy can drop hit ratio from 90% → 30%

## The Core Question
**Which cached item is least valuable right now?**

Different policies have different answers:
- **LRU**: Least recently used
- **LFU**: Least frequently used
- **FIFO**: Oldest item
- **Random**: Pick randomly
- **TTL**: Expired items

## LRU (Least Recently Used)

**Most Popular Policy**

**What It Does:**
- Removes item that hasn't been accessed for the longest time
- Maintains access timestamp for each item
- Every cache hit updates timestamp

**When To Use:**
- ✅ Temporal locality (recent items accessed again soon)
- ✅ General-purpose caching (safe default)
- ✅ Session data, user preferences, recent queries

**When To Avoid:**
- ❌ Sequential scans (one-time reads pollute cache)
- ❌ Periodic patterns (weekly reports evicted before next use)

**Performance:**
- Time: O(1) with doubly-linked list + hashmap
- Space: O(n)
- Typical Hit Ratio: 90-95%

**Examples:**
- Redis: `allkeys-lru`
- Web browser cache

## LFU (Least Frequently Used)

**What It Does:**
- Removes item accessed the fewest times
- Maintains access counter for each item
- Every cache hit increments counter

**When To Use:**
- ✅ Power-law distribution (few items very popular)
- ✅ Long-lived cache where frequency > recency
- ✅ Popular content (trending videos, top products)

**When To Avoid:**
- ❌ Access patterns change over time (stale popular items stick)
- ❌ One-time viral items (high counter, no longer accessed)

**The Problem:**
- **Stale Popular Item**: Item popular yesterday (high counter), not accessed today, still occupies cache
- **Solution**: LFU with decay (reduce counters over time)

**Performance:**
- Time: O(log n) with min-heap
- Space: O(n)
- Typical Hit Ratio: 85-90%

**Examples:**
- CDN caching for popular videos
- Redis: `allkeys-lfu`

## FIFO (First In, First Out)

**What It Does:**
- Removes oldest item, regardless of access patterns
- Simple queue behavior
- No tracking of access patterns

**When To Use:**
- ✅ Simple, predictable eviction needed
- ✅ All items have similar value
- ✅ Temporary buffers (logs, message queues)

**When To Avoid:**
- ❌ Variable access patterns (ignores popularity)
- ❌ Need high hit ratio (LRU/LFU perform better)

**Performance:**
- Time: O(1) with queue
- Space: O(n)
- Typical Hit Ratio: 60-70%

**Examples:**
- Message queues
- Log rotation

## Random Eviction

**What It Does:**
- Picks random item to evict
- No metadata tracking
- No coordination needed

**Surprising Facts:**
- Random is **80-90% as good as LRU** in many workloads
- Much simpler to implement (10 lines of code)
- No pathological worst cases
- No metadata overhead

**When To Use:**
- ✅ Simplicity > optimal hit ratio
- ✅ Distributed caches (coordination is expensive)
- ✅ Avoiding worst-case patterns

**When To Avoid:**
- ❌ Need maximum hit ratio
- ❌ Clear hot/cold data patterns

**Performance:**
- Time: O(1)
- Space: O(1) - no metadata
- Typical Hit Ratio: 70-80%

**Examples:**
- Memcached
- Redis: `allkeys-random`

## TTL (Time To Live)

**What It Does:**
- Removes items that have expired based on time limit
- Each item has expiration timestamp
- Background process scans for expired items
- Often combined with other policies (LRU + TTL)

**When To Use:**
- ✅ Data has natural expiration (API rate limits, session tokens)
- ✅ Freshness is critical (stock prices, weather)
- ✅ Regulatory requirements (delete after X days)

**TTL Patterns:**
- **Short (seconds/minutes)**: Real-time data, auth tokens
- **Medium (hours)**: API responses, computed results
- **Long (days)**: Static assets, config data

**Performance:**
- Time: O(1) to check, O(n) for cleanup
- Space: O(n)
- Typical Hit Ratio: 80-90%

**Examples:**
- Redis `EXPIRE` command
- HTTP `Cache-Control: max-age=3600`
- JWT tokens with `exp` claim

## Visual Comparison

![Eviction Policies Comparison](./diagrams/eviction-policies.svg)

## Policy Comparison Table

| Policy | Best For | Hit Ratio | Complexity | Memory |
|--------|----------|-----------|------------|--------|
| **LRU** | General purpose, temporal locality | ⭐⭐⭐⭐⭐ | Medium | O(n) |
| **LFU** | Popular items, power-law | ⭐⭐⭐⭐ | High | O(n) |
| **TTL** | Time-sensitive, freshness | ⭐⭐⭐⭐ | Low | O(n) |
| **Random** | Simplicity, distributed | ⭐⭐⭐ | Very Low | O(1) |
| **FIFO** | Simple queues, equal value | ⭐⭐ | Low | O(n) |

## Hybrid Approaches

### LRU + TTL (Most Common)
- LRU handles eviction when cache is full
- TTL ensures freshness (prevents stale data)
- **Best of both worlds**
- Example: Redis with `allkeys-lru` + `EXPIRE`

### Segmented LRU
- Split cache into "hot" and "cold" sections
- New items start in cold section
- Promoted to hot on second access
- Prevents one-time scans from polluting cache
- Example: Caffeine (Java caching library)

### Adaptive Replacement Cache (ARC)
- Dynamically balances between LRU and LFU
- Learns from workload patterns
- Self-tuning based on hit/miss history
- Example: PostgreSQL buffer pool

## Real-World Scenarios

**Scenario 1: API Response Cache**
- Pattern: Some endpoints 100x more popular
- Lifetime: Changes every 5 minutes
- **Best choice**: LRU + TTL (5 min)
- **Why**: Popular APIs stay (LRU), stale data expires (TTL)

**Scenario 2: User Session Cache**
- Pattern: Active users access repeatedly
- Lifetime: Expire after 30 min inactivity
- **Best choice**: LRU + TTL (30 min sliding)
- **Why**: Active sessions cached, inactive expire

**Scenario 3: CDN Video Streaming**
- Pattern: Power-law (top 1% = 90% traffic)
- Lifetime: Videos are immutable
- **Best choice**: LFU
- **Why**: Keep most frequently accessed videos

**Scenario 4: Log Aggregation Buffer**
- Pattern: Write once, read once, discard
- Lifetime: Processed within seconds
- **Best choice**: FIFO
- **Why**: Simple queue semantics

## Key Takeaways
- ✅ **Default to LRU + TTL** - works for 90% of cases
- ✅ **Wrong policy = 3x worse hit ratio** (seen 92% → 35%)
- ✅ **Random is shockingly good** - 80-90% of LRU's performance, 1% complexity
- ✅ **LFU + decay** for power-law patterns (prevents stale popular items)
- ✅ **TTL mandatory** for time-sensitive data
- ⚠️ **Sequential scans kill LRU** - consider segmented caching

## Interview Talking Points

**When asked about eviction:**

**You:** "I'd default to **LRU + TTL**. LRU handles eviction when memory fills up by removing least-recently-used items, which works well for temporal locality. TTL ensures we're not serving stale responses beyond their valid lifetime. We'd implement with a doubly-linked list + hashmap for O(1) access and eviction."

**If asked "Why not LFU?":**

**You:** "LFU has the stale popular item problem. If an endpoint was popular yesterday but traffic shifted today, LFU keeps it due to high historical counter. LRU adapts faster - if that endpoint isn't used recently, it gets evicted. For most API traffic, LRU is more adaptive."

**If asked "What about Random?":**

**You:** "Random is surprisingly effective - gets 80-90% of LRU's hit ratio with way less complexity. No metadata tracking, no coordination overhead. If this were a distributed cache across 100 nodes where coordination is expensive, Random would be worth considering. But for centralized cache with predictable patterns, LRU gives better hit ratio without much extra complexity."

## Related Concepts
- **[Previous: 02-benefits-and-tradeoffs.md](02-benefits-and-tradeoffs.md)** - When caching is worth it
- **[Next: 04-invalidation-strategies.md](04-invalidation-strategies.md)** - The hardest problem
- **[Topic 09: Cache Topologies](09-cache-topologies.md)** - Where eviction happens

---

*Last updated: 2026-03-30*
*Status: ✅ Complete (Concise Reference)*
