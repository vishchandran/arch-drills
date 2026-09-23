# Architecture Drills

A code-free System Architecture drill book covering architecture reasoning, distributed systems, trade-offs, reliability, security, and delivery.

## Purpose

Capture architecture reasoning, trade-offs, failure analysis, and delivery decisions from completed drills. Each topic is an independent chapter grounded in the actual exercise, including useful mistakes and rejected approaches.

## Method

Build from first principles, challenge assumptions, mutate constraints, review failure behavior, evaluate the AI intersection, and translate the architecture into a delivery plan. Competency determines pace.

## Reading order and completed topics

Follow the numbered folders in curriculum order. A topic is added here after its drill and final README are complete.

| Topic | Status |
|---|---|
| [1. Architecture Foundations](../01-architecture-foundations/README.md) | Complete |
| [2. URL Shortener](../02-url-shortener/README.md) | Complete |
| [3. Rate Limiter](../03-rate-limiter/README.md) | Complete |
| [4. API Gateway + Global Load Balancing](../04-api-gateway-global-load-balancing/README.md) | Complete |
| [5. Service Mesh + Service Discovery](../05-service-mesh-service-discovery/README.md) | Complete |
| [6. Distributed Cache](../06-distributed-cache/README.md) | Complete |

## Chapter structure

Completed chapters cover the applicable sections:

- Problem & Requirements
- Final Architecture with responsive SVG or compact Mermaid diagrams
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
- Production Readiness
- Rollout & Rollback
- Concepts Learned
- Mental Models

## AI curriculum

Each architecture topic evaluates whether AI belongs in the system. The dedicated AI track will cover foundations, inference economics, context engineering, retrieval, RAG, agents, MCP, memory, model strategy, evaluations, observability, security, and enterprise AI platforms. Its chapters will be added as the track is organized.

## Content standards

Keep chapters code-free, architecture-focused, and understandable without the original conversation. Explain why decisions were made, expose trade-offs and failure behavior, expand abbreviations, and preserve useful mistakes. Avoid documenting uncompleted drills as established designs.
