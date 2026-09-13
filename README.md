# Arch Drills

A code-free System Architecture drill book covering architecture reasoning, distributed systems, trade-offs, reliability, security, and delivery.

## Purpose

Capture architecture reasoning, trade-offs, failure analysis, and delivery decisions from completed drills. Each topic is an independent chapter grounded in the actual exercise, including useful mistakes and rejected approaches.

## Method

Build from first principles, challenge assumptions, mutate constraints, review failure behavior, evaluate the AI intersection, and translate the architecture into a delivery plan. Competency determines pace.

## Reading order and topic index

Follow the numbered folders in curriculum order. Each folder contains its own README.

| Topic | Status |
|---|---|
| [1. Architecture Foundations](01-architecture-foundations/README.md) | Complete |
| [2. URL Shortener](02-url-shortener/README.md) | Complete |
| [3. Rate Limiter](03-rate-limiter/README.md) | Complete |
| [4. API Gateway + Global Load Balancing](04-api-gateway-global-load-balancing/README.md) | Planned |
| [5. Service Mesh + Service Discovery](05-service-mesh-service-discovery/README.md) | Planned |
| [6. Distributed Cache](06-distributed-cache/README.md) | Planned |
| [7. Kafka + Event-Driven Architecture](07-kafka-event-driven-architecture/README.md) | Planned |
| [8. Notification System](08-notification-system/README.md) | Planned |
| [9. Distributed Scheduler](09-distributed-scheduler/README.md) | Planned |
| [10. CDN + Edge Architecture](10-cdn-edge-architecture/README.md) | Planned |
| [11. Distributed File/Object Storage](11-distributed-file-object-storage/README.md) | Planned |
| [12. Search + Autocomplete](12-search-autocomplete/README.md) | Planned |
| [13. News Feed / Fan-out Systems](13-news-feed-fan-out/README.md) | Planned |
| [14. Distributed Transactions — Saga, Outbox, Idempotency](14-distributed-transactions/README.md) | Planned |
| [15. Consistency Models + Quorums + Consensus](15-consistency-quorums-consensus/README.md) | Planned |
| [16. Multi-Region Architecture](16-multi-region-architecture/README.md) | Planned |
| [17. Fraud Detection Platform](17-fraud-detection-platform/README.md) | Planned |
| [18. Payment Gateway](18-payment-gateway/README.md) | Planned |
| [19. Payment Switch + ATM/POS Routing](19-payment-switch-atm-pos-routing/README.md) | Planned |
| [20. Settlement + Reconciliation](20-settlement-reconciliation/README.md) | Planned |
| [21. Legacy Modernization + Strangler Architecture](21-legacy-modernization/README.md) | Planned |
| [22. Platform Engineering / Internal Developer Platform](22-platform-engineering/README.md) | Planned |
| [23. Enterprise Payments Capstone](23-enterprise-payments-capstone/README.md) | Planned |

## Chapter structure

Completed chapters cover the applicable sections:

- Problem & Requirements
- Final Architecture with responsive SVG diagrams
- Component Responsibilities
- Key Architecture Decisions
- Data & Consistency
- Scaling
- Reliability & Failure Modes
- Security
- Observability
- Constraint Mutations
- Adversarial Architecture Review
- AI Intersection
- TPM Delivery
- Program Risks
- TPM Constraint Mutations
- Concepts Learned
- Mental Models

## AI curriculum

Each architecture topic evaluates whether AI belongs in the system. The dedicated AI track will cover foundations, inference economics, context engineering, retrieval, RAG, agents, MCP, memory, model strategy, evaluations, observability, security, and enterprise AI platforms. Its chapters will be added as the track is organized.

## Progress

- Architecture drills completed: 3 of 23.
- Final chapter READMEs completed: 3 of 23.
- Current chapter: Topic 4 — API Gateway + Global Load Balancing.

## Content standards

Keep chapters code-free, architecture-focused, and understandable without the original conversation. Explain why decisions were made, expose trade-offs and failure behavior, expand abbreviations, and preserve useful mistakes. Avoid documenting uncompleted drills as established designs.
