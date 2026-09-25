# SRE-Specific Rounds

**The rounds that replace or supplement product-style system design at companies with strong SRE/infrastructure loops — and the ones generic prep material skips entirely.**

Each round follows a scenario-driven template rather than the case-study template: you're handed
a live situation (an existing service, an incident, a growth event) and structured on how to work
through it, not asked to design a system from a blank page.

| Round | Scenario |
|---|---|
| [S01 — SLO Design](s01-slo-design.md) | Design SLOs from scratch for a real, already-live service |
| [S02 — Multi-Region Active-Active](s02-multi-region-active-active.md) | Migrate a single-region service to active-active with zero downtime |
| [S03 — Zero-Downtime Migration](s03-zero-downtime-migration.md) | Migrate a billions-of-rows table to a new schema with no data loss |
| [S04 — Capacity for 10x Growth](s04-capacity-planning-10x.md) | Plan capacity for a known 10x traffic event |
| [S05 — Load Shedding & Brownout](s05-load-shedding-brownout.md) | Design system-wide graceful degradation under overload |
| [S06 — Incident Response System](s06-incident-response-system.md) | Design the tooling behind detection, paging, comms, and postmortems |
| [S07 — Chaos Engineering Platform](s07-chaos-engineering-platform.md) | Design a platform for safely injecting failure in production |
| [S08 — Deployment Safety at Scale](s08-deployment-safety.md) | Design a release pipeline that can't cause a global outage |
| [S09 — Cell-Based Architecture](s09-cell-based-architecture.md) | Redesign a monolithic-blast-radius service into isolated cells |
| [S10 — Cost & Efficiency Review](s10-cost-efficiency-review.md) | Cut infrastructure cost without hurting reliability |
| [S11 — Debugging a p99 Regression](s11-latency-debugging.md) | Live debug: latency tripled, no deploys — find the cause |
| [S12 — Auto-Remediation System](s12-auto-remediation.md) | Design self-healing automation that leadership can actually trust |

## Why these matter more than they seem to

Product-style system design rounds test whether you can design a system. These rounds test
whether you've **run** one — whether you reach for the right framework under time pressure when
the system already exists, is already on fire, or already needs to change without anyone noticing.

!!! interview "How these are usually scored differently"
    Interviewers running an SRE round are often less interested in a novel architecture and more
    interested in **judgment under constraint**: what you check first, what you refuse to do
    without more signal, and whether your mitigation-first instinct is correct. A candidate who
    proposes an elegant redesign when the actual ask was "stop the bleeding in the next 10
    minutes" reads as junior, regardless of technical depth.

Each page's **Framework / Approach** section gives you the structured method to reach for, and the
**Worked Example** shows it applied end to end with real numbers and artifacts — not just theory.
