# Topic 9 — Distributed Scheduler

[Back to Architecture Drills](../README.md)

**Status:** Complete

At 8:30 AM, a customer schedules a **$5,000 transfer for 9:00 AM**. The bank accepts the request and shows it as scheduled. At 9:00, a Scheduler node finds the payment and begins dispatching it. Then the node pauses, another node takes over, and Kafka later redelivers the same command. The customer intended one transfer. A design that turns those retries into two debits is incorrect, even if every individual service behaved as configured.

This chapter designs the distributed scheduling platform that prevents that outcome. The accepted path is **Scheduling API → sharded Schedule Database → due-job detector → atomic claim → transactional outbox → Kafka → workers → Transfer Service → ledger**. Leases and fencing tokens control competing Scheduler nodes. Stable execution IDs and authoritative idempotency protect the financial effect. Synchronous cross-region quorum protects committed scheduler state.

The central ownership rule is: **the Scheduler owns time and dispatch intent; the Transfer Service and ledger own the financial outcome.** Delivery is at least once, so correctness comes from durable identity and reconciliation rather than a claim of exactly-once transport. The recovery objectives are **RPO 0** for acknowledged scheduler state and **RTO 5 minutes** for regional recovery. Numerical scenarios are teaching exercises; production thresholds, capacity and sign-offs still require measured evidence.

## Contents

- [Problem & Requirements](#problem--requirements)
- [Final Architecture](#final-architecture)
- [End-to-End Flow](#end-to-end-flow)
- [Component Responsibilities](#component-responsibilities)
- [Data & Consistency](#data--consistency)
- [Scaling](#scaling)
- [Reliability & Failure Modes](#reliability--failure-modes)
- [Security](#security)
- [Observability](#observability)
- [Key Architecture Decisions](#key-architecture-decisions)
- [Constraint Mutations](#constraint-mutations)
- [Adversarial Architecture Review](#adversarial-architecture-review)
- [AI Intersection](#ai-intersection)
- [TPM Delivery](#tpm-delivery)
- [Program Risks](#program-risks)
- [TPM Constraint Mutations](#tpm-constraint-mutations)
- [Production Readiness](#production-readiness)
- [Rollout & Rollback](#rollout--rollback)
- [Concepts Learned](#concepts-learned)
- [Mental Models](#mental-models)

## Problem & Requirements

A customer schedules a **$5,000 transfer for 9:00 AM**. The system must durably remember that intent, dispatch it close to the requested time, survive crashes and regional failure, and still produce **one financial effect** even when claims, messages or calls repeat.

### Scheduling fundamentals

A **schedule** stores the customer's timing instruction. A recurring rule such as cron describes when future occurrences become due; it does not provide durability, ownership or financial correctness. Each occurrence gets a stable **execution ID**. Retries reuse that ID; rescheduling changes future timing.

- **Scheduled time:** what the customer requested.
- **Due time:** when the occurrence becomes eligible.
- **Execution time:** when financial processing begins/completes.
- **Misfire:** the occurrence was not dispatched within its expected window.

Cancellation is a durable state transition, not deletion. Time zone and daylight-saving policy are part of the schedule definition.

### Why distributed scheduling is hard

Multiple Scheduler nodes may discover the same due row. A node may pause after claiming it. Kafka may redeliver. The Transfer Service may commit and lose its response. Correctness therefore comes from **durable intent + atomic claim + fencing + stable identity + financial idempotency**, not from assuming exactly-once execution.

| Requirement | Decision and consequence |
|---|---|
| Durable schedules | Persist before acknowledging; process memory is never the authority. |
| One-time + recurring jobs | Materialize each occurrence with a stable execution ID. |
| Competing schedulers | Atomic claim, renewable lease and increasing fencing token. |
| Lost dispatch protection | Commit execution state + transactional outbox in one DB transaction. |
| Duplicate delivery | Kafka is at least once; workers preserve the same execution ID. |
| Duplicate debit prevention | Transfer Service / ledger atomically enforce one effect per execution ID. |
| Time zones / DST | Store local intent, IANA zone, recurrence rule and explicit DST policy. |
| Capacity protection | Bounded batches, priority lanes, backpressure and admission control. |
| Regional recovery | Synchronous cross-region quorum, fenced authority, **RPO 0 / RTO 5 min**. |
| Auditability | Trace schedule → occurrence → claim → dispatch → financial result. |

**Scope boundary.** The Scheduler owns timing and dispatch intent. The Transfer Service and ledger own authorization and money movement. The chapter does not redesign sanctions, account limits or double-entry accounting.

**Still to validate in production:** traffic mix, record size, shard count, Kafka partitions, safe transfer rate, cross-region latency and alert thresholds.

## Final Architecture

The baseline diagram comes first so the normal payment path is clear before leases, fencing and regional recovery are introduced.

### Architecture at a glance

![Distributed scheduler final architecture](assets/scheduler-flow.svg)

[Open the scalable scheduler-flow diagram](assets/scheduler-flow.svg)

The API stores schedule intent in the authoritative database. Due-job detectors find eligible executions through indexed queries. A successful claim creates durable dispatch intent in the transactional outbox. Kafka buffers that intent for workers, while the Transfer Service and ledger enforce the financial result. The normal path remains valid even though the transport can redeliver.

### Baseline and extensions

The baseline contains the Scheduling API, one logical authoritative Schedule Database, due-job detection, atomic claims, the outbox/Kafka handoff, workers and Transfer Service idempotency. Sharding and cross-region quorum extend that baseline to meet scale and recovery requirements. They do not change the stable schedule and execution identities.

## End-to-End Flow

The customer schedules transfer `S-501` for 9:00 AM. Its 9:00 occurrence becomes execution `E-501-0900`.

1. **Accept the schedule.** The channel sends the payment details, requested local time, time zone and an idempotency key. The API authenticates the caller, validates the rule and maps repeated identical requests to the same `scheduleId=S-501`.
2. **Persist durable intent.** The database stores the schedule, normalized timing data and next occurrence. The API responds only after the write satisfies the cross-region commit policy. A successful response therefore means the bank can recover the schedule after a process or regional failure.
3. **Materialize the occurrence.** The platform creates `executionId=E-501-0900` under a uniqueness constraint. A recurring schedule receives a different deterministic execution ID for every occurrence.
4. **Discover due work.** At 9:00 AM, shard-local detectors query an indexed due-time range in bounded batches. More than one detector may see the row; discovery does not grant execution authority.
5. **Claim atomically.** One contender changes the row from `Due` to `Claimed`, records its lease and receives the next fencing token. Other contenders fail the conditional update.
6. **Commit dispatch intent.** The same database transaction writes an outbox record for `E-501-0900`. This prevents a crash between “claimed” and “remembered for publication” from losing the payment.
7. **Publish and consume.** The outbox relay publishes to Kafka. A worker consumes the command and calls the Transfer Service with the original execution ID. Publication and consumption are at least once, so duplicates are expected.
8. **Apply one financial effect.** The Transfer Service stores `E-501-0900` atomically with the transfer and ledger entries. A repeated request with the same identity returns the stored result rather than creating another $5,000 debit.
9. **Record the outcome.** The Transfer Service publishes its outcome through its own outbox. The Scheduler records success, failure or an unknown result, while the ledger remains the authority for the money movement.
10. **Recover safely.** If the first Scheduler pauses, a new owner may reclaim the expired lease with a higher token. If the worker times out after the transfer committed, reconciliation looks up `E-501-0900` before any retry.

**Recurring schedules.** After materializing the current occurrence, the Scheduler calculates the next occurrence from the stored local-time rule and time-zone policy. It does not derive “9:00 AM tomorrow” by blindly adding 24 hours to a UTC timestamp.

## Component Responsibilities

| Component | Core responsibility / boundary |
|---|---|
| Scheduling API | Validate create/update/cancel/status, authenticate/authorize, and make create requests idempotent. Does not execute transfers. |
| Schedule DB | Own schedules, occurrences, claims, leases, fencing tokens, outbox and audit state. Does not own balances. |
| Due-job detector | Find eligible work in indexed bounded batches. Discovery does not grant authority. |
| Claim coordinator | Atomically claim work, assign lease and increment fencing token. |
| Outbox relay | Publish committed dispatch intent to Kafka and expose publication lag. |
| Kafka | Buffer and redeliver commands. Broker delivery does not provide ledger idempotency. |
| Workers | Preserve execution identity, obey capacity controls and call Transfer Service. |
| Transfer Service + ledger | Reauthorize and atomically enforce one financial effect per execution ID. |
| Retry / DLQ / reconciliation | Resolve recoverable failures, unknown outcomes and governed replay. |
| Shard / regional authority | Maintain one fenced owner per shard and stop minority authority. |
| SRE / audit | Measure lateness, backlog, correctness, capacity and recovery evidence. |

## Data & Consistency

The **Schedule DB** is authoritative for scheduling/dispatch state. The **Transfer Service and ledger** are authoritative for the financial result.

A schedule, an occurrence and an execution are different:
- `scheduleId` = customer's instruction.
- `occurrenceKey` = one calendar firing.
- `executionId` = stable identity used for dispatch and financial idempotency.

### Scheduling policy, time and cancellation

Persist the execution instant in UTC while preserving original local time, IANA zone, recurrence rule and DST policy. Daily local-time recurrence is recalculated from calendar intent; it is not “previous UTC + 24h.”

| Case | Policy |
|---|---|
| Spring-forward nonexistent time | Reject, skip or move according to an explicit product rule. |
| Fall-back ambiguous time | Persist which occurrence is intended. |
| Recurring local time | Recalculate from local calendar + zone rules. |
| Cancellation before claim | Suppress the occurrence. |
| Cancellation after committed dispatch | Return too-late/pending and follow Transfer Service semantics. |
| Long outage / misfire | Execute, skip or coalesce according to policy; never blindly replay every missed debit. |

Use DB time for due checks and leases where possible. Clock synchronization helps operations, but safety comes from persisted state, atomic updates and fencing.

### Core records

| Record | Key invariant |
|---|---|
| Schedule | Durable user intent; scoped idempotency key prevents duplicate schedule creation. |
| Execution | One logical occurrence; unique `(scheduleId, occurrenceKey)`. |
| Claim fields | `ownerId`, `leaseUntil`, monotonically increasing `fencingToken`. |
| Outbox | Dispatch intent committed with execution state. |
| Transfer idempotency | Unique execution ID + request hash stored with financial mutation. |
| Outcome / audit | Durable financial result and append-only transition history. |

### Atomic claim and fencing

A lease says who may work **for a period**. A fencing token protects against an old owner that resumes after its lease expired. If Scheduler A holds token 41, pauses, and B reclaims with token 42, downstream state must reject A's older token.

The claim itself is a conditional atomic state change such as `Due → Claimed`; read-then-write claiming is race-prone.

### Transactional outbox and Kafka

Write the execution transition and outbox row in one DB transaction, then publish asynchronously. Kafka remains **at least once**; duplicate publish or redelivery is acceptable because the execution ID stays stable.

### Financial idempotency and reconciliation

A timeout means **unknown outcome**, not “payment failed.” Retries reuse the same execution ID. The Transfer Service atomically stores that ID with the transfer/ledger mutation and returns the prior result on duplicates. If the outcome is uncertain, reconcile against the Transfer Service / ledger before retrying or declaring failure.

## Scaling

Scale the Scheduler by separating **due discovery**, **dispatch**, and **financial execution**. The bottleneck may be the due index, claim rate, Kafka, worker fleet or Transfer Service; adding Scheduler nodes does not automatically increase end-to-end capacity.

| Area | Scaling rule |
|---|---|
| Due detection | Indexed `dueAt` ranges, bounded batches and shard-local scans. |
| Sharding | Route by stable key; each shard retains its own due index and authority. |
| Workers | Scale only within Kafka partition and Transfer Service capacity. |
| Backlog | Measure oldest due age and catch-up rate, not queue size alone. |
| Priority | Reserve lanes/capacity for critical payments rather than letting bulk work starve them. |
| Hot shard | Rebalance/split ownership when measured skew justifies migration. |

Capacity exercises in the drill showed that predictable 9:00 AM bursts should be **pre-scaled**, and recovery throughput must exceed incoming work or lateness never converges.

## Reliability & Failure Modes

| Failure | Safe response |
|---|---|
| Scheduler dies before claim | Another node discovers and claims the still-due row. |
| Scheduler pauses after claim | Lease expires; new owner receives a higher fencing token. |
| DB commit succeeds, publish not completed | Outbox relay resumes from durable intent. |
| Kafka redelivers | Worker reuses the same execution ID; Transfer/ledger returns prior result. |
| Transfer call times out | Treat as unknown; reconcile before retrying. |
| Transfer dependency is down | Backoff + jitter, bounded retries, priority protection and durable backlog. |
| Poison execution | Quarantine to owned DLQ; preserve identity and audit history for governed replay. |
| Region fails | Promote only valid fenced authority under the cross-region quorum policy. |

### Multi-region durability and recovery

The drill requires **RPO 0** for acknowledged scheduler state and **RTO 5 minutes**. That implies synchronous cross-region durability for the committed scheduling state, one authoritative region/shard owner at a time, fencing of the old authority, and tested failback.

A minority partition must stop. Regional recovery is not complete when routing changes; it is complete when due detection, dispatch, idempotency, reconciliation and capacity are working under the new authority.

## Security

Security follows the same ownership boundaries as the architecture:

- authenticate users/services and authorize schedule/account operations;
- use TLS/mTLS and least-privilege credentials between services;
- restrict who can create, modify, cancel or replay schedules;
- protect Kafka topics, DB shards, DLQs and admin/replay paths separately;
- encrypt sensitive payloads and redact logs;
- audit schedule changes, claims, retries, overrides and operator actions.

Internal network location is not authorization. Replay and operational tools are especially sensitive because they can trigger real financial work.

## Observability

| Signal | Why it matters |
|---|---|
| Due-to-dispatch lateness p95/p99 | Customer timing outcome. |
| Oldest due age / backlog / catch-up rate | Whether the system is converging. |
| Claim conflicts / lease expiry / fencing rejects | Ownership instability. |
| Outbox age / Kafka lag / worker errors | Dispatch-path health. |
| Transfer latency / unknown outcomes / dedupe hits | Financial dependency and retry behavior. |
| DLQ age / replay volume | Unresolved failures and operational debt. |
| Cross-region replication / failover time | RPO/RTO exposure. |

Correlate `scheduleId` and `executionId` end to end. Healthy Scheduler CPU does not prove payments are on time or financially reconciled.

## Key Architecture Decisions

The decisions below follow from the failure examples rather than from a preference for distributed components.

| Decision / disposition | Reasoning and trade-off |
|---|---|
| **Accepted:** durable database is schedule authority | Process timers and Kafka delays cannot be the only copy of a bank-acknowledged instruction. Indexed polling adds database load but makes work recoverable and queryable. |
| **Accepted:** atomic claim, lease and fencing token | The claim chooses one current owner; the lease permits recovery; fencing rejects a former owner that resumes. Token enforcement adds state to downstream boundaries. |
| **Accepted:** transactional outbox before Kafka | It closes the local database-commit/publish gap. The cost is an outbox relay, retained rows and duplicate publication handling. |
| **Accepted:** stable execution ID at Scheduler, Transfer Service and ledger | It converts redelivery, retry and failover into one financial effect. Idempotency history and payload hashes must outlive every supported replay horizon. |
| **Accepted:** one occurrence row per recurring firing | Every run receives independent identity, status and audit evidence. Storage and lifecycle management increase. |
| **Accepted:** sharded database with shard-local due indexes | It distributes storage, writes and polling. Routing, rebalancing, hot shards and per-shard recovery become explicit operational work. |
| **Accepted:** global backpressure and reserved priority capacity | It protects Transfer Service and critical payments during overload. Standard or bulk work can become late and must expose that impact. |
| **Accepted:** synchronous cross-region quorum per shard | It supports RPO 0 and safe authority transfer. Cross-region latency rises and the system must stop when it cannot form a safe majority. |
| **Accepted:** RTO 5 minutes is a release gate | Recovery becomes a measurable capability rather than a diagram. Pre-provisioning, automation and repeated rehearsals are required. |
| **Accepted:** AI remains asynchronous and advisory | Forecasting and anomaly detection may help operations without becoming a financial correctness dependency. |
| **Rejected:** one cron leader with local memory | A process crash loses intent or ownership evidence and creates one scale/failure boundary. |
| **Rejected:** `SELECT` followed later by `UPDATE` as a claim | Two workers can read the same due row before either update occurs. |
| **Rejected:** lease expiry proves the old worker stopped | A paused worker can resume after expiry; only enforceable fencing closes the stale-owner race. |
| **Rejected:** Kafka exactly-once means exactly-once debit | Broker guarantees do not atomically include the Transfer Service and ledger. |
| **Rejected:** asynchronous regional replication can provide RPO 0 | An acknowledged write can be absent from the surviving region. |
| **Rejected:** independent active-active regional writers | A partition can create competing owners and duplicate dispatch. |
| **Rejected:** unlimited retry or unrestricted backlog release | Recovery traffic can amplify the original failure and overwhelm the Transfer Service. |

The final Architecture Review Board defense prioritized **financial correctness**, **exclusive distributed ownership with durable recovery**, and **controlled execution under constrained capacity**.

## Constraint Mutations

| Changed constraint | Accepted response / correction |
|---|---|
| Transfer capacity falls by 50% | Reduce global dispatch, preserve critical reserve, defer standard/bulk, stop retry amplification and publish drain forecast. |
| 10× schedules become due together | Fair bounded claims, isolated priority lanes and paced release; do not load all rows or flood Kafka/Transfer. |
| Kafka unavailable for 30 minutes | Outbox accumulates durable intent; alert on age/storage, then drain under Transfer capacity after recovery. |
| Worker pauses beyond its lease | New owner receives higher token; stale worker is rejected and repeated execution ID cannot debit again. |
| Region A loses network to Region B/witness | Only the side with quorum remains authoritative; the minority stops writes and dispatch. |
| Cancellation races with dispatch | Conditional state decides the winner; return explicit result and reconcile if external execution may have begun. |
| DST skips 02:30 | Apply the stored nonexistent-time policy and audit the resolution; never infer silently. |
| Three days of missed recurring payments | Apply configured skip/coalesce/bounded catch-up policy and capacity control; never blindly issue all debits. |
| AI service unavailable | Core scheduling and financial flow continue unchanged; defer forecasts/recommendations. |

## Adversarial Architecture Review

| Challenge | Outcome retained |
|---|---|
| Replace the database with Kafka delayed messages? | Reject as sole authority: cancellation, recurrence, queries, long horizons and replay need durable schedule state. Kafka may assist dispatch. |
| Use Redis TTL for every job? | Reject as sole durable source for financial intent; optional near-term acceleration must rebuild from the database. |
| One elected scheduler means claims are unnecessary? | Reject: leadership can overlap during pause/partition and workers still crash. Atomic state and fencing remain required. |
| A long lease prevents duplicates? | Reject: it slows recovery and still cannot stop a stale process. |
| Transfer Service is reliable, so Scheduler dedup is enough? | Reject: the remote commit/response gap makes authoritative idempotency mandatory. |
| Async replication plus fast failover can claim RPO 0? | Reject: fast recovery does not restore an unreplicated acknowledged write. |
| Keep both regions active for availability? | Only with one quorum-controlled owner per shard; two independent writers violate safety. |
| Critical priority may bypass financial controls? | Reject: priority allocates capacity, never authorization, limits or ledger correctness. |
| Retry until success? | Reject for permanent and unknown outcomes; bound retry and reconcile. |
| DR can launch after production? | Reject. Failed or unproven five-minute regional recovery is a No-Go. |

## AI Intersection

**Rejected:** synchronous AI deciding whether a financial job executes, assigning authoritative priority, resolving ambiguous time, or approving a retry. Model latency, availability and nondeterminism do not belong in the correctness path.

Useful asynchronous work includes arrival/capacity forecasting, hot-shard prediction, anomaly detection across lateness and retries, operator summaries, and recommendations for resharding or capacity allocation. Inputs are minimized and redacted. Every recommendation stays within deterministic policy and requires measurable evaluation: offline replay → human review → shadow mode → limited canary → monitored expansion. Record model/version and recommendation provenance. AI failure leaves the core scheduler unchanged.

## TPM Delivery

| Workstream | Delivery evidence |
|---|---|
| Scheduler/API | Schedule lifecycle, recurrence, cancellation, claim and dispatch semantics. |
| Data/platform | Schema, indexes, leases/fencing, outbox, sharding and cross-region authority. |
| Kafka/workers | Stable IDs, partitioning, bounded retries, DLQ/replay and capacity controls. |
| Transfer/ledger | Atomic idempotency and reconciliation contract. |
| Security | Identity, authorization, encryption, least privilege and audit. |
| SRE/operations | Lateness/backlog SLOs, dashboards, failover runbooks and support model. |
| QA/performance | Functional, concurrency, burst, failover, replay and recovery evidence. |

### Sequence and critical path

**Scheduling contract → durable data model → atomic claim/outbox → worker/Transfer integration → peak/failure testing → regional recovery → operational/security sign-off.**

The data model and Transfer Service idempotency contract are the main correctness dependencies. SRE, security and test design begin in parallel.

### Milestones and evidence

1. **Integration-ready scheduler:** durable create/update/cancel, due index, claim/fencing and outbox.
2. **End-to-end execution:** Scheduler → Kafka → worker → Transfer Service using stable execution IDs.
3. **Scale/resilience:** burst, backlog, node pause, lease expiry, duplicate delivery and dependency outage.
4. **Regional recovery:** prove RPO 0 / RTO 5 min with fencing and failback.
5. **Go/No-Go:** operational, security, rollback and reconciliation evidence with accountable sign-off.

## Program Risks

| Risk | Mitigation / decision boundary |
|---|---|
| Duplicate financial effect | Stable execution ID + atomic Transfer/ledger idempotency; unresolved proof is a launch blocker. |
| Missed/late jobs | Indexed due scans, capacity headroom, backlog/lateness alerts and bounded catch-up. |
| Stale scheduler owner | Lease + fencing token; downstream rejects older authority. |
| Outbox/Kafka backlog | Monitor oldest age and catch-up rate; throttle within downstream capacity. |
| Time-zone/DST defect | Versioned rules, explicit ambiguous/nonexistent-time policy and representative calendar tests. |
| Regional authority conflict | Quorum + fencing + tested failover/failback; minority must stop. |
| Transfer capacity shortfall | Priority lanes, admission control and protected capacity. |
| Operational readiness gap | Runbooks, ownership and recovery rehearsal are release gates. |

## TPM Constraint Mutations

| Constraint | Delivery response |
|---|---|
| Development complete; Kafka and idempotency pass; DR exceeds five minutes; launch date fixed | **No-Go.** Escalate to sponsor/steering committee, keep remediation on the critical path, retest and obtain formal sign-off. |
| Cross-region write latency threatens the API SLO | Quantify and optimize quorum placement/batching; do not silently relax RPO 0. Escalate the business trade-off if objectives conflict. |
| Peak load passes but backlog drain starves new critical work | Fail the capacity gate; require priority isolation and recovery-envelope retest. |
| Non-critical documentation remains after all safety paths pass | Conditional Go only with named owner/date, explicit risk acceptance and no effect on operational response. |

Business pressure does not erase an agreed resilience requirement. Changing RPO/RTO requires formal risk governance; it is not an informal release workaround.

## Production Readiness

Required gates, **not claims of completed production tests**.

| Gate | Evidence before Go |
|---|---|
| Correctness | Stable schedule/execution IDs, atomic claims, fencing and Transfer/ledger idempotency. |
| Time semantics | One-time, recurring, DST, cancellation and misfire policies validated. |
| Integration | DB → outbox → Kafka → worker → Transfer Service → outcome works end to end. |
| Performance | Due-time bursts, bounded batches, backlog catch-up and downstream capacity tested. |
| Failure/recovery | Node pause, lease expiry, duplicate delivery, timeout/unknown outcome, DLQ/replay and dependency outage. |
| Regional DR | Fenced failover/failback demonstrates **RPO 0 / RTO 5 min**. |
| Security / operations | Authorization, least privilege, telemetry, alerts, runbooks, on-call and escalation. |
| Final Go/No-Go | Residual risks and rollback/recovery evidence have accountable owners. |

## Rollout & Rollback

Use **internal validation → shadow/read-only observation where safe → limited cohort → progressive expansion**. Never duplicate real financial side effects merely to compare systems.

Freeze expansion if lateness, duplicate-effect signals, queue/backlog growth, fencing errors, reconciliation gaps or regional recovery exceed agreed thresholds.

Rolling back application code does not recall an already published command or reverse a committed transfer. Recovery may require stopping dispatch, preserving durable state, reconciling outcomes and resuming from known checkpoints with the same execution identities.

## Concepts Learned

| Concept | Working meaning / correction |
|---|---|
| Schedule / occurrence / execution | User recurrence intent / one calendar fire / one stable unit of work and identity. |
| Due detection / claim | Find candidates / atomically grant recoverable authority. |
| Lease / fencing token | Time-bounded liveness / monotonic protection against stale owners. |
| Outbox / Kafka | Atomic local dispatch intent / durable at-least-once transport. |
| Idempotency / deduplication | Same identity produces one authoritative effect / repeated delivery is recognized. |
| Failed / unknown | Confirmed negative outcome / result not yet known and requiring reconciliation. |
| Misfire / retry | Missed scheduling window / another attempt after a classified failure. |
| Priority / backpressure | Allocate scarce capacity / constrain offered load. |
| Shard / quorum | Scale and failure unit / votes required for safe committed authority. |
| RPO / RTO | Acceptable committed-data loss / time to restore useful service. Here: 0 and 5 minutes. |

## Mental Models

- **The Scheduler owns time; the ledger owns money.**
- Durable intent comes before acknowledgement.
- Detect broadly; claim atomically; execute with stable identity.
- A lease enables recovery. A fencing token stops a stale owner.
- The outbox closes the local commit/publish gap; downstream idempotency closes the remote effect gap.
- Every recurring occurrence gets its own deterministic execution ID.
- A timeout is an unknown outcome until reconciled.
- Backpressure protects capacity; priority decides who uses it; neither creates capacity.
- RPO 0 requires synchronous commit to a quorum, not hopeful replication.
- Failover transfers authority per shard; failback is another controlled migration.
- AI advises outside the financial path; deterministic controls remain accountable.

---

**Design basis:** 44 completed questions across 11 stages, accepted/refined/challenged/rejected proposals, constraint mutations, AI boundaries and Staff TPM delivery. Previous: [Topic 8 — Notification System Architecture](../08-notification-system/README.md). Next: [Topic 10 — CDN + Edge Architecture](../10-cdn-edge-architecture/README.md).
