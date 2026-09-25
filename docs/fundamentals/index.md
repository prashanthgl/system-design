# Fundamentals

**28 building blocks every case study composes — get these solid first, because a shaky fundamental shows up as a shaky answer in every design that touches it.**

Each page goes deep enough to survive a 15-minute interviewer deep dive: real numbers, explicit
trade-off tables, and a **Gotchas & Corner Cases** section covering the mechanistic traps that
separate a senior answer from a textbook one.

## Networking & Traffic

<div class="grid cards" markdown>

- **[F01 — Networking Foundations](f01-networking-foundations.md)**

    TCP handshake and congestion control, TLS 1.3 handshake cost, HTTP/1.1 vs HTTP/2 vs
    HTTP/3-QUIC, head-of-line blocking, MTU/MSS blackholes, port exhaustion.

- **[F02 — DNS & Global Traffic Management](f02-dns-traffic-management.md)**

    Resolution path, TTL trade-offs, Anycast vs GeoDNS vs GSLB, why DNS failover is slower than
    you think.

- **[F03 — Load Balancing](f03-load-balancing.md)**

    L4 vs L7, consistent hashing (Maglev), least-connections vs power-of-two-choices, health
    checks and cascading failure from over-aggressive checking.

- **[F05 — CDN & Edge](f05-cdn-edge.md)**

    PoP hierarchy, cache key normalization, purge propagation, origin shielding, signed URLs.

</div>

## Data & Consistency

<div class="grid cards" markdown>

- **[F04 — Caching](f04-caching.md)**

    Cache-aside/write-through/write-behind, stampede control, hot keys, invalidation races.

- **[F06 — Partitioning & Sharding](f06-partitioning-sharding.md)**

    Range vs hash vs directory partitioning, consistent hashing with vnodes, online resharding.

- **[F07 — Replication & Consistency](f07-replication-consistency.md)**

    Sync/async/semi-sync, quorums, read-your-writes, replication lag, conflict resolution.

- **[F08 — CAP, PACELC & Consistency Models](f08-cap-pacelc.md)**

    The precise CAP statement, the consistency hierarchy, isolation anomalies with real examples.

- **[F09 — Consensus](f09-consensus.md)**

    Raft/Paxos leader election and log replication, quorum sizing, why you rarely write your own.

- **[F10 — Distributed Transactions](f10-distributed-transactions.md)**

    2PC and its blocking problem, Sagas with compensating actions, the transactional outbox.

- **[F11 — Idempotency & Exactly-Once](f11-idempotency.md)**

    Idempotency keys, dedup windows, effectively-once vs exactly-once semantics.

- **[F19 — Concurrency Control](f19-concurrency-control.md)**

    Optimistic vs pessimistic locking, distributed locks and fencing tokens, lease-expiry hazards.

- **[F20 — Time, Clocks & Ordering](f20-time-clocks-ordering.md)**

    NTP skew, logical/vector/hybrid-logical clocks, TrueTime, why timestamps aren't IDs.

</div>

## Storage & Messaging

<div class="grid cards" markdown>

- **[F12 — Message Queues & Streams](f12-queues-streams.md)**

    Queue vs log semantics, delivery guarantees, consumer groups, backpressure, DLQs.

- **[F13 — Storage Engines](f13-storage-engines.md)**

    B-tree vs LSM-tree, write/read/space amplification, compaction, WAL durability.

- **[F14 — SQL vs NoSQL Selection](f14-sql-vs-nosql.md)**

    Access-pattern-driven selection, single-table design, when a single Postgres node is enough.

- **[F15 — Object & Blob Storage](f15-object-storage.md)**

    S3 semantics, multipart upload, storage classes, erasure coding vs replication durability math.

- **[F16 — Search & Indexing](f16-search-indexing.md)**

    Inverted indexes, BM25 scoring, near-real-time indexing, scatter-gather tail latency, ANN.

- **[F21 — Probabilistic Data Structures](f21-probabilistic-data-structures.md)**

    Bloom filters, HyperLogLog, Count-Min Sketch, t-digest — with the sizing math worked out.

</div>

## Reliability & Operations

<div class="grid cards" markdown>

- **[F17 — Rate Limiting & Load Shedding](f17-rate-limiting-load-shedding.md)**

    Token/leaky bucket, distributed rate limiting, adaptive concurrency limits, brownout ladders.

- **[F18 — Resilience Patterns](f18-resilience-patterns.md)**

    Timeouts, retries with jitter, retry budgets, circuit breakers, metastable failure states.

- **[F22 — Observability Fundamentals](f22-observability-fundamentals.md)**

    RED/USE methods, cardinality explosion, histogram quantiles, sampling, trace propagation.

- **[F23 — SLI/SLO & Error Budgets](f23-slo-error-budgets.md)**

    Choosing good SLIs, multi-window multi-burn-rate alerting, error budget policy.

- **[F24 — Capacity Planning](f24-capacity-planning.md)**

    Little's Law, the utilization/latency knee, N+1/N+2 regional planning, closed-loop test traps.

- **[F25 — Deployment & Release Safety](f25-deployment-release-safety.md)**

    Canary analysis, progressive delivery, expand/contract schema migration, rollback limits.

- **[F26 — Multi-Region & DR](f26-multi-region-dr.md)**

    Active-active vs active-passive, RTO/RPO, split-brain, failover timing, DR drills.

- **[F27 — Security in Design](f27-security-design.md)**

    AuthN/AuthZ, OAuth2/OIDC, mTLS rotation, secrets management, tenant isolation, OWASP mapping.

- **[F28 — Cost Engineering](f28-cost-engineering.md)**

    Cost-per-request models, cross-AZ traffic traps, storage tiering, the price of one more nine.

</div>
