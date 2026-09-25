# 47 — Secrets Management (Vault-style)

<span class="pill pill-core">Core</span> <span class="pill pill-hard">Hard</span>

**A system whose entire purpose is to never store a usable secret at rest — which means it must start up unable to decrypt its own data, and the hardest design problem is how it ever becomes useful again after every node restarts at 3 a.m.**

| | |
|---|---|
| **Commonly asked at** | HashiCorp, AWS, Google, Stripe, Cloudflare, Datadog, Square, Coinbase, any payments, fintech or security-platform org |
| **Time budget** | 45 min |
| **Core tension** | Every security property you want — sealed storage, a quorum to unseal, short-lived credentials, deny-by-default policy, an audit record of every access — makes the system *less* available and *more* of a bootstrap dependency. But it sits underneath every application in the company, so it needs higher availability than any of them. You are being asked to make a system maximally hard to compromise and maximally easy to depend on, and those pull in opposite directions in every single decision |
| **Prerequisites** | [F04 Caching](../fundamentals/f04-caching.md), [F07 Replication & Consistency](../fundamentals/f07-replication-consistency.md), [F08 CAP & PACELC](../fundamentals/f08-cap-pacelc.md), [F09 Consensus](../fundamentals/f09-consensus.md), [F11 Idempotency](../fundamentals/f11-idempotency.md), [F13 Storage Engines](../fundamentals/f13-storage-engines.md), [F17 Rate Limiting & Load Shedding](../fundamentals/f17-rate-limiting-load-shedding.md), [F18 Resilience Patterns](../fundamentals/f18-resilience-patterns.md), [F20 Time, Clocks & Ordering](../fundamentals/f20-time-clocks-ordering.md), [F22 Observability Fundamentals](../fundamentals/f22-observability-fundamentals.md), [F23 SLOs & Error Budgets](../fundamentals/f23-slo-error-budgets.md), [F24 Capacity Planning](../fundamentals/f24-capacity-planning.md), [F25 Deployment & Release Safety](../fundamentals/f25-deployment-release-safety.md), [F26 Multi-Region & DR](../fundamentals/f26-multi-region-dr.md), [F27 Security in Design](../fundamentals/f27-security-design.md) |

---

## 1. Problem Statement

Design the system that holds every credential in the company: database passwords, cloud API keys, TLS private keys, third-party API tokens, encryption keys, signing keys. Roughly 5,000 workloads across 40,000 pods need credentials, and none of them should have a long-lived one.

The naive framing — "an encrypted key-value store with access control" — misses four things that actually define the design.

**First: the system must be unable to read its own data at rest, which creates a bootstrapping problem with no clean solution.** If the process can decrypt storage the moment it starts, then anyone who can start the process, or copy the disk, or dump the memory, has the secrets. So the data encryption key is itself encrypted by a master key that is *not on disk*, and the system boots **sealed** — running, answering health checks, and refusing every request because it cannot decrypt anything. Something external must supply the master key. That "something" is the entire problem: a human quorum is secure and operationally miserable, a cloud KMS is operationally excellent and moves your trust root to a third party, and an HSM is both expensive and another thing that can be down. **There is no option where the system is both self-sufficient and secure**, and a senior answer says that out loud rather than hand-waving past it.

**Second: the highest-value feature is not storing secrets, it is *not* storing them.** A static database password has an exposure window measured from the moment it leaks to the moment somebody notices and rotates it — industry data puts the median discovery time for leaked credentials in the hundreds of days. A credential minted on demand with a one-hour TTL has an exposure window of at most one hour. That is a four-thousand-fold reduction in blast radius, and it comes from the system acting as a **credential broker to the backend** rather than as a vault of stored strings. Most of the interesting engineering here — lease tracking, revocation, TTL negotiation, backend load — follows from that shift.

**Third: authentication to the secrets store is a chicken-and-egg problem you can shrink but never eliminate.** If a workload needs a credential to fetch credentials, you have not removed the secret, you have concentrated it. The answer is identity-based authentication: the workload proves *what it is* using an unforgeable property of its runtime — a projected service-account token, a cloud instance identity document, a SPIFFE SVID, an attested TPM quote — rather than presenting a stored password. This pushes the root of trust into the platform, where it belongs. But the platform's attestation is itself rooted in something, and at the very bottom of the chain there is always **one credential that had to be placed by a process outside the system**. You cannot delete it; you can only make it small, short-lived, non-transferable, and heavily audited.

**Fourth: it sits at the bottom of the dependency graph and its outage is unlike other outages.** When the secrets store is down, existing workloads with cached credentials keep running — but nothing can start, nothing can renew, and when the first leases expire the failure begins spreading in a wave. The blast radius is not instantaneous; it is a *clock* that starts when the outage starts and detonates one TTL later. Designing for that means caching, graceful degradation, and TTLs chosen to give humans time.

### Out of scope

The cryptographic primitives themselves (AES-GCM, key derivation), PKI certificate policy and CA hierarchy design beyond the issuance path, hardware security module internals, and enterprise key-management compliance frameworks.

---

## 2. Requirements

### Functional

| # | Requirement | Notes |
|---|---|---|
| F1 | Encrypted storage of arbitrary key-value secrets, versioned | Static secrets, with history and rollback |
| F2 | Sealed-at-rest storage; explicit unseal to become operational | The core security property |
| F3 | Unseal via threshold quorum, cloud KMS, or HSM | All three paths supported |
| F4 | Dynamic credential generation for databases, cloud providers, message brokers | The main event |
| F5 | Leases with TTL, renewal, and revocation — individually and by prefix | Revocation must be able to cascade |
| F6 | Identity-based authentication: Kubernetes SA, cloud IAM, SPIFFE, OIDC, TLS certs | No bootstrap password |
| F7 | Path-based policy with deny-by-default and capability granularity | `read` is not `list` is not `delete` |
| F8 | Append-only, tamper-evident audit of every request and response | Including denials |
| F9 | Encryption as a service: encrypt/decrypt/sign/verify without exposing keys | The "transit" pattern |
| F10 | Certificate issuance (an internal CA) with short TTLs | |
| F11 | Automatic and on-demand rotation of static secrets with an overlap window | |
| F12 | Response wrapping: single-use tokens that unwrap to a secret | Safe hand-off of a secret through an untrusted channel |
| F13 | Namespaces / multi-tenancy with isolated policy and audit | |
| F14 | Disaster recovery and performance replication across regions | |

### Non-functional

| # | Requirement | Target |
|---|---|---|
| N1 | Availability | **99.99%** (bottom of the dependency graph) |
| N2 | Read latency (cached / uncached) | p99 < 10 ms / < 50 ms |
| N3 | Dynamic credential issuance latency | p99 < 500 ms (bounded by the backend) |
| N4 | Throughput | 2,500 ops/s peak, 250 ops/s sustained |
| N5 | Unseal time after a full restart — automated | < 60 s |
| N6 | Unseal time after a full restart — human quorum | < 20 min (and this is the problem) |
| N7 | Audit completeness | **100%** — no request served without an audit record committed |
| N8 | Revocation propagation | < 5 s for a direct lease, < 60 s for a cascading prefix revoke |
| N9 | Maximum credential TTL | 24 h hard cap; 1 h default for database credentials |
| N10 | Cross-region DR RPO / RTO | RPO < 30 s, RTO < 15 min |
| N11 | Key rotation without downtime | Zero failed requests during rotation |

!!! danger "N7 is the requirement that makes this system slower than you want"
    "No request is served without a committed audit record" means the audit write is **synchronous and on the critical path**, and if every audit device fails, the system must **refuse to serve** rather than serve unaudited. That is a deliberate availability sacrifice: a broken log shipper can take down credential issuance for the entire company.

    It is nonetheless correct. The audit log is the only thing that lets you answer "what did the attacker access?" after a compromise, and an audit log with gaps is worth close to nothing during an investigation — you cannot prove a negative about the gap. The mitigation is not to make the audit asynchronous; it is to run **multiple independent audit devices** (a local file and a remote socket) and require at least one to succeed, so a single device's failure degrades redundancy rather than availability.

---

## 3. Scale Estimation

### Fleet and load

$$
\begin{aligned}
\text{workloads (distinct identities)} &= 5{,}000 \\
\text{pods} &= 40{,}000 \\
\text{static secrets (KV entries)} &= 20{,}000 \\
\text{active dynamic leases} &\approx 5{,}000 \\
\text{sustained ops} &= 250\ \text{/s},\quad \text{peak} = 2{,}500\ \text{/s}
\end{aligned}
$$

### Where the steady-state load comes from

$$
\begin{aligned}
\text{token renewals} &: \frac{40{,}000\ \text{tokens}}{40\ \text{min}} = \frac{40{,}000}{2{,}400\ \text{s}} = 16.7\ \text{/s} \\
\text{lease renewals} &: \frac{5{,}000\ \text{leases}}{40\ \text{min}} = 2.1\ \text{/s} \\
\text{secret reads (cached, on change or restart)} &\approx 30\ \text{/s} \\
\text{transit encrypt/decrypt} &\approx 180\ \text{/s} \\
\text{logins (pod churn)} &: \frac{40{,}000}{3\ \text{day pod lifetime}} = 0.15\ \text{/s} \\
\hline
\text{sustained} &\approx \mathbf{229\ \text{/s}}
\end{aligned}
$$

Note how small the "read a secret" portion is. **Most of the load is lease and token lifecycle management, not secret retrieval** — which means the TTL you choose is the single biggest input to capacity.

$$
\text{load} \propto \frac{N_{\text{tokens}} + N_{\text{leases}}}{\text{TTL} \times \frac{2}{3}}
$$

Halving the TTL doubles the steady-state load on a system that is already a hard dependency for everything. This is the first place where a security instinct ("shorter is better") collides with an availability constraint.

### The restart storm — the number that sizes the system

$$
\begin{aligned}
\text{full cluster restart} &: 40{,}000\ \text{pods over } 10\ \text{min} \\
\text{logins} &= \frac{40{,}000}{600} = 66.7\ \text{/s (each is a WRITE — token creation)} \\
\text{secret reads} &= 40{,}000 \times 3 = 120{,}000\ \text{over } 600\ \text{s} = 200\ \text{/s} \\
\text{dynamic creds} &= 5{,}000\ \text{over } 600\ \text{s} = 8.3\ \text{/s} \\
\hline
\text{peak} &\approx \mathbf{275\ \text{/s, with } 67\ \text{/s of writes}}
\end{aligned}
$$

A regional failover compresses the same work into two minutes:

$$
\frac{40{,}000\ \text{logins}}{120\ \text{s}} = \mathbf{333\ \text{writes/s}}
$$

Writes go through the storage backend's consensus layer, so 333 writes/s is the number that must be survivable. This is why N4's headroom exists and why login admission control matters more than read capacity.

### Shamir's Secret Sharing: the threshold math

The master key $S$ is split by constructing a random polynomial of degree $k-1$ over a finite field, with $S$ as the constant term:

$$
f(x) = S + a_1 x + a_2 x^2 + \dots + a_{k-1}x^{k-1} \pmod p
$$

Each of the $n$ holders receives a point $(x_i, f(x_i))$ with $x_i \neq 0$. Any $k$ points determine the polynomial uniquely by Lagrange interpolation:

$$
S = f(0) = \sum_{i=1}^{k} f(x_i) \prod_{\substack{j=1 \\ j \neq i}}^{k} \frac{-x_j}{x_i - x_j} \pmod p
$$

The security property is **information-theoretic, not computational**: with $k-1$ shares, every possible value of $S$ remains exactly equally likely. No amount of compute helps. That is a genuinely strong guarantee and it is why this scheme is used for the one key that matters most.

#### The operational cost of that guarantee

Model each key holder as independently reachable-and-able-to-act within the recovery window with probability $p$. The unseal ceremony succeeds only if at least $k$ of $n$ are available:

$$
P_{\text{unseal}} = \sum_{i=k}^{n} \binom{n}{i} p^{i}(1-p)^{n-i}
$$

At $p = 0.8$ (a realistic 3 a.m. figure across holiday seasons, time zones, flights and dead phones):

| $n$ | $k$ | $P_{\text{unseal}}$ | Shares an attacker needs | Verdict |
|---|---|---|---|---|
| 3 | 2 | 0.896 | 2 | Both numbers bad |
| 5 | 3 | **0.942** | 3 | The common default — a 5.8% chance the ceremony stalls |
| 5 | 2 | 0.993 | 2 | Available, but two insiders can collude |
| 7 | 4 | **0.967** | 4 | Better on both axes; more holders to manage |
| 7 | 3 | 0.995 | 3 | The sweet spot for a human quorum |
| 9 | 5 | 0.980 | 5 | Diminishing returns, high coordination cost |

$$
\text{Worked: } n=7,\ k=3:\ P = \sum_{i=3}^{7}\binom{7}{i}(0.8)^i(0.2)^{7-i} = 0.9953
$$

!!! warning "A 94% unseal probability is an unacceptable availability number for a 99.99% service"
    Read the $n=5, k=3$ row again: **roughly one in seventeen full-restart events cannot complete the ceremony on the first attempt.** That is not a tail risk; that is a recurring operational event. And the time cost of the successful cases is not small either:

    $$
    \begin{aligned}
    t_{\text{ceremony}} &= t_{\text{page}} + t_{\text{holder wakes and authenticates}} + t_{\text{share entry}} \\
    &\approx 8\ \text{min} + 5\ \text{min} + 2\ \text{min} = \mathbf{15\ \text{min}}\ \text{(parallel, best case)}
    \end{aligned}
    $$

    Fifteen minutes of total outage for the credential layer of the entire company, *if everything goes well*, and this is on top of whatever caused the restart. Against a 99.99% target of 4.32 minutes per month, **one human unseal blows three and a half months of error budget.**

    This arithmetic is the entire argument for auto-unseal, and it is the argument a strong candidate makes rather than describing Shamir as though the ceremony were free.

### Dynamic versus static credentials: blast-radius arithmetic

$$
\begin{aligned}
\text{exposure window}_{\text{static}} &= t_{\text{leak}} \to t_{\text{detect}} + t_{\text{rotate}} \approx 200\ \text{days} \\
\text{exposure window}_{\text{dynamic, 1h TTL}} &\leq 1\ \text{hour} \\
\hline
\text{reduction} &= \frac{200 \times 24}{1} = \mathbf{4{,}800\times}
\end{aligned}
$$

But the TTL cannot go arbitrarily low, because the **backend** has to mint each credential:

$$
\begin{aligned}
\text{creation rate} &= \frac{N_{\text{leases}}}{\text{TTL} \times \frac{2}{3}} \\
\text{TTL} = 1\ \text{h} &: \frac{5{,}000}{2{,}400} = 2.1\ \text{CREATE ROLE/s} \\
\text{TTL} = 15\ \text{min} &: \frac{5{,}000}{600} = 8.3\ \text{CREATE ROLE/s} \\
\text{TTL} = 5\ \text{min} &: \frac{5{,}000}{200} = 25\ \text{CREATE ROLE/s}
\end{aligned}
$$

`CREATE ROLE` is DDL. On PostgreSQL it takes a lock on the shared catalog, writes to `pg_authid`, and — critically — **is replicated to every standby**. At 25 role creations per second, plus 25 drops, you are running 50 DDL statements per second against a production database catalog, which causes lock contention, catalog bloat and replication lag on a system that is supposed to be serving your application.

!!! tip "The TTL floor is set by the backend, not by the secrets store"
    This is the constraint nobody anticipates. The secrets store can mint credentials as fast as you like; the database cannot absorb them. Practical floors: **PostgreSQL/MySQL, 1 hour** (role churn is the binding constraint); **cloud IAM via STS, 15 minutes** (STS is designed for exactly this and scales fine); **internal PKI certificates, 1 hour or less** (issuance is cheap, signing is local). Choose TTL per backend, defend the number with the DDL rate, and never apply a single global TTL policy across backends with wildly different minting costs.

### Audit volume

$$
\begin{aligned}
\text{entries} &= 250\ \text{/s} \times 2\ (\text{request + response}) = 500\ \text{/s} \\
\text{entry size} &\approx 1.6\ \text{KB (JSON, HMACed values, full request context)} \\
\text{rate} &= 800\ \text{KB/s} = 69\ \text{GB/day} \\
\text{7-year compliance retention} &= 69 \times 365 \times 7 = \mathbf{176\ \text{TB}}
\end{aligned}
$$

At cold-tier object-storage pricing, 176 TB is roughly $700/month — cheap, and a good thing to know, because the instinct to trim audit retention for cost is trimming the smallest line on the bill while destroying the ability to investigate a breach.

### Throughput ceiling

$$
\begin{aligned}
t_{\text{TLS + auth + policy eval}} &\approx 180\ \mu\text{s} \\
t_{\text{decrypt + serialize}} &\approx 40\ \mu\text{s} \\
t_{\text{audit fsync (group commit, } B{=}32)} &\approx \frac{300\ \mu\text{s}}{32} = 9.4\ \mu\text{s amortised} \\
t_{\text{storage write (writes only, via Raft)}} &\approx 1.8\ \text{ms} \\
\hline
T_{\text{read, per core}} &= \frac{1}{230\ \mu\text{s}} = 4{,}350\ \text{/s} \Rightarrow \text{8 cores} \approx 30\text{k/s} \\
T_{\text{write (Raft-bound, single active node)}} &\approx 3{,}000\ \text{/s}
\end{aligned}
$$

Reads scale with performance standbys; **writes are pinned to the single active node** because the storage backend is a consensus log ([F09](../fundamentals/f09-consensus.md)). Since logins and lease creations are writes, the 333 writes/s regional-failover number sits at about 11% of the write ceiling — comfortable, but only because we chose a 1-hour TTL rather than a 5-minute one.

---

## 4. API Design

### Authentication — proving identity without a secret

```bash
# Kubernetes workload identity. The JWT is a PROJECTED service-account token:
# audience-bound, time-bound, and rotated by the kubelet. It is not a static
# secret sitting in a Secret object.
POST /v1/auth/kubernetes/login
{
  "role": "checkout-service",
  "jwt":  "<projected SA token, aud=vault, exp=10m>"
}
# The server calls the cluster's TokenReview API to verify the JWT against the
# cluster's own signing key. It never trusts a claim it cannot independently check.
-> {
     "auth": {
       "client_token": "hvs.CAESIJ...",
       "lease_duration": 3600,
       "renewable": true,
       "policies": ["checkout-read", "default"],
       "metadata": { "service_account_name": "checkout",
                     "namespace": "commerce", "pod_name": "checkout-7f9-xk2" }
     }
   }
```

```bash
# Cloud instance identity: the signed instance identity document, verified
# against the cloud provider's public key. Again: an attestation, not a secret.
POST /v1/auth/aws/login   { "role": "ingest", "pkcs7": "<signed IID>" }

# SPIFFE / mTLS: the client presents its workload certificate; the SVID's
# SAN URI is the identity.
POST /v1/auth/cert/login  # identity comes from the TLS peer certificate
```

### Static secrets — versioned KV

```bash
GET    /v1/secret/data/commerce/stripe?version=4
POST   /v1/secret/data/commerce/stripe
       { "data": {...}, "options": { "cas": 3 } }   # compare-and-set
DELETE /v1/secret/data/commerce/stripe              # soft delete a version
POST   /v1/secret/destroy/commerce/stripe           # irreversible
GET    /v1/secret/metadata/commerce/stripe          # versions, timestamps, no data
```

`cas` (check-and-set) is not decoration: two concurrent rotation jobs without it will silently lose one another's write, and the losing job will believe it succeeded.

### Dynamic credentials — the main event

```bash
# Configure once, by an operator, with a ROOT credential that is then rotated
# so that not even the operator knows it any more.
POST /v1/database/config/orders-primary
{
  "plugin_name": "postgresql-database-plugin",
  "connection_url": "postgresql://{{username}}:{{password}}@orders-primary:5432/orders",
  "allowed_roles": ["orders-ro", "orders-rw"],
  "username": "vault-root", "password": "<initial>",
  "password_policy": "strong-64"
}
POST /v1/database/rotate-root/orders-primary   # now ONLY the vault knows it

POST /v1/database/roles/orders-ro
{
  "db_name": "orders-primary",
  "creation_statements": [
    "CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}';",
    "GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";"
  ],
  "revocation_statements": [
    "REASSIGN OWNED BY \"{{name}}\" TO vault_owner;",
    "DROP OWNED BY \"{{name}}\";",
    "DROP ROLE IF EXISTS \"{{name}}\";"
  ],
  "default_ttl": "1h",
  "max_ttl": "24h"
}

# The request path, called by the application.
GET /v1/database/creds/orders-ro
-> {
     "lease_id": "database/creds/orders-ro/8x2Kq...",
     "lease_duration": 3600,
     "renewable": true,
     "data": { "username": "v-kube-orders-ro-8x2Kq-1742", "password": "A1b2..." }
   }
```

!!! tip "`VALID UNTIL` is the belt to the lease's braces"
    Notice that the creation statement sets a database-enforced expiry in addition to the lease TTL. If the secrets store is down, loses its lease table, or fails to run the revocation statement, the database still expires the credential on its own. **Defence in depth where the enforcement lives in the resource rather than in the broker** — the same principle as fencing tokens in a coordination service. Any backend that supports a native expiry should have it set, and the value should be slightly longer than `max_ttl` so the normal revocation path wins the race.

```bash
# Lease lifecycle. Renewal is what the sidecar/agent does continuously.
PUT  /v1/sys/leases/renew     { "lease_id": "...", "increment": 3600 }
PUT  /v1/sys/leases/revoke    { "lease_id": "..." }
PUT  /v1/sys/leases/revoke-prefix/database/creds/orders-ro   # the break-glass lever
```

### Encryption as a service (transit)

```bash
# The key NEVER leaves. The application sends plaintext and gets ciphertext.
POST /v1/transit/encrypt/pii
{ "plaintext": "<base64>", "context": "<base64 tenant id>" }
-> { "ciphertext": "vault:v3:8SDd3WHDOjf7mq69CyCqYjBXAiQQAVZ..." }
#                         ^^ key version is embedded in the ciphertext

POST /v1/transit/decrypt/pii   { "ciphertext": "vault:v3:..." }
POST /v1/transit/rotate/pii    # creates v4; v1..v3 still decrypt
POST /v1/transit/rewrap/pii    { "ciphertext": "vault:v1:..." }
     # -> "vault:v4:..." without ever exposing plaintext to the caller
POST /v1/transit/sign/signing-key    { "input": "<base64>" }
```

The `vault:vN:` prefix is the whole design in three characters: **every ciphertext carries the version of the key that produced it**, so rotation creates a new version without invalidating anything, old ciphertext keeps decrypting forever, and `rewrap` upgrades ciphertext in the background without the caller ever holding plaintext.

### Policy

```hcl
# Deny by default. Every grant is explicit, path-scoped and capability-scoped.
path "database/creds/orders-ro" {
  capabilities = ["read"]
}

# Templated policy: the path is derived from the authenticated identity, so one
# policy serves thousands of workloads without enumerating them.
path "secret/data/{{identity.entity.aliases.auth_kubernetes_9f2.metadata.service_account_namespace}}/*" {
  capabilities = ["read"]
}

path "transit/encrypt/pii" { capabilities = ["update"] }
path "transit/decrypt/pii" {
  capabilities = ["update"]
  # Decryption of production PII requires the request to originate from the
  # production CIDR AND be made with an MFA-backed token.
  allowed_parameters = { "context" = [] }
}

# Explicitly deny even if another policy grants it. Deny always wins.
path "secret/data/commerce/root-*" { capabilities = ["deny"] }

# Break-glass: a separate, MFA-gated, heavily-alerted path.
path "sys/leases/revoke-prefix/*" {
  capabilities = ["update"]
  required_parameters = ["confirm"]
}
```

```hcl
# Response wrapping: hand a secret through an untrusted intermediary safely.
# The CI system receives a single-use wrapping token, not the secret. If the
# token has already been used when the recipient tries it, you KNOW it was
# intercepted -- detection, not just prevention.
POST /v1/secret/data/ci/deploy-key
X-Vault-Wrap-TTL: 120
-> { "wrap_info": { "token": "hvs.CAESIA...", "ttl": 120, "creation_path": "secret/data/ci/deploy-key" } }

POST /v1/sys/wrapping/unwrap   # single use; second attempt fails AND alerts
```

---

## 5. Data Model

### The key hierarchy

```mermaid
flowchart TB
    UK["Unseal keys - Shamir shares or KMS key"] --> MK["Master key - in memory only"]
    MK --> EK["Encryption key ring - v1..vN"]
    EK --> D1["Storage entry: secret/commerce/stripe"]
    EK --> D2["Storage entry: sys/policy/checkout"]
    EK --> D3["Storage entry: sys/expire/lease/..."]
    EK --> D4["Storage entry: auth/token/..."]
```

Four layers, each with a distinct purpose:

| Layer | Where it lives | Rotated | Purpose |
|---|---|---|---|
| **Unseal keys / KMS key** | Human custody, or a cloud KMS, or an HSM | Rarely, via a rekey ceremony | Protects the master key; the root of the trust chain |
| **Master key** | **Memory only, never on disk** | On rekey | Encrypts the encryption key ring |
| **Encryption key ring** | On disk, encrypted by the master key | Routinely; old versions retained | Encrypts every storage entry |
| **Data encryption** | Each storage entry is individually encrypted | Lazily, on write | The actual data |

The key ring keeps old versions so that entries written under an old key still decrypt. Rotation therefore creates a new version for *new* writes and leaves everything else alone, with an optional background rewrap. **This is the same versioned-key pattern as transit's `vault:vN:` prefix, applied internally.**

### Storage entries

```text
core/master                      Master key, encrypted by the unseal mechanism
core/keyring                     Encryption key ring, encrypted by master key
core/seal-config                 { type: shamir|awskms|pkcs11, n: 5, k: 3 }

sys/policy/<name>                Policy documents
sys/mounts/<path>                Which engine is mounted where
sys/expire/lease/<lease_id>      { path, issue_time, expire_time, renewable,
                                   auth_entity, secret_internal_data }
sys/token/id/<sha256(token)>     { policies, ttl, parent, num_uses, entity_id }
sys/token/parent/<parent>/<child>   The revocation tree

identity/entity/<id>             A logical identity
identity/entity-alias/<id>       Binding from an auth method to an entity

secret/<path>                    Versioned KV data
database/config/<name>           Backend connection + ROOT credential
transit/keys/<name>              Key material, versioned, never exported
```

### Two structures that carry the whole system

```sql
-- THE LEASE TABLE. Every dynamic credential in existence is here. If this is
-- lost, credentials exist in backends with nothing tracking or revoking them --
-- the "orphaned credential" disaster.
lease(
  lease_id          PK,
  path,                       -- database/creds/orders-ro
  entity_id,                  -- who holds it
  issue_time, expire_time,
  renewable, max_ttl,
  backend_internal            -- e.g. the generated DB username, needed to revoke
)
CREATE INDEX ON lease (expire_time);   -- the expiry sweeper's query
CREATE INDEX ON lease (path);          -- revoke-prefix
CREATE INDEX ON lease (entity_id);     -- "revoke everything this workload holds"

-- THE TOKEN TREE. Revocation cascades to children, which is how a compromised
-- parent identity can be cut off completely in one operation.
token(token_hash PK, parent_hash, entity_id, policies[], ttl, num_uses, expire_time)
```

!!! danger "The lease table is the system's most dangerous piece of state"
    Every dynamic credential is a **pair**: a real user in a real backend, and a row here that remembers it exists. If the row is lost, the backend user survives with no expiry driven by the broker and nothing that will ever revoke it. Over months, an untracked backend accumulates thousands of orphaned roles nobody can safely delete because nobody knows which are in use.

    Three defences, all needed. **Durability**: the lease table lives in the same consensus-replicated storage as everything else, never in a cache. **Native backend expiry**: `VALID UNTIL` on the database side so the backend cleans up even if the broker never asks. **Reconciliation**: a periodic job that lists roles matching the broker's naming prefix in each backend and compares them with the lease table, alerting on either direction of drift. The reconciliation job is the one people skip and the one that finds real problems.

### Audit entry

```json
{
  "time": "2026-03-14T09:22:41.880Z",
  "type": "response",
  "auth": {
    "client_token": "hmac-sha256:9a1f...",
    "accessor": "hmac-sha256:c3b2...",
    "display_name": "kubernetes-commerce-checkout",
    "policies": ["checkout-read", "default"],
    "entity_id": "8f2c1a...",
    "metadata": { "service_account_name": "checkout", "namespace": "commerce" }
  },
  "request": {
    "id": "b41e...", "operation": "read",
    "path": "database/creds/orders-ro",
    "remote_address": "10.42.17.9",
    "client_certificate_serial": "hmac-sha256:11ab..."
  },
  "response": {
    "data": { "username": "hmac-sha256:7d3e...", "password": "hmac-sha256:aa91..." },
    "lease_id": "hmac-sha256:5511...", "lease_duration": 3600
  },
  "error": ""
}
```

Two properties make this a usable audit log rather than a liability:

- **Every sensitive value is HMACed with a dedicated audit key, never written in the clear.** The log is therefore not itself a secrets leak, and an attacker who steals the audit log gains nothing. Because HMAC is deterministic, you can still answer "was *this specific* credential accessed?" by HMACing the candidate and searching — which is exactly the investigation query you need.
- **Denials are logged identically to successes.** A burst of denials is the single best signal of a compromised identity probing for reachable paths, and a log that records only successes is blind to the most interesting part of an attack.

---

## 6. High-Level Architecture

```mermaid
flowchart TB
    subgraph CLIENTS["Workloads"]
        A["App pod + agent sidecar"]
        B["Batch job"]
        C["CI runner"]
    end

    subgraph CORE["Secrets core - one active, N standby"]
        ACT["Active node"]
        SB1["Performance standby 1"]
        SB2["Performance standby 2"]
    end

    subgraph SEAL["Unseal path"]
        KMS["Cloud KMS auto-unseal"]
        SHA["Shamir shares - recovery"]
        HSM["HSM - optional"]
    end

    subgraph STORE["Storage - Raft consensus"]
        R1["Raft 1"]
        R2["Raft 2"]
        R3["Raft 3"]
    end

    subgraph BACK["Credential backends"]
        PG["PostgreSQL"]
        IAM["Cloud IAM STS"]
        PKI["Internal CA"]
    end

    subgraph AUD["Audit devices"]
        F["Local file"]
        S["Remote sink"]
    end

    A --> ACT
    B --> SB1
    C --> SB2
    SB1 -->|"forward writes"| ACT
    SB2 -->|"forward writes"| ACT
    ACT --> R1
    R1 --- R2
    R1 --- R3
    KMS -.->|"unwrap master key"| ACT
    SHA -.->|"recovery only"| ACT
    HSM -.-> ACT
    ACT --> PG
    ACT --> IAM
    ACT --> PKI
    ACT --> F
    ACT --> S
```

### The seal/unseal state machine

```mermaid
stateDiagram-v2
    [*] --> Sealed: "process starts"
    Sealed --> Unsealing: "KMS unwrap, or k of n shares"
    Unsealing --> Sealed: "insufficient shares or KMS unreachable"
    Unsealing --> Standby: "master key recovered"
    Standby --> Active: "wins leader election"
    Active --> Standby: "steps down"
    Active --> Sealed: "manual seal, or seal on critical error"
    Standby --> Sealed: "manual seal"
    note right of Sealed
      Process is RUNNING.
      Health endpoint answers 503.
      Every API call returns 503.
      Data on disk is unreadable.
    end note
```

**Sealed is a running, healthy, completely useless process.** It answers health checks, it has network connectivity, it can be scraped for metrics, and it cannot serve a single request because the master key is not in memory. This is the property that makes a stolen disk image worthless, and it is the property that makes a cold start an incident.

### Read path — a cached secret

```mermaid
sequenceDiagram
    autonumber
    participant P as "App process"
    participant AG as "Agent sidecar"
    participant V as "Secrets core"
    participant PG as "PostgreSQL"

    Note over AG: "Startup"
    AG->>AG: "Read projected SA token from tmpfs"
    AG->>V: "POST auth/kubernetes/login"
    V->>V: "TokenReview against cluster API"
    V->>V: "Map SA to entity, resolve policies"
    V->>V: "Write token to storage - a Raft write"
    V->>V: "Commit audit entry"
    V-->>AG: "client_token, ttl 3600"

    AG->>V: "GET database/creds/orders-ro"
    V->>V: "Policy check: read allowed"
    V->>PG: "CREATE ROLE v-kube-... VALID UNTIL ..."
    PG-->>V: "ok"
    V->>V: "Persist lease - Raft write"
    V->>V: "Commit audit entry"
    V-->>AG: "username, password, lease_id, ttl 3600"

    AG->>AG: "Render template to tmpfs, signal app"
    P->>P: "Reload credentials without restart"

    loop "every 40 min"
        AG->>V: "renew lease and token"
    end
```

Three things this sequence makes explicit:

1. **The agent, not the application, talks to the secrets store.** The application reads a file or an environment-free in-memory path. This means the retry logic, the renewal loop, the caching and the failure handling live in one audited component rather than in 400 applications written in 11 languages.
2. **The audit entry is committed before the response is sent.** N7, on the critical path, by design.
3. **The application reloads without restarting.** This is the requirement that makes rotation survivable and it is the one most often missing — see the deep dive.

### Write path — a rotation

```mermaid
sequenceDiagram
    autonumber
    participant R as "Rotation job"
    participant V as "Secrets core"
    participant RF as "Raft"
    participant T as "Third-party API"
    participant AG as "Agents"

    R->>V: "POST secret/data/x with cas=7"
    V->>T: "Create NEW credential - both now valid"
    T-->>V: "new key id + secret"
    V->>RF: "Write version 8, retain 7"
    RF-->>V: "committed on quorum"
    V-->>R: "version 8"
    Note over AG: "overlap window begins - both work"
    AG->>V: "poll or watch, see version 8"
    AG->>AG: "render, signal app to reload"
    Note over AG: "wait for full convergence"
    R->>T: "Revoke OLD credential"
```

The overlap is the entire point: **create new, converge, then destroy old.** Reversing the order — destroy then create, or destroy before converging — is the mechanism behind nearly every rotation-caused outage.

---

## 7. Deep Dives

### 7.1 Sealed storage and the bootstrapping-trust problem

The chain: unseal mechanism → master key → key ring → every storage entry. Break the top link and everything below is unreadable ciphertext.

```mermaid
flowchart LR
    subgraph OPT1["Shamir quorum"]
        H1["Holder 1"] --> Q["k of n"]
        H2["Holder 2"] --> Q
        H3["Holder 3"] --> Q
        Q --> M1["Master key"]
    end
    subgraph OPT2["Cloud KMS"]
        IID["Instance identity"] --> K["KMS decrypt call"]
        K --> M2["Master key"]
    end
    subgraph OPT3["HSM"]
        PIN["HSM credential"] --> HS["PKCS11 unwrap"]
        HS --> M3["Master key"]
    end
```

=== "Shamir quorum (human)"

    $k$ of $n$ holders each enter a share; the master key is reconstructed in memory.

    **Security:** information-theoretically strong, and no single party — including the cloud provider, including your own platform team — can unseal alone. Compromise requires colluding with $k$ humans across (ideally) different organisational reporting lines.

    **Availability:** from the section 3 table, $n{=}5, k{=}3$ at $p{=}0.8$ gives **94.2%** ceremony success and roughly **15 minutes** when it works. Against a 4.32 min/month budget, one ceremony is three and a half months of budget.

    **The failure that actually happens:** shares live in password managers whose master passwords are themselves in the vault; a holder left the company and nobody rotated; a holder is on a plane; the runbook says "contact the key holders" and the contact list was in a wiki behind SSO that depends on the vault.

    **Chosen for:** the **recovery** path only — a break-glass mechanism used when auto-unseal is unavailable, exercised quarterly.

=== "Cloud KMS auto-unseal"

    The master key is stored encrypted by a KMS key. At startup the node authenticates to the KMS using its instance identity and calls `Decrypt`.

    **Security:** no human holds anything. Access is controlled by cloud IAM, is itself audited by the provider, and the KMS key can be given a policy restricting decryption to specific roles from specific VPC endpoints. The master key never exists outside the node's memory.

    **Availability:** unseal is automatic in **under 60 seconds** with no human in the loop. A node that reboots at 3 a.m. rejoins by itself.

    **What you give up:** the trust root moves to the cloud provider. Anyone who can assume the node's role and call `Decrypt` on that key can unseal — so the **cloud IAM policy on that one key becomes the most security-critical configuration in your company**. It also creates a hard dependency on a regional KMS endpoint, which is a real availability coupling.

    **Chosen** as the primary mechanism, with Shamir **recovery keys** retained for the case where the KMS key is deleted, the account is lost, or you need to migrate providers.

=== "HSM (PKCS#11)"

    A hardware module holds the wrapping key and performs the unwrap; the key material is non-exportable by construction.

    **Security:** the strongest available, and often a hard compliance requirement (FIPS 140-2/3 Level 3, PCI). Physical extraction is the attack you are defending against and the HSM is designed for it.

    **Costs:** expensive, slow to provision, awkward to make highly available (HSM clusters are their own operational discipline), and a hard dependency on a physical device in a specific place — which makes cross-region DR genuinely difficult.

    **Chosen for:** regulated environments where it is required. Otherwise the KMS path gets you most of the security for a fraction of the operational burden.

#### The full-restart scenario

```text
02:14  A control-plane upgrade goes wrong. Every node in the cluster restarts,
       including all secrets-core nodes.
02:15  All nodes come back SEALED. Process healthy, API returning 503.
02:15  Applications with cached credentials keep running normally.
       NOTHING IS VISIBLY BROKEN YET. This is the dangerous part.
02:16  New pods cannot authenticate. Deploys and scale-outs fail.
02:20  On-call is paged for "vault sealed". Opens the runbook.
02:22  Runbook says: contact 3 of 5 key holders. Contact list is in the wiki.
       The wiki is behind SSO. SSO's client secret comes from the vault.
02:35  Two holders reached. A third is found after escalating to a director.
02:48  Ceremony complete. Unsealed.
02:49  40,000 pods that have been retrying logins for 34 minutes all succeed
       within seconds: 40,000 logins, each a Raft write.
       The freshly-unsealed cluster is immediately saturated.
02:52  Login admission control sheds the excess; recovery takes another 6 min.
03:00  Fully recovered. Total: 46 minutes, of which 33 were the ceremony.
```

!!! danger "Every line of that timeline is a design lesson"
    **02:15 — sealed is silent.** Running workloads do not notice. The first symptom is a deploy failing, which looks like a CI problem. Alert on `vault_core_unsealed == 0` directly, as a page, within 60 seconds. Do not wait for a downstream symptom.

    **02:22 — the recovery path depended on the thing that was down.** The key-holder contact list must live somewhere that does not require the vault: a printed card in the on-call binder, a separate identity provider, an offline copy. Test this by asking, for each step of the runbook, "does this work if the vault is sealed?" The answer is usually no for at least one step, and that step is your real RTO.

    **02:35 — a 5-of-3 quorum is a distributed-systems availability problem with humans as the nodes.** Model it as such. The 94.2% number is not pessimism; it is the binomial.

    **02:49 — recovery causes a second incident.** Forty thousand clients that have been retrying for half an hour are a thundering herd. You need jittered client backoff and server-side login admission control, or the unseal is immediately followed by a self-inflicted overload.

    **The fix for all of it is auto-unseal.** It collapses 33 minutes into 45 seconds and removes the humans entirely. Keep Shamir recovery keys, practise the ceremony quarterly, but do not make it the primary path for a service with a 99.99% target.

### 7.2 Dynamic credentials and the blast-radius collapse

```mermaid
flowchart TB
    subgraph ST["Static credential"]
        S1["One password"] --> S2["In 40 pods, 3 config repos, CI, a wiki page"]
        S2 --> S3["Leaks to a log"]
        S3 --> S4["Undetected: median ~200 days"]
        S4 --> S5["Rotation requires coordinating every consumer"]
    end
    subgraph DY["Dynamic credential"]
        D1["Per-pod credential, TTL 1h"] --> D2["Leaks to a log"]
        D2 --> D3["Expires within 1h regardless"]
        D3 --> D4["Attribution: the audit log names the exact pod"]
    end
```

Three benefits, and the third is the one people forget:

**Exposure window collapses by ~4,800x.** From section 3: 200 days to 1 hour.

**Rotation becomes a non-event.** Rotating a static credential means finding every consumer, coordinating a change, and hoping you found them all. Dynamic credentials rotate continuously by construction — there is no rotation event to coordinate, because every credential is already ephemeral.

**Attribution becomes possible.** With a shared static password, a malicious query in the database log is attributable to "the application". With per-workload dynamic credentials, the database's own log contains `v-kube-orders-ro-8x2Kq-1742`, which the lease table maps to an entity, which the audit log maps to a specific pod, service account and request. **The forensic difference between "someone" and "checkout pod 7f9-xk2 at 09:22:41" is the difference between an unresolvable incident and a fifteen-minute investigation.**

#### What it costs

| Cost | Detail | Mitigation |
|---|---|---|
| Backend DDL load | 2.1 `CREATE ROLE`/s at 1 h TTL; 25/s at 5 min | TTL floor per backend; never a global TTL |
| Connection pool churn | A pool built on a credential must be rebuilt when it rotates | Overlap window + graceful pool drain, not a hard swap |
| Lease table growth | One durable row per active credential | It is bounded by $N \times \text{TTL}$, so short TTLs *shrink* it |
| Orphaned backend users | Lost lease rows leave real users behind | Native `VALID UNTIL` + a reconciliation job |
| Hard dependency at startup | No credential, no start | Agent caching + graceful degradation (7.5) |
| Application must reload | A credential that changes hourly cannot require a restart | 7.4 — and this is the real blocker to adoption |

!!! example "The migration order that actually works"
    Teams try to go straight from static to dynamic and stall, because the application cannot reload credentials without restarting and nobody will accept hourly restarts. The sequence that works:

    1. **Move the static secret into the vault, unchanged.** No behaviour change, but now it is audited, access-controlled and versioned. This alone is most of the compliance value and is a one-day change.
    2. **Add the agent sidecar** rendering it to a file. Still static, but the delivery mechanism is now in place and battle-tested.
    3. **Make the application reload on file change** without restarting. This is the hard engineering step and it is where the real work is. Do it while the credential is still static, so a bug costs nothing.
    4. **Switch to dynamic with a long TTL** (24 h). Reload is exercised daily.
    5. **Shorten the TTL** to the backend's floor.

    Attempting step 5 before step 3 is how these migrations fail, and it fails loudly at 3 a.m. one hour after the switch.

### 7.3 Identity-based auth and the irreducible chicken-and-egg

```mermaid
sequenceDiagram
    autonumber
    participant K as Kubelet
    participant P as Pod
    participant V as "Secrets core"
    participant API as "Cluster API"

    K->>P: "Project SA token into tmpfs - aud=vault, exp=10m"
    P->>V: "login with that token"
    V->>API: "TokenReview - is this token valid?"
    API-->>V: "valid: ns=commerce sa=checkout"
    V->>V: "Map to entity, resolve policies"
    V-->>P: "client_token"
```

The critical property: **the vault does not trust the claim, it verifies the token against the issuer's own signing key.** The pod cannot forge it because it cannot sign with the cluster's key. The token is audience-bound so a token minted for one service cannot be replayed at the vault. It is short-lived so a stolen one is useful for minutes. And it is projected into tmpfs, so it is not in etcd and not in a disk snapshot.

#### Where the chain bottoms out

```mermaid
flowchart TB
    APP["Application"] -->|"proves with"| SA["Projected SA token"]
    SA -->|"signed by"| CK["Cluster signing key"]
    CK -->|"protected by"| CP["Control plane"]
    CP -->|"provisioned by"| TF["Infrastructure automation"]
    TF -->|"authenticated by"| CIAM["Cloud IAM role"]
    CIAM -->|"assumed via"| INST["Instance identity document"]
    INST -->|"signed by"| HW["Provider hardware root"]
    HW --> TURTLE["The bottom: trust in the provider"]
```

Every link removes a stored secret and replaces it with an attestation. But **you cannot remove the last link.** Somewhere there is a root: trust in the cloud provider's hardware attestation, or a TPM's endorsement key, or — in an on-premises environment — an operator who physically typed something once.

Honest positions on the irreducible root:

| Environment | The root | Honest assessment |
|---|---|---|
| Cloud | The provider's instance identity signing key | You already trust the provider with the VM's memory. This is not a new trust assumption |
| On-prem with TPM | The TPM endorsement key, burned at manufacture | Strong, but you now trust the TPM vendor and the supply chain |
| On-prem without TPM | An operator placing a token on first boot | A real secret, placed by a human. Minimise it: one-time-use, short-lived, network-scoped, alerted on every use |
| CI/CD | The CI platform's OIDC issuer | Workload identity federation removes the stored secret; you trust the CI provider's issuer instead |

!!! tip "The right framing for the chicken-and-egg question"
    The goal is not to eliminate the root of trust — that is impossible, and a candidate who claims otherwise is wrong. The goal is to **make the irreducible secret as small, short-lived and non-transferable as possible**, and then to **detect its use**.

    Concretely: one credential rather than five thousand; single-use rather than reusable; bound to a machine or network location rather than portable; valid for minutes rather than years; and alerting on *every single use*, because in a healthy system it is used approximately never. A bootstrap credential that fires an alert every time it is used converts an undetectable compromise into a paged incident.

    That is a defensible, senior answer. "We use workload identity so there are no secrets" is not, because it is false.

### 7.4 Rotation without downtime

The naive rotation — change the credential, hope everyone picks it up — fails because propagation is not instantaneous and in-flight work exists.

```mermaid
flowchart LR
    subgraph BAD["Rotate in place"]
        B1["Change password"] --> B2["Old fails immediately"]
        B2 --> B3["Every pool breaks until reload"]
    end
    subgraph GOOD["Overlap window"]
        G1["Create new, keep old"] --> G2["Both valid"]
        G2 --> G3["Converge: all consumers on new"]
        G3 --> G4["Verify: zero old usage"]
        G4 --> G5["Revoke old"]
    end
```

The overlap window must satisfy:

$$
W > t_{\text{detect}} + t_{\text{reload}} + t_{\text{in-flight}} + t_{\text{stragglers}}
$$

$$
\begin{aligned}
t_{\text{detect}} &= \text{agent poll interval} = 60\ \text{s} \\
t_{\text{reload}} &= \text{connection pool drain and rebuild} = 120\ \text{s} \\
t_{\text{in-flight}} &= \text{longest request holding a connection} = 300\ \text{s} \\
t_{\text{stragglers}} &= \text{a pod that was down during the window} = ?\ \text{(unbounded)} \\
\hline
W &\geq 8\ \text{min},\ \text{choose } \mathbf{24\ \text{h}}
\end{aligned}
$$

The straggler term is why 24 hours rather than 10 minutes: a pod that was `CrashLoopBackOff` during the window, a batch job that runs nightly, a canary sitting at 1% that nobody restarted. **Long overlaps cost almost nothing and prevent the entire class of "we rotated and three obscure things broke" incidents.**

!!! danger "Never revoke on a timer — revoke on verified convergence"
    The rotation job must not simply sleep 24 hours and then revoke. It must **verify that the old credential is no longer in use** before revoking: query the backend for active sessions authenticated as the old user, check the audit log for reads of the old version, and confirm that every known consumer has acknowledged the new version. Only then revoke.

    Revocation on a timer means that a consumer which failed to converge loses access at a moment chosen by a cron schedule rather than by anyone watching — typically hours after everyone stopped paying attention. Verified convergence turns a silent time bomb into an alert that says "three pods still using the old credential", which someone can act on.

#### Application reload without restart

This is where rotation projects actually stall.

```go
// The pattern: credentials behind an atomic pointer, swapped on file change.
// Every caller reads through the accessor, so the swap is invisible.
type CredStore struct{ v atomic.Pointer[Creds] }

func (c *CredStore) Get() *Creds { return c.v.Load() }

func (c *CredStore) watch(path string) {
    for range fileChanges(path) {
        next, err := load(path)
        if err != nil {
            metrics.CredReloadFailures.Inc()
            continue          // CRITICAL: keep the old creds on a bad read.
        }                     // A partial file must never blank the credential.
        if err := verify(next); err != nil {
            metrics.CredVerifyFailures.Inc()
            continue          // Verify BEFORE swapping, not after.
        }
        c.v.Store(next)
        pool.DrainGracefully() // New connections use new creds; in-flight finish.
        metrics.CredReloads.Inc()
    }
}
```

Four details that separate a working implementation from one that causes an outage:

1. **Verify before swapping.** Test the new credential against the backend first. Swapping to a credential that does not work turns a rotation into an outage with no rollback.
2. **Keep the old credentials on any error.** A truncated file mid-write, a permission problem, a parse failure — all must leave the working credential in place. Blanking on error is the most common bug in this code.
3. **Drain, do not kill.** New connections use the new credential; existing ones finish their work. Killing the pool mid-request converts rotation into user-visible errors.
4. **Emit metrics for reloads, failures and credential age.** A pod whose credential age exceeds the TTL is about to fail and is currently invisible. `credential_age_seconds` with an alert at 80% of TTL is the single most useful metric in this subsystem.

### 7.5 Availability, caching, and graceful degradation

```mermaid
flowchart TB
    OUT["Secrets store unavailable at T0"] --> C1["T0: running pods unaffected - cached creds"]
    C1 --> C2["T0: new pods cannot start"]
    C2 --> C3["T0+40min: first renewals fail"]
    C3 --> C4["T0+60min: first leases expire"]
    C4 --> C5["T0+60min onward: failures spread as a wave"]
    C5 --> C6["T0+24h: everything is down"]
```

**The blast radius is a clock, not an explosion**, and understanding that shapes the entire response.

| Time since outage | What breaks | Severity |
|---|---|---|
| 0–15 min | Deploys, scale-outs, new pods | Annoying; no user impact |
| 15–40 min | Batch jobs and CI start failing | Visible internally |
| 40–60 min | Renewals fail; agents start warning | The clock is now audible |
| 60 min+ | Leases expire; database connections rejected | **User-visible, spreading** |
| 24 h | Everything with a long TTL is gone | Total |

That one-hour fuse is the argument for the TTL choice in N9, and it is a deliberate reliability decision rather than a security one: **a 1-hour TTL gives humans an hour to fix the secrets store before user impact begins.** A 15-minute TTL gives them fifteen minutes. A 24-hour TTL gives them a day but makes a leaked credential useful for a day. The TTL is the dial between "how long does a stolen credential work" and "how long do I have to fix an outage", and the honest answer is that it should be set per backend by weighing exactly those two.

#### Degradation strategy

```mermaid
flowchart LR
    A["Agent"] --> C{"Store reachable?"}
    C -->|yes| N["Normal: renew, refresh"]
    C -->|no| D["Degraded"]
    D --> D1["Serve cached creds until expiry"]
    D --> D2["Retry with jittered exponential backoff"]
    D --> D3["Emit credential_age and time_to_expiry"]
    D --> D4["Alert at 50 pct of remaining TTL"]
    D --> D5["Never blank an existing credential"]
```

!!! warning "The cache must be memory-only, and that is a real trade-off"
    Persisting cached credentials to disk would let a pod survive a restart during an outage — genuinely useful. It would also put plaintext credentials on disk, which is the exact thing the whole system exists to prevent.

    The resolution is asymmetric by criticality: **memory-only by default**, accepting that a pod restarting during an outage cannot start. For a small set of tier-0 services where that is unacceptable, use an **encrypted on-disk cache whose key comes from the platform** (a TPM-sealed key, or a KMS call the node can make with its instance identity), so the cached credential is unreadable without the same attestation that would have authenticated the pod anyway. Do not make that the fleet-wide default — the complexity and the risk are only justified where a cold start during an outage is genuinely intolerable.

    Also make the failure mode explicit at the application layer: a service that cannot get a database credential should serve read-only from cache, or shed load with a clear error, rather than crash-looping. Crash-looping during a secrets outage generates a login storm that prevents recovery.

### 7.6 Encryption as a service

```mermaid
flowchart LR
    APP["Application"] -->|"plaintext"| T["Transit engine"]
    T -->|"vault:v3:..."| APP
    APP --> DB[("Database stores ciphertext")]
    T --- K["Key never leaves"]
```

Compare with the alternative of the application holding a key:

| Property | App holds the key | Encryption as a service | Chosen / rejected |
|---|---|---|---|
| Key exposure | In app memory, in config, in core dumps, in heap snapshots | Never leaves the vault | **Chosen** — this is the whole point |
| Rotation | Requires re-encrypting all data, coordinated across every consumer | New key version; old ciphertext still decrypts; `rewrap` in the background | **Chosen** |
| Audit | Invisible — the app decrypts locally and nobody knows | Every decrypt is an audited, attributable event | **Chosen** |
| Latency | Microseconds, local | A network round trip: ~2–5 ms | Rejected as a blocker: batch and cache |
| Availability | Independent of the vault | **Every decrypt becomes a hard dependency** | The serious cost |
| Throughput | Unlimited | Bounded by the vault's capacity | Batch operations; `datakey` for bulk |

The availability coupling is the real objection, and the answer is the **data-key pattern** for bulk work:

```bash
# Get a data key: plaintext for immediate local use, wrapped for storage.
POST /v1/transit/datakey/plaintext/pii
-> { "plaintext": "<base64 AES key>",
     "ciphertext": "vault:v3:<wrapped key>" }

# Encrypt a million rows LOCALLY with the plaintext key, store the wrapped
# key alongside. One vault call, then local AES-GCM at GB/s.
# To decrypt later: one vault call to unwrap, then local decryption.
```

This keeps the key out of long-term application storage (only the wrapped form is persisted), bounds the vault calls to one per batch rather than one per row, and preserves auditability at batch granularity. **Use `encrypt`/`decrypt` for individual high-value items where per-item audit matters; use `datakey` for bulk where it does not.**

---

## 8. Scaling the Bottleneck

The bottleneck is **writes through the single active node's consensus log** — token creation, lease creation, lease renewal. Reads scale with performance standbys; writes do not scale at all ([F09](../fundamentals/f09-consensus.md)).

$$
\text{write load} = \underbrace{\frac{N_{\text{tokens}}}{\text{TTL}_{\text{token}} \cdot \frac{2}{3}}}_{\text{renewals}} + \underbrace{\frac{N_{\text{leases}}}{\text{TTL}_{\text{lease}} \cdot \frac{2}{3}}}_{\text{renewals}} + \underbrace{\lambda_{\text{login}}}_{\text{churn + storms}}
$$

| Lever | Mechanism | Effect | Cost |
|---|---|---|---|
| **Longer TTLs** | 1 h → 4 h | Write load drops 4x | Longer exposure window for a leaked credential |
| **Batch token renewal** | Agent renews token and all leases in one request | ~3x fewer writes | Partial failure semantics get harder |
| **Performance standbys** | Serve reads locally, forward writes | Linear read scale-out | Zero help for writes |
| **Agent-side caching** | Cache and renew locally; do not re-read unchanged secrets | Eliminates most reads entirely | Staleness on rotation; needs a version check |
| **Batch tokens** | Lightweight, non-renewable, non-persisted tokens | **Not a write at all** | Cannot be renewed or individually revoked |
| **Login admission control** | Bounded login acceptance rate during a storm | Prevents recovery-time collapse | Some clients wait; they must back off with jitter |
| **Namespace sharding** | Separate clusters per tenant or per tier | Independent write ceilings; blast-radius isolation | No cross-namespace identity or policy |
| **Regional replicas** | Performance replicas serving local reads and local token creation | Cross-region read latency solved | Writes still funnel to primary; DR replicas have replication lag |

!!! tip "Batch tokens are the highest-leverage optimisation and the one nobody knows about"
    A normal token is **persisted to the consensus log** so it can be renewed, revoked individually, and have children. That makes every login a write, and logins are the dominant write source during any storm.

    A batch token is a cryptographically signed blob validated without a storage lookup. Creating one is **not a write**. The trade-offs are real: it cannot be renewed, cannot be individually revoked (only by revoking its parent or its whole mount), and cannot create children.

    For short-lived workloads — CI jobs, batch pods, anything living less than the token TTL — batch tokens are exactly right, and they remove that population from the write path entirely. If 60% of your logins come from short-lived jobs, that is a 60% reduction in the write load of your most constrained component, and during a restart storm it is the difference between 333 writes/s and 133.

### Login storm control

The critical scenario: 40,000 pods restarting or a regional failover, generating 333 logins/s against a ~3,000/s write ceiling. Not fatal in isolation — but it follows an outage, so it lands on a cold cluster while the clients have all been retrying in lockstep.

```mermaid
flowchart TB
    S["40k pods need tokens"] --> AC{"Admission control"}
    AC -->|"within budget"| OK["Login proceeds"]
    AC -->|"over budget"| R["429 with Retry-After"]
    R --> J["Agent: jittered exponential backoff"]
    J --> AC
    OK --> W["Raft write"]
```

Three controls, all necessary: a **server-side login rate limit** returning `429` with `Retry-After` rather than timing out (a timeout gives the client no information and it retries immediately); **jittered exponential backoff in the agent**, spreading a synchronized herd over minutes; and **batch tokens for short-lived workloads**, removing them from the write path entirely. Serving 5,000 pods correctly while telling 35,000 to wait is far better than failing all 40,000.

---

## 9. Failure Modes

| Failure | Blast radius | Detection | Mitigation | Degraded behaviour |
|---|---|---|---|---|
| **All nodes sealed after restart** | Nothing starts; a 1-hour fuse before running workloads break | `vault_core_unsealed == 0` — page in 60 s | Auto-unseal via KMS; Shamir only as recovery; runbook independent of the vault | Cached credentials keep running workloads alive until their TTL |
| **Unseal quorum unreachable** | Total, extended outage | Ceremony not completing | $n{=}7, k{=}3$ if human; offline contact list; quarterly drills | None — this is a hard down until humans converge |
| **Cloud KMS unavailable in-region** | Cannot unseal or restart nodes | KMS API error rate | Multi-region KMS key replication; Shamir recovery keys as the fallback | Already-unsealed nodes keep serving; only restarts are blocked |
| **Active node fails** | 5–15 s of write unavailability | Leader election duration | Standby promotion; clients retry | Reads continue from standbys throughout |
| **Storage quorum lost** | Total outage; no reads, no writes | Raft peer count | 5 nodes across 3 AZs; never 2 nodes in one failure domain | Hard down. Restore quorum; never force a single-node quorum |
| **Audit device failure** | **All requests refused** (N7) | Audit write error rate | Two independent devices, require ≥1 success | Deliberate: refuse rather than serve unaudited |
| **Backend (database) unreachable** | Dynamic credentials for that backend fail | Credential issuance error rate by backend | Per-backend circuit breaker; existing leases unaffected | Only new credentials for that backend fail; everything else works |
| **Lease table inconsistency** | Orphaned users accumulate in backends | Reconciliation job drift | Native `VALID UNTIL`; periodic reconciliation | Backend cleans up on its own; the drift alert finds the rest |
| **Rotation revokes before convergence** | Every consumer that had not reloaded is down | Auth failure spike on the backend at the revoke timestamp | Verified convergence before revoke, never a timer; long overlap | Immediate outage for stragglers; rollback means recreating the old credential |
| **Credential leaked into logs** | Depends entirely on TTL | Secret scanning in log pipelines and repos | Short TTLs; HMACed audit; scanning at ingest | With a 1 h TTL, bounded. With a static credential, unbounded |
| **Login storm after recovery** | The just-recovered cluster saturates again | Login rate vs write ceiling | Admission control with `429`; jittered agent backoff; batch tokens | Some pods wait minutes; the cluster survives |
| **Policy misconfiguration (over-grant)** | Silent — nothing breaks, which is the problem | Policy diff in CI; anomaly on newly-accessed paths per identity | Deny-by-default; templated paths; CI review of capability additions | No symptom. Found only by review or after a breach |
| **Policy misconfiguration (under-grant)** | That workload fails | Denial rate by identity | Staged policy rollout; denials logged | Loud and obvious — the safe direction to fail |
| **Root token left active** | Total compromise if leaked | Alert on any root-token use, always | Generate only for break-glass, revoke immediately after | Any use is an incident until proven otherwise |
| **Region loss** | Local workloads lose the vault | Regional health | DR replica with RPO < 30 s; documented promotion | Promotion loses < 30 s of leases; those clients re-request |

---

## 10. SRE Lens

### SLIs and SLOs

| SLI | Definition | SLO | Rationale |
|---|---|---|---|
| Availability | Non-5xx, unsealed, quorum present | **99.99%** | Bottom of the dependency graph |
| Sealed time | Minutes with `unsealed == 0` | **0 unplanned** | Sealed is a total outage with a delayed fuse |
| Read latency | p99 secret read | < 50 ms | On pod startup paths |
| Dynamic credential latency | p99 issuance | < 500 ms | Bounded by backend DDL |
| Login success rate | Successful / attempted, excluding policy denials | > 99.95% | Denials are a separate, deliberate signal |
| Audit completeness | Requests served with a committed audit record | **100%** | Non-negotiable; a gap is unprovable |
| Unseal time (auto) | Process start to unsealed | < 60 s | N5 |
| Unseal time (human) | Ceremony duration | < 20 min | N6, and measured at drills |
| Lease reconciliation drift | Backend users with no lease row | 0, alert on any | Orphan detection |
| Credential age at the client | Time since last successful refresh | < 80% of TTL | The leading indicator of a client-side failure |
| Rotation success | Rotations completing with zero auth failures | 100% | N11 |
| Write headroom | Sustained writes / ceiling | < 30% | Storms need room |
| DR replication lag | Primary to DR replica | < 30 s | N10 |

!!! tip "`credential_age_seconds` at the client is the most underrated metric here"
    Server-side metrics tell you the vault is healthy. They cannot tell you that 40 pods stopped renewing an hour ago and will fail when their leases expire — from the server's view, those pods simply stopped asking, which is indistinguishable from them having shut down.

    Exporting credential age (and time-to-expiry) from every agent, with an alert at 80% of TTL, turns a future outage into a present warning. It is also the only way to detect the "rotation happened but three stragglers never converged" case *before* revocation, which is the failure mode of 7.4.

### Error budget policy

$$
\begin{aligned}
\text{budget} &= (1 - 0.9999) \times 30\ \text{days} = \mathbf{4.32\ \text{min/month}} \\
\text{one human unseal ceremony} &\approx 15\ \text{min} = \mathbf{347\%\ \text{of the monthly budget}} \\
\text{one auto-unseal} &\approx 45\ \text{s} = 17\% \\
\text{one active-node failover} &\approx 10\ \text{s of write unavailability} = 3.9\%
\end{aligned}
$$

That first line is the whole argument, expressed as a number a director understands: **a single human unseal event consumes three and a half months of error budget.** Auto-unseal is not a convenience feature; it is the difference between meeting the SLO and not.

Policy consequences:

- **Changes are conservative and rare.** Upgrades are quarterly, soaked in a mirror cluster for two weeks, rolled one node at a time with the active node last.
- **Policy changes are a separate, stricter class.** An over-permissive policy has no symptom, so CI computes a capability diff and any widening requires named security review.
- **Every root-token generation is an incident record**, reviewed weekly, even when legitimate.
- **The budget is shared.** Burning it burns a slice of every dependent service's budget at once.

### Rollout plan

```yaml
version_upgrade:
  - stage: mirror
    description: >
      Shadow cluster restored from a production snapshot, replaying captured
      traffic at 2x. Verify storage format compatibility BOTH directions --
      a one-way upgrade has no rollback and that must be a recorded decision.
    duration: 14d
    gate: [zero unseal anomalies, p99 within 10pct, audit format unchanged]

  - stage: dr_replica
    description: Upgrade the DR replica first. It serves no live traffic.
    gate: replication healthy 48h

  - stage: standbys
    description: >
      One standby at a time, verifying it rejoins Raft and reaches full
      log sync before touching the next. Never two nodes out at once.
    gate_each: [raft_peers == expected, unsealed, log index caught up]

  - stage: active_last
    description: Controlled step-down to an upgraded standby, THEN upgrade the old active.
    gate: [failover < 15s, zero failed logins]

policy_change:
  - stage: ci_validate
    checks: [hcl syntax, capability diff, "does this widen any path?", deny-rule preservation]
  - stage: audit_mode
    description: >
      Apply the new policy in a mode that LOGS what it would allow or deny
      without enforcing, for 24h. Compare against actual traffic.
    gate: zero unexpected denials in the simulated evaluation
  - stage: enforce
    gate: denial rate by identity within baseline for 1h
  rollback: previous policy version, applied in seconds

secret_rotation:
  - create_new_keep_old
  - wait_for_convergence     # measured, not timed
  - verify_zero_old_usage    # backend sessions + audit log
  - revoke_old
  overlap_window: 24h
```

### Runbook notes

| Symptom | First checks | Action |
|---|---|---|
| "Vault is sealed" | Auto-unseal configured? KMS reachable from the node? IAM role intact? Was the KMS key policy changed? | 95% of the time it is KMS access, not the vault. Check the KMS call's error before assembling humans |
| Deploys failing, running pods fine | Sealed state; login error rate; policy change in the last hour | The signature of a sealed or newly-misconfigured vault. **You have roughly one TTL before it becomes user-visible** — use that hour |
| Auth failures spiking on a database | Was a credential just revoked? Rotation job history; lease count for that role | A rotation that revoked before convergence. Recreate the old credential immediately; investigate afterwards |
| Orphaned database roles growing | Reconciliation drift; lease table health; revocation statement errors | Check whether `revocation_statements` handles owned objects — `DROP ROLE` fails if the role owns anything, and the failure is often silent |
| Login rate at the ceiling | Is this a restart storm? Admission control engaged? Batch tokens in use? | Enable admission control before adding capacity. Shedding with `429` beats collapsing |
| Audit device failing | Disk full? Remote sink reachable? | **The system will refuse requests.** Fix or disable the failed device — but disabling audit is a security decision requiring sign-off, not an on-call judgement call |
| Someone used the root token | Audit entry: which identity, which path, which source IP | Treat as an incident until proven otherwise. Revoke, rotate, review |
| Credential age alerts on a subset of pods | Are those pods' agents healthy? Network path to the vault? Policy change affecting that identity? | Catches the straggler case *before* a revocation turns it into an outage |

### Capacity model

$$
\begin{aligned}
\text{core nodes} &= 5\ (\text{1 active} + 2\ \text{perf standby} + 2\ \text{Raft-only}),\ \text{3 AZs} \\
\text{write headroom} &= \frac{333\ \text{/s failover peak}}{3{,}000\ \text{/s ceiling}} = 11\% \\
\text{read capacity} &= 3\ \text{serving nodes} \times 30\text{k/s} = 90\text{k/s} \gg 2{,}500\ \text{peak} \\
\text{storage} &= 20\text{k secrets} + 40\text{k tokens} + 5\text{k leases} \approx 400\ \text{MB} \\
\text{audit} &= 69\ \text{GB/day},\ 176\ \text{TB at 7-year retention}
\end{aligned}
$$

Read capacity is 36x the peak requirement, which is correct: the expensive failure is not being slow, it is being unavailable, and read headroom is cheap. Write headroom at 11% is the number that actually needs watching, and it is entirely a function of the TTL choice.

### Cost

| Component | Sizing | Monthly |
|---|---|---|
| Core cluster | 5 × 8 vCPU / 32 GB / NVMe, 3 AZs | $3.2k |
| DR replica cluster | 3 nodes, second region | $1.6k |
| Cloud KMS | Key + ~50k decrypt calls | $0.1k |
| Audit hot storage | 30 days, 2 TB, searchable | $1.1k |
| Audit cold storage | 7 years, 176 TB, object storage | $0.7k |
| Agent sidecars | 40,000 × 0.02 vCPU / 24 MB | $23.4k |
| Monitoring | | $0.5k |
| **Total** | | **~$30.6k/month** |

!!! tip "The sidecars are 76% of the bill, and that is the number to attack"
    The vault cluster itself costs $3.2k. The 40,000 agent sidecars cost $23.4k — seven times more. This is the same structural pattern as any per-pod component: the fleet multiplier dominates.

    Two levers. **Node-level agent** instead of per-pod: one agent per node serving all pods on it via a local socket, with pod identity established by the kubelet rather than by the pod's own token. Forty thousand sidecars become 2,000 node agents: roughly $18k/month saved. The cost is a shared failure domain per node and a softer identity boundary — the same trade-off as a sidecar-less service mesh, and worth making for the same reasons.

    **Skip the agent for short-lived workloads** and use batch tokens with a direct API call. A CI job living 90 seconds does not need a renewal loop.

    What does *not* work is reducing audit retention, which is the smallest meaningful line on the bill and the one thing you cannot reconstruct after a breach.

---

## 11. Trade-offs & Alternatives

| Decision | Chosen | Rejected | Why |
|---|---|---|---|
| Unseal mechanism | **Cloud KMS auto-unseal, Shamir as recovery** | Shamir as primary | 94.2% ceremony success and 15 minutes when it works — 347% of the monthly error budget per event |
| | | HSM | Correct where compliance requires it; otherwise high cost and hard cross-region DR for marginal extra security |
| Shamir parameters (recovery) | **$n{=}7, k{=}3$** | $n{=}5, k{=}3$ | 99.5% vs 94.2% availability at the same attacker threshold |
| | | $n{=}5, k{=}2$ | Two colluding insiders is too low a bar for the company's root of trust |
| Credential model | **Dynamic, per-workload, 1 h TTL** | Static secrets | ~4,800x exposure-window reduction, plus per-workload attribution in backend logs |
| TTL | **Per-backend: 1 h DB, 15 min cloud IAM, 1 h PKI** | One global TTL | The floor is set by the backend's minting cost — DDL rate for databases, nothing for STS |
| Auth | **Platform attestation (SA token, IID, SPIFFE)** | A bootstrap secret per workload | Concentrating 5,000 secrets into one is not removing them |
| Client integration | **Agent sidecar or node agent** | Direct SDK calls from every application | Retry, renewal, caching and failure handling written once and audited, not 400 times in 11 languages |
| Agent topology | **Node-level agent** | Per-pod sidecar | 76% of the bill is sidecars; node-level saves ~$18k/month with a mesh-style isolation trade-off |
| Audit | **Synchronous, multiple devices, ≥1 must succeed** | Asynchronous audit | A log with gaps cannot prove what an attacker did not access |
| | | A single device | One disk-full event becomes a company-wide credential outage |
| Cache | **Memory only; encrypted disk cache for tier-0 only** | Disk cache everywhere | Plaintext credentials on disk defeats the system's core purpose |
| Rotation | **Overlap window, revoke on verified convergence** | Rotate in place | Propagation is not instantaneous and stragglers are unbounded |
| | | Revoke on a 24 h timer | Stragglers lose access at a cron-chosen moment nobody is watching |
| Encryption | **Transit for high-value items, data keys for bulk** | App-held keys | Keys in application memory, invisible decrypts, coordinated re-encryption on rotation |
| | | Transit for every row | A network round trip per row, and every decrypt becomes a hard dependency |
| Tokens | **Batch tokens for short-lived workloads** | Service tokens for everything | Every login is a consensus write; batch tokens remove that population entirely |
| Storage | **Integrated Raft** | External consensus store | One fewer operational surface and one fewer thing that can be down |
| Multi-region | **DR replica with documented promotion** | Stretched cluster across regions | A remote voter puts the cross-region RTT on every write in the system |
| Policy | **Deny by default, templated by identity** | Enumerate every workload | Thousands of near-identical policies is unmaintainable and drifts |

??? note "Cloud-native secret managers versus running your own"
    **AWS Secrets Manager / Google Secret Manager / Azure Key Vault** are genuinely good and, for most companies, the right answer. Managed availability, no unseal ceremony, IAM integration that already exists, rotation Lambdas, and no cluster to operate. If you are single-cloud and your needs are "store secrets, rotate them, control access", building and operating your own is hard to justify.

    **Where a self-managed vault wins:** multi-cloud or hybrid, where one system spans every environment with a single policy model and a single audit trail. Dynamic credentials for a much wider set of backends, including your own internal services. Encryption as a service with your own key hierarchy. An audit log you control end to end. And the ability to run in environments where the cloud provider is not in your trust boundary.

    **The honest position:** the cloud-native option is the correct default. Reach for self-managed when you have a specific requirement it cannot meet — usually multi-cloud, a compliance boundary, or dynamic credentials against internal systems — and be aware that you have taken on a 99.99% service with an unseal ceremony.

??? note "Why not just use Kubernetes Secrets?"
    Kubernetes `Secret` objects are base64-encoded, not encrypted — a widespread misunderstanding. With encryption-at-rest enabled they are encrypted in etcd, which is better, but the fundamental problems remain.

    They are **readable by anyone with `get secrets` in the namespace**, which is a wide and hard-to-audit set once you include controllers and operators. They are **static and long-lived**, so they carry the full 200-day exposure window. They are **mounted as files or environment variables** inside the pod, so they leak into crash dumps, `/proc` listings, and any library that logs its configuration. There is **no audit trail of reads** — you cannot answer "who read this secret and when", which is the first question in every investigation. **Rotation requires a pod restart**, so nobody does it. And they appear in **etcd backups**, which are frequently stored with far weaker controls than the secrets themselves.

    The pragmatic middle ground is a **secrets-store CSI driver or an external-secrets operator** that syncs from a real secrets manager into the pod. You keep the Kubernetes-native ergonomics and gain central policy, audit and rotation, at the cost of one more controller. That is a reasonable first step and is strictly better than raw Secrets, though it still lands a static credential in the pod rather than an ephemeral one.

---

## 12. Gotchas & Corner Cases

!!! gotcha "Everything restarts at 3 a.m. and the vault comes back sealed"
    **Symptom.** Running workloads are fine. Deploys fail. An hour later, database connections start being rejected across unrelated services, and the failures spread as leases expire in a wave.
    **Mechanism.** A sealed vault is a *running, healthy-looking* process that cannot decrypt its own storage. There is no crash, no restart loop, and often no alert — the container is `Ready`. The first observable symptom is a failing deploy, which gets triaged as a CI problem, burning the one hour of grace that cached credentials bought you.
    **Mitigation.** Alert on `vault_core_unsealed == 0` directly, as a page, within 60 seconds — never on a downstream symptom. Use auto-unseal so the node recovers by itself in under a minute. And rehearse the timeline: the gap between "sealed" and "users notice" is exactly one TTL, and that is your entire response window.

!!! gotcha "The unseal runbook depends on the thing that is down"
    **Symptom.** During a seal incident, the on-call opens the runbook and every step leads to a system that cannot be reached: the key-holder contact list is in a wiki behind SSO, SSO's client secret is in the vault, the escalation rota is in a tool that authenticates through the same path.
    **Mechanism.** Recovery procedures are written by people with working access to everything, and are never tested in the state they are meant to handle. The circular dependency is invisible until the circle breaks.
    **Mitigation.** Walk the runbook step by step asking "does this work while the vault is sealed?" of each line. Whatever fails is your real RTO. Keep an offline copy of the key-holder list — a printed card in the on-call binder is not a joke — and an independent authentication path for the break-glass procedure. Then run the drill quarterly with the primary path deliberately disabled.

!!! gotcha "Rotation revoked the old credential before every consumer had reloaded"
    **Symptom.** A rotation completes cleanly. Twenty-four hours later, at the exact moment the revoke job fires, three services start failing authentication — including a nightly batch job and a canary deployment nobody had restarted.
    **Mechanism.** The rotation job slept for the overlap window and then revoked on a timer. A pod that was in `CrashLoopBackOff` during the window, a job that runs once a day, and a 1% canary never picked up the new version. Their failure is triggered by a cron schedule, hours after anyone was watching the rotation.
    **Mitigation.** **Never revoke on a timer. Revoke on verified convergence.** Query the backend for active sessions authenticating as the old identity, check the audit log for reads of the old version, and confirm every known consumer has acknowledged the new one. If anything is outstanding, alert with the specific list rather than revoking. A long overlap window costs essentially nothing; a premature revoke is an outage with an awkward rollback.

!!! gotcha "The application cannot reload credentials without restarting"
    **Symptom.** A dynamic-credentials migration stalls indefinitely. Teams agree with the security goal and cannot adopt it, because a 1-hour TTL would mean hourly restarts.
    **Mechanism.** The application reads its credential once at startup into a connection pool. Nothing in its design anticipated the credential changing while the process lives. Retrofitting this touches the connection pool, the config loader, and every code path that assumed the credential was immutable.
    **Mitigation.** Sequence the migration so reload is built and proven *while the credential is still static*: move the secret into the vault, add the agent, make the app reload on file change, then switch to dynamic with a 24-hour TTL, then shorten it. Implement reload with an atomic pointer swap, verify the new credential before swapping, **keep the old one on any error**, and drain the pool gracefully rather than killing it. Export `credential_age_seconds` so a pod that silently stopped reloading is visible before it fails.

!!! gotcha "Secrets leak into logs, environment variables and crash dumps"
    **Symptom.** A credential appears in a log aggregator, in a stack trace, in an error-tracking service, or in a `kubectl describe` output pasted into a ticket. This is the most common real-world leak vector by a wide margin — far more common than an attacker breaking the vault.
    **Mechanism.** The credential is passed as an environment variable, so it appears in `/proc/<pid>/environ`, in crash dumps, in `docker inspect`, in any library that logs its configuration at startup, and in child processes that inherit the environment. Or a debug log statement prints a config struct. Or an exception message includes the connection string.
    **Mitigation.** **Never use environment variables for secrets** — use a file on tmpfs, read once, or an in-memory fetch. Wrap credential types so their `String()`/`repr()`/`toString()` returns a redaction marker, which stops the entire class of accidental logging at the type level. Scan logs at ingest for credential patterns and alert. Scan repositories continuously. And keep TTLs short so that when it happens anyway — it will — the window is an hour rather than two hundred days.

!!! gotcha "The audit device filled its disk and the entire vault stopped serving"
    **Symptom.** Every credential request in the company fails simultaneously. The vault is unsealed, quorum is healthy, the storage backend is fine, and the only error is about audit logging.
    **Mechanism.** The system refuses to serve a request it cannot audit (N7). With a single audit device, a full disk, a permission change, or an unreachable remote sink becomes a total outage. This is the design working exactly as intended, which makes it especially frustrating during the incident.
    **Mitigation.** Configure **at least two independent audit devices** — a local file and a remote socket — and require only one to succeed, so a single device's failure degrades redundancy rather than availability. Monitor audit-device write errors as a first-class SLI. Alert on the audit filesystem at 70% rather than 90%. And decide *in advance*, in writing, who is authorised to disable a failing audit device during an incident, because that is a security decision and it should not be made by a tired on-call engineer at 4 a.m.

!!! gotcha "Orphaned database roles accumulate for months"
    **Symptom.** A database has 14,000 roles matching the vault's naming prefix, most expired, some not. Nobody can safely delete them because nobody knows which are still in use. `pg_authid` is bloated and catalog queries have slowed.
    **Mechanism.** Two independent causes. Lease rows were lost or never written, so nothing ever revokes the corresponding role. Or the revocation statement failed silently — `DROP ROLE` fails if the role owns any objects, and if the statement does not first `REASSIGN OWNED` and `DROP OWNED`, the drop errors and the role remains.
    **Mitigation.** Write revocation statements that handle owned objects explicitly. Set a native `VALID UNTIL` slightly beyond `max_ttl` so the database enforces expiry even if the broker never asks. Run a **reconciliation job** that lists prefix-matching roles in each backend and diffs them against the lease table, alerting in both directions — orphans in the backend, and leases with no corresponding role. This job is routinely skipped and is the only thing that finds the problem before it is measured in tens of thousands.

!!! gotcha "The root token from the initial setup is still valid two years later"
    **Symptom.** A security review finds an unexpired root token in a password manager, a wiki page, a Terraform state file, or a CI variable. It has full access to everything and bypasses all policy.
    **Mechanism.** The initialisation procedure produces a root token. It is used to configure auth methods and policies, and then — because everything is working and nobody wants to touch it — it is never revoked. It persists through team changes and offboarding, and it is frequently stored somewhere with far weaker controls than the vault it unlocks.
    **Mitigation.** Revoke the initial root token as the **final step of initialisation**, non-negotiably, and generate a new one only via the recovery-key-gated procedure when genuinely needed — then revoke it immediately after. Alert on *any* root-token authentication, always, with no exceptions and no allowlist: in a correctly-operated system that alert fires approximately never, which makes it an extremely high-signal detector.

!!! gotcha "A short TTL that improves security creates a 15-minute outage fuse"
    **Symptom.** A security review mandates 15-minute credential TTLs. Three weeks later, a 20-minute vault outage — previously a non-event — becomes a full user-facing outage because every credential in the fleet expired during it.
    **Mechanism.** The TTL is simultaneously the exposure window for a leaked credential *and* the time you have to fix a secrets outage before user impact begins. Shortening it improves one and degrades the other, one-for-one. It also multiplies the renewal write rate: 5,000 leases at 1 h is 2.1 writes/s; at 15 min it is 8.3, plus four times the backend DDL.
    **Mitigation.** Set TTL by weighing both sides explicitly and per backend. One hour is a reasonable default for databases — it caps leak exposure at an hour and gives an hour to fix an outage, and it keeps `CREATE ROLE` load at 2/s. Fifteen minutes is fine for cloud STS, which is designed for it and where the renewal cost is negligible. **Present the TTL as a reliability decision as much as a security one**, because a security review that only sees one side will always choose the shorter number.

!!! gotcha "Two concurrent rotation jobs silently lose one another's write"
    **Symptom.** A secret is rotated, everything converges, and then consumers start failing against a value nobody recognises. The audit log shows two writes seconds apart.
    **Mechanism.** Two rotation jobs ran concurrently — a scheduled run and a manual one, or two replicas of the same job without leader election. Both read version 7, both created a new credential in the third-party system, both wrote version 8. The last write wins; the other job's credential exists in the backend, is untracked, and the job reports success.
    **Mitigation.** Use **check-and-set on every rotation write** (`"cas": 7`), so the losing writer receives a conflict and can abort and clean up its orphaned backend credential. Ensure rotation jobs are singletons via leader election ([F19](../fundamentals/f19-concurrency-control.md)). And make the job's cleanup path revoke the credential it created when its write is rejected, or you have traded a silent conflict for a silent orphan.

!!! gotcha "Auto-unseal works perfectly until someone changes a KMS key policy"
    **Symptom.** Nodes have been restarting cleanly for a year. After an unrelated cloud IAM cleanup, the next node to restart comes up sealed and stays sealed. The change that caused it happened days earlier and touched nothing that looked related.
    **Mechanism.** Auto-unseal depends on the node's instance role being permitted to call `Decrypt` on one specific KMS key. That permission is a line in a cloud IAM policy, maintained by a different team, with no indication that the entire company's secrets layer depends on it. IAM tightening exercises routinely remove it.
    **Mitigation.** Treat the KMS key policy as tier-0 configuration: tagged, change-protected, alerted on modification, and covered by a policy-as-code test that asserts the vault role retains `Decrypt`. Add a **synthetic check that actually performs a KMS decrypt every five minutes** from a node's identity, so the permission is verified continuously rather than only at the next restart — which might be months away. And always retain Shamir recovery keys, which are the only path back when the KMS access is gone.

!!! gotcha "Everyone reconnects the instant the vault comes back and knocks it over again"
    **Symptom.** After a 30-minute outage is fixed, the cluster becomes unavailable again within seconds, this time from load rather than from the original fault.
    **Mechanism.** Forty thousand agents have been retrying on a fixed interval for half an hour and are now perfectly synchronized. The moment the vault answers, they all log in at once — and every login is a consensus write. At 40,000 logins arriving over a few seconds against a ~3,000 writes/s ceiling, the freshly-recovered cluster is immediately saturated, times out, and the clients retry again.
    **Mitigation.** **Jittered exponential backoff in the agent**, capped at a few minutes with full jitter, so a synchronized herd spreads over minutes. **Server-side login admission control** returning `429` with `Retry-After` rather than timing out — a timeout gives the client no information and it retries immediately, while a `429` with a hint is actionable. And **batch tokens for short-lived workloads**, which removes that entire population from the write path. Serving 5,000 clients correctly while telling 35,000 to wait beats failing all 40,000.

---

## 13. Interview Angle

!!! interview "Open with the bootstrapping paradox, not with the feature list"
    The weak opening enumerates the product: "encrypted KV store, policies, dynamic secrets, audit log". Correct and forgettable.

    The strong opening names the structural problem: **"This system's core security property is that it cannot read its own data at rest. It boots sealed — running, healthy, answering 503 to everything — until something external supplies the master key. That 'something' is the entire design problem: a human quorum is secure and operationally unacceptable, a cloud KMS is operationally excellent but moves your trust root to the provider, and an HSM is both expensive and another thing that can be down. There is no option that is both self-sufficient and secure, and I want to be explicit about which trade I am making."**

    Then add the second structural fact: **"Its blast radius is a clock, not an explosion. When it goes down, running workloads are fine — until their credentials expire one TTL later, at which point failures spread as a wave. So the TTL is simultaneously my leak-exposure window and my time-to-fix budget, and I will set it per backend by weighing both."**

!!! interview "Four numbers that do the heavy lifting"
    - **$n{=}5, k{=}3$ at $p{=}0.8$ gives 94.2% ceremony success and ~15 minutes when it works**, which against a 4.32 min/month budget is 347% of the budget per event. This number *is* the argument for auto-unseal.
    - **4,800x blast-radius reduction** from 200 days of static-credential exposure to a 1-hour dynamic TTL.
    - **The TTL floor is set by the backend, not by the vault**: 5,000 leases at a 5-minute TTL is 25 `CREATE ROLE`/s against a production catalog, which is a database incident.
    - **A regional failover is 40,000 logins in 2 minutes = 333 consensus writes/s**, which is why login admission control exists.

!!! interview "The three things being tested"
    Whether you understand that **the seal is the point** and can discuss the unseal trade-off with real numbers. Whether you know that **the most common leak vector is environment variables and logs**, not a compromised vault. And whether you can articulate the **chicken-and-egg problem honestly** — shrinking the irreducible root of trust rather than claiming to have eliminated it. Volunteer all three.

??? question "Follow-up 1: Who unseals it after every node restarts at 3 a.m.?"
    **Answer.** Ideally nobody, and designing for "nobody" is the single highest-leverage decision in this system.

    **Why the human path is unacceptable as a primary mechanism.** With $n{=}5, k{=}3$ and each holder independently available with probability 0.8 at 3 a.m., the binomial gives a 94.2% chance the ceremony completes — meaning roughly one in seventeen restart events stalls on the first attempt. And when it works, it takes about fifteen minutes: paging, holders waking and authenticating, share entry. Against a 99.99% target's 4.32 minutes per month, **one ceremony burns 347% of the monthly budget.** That is not a tail risk to accept; it is a design defect.

    **The realistic timeline is worse than the arithmetic.** The recovery runbook typically depends on the thing that is down: the key-holder contact list is in a wiki behind SSO, and SSO's client secret is in the vault. I have seen a 15-minute ceremony take 33 minutes for exactly that reason. And when it finally unseals, 40,000 pods that have been retrying in lockstep all log in at once — each login being a consensus write — and the freshly-recovered cluster immediately saturates.

    **So: auto-unseal via cloud KMS as the primary path.** The master key is stored wrapped by a KMS key; on startup the node authenticates with its instance identity and calls `Decrypt`. Unseal completes in under a minute with no humans. What I give up is honest and worth stating: the trust root moves to the cloud provider, and the IAM policy on that one KMS key becomes the most security-critical configuration in the company. I would treat it as tier-0 — change-protected, policy-as-code tested, and verified by a synthetic decrypt every five minutes rather than discovered at the next restart.

    **Shamir stays, as recovery keys**, for the cases auto-unseal cannot cover: the KMS key is deleted, the cloud account is lost, or you are migrating providers. I would use $n{=}7, k{=}3$ rather than $5{/}3$ — 99.5% versus 94.2% at the same attacker threshold — keep an offline contact list that does not require the vault to reach, and drill quarterly with the automated path deliberately disabled.

    **And regardless of mechanism, alert on `vault_core_unsealed == 0` directly.** A sealed vault is a running, `Ready`, healthy-looking process. The first downstream symptom is a failed deploy, which gets triaged as a CI problem — burning the one hour of grace that cached credentials bought you.

??? question "Follow-up 2: Why are dynamic credentials so much better than static ones?"
    **Answer.** Three reasons, and the third is the one people forget.

    **Exposure window.** A static database password lives from the moment it leaks until somebody notices and rotates it — industry data puts median discovery for leaked credentials in the hundreds of days. A dynamic credential with a 1-hour TTL has an exposure window of at most an hour. That is roughly a 4,800x reduction in blast radius, and it is a *structural* property rather than a detection improvement: it holds even if nobody ever notices the leak.

    **Rotation stops being an event.** Rotating a static credential means finding every consumer — the 40 pods, the three config repos, the CI variable, the wiki page — coordinating a simultaneous change, and hoping the inventory was complete. That is why static credentials in practice are never rotated. Dynamic credentials rotate continuously by construction; there is no event to coordinate because nothing is long-lived.

    **Attribution.** With a shared static password, a suspicious query in the database log is attributable to "the application" — which is useless. With per-workload dynamic credentials, the database's own log contains `v-kube-orders-ro-8x2Kq-1742`, the lease table maps that to an entity, and the audit log maps the entity to a specific pod, service account, source IP and request at a specific millisecond. **That is the difference between an unresolvable incident and a fifteen-minute investigation**, and it is why I would adopt dynamic credentials even at TTLs long enough that the exposure-window argument was weak.

    **The costs, stated honestly.** The backend has to mint each credential, and `CREATE ROLE` on PostgreSQL is DDL that locks the shared catalog and replicates to every standby — 5,000 leases at a 5-minute TTL is 25 role creations and 25 drops per second, which is a database incident. So the TTL floor is set by the backend, not by the vault: an hour for relational databases, fifteen minutes for cloud STS which is built for it. The application must also be able to reload without restarting, which is the real adoption blocker. And lease-table durability becomes critical, because a lost lease row leaves a real user in a real database with nothing that will ever revoke it — which is why I would also set a native `VALID UNTIL` on the backend and run a reconciliation job.

??? question "Follow-up 3: How does a workload authenticate to the secrets store without a secret?"
    **Answer.** By proving what it *is* rather than presenting something it *knows*, using an unforgeable property of its runtime that a third party can verify.

    **The concrete flow.** The kubelet projects a service-account token into the pod's tmpfs — audience-bound to the vault, expiring in ten minutes, rotated automatically. The pod presents it. The vault does **not** trust the claims in it; it calls the cluster's `TokenReview` API, which verifies the signature against the cluster's own signing key and returns the authenticated namespace and service account. The vault maps that to an identity entity and resolves policies. The pod cannot forge the token because it cannot sign with the cluster key, cannot replay it at another service because it is audience-bound, and cannot benefit much from stealing one because it expires in minutes and is not in etcd or any disk snapshot.

    Equivalent mechanisms exist elsewhere: a cloud instance identity document signed by the provider, a SPIFFE SVID from a workload attestation system, a TPM quote, or an OIDC token from a CI platform's issuer. In every case the pattern is the same — **an attestation verified against an issuer's key, not a stored secret**.

    **And then the honest part, which is the actual question being asked.** This chain does not terminate in nothing. The SA token is signed by the cluster key, which is protected by the control plane, which was provisioned by automation, which authenticated with a cloud IAM role, which was assumed via an instance identity document, which is signed by the provider's hardware root. At the bottom there is always **trust in something outside the system**, and in an on-premises environment without a TPM it is literally an operator who typed something on first boot.

    **So the correct framing is not elimination, it is reduction.** The goal is to make the irreducible secret as small, short-lived and non-transferable as possible, and then to detect its use: one credential instead of five thousand, single-use rather than reusable, bound to a machine or network location rather than portable, valid for minutes rather than years, and **alerting on every single use** — because in a healthy system it is used approximately never, which makes that alert extraordinarily high-signal.

    Saying "we use workload identity so there are no secrets" is a weaker answer, because it is not true and an interviewer who has built one of these knows it.

??? question "Follow-up 4: How do you rotate a credential without downtime?"
    **Answer.** Create the new one before destroying the old one, converge, **verify**, and only then revoke. The ordering is the entire answer and getting it backwards is the mechanism behind essentially every rotation outage.

    **The overlap window has to cover four terms:** the agent's detection interval, the time to rebuild a connection pool, the longest in-flight request still holding an old connection, and — the unbounded one — stragglers. A pod in `CrashLoopBackOff` during the window, a batch job that runs nightly, a 1% canary nobody restarted. The first three sum to about eight minutes; the straggler term is why I choose twenty-four hours. A long overlap costs essentially nothing and eliminates an entire class of "we rotated and three obscure things broke" incidents.

    **Never revoke on a timer.** A job that sleeps for the overlap window and then revokes means any consumer that failed to converge loses access at a moment chosen by cron, typically hours after anyone stopped paying attention. Instead, verify convergence: query the backend for sessions still authenticating as the old identity, check the audit log for reads of the old version, confirm every known consumer has acknowledged the new one. If anything is outstanding, **alert with the specific list instead of revoking**. That turns a silent time bomb into an actionable page.

    **The application side is where these projects actually stall.** The app must swap credentials without restarting: an atomic pointer holding the credential, a watcher on the rendered file, and four details that matter. Verify the new credential against the backend *before* swapping, because swapping to a broken credential is an outage with no rollback. **Keep the old credential on any error** — a truncated file mid-write must never blank a working credential, and that is the most common bug in this code. Drain the connection pool gracefully rather than killing it. And export `credential_age_seconds`, because a pod that silently stopped reloading is invisible from the server side — the vault just sees it stop asking, which looks identical to it shutting down.

    **One more failure I would raise unprompted:** two concurrent rotation jobs. A scheduled run and a manual one both read version 7, both mint a new credential in the third-party system, both write version 8, and the loser's credential is orphaned while its job reports success. Use check-and-set on the write so the loser gets a conflict, make rotation jobs singletons, and have the conflict path revoke the credential it created.

??? question "Follow-up 5: The secrets store is down for an hour. Walk me through what happens."
    **Answer.** The interesting property is that **the blast radius is a clock, not an explosion**, and that shapes the whole response.

    **Minute zero to fifteen:** nothing user-visible. Running workloads have cached credentials and valid leases and keep serving normally. What breaks is anything that needs to *start*: new pods cannot authenticate, deploys fail, scale-outs fail, CI fails. This is dangerous precisely because it looks like a CI problem, and triaging it as one burns the grace period.

    **Fifteen to forty minutes:** batch jobs and scheduled work start failing. Visible internally, still not to users.

    **Forty to sixty minutes:** agents begin failing their renewals at two-thirds of TTL. Alerts should fire here at the latest, on client-side `credential_age_seconds` exceeding 80% of TTL.

    **Sixty minutes onward:** leases expire and the database starts rejecting connections. **Now it is user-visible, and it spreads as a wave** as each workload hits its own expiry rather than all at once.

    **That one-hour fuse is a design choice, not an accident.** The TTL is simultaneously my exposure window for a leaked credential and my time-to-fix for an outage. One hour gives a human an hour. Fifteen minutes gives fifteen. That is the argument I would make to a security review that wants shorter TTLs: they are also choosing the outage fuse length.

    **Degradation strategy.** Agents serve cached credentials until expiry, retry with jittered exponential backoff, and **never blank an existing credential on a failed refresh**. The cache is memory-only, because persisting plaintext credentials to disk defeats the system's purpose — with the narrow exception of tier-0 services, where an encrypted on-disk cache keyed by a TPM-sealed or KMS-derived key is defensible, since the cached credential is then unreadable without the same attestation that would have authenticated the pod anyway.

    **Applications must degrade rather than crash-loop.** A service that cannot get a database credential should serve read-only from cache or shed load with a clear error. Crash-looping generates a login storm that actively prevents recovery.

    **And the recovery is its own incident.** Forty thousand agents that have been retrying in lockstep for an hour all succeed within seconds of the vault returning — 40,000 consensus writes against a ~3,000/s ceiling. Without jittered client backoff, server-side login admission control returning `429` with `Retry-After`, and batch tokens for short-lived workloads, the fix is immediately followed by a self-inflicted second outage.

??? question "Follow-up 6: What is the most common way secrets actually leak in practice?"
    **Answer.** Not through the vault. Through **environment variables and logs**, by a wide margin, and any design that ignores this is optimising the wrong end of the problem.

    **Environment variables are the worst offender** and they are the default in almost every deployment tutorial. A secret in the environment appears in `/proc/<pid>/environ`, in every core dump, in `docker inspect` and `kubectl describe` output that gets pasted into tickets, in any library that logs its configuration at startup, and in every child process that inherits the environment. It is visible to anything that can read the process table. The fix is to use a file on tmpfs, read once at startup and on reload, or an in-memory fetch — never the environment.

    **Logs are the second.** An exception message containing a connection string, a debug statement printing a config struct, an HTTP client logging request headers including `Authorization`. The structural fix is to **wrap credential types so their string representation returns a redaction marker** — `String()`, `repr()`, `toString()` — which kills the entire class at the type level rather than relying on every developer remembering. Then scan logs at ingest for credential patterns and alert, because something will always get through.

    **Third: version control.** Committed `.env` files, Terraform state with secrets in plaintext, a Jupyter notebook with a token. Continuous repository scanning plus pre-commit hooks, and treat every hit as a rotation event rather than a deletion — removing the commit does not remove it from forks, clones, or anyone's local history.

    **Fourth: sharing between humans.** A credential pasted into Slack to unblock a colleague. **Response wrapping** solves this well: the sender wraps the secret into a single-use token with a short TTL and sends *that*. If the recipient's unwrap fails because the token was already consumed, you have *detected* an interception rather than merely hoping to have prevented one — which is a much stronger property than it first appears.

    **And the meta-point I would end on:** all of this is why short TTLs matter more than they seem. The leak will happen; you cannot engineer humans and libraries into perfection. What you control is how long the leaked thing is useful. Static credential: two hundred days. Dynamic credential with a 1-hour TTL: one hour. **The TTL is the mitigation for the leak vector you cannot close.**

??? question "Follow-up 7: Why is the audit log synchronous, and what does that cost you?"
    **Answer.** Because an audit log with gaps is close to worthless during the only event it exists for, and the cost is a deliberate, well-understood availability sacrifice.

    **Why synchronous.** If auditing were asynchronous, a request could be served and the record lost — to a buffer flush that never happened, a crash, a full disk. After a compromise, the first question is "what did the attacker access?", and with a gap in the log you cannot answer it. Worse, you cannot prove a *negative*: you cannot tell a regulator or a customer that a particular secret was not accessed, because the gap might contain exactly that. So the rule is that no request is served without a committed audit record, which puts the audit write directly on the critical path.

    **What it costs.** If every audit device fails, the system refuses all requests. A full disk or an unreachable log sink becomes a company-wide credential outage, with the vault unsealed, quorum healthy and storage fine. That is the design working as intended, which is especially frustrating at 4 a.m.

    **How I bound that cost without giving up the property.** Configure at least two independent devices — a local file and a remote socket — and require only one to succeed, so a single failure degrades redundancy rather than availability. Monitor audit write errors as a first-class SLI and alert on the audit filesystem at 70%, not 90%. Use group commit so the fsync is amortised across a batch — at batch size 32 the per-request cost drops to roughly 9 microseconds, which makes the whole argument about availability rather than latency. And **decide in advance, in writing, who may disable a failing audit device during an incident**, because that is a security decision and it should not be improvised by whoever is on call.

    **Two design properties that make the log usable rather than a liability.** Every sensitive value is HMACed with a dedicated audit key, so the log is not itself a secrets leak and an attacker who steals it gains nothing — while HMAC's determinism still lets you answer "was *this* credential accessed?" by hashing the candidate and searching. And **denials are logged identically to successes**: a burst of denials is the best available signal of a compromised identity probing for reachable paths, and a log that records only successes is blind to the most informative part of an attack.

### Strong answer versus weak answer

| Dimension | Weak | Strong |
|---|---|---|
| Framing | "Encrypted KV store with policies and dynamic secrets" | "It cannot read its own data at rest, so it boots sealed — and the unseal mechanism is the central trade-off between security and availability" |
| Unseal | "Shamir's Secret Sharing splits the master key" | Computes 94.2% ceremony success and 15 minutes = 347% of the monthly error budget; chooses auto-unseal with Shamir as recovery and names what that gives up |
| Dynamic credentials | "Short-lived credentials are more secure" | Quantifies 4,800x exposure reduction, adds the attribution argument, and names the backend DDL rate as the TTL floor |
| TTL | "As short as possible" | "The TTL is both the leak window and the outage fuse; 1 h for databases, 15 min for STS, set per backend with both sides stated" |
| Bootstrap identity | "Workload identity means no secrets" | Traces the trust chain to its root, admits it is irreducible, and argues for making it small, single-use, machine-bound and alerted on every use |
| Rotation | "Rotate the secret and restart" | Overlap window with the four terms computed, revoke on verified convergence not on a timer, and the atomic-swap reload pattern with keep-old-on-error |
| Leak vectors | Focuses on attacking the vault | Names environment variables, logs and repos as the real vectors, and fixes them at the type level with redacting credential types |
| Outage | "It's HA so it won't go down" | Walks the clock: 0 min deploys, 40 min renewals, 60 min user-visible wave — and designs the TTL to give humans an hour |
| Audit | "Everything is logged" | Synchronous and on the critical path, multiple devices with ≥1 required, HMACed values, denials logged, and the disk-full outage acknowledged |
| Recovery | "Then it comes back up" | Anticipates the login storm — 40k consensus writes — and prescribes jittered backoff, `429` admission control and batch tokens |
| Cost | Does not mention it | $30.6k/month of which 76% is sidecars; proposes node-level agents for ~$18k saving and refuses to cut audit retention |
| Alternatives | Builds it regardless | "Cloud-native secret managers are the correct default; self-managed wins for multi-cloud, dynamic credentials against internal backends, or a compliance boundary" |

---

## 14. Key Takeaways

1. **The system must be unable to read its own data at rest, so it boots sealed.** A sealed vault is a running, `Ready`, healthy-looking process that answers 503 to everything. That property makes a stolen disk worthless and makes a cold start an incident, and alerting directly on `unsealed == 0` is the difference between a one-hour warning and a user-visible outage.

2. **Human unseal is a distributed-availability problem with people as the nodes.** $n{=}5, k{=}3$ at 80% holder availability gives 94.2% ceremony success and about fifteen minutes when it works — 347% of a 99.99% monthly error budget per event. Auto-unseal via KMS is the answer; Shamir stays as the recovery path, drilled quarterly, with an offline contact list.

3. **The best secret is one that did not exist five minutes ago.** Dynamic, per-workload credentials with a 1-hour TTL cut the exposure window by roughly 4,800x versus a static credential, make rotation a non-event, and — the underrated benefit — give you per-pod attribution in the backend's own logs.

4. **The TTL floor is set by the backend, not by the secrets store.** `CREATE ROLE` is DDL that locks a shared catalog and replicates to every standby: 5,000 leases at a 5-minute TTL is 50 DDL statements per second against a production database. Choose TTL per backend and defend the number with the minting rate.

5. **The TTL is simultaneously the leak window and the outage fuse.** Shorten it and a stolen credential dies sooner, but so does your time to fix a secrets outage before users notice. Present it as a reliability decision as much as a security one, or a security review will always pick the shorter number.

6. **Identity-based authentication shrinks the chicken-and-egg problem; it never eliminates it.** Attestations replace stored secrets all the way down, and then bottom out in trust in a cloud provider's hardware, a TPM vendor, or an operator who typed something once. Make the irreducible root small, single-use, machine-bound, short-lived, and **alert on every use** — in a healthy system it fires approximately never.

7. **Rotation is create-new, converge, verify, revoke — in that order, and never on a timer.** The overlap window must cover stragglers, which is unbounded, so choose 24 hours. Revoke only after verifying zero old usage, and make the application swap credentials atomically with the old value preserved on any error.

8. **The audit log is synchronous because a log with gaps cannot prove a negative.** That means a failed audit device can refuse every request in the company — bound the cost with two independent devices requiring one success, group commit, and a pre-agreed decision about who may disable a failing device.

9. **The real leak vector is environment variables and logs, not a compromised vault.** Use tmpfs files instead of environment variables, wrap credential types so their string representation redacts, scan logs and repositories at ingest, and keep TTLs short because the leak will happen anyway.

10. **The blast radius is a clock, not an explosion.** Running workloads survive an outage until their leases expire, then failures spread as a wave. Cache in memory, back off with jitter, never blank a working credential on a failed refresh, and make applications degrade rather than crash-loop — because crash-looping generates the login storm that prevents recovery.

11. **Recovery is its own incident.** Forty thousand synchronized clients logging in at once is 333 consensus writes per second against a cold cluster. Jittered backoff, `429` with `Retry-After` rather than timeouts, and batch tokens for short-lived workloads are what turn a fix into a recovery instead of a second outage.
