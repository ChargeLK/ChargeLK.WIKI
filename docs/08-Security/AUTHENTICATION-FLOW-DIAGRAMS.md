# ChargeLK Authentication & Authorization Flow Diagrams

## 1. Authentication Flow

```
┌─────────────┐
│   Client    │
│ (Web/Mobile)│
└──────┬──────┘
       │
       │ 1. POST /api/auth/login
       │    { email, password }
       ▼
┌─────────────────────────────────────────────────────────┐
│              ChargeLK.UM Service (Port 5001)            │
├─────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────┐  │
│  │ ApplicationUserController                         │  │
│  │  └─> LoginApplicationUserCommand                 │  │
│  └────────────────────┬─────────────────────────────┘  │
│                       │                                 │
│  ┌────────────────────▼─────────────────────────────┐  │
│  │ LoginUserCommandHandler                          │  │
│  │  1. Find user by email                          │  │
│  │  2. Verify password                             │  │
│  │  3. Check lockout status                        │  │
│  │  4. Check user claims (IsBlocked)               │  │
│  │  5. Call TokenService.GenerateTokensAsync()     │  │
│  └────────────────────┬─────────────────────────────┘  │
│                       │                                 │
│  ┌────────────────────▼─────────────────────────────┐  │
│  │ TokenService                                      │  │
│  │  1. BuildUserClaimsAsync()                       │  │
│  │     - User ID, email, name                      │  │
│  │     - Roles from UserManager                    │  │
│  │     - User claims from database                 │  │
│  │     - Auto-mapped role permissions              │  │
│  │  2. GenerateAccessToken()                       │  │
│  │     - Create JWT with claims                    │  │
│  │     - Sign with HMAC-SHA256                     │  │
│  │     - Set 60-minute expiration                  │  │
│  │  3. StoreRefreshTokenAsync()                    │  │
│  │     - Generate 256-bit random token             │  │
│  │     - Store in database with device/IP          │  │
│  │     - Set 7-day expiration                      │  │
│  └────────────────────┬─────────────────────────────┘  │
│                       │                                 │
└───────────────────────┼─────────────────────────────────┘
                        │
       2. Response      │
       ┌────────────────▼────────────────┐
       │ {                               │
       │   accessToken: "eyJhbGc...",    │
       │   refreshToken: "base64...",    │
       │   expiration: "2025-12-16...",  │
       │   userData: { ... }             │
       │ }                               │
       └─────────────────────────────────┘
                        │
                        ▼
       ┌─────────────────────────────────┐
       │ Client stores tokens securely   │
       └─────────────────────────────────┘
```

## 2. Authorization Flow (Protected Endpoint)

```
┌─────────────┐
│   Client    │
└──────┬──────┘
       │
       │ GET /api/ocpp/chargepoints
       │ Authorization: Bearer eyJhbGc...
       ▼
┌─────────────────────────────────────────────────────────┐
│            ChargeLK.OCPP Service (Port 5002)            │
├─────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────┐  │
│  │ ASP.NET Core Middleware Pipeline                 │  │
│  └────────────────────┬─────────────────────────────┘  │
│                       │                                 │
│  ┌────────────────────▼─────────────────────────────┐  │
│  │ JWT Bearer Authentication Middleware             │  │
│  │  1. Extract token from header                   │  │
│  │  2. Validate signature (HMAC-SHA256)            │  │
│  │  3. Validate issuer (UM service)                │  │
│  │  4. Validate audience (chargelk-services)       │  │
│  │  5. Validate expiration                         │  │
│  │  6. Create ClaimsPrincipal                      │  │
│  └────────────────────┬─────────────────────────────┘  │
│                       │ ✓ Authenticated                 │
│  ┌────────────────────▼─────────────────────────────┐  │
│  │ Authorization Middleware                         │  │
│  │  1. Get required policy from [Authorize] attr   │  │
│  │     Policy: "CanViewChargePoints"               │  │
│  │  2. Find policy requirements                    │  │
│  │     Requirement: Permission("ocpp:chargepoint:view")│
│  └────────────────────┬─────────────────────────────┘  │
│                       │                                 │
│  ┌────────────────────▼─────────────────────────────┐  │
│  │ PermissionAuthorizationHandler                   │  │
│  │  1. Get user claims from ClaimsPrincipal        │  │
│  │  2. Check if "permissions" claim exists         │  │
│  │  3. Verify claim value matches requirement      │  │
│  │     ✓ "ocpp:chargepoint:view" found in claims   │  │
│  │  4. Call context.Succeed(requirement)           │  │
│  └────────────────────┬─────────────────────────────┘  │
│                       │ ✓ Authorized                    │
│  ┌────────────────────▼─────────────────────────────┐  │
│  │ ChargePointsController                           │  │
│  │  └─> GetChargePointsQuery (MediatR)             │  │
│  └────────────────────┬─────────────────────────────┘  │
│                       │                                 │
│  ┌────────────────────▼─────────────────────────────┐  │
│  │ Query Handler → Repository → Database           │  │
│  └────────────────────┬─────────────────────────────┘  │
└───────────────────────┼─────────────────────────────────┘
                        │
                        ▼
       ┌─────────────────────────────────┐
       │ 200 OK                          │
       │ [ { chargePoint data } ]        │
       └─────────────────────────────────┘
```

## 3. Token Refresh Flow

```
┌─────────────┐
│   Client    │
│ (Token expired after 60 min)
└──────┬──────┘
       │
       │ POST /api/auth/refresh
       │ { accessToken, refreshToken }
       ▼
┌─────────────────────────────────────────────────────────┐
│              ChargeLK.UM Service (Port 5001)            │
├─────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────┐  │
│  │ RefreshTokenCommand → RefreshTokenCommandHandler │  │
│  └────────────────────┬─────────────────────────────┘  │
│                       │                                 │
│  ┌────────────────────▼─────────────────────────────┐  │
│  │ TokenService.RefreshTokenAsync()                 │  │
│  │  1. ValidateAndGetUserFromTokenAsync()          │  │
│  │     - Validate access token (can be expired)    │  │
│  │     - Extract user ID from token                │  │
│  │     - Find user in database                     │  │
│  │     - Find refresh token in database            │  │
│  │     - Validate refresh token not expired        │  │
│  │  2. GenerateTokensAsync()                       │  │
│  │     - Generate new access token                 │  │
│  │     - Generate new refresh token                │  │
│  │     - Update/replace old refresh token          │  │
│  └────────────────────┬─────────────────────────────┘  │
└───────────────────────┼─────────────────────────────────┘
                        │
                        ▼
       ┌─────────────────────────────────┐
       │ New tokens issued               │
       │ Old refresh token invalidated   │
       └─────────────────────────────────┘
```

## 4. Cross-Service Authentication Flow

```
┌─────────────┐
│   Client    │
└──────┬──────┘
       │
       │ 1. Login to UM
       ▼
┌──────────────────┐
│  ChargeLK.UM     │
│  (Port 5001)     │
└────────┬─────────┘
         │
         │ 2. Receives JWT token
         │    Issuer: "https://api.chargelk.com/um"
         │    Audience: "chargelk-services"
         │    Permissions: ["ocpp:chargepoint:view", ...]
         ▼
┌─────────────┐
│   Client    │
└──────┬──────┘
       │
       │ 3. Call OCPP endpoint with same token
       │    Authorization: Bearer <UM-issued-token>
       ▼
┌──────────────────┐
│  ChargeLK.OCPP   │
│  (Port 5002)     │
├──────────────────┤
│ JWT Validation:  │
│ ✓ Same secret    │
│ ✓ Issuer matches │
│ ✓ Audience OK    │
│ ✓ Signature OK   │
│ ✓ Not expired    │
├──────────────────┤
│ Authorization:   │
│ ✓ Check permission│
│   in token claims│
└──────────────────┘
       │
       ▼
┌─────────────────────────────────┐
│ Access granted to OCPP resource │
└─────────────────────────────────┘
```

## 5. Role-Permission Mapping

```
┌─────────────────────────────────────────────────────────┐
│                    User Login                           │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│ TokenService.BuildUserClaimsAsync()                     │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  1. Get user from database                              │
│     User: john.doe@example.com                          │
│                                                          │
│  2. Get roles from UserManager                          │
│     Roles: ["Admin", "Operator"]                        │
│                                                          │
│  3. Get user-specific claims                            │
│     UserClaims: [{ type: "IsBlocked", value: "false" }] │
│                                                          │
│  4. Map roles to permissions (GetPermissionsForRole)    │
│     ┌───────────────────────────────────────────┐       │
│     │ Admin → [                                 │       │
│     │   "users:view", "users:create",          │       │
│     │   "users:update", "users:manage",        │       │
│     │   "roles:view", "roles:assign",          │       │
│     │   "documents:*",                         │       │
│     │   "ocpp:*",                              │       │
│     │   "system:audit:view"                    │       │
│     │ ]                                        │       │
│     │                                          │       │
│     │ Operator → [                             │       │
│     │   "users:view",                          │       │
│     │   "documents:view",                      │       │
│     │   "ocpp:chargepoint:*",                  │       │
│     │   "ocpp:tag:*",                          │       │
│     │   "ocpp:transaction:*"                   │       │
│     │ ]                                        │       │
│     └───────────────────────────────────────────┘       │
│                                                          │
│  5. Build final claims list                             │
│     Claims: [                                           │
│       { type: "sub", value: "1" },                      │
│       { type: "email", value: "john.doe@example.com" }, │
│       { type: "role", value: "Admin" },                 │
│       { type: "role", value: "Operator" },              │
│       { type: "permissions", value: "users:view" },     │
│       { type: "permissions", value: "users:create" },   │
│       { type: "permissions", value: "ocpp:chargepoint:manage" },│
│       ... (all unique permissions)                      │
│     ]                                                   │
└─────────────────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│           JWT Token with all claims                     │
└─────────────────────────────────────────────────────────┘
```

## 6. Permission Check Decision Tree

```
                    Request arrives
                          │
                          ▼
              ┌───────────────────────┐
              │ [Authorize] attribute?│
              └───────┬───────────────┘
                      │
         ┌────────────┴────────────┐
         │                         │
        YES                       NO
         │                         │
         ▼                         ▼
   ┌─────────────┐          ┌──────────┐
   │ Check Policy│          │  ALLOW   │
   └──────┬──────┘          └──────────┘
          │
          ▼
   ┌──────────────────┐
   │ User authenticated?│
   └──────┬───────────┘
          │
    ┌─────┴─────┐
    │           │
   YES         NO
    │           │
    ▼           ▼
┌───────────┐ ┌────────┐
│Get Policy │ │401     │
│Requirements│ │UNAUTHORIZED│
└─────┬─────┘ └────────┘
      │
      ▼
┌───────────────────────┐
│PermissionRequirement? │
└─────┬─────────────────┘
      │
  ┌───┴───┐
  │       │
 YES     NO (Role-based)
  │       │
  ▼       ▼
┌──────────────────┐  ┌─────────────┐
│Check if user has │  │Check if user│
│permission claim  │  │has role     │
└────┬──────────┬──┘  └──────┬──────┘
     │          │            │
   FOUND    NOT FOUND     FOUND/NOT
     │          │            │
     ▼          ▼            ▼
  ┌─────┐   ┌──────┐    ┌──────┐
  │ALLOW│   │403   │    │ALLOW/│
  │     │   │FORBIDDEN│  │403   │
  └─────┘   └──────┘    └──────┘
```

---

**Legend:**
- `┌─┐` - Process/Component
- `│` - Flow direction
- `▼` - Next step
- `✓` - Success/Valid
- `✗` - Failure/Invalid

These diagrams illustrate the complete authentication and authorization flow in the ChargeLK system.

