# Topic 10 — CDN + Edge Architecture

[Back to Architecture Drills](../README.md)

**Status:** Complete

At 9:00 AM, a bank launches a campaign across Canada, the US, the UK and Ireland. Traffic grows from **100,000 requests per second at a 90% cache hit ratio** to **500,000 requests per second at a 70% hit ratio**. The public website still appears to be working at many edge locations, but origin demand has risen from **10,000 to 150,000 requests per second**. The origin's safe limit is **15,000**. At the same time, customers need to check balances and make payments. Serving the campaign must not exhaust the systems that serve their accounts.

This chapter designs the CDN and edge architecture that contains that failure. Start with two distinct paths: **public content is accelerated at the edge**, while **sensitive balance and payment requests remain on an approved regional banking path**. The architecture first shows those paths end to end; CDN, PoP, Anycast, DNS and GLB mechanics are introduced afterward, once the reader can see what problem each mechanism solves.

The central ownership rule is: **the edge accelerates approved content; regional banking services retain authority over identity, data access and financial outcomes.** Never cache sensitive balance or payment APIs. Never relax residency boundaries to rescue availability. Keep AI outside the synchronous banking path. The drill's recovery target is **five minutes**; a measured **12-minute failover is a NO-GO**. Numerical scenarios are teaching exercises, not production benchmarks or evidence of a successful launch.

## Contents

- [Problem & Requirements](#problem--requirements)
- [Final Architecture](#final-architecture)
- [End-to-End Flow](#end-to-end-flow)
- [Edge & Routing Fundamentals](#edge--routing-fundamentals)
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

A customer opening the campaign page should download images, scripts and approved public information from a nearby delivery point. A customer opening an account balance needs a current, authorized response from the appropriate banking service. Both requests may arrive through the bank's internet-facing estate, but they have different correctness, privacy and recovery contracts.

For the campaign example, four conditions must all hold:

1. Public content remains fast without assuming that every request is a cache hit.
2. Misses, refreshes, retries and failover traffic stay within the origin's safe capacity.
3. Sensitive Canadian and UK banking data stays within the processing and storage boundaries approved for this scenario.
4. A CDN, origin or regional failure has a rehearsed recovery path that meets the five-minute target without bypassing the first three conditions.

| Requirement | Decision and consequence |
|---|---|
| Fast public delivery across markets | Cache approved public content at suitable PoPs; measure user latency by geography and provider. |
| Correct content and safe updates | Explicit keys, TTLs, validators, versioned assets and verified invalidation; cache behavior is a release contract. |
| Protect sensitive banking APIs | Explicit cache bypass plus `Cache-Control: no-store`; authorization and financial correctness remain regional service responsibilities. |
| Survive provider and origin failure | Multi-CDN where justified, independent origin failure domains, eligible failover destinations and rehearsed failback. |
| Campaign elasticity | Prescale predictable demand; autoscale measured bottlenecks; cap misses, retries and background refresh. |
| Canadian and UK residency | Enforce the scenario's approved boundaries at ingress, gateway, service, data, telemetry and operational access layers. |
| Defense in depth | DDoS controls, WAF where TLS inspection is permitted, origin authentication, network restrictions and service authorization. |
| Recovery within five minutes | Measure detection through useful service restoration under realistic load; 12 minutes fails the gate. |
| Governed operations | Named owners, test evidence, runbooks, change control, privacy approval and formal ARB decisions. |

**Scope boundary.** This chapter covers delivery, caching, traffic steering, origin protection and operational governance. It does not redesign the bank's ledger, payment authorization or database recovery protocol. CDN recovery cannot establish the RPO of banking records; those systems need their own approved durability and reconciliation contracts.

**Inputs still requiring production validation.** Request mix, object sizes, geographic distribution, provider features, minimum hit ratio, safe origin rate, dependency limits, scaling time, purge convergence and alert thresholds must be measured. The 15,000-RPS safe origin rate is the drill's stated constraint. A production safe rate must be established at the required latency and error SLO, not inferred from CPU alone.

## Final Architecture

The baseline diagram separates public delivery from the sensitive banking path before introducing provider failure and regional restrictions.

### Architecture at a glance

![CDN delivery and regional banking boundaries](assets/edge-flow.svg)

[Open the scalable edge-flow diagram](assets/edge-flow.svg)

Public requests enter an approved CDN, pass applicable security checks and use a cache only when the content policy permits it. Cache misses go through origin admission controls. Sensitive API requests are directed to an approved regional entry point and bypass shared caches. If the bank permits a CDN proxy for these APIs, its TLS termination, processing and logging locations must satisfy that same boundary.

The diagram's two paths are logical policies. Separate hostnames such as `www.bank.example` and `api-ca.bank.example` make them easier to isolate, but a hostname is not authorization. Every gateway and service must still enforce the trusted account-region policy.

### Baseline and extensions

Begin with explicit content classification, one provider, a resilient origin tier and a regional banking gateway. Add a second CDN to address measured provider risk or delivery needs. Add independent origins and suitable regional deployments to address origin and jurisdictional failure. Each extension must preserve the same security and capacity invariants.

A one-to-one mapping between CDN providers and origins is an option, not a requirement. Two CDNs can use a resilient shared origin tier. Two dedicated origins can still share a database, power domain, DNS zone, secret, deployment pipeline or overloaded connection pool. The required property is **independent failure domains with enough tested survivor capacity**, not a particular box count.

## End-to-End Flow

### Public campaign request

1. **Resolve and steer.** A browser resolves the public hostname. Global policy selects an eligible CDN using health, geography, capacity and configuration readiness.
2. **Reach an edge.** The selected provider routes the connection to a PoP, potentially through Anycast. The browser validates the certificate for the bank's hostname.
3. **Terminate and inspect where approved.** An HTTP CDN normally terminates client TLS, applies supported WAF and request policies, then uses a separately authenticated TLS connection upstream when necessary.
4. **Classify the request.** The edge checks the host, path, method, authorization context and content policy. Only approved public routes are cacheable. A filename extension alone does not establish that a response is public.
5. **Construct the key.** The key includes all approved dimensions that change the representation, such as canonical path, locale and selected query parameters. Untrusted or irrelevant variation does not create unlimited variants.
6. **Serve or fetch.** A fresh eligible hit returns immediately. A miss or required revalidation uses the shield and protected origin budget. Concurrent refreshes for the same object should be coalesced where supported.
7. **Validate the response policy.** The origin returns content, cache directives and a validator such as an ETag. The edge stores it only if both the response and the configured policy permit storage.
8. **Observe delivery.** Record status, cache outcome, object version, provider, PoP, origin latency and policy version without placing sensitive content in ordinary logs.

### Sensitive balance or payment request

1. The client reaches an approved regional TLS endpoint. A Canadian customer travelling in Ireland does not become an Irish banking-data tenant because of their IP address.
2. The gateway validates identity, token issuer, audience, expiry and the trusted account or tenant mapping. It derives the permitted region from authoritative policy, not from a caller-supplied `X-Region` header.
3. Balance and payment routes explicitly bypass caches. Responses use `Cache-Control: no-store`; errors, redirects and edge-generated responses for these routes require the same review.
4. The regional service independently checks tenant, account authorization and processing region. Network policy and data credentials limit access to approved stores and dependencies.
5. The banking service owns the result. Payment retries preserve the application's idempotency contract; an edge timeout does not prove a payment failed and must not trigger a blind cross-origin replay.
6. If no compliant, sufficiently provisioned destination is available, return a controlled failure. Do not redirect a sensitive request to another country or fabricate a cached balance to make the site look available.

These paths reconnect only at operational oversight: both need telemetry, rate budgets, release governance and incident ownership. Their caching and data-processing permissions remain distinct.

## Edge & Routing Fundamentals

Now that the two request paths are clear, the supporting edge and routing concepts can be read in context. These mechanisms help deliver or steer traffic; none of them replaces the banking services that own identity, authorization, residency, or financial correctness.

### CDN and edge fundamentals

A **Content Delivery Network (CDN)** is a distributed delivery layer between clients and origins. A **point of presence (PoP)** is a location containing edge infrastructure. An **edge cache** stores reusable HTTP representations near clients. The **origin** supplies the authoritative version of the content; it may be an object store, web server or application tier.

A **cache hit** serves an eligible stored representation. A **miss** requires an upstream fetch. An expired representation may be **revalidated** rather than downloaded again. An **origin shield** is an intermediate cache that can consolidate fetches from many edge locations. It reduces duplicate upstream work but becomes another dependency that must be included in failure analysis.

**Edge compute** runs bounded logic near the user: redirects, request normalization, lightweight public-content selection or access checks supported by the platform. It is not a reason to move authoritative balance calculations, payment execution or unapproved personal-data processing to arbitrary PoPs. A small function can still violate a data boundary or create a large cache-key explosion.

A CDN is also not simply a database cache. It applies HTTP request, response, freshness and variation rules across clients, intermediary caches and origins. A fast response is useful only if it belongs to the right audience, is sufficiently current and is allowed to be served from that location.

### DNS, Anycast and global load balancing

These mechanisms act at different layers:

| Mechanism | Working meaning and limitation |
|---|---|
| DNS | Resolves the bank's hostname, often through a CDN alias or traffic-steering service. Resolver caches and client behavior mean a DNS change does not instantly move every user. |
| Anycast | Advertises the same IP prefix from multiple locations; network routing selects a reachable path. The selected PoP is not necessarily the geographically closest or the least loaded application endpoint. |
| Global Load Balancing (GLB) | Chooses eligible providers or regions using policy, health, capacity and geography. It may operate through DNS or a proxy; the implementation determines convergence and failure behavior. |
| Regional load balancing | Distributes admitted traffic among healthy instances within an approved regional boundary. It cannot make an unhealthy shared database disappear. |
| TLS | Authenticates the server and encrypts a connection. Where TLS terminates determines where plaintext can be inspected or processed. |

DNS can choose a CDN; that CDN can use Anycast to reach a PoP; the edge can then select an origin. These are separate decisions, not three names for the same routing operation. A GLB health result must be combined with residency eligibility and survivor capacity before it authorizes traffic movement.

## Component Responsibilities

| Component | Responsibility, purpose and boundary |
|---|---|
| DNS / global steering | Selects eligible providers and destinations.<br>**Why:** Clients need reachable delivery endpoints.<br>**Boundary:** DNS geography and health are not account authorization or instant failover. |
| CDN edge / PoP | Terminates approved connections, delivers eligible cached content and executes bounded edge policy.<br>**Why:** Reduce latency and repeated origin work.<br>**Boundary:** It is not the authority for a customer's balance or payment outcome. |
| WAF / DDoS / bot controls | Filter malicious traffic and constrain abuse where inspection is possible.<br>**Why:** Protect service availability and reduce attack exposure.<br>**Boundary:** A WAF cannot inspect encrypted HTTP it cannot decrypt, or enforce every business rule. |
| Cache policy and key rules | Define storage eligibility, variants, freshness and stale behavior.<br>**Why:** Reuse must preserve correctness.<br>**Boundary:** A high hit ratio never justifies cross-user leakage. |
| Origin shield / request coalescing | Consolidate repeated upstream fetches and refreshes.<br>**Why:** Many PoPs can miss simultaneously.<br>**Boundary:** Shield failure and cold starts still need bounded origin admission. |
| Origin admission / regional load balancer | Enforce safe aggregate rates, concurrency and priority allocation.<br>**Why:** Cache misses are real application load.<br>**Boundary:** Per-provider quotas must sum to a safe total. |
| Public origin tier | Serves versioned public artifacts and approved public responses.<br>**Why:** Caches need an authoritative refill source.<br>**Boundary:** Replicas must not conceal common dependencies or serve unapproved versions. |
| Regional gateway | Validates identity, account-region policy and API admission.<br>**Why:** Routing intent needs enforceable access control.<br>**Boundary:** It cannot substitute for service-level authorization. |
| Banking services and data | Enforce account access, residency and financial semantics.<br>**Why:** Sensitive operations require authoritative controls.<br>**Boundary:** Edge retries and failover cannot override those controls. |
| Configuration / certificates / purge control | Versions policy, rotates trust and coordinates provider changes.<br>**Why:** Multi-CDN behavior must remain consistent.<br>**Boundary:** An accepted API call is not evidence of global propagation. |
| Telemetry / SRE / AI advisory | Measures customer outcomes, predicts pressure and supports operators.<br>**Why:** Provider health alone misses end-to-end failure.<br>**Boundary:** AI cannot become a synchronous or autonomous banking decision dependency. |

## Data & Consistency

The public origin owns the published representation. A cache owns a reusable copy under a time- and policy-bounded contract. Banking services own sensitive data and financial outcomes. Treating those three forms of ownership as interchangeable creates either stale business decisions or accidental disclosure.

### Content classification comes before TTL

| Content class | Storage and freshness policy |
|---|---|
| Fingerprinted public JavaScript, CSS and images | Long freshness lifetime with immutable URLs; publish a new URL when bytes change. Keep older referenced versions available during rollout and rollback. |
| Public campaign HTML or product information | Explicit short freshness policy, validators and approved stale allowance where content risk permits. Locale, market and legal version may affect the key. |
| Time-sensitive public notices or rates | Product owner defines freshness and stale tolerance. “Public” permits distribution; it does not mean stale content is harmless. |
| Authenticated balances, payment initiation, payment status and sensitive account APIs | **Never cache.** Explicit route bypass, `no-store`, no stale fallback and no reuse across requests. |
| Personalized pages, session or token responses | Non-cacheable by default in this design. Review redirects and failures as carefully as successful responses. |

Cache eligibility is an allowlist. Newly introduced APIs do not become cacheable merely because the CDN's default policy caches `GET` responses. A balance lookup may use `GET` and still be highly sensitive.

### Cache keys: decide which requests may share a response

A cache key is an equivalence claim: every request mapped to this key is allowed to receive the same representation. For a public campaign asset, an illustrative key is:

```text
canonical host + normalized path + approved market/locale
               + allowlisted representation-changing query parameters
               + supported content-encoding variation
```

Suppose `/campaign?lang=en&utm_source=mail` and `/campaign?lang=fr&utm_source=mail` return different text. Dropping `lang` can serve the wrong language. Retaining every random tracking parameter can destroy cache reuse. The policy must preserve meaningful variation while excluding only parameters proven irrelevant to the response.

Do not normalize paths or query strings differently from the origin. If the edge thinks two requests are equivalent while the origin treats them differently, an attacker may poison the shared representation. Test duplicate parameters, encoded paths, case sensitivity, host headers, cookies and `Vary` behavior across both providers. The safe response to uncertain personalization is bypass, not a speculative universal key.

For this banking design, adding customer IDs or authorization tokens to a cache key is not an alternative to the no-cache rule for sensitive APIs. It creates more sensitive storage and more ways to mis-key a response.

### TTL, validators and storage directives

**TTL** is a freshness lifetime, not a promise that an object remains physically resident. Eviction may happen sooner. `max-age` sets freshness for caches generally; `s-maxage` controls shared-cache freshness. `no-cache` permits storage but requires validation before reuse. `no-store` forbids storage. `private` excludes shared-cache storage but is insufficient by itself for this design's sensitive APIs. These distinctions follow [HTTP caching semantics in RFC 9111](https://www.rfc-editor.org/rfc/rfc9111.html).

An **ETag** identifies a representation version. After freshness expires, a cache can send `If-None-Match` with the validator. A `304 Not Modified` response allows reuse of the stored body under updated metadata; a changed representation requires a new response body. Validation saves bytes, but it still reaches an upstream service and can consume connections, CPU or database work. A fleet of simultaneous validations can overload an origin even when transfer bandwidth stays modest.

Illustrative public HTML policy, subject to product approval:

```http
Cache-Control: public, max-age=60, s-maxage=300, stale-while-revalidate=30, stale-if-error=600
ETag: "campaign-2026-11-v3"
```

This example gives browsers a 60-second freshness lifetime and shared caches 300 seconds. The extensions permit bounded stale reuse during background validation or eligible errors, where implemented. They do not create a copy on a cold cache and must not override stricter content restrictions. See [RFC 5861](https://httpwg.org/specs/rfc5861.html) for the stale extensions. These numbers are illustrative, not drill-approved production settings.

Illustrative versioned public asset:

```http
GET /assets/campaign.8d42c1.js
Cache-Control: public, max-age=31536000, immutable
```

`immutable` means the representation will not change during its freshness lifetime; it is appropriate only when publication preserves that promise. A new build receives a new URL. See [RFC 8246](https://httpwg.org/specs/rfc8246.html).

Sensitive API response policy:

```http
Cache-Control: no-store
```

Apply this together with explicit CDN bypass and checks at every intermediary. A response header cannot repair a configuration that overrides it or remove a sensitive copy previously cached by mistake. Such an event requires containment, purge verification and incident handling.

### Invalidation, publication and convergence

A **purge** removes or invalidates a stored entry. A **versioned URL** creates a new identity. Purging every object during a campaign turns cache capacity into synchronized origin demand; versioned assets make a normal deployment more predictable.

Publish assets before publishing HTML that references them. Verify the new objects at each origin and CDN, warm a bounded set of hot public assets, then release the HTML. Retain previous assets for older pages, in-flight clients and rollback. Never overwrite an immutable URL with different bytes.

For mutable public content, use targeted URL or tag invalidation, track a request identifier per provider and verify content versions from representative PoPs. Multi-CDN purge is not one atomic transaction. One provider may serve the new version while another serves the old version until propagation completes. Define the allowed convergence window and what to do if it is exceeded.

Deleting or purging an edge entry does not recall a browser copy. Sensitive content therefore must not enter caches in the first place. Urgent legal or security removal may require disabling the route or serving a controlled error while global removal is verified; the ordinary stale policy must not keep prohibited content alive.

## Scaling

The CDN absorbs reusable requests. The origin must survive everything else. Capacity planning starts with misses and uncached traffic, then adds revalidation, retries, refresh, warm-up and failover. Edge throughput is not evidence of origin safety.

### Capacity exercises that changed the reasoning

For the drill's simplified aggregate request mix:

```text
Origin RPS = incoming RPS × (1 − request hit ratio)

Baseline: 100,000 × (1 − 0.90) = 10,000 origin RPS
Campaign: 500,000 × (1 − 0.70) = 150,000 origin RPS
Safe origin ceiling: 15,000 RPS
Campaign overload: 150,000 / 15,000 = 10× safe capacity
```

| Scenario | Result and implication |
|---|---|
| 100k RPS, 90% hits | 10k origin RPS; 5k RPS remains below the stated ceiling, before extra overhead. |
| 500k RPS, 90% hits | 50k origin RPS; preserving the original hit ratio still exceeds safe capacity. |
| 500k RPS, 70% hits | 150k origin RPS; 135k RPS above the safe ceiling. The origin sees 15× its baseline demand. |
| 500k RPS against 15k capacity | Required hit ratio is at least `1 − 15k/500k = 97%`, ignoring all overhead. This is an arithmetic boundary, not a safe operating target. |
| 70% hits with a 15k origin limit | At most `15k/0.30 = 50k` total RPS is supportable under this simplified mix without additional measures. |
| Cold survivor cache at 500k RPS | Offered origin load can approach 500k RPS. Admission controls must remain safe even before cache warmth returns. |

![Origin pressure and the campaign overload calculation](assets/origin-capacity.svg)

[Open the scalable origin-capacity diagram](assets/origin-capacity.svg)

The campaign has two multipliers: total requests grow fivefold, and the miss fraction grows from 10% to 30%. Origin demand therefore grows **5 × 3 = 15 times**. A dashboard focused only on overall RPS or hit percentage can hide that relationship.

For a mixed workload, use a more explicit model:

```text
Origin offered load = public RPS × (1 − public hit ratio)
                    + uncached API RPS
                    + validation / refresh / retry / warm-up load
```

Do not add uncached traffic twice if the measured aggregate hit ratio already includes it. Distinguish request hit ratio from byte hit ratio: serving a few large images from cache can save bandwidth while thousands of small dynamic calls still saturate the application.

RPS also hides work per request. Measure concurrency, response time, CPU, network throughput, database calls, connection pools and expensive routes. At 15k RPS and 200 ms mean service time, roughly 3,000 requests are in flight in steady state; longer service time increases concurrency even if arrivals stay fixed. This is an illustrative application of `concurrency ≈ arrival rate × mean time`, not a sizing guarantee.

### Prescaling and bounded elasticity

Prescaling prepares known campaign demand before users arrive. It includes application instances, connection pools, downstream quotas, object-store limits, TLS capacity and provider commitments. Autoscaling responds to observed demand, but provisioning, initialization and cache warming take time. Scaling the web tier cannot overcome a fixed database bottleneck.

1. Forecast demand and plausible low-hit or cold-cache conditions.
2. Load-test the full dependency chain and establish a safe envelope with reserve.
3. Prescale before the campaign, verify readiness and warm approved hot assets at a bounded rate.
4. Apply aggregate origin admission across all CDNs, shields and background jobs.
5. Reserve banking-service capacity; defer or shed lower-priority public work when the budget tightens.
6. Ramp only while customer latency, errors and dependency saturation remain inside the approved envelope.

Two CDNs each allowing 15k RPS into one 15k-RPS origin tier do not provide protection. Budgets must share the same global envelope, with conservative behavior when coordination is unavailable. Separate public and banking pools where needed so a public miss storm cannot exhaust the banking connection, CPU or database budget.

### Emergency campaign response

At 150k offered origin RPS, the immediate goal is to reduce work before an overloaded origin collapses. Scaling may help later; it is not the first line of protection during a provisioning delay.

| Strategy | When it helps / required boundary |
|---|---|
| Serve approved stale public content | Buys time when a usable cached copy exists and the stale window permits it. Never serve stale balances, payment responses, withdrawn notices or expired offers that are no longer valid. |
| Edge-generated temporary redirect | Sends a public campaign route to a prevalidated lightweight page or separately provisioned destination. Generate it before the overloaded origin, avoid loops and confirm destination capacity, certificates and policy. |
| Versioned assets and controlled warm-up | Keep existing public assets reusable and prepare new versions gradually. Versioning alone does not fix an already cold fleet; a mass new version can increase misses. |
| Request coalescing / shield | Reduces repeated fetches of the same object. It does little for millions of distinct keys or uncached APIs. |
| Rate limits / concurrency limits / circuit breakers | Bound work and stop futile upstream calls. Return explicit failure or approved fallback rather than unbounded queues and retries. |
| Reduce campaign demand | Pause rollout, reduce paid traffic or disable optional features with the business owner. Protect critical banking capacity while recovery proceeds. |

A redirect is useful only when its destination escapes the bottleneck. Redirecting to another hostname backed by the same saturated origin merely moves the queue. Do not redirect sensitive API operations or replay payment bodies as a campaign workaround.

## Reliability & Failure Modes

Failure analysis follows the whole path: resolver, provider, edge policy, shield, origin, network, identity, banking service and data. At each boundary ask **what remains healthy, what traffic moves, what becomes cold, which policy still applies and whether the survivor can carry the load**.

| Failure and remaining state | Detection, recovery and safety |
|---|---|
| One PoP is unreachable; other provider locations remain healthy | **Detect:** Geographic synthetics and user errors.<br>**Recover:** Provider routing may move connections; verify the actual client experience and cold-cache pressure. |
| CDN A fails; CDN B and its origin path remain available | **Detect:** Independent probes and customer signals.<br>**Recover:** Shift only eligible traffic, progressively where possible; ensure B's configuration, certificates and capacity are ready. |
| Both CDNs depend on a failed shared origin | **Detect:** Edge delivery works for hits but both providers fail on misses.<br>**Recover:** Healthy independent origin capacity or approved stale public content. Provider switching alone does not repair the origin. |
| Origin replica fails while another survives | **Detect:** End-to-end health and dependency checks.<br>**Recover:** Route within permitted boundaries and measured survivor capacity; constrain misses and retries during the transition. |
| Origin instances survive but the shared database fails | **Detect:** Application probes fail despite healthy instance checks.<br>**Recover:** Use the data tier's recovery procedure or fail safely. More origin replicas do not remove this dependency. |
| Global purge, expiry or new release creates a miss storm | **Detect:** Miss/validation RPS and origin concurrency rise.<br>**Recover:** Coalesce, admit within budget, use permitted stale copies and stop broad invalidation. |
| Purge succeeds in one CDN but lags in the other | **Detect:** Object-version probes and provider purge status disagree.<br>**Recover:** Complete targeted invalidation or withdraw affected content; do not call the release globally complete. |
| DNS changes but some clients retain the old destination | **Detect:** Requests continue on old provider; regional probes diverge.<br>**Recover:** Keep safe overlap where possible and account for convergence in recovery time. |
| TLS certificate or origin authentication breaks | **Detect:** Handshake errors, expiry alerts and authenticated-origin probes.<br>**Recover:** Restore valid trust using a tested rotation/rollback procedure; never disable verification to recover traffic. |
| Residency metadata is missing or inconsistent | **Detect:** Gateway/service policy rejects and audit correlation.<br>**Recover:** Reject or use an explicitly authorized identity-resolution flow. Do not default to the nearest country. |
| Edge deployment makes private routes cacheable | **Detect:** Negative cache tests, response-policy checks and synthetic cross-user probes.<br>**Recover:** Disable the rule, contain and purge, investigate exposure and retest before expansion. |
| AI advisory service fails | **Detect:** Telemetry/model health and stale recommendation timestamps.<br>**Recover:** Stop relying on recommendations; deterministic delivery and protection continue. |

### Multi-CDN and independent origins

![Multi-CDN recovery within jurisdiction and origin failure domains](assets/multi-cdn-residency.svg)

[Open the scalable multi-CDN and residency diagram](assets/multi-cdn-residency.svg)

The public plane may use origins across regions when content policy permits. The sensitive plane uses independently resilient Canadian and UK service boundaries. Canada must not automatically fail over to the US, nor the UK to Ireland, merely because those locations are nearby. For this scenario, fail over within the approved country boundary or fail safely. A separately approved transfer policy would be a changed requirement, not an implicit consequence of GLB routing.

Multi-CDN introduces its own costs: duplicated configuration, certificate lifecycle, cache fragmentation, inconsistent WAF semantics, purge coordination, split telemetry and different failure behavior. A lightly used secondary provider may have a colder cache than the primary. Test that condition rather than assuming its advertised edge capacity proves readiness.

Maintain independent failure-domain analysis for DNS/steering, control planes, cloud regions or zones, origin networks, identity, data, secrets and deployment automation. Independence is never absolute; document unavoidable shared dependencies and their recovery contracts. A resilient shared origin tier can be simpler than dedicated per-CDN origins, while dedicated origins can provide useful isolation when downstream dependencies and capacity support it.

### Five-minute recovery means useful service

The recovery clock starts at the agreed incident onset and ends when the affected service is useful at its defined success, latency and policy criteria. It includes detection, decision, steering propagation, connection recovery, cold-cache refill and origin stabilization. Some phases overlap; measure the complete event rather than adding optimistic component estimates.

An illustrative five-minute budget might allocate 30 seconds to detection, 30 seconds to decision, 120 seconds to steering/client convergence and 120 seconds to restoration verification. This is a planning allocation, not a measured result or a guarantee that DNS respects a two-minute deadline.

The drill observed **12 minutes against a five-minute target**: a seven-minute miss, or 2.4 times the allowed recovery duration. The release decision is **NO-GO**. Diagnose the slow phases, remediate, repeat a representative exercise and obtain fresh sign-off. Passing functional tests or switching DNS within seconds does not change that outcome.

Failback is another controlled traffic migration. Repair and verify the recovered destination, compare policy and content versions, warm within budget, canary traffic, observe and then ramp. Health hysteresis and hold-down periods prevent repeated provider oscillation from creating successive cold-cache storms.

## Security

The design protects availability and confidentiality at different layers. A **Web Application Firewall (WAF)** evaluates supported application-layer request rules. DDoS protection absorbs or filters network and application abuse. A WAF detects supported HTTP attack patterns. Authentication establishes identity. Authorization determines permitted actions. Residency policy determines allowed processing and storage. None of these controls replaces the others.

### TLS termination and conditional passthrough

An HTTP CDN normally terminates TLS to read requests, evaluate cache keys, inspect HTTP and serve cached responses. Re-encrypting the edge-to-origin connection protects that second link; it does not erase plaintext processing at the edge.

**TLS passthrough is conditional.** It requires a provider and service mode that supports forwarding encrypted traffic to a permitted termination point. With genuine passthrough, the intermediary cannot perform ordinary HTTP caching or inspect encrypted HTTP with a WAF. Provider L4 services are not interchangeable with their HTTP CDN products; [Cloudflare Spectrum's configuration documentation](https://developers.cloudflare.com/spectrum/reference/configuration-options/) illustrates this distinction. Validate the contracted product's actual behavior instead of assuming that every CDN offers equivalent passthrough.

Where the bank requires sensitive plaintext to remain within Canada or the UK, use approved regional TLS termination or a verified passthrough/direct regional design. Account for metadata, logs and support access even when payloads remain encrypted. A “cache bypass” setting alone does not establish residency.

### Origin restriction and defense in depth

| Area | Required control and boundary |
|---|---|
| Public edge | DDoS protection, WAF where inspection is allowed, bot controls and route-specific rate limits; test false positives on genuine users. |
| Origin network | Private connectivity where supported, or restricted ingress from approved delivery paths. Block direct public bypass. |
| Origin authentication | mTLS or equivalent authenticated origin access, with certificate rotation and tenant-specific trust. Provider IP allowlists alone do not identify the bank's distribution on shared infrastructure. |
| Forwarded identity and IP | Strip untrusted forwarding headers; accept metadata only from authenticated proxies. Do not authorize on a spoofable header. |
| Gateway and services | Validate identity and account scope independently; enforce trusted region mapping and least-privilege data credentials. |
| Cache safety | Public-route allowlist, sensitive-route bypass, poisoning/deception tests and no sensitive bodies, tokens or identifiers in cache keys. |
| Control plane | Least privilege, reviewed changes, separate deployment roles, protected purge credentials and immutable audit of emergency actions. |
| Logs and analytics | Minimize personal data, redact tokens, restrict retention and keep processing/storage/access within approved boundaries. |

Origin restrictions must apply to alternate hostnames, old addresses and disaster-recovery endpoints too. A hidden origin hostname is not an access control. Otherwise attackers can bypass the CDN's protection and hit the constrained origin directly.

### Canadian and UK banking data residency

The drill imposes Canadian and UK processing boundaries. Treat them as **scenario requirements approved by the bank**, not as a blanket statement that all Canadian or UK law universally prohibits international processing. Canada's privacy regulator explains that PIPEDA does not generally prohibit cross-border processing; UK rules govern restricted transfers and the conditions for them. See the [Canadian cross-border processing guidance](https://www.priv.gc.ca/en/privacy-topics/airports-and-borders/gl_dab_090127/) and the [ICO guide to international transfers](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/international-transfers/a-brief-guide-to-international-transfers/). Privacy, Legal and Compliance must determine the actual production obligations.

For this design, enforce the stricter stated boundary throughout the request lifecycle:

1. **Classify:** Identify public content, personal information, sensitive financial payloads and operational metadata.
2. **Bind identity:** Derive account/tenant jurisdiction from trusted records or validated claims bound to those records. A user's current location is not the data's jurisdiction.
3. **Constrain entry:** Ensure permitted TLS termination and processing before sensitive payloads reach an unauthorized edge.
4. **Enforce twice:** Gateway rejects ineligible regional requests; services independently verify region and account access.
5. **Constrain dependencies:** Apply policy to databases, replicas, backups, queues, fraud dependencies, logs, traces, exports and administrative access.
6. **Constrain recovery:** Select only compliant destinations with validated capacity; deny service rather than silently crossing a forbidden boundary.

Public content may cross regions when its content and distribution policy permits it. Public pages can still generate personal telemetry such as IP addresses or identifiers, so approval of the page does not automatically approve every analytics destination. Likewise, encrypting data in transit does not settle where storage, decryption, support access or processing is allowed.

## Observability

Measure the user's outcome, the cache's behavior and the origin's health together. The **95th and 99th percentile latencies (p95/p99)** expose slow-user experience; **Service Level Objectives (SLOs)** define the agreed success and latency targets. A provider can report healthy edges while misses fail, one country violates its latency target, or all banking calls are rejected by a broken gateway rule.

| Signals | Meaning / response |
|---|---|
| User success, p95/p99 latency and errors by market/provider/route | Shows whether delivery is useful; separates public browsing from banking availability. |
| Hit, miss, bypass, expired, revalidated and stale outcomes | Explains origin demand. Separate public request hit ratio, aggregate ratio and byte hit ratio. |
| Miss/validation RPS and origin concurrency | Predicts overload sooner than total CDN volume alone. Include refresh and retry traffic. |
| Origin latency, CPU, connections, database pressure and `429/5xx` | Identifies the actual bottleneck and whether admission limits are protecting it. |
| WAF actions, DDoS volume, bot scores and false positives | Distinguishes malicious load from legitimate campaign demand and blocked customers. |
| DNS answers, provider steering share and client convergence | Shows whether the intended traffic shift actually happened. |
| Certificate expiry, TLS failures and origin-auth rejects | Exposes trust and rotation failures before or during cutover. |
| Content version and purge convergence by provider/PoP | Detects stale or mixed releases that aggregate availability misses. |
| Gateway/service residency rejects and policy versions | Finds misrouting and configuration drift; rejected requests alone do not prove no earlier unauthorized processing occurred. |
| Failover phase timestamps and sustained restoration | Provides evidence for the five-minute target, including cold-cache and survivor load. |
| AI forecast error, lead time and recommendation age | Evaluates advisory value without making model availability a service dependency. |

Correlate request ID, provider, PoP, route class, region, policy version and origin target. Keep tokens, account data and sensitive query values out of general telemetry. Edge `Age` and provider cache-status fields are diagnostic signals, not authorization proofs.

Dashboards separate **User Experience / Public Cache / Banking APIs / Origin Capacity / Security / Residency / Recovery**. Alerts need a sustained-window policy, named owner and runbook. Use a leading capacity signal such as predicted miss demand plus rising concurrency, alongside hard limits that act without waiting for a dashboard or human response.

Synthetic checks should include public hits, forced safe misses, authenticated test-account flows, wrong-region rejection and direct-origin denial from relevant geographies. A public homepage returning `200` is insufficient evidence that banking recovery succeeded.

## Key Architecture Decisions

| Decision / disposition | Reasoning and trade-off |
|---|---|
| **Accepted:** multi-region, multi-CDN where justified | Improves geographic delivery and provider resilience. Adds configuration, cost, cache fragmentation and operational complexity; all routing remains policy-constrained. |
| **Accepted:** regional residency and security enforcement | Trusted identity, gateway controls and independent service checks enforce the banking boundary. Some failures require controlled unavailability. |
| **Accepted:** elastic origins with prescaling | Known campaigns receive capacity before launch; autoscaling supports measured variation. Dependencies and hard budgets still limit throughput. |
| **Required property:** independent origin failure domains | Removes common origin failure risks only when networks, data dependencies and survivor capacity are addressed. |
| **Refined:** dedicated origin per CDN | Useful in some isolation designs, but optional. A resilient shared tier may satisfy the same requirement with less duplication. |
| **Accepted:** explicit public cache policy and immutable assets | Improves reuse and predictable publication. Mutable content still needs freshness, invalidation and convergence controls. |
| **Accepted:** deterministic origin protection | Rate/concurrency limits, circuit breakers and priority isolation remain active during every provider transition. |
| **Rejected:** cache sensitive balance/payment APIs | Speed does not justify financial staleness or disclosure. Bypass and no-store are mandatory here. |
| **Conditional:** TLS passthrough | Only with verified provider support and understood loss of HTTP inspection/caching at that intermediary. |
| **Rejected:** nearest region determines residency | Client geography and untrusted headers are not authoritative account-policy inputs. |
| **Rejected:** 12-minute recovery meets a five-minute contract | Release remains NO-GO until remediation and representative retesting establish compliance. |
| **Accepted:** asynchronous AI with human-reviewed changes | Forecasting helps preparation; deterministic safeguards own emergency protection. No AI in synchronous banking. |

The final **Architecture Review Board (ARB)** defense prioritizes **multi-region, multi-CDN resilience**, **regional data-residency and security enforcement**, and **elastic origin capacity with prescaling**. Independent origin failure domains remain a required property of that architecture even though dedicated origins are not one of the three headline decisions.

## Constraint Mutations

The following mutations consolidate the drill's reasoning into reviewable operating responses. They are architectural exercises, not claims that these incidents occurred in production.

| Changed constraint | Accepted response / correction |
|---|---|
| Traffic becomes 500k RPS and hit ratio falls to 70% | Calculate 150k offered origin RPS, constrain to the safe envelope, preserve banking reserve and activate approved public fallback. |
| Autoscaling cannot finish before the campaign spike | Prescale earlier; use bounded stale/redirect/degradation now. Waiting for instances is not origin protection. |
| CDN A fails while CDN B's cache is cold | Model cold misses, confirm B's policy and capacity, warm/ramp within budget and shed optional load when needed. |
| Both CDNs share one origin | Replicate the origin across independent failure domains and validate its dependencies; another CDN alone is insufficient. |
| Canada is unhealthy but the US is available | Under this scenario, use approved Canadian recovery or fail safely. Availability does not grant a residency exception. |
| Public campaign content may be served internationally | Use permitted cross-region delivery for that content while retaining controls on sensitive APIs and telemetry. |
| A legal notice must be withdrawn immediately | Disable stale reuse for that content, issue targeted invalidation, verify both providers and use a safe fallback if removal lags. |
| Residency header is spoofed | Ignore untrusted routing assertions; use validated identity plus authoritative account mapping and independent service enforcement. |
| AI forecasts an overload in 30 minutes | Present evidence to SRE for an approved prescale or policy change; deterministic guards continue regardless of model output. |

## Adversarial Architecture Review

| Challenge | Outcome retained |
|---|---|
| “The CDN supports millions of RPS, so 500k is safe.” | Reject. Origin demand depends on misses, uncached work and amplification; 150k against 15k is still 10× overload. |
| “Raise TTL everywhere until hit ratio reaches 97%.” | Reject. Content freshness and privacy constrain reuse. 97% also leaves no reserve in the simplified arithmetic. |
| “We can serve a cached balance during an outage.” | Reject. A stale balance is not an authoritative banking response; the API is non-cacheable in this design. |
| “Two CDNs guarantee independent availability.” | Reject. A shared origin, DNS provider, database or bad common configuration can fail both. |
| “Every CDN must have its own origin.” | Refine. Independence and capacity are mandatory; one-to-one mapping is optional. |
| “TLS to the origin means the edge never sees data.” | Reject. Edge termination followed by re-encryption still exposes plaintext at the edge. |
| “Passthrough gives us the same WAF and caching.” | Reject for encrypted HTTP at that intermediary. Validate the selected L4/L7 service and termination boundary. |
| “A regional hostname or IP geolocation proves residency.” | Reject. Trusted account policy and enforcement throughout the data path are required. |
| “Failover is just changing DNS.” | Reject. Client convergence, TLS, cold caches, origin capacity and useful-service verification belong in the recovery measurement. |
| “AI should move traffic automatically to save time.” | Reject autonomous model-driven changes. Human review governs recommendations; preapproved deterministic emergency controls act immediately. |
| “The campaign deadline makes 12 minutes acceptable.” | Reject under the five-minute requirement. Escalate the failed gate, remediate and retest. |

## AI Intersection

The useful AI capability is **asynchronous traffic prediction**. It analyzes minimized traffic trends, cache misses, regional demand and origin utilization, then warns that a campaign may exceed safe origin capacity in 30 minutes. The prediction gives operators time to prepare; it does not grant authority to process a banking request or change its region.

![Asynchronous forecasting with human approval and deterministic safeguards](assets/ai-governance.svg)

[Open the scalable AI governance diagram](assets/ai-governance.svg)

The advisory path is **telemetry → forecast → recommendation with evidence → human review → controlled change pipeline**. A recommendation should identify the forecast horizon, uncertainty, expected miss load, affected region, policy version, proposed action and rollback condition. Reject stale or incompatible recommendations.

SRE may approve prescaling, a bounded traffic-weight change or an allowed public-content degradation. The change pipeline checks residency eligibility, origin budgets, configuration parity and approval scope before canarying the change. Record the model/version, input window, recommendation, approver, applied configuration and outcome.

**Human approval does not sit in front of emergency protection.** Preapproved deterministic rate limits, concurrency caps, circuit breakers, regional restrictions and load shedding act immediately when their defined conditions occur. Humans approve policy changes; a model prediction does not suspend established guards.

Evaluate forecasts through historical replay, shadow mode and measured lead time, false alarms and missed overloads before relying on them operationally. Use aggregate or redacted inputs with approved processing locations. If the model is unavailable, inaccurate or compromised, existing CDN policies and banking services continue without waiting for it. No AI call belongs in the synchronous banking path.

## TPM Delivery

**Technical Program Management (TPM)** organizes delivery around evidence of safe behavior across teams. **Site Reliability Engineering (SRE)** owns operational reliability with the service teams. Completing CDN configuration is only one dependency; origins, identity, privacy, capacity and operations must work together under failure.

| Outcome-based workstream | Completion evidence |
|---|---|
| Cross-regional CDN | Provider configuration, routing eligibility, cache behavior, certificates, purge parity and tested provider failure. |
| Cache engineering | Content classification, key tests, TTL/validator policy, publication ordering, invalidation and stale boundaries. |
| GLB / network | Health semantics, traffic steering, DNS convergence, origin restrictions and controlled failback. |
| Origin / replica | Independent failure-domain analysis, measured capacity, prescaling, downstream limits and survivor readiness. |
| Gateway / banking services | Trusted identity-region mapping, service authorization, cache bypass and negative residency tests. |
| Performance engineering | Peak and low-hit tests, cold-cache failover, p95/p99 latency, global admission and banking reserve. |
| Integrated testing | End-to-end public and banking flows, security failures, configuration drift, deployment and recovery scenarios. |
| Information Security | TLS, WAF, DDoS, origin trust, threat model, control-plane permissions and incident containment. |
| Privacy / Compliance / Data Governance | Explicit approval of Canadian and UK processing/storage boundaries, telemetry, backups, support access and transfer policy. |
| Operational readiness / SRE | Dashboards, alerts, runbooks, on-call, vendor escalation, recovery evidence and handover. |

### Sequence and critical path

**Content and residency classification → routing/TLS/cache contracts → origin and regional enforcement → cross-provider integration → peak and cold-failover evidence → five-minute recovery evidence → security/privacy/operations sign-offs → controlled canary.**

CDN, origin, gateway and observability teams can build in parallel once the contracts are stable. The integrated failure exercise drives readiness. Privacy review starts before choosing TLS termination and logging locations, because those decisions can invalidate the topology rather than merely add a late checklist item.

### Milestones and evidence

1. **Architecture baseline:** Approved public/sensitive split, data boundaries, recovery target, origin budgets and decision records.
2. **Integration-ready components:** Provider parity, certificates, authenticated origins, regional enforcement and observable request paths.
3. **Correctness and security:** Cache-key isolation, no sensitive caching, direct-origin denial, wrong-region rejection and controlled publication.
4. **Capacity and resilience:** Campaign arithmetic validated with representative tests, cold-cache origin protection, provider/origin failures and failback.
5. **Operational acceptance:** Recovery within five minutes, actionable alerts, runbooks, on-call coverage and tested rollback.
6. **Go/No-Go:** Dated evidence and accountable sign-off from workstream owners, including explicit privacy and compliance acceptance.

Each milestone has entry and exit criteria, an owner, a due date and linked evidence. Track dependencies and unresolved decisions in the **Risks, Assumptions, Issues and Dependencies (RAID)** register. Verbal confidence or a green status slide cannot substitute for a failed recovery test.

## Program Risks

| Risk / owner function | Mitigation → trigger → contingency |
|---|---|
| Campaign miss load exceeds origin capacity / Performance + SRE | Prescale, cap aggregate demand and test low-hit scenarios → safe envelope threatened → shed optional work, activate approved fallback and pause campaign expansion. |
| CDN diversity hides a common origin dependency / Architecture + Infrastructure | Map failure domains and test dependency loss → both providers fail together → repair origin resilience before claiming continuity. |
| Residency policy is incomplete / Privacy + Gateway teams | Approve end-to-end data flows and negative tests → unapproved processing or uncertain routing → stop affected rollout and correct controls. |
| Multi-CDN policy drift / CDN + Security | Versioned configuration, parity tests and audit → inconsistent cache/WAF/certificate behavior → hold traffic shift or revert affected policy. |
| Sensitive response cached / Security + Application owners | Explicit bypass and cross-user tests → any sensitive cache hit → contain, purge, investigate and block expansion. |
| Recovery remains above five minutes / Network + SRE | Phase timings, automated eligible failover and repeated rehearsal → 12-minute test → NO-GO, remediation and sponsor escalation. |
| Secondary capacity or commercial limits unproven / Infrastructure + Vendor management | Validate quotas, surge commitments and support paths → cold failover saturates → provision capacity or revise release scope through governance. |
| AI produces unsafe or stale advice / SRE + Model owner | Shadow evaluation, human review and deterministic validators → policy-conflicting proposal → reject recommendation; preserve existing controls. |

Residual risks require an accountable owner and explicit acceptance. Mandatory security, residency and recovery gaps remain release blockers under the agreed requirements.

## TPM Constraint Mutations

| Constraint | Delivery response |
|---|---|
| Functional testing passes, launch date is fixed, failover takes 12 minutes | **NO-GO.** Escalate the seven-minute gap, identify the slow phases, assign remediation owners and dates, retest and obtain renewed sign-off. |
| Marketing increases campaign reach after capacity sign-off | Reopen the demand model and low-hit tests; update prescaling and fallback plans before increasing traffic. |
| Second CDN is contracted but origin resilience is unfinished | Do not count procurement as resilience delivery. Close the shared failure-domain gap and prove survivor capacity. |
| Provider cannot meet required TLS processing geography | Rework the sensitive path using approved regional termination or verified passthrough; do not equate encryption with compliant processing. |
| Budget cuts remove standby capacity | Quantify the impact on recovery and admission. Escalate an explicit architecture/scope decision; do not retain an unsupported five-minute claim. |
| AI rollout is delayed | Core CDN and banking readiness can proceed if deterministic controls and all mandatory gates pass. Advisory AI is not on the critical request path. |
| Minor documentation remains after all required safety and operational gates pass | Consider conditional acceptance only with a named owner/date and no impairment to incident response. |

A formal change to the recovery requirement would require accountable business and risk governance and revised acceptance criteria. It would not retroactively make a 12-minute result satisfy a five-minute target.

## Production Readiness

Required gates, **not claims of passed production tests**. Every gate needs agreed criteria, evidence, an owner and sign-off.

| Gate | Evidence before Go |
|---|---|
| Functional / content | Public and sensitive route classification, locale/market behavior, redirects, errors and approved fallback content. |
| Cache correctness | Key equivalence tests, query/header/cookie variation, poisoning/deception resistance, TTL/ETag behavior and no sensitive API caching. |
| Publication / invalidation | Asset-first deployment, immutable version integrity, targeted multi-CDN purge, convergence measurement and prior-version rollback. |
| Capacity | 100k/90% and 500k/70% scenarios, uncached API mix, cold caches, refresh/retry overhead, safe global admission and protected banking capacity. |
| Provider / origin resilience | CDN loss, independent origin loss, shared-dependency failures, healthy survivor capacity and controlled failback. |
| Regional enforcement | Trusted account mapping, wrong-region rejection at gateway and service, approved TLS processing, data stores, logs, backups and access. |
| Recovery | Representative end-to-end failover restores useful service within **five minutes**, with timestamps and policy/capacity evidence. A 12-minute result fails. |
| Security | WAF/DDoS behavior, direct-origin denial, origin authentication, certificate rotation, least privilege and control-plane audit. |
| Operations / release | User-centered SLOs, alerts, runbooks, on-call and vendor escalation, approved change/rollback plans and formal sign-offs. |
| AI, if enabled | Asynchronous-only integration, approved data inputs, evaluated forecasts, human-approved changes, deterministic guards and model-outage tests. |

No unresolved sensitive cache exposure, unapproved regional processing, unsafe origin overload path or failed mandatory recovery gate. Functional completion is not production readiness.

## Rollout & Rollback

Start with configuration comparison and synthetic traffic, then a small public-content cohort, a limited geographic/provider canary and progressive expansion. Introduce sensitive-path changes only after their independent security and residency gates pass. Freeze unrelated changes during the decisive recovery exercise so evidence can be attributed to a known configuration.

Before each ramp, verify provider certificates, origin trust, policy parity, object versions, safe origin budgets and rollback readiness. Watch customer latency and success, miss RPS, origin concurrency, WAF false positives, regional rejects and configuration drift. Stop on a sensitive cache hit, unauthorized processing, origin-envelope breach or sustained service regression.

Rollback restores a known-good configuration or traffic split only when the destination remains healthy, compliant and sufficiently provisioned. Keep older versioned assets available; restore the corresponding HTML references. A DNS rollback also has convergence delay. Do not perform a global purge as a reflex, disable certificate verification or route banking data into an unapproved country.

If rollback would reintroduce a security defect or exceed survivor capacity, use controlled degradation or failure instead. Recovery completes only after useful service and the required boundaries are verified, then incident evidence and runbooks are updated.

## Concepts Learned

| Concept | Working meaning / correction |
|---|---|
| CDN / PoP / origin | Delivery network / edge location / authoritative refill source. |
| DNS / Anycast / GLB | Name resolution / network route selection / policy-based destination choice. |
| Cache key | A claim that requests can safely share the same representation. |
| TTL / ETag | Freshness lifetime / representation validator; validation still consumes upstream work. |
| `no-cache` / `no-store` | Validate before reuse / do not store. Sensitive banking APIs use bypass plus no-store here. |
| Purge / versioned URL | Invalidate a copy / publish a new immutable identity; neither recalls browser copies automatically. |
| Stale allowance | Bounded public-content reuse under approved conditions, not a banking correctness mechanism. |
| Shield / coalescing | Consolidate upstream cache work / combine concurrent requests for the same representation. |
| Request / byte hit ratio | Share of requests served from cache / share of bytes served from cache; different capacity signals. |
| Multi-CDN / origin resilience | Provider diversity / survival of upstream failure; both need independent analysis. |
| TLS termination / passthrough | Decrypt at an endpoint / forward encrypted traffic using supported service capabilities. |
| Residency / geolocation | Approved data-processing boundary / observed client location; they are not equivalent. |
| Prescaling / autoscaling | Prepare known demand in advance / react to observed demand with a provisioning delay. |
| Recovery target / measured recovery | Required useful-service restoration time / tested end-to-end result. Five minutes versus 12 is a failed gate. |

## Mental Models

- **The edge accelerates public content; banking services own sensitive decisions.**
- Classify content before choosing a cache key or TTL.
- Every cache key is a promise about who may share a response.
- A hit saves origin work; a miss, refresh or retry spends origin capacity.
- Five times the traffic and three times the miss fraction creates 15 times the origin demand.
- More CDN capacity does not create more origin capacity.
- Two providers do not remove one shared failure domain.
- Dedicated origins are optional; independent failure domains and survivor capacity are required.
- Encryption on two links still permits plaintext at the termination point between them.
- Client geography is a routing hint; trusted identity and service policy enforce residency.
- Public content may cross permitted regions; sensitive banking requests may not inherit that permission.
- A recovery target is measured at useful service, not at a DNS update.
- AI advises asynchronously; humans approve changes; deterministic guards protect immediately.

---

**Design basis:** The completed 11-stage Topic 10 drill, culminating in Question 34 and the final ARB refinement. This chapter consolidates the supplied coverage of fundamentals, networking/TLS, cache semantics, security, capacity, emergency strategies, resilience, residency, monitoring/readiness, constraint and adversarial reviews, AI and TPM governance. Explanatory examples are illustrative, not reconstructed dialogue or production test results. Protocol and privacy references are linked at the relevant claims. Previous: [Topic 9 — Distributed Scheduler](../09-distributed-scheduler/README.md). Next: Topic 11 — Distributed File / Object Storage (planned).
