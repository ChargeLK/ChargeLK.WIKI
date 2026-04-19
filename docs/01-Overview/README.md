# System Overview

## Introduction

ChargeLK is an EV (Electric Vehicle) charging management solution designed to manage charging infrastructure, user accounts, wallets, pricing, and real-time communication with charging stations using the **OCPP 1.6 JSON** standard over WebSocket.

The platform is implemented as two independent .NET 9 microservices plus a React admin portal:

- **ChargeLK.OCPPServer** — OCPP 1.6 server (WebSocket) for charge points, plus REST APIs for charger, tariff, tag, review, and transaction management.
- **ChargeLK.UM** — User Management service: identity, RBAC/permissions, wallet, documents, mobile app APIs (including Google Sign-In), and system settings.
- **ChargeLK.AdminUI** — React 18 + TypeScript + Vite + Material UI portal backed by RTK Query.

## Business Context

The ChargeLK platform supports:

- **Charge Point Operators (CSOs)** — onboard chargers, set tariffs, monitor operations, schedule maintenance, moderate reviews.
- **EV Drivers (Mobile App)** — sign up / log in (password or Google), receive a virtual OCPP tag, top up a wallet in LKR, authorise at the charge point, track sessions and history, leave reviews.
- **System Administrators (Admin Portal)** — manage users, roles, permissions, OCPP tags, charge points, transactions, and global system settings.

## Key Features

### OCPP Server (ChargeLK.OCPPServer)
- WebSocket endpoint `/ocpp/{chargeBoxId}` upgrading to the `ocpp1.6` subprotocol.
- OCPP 1.6 actions handled today: `BootNotification`, `Heartbeat`, `StatusNotification`, `Authorize`, `StartTransaction`, `StopTransaction`, `MeterValues`, `DataTransfer`.
- Server-initiated operations: `RemoteStartTransaction`, `RemoteStopTransaction`, `UnlockConnector`, `Reset`, `ClearCache`.
- **Tariffs** — default and per-charge-point / per-connector pricing with peak windows, session fees, minimum/maximum caps.
- **OCPP Tags** — physical and virtual tags, status lifecycle (Active / Suspended / Blocked / Deactivated), assignment to users, full audit log, bulk import.
- **Charger Maintenance** — scheduled/logged maintenance records per charge point.
- **Charger Reviews** — rating + message reviews keyed to a charge point.
- **Mobile bridge endpoints** (under `/api/ocpp/*`) for the UM service: create virtual tag, user summary, active session, user transactions.

### User Management (ChargeLK.UM)
- ASP.NET Core Identity on a `long` key with a custom `ApplicationUser` (FirstName / LastName / UserId GUID / Docket).
- JWT bearer + refresh token flow; cross-service validation via a shared JWT scheme.
- **Permission-based authorisation** (`resource:action`) with a custom `PermissionPolicyProvider` — policies are materialised on demand.
- **Roles**: `SystemAdmin`, `Admin`, `Operator`, `Support`, `User` (seeded, plus permissions seed).
- **Mobile auth**: email/password login, signup, forgot/reset password, Google ID-token sign-in.
- **Wallet**: per-user `UserWallet` in LKR, `WalletTransaction` ledger, top-up stub (payment gateway TODO), admin balance adjustments, balance checks.
- **Documents + Dockets**: every user gets a `Docket` (document bundle); documents live in Azure Blob Storage with generated thumbnails (SixLabors.ImageSharp) and a `DocumentCategory` taxonomy.
- **System Settings**: grouped settings (General / Notifications / Security / Appearance) editable via `/api/um/settings`.
- **Domain events**: `UserCreatedDomainEvent` triggers virtual OCPP tag generation (via the OCPP service) and `CreateBlobContainerEvent` for document storage bootstrap.
- **Messaging**: `AzureServiceBusPublisher` / `AzureServiceBusSubscriber` are wired up in Infrastructure for event integration.

### Admin Portal (ChargeLK.AdminUI)
- React 18, TypeScript, Vite, Material UI v6 (including `x-data-grid`, `x-date-pickers`), Recharts, framer-motion, notistack.
- **Redux Toolkit + RTK Query** with redux-persist; central `baseApi` with automatic JWT refresh, mutex-guarded against concurrent 401s, and unwrapping of the backend `Result<T>` envelope.
- Pages: `Dashboard`, `ChargePoints`, `ChargePointDetail`, `OcppTags`, `Transactions`, `Users`, `Settings`, `Login`.
- Feature dialogs: assign tag ↔ user, bulk tag import, tag assignment history, user roles, user permissions.
- QR code generation for charge points (`qrcode.react`, `html2canvas`, `jspdf`).

## Stakeholders

- **EV Drivers** — mobile app clients consuming `/api/um/auth/mobile/*` and `/api/um/mobile/*`.
- **CSO Administrators / Operators** — admin portal users, authenticated via `/api/auth/login`.
- **System Administrators** — `SystemAdmin` / `Admin` roles; full access to users, roles, permissions, settings.
- **Charge Points** — OCPP 1.6 JSON clients connecting over WebSocket.

## Non-Functional Notes

- **Databases**: PostgreSQL per service (EF Core 9 for UM, EF Core 8 for OCPP). No MongoDB in the current build.
- **File storage**: Azure Blob Storage (`Azure.Storage.Blobs`) for user documents.
- **Deployment**:
  - OCPP Server: built and shipped via `.github/workflows/deploy.yml`, run under `chargelk-ocpp-imesh.service` systemd unit. Migrations applied at deploy time using an `ef migrations bundle`.
  - UM: containerised (Dockerfile at the solution root).
  - Admin UI: containerised (Dockerfile + nginx).
- **Secrets**: `DATABASE_CONNECTION_STRING`, `MediatR__LicenseKey`, `Jwt__SecretKey` injected at deploy time; placeholders in `appsettings.json`.

## Related Documentation

- [System Architecture](../02-Architecture/README.md)
- [Components Documentation](../03-Components/README.md)
- [API Reference](../04-API-Reference/README.md)
- [Data Architecture](../05-Data-Architecture/README.md)
- [Development Guide](../07-Development/README.md)
- [Security](../08-Security/README.md)
- [Operations & Deployment](../09-Operations/README.md)
