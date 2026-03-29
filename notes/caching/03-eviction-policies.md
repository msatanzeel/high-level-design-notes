# Cache Eviction Policies

## The Problem

Picture this: You've got a Redis instance with 8GB of memory caching API responses for your microservice. It's Friday afternoon during Black Friday, hit ratio is at 92%, and everything's humming along beautifully. Then suddenly, your memory fills up completely. New data needs to go in, but the cache is packed. Something has to go. But what?

You could evict the oldest item. Or the least-used item. Or just... pick one at random and hope for the best? Here's the kicker: **the wrong choice can tank your hit ratio from 92% to 35% in minutes**. I've seen it happen. The database starts sweating, response times spike from 50ms to 800ms, and suddenly everyone's in a war room wondering what went wrong.

The eviction policy is the algorithm that decides which cached item gets the boot when memory runs out. It's not academic - it's the difference between smooth scaling and a production incident.

---

## Understanding Eviction Policies

When your cache fills up, you're basically asking: **"Which piece of cached data is the least valuable right now?"** But "valuable" means different things depending on your access patterns.

Think about it like this: If you're caching user session data, recently-accessed sessions are probably still active - kick them out and those users immediately hit the database on their next request. But if you're caching product catalog data where the top 10 products get 80% of the traffic, you care more about frequency than recency - evicting that bestselling product would hurt way more than evicting some obscure item from page 47.

Different policies answer the "least valuable" question differently:
- **LRU (Least Recently Used)**: Haven't touched it in a while? Out you go.
- **LFU (Least Frequently Used)**: Only accessed once or twice total? Not valuable enough.
- **FIFO (First In, First Out)**: Been here the longest? Time to leave.
- **Random**: Eeny, meeny, miny, moe.
- **TTL (Time To Live)**: Your time is literally up.

Let's walk through each one with real scenarios where they shine (or fail spectacularly).

---

## LRU (Least Recently Used)

**The Default Choice**

Imagine you're building a microservice that caches database queries. Your traffic pattern looks like this: users hit their dashboard repeatedly during their session (temporal locality - same data accessed multiple times in a short window). LRU is your friend here.

Here's how LRU thinks: "This data was accessed 5 minutes ago, that data was accessed 30 seconds ago. If I have to evict something, I'm betting the 5-minute-old data is less likely to be accessed again than the 30-second-old data."

**How it actually works**:
- Every cached item gets a timestamp when it's accessed
- When the cache fills up, scan for the item with the oldest timestamp
- Kick it out, make room for the new data
- Every time someone reads from cache, update that item's timestamp

**When LRU makes sense**:
- ✅ **User sessions**: Active users keep accessing their session data repeatedly
- ✅ **Recent queries**: Search results, dashboard data - stuff that gets reused within minutes
- ✅ **API responses**: Recent API calls tend to repeat (users refresh, retry, paginate)
- ✅ **General purpose caching**: When you don't know the access pattern, LRU is a safe bet

**When LRU fails**:
- ❌ **Sequential scans**: Imagine a batch job reading through 1 million user records sequentially. Each record gets cached, accessed once, never again. Your entire cache fills with useless one-time-read data, evicting actually valuable frequently-accessed items. LRU can't tell the difference between "accessed recently because it's hot" and "accessed recently because a batch job touched it once."
- ❌ **Periodic patterns**: Weekly reports accessed every Monday at 9am. LRU evicts them Tuesday morning. Next Monday at 9am? Cache miss. Every week. Forever.

**Performance characteristics**:
- **Time**: O(1) per access if you use a doubly-linked list + hashmap (the classic implementation)
- **Space**: O(n) - you need to track access order for every item

**Real production example**: Redis defaults to `allkeys-lru`. Your web browser's cache uses LRU. It's battle-tested and works for 90% of use cases.

---

## LFU (Least Frequently Used)

**For When Popularity Matters More Than Recency**

Now imagine you're running a CDN serving video content. You've got 10,000 videos, but the top 100 videos account for 95% of your traffic (classic power-law distribution). A new video gets uploaded and gets 1000 views in the first hour, then drops to 5 views per day after the hype dies. LRU would keep that video in cache because it was "recently" accessed. LFU is smarter.

LFU says: "Show me the receipts. How many times has this item been accessed *in total*? Oh, only 3 times? Meanwhile this other item has been accessed 50,000 times? Yeah, the 3-time item goes."

**How it works**:
- Every item has a counter starting at 0
- Each cache hit increments that counter
- When you need to evict, pick the item with the lowest counter
- High-frequency items stay, low-frequency items go

**When LFU shines**:
- ✅ **Popular content**: Trending videos, bestselling products, hot articles
- ✅ **Long-lived caches**: When the cache runs for weeks/months and you want to keep consistently popular items
- ✅ **Power-law distributions**: When 20% of items get 80% of traffic

**When LFU hurts you**:
- ❌ **Changing trends**: Remember that viral video from yesterday with 100,000 hits? Its counter is now 100,000. But today it gets zero views. LFU keeps it in cache anyway because of that high historical counter, evicting actually relevant items. This is the "stale popular item" problem.
- ❌ **New hot items**: A brand new item starts with counter = 0. Even if it's getting hammered right now, its counter is still low compared to old popular items, so it keeps getting evicted. New trends can't establish themselves.

**The fix**: LFU with decay. Reduce all counters by some factor over time (e.g., divide by 2 every hour). Old popularity fades, new popularity can rise.

**Performance**:
- **Time**: O(log n) if you use a min-heap to find the minimum-frequency item
- **Space**: O(n) - need a counter per item

**Real example**: CDNs use LFU variants for popular content. Redis supports `allkeys-lfu`. YouTube's cache probably uses something like this (with decay) to keep trending videos hot.

---

## FIFO (First In, First Out)

**Simple, Dumb, Predictable**

FIFO doesn't care about access patterns at all. It's a queue. First item in, first item out. That's it.

Picture a message buffer that temporarily stores events before processing. Events come in, get processed once, get discarded. There's no concept of "popular" events or "frequently accessed" events. It's just a stream. FIFO makes perfect sense here.

**How it works**:
- Maintain insertion order (literally a queue)
- When cache fills up, evict the front of the queue (oldest item)
- New items get added to the back of the queue

**When FIFO works**:
- ✅ **Temporary buffers**: Log aggregation, message queues, streaming data
- ✅ **All items are equal**: No access pattern to optimize for
- ✅ **Simplicity is key**: You want dead-simple, predictable behavior

**When FIFO is the wrong call**:
- ❌ **Variable access patterns**: Some items get hit 100x more than others - FIFO doesn't care, it'll evict popular items just because they're old
- ❌ **Need good hit ratio**: FIFO typically gives you 60-70% hit ratio where LRU would give you 90%

**Performance**:
- **Time**: O(1) with a simple queue
- **Space**: O(n) - track insertion order

**Real example**: Kafka-style message brokers, log rotation systems. Not for general-purpose caching.

---

## Random Eviction

**Surprisingly Effective (and Hilariously Simple)**

Here's where things get interesting. Random eviction picks... a random item. No tracking access patterns, no counters, no timestamps. Just `Math.random()` and evict.

You'd think this would be terrible, right? Wrong. Random eviction gets you **80-90% of LRU's hit ratio** with **zero metadata overhead**. Let that sink in.

**Why random works**:
- In most workloads, the majority of cached items are "warm but not hot" - they get some traffic, but not massive traffic
- Random eviction spreads the pain evenly - no single item is safe, but also no systematic worst-case pattern
- LRU can be adversarial: sequential scans destroy LRU but barely hurt Random
- No metadata = no memory overhead, no coordination cost in distributed systems

**When random is brilliant**:
- ✅ **Distributed caches**: Coordinating access patterns across 100 cache nodes is expensive. Random? No coordination needed.
- ✅ **Simple systems**: You don't want to implement doubly-linked lists and hashmaps. Random is 10 lines of code.
- ✅ **Avoiding pathological cases**: LRU and LFU can be gamed or have worst-case scenarios. Random can't - it's unpredictable by definition.

**When random isn't enough**:
- ❌ **Need maximum hit ratio**: If you're optimizing for every percentage point, LRU beats Random
- ❌ **Clear hot data patterns**: If 5% of items get 95% of traffic, LFU will massively outperform Random

**Performance**:
- **Time**: O(1) - literally just pick random index
- **Space**: O(1) - no extra metadata at all

**Real example**: Memcached uses random eviction. Redis supports `allkeys-random`. It's more common than you'd think.

---

## TTL (Time To Live)

**When Freshness Matters**

TTL isn't really a cache-full eviction policy - it's a staleness policy. But it's critical enough to cover here because in production, you almost always combine TTL with another policy (usually LRU).

Imagine you're caching weather data from an external API. That API updates every 10 minutes. If you cache weather data for 2 hours, you're serving stale forecasts for 110 minutes out of every 120 minutes. Users see wrong data. TTL solves this: "No matter what, this data expires in 10 minutes."

**How it works**:
- Each cached item gets an expiration timestamp
- Background process periodically scans for expired items and deletes them
- Can combine with LRU: use LRU when cache is full, use TTL to ensure freshness

**When TTL is mandatory**:
- ✅ **External API responses**: Currency exchange rates, weather data, stock prices - they have natural refresh intervals
- ✅ **Session tokens**: Security requirement - expire after 30 minutes of inactivity
- ✅ **Regulatory compliance**: "Delete user data after 90 days" - TTL handles it automatically

**TTL patterns by duration**:
- **Short (seconds to minutes)**: Real-time data, auth tokens, rapidly changing content
- **Medium (hours)**: API responses, computed recommendations, search results
- **Long (days to weeks)**: Static assets, config data, rarely changing reference data

**The trap**: Setting TTL too short means constant cache misses (why even cache?). Too long means serving stale data. Finding the sweet spot is an art.

**Performance**:
- **Time**: O(1) to check if expired on access, O(n) for background cleanup
- **Space**: O(n) - one timestamp per item

**Real example**: Every single production cache uses TTL. Redis `EXPIRE` command, HTTP `Cache-Control: max-age=3600`, JWT tokens with `exp` claim.

---

## Visual Comparison

![Eviction Policies Comparison](./diagrams/eviction-policies.svg)

---

## Choosing Your Policy (Decision Framework)

Here's how I think about it when designing a cache:

**Step 1: Do you have time-sensitive data?**
- If yes → TTL is non-negotiable. Combine it with something else.

**Step 2: What does your access pattern look like?**
- **Temporal locality** (same data accessed repeatedly in a short window) → **LRU**
- **Power-law distribution** (few items very popular, long tail barely accessed) → **LFU**
- **Streaming / one-time access** → **FIFO** or don't cache at all
- **No clear pattern or don't know yet** → **LRU** (safe default) or **Random** (if simplicity matters)

**Step 3: How much do you care about squeezing out every percentage point of hit ratio?**
- Care a lot → LRU or LFU
- Care about simplicity more → Random

**Step 4: Are you distributed?**
- Yes → Random is very attractive (no coordination overhead)
- No → LRU is fine

**The 90% solution**: **LRU + TTL**. Use LRU for eviction when memory fills up, use TTL to prevent stale data. This combo handles the vast majority of real-world caching scenarios.

---

## Quick Comparison Table

| Policy | Best For | Typical Hit Ratio | Complexity | Memory Overhead |
|--------|----------|-------------------|------------|-----------------|
| **LRU** | General purpose, sessions, recent queries | 90-95% | Medium | O(n) |
| **LFU** | Popular content, power-law patterns | 85-90% | High | O(n) |
| **TTL** | Time-sensitive data, freshness critical | 80-90% | Low | O(n) |
| **Random** | Simplicity, distributed caches | 70-80% | Very Low | O(1) |
| **FIFO** | Streaming data, equal-value items | 60-70% | Low | O(n) |

---

## Hybrid Approaches (How the Pros Do It)

Real systems rarely use a single policy. Here are the battle-tested combos:

**LRU + TTL** (Most Common)
Think of it like this: TTL is your "data freshness guardian" and LRU is your "memory pressure valve." When data expires, TTL kicks it out regardless of access patterns. When memory fills up, LRU evicts the least-recently-used item. You get both freshness and decent hit ratio.

Example: Redis configured with `allkeys-lru` and explicit `EXPIRE` on keys.

**Segmented LRU** (Smarter)
Split your cache into "hot" and "cold" sections. New items start in the cold section (probation period). If they get accessed again, they're promoted to the hot section. This prevents one-time sequential scans from polluting your hot data.

When that batch job reads through 1 million user records, they all go to the cold section and quickly evict each other. Your actual hot data stays safe in the hot section. Caffeine (the Java caching library) does this. It's brilliant.

**Adaptive Replacement Cache (ARC)** (Self-Tuning)
ARC dynamically balances between LRU and LFU based on recent hit/miss patterns. If you're missing a lot of recently-evicted items, it shifts toward LRU behavior. If you're missing frequently-accessed items, it shifts toward LFU. It learns and adapts.

PostgreSQL uses an ARC-like algorithm for its buffer pool. It's complex to implement but handles changing workloads gracefully.

---

## Real-World Scenarios

**Scenario: API Response Cache**
- Access pattern: `/products/search` gets hit 1000x/sec, `/admin/stats` gets hit 5x/hour
- Data lifetime: Product data changes every 5 minutes
- Cache size: 2GB
- **Best choice**: LRU + TTL (5 min)
- **Why**: Popular endpoints stay in cache (LRU), stale data auto-expires (TTL)

**Scenario: User Session Store**
- Access pattern: Active users access sessions every few seconds, inactive users don't
- Data lifetime: Expire after 30 minutes of inactivity
- Cache size: 10GB
- **Best choice**: LRU + TTL (30 min sliding window)
- **Why**: Active sessions keep refreshing their TTL, inactive sessions expire, LRU evicts abandoned sessions when memory fills

**Scenario: CDN for Video Streaming**
- Access pattern: Power-law (top 1% of videos = 90% of traffic)
- Data lifetime: Videos are immutable, never stale
- Cache size: 1TB
- **Best choice**: LFU
- **Why**: Keep the most frequently accessed videos, evict the long-tail content

**Scenario: Log Aggregation Buffer**
- Access pattern: Write once, read once, discard
- Data lifetime: Processed within seconds
- Cache size: 100MB
- **Best choice**: FIFO
- **Why**: It's a queue. Simple as that.

---

## Key Takeaways

Here's what I wish someone had told me before I spent a weekend debugging a production cache meltdown:

- ✅ **Default to LRU + TTL** unless you have a specific reason not to. It handles 90% of cases well.
- ✅ **Wrong policy = 3x worse hit ratio**. I've seen 92% drop to 35% after switching from LRU to FIFO "for simplicity."
- ✅ **Random is shockingly good**. If you want simple and don't need perfection, Random gets you 80-90% of the way there with 1% of the complexity.
- ✅ **LFU + decay for power-law patterns**. If you have a popularity distribution, plain LFU will burn you with stale popular items. Add decay.
- ✅ **TTL is mandatory for time-sensitive data**. Don't rely on eviction policies to handle freshness. Set explicit expiration.
- ⚠️ **Sequential scans can kill LRU**. If you have batch jobs that do full table scans, consider segmented caching or separate caches for batch vs online traffic.

---

## Interview Talking Points

When someone asks about eviction policies in an interview, here's the rhythm:

**Interviewer**: "How would you design the caching layer for this system?"

**You**: "First, I'd ask about the access pattern. Are we seeing temporal locality - where recent items get accessed again soon? Or is it more of a power-law distribution where a few items are super popular? That determines whether I'd use LRU or LFU."

**Interviewer**: "Assume it's an API gateway caching responses."

**You**: "Got it. API traffic usually shows temporal locality - users hit the same endpoints repeatedly within their session. I'd go with **LRU + TTL**. LRU handles the eviction when memory fills up, and TTL ensures we're not serving stale responses beyond their valid lifetime. We'd implement it with a doubly-linked list + hashmap for O(1) access and eviction."

**Interviewer**: "Why not LFU since some endpoints are more popular?"

**You**: "LFU has the stale popular item problem. If an endpoint was popular yesterday but traffic shifted today, LFU would keep the old endpoint in cache due to its high historical counter. LRU adapts faster - if that endpoint isn't being used recently, it gets evicted. For API traffic patterns, LRU is more adaptive."

**Interviewer**: "What about Random eviction?"

**You**: "Random is surprisingly effective - gets you 80-90% of LRU's hit ratio with way less complexity. No metadata tracking, no coordination overhead. If this were a distributed cache across 100 nodes and coordination costs were significant, Random would be worth considering. But for a centralized cache with predictable access patterns, LRU gives us better hit ratio without much extra complexity."

---

## Related Concepts
- **[Previous: 02-benefits-and-tradeoffs.md](02-benefits-and-tradeoffs.md)** - When caching is worth the complexity
- **[Next: 04-invalidation-strategies.md](04-invalidation-strategies.md)** - The hardest problem in computer science
- **[Topic 09: Cache Topologies](09-cache-topologies.md)** - Where eviction happens (local vs distributed)

---

*Last updated: 2026-03-29*
*Status: ✅ Complete*
