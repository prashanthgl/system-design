# Papers & Further Reading

**The primary sources behind almost every pattern on this site, named precisely enough to find by title and author — read the originals, because the interview signal comes from knowing what a design *costs*, and the costs are in the papers, not in the summaries.**

No URLs are given deliberately: titles, authors, venues, and years are stable and searchable, while links rot. Where an edition matters, it is stated.

---

## How to Read These for Interview Value

| Read for | What to extract | What to ignore |
|---|---|---|
| The problem statement | What constraint forced the design; usually in the first two pages | Related-work sections |
| The mechanism | The one or two ideas that do the work | Formal proofs unless the proof *is* the idea |
| The costs | What the authors admit is expensive, slow, or operationally painful; usually in evaluation and lessons-learned | Benchmark tables from obsolete hardware |
| The lessons | The section titled "Experience" or "Lessons learned"; the highest interview value per page in the entire literature | Future-work sections |

!!! tip "Quotable costs beat quotable architectures"
    Anyone can say "like Dynamo". A senior answer says "the Dynamo model buys me always-writeable behaviour and pays for it with vector clocks, client-side conflict resolution, anti-entropy, and hinted handoff, which are the operational burden". The second sentence only comes from reading the paper.

---

## Foundational Distributed Systems

| Paper / Source | Authors / Org | Why it matters | Related pages |
|---|---|---|---|
| "Dynamo: Amazon's Highly Available Key-value Store" (SOSP 2007) | DeCandia, Hastorun, Jampani, Kakulapati, Lakshman, Pilchin, Sivasubramanian, Vosshall, Vogels; Amazon | The origin of the always-writeable AP design: consistent hashing with virtual nodes, sloppy quorums, hinted handoff, vector clocks, Merkle-tree anti-entropy. Also the clearest statement that conflict resolution is an application concern. | [F07 Replication & Consistency](../fundamentals/f07-replication-consistency.md), [F06 Partitioning & Sharding](../fundamentals/f06-partitioning-sharding.md), [05 Key-Value Store](../case-studies/05-key-value-store.md) |
| "Bigtable: A Distributed Storage System for Structured Data" (OSDI 2006) | Chang, Dean, Ghemawat, Hsieh, Wallach, Burrows, Chandra, Fikes, Gruber; Google | The wide-column model: row key ordering, column families, tablets, SSTables and memtables over a distributed filesystem, with Chubby for metadata. Every wide-column store descends from this. | [F13 Storage Engines](../fundamentals/f13-storage-engines.md), [F14 SQL vs NoSQL](../fundamentals/f14-sql-vs-nosql.md), [10 Chat System](../case-studies/10-chat-system.md) |
| "The Google File System" (SOSP 2003) | Ghemawat, Gobioff, Leung; Google | Single-master metadata with chunk servers, append-optimised semantics, and the argument that relaxing the consistency model is what makes the system tractable. The template for control-plane and data-plane separation. | [F15 Object & Blob Storage](../fundamentals/f15-object-storage.md), [18 S3-style Object Store](../case-studies/18-object-store-s3.md) |
| "MapReduce: Simplified Data Processing on Large Clusters" (OSDI 2004) | Dean, Ghemawat; Google | Restricting the programming model to make fault tolerance automatic. The backup-task section is the original published treatment of stragglers and speculative execution. | [F12 Queues & Streams](../fundamentals/f12-queues-streams.md), [42 Ad Click Aggregation](../case-studies/42-ad-click-aggregation.md) |
| "The Chubby Lock Service for Loosely-Coupled Distributed Systems" (OSDI 2006) | Burrows; Google | Why teams want a lock service and actually need a small, consistent, highly available store for metadata and election. The operational sections on client caching, session leases, and the consequences of an outage are exceptional. | [F09 Consensus](../fundamentals/f09-consensus.md), [46 Coordination Service](../case-studies/46-coordination-service.md) |
| "Spanner: Google's Globally-Distributed Database" (OSDI 2012) | Corbett, Dean, Epstein, Fikes, Frost, Furman, Ghemawat, and others; Google | External consistency at global scale using TrueTime: bounded clock uncertainty converted into commit-wait. The proof that CAP does not forbid a globally consistent database, only a globally available one during partitions. | [F20 Time, Clocks & Ordering](../fundamentals/f20-time-clocks-ordering.md), [F10 Distributed Transactions](../fundamentals/f10-distributed-transactions.md), [F08 CAP & PACELC](../fundamentals/f08-cap-pacelc.md) |
| "Paxos Made Simple" (2001) | Leslie Lamport | The canonical consensus algorithm stated in a few pages. Read it for the invariant, not for an implementation: it tells you why a majority quorum is sufficient and why two-phase structures recur everywhere. | [F09 Consensus](../fundamentals/f09-consensus.md) |
| "In Search of an Understandable Consensus Algorithm" (Raft; USENIX ATC 2014) | Diego Ongaro, John Ousterhout; Stanford | Consensus decomposed into leader election, log replication, and safety, plus membership change and log compaction. The practical reference when asked how a replicated log actually works. | [F09 Consensus](../fundamentals/f09-consensus.md), [46 Coordination Service](../case-studies/46-coordination-service.md) |
| "Large-scale Incremental Processing Using Distributed Transactions and Notifications" (Percolator; OSDI 2010) | Daniel Peng, Frank Dabek; Google | Snapshot-isolation transactions layered over Bigtable with a timestamp oracle, plus incremental processing instead of full recomputation. The origin of the client-coordinated 2PC pattern used by several modern distributed SQL engines. | [F10 Distributed Transactions](../fundamentals/f10-distributed-transactions.md), [F19 Concurrency Control](../fundamentals/f19-concurrency-control.md) |
| "Kafka: A Distributed Messaging System for Log Processing" (NetDB 2011) | Jay Kreps, Neha Narkhede, Jun Rao; LinkedIn | The log as a primitive: partitioned append-only segments, consumer-managed offsets, pull-based consumption, and sequential I/O with zero-copy as the throughput argument. | [F12 Queues & Streams](../fundamentals/f12-queues-streams.md), [30 Message Queue](../case-studies/30-message-queue-kafka.md) |

### Also worth knowing

| Paper / Source | Authors / Org | Why it matters | Related pages |
|---|---|---|---|
| "Time, Clocks, and the Ordering of Events in a Distributed System" (CACM 1978) | Leslie Lamport | Logical clocks and the happens-before relation. Everything about causality, versioning, and conflict detection starts here. | [F20 Time, Clocks & Ordering](../fundamentals/f20-time-clocks-ordering.md) |
| "Consistent Hashing and Random Trees" (STOC 1997) | Karger, Lehman, Leighton, Panigrahy, Levine, Lewin; MIT | The original consistent-hashing construction, written for web caching. The reason adding a node moves only a fraction of keys. | [F06 Partitioning & Sharding](../fundamentals/f06-partitioning-sharding.md), [35 Distributed Cache](../case-studies/35-distributed-cache.md) |
| "Impossibility of Distributed Consensus with One Faulty Process" (FLP; JACM 1985) | Fischer, Lynch, Paterson | Why every practical consensus system uses timeouts and is therefore only eventually live. The formal backing for "failure detection is a heuristic". | [F09 Consensus](../fundamentals/f09-consensus.md) |
| "Harvest, Yield, and Scalable Tolerant Systems" (HotOS 1999) | Armando Fox, Eric Brewer | Degradation as a design dimension: return a partial result rather than an error. The intellectual basis for degraded modes. | [F18 Resilience Patterns](../fundamentals/f18-resilience-patterns.md), [S05 Load Shedding & Brownout](../sre/s05-load-shedding-brownout.md) |
| "ZooKeeper: Wait-free coordination for Internet-scale systems" (USENIX ATC 2010) | Hunt, Konar, Junqueira, Reed; Yahoo | The API shape that coordination primitives actually take: znodes, sessions, watches, ephemeral nodes, and why the guarantees are FIFO-per-client rather than linearizable reads by default. | [F09 Consensus](../fundamentals/f09-consensus.md), [46 Coordination Service](../case-studies/46-coordination-service.md) |
| "Dapper, a Large-Scale Distributed Systems Tracing Infrastructure" (Google technical report, 2010) | Sigelman, Barroso, Burrows, Stephenson, Plakal, Beaver, Jaspan, Shanbhag; Google | Low-overhead distributed tracing: propagation, sampling, and the argument that ubiquitous deployment matters more than completeness. | [F22 Observability](../fundamentals/f22-observability-fundamentals.md), [33 Distributed Tracing](../case-studies/33-distributed-tracing.md) |
| "A comprehensive study of Convergent and Commutative Replicated Data Types" (INRIA technical report, 2011) | Shapiro, Preguiça, Baquero, Zawirski | CRDTs: how to get convergence without coordination, and precisely which operations admit it. | [F07 Replication & Consistency](../fundamentals/f07-replication-consistency.md), [41 Collaborative Editor](../case-studies/41-collaborative-editor.md) |
| "Amazon Aurora: Design Considerations for High Throughput Cloud-Native Relational Databases" (SIGMOD 2017) | Verbitski, Gupta, Saha, Brahmadesam, and others; Amazon | "The log is the database": pushing redo processing into a distributed storage tier, and quorum design for a 4-of-6 write set. | [F13 Storage Engines](../fundamentals/f13-storage-engines.md), [F07 Replication & Consistency](../fundamentals/f07-replication-consistency.md) |

---

## Storage Engines

| Paper / Source | Authors / Org | Why it matters | Related pages |
|---|---|---|---|
| "The Log-Structured Merge-Tree (LSM-Tree)" (Acta Informatica, 1996) | Patrick O'Neil, Edward Cheng, Dieter Gawlick, Elizabeth O'Neil | The original: convert random writes into sequential writes by buffering in memory and merging on disk. The read-amplification and write-amplification trade-off you will be asked to articulate comes from here. | [F13 Storage Engines](../fundamentals/f13-storage-engines.md), [F14 SQL vs NoSQL](../fundamentals/f14-sql-vs-nosql.md) |
| RocksDB documentation and wiki | Facebook / Meta engineering | The most useful practical source on compaction strategies (level versus universal), write stalls, bloom filters per SSTable, column families, and tuning. Read the compaction and write-stall pages specifically. | [F13 Storage Engines](../fundamentals/f13-storage-engines.md), [05 Key-Value Store](../case-studies/05-key-value-store.md) |
| LevelDB documentation and design notes | Sanjay Ghemawat, Jeff Dean; Google | The minimal, readable LSM implementation. The `impl.md` design notes explain the memtable, immutable memtable, and level structure in fewer pages than any textbook. | [F13 Storage Engines](../fundamentals/f13-storage-engines.md) |
| "Gorilla: A Fast, Scalable, In-Memory Time Series Database" (VLDB 2015) | Pelkonen, Franklin, Teller, Cavallaro, Huang, Meza, Veeraraghavan; Facebook | Delta-of-delta timestamp encoding and XOR float compression giving roughly 12x reduction, plus the argument that a monitoring store should trade durability for availability. The reason modern TSDBs look the way they do. | [F13 Storage Engines](../fundamentals/f13-storage-engines.md), [31 Metrics & Monitoring](../case-studies/31-metrics-monitoring.md) |
| "The Ubiquitous B-Tree" (ACM Computing Surveys, 1979) | Douglas Comer | The other half of the storage dichotomy. Read it to be able to state the B-tree versus LSM trade-off in terms of read amplification, write amplification, and space amplification rather than as a preference. | [F13 Storage Engines](../fundamentals/f13-storage-engines.md) |
| "ARIES: A Transaction Recovery Method Supporting Fine-Granularity Locking and Partial Rollbacks Using Write-Ahead Logging" (ACM TODS, 1992) | Mohan, Haderle, Lindsay, Pirahesh, Schwarz; IBM | Why write-ahead logging with redo and undo is the standard, and what crash recovery actually does. Background for any question about durability guarantees. | [F13 Storage Engines](../fundamentals/f13-storage-engines.md), [F19 Concurrency Control](../fundamentals/f19-concurrency-control.md) |

---

## Reliability and Operations

This is the category that most distinguishes an SRE-track candidate. Prioritise it.

### Google SRE Book (2016)

*Site Reliability Engineering: How Google Runs Production Systems*, edited by Betsy Beyer, Chris Jones, Jennifer Petoff, Niall Richard Murphy; O'Reilly.

| Chapter | Why it matters | Related pages |
|---|---|---|
| Ch. 3, "Embracing Risk" | Reliability as a cost-balanced target rather than a maximisation goal. The error-budget argument in its original form. | [F23 SLI/SLO & Error Budgets](../fundamentals/f23-slo-error-budgets.md) |
| Ch. 4, "Service Level Objectives" | The SLI/SLO/SLA distinction and how to choose indicators that reflect user experience. | [S01 SLO Design](../sre/s01-slo-design.md) |
| Ch. 5, "Eliminating Toil" | The definition of toil and why it is capped rather than tolerated. | [S12 Auto-Remediation](../sre/s12-auto-remediation.md) |
| Ch. 6, "Monitoring Distributed Systems" | The four golden signals and the case against cause-based alerting. | [F22 Observability](../fundamentals/f22-observability-fundamentals.md) |
| Ch. 14, "Managing Incidents" | Incident command structure: roles, handoffs, and communication separation. | [S06 Incident Response System](../sre/s06-incident-response-system.md) |
| Ch. 15, "Postmortem Culture" | Blameless postmortems as a mechanism, including what makes them fail. | [S06 Incident Response System](../sre/s06-incident-response-system.md) |
| Ch. 19-20, frontend and datacenter load balancing | Anycast, connection-level balancing, and the weighted-round-robin versus least-request discussion. | [F03 Load Balancing](../fundamentals/f03-load-balancing.md), [F02 DNS & Global Traffic](../fundamentals/f02-dns-traffic-management.md) |
| Ch. 21, "Handling Overload" | Client-side throttling, criticality levels, and per-customer limits. The retry-budget idea is here. | [F17 Rate Limiting & Load Shedding](../fundamentals/f17-rate-limiting-load-shedding.md) |
| Ch. 22, "Addressing Cascading Failures" | The single most interview-relevant chapter in the book: queue growth, retry amplification, and why restarting a cascaded system does not fix it. | [F18 Resilience Patterns](../fundamentals/f18-resilience-patterns.md), [S05 Load Shedding & Brownout](../sre/s05-load-shedding-brownout.md) |
| Ch. 23, "Managing Critical State" | Consensus in production: replica placement, quorum latency, and when not to use a consensus system. | [F09 Consensus](../fundamentals/f09-consensus.md) |
| Ch. 24, "Distributed Periodic Scheduling with Cron" | Exactly-once-ish scheduling, leader election, and the at-least-once versus at-most-once choice for jobs. | [37 Job Scheduler](../case-studies/37-job-scheduler.md) |
| Ch. 25, "Data Processing Pipelines" | Why periodic batch pipelines develop thundering-herd and stale-work problems. | [42 Ad Click Aggregation](../case-studies/42-ad-click-aggregation.md) |
| Ch. 26, "Data Integrity" | Backups versus archives, and the insight that "restore was never tested" is the real failure. | [F26 Multi-Region & DR](../fundamentals/f26-multi-region-dr.md), [20 Backup & Deduplication](../case-studies/20-backup-dedup.md) |

### Google SRE Workbook (2018)

*The Site Reliability Workbook: Practical Ways to Implement SRE*, edited by Betsy Beyer, Niall Richard Murphy, David K. Rensin, Kent Kawahara, Stephen Thorne; O'Reilly.

| Chapter | Why it matters | Related pages |
|---|---|---|
| Ch. 2, "Implementing SLOs" | The mechanics the first book left abstract: choosing windows, aggregating, and writing the SLI specification. | [F23 SLI/SLO & Error Budgets](../fundamentals/f23-slo-error-budgets.md) |
| Ch. 5, "Alerting on SLOs" | Multi-window multi-burn-rate alerting, with the arithmetic. The best single source on this topic. | [34 Alerting & Paging](../case-studies/34-alerting-paging.md), [S01 SLO Design](../sre/s01-slo-design.md) |
| Ch. 11, "Managing Load" | Load balancing, shedding, and autoscaling treated as one control problem rather than three features. | [F17 Rate Limiting & Load Shedding](../fundamentals/f17-rate-limiting-load-shedding.md) |
| Ch. 12, "Introducing Non-Abstract Large System Design" | A worked capacity-and-design exercise with explicit arithmetic. The closest published analogue to a system design interview. | [F24 Capacity Planning](../fundamentals/f24-capacity-planning.md), [Numbers & Estimation](numbers.md) |
| Ch. 16, "Canarying Releases" | Canary population selection, metric comparison, and the statistics of deciding a canary is bad. | [F25 Deployment & Release Safety](../fundamentals/f25-deployment-release-safety.md), [S08 Deployment Safety](../sre/s08-deployment-safety.md) |

### Amazon Builders' Library

Short, dense, production-grounded articles published by Amazon engineers. Four are close to mandatory.

| Article | Author | Why it matters | Related pages |
|---|---|---|---|
| "Timeouts, retries, and backoff with jitter" | Marc Brooker; Amazon | The definitive treatment of retry storms, full versus decorrelated jitter, retry budgets as a fraction of traffic, and why a retry is a load-amplification decision. | [F18 Resilience Patterns](../fundamentals/f18-resilience-patterns.md), [F11 Idempotency](../fundamentals/f11-idempotency.md) |
| "Static stability using Availability Zones" | Becky Weiss, Mike Furr; Amazon | Pre-provision for the failed state so recovery requires no control-plane action. The single most useful multi-AZ design principle, and the reason data-plane survival must not depend on the control plane. | [F26 Multi-Region & DR](../fundamentals/f26-multi-region-dr.md), [S02 Multi-Region Active-Active](../sre/s02-multi-region-active-active.md) |
| "Avoiding fallback in distributed systems" | Jacob Gabrielson; Amazon | Fallback paths are cold code that runs only during incidents and therefore fails during incidents. Argues for making the primary path robust instead. | [F18 Resilience Patterns](../fundamentals/f18-resilience-patterns.md) |
| "Workload isolation using shuffle sharding" | Colm MacCárthaigh; Amazon | Combinatorial assignment of customers to node subsets so a single bad tenant cannot affect everyone. The mathematics of blast-radius reduction. | [S09 Cell-Based Architecture](../sre/s09-cell-based-architecture.md), [F17 Rate Limiting & Load Shedding](../fundamentals/f17-rate-limiting-load-shedding.md) |
| "Using load shedding to avoid overload" | David Yanacek; Amazon | Shedding at the edge based on measured queue time rather than CPU, and why accepting work you cannot finish is worse than rejecting it. | [S05 Load Shedding & Brownout](../sre/s05-load-shedding-brownout.md) |
| "Implementing health checks" | David Yanacek; Amazon | Shallow versus deep health checks, and the failure mode where deep checks take an entire fleet out of service simultaneously. | [F03 Load Balancing](../fundamentals/f03-load-balancing.md) |
| "Caching challenges and strategies" | Matt Brinkley, Jas Chhabra; Amazon | Modality changes when a cache fails, cache-induced bimodal behaviour, and why a cache that improves latency can destroy availability. | [F04 Caching](../fundamentals/f04-caching.md) |

### Books and individual papers

| Source | Author / Venue | Why it matters | Related pages |
|---|---|---|---|
| *Release It! Design and Deploy Production-Ready Software*, 2nd edition (2018) | Michael T. Nygard; Pragmatic Bookshelf | The stability patterns and antipatterns vocabulary: circuit breaker, bulkhead, timeout, steady state, fail fast, integration points, blocked threads, attacks of self-denial. The chapter on integration points is the best short account of how systems actually die. | [F18 Resilience Patterns](../fundamentals/f18-resilience-patterns.md), [Review Checklists](checklists.md) |
| "The Tail at Scale" (CACM, February 2013) | Jeffrey Dean, Luiz André Barroso; Google | Why p99 of a component becomes the median of a fan-out request, and the mitigations: hedged requests, tied requests, micro-partitioning, selective replication. Essential for any latency question. | [S11 Debugging p99 Regression](../sre/s11-latency-debugging.md), [Numbers & Estimation](numbers.md) |
| "Metastable Failures in Distributed Systems" (HotOS 2021) | Nathan Bronson, Abutalib Aghayev, Aleksey Charapko, Timothy Zhu | Names the class of outage where the system stays broken after the trigger is gone, sustained by a feedback loop such as retries. Explains why "traffic is back to normal but we are still down" happens and why shedding is the exit. | [F18 Resilience Patterns](../fundamentals/f18-resilience-patterns.md), [S05 Load Shedding & Brownout](../sre/s05-load-shedding-brownout.md) |
| *Designing Data-Intensive Applications* (2017) | Martin Kleppmann; O'Reilly | The best single synthesis of the storage and consistency literature. Chapters 5, 7, 9, and 11 map almost exactly onto the fundamentals sections of this site. | [F07 Replication & Consistency](../fundamentals/f07-replication-consistency.md), [F19 Concurrency Control](../fundamentals/f19-concurrency-control.md) |
| "Fallacies of Distributed Computing" (formulated at Sun Microsystems, 1990s) | L. Peter Deutsch, James Gosling, and others | Eight assumptions that are always wrong: the network is reliable, latency is zero, bandwidth is infinite, and so on. A fast self-review for any design. | [F01 Networking Foundations](../fundamentals/f01-networking-foundations.md) |

---

## Scaling Real Systems

Case-study papers from production systems. These are the most directly reusable in an interview because the authors state the numbers and the operational pain.

| Paper / Source | Authors / Org | Why it matters | Related pages |
|---|---|---|---|
| "Scaling Memcache at Facebook" (NSDI 2013) | Nishtala, Fugal, Grimm, Kwiatkowski, Lee, Li, McElroy, Paleczny, Peek, Saab, Stafford, Tung, Venkataramani; Facebook | The best published account of operating a cache tier at scale: lease-based stampede control, memcache pools by workload, regional invalidation, cold-cluster warmup, and the incast congestion problem. Almost every caching gotcha worth naming is in this paper. | [F04 Caching](../fundamentals/f04-caching.md), [35 Distributed Cache](../case-studies/35-distributed-cache.md) |
| "TAO: Facebook's Distributed Data Store for the Social Graph" (USENIX ATC 2013) | Bronson, Amsden, Cabrera, Chakka, Dimov, Ding, Ferris, Giardullo, Kulkarni, Li, Marchukov, Petrov, Puzar, Song, Venkataramani; Facebook | A read-optimised graph cache over sharded MySQL, with a two-tier cache hierarchy and deliberately weak consistency. The canonical answer to "how do you serve a social graph at read scale". | [F04 Caching](../fundamentals/f04-caching.md), [09 News Feed](../case-studies/09-news-feed.md) |
| "Maglev: A Fast and Reliable Software Network Load Balancer" (NSDI 2016) | Eisenbud, Yi, Contavalli, Smith, Kononov, Mann-Hielscher, Cilingiroglu, Cheyney, Shang, Hosein; Google | Consistent hashing with connection tracking so that backend changes do not break existing flows, plus ECMP and packet-level processing. The reference for L4 load balancing questions. | [F03 Load Balancing](../fundamentals/f03-load-balancing.md), [45 API Gateway & Service Mesh](../case-studies/45-api-gateway-service-mesh.md) |
| "The Akamai Network: A Platform for High-Performance Internet Applications" (ACM SIGOPS OSR, 2010) | Erik Nygren, Ramesh Sitaraman, Jennifer Sun; Akamai | How a real CDN does mapping, overlay routing, cache hierarchies, and origin offload. The vocabulary for any edge or CDN design. | [F05 CDN & Edge](../fundamentals/f05-cdn-edge.md), [36 CDN Design](../case-studies/36-cdn-design.md) |
| "Borg: Large-scale cluster management at Google" (EuroSys 2015) | Verma, Pedrosa, Korupolu, Oppenheimer, Tune, Wilkes; Google | Scheduling, priorities and quota, bin packing, task eviction, and the declarative job model that Kubernetes inherited. The lessons section on naming, allocation, and per-job isolation is the valuable part. | [38 Container Orchestrator](../case-studies/38-container-orchestrator.md), [F24 Capacity Planning](../fundamentals/f24-capacity-planning.md) |
| "Mesos: A Platform for Fine-Grained Resource Sharing in the Data Center" (NSDI 2011) | Hindman, Konwinski, Zaharia, Ghodsi, Joseph, Katz, Shenker, Stoica; UC Berkeley | The resource-offer model and two-level scheduling, which is the main published alternative to a monolithic scheduler. | [38 Container Orchestrator](../case-studies/38-container-orchestrator.md) |
| "The Dataflow Model" (VLDB 2015) | Akidau, Bradshaw, Chambers, Chernyak, Fernández-Moctezuma, Lax, McVeety, Mills, Perry, Schmidt, Whittle; Google | Event time versus processing time, windowing, watermarks, triggers, and accumulation modes. The correct framework for any "what about late data" question. | [F12 Queues & Streams](../fundamentals/f12-queues-streams.md), [42 Ad Click Aggregation](../case-studies/42-ad-click-aggregation.md) |
| "MillWheel: Fault-Tolerant Stream Processing at Internet Scale" (VLDB 2013) | Akidau, Balikov, Bekiroglu, Chernyak, Haberman, Lax, McVeety, Mills, Nordstrom, Whittle; Google | Exactly-once record processing via strong productions and idempotent state updates, with low-watermark propagation. | [F11 Idempotency](../fundamentals/f11-idempotency.md), [F12 Queues & Streams](../fundamentals/f12-queues-streams.md) |
| "Photon: Fault-tolerant and Scalable Joining of Continuous Data Streams" (SIGMOD 2013) | Ananthanarayanan, Basker, Das, Gupta, Jiang, Qiu, Reznichenko, Ryabkov, Singh, Venkataraman; Google | Joining ad clicks to queries across datacenters with exactly-once semantics using a replicated IdRegistry. The reference design for click attribution. | [42 Ad Click Aggregation](../case-studies/42-ad-click-aggregation.md) |
| "Lightweight Asynchronous Snapshots for Distributed Dataflows" (2015) | Carbone, Fóra, Ewen, Haridi, Tzoumas; Apache Flink | The Chandy-Lamport-derived barrier snapshotting that gives stream processors consistent checkpoints without stopping the world. | [F12 Queues & Streams](../fundamentals/f12-queues-streams.md) |
| "Distributed Snapshots: Determining Global States of Distributed Systems" (ACM TOCS, 1985) | K. Mani Chandy, Leslie Lamport | The original snapshot algorithm every checkpointing stream processor is built on. | [F20 Time, Clocks & Ordering](../fundamentals/f20-time-clocks-ordering.md) |
| "The Anatomy of a Large-Scale Hypertextual Web Search Engine" (WWW 1998) | Sergey Brin, Lawrence Page; Stanford | Crawling, the inverted index layout, barrels and lexicon, and PageRank. Read for the index structure, not the ranking. | [F16 Search & Indexing](../fundamentals/f16-search-indexing.md), [06 Web Crawler](../case-studies/06-web-crawler.md) |
| "Earlybird: Real-Time Search at Twitter" (ICDE 2012) | Busch, Gade, Larson, Lok, Luckenbill, Lin; Twitter | Mutable, memory-resident inverted index segments for real-time indexing, and the single-writer-multiple-reader concurrency trick. | [F16 Search & Indexing](../fundamentals/f16-search-indexing.md), [12 Distributed Search](../case-studies/12-distributed-search.md) |
| "Firecracker: Lightweight Virtualization for Serverless Applications" (NSDI 2020) | Agache, Brooker, Iordache, Liguori, Neugebauer, Piwonka, Popa; Amazon | MicroVM isolation with millisecond boot, and the argument that multi-tenant isolation must be a hardware boundary rather than a process boundary. | [F27 Security in Design](../fundamentals/f27-security-design.md), [48 ML Inference Serving](../case-studies/48-ml-inference-serving.md) |
| "BBR: Congestion-Based Congestion Control" (ACM Queue, 2016) | Cardwell, Cheng, Gunn, Yeganeh, Jacobson; Google | Modelling bottleneck bandwidth and round-trip propagation instead of treating loss as congestion. Explains why bufferbloat hurts tail latency. | [F01 Networking Foundations](../fundamentals/f01-networking-foundations.md) |
| "The QUIC Transport Protocol: Design and Internet-Scale Deployment" (SIGCOMM 2017) | Langley, Riddoch, Wilk, Vicente, Krasic, Zhang, and others; Google | Why HTTP/3 exists: 0-RTT connection establishment, per-stream loss recovery without head-of-line blocking, and deployment at internet scale over UDP. | [F01 Networking Foundations](../fundamentals/f01-networking-foundations.md), [F05 CDN & Edge](../fundamentals/f05-cdn-edge.md) |

!!! note "Case-study papers are the best source of gotchas"
    Architecture papers tell you what was built; production papers tell you what went wrong. The Memcache paper's lease mechanism, the Borg paper's eviction and quota sections, and the Akamai paper's origin-offload numbers are all directly quotable when an interviewer asks what breaks first.

---

## Probabilistic Data Structures

| Paper / Source | Authors / Venue | Why it matters | Related pages |
|---|---|---|---|
| "Space/Time Trade-offs in Hash Coding with Allowable Errors" (CACM, 1970) | Burton H. Bloom | The Bloom filter. Read it for the false-positive rate derivation so you can size one on a whiteboard: $m$ bits, $k$ hashes, $n$ items, $p \approx (1 - e^{-kn/m})^k$, with the optimal $k = (m/n)\ln 2$. | [F21 Probabilistic Structures](../fundamentals/f21-probabilistic-data-structures.md), [06 Web Crawler](../case-studies/06-web-crawler.md) |
| "HyperLogLog: the analysis of a near-optimal cardinality estimation algorithm" (AOFA 2007) | Philippe Flajolet, Éric Fusy, Olivier Gandouet, Frédéric Meunier | Distinct-count estimation in kilobytes with roughly 2% error, and the mergeability property that makes distributed aggregation possible. | [F21 Probabilistic Structures](../fundamentals/f21-probabilistic-data-structures.md), [43 Web Analytics](../case-studies/43-web-analytics.md) |
| "HyperLogLog in Practice: Algorithmic Engineering of a State of the Art Cardinality Estimation Algorithm" (EDBT 2013) | Stefan Heule, Marc Nunkesser, Alexander Hall; Google | The corrections that made HLL usable: sparse representation, bias correction at low cardinality, 64-bit hashing. Cite this when asked about accuracy at small counts. | [F21 Probabilistic Structures](../fundamentals/f21-probabilistic-data-structures.md) |
| "An Improved Data Stream Summary: The Count-Min Sketch and its Applications" (Journal of Algorithms, 2005) | Graham Cormode, S. Muthukrishnan | Frequency estimation in sublinear space with one-sided error. The standard answer to heavy-hitter and hot-key detection questions. | [F21 Probabilistic Structures](../fundamentals/f21-probabilistic-data-structures.md), [03 Rate Limiter](../case-studies/03-rate-limiter.md), [44 Leaderboard](../case-studies/44-leaderboard.md) |

---

## Security

| Paper / Source | Authors / Org | Why it matters | Related pages |
|---|---|---|---|
| "Zanzibar: Google's Consistent, Global Authorization System" (USENIX ATC 2019) | Pang, Caceres, Burrows, Chen, Dave, Germer, Golynski, Graney, Kang, Kissner, Korn, Parmar, Richards, Wang; Google | Relationship-based authorisation as a global service: the relation-tuple model, namespace configuration, and "zookies" as consistency tokens that let clients demand a snapshot at least as fresh as a prior write. The canonical reference for any authorisation design question. | [F27 Security in Design](../fundamentals/f27-security-design.md), [17 Google Drive](../case-studies/17-google-drive.md) |
| OWASP Top 10 (OWASP Foundation; current edition 2021, periodically revised) | OWASP Foundation | The shared vocabulary for application security review: broken access control, cryptographic failures, injection, insecure design, security misconfiguration, and the rest. Use the category names directly in a design review. | [F27 Security in Design](../fundamentals/f27-security-design.md), [Review Checklists](checklists.md) |
| OWASP Application Security Verification Standard (ASVS) | OWASP Foundation | Where the Top 10 is a risk list, ASVS is a requirements checklist with levels. More useful than the Top 10 when you actually need to specify controls. | [F27 Security in Design](../fundamentals/f27-security-design.md) |
| "BeyondCorp: A New Approach to Enterprise Security" (USENIX ;login:, 2014, and subsequent papers) | Ward, Beyer, and others; Google | The zero-trust model stated concretely: no network-location trust, device and user identity on every request. Background for service-to-service authentication questions. | [F27 Security in Design](../fundamentals/f27-security-design.md), [47 Secrets Management](../case-studies/47-secrets-management.md) |

---

## Specifications Worth Reading Once

Not papers, but the normative text behind decisions people argue about from memory.

| Specification | Body / Year | Why it matters | Related pages |
|---|---|---|---|
| RFC 9110, HTTP Semantics | IETF, 2022 | The authoritative definition of method safety and idempotency, conditional requests, and status-code semantics. Settles most API-design disagreements. | [F11 Idempotency](../fundamentals/f11-idempotency.md), [Review Checklists](checklists.md) |
| RFC 9111, HTTP Caching | IETF, 2022 | Cache-control directives, freshness, validators, and the vocabulary for edge caching. | [F04 Caching](../fundamentals/f04-caching.md), [F05 CDN & Edge](../fundamentals/f05-cdn-edge.md) |
| RFC 5861, stale-while-revalidate and stale-if-error | IETF, 2010 | The standardised mechanism for serving stale content during origin failure: a degraded mode you get for free at the edge. | [F05 CDN & Edge](../fundamentals/f05-cdn-edge.md), [36 CDN Design](../case-studies/36-cdn-design.md) |
| RFC 8446, TLS 1.3 | IETF, 2018 | Why the handshake is one round trip, what 0-RTT costs in replay risk, and which ciphers remain. | [F01 Networking Foundations](../fundamentals/f01-networking-foundations.md), [F27 Security in Design](../fundamentals/f27-security-design.md) |
| RFC 9000, QUIC | IETF, 2021 | Transport-level streams, connection migration, and loss recovery without head-of-line blocking. | [F01 Networking Foundations](../fundamentals/f01-networking-foundations.md) |
| RFC 6749 and RFC 6750, OAuth 2.0 | IETF, 2012 | Delegated authorisation, correctly separated from authentication. Read alongside OpenID Connect. | [F27 Security in Design](../fundamentals/f27-security-design.md) |
| RFC 7519, JSON Web Token | IETF, 2015 | Token structure, the claims that matter, and why validating the algorithm field is non-optional. | [F27 Security in Design](../fundamentals/f27-security-design.md), [47 Secrets Management](../case-studies/47-secrets-management.md) |
| RFC 9457, Problem Details for HTTP APIs | IETF, 2023 | A standard machine-readable error body, which removes an entire category of API-review debate. | [45 API Gateway & Service Mesh](../case-studies/45-api-gateway-service-mesh.md) |

---

## Ongoing Sources

| Source | What it is | Why it is worth the time |
|---|---|---|
| USENIX OSDI, NSDI, ATC, FAST proceedings | Open-access systems conferences | Where most of the papers above were published; proceedings are free |
| USENIX SREcon talks | Practitioner conference | The operational content that rarely reaches paper form: overload, on-call, migrations |
| SIGMOD and VLDB proceedings | Database conferences | Storage engines, transactions, stream processing |
| HotOS workshop papers | Short position papers | Fast reads that name emerging failure classes, such as metastability |
| Jepsen analyses, Kyle Kingsbury | Independent consistency testing reports | Shows what distributed databases actually guarantee versus what they claim; excellent vocabulary for consistency anomalies |
| Netflix, Cloudflare, Discord, Stripe, Uber engineering blogs | Production write-ups | Concrete numbers and post-incident detail at scales you can cite |
| Marc Brooker's blog | Amazon principal engineer | Queueing, retries, and the economics of reliability, written for practitioners |

---

## The Claims Worth Memorising

One quotable, load-bearing fact from each of the highest-value sources.

| Source | The claim to remember |
|---|---|
| "The Tail at Scale" | In a fan-out to 100 servers where 1% of calls are slow, 63% of composite requests are slow; the component's p99 becomes the request's median |
| Dynamo | Amazon specified its internal SLAs at the 99.9th percentile rather than the mean, because averages hide the customers with the most data |
| Spanner | TrueTime exposes a bounded uncertainty interval, typically single-digit milliseconds, and transactions pay for it by waiting out that bound before committing |
| Gorilla | Delta-of-delta plus XOR float encoding compressed time-series points by roughly 12x, which is what made an all-in-memory monitoring store affordable |
| "Scaling Memcache at Facebook" | Leases solve both the stale-set race and the thundering herd with one mechanism, and cold clusters must warm from a warm cluster rather than from the database |
| Google SRE Book, Ch. 22 | A cascading failure will not resolve itself when load returns to normal; you must shed load or drop traffic to break the feedback loop |
| "Metastable Failures in Distributed Systems" | The trigger and the sustaining feedback loop are different things; removing the trigger does not end the outage |
| "Timeouts, retries, and backoff with jitter" | Retries are a load-amplification decision; without jitter and a retry budget, the retry policy becomes the outage |
| "Static stability using Availability Zones" | Provision for the failed state in advance, so that recovery requires no control-plane action during the failure |
| MapReduce | Stragglers, not failures, dominate job completion time; backup tasks are the fix |
| Raft | A leader never overwrites its own log; all log repair flows from leader to follower, which is what makes the algorithm explainable |
| Zanzibar | A consistency token returned on write lets a client demand a read snapshot no older than its own change, avoiding the "new ACL, stale check" bug |

---

## Transactions, Isolation and Concurrency

| Paper / Source | Authors / Venue | Why it matters | Related pages |
|---|---|---|---|
| "A Critique of ANSI SQL Isolation Levels" (SIGMOD 1995) | Berenson, Bernstein, Gray, Melton, O'Neil, O'Neil | Shows that the ANSI levels are underspecified and names the anomalies precisely, including snapshot isolation and write skew. Read this before claiming a system is "isolated". | [F19 Concurrency Control](../fundamentals/f19-concurrency-control.md) |
| "Serializable Isolation for Snapshot Databases" (SIGMOD 2008) | Michael Cahill, Uwe Röhm, Alan Fekete | How serializable snapshot isolation detects dangerous structures at runtime rather than locking. The mechanism behind PostgreSQL's serializable mode. | [F19 Concurrency Control](../fundamentals/f19-concurrency-control.md) |
| "Highly Available Transactions: Virtues and Limitations" (VLDB 2014) | Peter Bailis, Aaron Davidson, Alan Fekete, Ali Ghodsi, Joseph Hellerstein, Ion Stoica | Exactly which isolation levels are achievable without coordination during a partition, and which are not. The precise version of the CAP conversation. | [F08 CAP & PACELC](../fundamentals/f08-cap-pacelc.md), [F10 Distributed Transactions](../fundamentals/f10-distributed-transactions.md) |
| "Coordination Avoidance in Database Systems" (VLDB 2015) | Bailis, Fekete, Franklin, Ghodsi, Hellerstein, Stoica | Invariant confluence: a formal test for whether an application invariant actually requires coordination. The rigorous form of "redesign to avoid the distributed transaction". | [F10 Distributed Transactions](../fundamentals/f10-distributed-transactions.md), [Decision Trees](decision-trees.md) |
| "Calvin: Fast Distributed Transactions for Partitioned Database Systems" (SIGMOD 2012) | Alexander Thomson, Thaddeus Diamond, Shu-Chun Weng, Kun Ren, Philip Shao, Daniel Abadi | Deterministic execution of a globally agreed transaction order, removing the need for two-phase commit. A genuinely different answer to the distributed-transaction question. | [F10 Distributed Transactions](../fundamentals/f10-distributed-transactions.md) |
| "Sagas" (SIGMOD 1987) | Hector Garcia-Molina, Kenneth Salem | The original long-lived-transaction pattern with compensating actions, from long before microservices existed. | [F10 Distributed Transactions](../fundamentals/f10-distributed-transactions.md), [26 E-commerce Checkout](../case-studies/26-ecommerce-checkout.md) |
| "Life beyond Distributed Transactions: an Apostate's Opinion" (CIDR 2007) | Pat Helland; Amazon and Microsoft | Entities, activities, and the argument that at scale you give up distributed transactions and manage uncertainty in the application instead. Short and quotable. | [F10 Distributed Transactions](../fundamentals/f10-distributed-transactions.md), [F11 Idempotency](../fundamentals/f11-idempotency.md) |
| "Immutability Changes Everything" (CIDR 2015) | Pat Helland | Why append-only data, versioning, and derived views simplify distributed systems, and why storage economics made it practical. | [F13 Storage Engines](../fundamentals/f13-storage-engines.md), [F12 Queues & Streams](../fundamentals/f12-queues-streams.md) |

---

## Using a Paper in an Answer

Citing prior art is a signal only when it carries a constraint with it. The difference:

| Weak use | Strong use |
|---|---|
| "I would use a Dynamo-style store here." | "I would use a Dynamo-style quorum store because the workload is key-addressed and must stay writeable during a partition. The price is conflict resolution in the application and anti-entropy repair as operational load, and I accept that because the item is a shopping cart where merge-on-read is acceptable." |
| "We would use Spanner for consistency." | "Global external consistency needs bounded clock uncertainty, and the commit-wait that implies adds latency proportional to that bound. That is worth paying on the ledger path and not worth paying on the feed path." |
| "We would add retries." | "Retries without jitter and a budget are how this becomes an outage; I would cap total retries as a fraction of traffic, use decorrelated jitter, and require an idempotency key on the write path." |
| "We would cache it." | "A cache changes the failure mode from slow to bimodal: full speed when warm, origin-crushing when cold. So I want leases for the stampede case, jittered TTLs, and a warm-up path that does not read through to the database." |

!!! warning "Do not cite a paper you have not read"
    Naming a paper invites a follow-up question about its mechanism. The downside of a wrong answer there is larger than the upside of the reference. Cite the four or five you actually know, and describe the rest as patterns.

---

## A Reading Order

If the list is intimidating, this sequence gives the most interview value per hour.

| Order | Source | Why first |
|---|---|---|
| 1 | Google SRE Book, Ch. 22, "Addressing Cascading Failures" | Explains the failure mode you will be asked about most often, and teaches the vocabulary for the rest |
| 2 | "Timeouts, retries, and backoff with jitter", Amazon Builders' Library | Twenty minutes; changes how you answer every reliability follow-up |
| 3 | "The Tail at Scale" | Reframes latency from an average to a fan-out problem |
| 4 | Dynamo | The clearest paper on what availability actually costs |
| 5 | Google SRE Workbook, Ch. 2 and Ch. 5 | Makes SLO answers concrete rather than hand-waved |
| 6 | "Static stability using Availability Zones" | The principle behind almost every good multi-AZ answer |
| 7 | Bigtable, then the LSM-tree paper | Storage engine trade-offs stated from the source |
| 8 | Raft | Lets you describe a replicated log correctly under questioning |
| 9 | "Metastable Failures in Distributed Systems" | Six pages; names a failure class most candidates cannot name |
| 10 | Spanner | The strongest available answer to "can you have global consistency" |

---

## Related Pages

| Topic | Page |
|---|---|
| The arithmetic these systems are sized with | [Numbers & Estimation](numbers.md) |
| Where each paper's idea appears as a choice | [Decision Trees](decision-trees.md) |
| Turning the reliability sources into review items | [Review Checklists](checklists.md) |
| How to cite prior art without name-dropping | [Interview Framework](interview-framework.md) |
