# Numbers & Estimation Cheat Sheet

**Every estimate is the same three-step chain: turn the workload into a rate, turn the rate into a resource, then compare that resource against what one machine can actually do — this page is the lookup table for all three steps.**

The goal is never precision. The goal is to be wrong by less than an order of magnitude, quickly, out loud, with the arithmetic visible so the interviewer can follow it and correct an assumption instead of doubting the method.

---

## Latency Numbers

Order-of-magnitude values for a modern server-class machine. Memorise the *exponents*, not the digits.

<div class="numbers-table" markdown>

| Operation | Latency | Nanoseconds |
|---|---|---|
| L1 cache reference | 1 ns | 1 |
| Branch mispredict | 3 ns | 3 |
| L2 cache reference | 4 ns | 4 |
| Mutex lock/unlock, uncontended | 20 ns | 20 |
| Main memory reference | 100 ns | 100 |
| Compress 1 KB with LZ4/Snappy | 2 us | 2,000 |
| Send 1 KB over 10 Gbps NIC | 1 us | 1,000 |
| NVMe SSD random read, 4 KB | 50 us | 50,000 |
| Read 1 MB sequentially from memory | 50 us | 50,000 |
| Read 1 MB sequentially from NVMe SSD | 200 us | 200,000 |
| Round trip inside one datacenter | 500 us | 500,000 |
| TLS 1.3 full handshake, same region | 1 ms | 1,000,000 |
| Round trip within one region, cross-AZ | 1 ms | 1,000,000 |
| HDD seek | 5 ms | 5,000,000 |
| Read 1 MB sequentially from HDD | 10 ms | 10,000,000 |
| Round trip US East to US West | 60 ms | 60,000,000 |
| Round trip US East to Europe | 80 ms | 80,000,000 |
| Round trip US West to Singapore | 170 ms | 170,000,000 |

</div>

### How to read this table

There are only three regimes, each roughly 1000x apart:

| Regime | Scale | What lives there | Design consequence |
|---|---|---|---|
| Nanoseconds | $10^{-9}$ s | CPU cache, RAM, locks | Free at request granularity; only matters in hot loops |
| Microseconds | $10^{-6}$ s | NVMe, intra-DC network, memory scans | The budget you actually spend; count round trips |
| Milliseconds | $10^{-3}$ s | Disk seeks, WAN, TLS, human perception | Each one is visible in p99; cross-region is a design decision, not an implementation detail |

!!! tip "The only latency rule that matters in an interview"
    Count **network round trips on the critical path**, then multiply by the RTT of the widest hop. A design with 6 sequential intra-DC calls has a 3 ms floor before any work happens. A design with one synchronous cross-continent call has an 80 ms floor and nothing you do in code will fix it.

### TLS handshake cost

| Scenario | Extra round trips | Added latency at 1 ms RTT | Added latency at 80 ms RTT |
|---|---|---|---|
| TLS 1.2 full handshake | 2 | 2 ms | 160 ms |
| TLS 1.3 full handshake | 1 | 1 ms | 80 ms |
| TLS 1.3 session resumption | 1 | 1 ms | 80 ms |
| TLS 1.3 0-RTT resumption | 0 | 0 ms | 0 ms, replay-unsafe for non-idempotent requests |
| Connection reuse, keep-alive | 0 | 0 ms | 0 ms |

Asymmetric crypto adds CPU, not just RTT: an ECDSA P-256 signature costs roughly 50-100 us of CPU; RSA-2048 signing costs roughly 1 ms. At 10,000 handshakes/second that is one to ten full cores burned on handshakes alone, which is why TLS termination is a capacity line item and connection reuse is a throughput feature.

See [F01 Networking Foundations](../fundamentals/f01-networking-foundations.md) and [F05 CDN & Edge](../fundamentals/f05-cdn-edge.md).

---

## Powers of Two and Byte Magnitudes

$$
2^{10} = 1{,}024 \approx 10^3 \qquad 2^{20} \approx 10^6 \qquad 2^{30} \approx 10^9 \qquad 2^{40} \approx 10^{12} \qquad 2^{50} \approx 10^{15}
$$

| Unit | Bytes | Power of ten | Messages at 1 KB each | Concrete anchor |
|---|---|---|---|---|
| 1 KB | $10^3$ | $10^3$ | 1 | One JSON API response |
| 1 MB | $10^6$ | $10^6$ | 1 thousand | One photo thumbnail set |
| 1 GB | $10^9$ | $10^9$ | 1 million | Fits in RAM on anything |
| 1 TB | $10^{12}$ | $10^{12}$ | 1 billion | One NVMe drive; a large single-node DB |
| 1 PB | $10^{15}$ | $10^{15}$ | 1 trillion | A fleet, not a machine; ~1,000 drives |
| 1 EB | $10^{18}$ | $10^{18}$ | 1 quintillion | Hyperscaler total, not a system you design |

!!! note "Use decimal, not binary, in estimates"
    Say 1 GB = $10^9$ bytes. The 7% error versus $2^{30}$ is far smaller than the error in your assumption about average payload size, and it makes every subsequent multiplication doable in your head.

---

## Time Constants

| Interval | Seconds | Rounded form to use mentally |
|---|---|---|
| 1 minute | 60 | 60 |
| 1 hour | 3,600 | $3.6 \times 10^3$ |
| 1 day | 86,400 | $\approx 10^5$ |
| 1 week | 604,800 | $\approx 6 \times 10^5$ |
| 1 month (30 d) | 2,592,000 | $\approx 2.6 \times 10^6$ |
| 1 year | 31,536,000 | $\approx \pi \times 10^7$ |

The single most useful shortcut: **a day is about $10^5$ seconds**, so dividing daily volume by $10^5$ gives QPS that is 15% high — a deliberate, safe, stated-out-loud overestimate.

---

## Requests per Day to QPS

$$
\text{QPS}_{\text{avg}} = \frac{\text{requests per day}}{86{,}400}
$$

| Requests/day | Exact math | Average QPS | Peak QPS at 3x |
|---|---|---|---|
| 1 thousand | $10^3 / 86{,}400$ | 0.012 | 0.04 |
| 100 thousand | $10^5 / 86{,}400$ | 1.2 | 3.5 |
| 1 million | $10^6 / 86{,}400$ | 12 | 35 |
| 10 million | $10^7 / 86{,}400$ | 116 | 350 |
| 100 million | $10^8 / 86{,}400$ | 1,157 | 3,500 |
| 1 billion | $10^9 / 86{,}400$ | 11,574 | 35,000 |
| 10 billion | $10^{10} / 86{,}400$ | 115,741 | 350,000 |

!!! warning "Average QPS provisions nothing"
    Real traffic is diurnal. For a consumer product in one or two dominant timezones, **peak is 2-5x the daily average**; for a global product with flat follow-the-sun load it may be only 1.5-2x. For event-driven systems (ticket on-sale, sports, flash sale, push-notification fan-out), peak-to-average can exceed 100x and the average is meaningless — size from the event, not from the day.

State the multiplier explicitly: *"I will size for 3x average as steady-state peak, and treat launch spikes as a separate load-shedding problem."* See [F17 Rate Limiting & Load Shedding](../fundamentals/f17-rate-limiting-load-shedding.md) and [S05 Load Shedding & Brownout](../sre/s05-load-shedding-brownout.md).

---

## Write Rate to Storage

$$
\text{Bytes/day} = \text{writes/sec} \times \text{bytes/write} \times 86{,}400
$$

At **1 KB per record**, before replication and index overhead:

| Write rate | Per day | Per month | Per year |
|---|---|---|---|
| 10/s | 864 MB | 26 GB | 315 GB |
| 100/s | 8.6 GB | 260 GB | 3.2 TB |
| 1,000/s | 86 GB | 2.6 TB | 32 TB |
| 10,000/s | 864 GB | 26 TB | 315 TB |
| 100,000/s | 8.6 TB | 260 TB | 3.2 PB |
| 1,000,000/s | 86 TB | 2.6 PB | 32 PB |

### The multipliers people forget

| Multiplier | Typical factor | Why |
|---|---|---|
| Replication | 3x | Quorum durability; RF=3 is the default for a reason |
| Erasure coding instead of replication | 1.3-1.5x | Cheaper durability, higher read/repair cost |
| Secondary indexes | 1.2-2x | Each index is another copy of the indexed columns plus pointers |
| Write amplification, LSM | 10-30x of *disk writes*, not stored bytes | Compaction rewrites data; affects IOPS and SSD wear, not capacity |
| Free space headroom | 1.3-1.5x | Compaction, defragmentation, node-loss rebalance need room |
| Backups and snapshots | 1.2-2x | Retention policy dependent |

A useful composite: **logical bytes $\times$ 3 (RF) $\times$ 1.3 (indexes) $\times$ 1.4 (headroom) $\approx$ 5.5x** provisioned capacity per logical byte. Say that number out loud — it is a strong Staff-level signal.

See [F06 Partitioning & Sharding](../fundamentals/f06-partitioning-sharding.md), [F13 Storage Engines](../fundamentals/f13-storage-engines.md), and [F24 Capacity Planning](../fundamentals/f24-capacity-planning.md).

---

## Availability Math

$$
\text{Downtime} = (1 - A) \times T \qquad \text{where } T_{\text{year}} = 525{,}600 \text{ minutes}
$$

| Availability | Error budget | Per year | Per month (30 d) | Per week | Per day |
|---|---|---|---|---|---|
| 99% ("two nines") | $10^{-2}$ | 3.65 days | 7.2 hours | 1.68 hours | 14.4 min |
| 99.9% ("three nines") | $10^{-3}$ | 8.77 hours | 43.8 min | 10.1 min | 1.44 min |
| 99.95% | $5 \times 10^{-4}$ | 4.38 hours | 21.9 min | 5.04 min | 43.2 s |
| 99.99% ("four nines") | $10^{-4}$ | 52.6 min | 4.38 min | 1.01 min | 8.64 s |
| 99.999% ("five nines") | $10^{-5}$ | 5.26 min | 26.3 s | 6.05 s | 0.86 s |

### Composition

Serial dependencies multiply. A request that must traverse $n$ independent components:

$$
A_{\text{total}} = \prod_{i=1}^{n} A_i
$$

Four components at 99.99% each yield $0.9999^4 = 99.96\%$ — a 4x larger error budget than any single component. Redundancy within a tier does the reverse:

$$
A_{\text{redundant}} = 1 - (1 - A)^k
$$

Two independent 99% replicas give 99.99%, *if* the failures are truly independent. They usually are not: shared power, shared config push, shared dependency, shared bug. Say this explicitly.

!!! warning "Five nines is almost never the right answer"
    5.26 minutes per year is less than the time it takes a human to acknowledge a page. Anything at five nines must self-heal with no human in the loop, which means the failover logic itself becomes the dominant failure mode. In an interview, choosing 99.9% or 99.95% *and justifying it from user impact and cost* scores higher than reflexively claiming four or five nines.

See [F23 SLI/SLO & Error Budgets](../fundamentals/f23-slo-error-budgets.md) and [S01 SLO Design](../sre/s01-slo-design.md).

---

## Bandwidth Conversion

$$
\text{MB/s} = \frac{\text{Mbps}}{8} \qquad \text{GB/day} = \text{MB/s} \times 86{,}400 / 1000
$$

| Link speed | MB/s | GB/day | TB/day | TB/month |
|---|---|---|---|---|
| 1 Mbps | 0.125 | 10.8 | 0.011 | 0.32 |
| 10 Mbps | 1.25 | 108 | 0.11 | 3.2 |
| 100 Mbps | 12.5 | 1,080 | 1.08 | 32 |
| 1 Gbps | 125 | 10,800 | 10.8 | 324 |
| 10 Gbps | 1,250 | 108,000 | 108 | 3,240 |
| 25 Gbps | 3,125 | 270,000 | 270 | 8,100 |
| 100 Gbps | 12,500 | 1,080,000 | 1,080 | 32,400 |

Two anchors worth memorising: **1 Gbps = 125 MB/s = 10.8 TB/day**, and **a saturated 10 Gbps NIC moves about 100 TB/day**.

!!! note "Bandwidth is rarely the limit; packets per second and egress cost usually are"
    A 25 Gbps NIC will hit its packet-per-second ceiling on small-payload RPC traffic long before it hits bit rate. And at cloud egress pricing of roughly USD 0.05-0.09 per GB, 100 TB/month of internet egress is a five-figure monthly line item — which is the entire economic argument for a CDN. See [F28 Cost Engineering](../fundamentals/f28-cost-engineering.md) and [36 CDN Design](../case-studies/36-cdn-design.md).

---

## Single-Node Capacity Rules of Thumb

!!! warning "These are starting points, not laws"
    Every number below moves by an order of magnitude with payload size, query shape, index count, durability settings, and hardware generation. Use them to *open* a conversation — "a single Postgres primary handles low thousands of write TPS, so at my estimated 350 peak writes/sec one primary is fine; let me sanity-check that with you" — never to close one. The Staff-level move is to name the number, name its assumption, and name the measurement that would replace it.

| System | Metric | Rule of thumb | Dominant limiting factor |
|---|---|---|---|
| PostgreSQL, single primary | Point reads from cache | 10,000-50,000 QPS | CPU and connection handling |
| PostgreSQL, single primary | Write TPS, synchronous commit, NVMe | 2,000-10,000 TPS | WAL fsync, lock contention, index maintenance |
| PostgreSQL | Working set | Comfortable to a few TB; painful past ~10 TB | Vacuum, backup/restore time, index build time |
| PostgreSQL | Direct connections | 200-500 before a pooler is mandatory | Process-per-connection memory and scheduler |
| Redis, single instance | Simple GET/SET | 80,000-100,000 ops/s per core | Single-threaded command loop |
| Redis, pipelined | Batched ops | 500,000-1,000,000 ops/s | NIC and syscall batching |
| Redis | Memory per instance | 10-50 GB practical | Fork-based persistence, failover restore time |
| Kafka broker | Sustained throughput | 100-500 MB/s write | Disk sequential throughput and network |
| Kafka broker | Partition count | Low thousands of partitions | Controller metadata, replication fan-out, recovery time |
| Kafka | Consumer lag recovery | Plan for 2-3x produce rate | Replay must outrun production or lag never drains |
| App server, I/O-bound JSON API | 4 vCPU instance | 1,000-5,000 RPS | Downstream latency and concurrency limits |
| App server, CPU-heavy (serialisation, crypto, template render) | 4 vCPU instance | 200-1,000 RPS | CPU, per the formula below |
| L7 proxy (Envoy, NGINX) | Per instance | 10,000-50,000 RPS | CPU for TLS and header processing |
| Elasticsearch data node | Index size | 20-50 GB per shard, tens of shards per node | Heap pressure, merge cost |

For a CPU-bound service the ceiling is not a folklore number, it is arithmetic:

$$
\text{RPS}_{\max} = \frac{\text{cores} \times \text{target utilisation}}{\text{CPU seconds per request}}
$$

At 4 cores, 60% target utilisation, 3 ms of CPU per request: $\text{RPS}_{\max} = (4 \times 0.6) / 0.003 = 800$.

See [F24 Capacity Planning](../fundamentals/f24-capacity-planning.md) and [S04 Capacity for 10x Growth](../sre/s04-capacity-planning-10x.md).

---

## Worked Example: 50M DAU, 20 Requests per User per Day

A complete end-to-end sizing pass. This is the shape of the first five minutes of any design round.

### Step 1: Stated assumptions

| Assumption | Value | Why this value |
|---|---|---|
| DAU | 50,000,000 | Given |
| Requests per user per day | 20 | Given |
| Read:write ratio | 100:1 | Typical consumer read-heavy product; state it and move on |
| Peak-to-average | 3x | Two dominant timezones, diurnal load |
| Stored bytes per write | 2 KB | Record plus metadata, before indexes |
| Average response payload | 4 KB | JSON with an embedded object list |
| Replication factor | 3 | Quorum durability |
| Retention | Indefinite for writes | Product requirement to state and challenge |

### Step 2: Request rate

$$
R_{\text{day}} = 50 \times 10^6 \times 20 = 1 \times 10^9 \text{ requests/day}
$$

$$
\text{QPS}_{\text{avg}} = \frac{1 \times 10^9}{86{,}400} \approx 11{,}600
$$

$$
\text{QPS}_{\text{peak}} = 3 \times 11{,}600 \approx 35{,}000
$$

Splitting by the 100:1 ratio:

$$
\text{writes}_{\text{avg}} = \frac{11{,}600}{101} \approx 115/\text{s} \qquad \text{writes}_{\text{peak}} \approx 345/\text{s}
$$

$$
\text{reads}_{\text{avg}} \approx 11{,}500/\text{s} \qquad \text{reads}_{\text{peak}} \approx 34{,}500/\text{s}
$$

**Immediate conclusion to say out loud:** 345 peak writes/second fits comfortably on a single relational primary. The design problem is not write scale, it is the 34,500 peak reads/second and the storage growth. That sentence reframes the entire rest of the interview.

### Step 3: Storage

$$
\text{writes/day} = \frac{1 \times 10^9}{101} \approx 1 \times 10^7
$$

$$
\text{logical bytes/day} = 10^7 \times 2 \times 10^3 = 2 \times 10^{10} \text{ B} = 20 \text{ GB/day}
$$

$$
\text{logical bytes/year} = 20 \times 365 \approx 7.3 \text{ TB/year}
$$

Apply the multipliers:

$$
7.3 \times \underbrace{3}_{\text{RF}} \times \underbrace{1.3}_{\text{indexes}} \times \underbrace{1.4}_{\text{headroom}} \approx 40 \text{ TB/year provisioned}
$$

**Conclusion:** 40 TB in year one, 120 TB by year three. That exceeds comfortable single-node Postgres, so either shard by year two or move cold data to object storage with a tiering policy. Naming the *year* you must shard is far stronger than saying "we would shard".

### Step 4: Bandwidth

$$
\text{egress}_{\text{avg}} = 11{,}600 \times 4 \times 10^3 \text{ B} \approx 46 \text{ MB/s} \approx 0.37 \text{ Gbps}
$$

$$
\text{egress}_{\text{peak}} = 3 \times 0.37 \approx 1.1 \text{ Gbps}
$$

$$
\text{egress}_{\text{month}} = 46 \times 10^6 \times 2.6 \times 10^6 \text{ s} \approx 120 \text{ TB/month}
$$

At roughly USD 0.05-0.09 per GB, 120 TB/month of internet egress is USD 6,000-11,000 per month — the cost case for CDN offload of any cacheable fraction.

### Step 5: Cache sizing

Assume 80% of reads concentrate on 20% of objects, and 50M distinct objects are touched daily at 4 KB each:

$$
\text{daily touched set} = 50 \times 10^6 \times 4 \times 10^3 = 200 \text{ GB}
$$

$$
\text{hot 20\%} = 40 \text{ GB} \qquad \text{with 1.5x overhead} \approx 60 \text{ GB}
$$

Three Redis nodes at 32 GB with replication covers it. At an 80% hit rate the origin sees $34{,}500 \times 0.2 \approx 6{,}900$ peak reads/second — which read replicas can absorb. See [F04 Caching](../fundamentals/f04-caching.md).

### Step 6: Fleet size

At 2,000 RPS per instance and 35,000 peak QPS:

$$
N_{\text{min}} = \frac{35{,}000}{2{,}000} \approx 18 \text{ instances}
$$

To survive the loss of one of three AZs, each AZ must carry the full load on two-thirds of the fleet:

$$
N_{\text{provisioned}} = \frac{18}{2/3} = 27 \rightarrow 30 \text{ instances across 3 AZs}
$$

### Step 7: The summary table to draw on the board

| Dimension | Average | Peak | Design consequence |
|---|---|---|---|
| Total QPS | 11,600 | 35,000 | Stateless tier, ~30 instances over 3 AZs |
| Write QPS | 115 | 345 | One primary is sufficient; not the bottleneck |
| Read QPS | 11,500 | 34,500 | Cache plus read replicas; the real problem |
| Storage | 20 GB/day | — | 40 TB/yr provisioned; shard or tier by year 2 |
| Egress | 0.37 Gbps | 1.1 Gbps | 120 TB/mo; CDN the cacheable fraction |
| Cache working set | 60 GB | — | 3 nodes, 80% hit rate target |

!!! tip "Round aggressively and say why"
    11,574 becomes "about 12 thousand". 7.3 TB becomes "call it 10 TB with indexes". Nobody is grading the third significant digit; they are grading whether you noticed that the read path, not the write path, is the system.

---

## Queueing Theory Quick Reference

### Little's Law

$$
L = \lambda W
$$

$L$ is mean concurrency in the system, $\lambda$ is arrival rate, $W$ is mean time in system. It assumes nothing about distributions or scheduling, which is why it is the most reusable formula in capacity work.

| Rearrangement | Use |
|---|---|
| $L = \lambda W$ | Thread/connection pool sizing: 2,000 rps at 40 ms needs 80 concurrent slots |
| $W = L / \lambda$ | Infer latency from observed concurrency and rate |
| $\lambda = L / W$ | Max throughput given a hard concurrency cap: 100 connections at 50 ms caps you at 2,000 rps |

The dangerous property: $W$ is set by your dependencies. If a downstream service slows from 40 ms to 400 ms, required concurrency goes from 80 to 800 at *unchanged* traffic, and a fixed pool becomes 8x oversubscribed.

### M/M/1 waiting time

For Poisson arrivals, exponential service times, one server, service rate $\mu$, utilisation $\rho = \lambda / \mu$:

$$
W = \frac{1}{\mu - \lambda} \qquad W_q = \frac{\rho}{\mu - \lambda} = \frac{\rho}{\mu(1 - \rho)} \qquad L_q = \frac{\rho^2}{1 - \rho}
$$

The response-time inflation versus an idle system is the term everything hinges on:

$$
\frac{W}{W_{\rho \to 0}} = \frac{1}{1 - \rho}
$$

### The utilisation knee

| Utilisation $\rho$ | Latency multiplier $\frac{1}{1-\rho}$ | Queue length $L_q$ | Interpretation |
|---|---|---|---|
| 50% | 2.0x | 0.5 | Comfortable; headroom for a doubling |
| 70% | 3.3x | 1.6 | Normal steady-state target |
| 80% | 5.0x | 3.2 | The knee; past here, latency moves faster than load |
| 90% | 10x | 8.1 | One failed node away from collapse |
| 95% | 20x | 18 | Any jitter becomes a visible incident |
| 99% | 100x | 98 | Metastable; will not recover without shedding load |

!!! warning "Why 'we are only at 90% CPU' is not reassuring"
    Going from 80% to 90% utilisation is a 10% capacity gain and a 2x latency cost. Going from 90% to 95% is a 5% gain and another 2x. This is also why losing one node out of ten at 85% utilisation pushes the survivors to 94% and turns a capacity event into a latency incident — the knee, not the arithmetic, is what kills you.

Two important caveats to state if you use these formulas:

- **M/M/1 is pessimistic for multi-server pools.** An M/M/c system with many servers tolerates higher utilisation before the knee, because a free server is usually available. This is the theoretical basis for large shared pools beating many small ones.
- **Real traffic is burstier than Poisson.** Retries, cron alignment, and client-side batching create correlated arrivals, so the real knee arrives *earlier* than the table predicts. Retry storms are the classic mechanism: see [F18 Resilience Patterns](../fundamentals/f18-resilience-patterns.md).

### Tail latency and fan-out

If one request fans out to $n$ backends and must wait for all of them, the probability that at least one hits the slow tail is:

$$
P(\text{slow}) = 1 - (1 - p)^n
$$

At $p = 0.01$ (1% of calls are slow) and $n = 100$ fan-out, $P = 63\%$ — the p99 of one backend becomes the *median* of the composite request. This is the core argument of "The Tail at Scale" and the reason hedged requests, fan-out limits, and per-shard timeouts exist. See [S11 Debugging p99 Regression](../sre/s11-latency-debugging.md).

---

## Related Pages

| Topic | Page |
|---|---|
| Latency budgets and network behaviour | [F01 Networking Foundations](../fundamentals/f01-networking-foundations.md) |
| Cache hit-rate math and working sets | [F04 Caching](../fundamentals/f04-caching.md) |
| Shard counts and hot partitions | [F06 Partitioning & Sharding](../fundamentals/f06-partitioning-sharding.md) |
| Write amplification and storage sizing | [F13 Storage Engines](../fundamentals/f13-storage-engines.md) |
| Utilisation targets and headroom policy | [F24 Capacity Planning](../fundamentals/f24-capacity-planning.md) |
| Error budgets from availability targets | [F23 SLI/SLO & Error Budgets](../fundamentals/f23-slo-error-budgets.md) |
| Egress and unit-cost modelling | [F28 Cost Engineering](../fundamentals/f28-cost-engineering.md) |
| Cardinality estimation with sketches | [F21 Probabilistic Structures](../fundamentals/f21-probabilistic-data-structures.md) |
| Applying all of this to a growth plan | [S04 Capacity for 10x Growth](../sre/s04-capacity-planning-10x.md) |
