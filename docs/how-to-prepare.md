# How to Prepare

**A repeatable interview framework plus an 8-week plan for turning this curriculum into interview-ready reflexes rather than pages you've merely read.**

## The 45-minute interview framework

Practice the clock, not just the content. Interviewers calibrate heavily on whether you can
self-manage time without being told to move on.

| Phase | Time | Output |
|---|---|---|
| Requirements & scoping | 5 min | Written functional + non-functional list, agreed scope |
| Scale estimation | 5 min | QPS, storage, bandwidth — numbers on the board |
| API + data model | 5 min | Contracts and entities |
| High-level design | 10 min | Boxes, arrows, and the happy-path walkthrough |
| Deep dive | 15 min | Interviewer-directed; you drive if they don't |
| Failure / ops / wrap-up | 5 min | Bottlenecks, failure modes, SLOs, "what I'd do next" |

The full breakdown, including clarifying-question checklists and framing sentences that signal
seniority, lives in the [Interview Framework cheat sheet](cheatsheets/interview-framework.md).

!!! interview "Signals that get you the Staff/Lead bar"
    - You state the CAP/PACELC choice explicitly and justify it with the business requirement.
    - You quantify. "Roughly 50k QPS peak, so ~25 shards at 2k writes/shard" beats "we shard it".
    - You name the failure mode *before* the interviewer does.
    - You discuss migration and rollout, not just the steady-state end picture.
    - You talk cost — storage tiering, egress, instance mix, retention.
    - You know when *not* to distribute. A single Postgres box handles more than most people think.

## An 8-week study plan

Assumes roughly 6–8 hours/week. Adjust the pace, keep the ordering — fundamentals before case
studies, core case studies before the infrastructure set, infrastructure before stretch designs.

| Week | Focus | Deliverable |
|---|---|---|
| 1 | [Fundamentals](fundamentals/index.md) F01–F08 (networking, DNS, load balancing, caching, CDN, partitioning, replication, CAP) | Written notes + a consistency-model comparison table |
| 2 | Fundamentals F09–F16 (consensus, transactions, idempotency, queues, storage engines, SQL/NoSQL, object storage, search) | Sharding & replication decision tree you can redraw from memory |
| 3 | Fundamentals F17–F28 (rate limiting, resilience, concurrency, time, probabilistic structures, observability, SLOs, capacity, deployment, DR, security, cost) | Resilience + SLO cheat sheet; one capacity model worked end-to-end |
| 4 | [Case studies](case-studies/index.md) 01–08 (core classics) | 8 timed 45-minute mock designs, self-timed, written |
| 5 | Case studies 09–20 (social, messaging, media, storage) | Focus on fan-out and storage trade-offs |
| 6 | Case studies 21–29 (geo, marketplace, transactional) | Focus on consistency, transactions, and money |
| 7 | Case studies 30–40 (infrastructure & platform) | The set that plays to an SRE background — go deepest here |
| 8 | [SRE rounds](sre/index.md) S01–S12 + weak spots + case studies 41–48 | 4 peer/mock interviews, at least 2 SRE-flavoured |

## Weekly rituals

- One design done **cold**, under a 45-minute timer, spoken out loud, before reading the page.
- One "explain to a rubber duck in 5 minutes" summary per completed topic.
- Maintain a personal **mistakes log** — every time you miss a failure mode or a gotcha, write it
  down. Re-read it the night before the real interview.

## Reading a page efficiently

You do not need to read every section of every page at the same depth on every pass.

=== "First pass (cold recall check)"
    Read only the metadata table, Problem Statement, and Requirements. Try to produce your own
    architecture and API design before reading further. Then read the rest and diff against your
    own answer.

=== "Second pass (depth)"
    Read the full page. Spend disproportionate time in **Deep Dives**, **Gotchas & Corner
    Cases**, and **SRE Lens** — these are where marginal prep time has the highest payoff.

=== "Pre-interview refresh"
    Skim only the **Key Takeaways** and the **Interview Angle** follow-up question bank across
    every page in the relevant section.

## Tracking progress

Use the status column mentally (or in your own notes) as you go: `todo` → `drafted` → `drilled`
→ `solid`. A topic is `solid` only once you've done it cold, under time pressure, and gotten the
gotchas right without prompting.
