# Topic 5 — Service Mesh + Service Discovery

[Back to Arch Drills](../README.md)

**Status:** Complete

A banking platform with 200 services requires dynamic discovery, internal load balancing, consistent service-to-service policy, failure containment, and controlled platform delivery. The architecture separates **registry and discovery → locality-aware instance selection → distributed mesh data plane → mesh control plane**. Applications retain domain logic, proxies enforce networking policy locally, and a control-plane failure does not immediately stop healthy traffic.

The mesh is justified by standardized workload identity, mutual TLS (mTLS), authorization, resilience, traffic management, and telemetry. Service count alone is not sufficient justification. If these cross-cutting requirements disappear, service discovery and internal load balancing provide the preferred simpler design. Numerical values in this document are illustrative planning scenarios rather than measured production thresholds.

## Contents

- [Problem & Requirements](#problem--requirements)
- [Final Architecture](#final-architecture)
- [End-to-End Flow](#end-to-end-flow)
- [Component Responsibilities](#component-responsibilities)
- [Key Architecture Decisions](#key-architecture-decisions)
- [Service Registry & Discovery](#service-registry--discovery)
- [Discovery Models & Internal Load Balancing](#discovery-models--internal-load-balancing)
- [Mesh Control & Data Planes](#mesh-control--data-planes)
- [Reliability & Failure Modes](#reliability--failure-modes)
- [Security](#security)
- [Configuration Safety](#configuration-safety)
- [Traffic Management](#traffic-management)
- [Observability](#observability)
- [Multi-AZ & Multi-Region](#multi-az--multi-region)
- [Constraint Mutations](#constraint-mutations)
- [Adversarial Architecture Review](#adversarial-architecture-review)
- [AI Intersection](#ai-intersection)
- [TPM Delivery](#tpm-delivery)
- [Program Risks](#program-risks)
- [TPM Constraint Mutations](#tpm-constraint-mutations)
- [Production Readiness](#production-readiness)
- [Rollout & Rollback](#rollout--rollback)
- [Architecture Reference](#architecture-reference)
- [Operating Principles](#operating-principles)

## Problem & Requirements

Payment must call Fraud and Ledger without hard-coded addresses. Instances scale, restart, move, and deploy continuously across availability zones (AZs) and regions.

| Requirement | Established in the drill | Consequence |
|---|---|---|
| Dynamic endpoints | Instances are ephemeral and autoscaled | Call logical services; register endpoints and metadata dynamically. |
| Safe eligibility | A process may be alive but not ready | Discovery returns ready instances; shutdown removes readiness before draining. |
| Internal balancing | Choose among usable instances | Consider locality, load, latency, health, and outliers. |
| Consistent east-west policy | Heterogeneous services need common security, resilience, routing, and telemetry | A mesh is justified; applications keep domain logic. |
| Availability | Registry and control plane must not sit synchronously in every request | Data planes retain recent discovery state and last-known-good (LKG) policy. |
| Zero trust | Internal network location does not prove identity or permission | Unique workload identity, mTLS, explicit authorization, least privilege. |
| Failure containment | Slow dependencies, retries, bad endpoints, certificates, and config must not cascade | Deadlines, one retry owner, budgets, breakers, outlier ejection, staged config. |
| Regional independence | A healthy region must serve without another region's control plane | Regional data-plane autonomy and local dependencies. |
| Operability | Added proxies must remain diagnosable | Correlated metrics, logs, traces, maps, alerts, and runbooks. |

**Scope boundary.** Exact vendors, SLOs, certificate lifetimes, retry budgets, policy thresholds, regional data design, costs, and named owners were not finalized. The 200-service platform, 2,000-service mutation, 10→500 autoscaling event, and latency values are architecture exercises.

## Final Architecture

![Service-to-service runtime architecture](assets/runtime-architecture.svg)

[Open the scalable runtime architecture](assets/runtime-architecture.svg)

The caller-side proxy selects a ready endpoint, establishes mTLS, enforces authorization and resilience policy, and emits telemetry. The destination proxy authenticates and authorizes the workload before forwarding. Registry and control plane update local state outside the synchronous request path.

![Regional control and data planes](assets/regional-autonomy.svg)

[Open the scalable regional autonomy diagram](assets/regional-autonomy.svg)

Each regional data plane continues with safe LKG configuration if the control plane is unavailable. Regional serving must not require a cross-region management call.

## End-to-End Flow

1. **Register and qualify.** Fraud instances register identity, endpoint, locality, version, and status. Liveness proves the process exists; readiness makes it eligible.
2. **Distribute discovery state.** Payment's local data plane learns usable Fraud endpoints without querying the registry per request.
3. **Select an endpoint.** Prefer a performant same-AZ instance, then another AZ in-region. Cross-region routing requires an explicit recovery decision.
4. **Secure the call.** Payment proves its workload identity through mTLS; Fraud authenticates it and applies least-privilege authorization.
5. **Apply resilience.** A deadline bounds waiting. The designated layer alone retries safe failures using bounded attempts, backoff, jitter, and a retry budget. Breakers and outlier detection protect Fraud.
6. **Serve and observe.** Fraud's proxy forwards to Fraud; metrics, logs, trace spans, policy outcomes, and certificate signals are emitted.
7. **Manage asynchronously.** The control plane versions and distributes approved routing, security, resilience, traffic, and telemetry policy.

## Component Responsibilities

| Component | Responsibility | Boundary / failure impact |
|---|---|---|
| Instance / orchestrator | Register and deregister endpoint/metadata; expose liveness/readiness | Discovery itself does not normally register instances. |
| Registry | Store service-instance locations, status, and metadata | Outage delays topology changes; health, leases, and TTL handle stale entries. |
| Discovery | Produce the usable endpoint set | Answers **where**, not which endpoint handles this request. |
| Internal LB | Select by readiness, load, latency, locality, and outlier state | Least connections may mislead with multiplexing; concurrency can be better. |
| Application | Own business rules, operation semantics, idempotency, and domain authorization | The mesh must not encode payment thresholds or fraud rules. |
| Data-plane proxy | Enforce routing, mTLS, workload authz, deadlines, delegated retries, breakers, splits, telemetry | A local proxy failure can isolate its workload. |
| Mesh control plane | Define, validate, version, distribute, and reconcile desired policy | Failure freezes management; serving continues from safe local state. |
| Identity/certificate system | Issue identities/certificates; renew, revoke, publish trust | Expired credentials cannot be made valid by cached mesh config. |
| Telemetry platform | Correlate metrics, logs, traces, alerts, maps, and versions | Must distinguish app, proxy, discovery, identity, network, and downstream faults. |

## Key Architecture Decisions

| Decision / disposition | Why | Alternative and trade-off |
|---|---|---|
| **Accepted:** logical identity plus dynamic discovery | Hard-coded IPs break during scaling, restart, and deployment | Static endpoints suit only genuinely static topology. |
| **Accepted:** server-side discovery/LB baseline | Consistent behavior across Java, Go, Node.js, and Python | Client-side removes an intermediary but duplicates libraries and governance. |
| **Accepted:** mesh for the full requirements | Standardizes identity, mTLS/authz, resilience, traffic policy, telemetry | Service count alone is not justification. |
| **Rejected:** one central mesh hop | The data plane is distributed beside workloads | Control-plane and local-proxy failures have different effects. |
| **Accepted:** business logic stays in applications | Domain policy needs application ownership, testing, and auditability | Network config should not own financial decisions. |
| **Accepted:** one retry owner | Independent layers multiply load and obscure behavior | Four layers with three attempts each can create 81 Fraud attempts. |
| **Rejected:** timeout below normal tail latency | False timeouts add retries and cause degradation | Use the end-to-end budget, latency distribution, variance, and headroom. |
| **Accepted:** locality while healthy and performant | Reduces latency and network cost | Slow local endpoints can be reduced or ejected. |
| **Accepted:** unique workload identities | Shared certificates erase attribution and expand compromise blast radius | mTLS authenticates; authorization is separate. |
| **Accepted:** regional LKG plus staged config | Preserves serving and contains policy blast radius | Security-critical stale policy may require fail-closed handling. |

## Service Registry & Discovery

![Service instance lifecycle](assets/instance-lifecycle.svg)

[Open the scalable instance lifecycle](assets/instance-lifecycle.svg)

A record includes workload identity, instance ID, address/port, AZ/region, readiness, and version metadata. A crashed instance cannot deregister itself: health checks, heartbeats/leases, and TTL expiry remove it from the eligible set. A slow instance returning HTTP 200 may remain registered while the balancing layer reduces or ejects it.

If the registry is unavailable, existing calls continue from recent discovery state. New instances may start but cannot become reliably discoverable, leaving new compute unused. Consumers reconcile after recovery; state-age and stale-endpoint behavior need explicit limits.

## Discovery Models & Internal Load Balancing

| Model | Benefit | Cost / use |
|---|---|---|
| Client-side | Direct call after client selection | Client coupling and inconsistent libraries across languages. |
| Server-side | Simple clients and consistent selection | Extra highly available infrastructure; selected for discovery/LB-only needs. |
| Mesh proxy | Combines discovery/LB with mesh controls locally | Runtime and operational overhead; selected for the complete requirements. |

Discovery answers **which usable endpoints exist**. Internal LB answers **which endpoint gets this request**. Round robin is simple; least requests suits unequal concurrency. Locality order is normally **same AZ → same region/different AZ → another region when explicitly justified**. Runtime health overrides locality: a local endpoint at 900 ms should lose traffic to an in-region endpoint at 100 ms.

## Mesh Control & Data Planes

| Capability | Control plane | Data plane |
|---|---|---|
| Identity / mTLS | Distribute identity, trust, and certificate config | Present/verify identity and establish encrypted connections. |
| Authorization | Define and distribute policy | Enforce on live requests. |
| Deadline / retry / breaker | Define approved policy | Execute runtime behavior. |
| Traffic split | Define version and weights | Route the actual percentage. |
| Telemetry | Configure collection/propagation | Generate runtime signals. |

The control plane answers **what should happen**; the data plane executes it. During an outage, current traffic continues using local policy and discovery state. New changes pause, drift is observed, and proxies reconcile to the approved version after recovery. Cached config does not extend certificate validity or make stale authorization safe indefinitely.

## Reliability & Failure Modes

| Failure | Selected behavior | Protection / limit |
|---|---|---|
| Slow Fraud | Stop at caller deadline; retry only when safe | Total deadline, bounded connections, one owner, backoff/jitter, retry budget. |
| Persistent failure | Circuit opens and fails fast; half-open probes recovery | Circuit **closed** means normal flow; business fail-open/closed is separate. |
| One slow-ready instance | Reduce or eject the outlier | Do not fail the service; investigate CPU, memory, pools, runtime, downstreams. |
| Registry outage | Existing traffic uses recent endpoints | New topology/capacity may be unavailable; monitor state age. |
| Stale endpoint | Health/lease/TTL and passive failures remove or avoid it | Bound connect timeouts and stale-state duration. |
| Control-plane outage | Serve with LKG; freeze changes | Critical stale authorization can require fail closed. |
| Local proxy failure | Isolate and repair the workload | Distributed data plane avoids a central runtime failure. |
| Partial certificate expiry | Isolate affected workloads; valid fleet continues if capacity permits | Never weaken mTLS to admit expired identities. |
| AZ loss | Fail to pre-sized capacity with admission controls | N+1/headroom and priority shedding; autoscaling is not instant capacity. |
| Retry amplification | Remove retries from competing layers | Four layers × three attempts can produce 81 downstream attempts. |

For a high-value payment whose required Fraud decision is unavailable, **fail closed**: reject, hold, or route to manual review. This business outcome differs from a circuit breaker's OPEN state.

## Security

![Workload identity and authorization flow](assets/workload-security.svg)

[Open the scalable workload security diagram](assets/workload-security.svg)

- **Workload identity:** which service is calling, independent of IP.
- **mTLS:** encryption plus mutual authentication; it does not grant permission.
- **Authorization:** whether the authenticated workload may perform this operation. Domain checks may remain in Fraud or Ledger.
- **Least privilege:** Payment gets only required operations; Notification cannot post a ledger entry because it has a valid certificate.
- **Certificate lifecycle:** issue early, install on a small cohort, validate, rotate progressively with overlap, confirm fleet health, then retire the old certificate.

Monitor issuance, installation, expiry margin, rotation failures, trust versions, and workloads presenting old certificates. If an old certificate has expired, remove the workload, reissue and validate it, and return it only when secure and ready.

## Configuration Safety

![Safe configuration delivery](assets/configuration-safety.svg)

[Open the scalable configuration safety diagram](assets/configuration-safety.svg)

Immutable versions support deterministic rollback, audit, drift detection, and reconciliation. Validation must test critical permitted paths such as `Payment → Ledger`; valid syntax cannot prove business safety. Canary by cohort and region, observe denials and customer outcomes, then expand. Avoid simultaneous global changes.

If failure interrupts a partial rollout, proxies normally retain their current valid version and rollout freezes. After recovery, confirm the approved desired version before reconciliation. If the older version permits a critical exploit, affected operations may need an emergency path or fail-closed behavior.

## Traffic Management

| Technique | Behavior | Safety rule |
|---|---|---|
| Canary | Shift 1% → 5% → 20% → 50% → 100% | Predefine SLO, errors, retries, saturation, dependencies, and correctness gates. |
| Blue/green | Validate a complete inactive environment, then switch traffic | Keep the former environment compatible and available for rollback. |
| Weighted/version routing | Send an explicit share to a version | Scope by service/cohort/region and verify capacity. |
| Shadow | Copy requests; ignore the candidate response | Suppress side effects; never duplicate banking writes. |

Fraud v2 at p99 280 ms against a 200 ms service SLO fails its gate even with normal 5xx and correct results: roll back, diagnose, fix, and canary again. At 20%, p99 190 ms, retries 3×, CPU 85% and rising, and added downstream calls require pausing. Current SLO compliance does not erase a saturation trend.

## Observability

| Signals | What they reveal |
|---|---|
| Rate, p50/p95/p99, errors, timeouts, retries, budget, breaker/outlier events | Tail regression, amplification, and resilience actions. |
| Proxy latency, pools, queues, CPU/memory, response flags | Application, proxy, network, or dependency fault. |
| Discovery version/age, endpoint count, readiness changes | Stale topology, slow convergence, or unusable capacity. |
| mTLS failures, authz denials, expiry/rotation, trust version | Identity, policy, and certificate faults. |
| Config version, drift, cohort, regional propagation | Unsafe rollout or stalled reconciliation. |
| End-to-end traces and service maps | Time across Payment → proxies → Fraud → Ledger. |
| Business outcomes and SLO/error-budget burn | Customer impact despite green infrastructure. |

Metrics answer **is something wrong?** Logs answer **what happened?** Traces answer **where did it happen across this request?** Propagate trace IDs across applications and proxies. Dashboards, alerts, and runbooks must let support distinguish application, proxy, discovery, network, identity/certificate, authorization, control-plane, and downstream failures.

## Multi-AZ & Multi-Region

![Locality-aware routing and failover](assets/locality-routing.svg)

[Open the scalable locality routing diagram](assets/locality-routing.svg)

Prefer local traffic while healthy and performant. Test cross-AZ connectivity with probes and controlled exercises rather than forcing every request across AZs. Two AZs at 60% each cannot absorb all traffic in one AZ without roughly 120% demand. Reserve headroom, enforce admission/concurrency limits, and shed lower-priority work to preserve critical transactions.

Normal Canadian transactions use Canadian Fraud and Ledger. A synchronous US dependency adds latency, cost, residency concerns, and cross-region failure propagation. Regional data planes remain autonomous; cross-region routing is a deliberate capacity- and compliance-aware recovery action.

## Constraint Mutations

| Changed constraint | Accepted response | Rejected / refined proposal |
|---|---|---|
| 200 → 2,000 services | Scale registry, discovery fan-out, control distribution, certificates, and telemetry | Service count alone does not mandate redesign; measure bottlenecks. |
| Fraud 10 → 500 in 30 seconds | Rapid registration, readiness, propagation, convergence, and LB use | Undiscoverable compute is not usable capacity. |
| Registry unavailable 10 minutes | Continue on recent endpoints; reconcile later | New instances may remain unused; avoid per-request registry calls. |
| Stale crashed endpoint | Health, lease/TTL, connect failures, passive signals | A crashed process cannot self-degrade or deregister. |
| Control plane unavailable one hour | Continue with LKG; freeze changes | Critical security exposure may require emergency action or fail closed. |
| 10% certificate rotation failure | Isolate expired workloads; prove remaining capacity | Do not weaken mTLS or globalize the failing cohort. |
| Four layers retry three times | One owner; bound attempts, time, budget, idempotency | Worst case is 81 attempts, not 12. |
| AZ1 fails with both AZs at 60% | Preplanned headroom, admission, priority shedding | Blind failover cascades; gradual movement may be impossible in sudden failure. |
| Bad authz policy global | Semantic tests, regional canary, progressive rollout, LKG | Global simultaneous deployment creates avoidable blast radius. |
| One endpoint p99 4 s, peers 100 ms | Latency routing, outlier ejection, root-cause analysis | HTTP 200/readiness alone is insufficient. |

## Adversarial Architecture Review

| Challenge | Outcome retained |
|---|---|
| Mesh selected solely because the platform has 200 services | Rejected. For discovery, LB, basic timeout, existing TLS, and observability, simpler platform capabilities suffice. |
| Retries configured independently in Payment and the mesh | Rejected. Conflicting deadlines and nested retries obscure and amplify behavior. |
| A 100 ms timeout with Fraud p99 at 180 ms | Rejected. It converts normal tail latency into failure load. |
| Enhanced screening rules implemented in the mesh | Rejected. Payment thresholds and screening decisions are application and domain logic. |
| Gateway edge controls moved into the mesh | Rejected. The gateway owns north-south API protection; the mesh owns east-west workload traffic. |
| Every AZ1 request forced through AZ2 | Rejected. Prefer local healthy capacity and test cross-AZ resilience through probes and controlled exercises. |
| Extremely short certificate lifetimes | Refined. Balance credential exposure with rotation availability; rotate early with overlap and continuous monitoring. |
| All deployed components retained indefinitely | Rejected. Remove the mesh if advanced identity, authorization, resilience, and traffic-management requirements disappear. |

Every component must earn its place. Registry/discovery handles ephemeral endpoints; internal LB selects; the mesh earns its cost only from required cross-cutting controls; observability makes that extra layer supportable.

## AI Intersection

![Governed asynchronous AI analysis](assets/ai-governance.svg)

[Open the scalable AI governance diagram](assets/ai-governance.svg)

Use asynchronous AI for anomaly detection and prediction: emerging retry storms, saturation, certificate-rotation clusters, abnormal dependency behavior, and forecast error-budget exhaustion. Deterministic monitoring already measures latency and current budgets; AI adds correlation, pattern recognition, and forecasting.

Reject AI in each Payment request to select Fraud. It adds latency, nondeterminism, availability coupling, cost, audit difficulty, and blast radius where deterministic signals work. Global critical-policy recommendations require validation, engineering review, performance/resilience tests, risk-appropriate approval, canary, observation, and rollback. Low-risk reversible actions may be automated within approved bounds.

## TPM Delivery

| Outcome-based workstream | Completion evidence / dependency |
|---|---|
| Service Discovery Foundation | Registry, lifecycle, readiness, convergence, internal LB, failure behavior. |
| Service Mesh Platform | Regional control/data planes, distribution, versioning, LKG, scale. |
| Service Identity & mTLS | Unique identities, trust, authz, issuance/overlap/rotation/revocation, alerts. |
| Resilience Standardization | Ownership, deadlines, safe retries, budgets, breakers, outliers, pools. |
| Traffic Management | Canary, blue/green, weights, shadow safety, gates, rollback. |
| Observability | Correlated telemetry, maps, dashboards, alerts, SLOs, diagnostic runbooks. |
| Application Onboarding | Dependency maps, policies, ownership, compatibility, wave certification. |
| Multi-AZ / Multi-Region | Locality, failure capacity, regional autonomy, policy boundaries, exercises. |
| Operational Readiness | L1/L2/L3 walkthrough, RACI, escalation, hypercare/on-call, security/MOR, simulation. |

### Critical path and dependencies

![Delivery critical path](assets/delivery-critical-path.svg)

[Open the scalable delivery critical path](assets/delivery-critical-path.svg)

Security architecture and operational work start early even though final approvals gate launch. Application assessment, dependency mapping, telemetry, test design, runbooks, manifests, and lower-environment work can continue during a six-week platform delay. Final mesh integration, mTLS validation, performance/resilience certification, and production onboarding wait for platform readiness.

## Program Risks

| RAID item | Assessment | Mitigation, trigger, contingency, owner function |
|---|---|---|
| Certificate rotation fails for 5% | **Issue / high risk**: reproducible and connectivity-breaking | Root-cause and retest full lifecycle; trigger below threshold/expiry margin; pause cohort, retain valid overlap, emergency reissue and isolate; Identity owner. |
| Discovery convergence lags autoscaling | Risk | Load-test churn and propagation SLO; trigger usable capacity trails compute; preserve headroom or admit priority traffic; Discovery/platform. |
| Bad mesh policy has global blast radius | Risk | Semantic tests, region/cohort canary, version/LKG; trigger denials/customer regression; freeze/rollback; Mesh/SRE. |
| Retries overload a dependency | Risk | One owner, safe-operation catalog, total deadlines/budgets; trigger retry and saturation rise; disable retries/open breaker; Service owner/SRE. |
| AZ survivor lacks capacity | Assumption/risk until proven | N+1 and loss test; trigger projected saturation; shed low-priority work; Capacity/SRE. |
| Platform delay blocks onboarding | Dependency/schedule risk | Track required-by dates, prepare teams in parallel; phase/rebaseline at latest safe integration date; TPM/platform. |
| Mesh is hard to troubleshoot | Readiness issue if unresolved | Layer dashboards, traces, runbooks, simulation; trigger L1/L2 cannot isolate faults; delay pilot; SRE/platform. |

Mitigation reduces probability or impact before occurrence; contingency executes after the trigger. An undefined fix-forward approach is inadequate for security-critical certificate rotation.

## TPM Constraint Mutations

| Changed constraint | Accepted response | Guardrail / correction |
|---|---|---|
| 200 services in four rather than eight months | Deliver platform plus prioritized subset in four months; complete remaining waves by month eight | Start with a representative lower-risk pilot, then higher criticality. Trade scope for time. |
| Schedule compression | Parallelize assessment/config prep, observability, security, runbooks, environments, later waves | Identity correctness, resilience/performance, InfoSec, readiness, rollback, and production evidence are non-compressible. |
| Platform slips six weeks | Continue maps, readiness, identity/telemetry/test/runbook preparation | Integration, mTLS validation, mesh testing, certification, production wait. |
| Pilot in 48 hours; troubleshooting weak | Conditional GO only if mandatory conditions close before decision | Complete walkthrough, RACI/escalation, hypercare, discriminating dashboards, incident simulation; otherwise delay. |

## Production Readiness

These are required gates, not claims that an implementation has passed them. Every gate needs a named owner, objective criteria, evidence, and sign-off.

| Gate | Evidence required before Go |
|---|---|
| Development / contracts | Unit/component and policy tests; identity, route, timeout/retry, and error contracts. |
| E2E integration | Payment → Fraud → Ledger, correctness, authn/authz, trace continuity. |
| Performance / capacity | Proxy overhead, p99, pools, registry/control scale, churn, telemetry, dependency ceilings. |
| Discovery failure | Registry outage, stale endpoints, readiness/drain, rapid-scale convergence, reconciliation. |
| Resilience | Deadlines, safe retries, budgets, jitter, breakers, outliers, retry-storm containment. |
| Control/config | One-hour outage, LKG safety, drift, invalid-policy rejection, canary, rollback, regional containment. |
| Security | Unique identities, mTLS/authz, certificate lifecycle, InfoSec approval. |
| AZ/region | Locality, survivor capacity, shedding, autonomy, partition, approved failover. |
| Observability / SRE | Layer-specific dashboards/alerts, SLOs, runbooks, simulation, walkthrough, on-call, escalation. |
| Rollback / decision | Tested code/config/traffic rollback, owners, thresholds, communications, residual-risk acceptance, MOR. |

**Go/No-Go:** conditional GO is valid only if all mandatory conditions close before the decision point. If support cannot diagnose application, proxy, discovery, certificate/authentication, policy, or downstream failures using actual tooling and runbooks, delay the pilot. A condition cannot be carried into production merely by naming it.

## Rollout & Rollback

Use **lower-environment validation → representative low-risk pilot → 1% → 5% → 20/25% → 50% → 100%**, with observation and approval at each boundary. Expand one bounded cohort or region at a time and preserve a known-good version.

| Trigger | Action | Recovery check |
|---|---|---|
| Invalid/unsafe policy | Block before rollout | Critical allow/deny paths pass. |
| Canary SLO/error/authz regression | Freeze and restore LKG | Customer operation and layer signals recover. |
| Retry/saturation/downstream-call rise | Pause before SLO breach | Root cause fixed; capacity and dependency behavior bounded. |
| Certificate rotation misses threshold | Stop cohort; retain valid overlap; isolate/reissue expired workloads | Identity authenticates, capacity safe, alerts clear. |
| Discovery/config drift after outage | Reconcile to approved state | Endpoint/config versions converge without regression. |
| AZ/region capacity unsafe | Limit admission or shed lower-priority work | Survivor retains reserve; critical flows healthy. |

Policy, application, certificate/trust, and traffic rollback are distinct operations. Shadow traffic must suppress writes. Blue/green rollback works only while the old environment and contracts remain compatible.

## Architecture Reference

| Concept | Working meaning / correction |
|---|---|
| Registry / discovery / internal LB | Store instance state / find usable endpoints / select one. |
| Liveness / readiness | Process alive / safe for production traffic. |
| Graceful deregistration / degradation | Drain and leave cleanly / continue with reduced function. |
| Client-side / server-side discovery | Client selects / infrastructure selects. |
| Control / data plane | Define and distribute policy / enforce live traffic. |
| Identity / mTLS / authz | Who calls / authenticate and encrypt / what it may do. |
| Circuit OPEN / fail closed | Stop dependency calls / deny or hold a business action. |
| Retry budget | Bound aggregate retry traffic, not merely attempts per request. |
| Outlier / service failure | One abnormal endpoint / dependency-wide failure. |
| LKG / safe stale policy | Preserve availability / only while policy and credentials remain acceptable. |
| SLO / error budget | Reliability target / allowed miss over its window. |
| Canary / blue-green / shadow | Progressive exposure / environment switch / non-authoritative copy. |

## Operating Principles

- Discovery answers **where**; load balancing answers **which one**; mesh governs **how service traffic behaves**.
- Application = domain logic; control plane = desired rules; data plane = runtime enforcement.
- Registered, alive, ready, and performant are different states.
- Registry and control plane update local state; they are not per-request dependencies.
- Locality is a preference while healthy and performant.
- Retry at one layer that understands operation semantics; retries multiply across layers.
- Circuit OPEN stops calls; fail closed blocks the business action without assurance.
- mTLS proves/protects identity in transit; authorization grants an operation.
- LKG cannot extend expired credentials or make newly unsafe policy safe.
- Plan capacity after failure, not before it.
- A mesh is justified by cross-cutting requirements, not service count.
- AI predicts asynchronously; deterministic controls govern live requests and changes.
- Readiness requires evidence that people can detect, diagnose, contain, and roll back platform failures.

---

**Design basis:** Topic 5 Service Mesh + Service Discovery architecture review, including accepted and rejected decisions, constraint mutations, adversarial review, AI integration, TPM delivery, production-readiness gates, and end-to-end validation. Requirements and numerical scenarios are planning assumptions; implementation thresholds and formal sign-offs remain to be assigned. Previous: [Topic 4 — API Gateway + Global Load Balancing](../04-api-gateway-global-load-balancing/README.md).
