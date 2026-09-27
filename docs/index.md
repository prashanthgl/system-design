<div class="sd-hero" markdown>

# System Design for SRE Leads

**A depth-first system design curriculum built for a senior SRE / infrastructure Tech Lead preparing for top-tier interviews.**

Most system design material optimizes for "draw boxes, add a cache, mention sharding." This site
is written for someone who has been paged for the boxes falling over. Every case study has a
dedicated **SRE Lens** section — SLIs/SLOs, rollout risk, capacity model, runbook notes — and a
**Gotchas & Corner Cases** section that is, deliberately, the highest-value part of the page.

<div class="sd-stats" markdown>
- <span class="sd-stat-num">28</span><span class="sd-stat-label">Fundamentals</span>
- <span class="sd-stat-num">48</span><span class="sd-stat-label">Case Studies</span>
- <span class="sd-stat-num">12</span><span class="sd-stat-label">SRE Rounds</span>
- <span class="sd-stat-num">5</span><span class="sd-stat-label">Cheat Sheets</span>
</div>

[:material-rocket-launch-outline: Start the 8-week plan](how-to-prepare.md){ .md-button .md-button--primary }
[:material-view-list-outline: Full site index](sitemap.md){ .md-button }

</div>

<div class="grid cards" markdown>

- :material-book-open-variant:{ .lg .middle } **28 Fundamentals**

    ---

    The primitives every design composes: networking, caching, partitioning, consensus, storage
    engines, resilience patterns, observability, and cost.

    [:octicons-arrow-right-24: Start with fundamentals](fundamentals/index.md)

- :material-view-grid:{ .lg .middle } **48 Case Studies**

    ---

    From URL shorteners to stock exchanges. Every design follows a fixed 14-section template
    ending in gotchas, an interview follow-up bank, and key takeaways.

    [:octicons-arrow-right-24: Browse case studies](case-studies/index.md)

- :material-shield-alert-outline:{ .lg .middle } **12 SRE Rounds**

    ---

    The rounds generic prep material skips: SLO design, multi-region migration, load shedding,
    live latency debugging, auto-remediation.

    [:octicons-arrow-right-24: SRE-specific rounds](sre/index.md)

- :material-calculator-variant-outline:{ .lg .middle } **Cheat Sheets**

    ---

    Back-of-the-envelope numbers, the 45-minute interview framework, decision trees, and review
    checklists for the day before your interview.

    [:octicons-arrow-right-24: Open cheat sheets](cheatsheets/index.md)

</div>

---

## Who this is for

Written for roughly 10 years of experience in SRE / infrastructure, targeting Staff/Lead-level
system design loops at companies like Google, Meta, Amazon, Stripe, Cloudflare, Netflix,
Databricks, and similar. The bar assumed throughout is: **quantify, don't hand-wave; name the
failure mode before the interviewer does; talk about rollout and cost, not just the steady state.**

If you are earlier in your career, the fundamentals and core case studies (Section A) are still
the right starting point — just expect the SRE Lens sections to stretch you.

## How the content is structured

Every case study and SRE round follows a fixed template so you can drill a single topic in
30–45 minutes without re-orienting yourself each time:

| Section | What it answers |
|---|---|
| Problem Statement & Requirements | The literal question, scoped explicitly |
| Scale Estimation | Real arithmetic — QPS, storage, bandwidth, not vibes |
| API Design & Data Model | Contracts and the store choice, with rejected alternatives |
| High-Level Architecture | A diagram plus an explicit write-path and read-path walkthrough |
| Deep Dives | The 2–4 areas an interviewer will actually push on |
| Failure Modes | Blast radius, detection, mitigation, degraded behaviour — as a table |
| **SRE Lens** | SLIs/SLOs, rollout plan, runbook notes, capacity model, cost |
| **Gotchas & Corner Cases** | 8+ specific, mechanistic traps — symptom, mechanism, mitigation |
| **Interview Angle** | Follow-up questions with answers, and a strong-vs-weak answer contrast |

See [How to Prepare](how-to-prepare.md) for the full interview framework and an 8-week study plan.

## Where to start

1. Read [How to Prepare](how-to-prepare.md) for the 45-minute interview framework.
2. Work through [Fundamentals](fundamentals/index.md) — don't skip this even if you know the
   basics; the gotchas are where the depth lives.
3. Drill [Case Studies](case-studies/index.md), starting with Section E (Infrastructure &
   Platform) if your background is SRE-heavy — it plays directly to existing operational depth.
4. Finish with the [SRE Rounds](sre/index.md) and a handful of mock interviews using the
   [Interview Framework cheat sheet](cheatsheets/interview-framework.md).
