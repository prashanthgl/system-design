# Review Checklists

**Seven checklists to run against a design before you stop talking, before a design doc goes out, and before a service takes production traffic — every item is phrased so that the answer is verifiably yes or no.**

A checklist is not a substitute for thinking; it is a defence against the specific failure of forgetting something you already know. Run the relevant list in the last three minutes of a design round, and the whole list before a real launch.

---

## Before You Start Designing

- [ ] I can state the single most important user action this system performs, in one sentence.
- [ ] Functional requirements are written down as a numbered list, not held in my head.
- [ ] Non-goals are written down explicitly and were agreed, not assumed.
- [ ] I know the daily active user count and the actions per user per day, or I have stated the number I am assuming.
- [ ] I have converted those into average QPS and peak QPS with a named peak-to-average multiplier.
- [ ] I know the read-to-write ratio and have said which of the two is the harder problem.
- [ ] I have an estimate for stored bytes per year, including replication and index overhead.
- [ ] I know the p99 latency target for the critical read path and the critical write path, as numbers.
- [ ] I know the availability target as a number, and I have converted it into an error budget in minutes per month.
- [ ] I know the durability requirement: whether losing a single acknowledged write is acceptable, and if so how many.
- [ ] I know the consistency requirement per data path, not as a single blanket statement for the system.
- [ ] I know the retention requirement and whether data can be deleted or tiered.
- [ ] I know where users are geographically and whether this is single-region or multi-region.
- [ ] I know which existing systems I should reuse rather than introduce a new one.
- [ ] I know of any compliance constraint that affects data placement, encryption, or audit.
- [ ] I have named the binding constraint of the whole design in one sentence before drawing anything.

!!! tip "The single highest-value item on this list"
    Naming the binding constraint. "This is a read-fan-out problem, not a write-throughput problem" reframes the next thirty minutes and is the difference between a design that answers the question and one that answers a different question well.

See [Numbers & Estimation](numbers.md) and [Interview Framework](interview-framework.md).

---

## API Design Review

### Semantics and correctness

- [ ] Every mutating endpoint is either naturally idempotent or accepts a client-supplied idempotency key.
- [ ] The idempotency key has a documented retention window, and the behaviour after that window expires is defined.
- [ ] A retried request with the same idempotency key and a *different* body returns an error rather than silently applying one of them.
- [ ] Every endpoint documents whether it is synchronous or returns an acknowledgement plus a status handle.
- [ ] Asynchronous operations expose a status resource with defined terminal states and a TTL.
- [ ] Read-after-write expectations are documented per endpoint, including whether the caller may see its own write immediately.

### Pagination and result shape

- [ ] List endpoints are paginated; none can return an unbounded result set.
- [ ] Pagination is cursor-based rather than offset-based wherever the underlying set can change between pages.
- [ ] The cursor is opaque to clients and its encoding can change without a version bump.
- [ ] A maximum page size is enforced server-side, not merely documented.
- [ ] Deep pagination has a defined limit or an alternative export path rather than degrading silently.

### Versioning and evolution

- [ ] The versioning strategy is chosen and stated: URI path, header, or additive-only with feature negotiation.
- [ ] Adding a field is a non-breaking change, and clients are documented as required to ignore unknown fields.
- [ ] Removing or repurposing a field has a deprecation process with a stated minimum notice period.
- [ ] There is a way to measure which clients still use a deprecated field or endpoint.
- [ ] Enum values can be added without breaking existing clients, and unknown values have defined client behaviour.

### Errors and limits

- [ ] Error responses distinguish retryable from non-retryable conditions unambiguously.
- [ ] Retryable errors include a retry hint such as `Retry-After`, and clients are expected to honour it.
- [ ] The error body has a stable machine-readable code separate from the human-readable message.
- [ ] Rate limits are documented per endpoint with the limit, the window, and the identity the limit is keyed on.
- [ ] Rate-limit responses return the remaining quota and reset time in headers.
- [ ] Request size limits, timeout values, and maximum batch sizes are documented, not discovered in production.
- [ ] Partial failure in a batch endpoint has defined semantics: all-or-nothing, or per-item status.
- [ ] Long-running or expensive queries are bounded by a server-side deadline that is shorter than the client's timeout.
- [ ] Clients that ignore rate limits or retry aggressively can be identified and throttled per identity.

See [F11 Idempotency & Exactly-Once](../fundamentals/f11-idempotency.md), [F17 Rate Limiting & Load Shedding](../fundamentals/f17-rate-limiting-load-shedding.md), and [45 API Gateway & Service Mesh](../case-studies/45-api-gateway-service-mesh.md).

---

## Data Model Review

- [ ] Every table or collection has a written list of the queries it serves, and each query is satisfied by a key or an index.
- [ ] The schema was derived from the access patterns, not from an entity-relationship instinct.
- [ ] The partition key is stated explicitly for every partitioned dataset.
- [ ] I have written down what happens to the partition key distribution when one entity becomes 1,000x more active than the median.
- [ ] No partition key is monotonically increasing unless the write hotspot on the newest partition is explicitly accepted or salted.
- [ ] The maximum size of a single partition is bounded and I know the bound in bytes and in row count.
- [ ] Every secondary index is justified by a named query, and the write-amplification cost of each is accepted.
- [ ] Queries that fan out to all shards are enumerated, and their cost at full scale is estimated.
- [ ] Denormalised copies have a named source of truth and a documented rebuild procedure.
- [ ] Derived stores such as search indexes and caches have a stated maximum staleness and a way to detect divergence.
- [ ] Large blobs are stored in object storage with only references in the database.
- [ ] The row or item size is estimated, and it is well under the store's per-item limit at the 99th percentile.
- [ ] Deletes are handled explicitly: hard delete, soft delete with a purge job, or TTL, with the tombstone cost understood.
- [ ] Time-ordered data has a retention and tiering plan rather than unbounded growth.
- [ ] Schema migration for this model is possible without downtime, via expand-and-contract.
- [ ] Unique constraints that span partitions are identified, and the mechanism enforcing them is named.
- [ ] Counters and aggregates that would become write hotspots are either sharded or computed from an event log.
- [ ] The storage estimate includes replication factor, index overhead, and free-space headroom, not just logical bytes.
- [ ] Time zones and timestamp storage are specified: UTC at rest, with the source of truth for the clock named.
- [ ] Identifier generation is specified: whether IDs are sequential, random, or time-sortable, and what that implies for partitioning and for information leakage.

!!! warning "The hot partition question is not optional"
    Every real system has a celebrity key: the viral post, the largest tenant, the most-traded symbol, the busiest city. A data model review that does not name that key and its mitigation is incomplete. See [F06 Partitioning & Sharding](../fundamentals/f06-partitioning-sharding.md).

---

## Failure Mode Review

### Single points of failure

- [ ] Every component in the request path has a stated redundancy model, including the ones that feel like infrastructure.
- [ ] Coordination services, config stores, and service discovery are included in that list.
- [ ] The failure of any one availability zone leaves enough capacity to serve peak traffic, and the arithmetic is written down.
- [ ] No component requires a human decision to fail over within the availability target's error budget.
- [ ] Shared dependencies that would fail together are identified, so "redundant" is not merely two instances of the same fate.

### Timeouts, retries, and budgets

- [ ] Every outbound call has an explicit timeout; none inherit a library default.
- [ ] Each timeout is derived from the measured p99 of that dependency, and the derivation is recorded.
- [ ] Timeouts decrease along the call chain, so an inner call cannot outlive its caller's deadline.
- [ ] A request deadline is propagated across service boundaries and is honoured by downstream services.
- [ ] Retries have a maximum attempt count and an overall deadline, not just a per-attempt timeout.
- [ ] Retries use exponential backoff with jitter, and the jitter is full or decorrelated rather than a fixed additive value.
- [ ] Retries are only issued for idempotent operations or for operations carrying an idempotency key.
- [ ] There is a retry budget, so total retries are capped as a fraction of total requests rather than per-client.
- [ ] Retry amplification through multiple layers is bounded; the worst-case multiplier is computed.

### Cascades and overload

- [ ] Circuit breakers exist on calls to dependencies that can be slow, with named open and close conditions.
- [ ] The system has admission control that rejects work at the edge before queues grow unbounded.
- [ ] Every internal queue and pool is bounded, and the behaviour at the bound is reject rather than grow.
- [ ] The system sheds load by priority, and the priority classes are written down.
- [ ] The behaviour when a dependency is *slow* rather than *down* has been analysed separately, because it is the harder case.
- [ ] A metastable failure scenario has been considered: whether the system can recover on its own after load returns to normal.
- [ ] Cold-start behaviour is defined: empty caches, cold JIT, connection storms, and the traffic ramp that avoids them.

### Degraded mode and data loss

- [ ] A degraded mode is defined and named for each major dependency loss, with what the user sees in each.
- [ ] Serving stale data is explicitly allowed or forbidden per data path, and the staleness bound is stated.
- [ ] The maximum data loss during the worst credible failure is stated as an RPO, in seconds.
- [ ] The maximum recovery time is stated as an RTO, and it has been measured, not assumed.
- [ ] Backups are tested by restore, on a stated schedule, not merely taken.
- [ ] Corruption is treated separately from loss: there is a path to recover from bad data that has been replicated everywhere.
- [ ] The failure of the failover mechanism itself has been considered.
- [ ] The data plane continues serving when the control plane is unavailable, and that independence is stated explicitly.
- [ ] Fallback paths, if any, are exercised in normal operation rather than only during incidents.
- [ ] Dependency health checks distinguish "this instance is unhealthy" from "the dependency everyone shares is unhealthy", so a deep check cannot remove the entire fleet at once.

See [F18 Resilience Patterns](../fundamentals/f18-resilience-patterns.md), [F26 Multi-Region & DR](../fundamentals/f26-multi-region-dr.md), and [S07 Chaos Engineering Platform](../sre/s07-chaos-engineering-platform.md).

---

## SRE and Operability Review

### Measurement

- [ ] SLIs are defined from the user's perspective: availability and latency measured where the user experiences them.
- [ ] Each SLI has a written specification including what counts as a valid event and what counts as good.
- [ ] SLO targets are numbers with a stated measurement window, and they were chosen from user impact rather than from a round number.
- [ ] The error budget in minutes per month is computed and someone has agreed to it.
- [ ] There is a policy for what happens when the error budget is exhausted.
- [ ] Latency is reported as percentiles, never as an average, and p99 and p999 are distinguished.
- [ ] Percentiles are computed from histograms rather than averaged across instances.

### Alerting

- [ ] Every page alerts on a user-visible symptom, not on a cause such as CPU or memory.
- [ ] Every page is actionable: a human receiving it at 3 a.m. can do something specific.
- [ ] Cause-based signals exist as dashboards or tickets, not as pages.
- [ ] Alert thresholds are derived from the SLO burn rate, with fast-burn and slow-burn windows.
- [ ] There is an alert for the absence of data, so a broken pipeline does not look like health.
- [ ] Alert volume is tracked, and the page-per-shift rate is within the team's stated tolerance.

### Observability

- [ ] The four signals exist for every service: request rate, error rate, duration, and saturation.
- [ ] Saturation has a concrete definition per component, such as queue depth or pool utilisation, not just CPU.
- [ ] Distributed traces cover the critical path end to end, with a stated sampling strategy.
- [ ] Logs carry a request or trace identifier that correlates them to traces and to user reports.
- [ ] Metric cardinality is bounded, and there is a guard against a new label exploding the time-series count.
- [ ] A new engineer can determine which dependency is slow within five minutes using existing dashboards.

### Capacity and change

- [ ] Current utilisation is known per tier and compared against a stated headroom target.
- [ ] Headroom is sufficient to absorb the loss of the largest failure domain at peak, with the arithmetic written down.
- [ ] The lead time to acquire more capacity is known, including quota limits and instance-type availability.
- [ ] A load test exists that reaches the documented capacity limit, and the limit came from that test.
- [ ] Deployments are progressive: canary or staged, with a defined promotion metric.
- [ ] Rollback is a tested procedure with a known duration, and it does not depend on the thing being rolled back.
- [ ] Schema and data migrations are backward compatible for at least one release, so code rollback is safe.
- [ ] A runbook exists for each alert, with the diagnosis steps and the remediation, and it has been used at least once.
- [ ] Toil in operating this service is estimated in hours per week and has an owner.
- [ ] Configuration changes follow the same progressive rollout and rollback path as code changes.
- [ ] There is a documented way to disable any new feature without a deploy, and the flag's default is the safe value.
- [ ] The service can be restarted cleanly under load: connection draining, in-flight request handling, and a ramp that avoids a cold-cache stampede.

See [F22 Observability](../fundamentals/f22-observability-fundamentals.md), [F23 SLI/SLO & Error Budgets](../fundamentals/f23-slo-error-budgets.md), [F24 Capacity Planning](../fundamentals/f24-capacity-planning.md), [F25 Deployment & Release Safety](../fundamentals/f25-deployment-release-safety.md), and [S06 Incident Response System](../sre/s06-incident-response-system.md).

---

## Security Review

### Identity and access

- [ ] Every endpoint states who may call it, and authentication is enforced at a chokepoint rather than per handler.
- [ ] Authorisation is a separate, explicit decision from authentication, and it is evaluated per resource, not per route.
- [ ] Service-to-service calls are authenticated with workload identity, not with a shared static token.
- [ ] Authorisation decisions are made server-side; no client-supplied field determines access.
- [ ] Object-level authorisation is checked on every read path, including list endpoints and batch fetches.
- [ ] Privilege boundaries are written down: which components can read user data, which can write it, which can delete it.
- [ ] Admin and support access paths are audited, time-bounded, and require a reason.

### Secrets and keys

- [ ] No secret is present in source control, container images, environment dumps, or CI logs.
- [ ] Secrets are fetched from a secret manager at runtime with a short-lived credential.
- [ ] Every secret has a rotation procedure and a stated rotation period, and rotation does not require downtime.
- [ ] Application logs are verified not to contain credentials, tokens, full card numbers, or personal identifiers.
- [ ] Error messages returned to clients do not leak internal hostnames, stack traces, or query fragments.

### Data protection

- [ ] Data is encrypted in transit between every pair of components, including inside the datacenter.
- [ ] Data is encrypted at rest, and the key management responsibility is stated explicitly.
- [ ] Personally identifiable data is enumerated, and its storage locations and retention periods are documented.
- [ ] Deletion requests propagate to backups, caches, search indexes, and analytics copies, with a stated maximum delay.
- [ ] Data residency constraints are enforced by the partitioning scheme, not by convention.

### Abuse and isolation

- [ ] Rate limiting exists as an abuse control at the edge, keyed on an identity that an attacker cannot trivially rotate.
- [ ] Expensive operations have stricter limits than cheap ones, and the cost classes are defined.
- [ ] Input is validated against a schema at the boundary, with explicit size and depth limits on nested structures.
- [ ] Multi-tenant isolation is stated: shared, cell-partitioned, or fully isolated per tenant, with the blast radius of each.
- [ ] One tenant cannot exhaust a shared resource and degrade others, and the mechanism preventing it is named.
- [ ] Untrusted user content served to browsers has content-type, sandboxing, and origin controls defined.
- [ ] The design has been walked against the OWASP Top 10 categories relevant to it.

See [F27 Security in Design](../fundamentals/f27-security-design.md), [47 Secrets Management](../case-studies/47-secrets-management.md), and [S09 Cell-Based Architecture](../sre/s09-cell-based-architecture.md).

---

## Cost Review

- [ ] The unit economics are known: cost per request, per user per month, or per GB stored, as a number.
- [ ] The dominant cost line item is identified, and it is usually not what the team assumed.
- [ ] Internet egress volume is estimated in TB per month and priced.
- [ ] Cross-AZ traffic is estimated and priced separately, because it is charged in both directions on many providers.
- [ ] Cross-region replication traffic is included in the cost model, not just the storage it produces.
- [ ] Storage has a tiering plan: hot, warm, and archive, with the age thresholds stated.
- [ ] Data with no retention requirement has a TTL rather than growing forever.
- [ ] Replication factor and erasure-coding choices were made with their cost difference quantified.
- [ ] Observability cost is bounded: metric cardinality, log volume per request, and trace sampling rate are all capped.
- [ ] Log retention is tiered, with full-fidelity retention measured in days and aggregates in months.
- [ ] Idle and over-provisioned capacity is measured, and the headroom target is a deliberate choice rather than drift.
- [ ] Autoscaling scales down as well as up, and the scale-down behaviour has been verified.
- [ ] The cost of the disaster-recovery standby is known and compared against the business value of its RTO.
- [ ] The build-versus-buy comparison includes the engineering cost of operating the self-hosted option.
- [ ] A cost regression would be detected: there is a per-service cost signal, not just a monthly bill.
- [ ] The cost of a 10x traffic increase has been projected, and the line item that grows fastest is named.
- [ ] Cost per request is compared against the revenue or value per request, so the design is economically coherent.

!!! note "Cost is a design constraint, not an afterthought"
    Egress pricing is why CDNs exist. Cross-AZ pricing is why some systems accept zone-local reads with weaker consistency. Storage pricing is why tiering exists. Naming the cost driver behind an architectural pattern is a strong senior signal. See [F28 Cost Engineering](../fundamentals/f28-cost-engineering.md) and [S10 Cost & Efficiency Review](../sre/s10-cost-efficiency-review.md).

---

## The Three-Minute Version

When there is no time for the full lists, run these ten.

- [ ] Did I state my assumptions and the peak QPS number I designed for?
- [ ] Did I name the binding constraint and design around it specifically?
- [ ] Is the partition key chosen deliberately, with the hot-key case answered?
- [ ] Is the consistency model chosen per data path, with the money path identified?
- [ ] Are mutating operations idempotent or keyed?
- [ ] Does every call have a timeout, and does every retry have a budget with jitter?
- [ ] What is the degraded mode when the cache, the database, and a region are gone?
- [ ] What are the two SLIs and their targets?
- [ ] How is this deployed, and what triggers a rollback?
- [ ] What is the dominant cost line, and what would 10x growth do to it?

---

## Related Pages

| Checklist | Depth page |
|---|---|
| Requirements and estimation | [Numbers & Estimation](numbers.md), [Interview Framework](interview-framework.md) |
| API semantics | [F11 Idempotency](../fundamentals/f11-idempotency.md), [45 API Gateway & Service Mesh](../case-studies/45-api-gateway-service-mesh.md) |
| Data model | [F06 Partitioning & Sharding](../fundamentals/f06-partitioning-sharding.md), [F14 SQL vs NoSQL](../fundamentals/f14-sql-vs-nosql.md) |
| Failure modes | [F18 Resilience Patterns](../fundamentals/f18-resilience-patterns.md), [F17 Rate Limiting & Load Shedding](../fundamentals/f17-rate-limiting-load-shedding.md) |
| Operability | [F22 Observability](../fundamentals/f22-observability-fundamentals.md), [F23 SLI/SLO & Error Budgets](../fundamentals/f23-slo-error-budgets.md) |
| Release safety | [F25 Deployment & Release Safety](../fundamentals/f25-deployment-release-safety.md), [S08 Deployment Safety at Scale](../sre/s08-deployment-safety.md) |
| Security | [F27 Security in Design](../fundamentals/f27-security-design.md) |
| Cost | [F28 Cost Engineering](../fundamentals/f28-cost-engineering.md), [S10 Cost & Efficiency Review](../sre/s10-cost-efficiency-review.md) |
