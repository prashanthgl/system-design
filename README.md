# System Design for Senior SRE / Tech Lead Interviews

A structured, depth-first curriculum for preparing for system design rounds at top-tier
companies (FAANG+, Stripe, Databricks, Cloudflare, Snowflake, OpenAI, etc.).

**Audience:** ~10 years experience, SRE / Infrastructure Tech Lead track.
**Bias of this repo:** correctness under failure, operability, capacity, cost, and blast-radius
control — not just "draw boxes and add a cache". Every case study carries an explicit
**SRE lens** section because that is what differentiates a Staff/Lead SRE candidate.

---

## Table of Contents

1. [How to Use This Repo](#how-to-use-this-repo)
2. [Interview Framework](#interview-framework)
3. [Part I — Fundamentals & Building Blocks](#part-i--fundamentals--building-blocks)
4. [Part II — Case Studies (40 Designs)](#part-ii--case-studies-40-designs)
5. [Part III — SRE-Specific Design Rounds](#part-iii--sre-specific-design-rounds)
6. [Part IV — Deep Dives & Papers](#part-iv--deep-dives--papers)
7. [Back-of-the-Envelope Cheat Sheet](#back-of-the-envelope-cheat-sheet)
8. [Study Plan](#study-plan)
9. [Repo Structure & Publishing](#repo-structure--publishing)

---

## How to Use This Repo

- Each case study is a standalone page written to a fixed template (below), so you can drill a
  single topic in 30–45 minutes.
- Do **not** read passively. For each design: whiteboard it cold first, then read, then diff.
- Track your confidence per topic in the Status column (`todo` → `drafted` → `drilled` → `solid`).

### Per-Case-Study Template

Every page under `docs/case-studies/` follows this structure:

| Section | What it must answer |
|---|---|
| 1. Problem Statement | The literal question as asked in interviews |
| 2. Requirements | Functional, non-functional, explicitly out-of-scope |
| 3. Scale Estimation | DAU, QPS (avg/peak), storage/yr, bandwidth, connection count |
| 4. API Design | Public contract, idempotency, pagination, versioning |
| 5. Data Model | Entities, access patterns, chosen store + justification |
| 6. High-Level Architecture | Diagram + request/write path walkthrough |
| 7. Deep Dives | The 2–3 areas the interviewer will actually push on |
| 8. Scaling the Bottleneck | Sharding, caching, partitioning, hot keys, fan-out |
| 9. Failure Modes | What breaks, how it's detected, how it degrades |
| 10. **SRE Lens** | SLIs/SLOs, error budget, rollout strategy, on-call runbook, capacity model |
| 11. Trade-offs & Alternatives | What you'd do differently at 10x / 1/10th scale |
| 12. Follow-up Questions | The curveballs, with answers |

---

## Interview Framework

A repeatable 45-minute structure. Practice the clock, not just the content.

| Phase | Time | Output |
|---|---|---|
| Requirements & scoping | 5 min | Written functional + non-functional list, agreed scope |
| Scale estimation | 5 min | QPS, storage, bandwidth — numbers on the board |
| API + data model | 5 min | Contracts and entities |
| High-level design | 10 min | Boxes, arrows, and the happy-path walkthrough |
| Deep dive | 15 min | Interviewer-directed; you drive if they don't |
| Failure / ops / wrap-up | 5 min | Bottlenecks, failure modes, SLOs, "what I'd do next" |

**Signals that get you the Staff/Lead bar:**
- You state the CAP/PACELC choice explicitly and justify it with the business requirement.
- You quantify. "Roughly 50k QPS peak, so ~25 shards at 2k writes/shard" beats "we shard it".
- You name the failure mode *before* the interviewer does.
- You discuss migration and rollout, not just the steady-state end picture.
- You talk cost. Storage tiering, egress, instance mix, retention.
- You know when *not* to distribute. A single Postgres box handles more than most people think.

---

## Part I — Fundamentals & Building Blocks

These are the primitives every case study composes. Get these solid first.

| # | Topic | Key depth to reach | Status |
|---|---|---|---|
| F01 | Networking Foundations | TCP vs QUIC, TLS handshake cost, connection pooling, keep-alive, head-of-line blocking, MTU/MSS | todo |
| F02 | DNS & Global Traffic Management | Anycast, GeoDNS, TTL trade-offs, DNS failover lag, health-checked GSLB | todo |
| F03 | Load Balancing | L4 vs L7, consistent hashing, maglev, least-conn vs P2C, connection draining, LB as SPOF | todo |
| F04 | Caching | Cache-aside/write-through/write-behind, TTL + jitter, stampede control, hot keys, negative caching, invalidation | todo |
| F05 | CDN & Edge | Origin shielding, cache keys, purge propagation, edge compute, signed URLs | todo |
| F06 | Data Partitioning / Sharding | Range vs hash vs directory, consistent hashing + vnodes, resharding online, hot shard mitigation | todo |
| F07 | Replication & Consistency | Sync/async/semi-sync, quorum (R+W>N), read-your-writes, monotonic reads, replication lag effects | todo |
| F08 | CAP, PACELC & Consistency Models | Linearizable → serializable → SI → RC → eventual; what each costs in latency | todo |
| F09 | Consensus | Raft/Paxos leader election, log replication, membership change, why you rarely write your own | todo |
| F10 | Distributed Transactions | 2PC and its blocking problem, Saga + compensations, outbox pattern, TCC | todo |
| F11 | Idempotency & Exactly-Once | Idempotency keys, dedup windows, effectively-once vs exactly-once semantics | todo |
| F12 | Message Queues & Streams | Queue vs log, at-least/at-most/exactly-once, ordering guarantees, consumer groups, backpressure, DLQ | todo |
| F13 | Storage Engines | B-tree vs LSM, write/read/space amplification, compaction, WAL, fsync durability | todo |
| F14 | SQL vs NoSQL Selection | Access-pattern-driven choice, secondary indexes, materialized views, when Postgres is enough | todo |
| F15 | Object & Blob Storage | S3 semantics, multipart upload, storage classes, erasure coding vs replication | todo |
| F16 | Search & Indexing | Inverted index, tokenization, TF-IDF/BM25, near-real-time indexing, index sharding | todo |
| F17 | Rate Limiting & Load Shedding | Token/leaky bucket, sliding window, distributed counters, adaptive concurrency limits, priority shedding | todo |
| F18 | Resilience Patterns | Timeouts, retries + exponential backoff + jitter, retry budgets, circuit breakers, bulkheads, hedged requests | todo |
| F19 | Concurrency Control | Optimistic vs pessimistic locking, distributed locks + fencing tokens, lease expiry hazards | todo |
| F20 | Time, Clocks & Ordering | NTP skew, logical clocks, vector clocks, TrueTime/HLC, why timestamps aren't IDs | todo |
| F21 | Probabilistic Data Structures | Bloom/Cuckoo filters, HyperLogLog, Count-Min Sketch, t-digest for percentiles | todo |
| F22 | Observability Fundamentals | RED/USE methods, metric cardinality, sampling, trace propagation, log volume economics | todo |
| F23 | SLI / SLO / Error Budgets | Choosing good SLIs, multi-window multi-burn-rate alerting, error budget policy | todo |
| F24 | Capacity Planning | Little's Law, utilization vs latency knee, headroom targets, N+1 / N+2 regional planning | todo |
| F25 | Deployment & Release Safety | Blue/green, canary + automated analysis, feature flags, schema migration (expand/contract), rollback | todo |
| F26 | Multi-Region & DR | Active-active vs active-passive, RTO/RPO, data residency, split-brain, failover drills | todo |
| F27 | Security in Design | AuthN/AuthZ, OAuth2/OIDC, mTLS, secrets management, encryption at rest/in transit, tenant isolation | todo |
| F28 | Cost Engineering | $/request modelling, egress traps, storage tiering, spot/preemptible strategy, right-sizing | todo |

---

## Part II — Case Studies (40 Designs)

Ordered roughly by difficulty within each theme. Each row lists the **deep dives that decide the
outcome of the interview** — the parts you cannot hand-wave.

### A. Core / Warm-up Classics

| # | Design | Deep dives that matter | Status |
|---|---|---|---|
| 01 | **URL Shortener (TinyURL / bit.ly)** | Key generation (counter vs hash vs pre-gen pool), collision handling, 301 vs 302 and its analytics impact, read-heavy caching, custom aliases, link expiry & GC | todo |
| 02 | **Pastebin / Text Sharing** | Blob vs DB split, TTL & expiry sweeper, size limits, abuse/spam handling, cold-storage tiering | todo |
| 03 | **Distributed Rate Limiter** | Algorithm choice, per-user vs per-IP vs per-tenant, Redis vs local+gossip sync, clock skew, fail-open vs fail-closed, response headers | todo |
| 04 | **Unique ID Generator (Snowflake)** | Monotonicity, clock rollback handling, node ID assignment, k-sortability vs UUIDv7, ID leakage/enumeration | todo |
| 05 | **Distributed Key-Value Store (Dynamo-style)** | Consistent hashing + vnodes, quorum tuning, hinted handoff, Merkle-tree anti-entropy, vector clocks vs LWW, gossip membership | todo |
| 06 | **Web Crawler** | Frontier design, politeness & robots.txt, URL dedup at scale (Bloom), trap detection, freshness/recrawl scheduling, distributed coordination | todo |
| 07 | **Typeahead / Search Autocomplete** | Trie sharding + serialization, top-k per prefix, real-time vs batch index updates, personalization, spell correction, edge caching | todo |
| 08 | **Notification System (push/SMS/email)** | Fan-out, per-provider adapters & retries, dedup, user preferences & quiet hours, priority queues, delivery tracking, third-party outage handling | todo |

### B. Social, Feed & Messaging

| # | Design | Deep dives that matter | Status |
|---|---|---|---|
| 09 | **News Feed / Timeline (Twitter, Facebook)** | Fan-out-on-write vs read vs hybrid, celebrity problem, feed ranking pipeline, pagination with changing data, cache warming | todo |
| 10 | **Chat / Messaging (WhatsApp, Slack)** | WebSocket connection management at millions of conns, presence, message ordering & delivery receipts, offline queue, group fan-out, E2E encryption impact | todo |
| 11 | **Live Comments / Reactions at Scale** | Pub/sub fan-out, connection sharding, backpressure on hot streams, sampling for display, ordering vs latency | todo |
| 12 | **Twitter Search / Distributed Search Engine** | Index sharding (doc vs term), real-time index + segment merge, query fan-out & scatter-gather tail latency, relevance ranking | todo |
| 13 | **Email Service (Gmail-like)** | Mailbox storage & threading, full-text search index, SMTP ingest + spam pipeline, per-user quota, sync protocol (IMAP/push) | todo |
| 14 | **Content Moderation Pipeline** | Sync vs async classification, ML inference budget, human-review queue prioritization, appeal workflow, hash-matching (PhotoDNA-style) | todo |

### C. Media & Storage

| # | Design | Deep dives that matter | Status |
|---|---|---|---|
| 15 | **YouTube / Video Platform** | Upload → transcode pipeline (chunked, parallel), ABR/HLS packaging, CDN strategy & cache hit ratio, thumbnail/metadata service, view-count aggregation | todo |
| 16 | **Netflix-style Streaming** | Open Connect edge appliances, pre-positioning content, playback manifest, per-title encoding, recommendation serving, chaos-tested failover | todo |
| 17 | **Google Drive / Dropbox** | Chunking + content-addressed dedup, delta sync, conflict resolution, metadata service scale, sharing/ACL model, client-side watcher | todo |
| 18 | **S3-style Object Store** | Namespace partitioning, erasure coding & durability math (11 nines), multipart upload, consistency model, background repair, lifecycle tiering | todo |
| 19 | **Photo/Media Store (Instagram)** | Write path & image variants, haystack-style small-file problem, CDN + signed URLs, EXIF/privacy, cold storage migration | todo |
| 20 | **Backup & Deduplication System** | Variable-length chunking, dedup index scale, incremental-forever, restore latency, encryption + dedup tension, immutability/ransomware protection | todo |

### D. Geo, Marketplace & Transactional

| # | Design | Deep dives that matter | Status |
|---|---|---|---|
| 21 | **Proximity Service / Yelp (Nearby)** | Geohash vs quadtree vs S2/H3, boundary problem, index update rate, ranking + filtering, read replica geo-distribution | todo |
| 22 | **Uber / Ride-Hailing Dispatch** | Driver location ingest (high write rate), matching algorithm & latency budget, surge computation, state machine for trips, ETA service, exactly-once dispatch | todo |
| 23 | **Google Maps / Routing** | Map tiling & serving, graph partitioning, contraction hierarchies, live traffic ingestion, ETA modelling, offline map packs | todo |
| 24 | **Ticketmaster / Event Booking** | Inventory reservation & holds, oversell prevention, virtual waiting room, thundering herd on drop, payment coordination, idempotent booking | todo |
| 25 | **Hotel / Airbnb Reservation** | Availability calendar modelling, overlapping date-range queries, search + filter at scale, pricing service, cancellation & consistency | todo |
| 26 | **E-commerce Checkout & Inventory** | Cart service, inventory reservation vs oversell, distributed transaction via Saga, order state machine, catalog search, flash-sale handling | todo |
| 27 | **Payment System / Digital Wallet** | Double-entry ledger, idempotency keys, PSP integration + webhooks, reconciliation, exactly-once money movement, PCI scope, audit trail | todo |
| 28 | **Stock Exchange / Matching Engine** | Single-threaded matching for determinism, order book data structure, sequencer + replicated log, latency budget (µs), market data fan-out, failover without losing order | todo |
| 29 | **Fraud / Abuse Detection** | Streaming feature computation, feature store online/offline skew, model serving latency, rules vs ML, feedback loop, false-positive cost | todo |

### E. Infrastructure & Platform (highest value for SRE candidates)

| # | Design | Deep dives that matter | Status |
|---|---|---|---|
| 30 | **Distributed Message Queue (Kafka-like)** | Partitioned log, leader/ISR replication, ack levels & durability, consumer offsets, rebalancing, retention/compaction, tiered storage | todo |
| 31 | **Metrics & Monitoring System (Prometheus/Datadog)** | Ingest path, TSDB compression (Gorilla/delta-of-delta), cardinality explosion control, downsampling & retention, query fan-out, HA scrape/dedup | todo |
| 32 | **Distributed Logging (ELK / Loki)** | Ingest buffering, structured logs, index vs label-only, hot/warm/cold tiers, retention cost, PII scrubbing, query at petabyte scale | todo |
| 33 | **Distributed Tracing (Jaeger / Zipkin)** | Context propagation, head vs tail sampling, span storage & cardinality, trace assembly, dependency graph generation, overhead budget | todo |
| 34 | **Alerting & On-Call Paging (PagerDuty)** | Dedup & grouping, escalation policy engine, schedule/rotation computation, notification reliability (must not fail with the platform), flapping suppression | todo |
| 35 | **Distributed Cache Service (Redis at scale)** | Cluster topology & slot migration, hot-key mitigation, eviction policy, persistence trade-offs, failover & split-brain, multi-tenant noisy neighbours | todo |
| 36 | **CDN Design** | PoP hierarchy & tiered caching, cache key normalization, purge propagation, origin protection, TLS termination at edge, log collection from edge | todo |
| 37 | **Distributed Job Scheduler / Cron** | Leader election, exactly-once triggering, missed-run policy, DAG dependencies, long-running job checkpointing, tenant fairness & isolation | todo |
| 38 | **Container Orchestrator / Cluster Scheduler** | Declarative reconciliation loops, scheduling constraints & bin-packing, etcd as the bottleneck, node lifecycle, autoscaling, resource overcommit | todo |
| 39 | **CI/CD Platform at Scale** | Build graph & caching, runner fleet autoscaling, artifact store, hermetic builds, deployment orchestration, supply-chain security (SLSA/provenance) | todo |
| 40 | **Feature Flag & Config Service** | Low-latency evaluation (SDK-local), propagation & consistency, targeting rules, kill-switch guarantees, audit, failing safe when control plane is down | todo |

### F. Stretch / Differentiator Designs

| # | Design | Deep dives that matter | Status |
|---|---|---|---|
| 41 | **Collaborative Editor (Google Docs)** | OT vs CRDT, cursor/presence, offline merge, document history & compaction, per-doc server affinity | todo |
| 42 | **Ad Click Aggregation / Real-time Analytics** | Stream processing (windowing, watermarks), exactly-once sinks, late/duplicate events, lambda vs kappa, reconciliation with batch | todo |
| 43 | **Web Analytics (Google Analytics)** | Event collection at edge, sessionization, approximate distinct counts (HLL), pre-aggregated rollups, OLAP query serving | todo |
| 44 | **Leaderboard / Gaming Ranking** | Sorted-set sharding, approximate ranking at scale, time-window leaderboards, hot update contention, anti-cheat | todo |
| 45 | **API Gateway / Service Mesh** | Routing, authN offload, rate limiting, mTLS & cert rotation, sidecar vs sidecar-less, control-plane/data-plane split, config propagation latency | todo |
| 46 | **Distributed Coordination Service (ZooKeeper/Chubby)** | Consensus core, sessions & ephemeral nodes, leases + fencing, watch fan-out, why it's a coordination store and not a database | todo |
| 47 | **Secrets Management (Vault-like)** | Sealed storage & unseal quorum, dynamic short-lived credentials, identity-based auth, rotation without downtime, audit device, availability during outage | todo |
| 48 | **ML Inference / Recommendation Serving** | Candidate generation → ranking funnel, feature store, embedding retrieval (ANN), model rollout & shadow traffic, GPU batching, latency SLO | todo |

> 48 designs listed; **items 01–40 are the core set** to have interview-ready.
> Section F is for staying above the bar in follow-ups and for platform/infra-flavoured loops.

---

## Part III — SRE-Specific Design Rounds

Many SRE loops replace or supplement the product design round with these. They are rarely covered
in generic system design material, and they are where 10 years of operational experience pays off.

| # | Topic | What you must be able to design on a whiteboard | Status |
|---|---|---|---|
| S01 | Design an SLO for an existing service | SLI selection, measurement point (client vs server), window, burn-rate alerts, error budget policy | todo |
| S02 | Multi-region active-active migration | Traffic routing, data replication topology, conflict handling, cutover plan, rollback, drill cadence | todo |
| S03 | Zero-downtime schema & data migration | Expand/contract, dual-write + backfill + verify, shadow reads, cutover, rollback safety | todo |
| S04 | Capacity planning for a 10x growth event | Load model, Little's Law, headroom, lead-time constraints, autoscaling limits, cost envelope | todo |
| S05 | Global load shedding & brownout strategy | Priority tiers, adaptive concurrency, degradation ladder, criticality tagging, testing it | todo |
| S06 | Incident response system design | Detection → paging → triage → comms → mitigation → postmortem; tooling and the data model behind it | todo |
| S07 | Chaos / resilience testing platform | Fault injection primitives, blast radius controls, steady-state hypothesis, safety interlocks, scheduling | todo |
| S08 | Deployment safety at scale | Canary analysis automation, staged rollout across cells/regions, automatic rollback triggers | todo |
| S09 | Cell-based / bulkhead architecture | Cell sizing, routing & shuffle sharding, poison-pill containment, per-cell deploys, cost overhead | todo |
| S10 | Cost / efficiency review of a large service | Unit economics, top cost drivers, tiering, right-sizing, spot strategy, measuring efficiency wins | todo |
| S11 | Debug: "latency p99 tripled, no deploys" | Structured hypothesis tree: saturation, GC, dependency, network, cache, noisy neighbour, data skew | todo |
| S12 | Design a self-healing/auto-remediation system | Signal quality, safe actions, rate limits on remediation, human-in-the-loop, feedback to postmortems | todo |

---

## Part IV — Deep Dives & Papers

The papers behind the patterns. Read the ones marked ★ at minimum.

| Paper / Source | Why it matters |
|---|---|
| ★ Dynamo (Amazon, 2007) | Consistent hashing, quorums, hinted handoff, vector clocks |
| ★ Google File System / Bigtable | Master + chunkserver split, LSM-based storage, tablet splitting |
| ★ MapReduce & Dataflow model | Batch vs streaming, windowing, watermarks |
| ★ Kafka: a Distributed Messaging System | Partitioned log as the universal integration primitive |
| ★ Raft (In Search of an Understandable Consensus Algorithm) | Leader election and log replication you can actually explain |
| Spanner | TrueTime, external consistency, global transactions |
| Chubby | Lock service design, and the lessons on how it gets misused |
| Borg / Omega / Kubernetes | Cluster scheduling and reconciliation |
| Gorilla (Facebook TSDB) | Time-series compression — directly relevant to monitoring designs |
| Scuba / Monarch | Large-scale in-memory analytics and global metrics |
| Haystack (Facebook photo store) | Small-file problem and metadata reduction |
| Maglev (Google) | Consistent-hash software load balancing |
| Zanzibar (Google authorization) | Global ACL/permission checks at scale with bounded staleness |
| Amazon Builders' Library | Timeouts, retries, shuffle sharding, workload isolation — SRE gold |
| Google SRE Book + SRE Workbook | SLO math, error budgets, overload handling, cascading failures |

---

## Back-of-the-Envelope Cheat Sheet

**Latency numbers (order of magnitude, modern hardware)**

| Operation | Time |
|---|---|
| L1 / L2 cache reference | ~1 ns / ~4 ns |
| Main memory reference | ~100 ns |
| Mutex lock/unlock | ~25 ns |
| Compress 1 KB (Snappy) | ~2 µs |
| SSD random read (4 KB) | ~50–150 µs |
| Read 1 MB sequentially from memory | ~10 µs |
| Read 1 MB sequentially from SSD | ~100–300 µs |
| Round trip within same datacenter | ~0.5 ms |
| Read 1 MB from spinning disk | ~5–10 ms |
| Round trip CA → Netherlands | ~150 ms |

**Conversions worth memorizing**

- 1 million requests/day ≈ **12 QPS**. 1 billion/day ≈ **11.6k QPS**.
- Peak is typically **2–5x** average. Design for peak, budget for average.
- 1 day ≈ 86,400 s ≈ **10^5 s**. 1 month ≈ **2.6 × 10^6 s**.
- 1 KB × 1M = 1 GB. 1 KB × 1B = 1 TB.
- 1 Gbps ≈ 125 MB/s ≈ **~10 TB/day**.
- Human-readable: 10^3 KB, 10^6 MB, 10^9 GB, 10^12 TB, 10^15 PB.

**Rules of thumb**

- A single modern Postgres node: ~10–50k simple reads/s, low tens of thousands of writes/s with
  good schema and NVMe. Don't shard until you've justified it.
- A single Redis node: ~100k+ ops/s; the limit is usually network and hot keys, not CPU.
- Utilization above ~70–80% turns queueing delay into a cliff (M/M/1 intuition).
- Availability: 99.9% = 43.2 min/month down. 99.99% = 4.3 min/month. 99.999% = 26 s/month.
- Serial dependencies multiply: five 99.9% dependencies ⇒ ~99.5% best case.

---

## Study Plan

An 8-week plan assuming ~6–8 hours/week. Adjust the pace, keep the ordering.

| Week | Focus | Deliverable |
|---|---|---|
| 1 | Fundamentals F01–F08 | Written notes + a consistency-model comparison table |
| 2 | Fundamentals F09–F16 | Sharding & replication decision tree you can redraw from memory |
| 3 | Fundamentals F17–F28 | Resilience + SLO cheat sheet; one capacity model worked end-to-end |
| 4 | Case studies 01–08 | 8 timed 45-min mock designs (self-timed, written) |
| 5 | Case studies 09–20 | Focus on fan-out and storage trade-offs |
| 6 | Case studies 21–29 | Focus on consistency, transactions, and money |
| 7 | Case studies 30–40 | The infra set — your differentiator; go deepest here |
| 8 | Part III (S01–S12) + weak spots | 4 peer/mock interviews, 2 of them SRE-flavoured |

**Weekly rituals**
- One design done *cold* under a 45-minute timer, spoken out loud, before reading anything.
- One "explain to a rubber duck in 5 minutes" summary per completed topic.
- Maintain a personal "mistakes log": every time you miss a failure mode, write it down.

---

## Repo Structure & Publishing

Planned layout (built out incrementally):

```
.
├── README.md                  # this index
├── docs/
│   ├── index.md               # site landing page
│   ├── fundamentals/          # F01–F28
│   ├── case-studies/          # 01–48
│   ├── sre/                   # S01–S12
│   ├── cheatsheets/           # numbers, checklists, decision trees
│   └── assets/diagrams/       # exported diagram images
├── mkdocs.yml                 # or _config.yml if Jekyll
└── .github/workflows/pages.yml
```

**Publishing plan (GitHub Pages)**

- Static site generator: **MkDocs + Material** (good search, admonitions, native Mermaid support)
  as the default choice; Jekyll remains a fallback if zero-build Pages is preferred.
- Diagrams authored as **Mermaid** in-page so they stay diffable in git; only export to images
  where Mermaid can't express the layout.
- Deploy via GitHub Actions on push to `main`, publishing to the `gh-pages` branch.

**Naming convention**

- Case studies: `docs/case-studies/01-url-shortener.md`
- Fundamentals: `docs/fundamentals/f04-caching.md`
- SRE rounds: `docs/sre/s02-multi-region-active-active.md`

---

## Progress Tracker

| Section | Total | Drafted | Drilled | Solid |
|---|---|---|---|---|
| Fundamentals (F) | 28 | 0 | 0 | 0 |
| Case Studies (01–48) | 48 | 0 | 0 | 0 |
| SRE Rounds (S) | 12 | 0 | 0 | 0 |

---

## License

Personal study notes. See `LICENSE` if/when added.
