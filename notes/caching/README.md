# Caching - Study Notes

> **Goal**: Build a practical understanding of caching strategies, patterns, and implementations for distributed systems
>
> **Audience**: Backend engineers working with Go/Java microservices
>
> **Approach**: Concepts + Diagrams + Trade-offs (No code snippets - focus on understanding)
>
> **Purpose**: Both daily work implementation AND interview preparation

---

## 📚 Learning Path

### 1. Foundations
- [ ] **[01-what-is-caching.md](01-what-is-caching.md)** - Introduction to caching, why it exists, and when to use it
- [ ] **[02-benefits-and-tradeoffs.md](02-benefits-and-tradeoffs.md)** - Performance gains, cost savings, and hidden costs

### 2. Cache Eviction Policies
- [ ] **[03-eviction-policies.md](03-eviction-policies.md)** - LRU, LFU, FIFO, Random, TTL-based eviction
  - When to use each policy
  - Implementation patterns in Go/Java
  - Performance characteristics

### 3. Cache Invalidation Strategies
- [ ] **[04-invalidation-strategies.md](04-invalidation-strategies.md)** - The "hardest problem in computer science"
  - TTL-based invalidation
  - Event-driven invalidation
  - Version-based invalidation
  - Cache stampede prevention

### 4. Caching Patterns
- [ ] **[05-cache-aside.md](05-cache-aside.md)** - Lazy loading pattern (most common)
- [ ] **[06-read-through-write-through.md](06-read-through-write-through.md)** - Cache as primary interface
- [ ] **[07-write-back-write-behind.md](07-write-back-write-behind.md)** - Async writes for performance
- [ ] **[08-refresh-ahead.md](08-refresh-ahead.md)** - Proactive cache warming

### 5. Cache Architectures
- [ ] **[09-cache-topologies.md](09-cache-topologies.md)** - Local, distributed, layered caching
- [ ] **[10-distributed-cache-systems.md](10-distributed-cache-systems.md)** - Redis, Memcached, Hazelcast
- [ ] **[11-consistency-models.md](11-consistency-models.md)** - Strong vs eventual consistency in caches

### 6. Advanced Topics
- [ ] **[12-cache-warming.md](12-cache-warming.md)** - Strategies to prevent cold start
- [ ] **[13-cache-stampede.md](13-cache-stampede.md)** - Thundering herd problem and solutions
- [ ] **[14-multi-layer-caching.md](14-multi-layer-caching.md)** - Browser → CDN → App → Database
- [ ] **[15-monitoring-and-metrics.md](15-monitoring-and-metrics.md)** - Hit ratio, latency, memory usage

### 7. Practical Implementation
- [ ] **[16-common-pitfalls.md](16-common-pitfalls.md)** - What can go wrong and how to avoid it
- [ ] **[17-case-studies.md](17-case-studies.md)** - Real-world examples (Netflix, Amazon, etc.)
- [ ] **[18-implementation-checklist.md](18-implementation-checklist.md)** - Decision framework for adding caching

---

## 🗂️ Structure

```
notes/caching/
├── README.md                          # This file - curriculum overview
├── 00-curriculum-tracker.md          # Progress tracking with checkboxes
├── diagrams/                          # Mermaid diagrams (SVG exports)
│   ├── cache-aside-flow.svg
│   ├── write-through-flow.svg
│   ├── cache-topologies.svg
│   └── ...
├── 01-what-is-caching.md             # Individual topic notes
├── 02-benefits-and-tradeoffs.md
└── ...
```

---

## 🎯 How to Use These Notes

1. **Follow the sequence** - Topics build on each other
2. **Run the code examples** - All snippets are runnable
3. **Study the diagrams** - Visual understanding is key
4. **Connect to your work** - Think about where caching applies in your systems
5. **Come back often** - Use as a reference when implementing caching

---

## 📖 Additional Resources

- [Redis Documentation](https://redis.io/docs/)
- [Memcached Wiki](https://github.com/memcached/memcached/wiki)
- Martin Kleppmann - Designing Data-Intensive Applications (Chapter on Caching)
- [Cloudflare: What is Caching?](https://www.cloudflare.com/learning/cdn/what-is-caching/)

---

*Started: 2026-03-29*
*Status: In Progress*
