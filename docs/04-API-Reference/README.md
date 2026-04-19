# API Reference

This document enumerates every public surface of the ChargeLK platform:

- **OCPP WebSocket** — charge-point ↔ server (OCPP 1.6 JSON)
- **OCPP Server REST** — admin, mobile bridge, tag/tariff/review/transaction management
- **User Management REST** — admin portal auth + user/role/permission/wallet/document/settings; mobile app auth + summary

All REST responses are wrapped in the backend `Result<T>` envelope (see [Response Envelope](#response-envelope)). All error/success HTTP status semantics are preserved; the envelope sits in the body.

---

## 1. OCPP WebSocket Protocol

### Endpoint

```
ws(s)://<ocpp-server-host>/ocpp/{chargeBoxId}
Sec-WebSocket-Protocol: ocpp1.6
```

- The middleware `OcppWebSocketMiddleware` upgrades connections whose path starts with `/ocpp`. The last path segment is the `chargeBoxId`. `chargeBoxId` may also be supplied via the `chargeBoxId` query parameter.
- The only accepted subprotocol is `ocpp1.6`.
- Each accepted socket is registered with `OcppServerService` so the server can initiate follow-up calls.

### Message envelope (OCPP 1.6 JSON)

```jsonc
// CALL  (client → server or server → client)
[2, "<messageId>", "<Action>", { /* payload */ }]
// CALLRESULT
[3, "<messageId>", { /* payload */ }]
// CALLERROR
[4, "<messageId>", "<errorCode>", "<description>", { /* details */ }]
```

### Inbound actions (charge point → server)

| Action              | Purpose                                                                 |
| ------------------- | ----------------------------------------------------------------------- |
| `BootNotification`  | Register/refresh charge-point metadata; receive heartbeat interval.     |
| `Heartbeat`         | Keep-alive. Server returns current UTC.                                 |
| `StatusNotification`| Connector status + error code reporting.                                |
| `Authorize`         | Validate an `idTag` before starting a transaction.                      |
| `StartTransaction`  | Open a new charging transaction; server issues `transactionId`.         |
| `StopTransaction`   | Close transaction; triggers billing against the applicable `Tariff`.    |
| `MeterValues`       | Periodic energy/phase/temperature readings.                             |
| `DataTransfer`      | Vendor-specific messages.                                               |

### Server-initiated actions (server → charge point)

Triggered by the REST endpoints on `ChargePointsController` listed in §2.

| Action                   | Triggering REST endpoint                                     |
| ------------------------ | ------------------------------------------------------------ |
| `RemoteStartTransaction` | `POST /api/ChargePoints/{chargeBoxId}/remote-start`          |
| `RemoteStopTransaction`  | `POST /api/ChargePoints/{chargeBoxId}/remote-stop`           |
| `UnlockConnector`        | `POST /api/ChargePoints/{chargeBoxId}/unlock-connector`      |
| `Reset`                  | `POST /api/ChargePoints/{chargeBoxId}/reset`                 |
| `ClearCache`             | `POST /api/ChargePoints/{chargeBoxId}/clear-cache`           |

---

## 2. OCPP Server REST API

Base URL: `http://<ocpp-server-host>` (default port `5001` in dev).

`BaseApiController` routes to `api/ocpp/[controller]`, but most feature controllers override with `[Route("api/[controller]")]`. The table below reflects the actual route template.

### Health

| Method | Route                 | Notes                                    |
| ------ | --------------------- | ---------------------------------------- |
| GET    | `/api/health`         | Full report via `HealthChecks.UI.Client` |
| GET    | `/api/health/ping`    | Simple liveness                          |
| GET    | `/api/health/cors-test` | CORS smoke test                        |

### Charge Points — `/api/ChargePoints`

| Method | Route                                    | Body / Query                                                     | Description                                              |
| ------ | ---------------------------------------- | ---------------------------------------------------------------- | -------------------------------------------------------- |
| GET    | `/`                                      | `?status&search&page&pageSize`                                   | Paged list with filtering                                |
| GET    | `/{chargeBoxId}`                         |                                                                  | Details + connector snapshot + last transactions         |
| POST   | `/`                                      | `CreateChargePointCommand`                                       | Register a new charger                                   |
| DELETE | `/{id}`                                  |                                                                  | Soft-delete a charger                                    |
| POST   | `/{chargeBoxId}/remote-start`            | `{ idTag, connectorId }`                                         | Sends OCPP `RemoteStartTransaction`                      |
| POST   | `/{chargeBoxId}/remote-stop`             | `{ transactionId }`                                              | Sends OCPP `RemoteStopTransaction`                       |
| POST   | `/{chargeBoxId}/unlock-connector`        | `{ connectorId }`                                                | Sends OCPP `UnlockConnector`                             |
| POST   | `/{chargeBoxId}/reset`                   | `{ type: "Soft" \| "Hard" }`                                     | Sends OCPP `Reset`                                       |
| POST   | `/{chargeBoxId}/clear-cache`             |                                                                  | Sends OCPP `ClearCache`                                  |

### OCPP Tags — `/api/OcppTags`

| Method | Route                              | Description                                                |
| ------ | ---------------------------------- | ---------------------------------------------------------- |
| GET    | `/`                                | Paged list (`?status&type&search&page&pageSize`)           |
| GET    | `/{id}`                            | Get by id                                                  |
| GET    | `/by-idtag/{idTag}`                | Get by raw `idTag`                                         |
| GET    | `/by-user/{externalUserId}`        | All tags for a UM user                                     |
| POST   | `/`                                | Create a tag (physical/virtual/certificate/fleet/etc.)     |
| PUT    | `/{id}`                            | Update metadata                                            |
| DELETE | `/{id}`                            | Soft-delete                                                |
| POST   | `/{id}/activate`                   | Transition → `Active`                                      |
| POST   | `/{id}/deactivate`                 | Transition → `Deactivated`                                 |
| POST   | `/{id}/suspend`                    | Transition → `Suspended`                                   |
| POST   | `/{id}/block`                      | Transition → `Blocked` (reason required)                   |
| POST   | `/{id}/unblock`                    | Transition back → `Active`                                 |
| POST   | `/{id}/assign`                     | Assign to UM user (`{ externalUserId, userDisplayName }`)  |
| POST   | `/{id}/unassign`                   | Clear user assignment                                      |
| POST   | `/{id}/change-status`              | Arbitrary status change with validated transitions         |
| GET    | `/{id}/audit-log`                  | Paged `OcppTagAuditLog` for the tag                        |
| POST   | `/bulk-import`                     | CSV or structured batch import                             |
| POST   | `/virtual`                         | Create a virtual tag (used by UM on user creation)         |

### Tariffs — `/api/Tariffs`

| Method | Route     | Description                                     |
| ------ | --------- | ----------------------------------------------- |
| GET    | `/`       | List all tariffs                                |
| GET    | `/{id}`   | Get tariff by id                                |
| POST   | `/`       | Create tariff (default, per-CP or per-connector)|
| PUT    | `/{id}`   | Update pricing / peak window / validity / caps  |
| DELETE | `/{id}`   | Delete tariff                                   |

Tariff resolution at StopTransaction: `connector-specific → charge-point-specific → default`, filtered by `IsActive` and current time window.

### Transactions — `/api/Transactions`

| Method | Route                                  | Description                                                       |
| ------ | -------------------------------------- | ----------------------------------------------------------------- |
| GET    | `/`                                    | Paged list (`?chargePointId&idTag&from&to&page&pageSize`)         |
| GET    | `/{id}`                                | Single transaction with `MeterValues`                             |
| GET    | `/active`                              | In-flight transactions (no `StopTimestamp`)                       |
| GET    | `/by-chargepoint/{chargePointId}`      | Transactions for a charge point                                   |
| GET    | `/last/by-ocpptag/{ocppTag}`           | Most recent transaction for a tag                                 |
| GET    | `/by-ocpptag/{ocppTag}`                | All transactions for a tag                                        |

### Charger Maintenance — `/api/ChargerMaintenance`

| Method | Route     | Description                                 |
| ------ | --------- | ------------------------------------------- |
| GET    | `/`       | List maintenance entries (optional filters) |
| POST   | `/`       | Log a maintenance action                    |
| PUT    | `id`      | Update entry _(route template is `"id"`, not `"{id}"` — update the controller if you change this)_ |
| DELETE | `/{id}`   | Remove a maintenance entry                  |

### Charger Reviews — `/api/ChargerReviews`

| Method | Route                                | Description                               |
| ------ | ------------------------------------ | ----------------------------------------- |
| GET    | `/by-chargepoint/{chargePointId}`    | Paged reviews for a charge point          |
| POST   | `/`                                  | Create a review (`rating 1–5`, message)   |
| PUT    | `/{id}`                              | Update review text/rating                 |
| DELETE | `/{id}`                              | Delete review                             |

### Mobile Bridge — `/api/ocpp/*` and `/api/ocpp/mobile/*`

Intended for the UM microservice to call on behalf of mobile clients.

| Method | Route                                            | Description                                          |
| ------ | ------------------------------------------------ | ---------------------------------------------------- |
| POST   | `/api/ocpp/tags/virtual`                         | Create a virtual tag for a new mobile user          |
| GET    | `/api/ocpp/mobile/user/{userId}/summary`         | Totals: energy, sessions, lifetime spend            |
| GET    | `/api/ocpp/mobile/user/{userId}/transactions`    | Paged transaction history                           |
| GET    | `/api/ocpp/mobile/user/{userId}/active-session`  | Current in-progress session, if any                 |

---

## 3. User Management REST API

Base URL: `http://<um-host>` (default port `5002` in dev).

`BaseApiController` routes to `api/um/[controller]`. Several controllers define custom routes — shown below.

### Admin Portal Auth — `/api/auth`

From `ApplicationUserController`.

| Method | Route                                              | Description                                  |
| ------ | -------------------------------------------------- | -------------------------------------------- |
| POST   | `/register`                                        | Create a new staff user                      |
| POST   | `/login`                                           | Email/password login → `{ accessToken, refreshToken }` |
| POST   | `/refresh`                                         | Exchange `{ accessToken, refreshToken }` for a new pair |
| POST   | `/logout`                                          | Revoke current refresh token                 |
| POST   | `/role`                                            | Create a role                                |
| POST   | `/role-claim`                                      | Attach a claim to a role                     |
| POST   | `/user-role`                                       | Assign a role to a user                      |
| POST   | `/user-claim`                                      | Attach a claim to a user                     |
| GET    | `/`                                                | Paged user list                              |
| GET    | `/{id:long}`                                       | User detail                                  |
| GET    | `/roles`                                           | All roles                                    |
| GET    | `/permissions`                                     | All permissions                              |
| GET    | `/{id:long}/roles`                                 | Roles for a user                             |
| GET    | `/{id:long}/permissions`                           | Permissions for a user (via roles)           |
| DELETE | `/user-role/{userId:long}/{roleId:long}`           | Revoke role from user                        |
| DELETE | `/user-claim/{userId:long}/{claimType}/{claimValue}`| Revoke claim from user                      |

### Mobile Auth — `/api/um/auth/mobile`

From `MobileAuthController`. Used by the EV-driver mobile app.

| Method | Route              | Body                                            | Description                                 |
| ------ | ------------------ | ----------------------------------------------- | ------------------------------------------- |
| POST   | `/login`           | `{ email, password }`                           | Mobile login                                |
| POST   | `/signup`          | `{ firstName, lastName, email, password }`      | Creates user + triggers `UserCreatedDomainEvent` (virtual tag + docket) |
| POST   | `/google`          | `{ idToken }`                                   | Google Sign-In via `Google.Apis.Auth`       |
| POST   | `/forgot-password` | `{ email }`                                     | Starts reset flow                           |
| POST   | `/reset-password`  | `{ email, token, newPassword }`                 | Completes reset                             |

### Mobile Summary — `/api/um/mobile`

From `MobileSummaryController`. Aggregates UM wallet data with OCPP transactions.

| Method | Route            | Description                                                         |
| ------ | ---------------- | ------------------------------------------------------------------- |
| GET    | `/summary`       | Wallet balance + lifetime energy/sessions/spend (calls OCPP bridge) |
| GET    | `/transactions`  | Paged history joined to cost/energy data                            |

### Wallet — `/api/Wallet`

From `WalletController`. Some routes require admin permissions; the rest are resolved per caller.

| Method | Route              | Description                                                         |
| ------ | ------------------ | ------------------------------------------------------------------- |
| GET    | `/`                | Current user's wallet                                               |
| GET    | `/balance`         | Balance only                                                        |
| GET    | `/check-balance`   | Sufficiency check (`?amount=...`)                                   |
| POST   | `/topup`           | Top up wallet (payment gateway hook — currently a stub)             |
| GET    | `/transactions`    | Ledger for the current user                                         |
| GET    | `/all`             | Admin: wallets across all users                                     |
| POST   | `/adjust`          | Admin: credit/debit adjustment with reason                          |

### Documents — `/api/documents`

From `DocumentController`.

| Method | Route     | Body (multipart)                                   | Description                                   |
| ------ | --------- | -------------------------------------------------- | --------------------------------------------- |
| POST   | `/upload` | `file, category, altText?, caption?`               | Uploads to Azure Blob; creates thumbnail; links to caller's `Docket` |

### Settings — `/api/um/settings`

From `SettingsController`.

| Method | Route          | Description                                             |
| ------ | -------------- | ------------------------------------------------------- |
| GET    | `/`            | All settings grouped by category                        |
| PUT    | `/`            | Bulk update of settings                                 |
| GET    | `/{category}`  | Settings for one category (`General`, `Notifications`, `Security`, `Appearance`) |

### Admin Dashboard — `/api/admin`

| Method | Route             | Description                                 |
| ------ | ----------------- | ------------------------------------------- |
| GET    | `/dashboard`      | Admin dashboard aggregates                  |
| GET    | `/system-stats`   | System-wide statistics                      |

### Permission Examples — `/api/Permissions`

From `PermissionsController`. Primarily sample endpoints demonstrating policy usage; not part of the app surface consumed by the portal.

---

## Response Envelope

All successful responses are unwrapped by the portal's RTK Query `baseApi` (see `ChargeLK.AdminUI/src/store/api/baseApi.ts`). Directly called from outside the portal the envelope is:

```json
{
  "isSuccess": true,
  "data": { /* payload */ },
  "error": null
}
```

```json
{
  "isSuccess": false,
  "data": null,
  "error": {
    "code": "TAG.ALREADY_ACTIVE",
    "description": "Tag is already active",
    "type": "Validation"
  }
}
```

## Authentication

- **Admin portal & mobile app**: Bearer JWT in the `Authorization` header. Obtained via `/api/auth/login` (portal) or `/api/um/auth/mobile/login` or `/api/um/auth/mobile/google` (mobile).
- **Refresh flow**: `POST /api/auth/refresh` with `{ accessToken, refreshToken }`. The portal handles this transparently via a mutex-guarded interceptor.
- **Permission policies**: authorization uses the `resource:action` pattern (see `ChargeLK.Domain/Constants/Permissions.cs`). Policies are materialised on demand by `PermissionPolicyProvider` and enforced by `PermissionAuthorizationHandler`.
- **Cross-service trust**: OCPP server accepts JWTs signed by the UM service using the shared `Jwt__SecretKey` and the same issuer/audience configuration. A separate scheme is registered on each service for the other's tokens.

## Related Documentation

- [System Architecture](../02-Architecture/README.md)
- [Components Documentation](../03-Components/README.md)
- [Data Architecture](../05-Data-Architecture/README.md)
- [Security](../08-Security/README.md)
