# Components Documentation

ChargeLK ships three components from two independent .NET 9 solutions plus one React app. This page describes each component's responsibilities, current modules, and external touchpoints.

---

## 1. ChargeLK.OCPPServer

**Solution**: `ChargeLK.OCPPServer/ChargeLKOCPP.sln`
**Projects**: `OcppServer.Domain`, `OcppServer.Application`, `OcppServer.Infrastructure`, `OcppServer.API`
**Target framework**: .NET 9
**Transport**: WebSocket (OCPP 1.6 JSON) + REST

### Responsibilities

- Serve OCPP 1.6 charge points over WebSocket.
- Persist charge point registrations, connectors, transactions, and meter values.
- Expose REST APIs for management (charge points, tags, tariffs, maintenance, reviews, transactions) and for UM-to-OCPP integration.
- Calculate charging cost based on tariffs.
- Maintain a tag audit log for compliance and forensics.

### Modules

#### Domain (`OcppServer.Domain/Entities`)

| Entity                   | Purpose                                                     |
|--------------------------|-------------------------------------------------------------|
| `ChargePointEntity`      | Charger registration, vendor/firmware, location, status     |
| `Connector`              | Connector per charge point with status                      |
| `TransactionEntity`      | A charging transaction (start → stop)                       |
| `TransactionMeterValue`  | Sampled meter values during a transaction                   |
| `OcppTag`                | RFID / virtual tag                                          |
| `OcppTagAuditLog`        | Every status or assignment change on a tag                  |
| `OcppOperation`          | Server-initiated operation log (RemoteStart, Reset, etc.)   |
| `Reservation`            | Connector reservation (OCPP `ReserveNow`)                   |
| `Tariff`                 | Pricing: per kWh, per minute, session fee, peak windows     |
| `ChargerMaintenanceEntity`| Maintenance record for a charge point                      |
| `ChargerReview`          | Driver-submitted review (rating + message)                  |
| `ChargingProfile`        | OCPP smart-charging profile                                 |
| `ChargingSchedulePeriod` | Period row of a charging profile                            |

#### Application (`OcppServer.Application/Handlers`)

- **ChargePoint**: `RegisterChargePointCommand`, `DeleteChargePointCommand`, `GetChargePointsQuery`, `GetChargePointByChargeBoxIdQuery`.
- **OcppTag**: `CreateOcppTagCommand`, `Update…`, `Activate…`, `Deactivate…`, `Suspend…`, `Block…`, `Unblock…`, `AssignTagToUserCommand`, `UnassignTagFromUserCommand`, `ChangeTagStatusCommand`, `BulkImportTagsCommand`, `DeleteOcppTagCommand`; queries by ID / by `IdTag` / list.
- **Tariff**: `CreateTariffCommand`, `UpdateTariffCommand`, `DeleteTariffCommand`; list + by ID.
- **ChargerMaintenance**: Create / Update / Delete + `GetMaintenanceRecordsQuery`.
- **Transaction** (query-only): paged list, by ID, active only, by charge point, by OCPP tag (latest + paged).
- **Mobile** (`MobileHandlers.cs`): `CreateVirtualTagCommand`, `GetUserChargingSummaryQuery`, `GetUserTransactionsQuery`, `GetActiveChargingSessionQuery` — called from the UM service.

#### Infrastructure (`OcppServer.Infrastructure`)

- **Persistence**: `ApplicationDbContext` (EF Core 8, PostgreSQL via Npgsql), `EntityConfigurations/` per aggregate, `Migrations/`.
- **Services**:
  - `Services/Ocpp/OcppMessageHandler.cs` — action dispatcher for BootNotification, Heartbeat, Authorize, StartTransaction, StopTransaction, MeterValues, StatusNotification, DataTransfer.
  - `Services/Ocpp/OcppServerService.cs` — connection registry (static dictionary keyed by `chargeBoxId`) and server-initiated calls (RemoteStart/Stop, Reset, UnlockConnector, ClearCache).
  - `Services/Tag/VirtualTagService.cs` — generates deterministic virtual tags for registered mobile users.
  - `Services/Billing/CostCalculationService.cs` — applies tariff rules (peak pricing, min/max, session fee).
  - `Services/Time/*`, `Services/User/CurrentUserService`.
- **Interceptors**: audit / domain-event dispatching.
- **Repositories**: one per aggregate.

#### API (`OcppServer.API`)

- Controllers:
  - `ChargePointsController` — CRUD + Remote (start/stop, unlock, reset, clear cache).
  - `OcppTagsController` — full tag lifecycle + bulk import + audit-log view.
  - `TariffsController` — CRUD.
  - `ChargerMaintenanceController` — CRUD.
  - `ChargerReviewsController` — list by charge point (anon), create/update (auth), hide (admin).
  - `TransactionsController` — read-only queries.
  - `MobileController` (`/api/ocpp/*`) — internal integration endpoints for UM.
- Middleware: `OcppWebSocketMiddleware` (upgrades `/ocpp/*` to `ocpp1.6` sub-protocol), `RequestContextLoggingMiddleware`.
- Extensions (`OcppServer.API/Extentions/`, note spelling): `ServiceCollectionExtensions` (`UseSwaggerWithUi`, `AddSwaggerGenWithAuth` — both `internal`), `MigrationExtentions.ApplyMigrations` (Development only), `FileUploadOperationFilter`.

### External Integrations

| Direction         | Endpoint / Mechanism                                      |
|-------------------|-----------------------------------------------------------|
| Charge Point → OCPP | WebSocket `/ocpp/{chargeBoxId}` (`ocpp1.6`)            |
| UM → OCPP           | `POST /api/ocpp/tags/virtual`, `GET /api/ocpp/mobile/user/{id}/summary|transactions|active-session` |
| Admin UI → OCPP     | REST under `/api/ChargePoints`, `/api/OcppTags`, `/api/Tariffs`, `/api/Transactions`, `/api/ChargerMaintenance`, `/api/ChargerReviews` |

---

## 2. ChargeLK.UM

**Solution**: `ChargeLK.UM/ChargeLK.UM.sln`
**Projects**: `ChargeLK.Domain`, `ChargeLK.Application`, `ChargeLK.Infrastructure`, `ChargeLK.API`
**Target framework**: .NET 9
**Transport**: REST (JSON), event publishing via Azure Service Bus

### Responsibilities

- ASP.NET Identity-based user management (long-keyed `ApplicationUser`).
- JWT access + refresh token flow, with refresh-token persistence through Identity's `ApplicationUserToken`.
- RBAC + fine-grained permission management (`resource:action`).
- Mobile-app facing authentication (email/password, Google Sign-In, forgot/reset password).
- Wallet, wallet transactions, and wallet-based authorisation for charging.
- Document + Docket management backed by Azure Blob Storage.
- System settings (General, Notifications, Security, Appearance).
- Bridge to the OCPP server for charging summaries and virtual-tag provisioning.

### Modules

#### Domain (`ChargeLK.Domain/Entities`)

| Aggregate              | Key entities                                                                 |
|------------------------|------------------------------------------------------------------------------|
| User                   | `ApplicationUser`, `ApplicationRole`, `ApplicationUserRole`, `ApplicationUserClaim`, `ApplicationUserLogin`, `ApplicationUserToken`, `ApplicationRoleClaim`, `Permission`, `RolePermission` |
| Dockets & Documents    | `Docket`, `Document` (+ `DocumentCategory`, `DocumentStatus` enums)          |
| Wallet                 | `UserWallet`, `WalletTransaction`                                            |
| Charging Session       | `MobileChargingSession`                                                      |
| Settings               | `SystemSettings`                                                             |

Domain events (`ChargeLK.Domain/DomainEvents/Users/Events`):
- `UserCreatedDomainEvent` — raised from `ApplicationUser.Create`.
- `CreateBlobContainerEvent` — raised to bootstrap Blob storage for a new user.

Constants (`ChargeLK.Domain/Constants`):
- `Roles` — `SystemAdmin`, `Admin`, `Operator`, `Support`, `User`.
- `Permissions` — nested under `Users`, `Roles`, `Documents`, `System`, `Ocpp` (`chargepoint:view|manage|control`, `tag:view|manage`, `transaction:view|manage`, `reservation:*`, `review:*`).
- `AuthenticationSchemes`, `CustomClaimTypes`.

#### Application

- **Handlers**:
  - `Auth/Commands`: `RefreshTokenCommand`, `RevokeTokenCommand`.
  - `User/Commands`: Register, Login, CreateRole, CreateRoleClaim, CreateUserRole, CreateUserClaim, RemoveUserClaim, RemoveUserRole.
  - `User/Queries`: `GetAllUsersQuery`, `GetUserByIdQuery`, `GetUserRolesQuery`, `GetUserPermissionsQuery`.
  - `User/EventHandlers`: `CreateBlobContainerEventHandler`.
  - `Role/Queries`: `GetAllRolesQuery`. `Permission/Queries`: `GetAllPermissionsQuery`.
  - `Mobile/Commands`: `LoginMobileCommand`, `SignupMobileCommand`, `GoogleSignInCommand`, `ForgotPasswordCommand`, `ResetPasswordCommand`.
  - `Mobile/Queries`: `GetUserSummaryQuery`, `GetUserTransactionsQuery`.
  - `Document/Commands`: `UploadDocumentCommand`, `CreateDocumentCommand`.
  - `Settings/*`: `GetAllSettingsQuery`, `UpdateAllSettingsCommand`.
  - `DomainEvents/UserCreatedOcppTagGenerationHandler` — calls the OCPP server to provision a virtual tag when a user is created.
- **Behaviors** (MediatR pipeline): `ValidationPipelineBehavior`, `RequestLoggingPipelineBehavior`.
- **Interfaces** — ports for Authentication, ChargingSession, Document, Messaging, SystemSetting, Token, User, Wallet.

#### Infrastructure

- **Database** (`ChargeLK.Infrastructure/Database`): `ApplicationDbContext` extending `IdentityDbContext<ApplicationUser, ApplicationRole, long, …>`; snake_case naming convention via `EFCore.NamingConventions`; seeds in `EntityConfigurations/Seed/` (SQL + `IdentitySeedData`, `SystemSettingsSeedData`).
- **Authorization**: custom `PermissionPolicyProvider` + `PermissionAuthorizationHandler` + `PermissionRequirement`.
- **Services**:
  - `Authentication/EmailService`, `Authentication/GoogleAuthService`.
  - `Token/TokenService`, `Token/JwtSecurityTokenValidator`.
  - `Permission/PermissionService`, `Permission/PermissionClaimsTransformation`.
  - `Document/DocumentService`, `DocumentStorageService` (Blob), `ThumbnailService` (ImageSharp), `ContainerStrategyService`.
  - `Messaging/AzureServiceBusPublisher`, `Messaging/AzureServiceBusSubscriber`.
  - `Wallet/WalletService`.
  - `User/*`, `Time/*`.
- **Interceptors**: `AuditableEntitySaveChangesInterceptor`.
- **Repositories**: per aggregate (User, Docket, Documents, Wallet, ChargingSession, Settings).

#### API

- Controllers under `ChargeLK.API/Controllers/RestApi/`:
  - `ApplicationUser/ApplicationUserController` — `/api/auth/*` (register, login, refresh, logout, role-claim, role, user-role, user-claim, list users, get user, get roles, get permissions, user roles, user permissions, remove role/claim).
  - `Mobile/MobileAuthController` — `/api/um/auth/mobile/*` (login, signup, google, forgot-password, reset-password).
  - `Mobile/MobileSummaryController` — `/api/um/mobile/{summary,transactions}`.
  - `Wallet/WalletController` — `/api/Wallet/*`.
  - `Document/DocumentController` — `/api/documents/upload` (multipart).
  - `Settings/SettingsController` — `/api/um/settings` (+ by-category).
  - `Admin/AdminController` — `/api/admin/*`.
- Middleware: `RequestContextLoggingMiddleware`.
- Extensions: `ServiceCollectionExtensions`, `ApplicationBuilderExtensions`, `MiddlewareExtensions`, `MigrationExtensions`, `FileUploadOperationFilter`.

### External Integrations

| Direction        | Endpoint / Mechanism                                                     |
|------------------|--------------------------------------------------------------------------|
| UM → OCPP        | `POST /api/ocpp/tags/virtual`, mobile summary/transactions/active-session |
| Mobile → UM      | `/api/um/auth/mobile/*`, `/api/um/mobile/*`                              |
| Admin UI → UM    | `/api/auth/*`, `/api/um/settings/*`, `/api/Wallet/*`, `/api/documents/upload`, `/api/admin/*` |
| UM → Google      | ID-token verification (`Google.Apis.Auth`)                               |
| UM → Azure Blob  | Document upload / thumbnail storage                                      |
| UM → Service Bus | Publisher + subscriber (integration events)                              |

---

## 3. ChargeLK.AdminUI

**Path**: `ChargeLK.AdminUI/`
**Framework**: React 18 + TypeScript + Vite
**UI**: Material UI v6 (+ `x-data-grid`, `x-date-pickers`), Recharts, framer-motion, notistack
**State**: Redux Toolkit + RTK Query + redux-persist

### Responsibilities

- Operator / admin portal for managing users, roles, permissions, tags, charge points, transactions, and settings.
- Present live data from both backend services through a single RTK Query `baseApi`.
- Handle JWT access/refresh automatically, including 401 retry with mutex-guarded refresh.
- Generate QR codes for charge points (print/export to PDF).

### Source layout (`src/`)

| Folder                | Contents                                                                      |
|-----------------------|------------------------------------------------------------------------------|
| `pages/`              | `Dashboard`, `ChargePoints`, `ChargePointDetail`, `OcppTags`, `Transactions`, `Users`, `Settings`, `Login` |
| `components/Layout/`  | `Layout.tsx` — shell layout                                                   |
| `components/OcppTags/`| `AssignUserToTagDialog`, `TagAssignmentHistory`, `TagBulkImport`              |
| `components/Users/`   | `AssignTagToUserDialog`, `UserRolesDialog`, `UserPermissionsDialog`           |
| `components/QrCode/`  | QR export helpers                                                             |
| `store/api/`          | `baseApi`, `authApi`, `chargePointsApi`, `ocppTagsApi`, `transactionsApi`, `usersApi`, `rolesApi`, `settingsApi` |
| `store/slices/`       | `authSlice.ts`                                                                |

### API contract assumptions

- Base URL via `VITE_API_BASE_URL` (default `/api` through Vite dev proxy).
- `/api/ocpp/*` reaches OCPP Server; `/api/um/*` reaches UM.
- Backend responses are unwrapped from a `Result<T>` envelope (`{ isSuccess, data, error }`).
- RTK Query cache tags: `User`, `ChargePoint`, `OcppTag`, `Transaction`, `Dashboard`, `Role`, `Permission`, `Settings`.

---

## Related

- [System Architecture](../02-Architecture/README.md)
- [API Reference](../04-API-Reference/README.md)
- [Data Architecture](../05-Data-Architecture/README.md)
- [Security](../08-Security/README.md)
