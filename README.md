# ChargeLK System Documentation

Welcome to the ChargeLK system documentation repository. This wiki is the reference knowledge base for the ChargeLK EV Charging Solution.

ChargeLK is composed of three deployable components living in sibling repositories:

- **ChargeLK.OCPPServer** — .NET 9 OCPP 1.6 JSON WebSocket server + REST admin/management APIs.
- **ChargeLK.UM** — .NET 9 User Management microservice (Identity, RBAC, wallet, documents, mobile auth).
- **ChargeLK.AdminUI** — React 18 + TypeScript + Vite + MUI v6 admin portal (Redux Toolkit + RTK Query).

An EV-driver mobile app consumes UM's `/api/um/auth/mobile/*` and `/api/um/mobile/*` endpoints; it is not in this repository.

## Documentation Index

### 1. [System Overview](./docs/01-Overview/README.md)
Introduction, business context, stakeholders, and per-component key features.

### 2. [System Architecture](./docs/02-Architecture/README.md)
Clean Architecture layering per service, C4 context/container diagrams, technology stack, OCPP WebSocket + mobile session flows.

### 3. [Components Documentation](./docs/03-Components/README.md)
Detailed breakdown of the OCPP server, UM service, and Admin UI — entities, handlers, infrastructure services, controllers, and API contracts.

### 4. [API Reference](./docs/04-API-Reference/README.md)
OCPP WebSocket protocol (inbound + server-initiated actions), OCPP REST endpoints, UM REST endpoints (admin auth, mobile auth, wallet, documents, settings), and the `Result<T>` envelope.

### 5. [Data Architecture](./docs/05-Data-Architecture/README.md)
PostgreSQL schemas per service (`chargelk_ocpp`, `public`), entity relationships, domain events, cross-service data bridges, Azure Blob layout.

### 6. [Infrastructure](./docs/06-Infrastructure/README.md)
Runtime/deployment topology. OCPP server runs bare-metal under `systemd` (`chargelk-ocpp-imesh.service`). UM and Admin UI are containerised.

### 7. [Development Guide](./docs/07-Development/README.md)
- [User Secrets Setup Guide](./docs/07-Development/USER-SECRETS-SETUP.md) — .NET user secrets
- [Admin Portal Quick Start](./docs/07-Development/ADMIN-PORTAL-QUICK-START.md) — React app setup
- [RTK Implementation Plan](./docs/07-Development/RTK-IMPLEMENTATION-PLAN.md) — RTK Query design notes
- Local setup, coding standards, testing

### 8. [Security](./docs/08-Security/README.md)
- [Authentication & Authorization Guide](./docs/08-Security/AUTHENTICATION-AUTHORIZATION.md)
- [Authentication Flow Diagrams](./docs/08-Security/AUTHENTICATION-FLOW-DIAGRAMS.md)
- [Microservices Authentication](./docs/08-Security/MICROSERVICES-AUTHENTICATION-IMPLEMENTATION.md)
- [Authentication Testing Guide](./docs/08-Security/AUTHENTICATION-TESTING-GUIDE.md)
- [Permission Practical Examples](./docs/08-Security/PERMISSION-PRACTICAL-EXAMPLES.md)
- [Quick Reference - Auth](./docs/08-Security/QUICK-REFERENCE-AUTH.md)

### 9. [Operations & Deployment](./docs/09-Operations/README.md)
- [Deployment Guide](./docs/09-Operations/DEPLOYMENT-GUIDE.md)
- [Deployment Quickstart](./docs/09-Operations/DEPLOYMENT-QUICKSTART.md)
- [Deployment Cheatsheet](./docs/09-Operations/DEPLOYMENT-CHEATSHEET.md)

## Quick Start Guides

### For developers
1. [Development Guide](./docs/07-Development/README.md) — local environment setup
2. [User Secrets Setup](./docs/07-Development/USER-SECRETS-SETUP.md) — dev secrets
3. [Admin Portal Quick Start](./docs/07-Development/ADMIN-PORTAL-QUICK-START.md) — Vite dev server
4. [Authentication Testing](./docs/08-Security/AUTHENTICATION-TESTING-GUIDE.md)

### For DevOps
1. [Deployment Quickstart](./docs/09-Operations/DEPLOYMENT-QUICKSTART.md)
2. [Deployment Cheatsheet](./docs/09-Operations/DEPLOYMENT-CHEATSHEET.md)

### For understanding the system
1. [System Overview](./docs/01-Overview/README.md)
2. [System Architecture](./docs/02-Architecture/README.md)
3. [API Reference](./docs/04-API-Reference/README.md)

## Documentation Standards

- Diagrams use Mermaid so they render in GitHub/GitLab and stay versionable.
- Architecture documentation follows C4 model principles (Context + Container diagrams).
- API documentation reflects the actual route templates on each controller — update the wiki when a route changes.
- Code examples use the same languages as the underlying services (C# 12 for backend, TypeScript 5 for the portal).

## Directory Layout

```
ChargeLK.WIKI/
├── README.md                     # this index
├── docs/
│   ├── 01-Overview/              # System introduction
│   ├── 02-Architecture/          # Layering, diagrams, tech stack
│   ├── 03-Components/            # Per-component deep dive
│   ├── 04-API-Reference/         # REST + OCPP WS reference
│   ├── 05-Data-Architecture/     # PostgreSQL schemas, domain events
│   ├── 06-Infrastructure/        # Runtime topology
│   ├── 07-Development/           # Developer setup
│   ├── 08-Security/              # Authentication & authorization
│   └── 09-Operations/            # Deployment & operations
└── Resource Documents/           # Supporting assets
```

## Last Updated

**Version:** 2.0.0
**Maintainers:** ChargeLK Development Team

## Recent Changes

- Rewrote System Overview to reflect the current three-component layout (OCPP .NET 9 + UM .NET 9 + React admin UI) and removed stale references to technologies not in use.
- Rewrote System Architecture with accurate Clean Architecture layering per service (noting the `Extentions/` vs `Extensions/`, `Middlewares/` vs `Middleware/`, and `Persistence/` vs `Database/` differences between the two services) and updated the C4 and sequence diagrams.
- Added a new Components Documentation chapter covering entities, handlers, infrastructure services, and controllers for all three components.
- Added a new API Reference chapter enumerating every REST route from source and the OCPP WebSocket protocol, including the `Result<T>` envelope used by all responses.
- Added a new Data Architecture chapter with per-service PostgreSQL schema diagrams, cross-service identifier bridges, and the `MongoDbContext`-exists-but-unused clarification.

## Contributing to Documentation

1. Follow the existing folder structure.
2. Use Markdown; use Mermaid for diagrams.
3. When you change a route, entity, or domain event, update the relevant doc in the same PR.
4. Keep documentation concise and verifiable — when in doubt, cite the source file/route.
