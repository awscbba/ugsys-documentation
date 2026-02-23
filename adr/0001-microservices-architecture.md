# ADR-0001: Microservices Architecture for ugsys Platform

**Date**: 2026-02-23
**Status**: Accepted
**Deciders**: AWSUGCBBA Platform Team

## Context

The existing `registry-api` is a monolith handling identity, projects, messaging, and admin concerns in a single FastAPI application backed by a single DynamoDB table. As the platform grows, this creates coupling, deployment risk, and scaling constraints.

## Decision

Decompose the monolith into 5 independent microservices:

| Service | Repo | Responsibility |
|---|---|---|
| Identity Manager | `ugsys-identity-manager` | Auth, JWT, user management |
| Projects Registry | `ugsys-projects-registry` | Projects, forms, submissions |
| Admin Panel | `ugsys-admin-panel` | Admin UI and operations |
| Mass Messaging | `ugsys-mass-messaging` | Email/SMS campaigns |
| Omnichannel Service | `ugsys-omnichannel-service` | WhatsApp, Telegram, social |

Plus two supporting repos:

| Repo | Responsibility |
|---|---|
| `ugsys-shared-libs` | Shared Python packages (auth, logging, events, testing) |
| `ugsys-platform-infrastructure` | CDK stacks for shared AWS resources |

## Architecture Principles

- **Hexagonal architecture** (ports & adapters) in every service
- **Serverless-first**: Lambda + API Gateway, no ECS
- **Event-driven**: services communicate via EventBridge, not direct HTTP calls
- **Separate data stores**: each service owns its DynamoDB tables
- **No shared database**: cross-service reads go through APIs or events

## Consequences

- Each service can be deployed, scaled, and versioned independently
- Teams can work in parallel without merge conflicts
- Requires shared libraries for cross-cutting concerns (auth, logging)
- Requires an event bus for async communication
- Initial migration effort from monolith to services

## References

- `MICROSERVICES_DECOUPLING_PROPOSAL_EN.md` — full proposal with cost estimates and migration strategy
