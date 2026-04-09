# Authentication & Authorization Implementation Guide

## Overview

ChargeLK implements industry-standard authentication and authorization using JWT tokens and permission-based access control (PBAC) combined with role-based access control (RBAC).

## Architecture

### ChargeLK.UM (User Management Service)
- **Responsibility**: Issues and manages JWT tokens
- **Authentication**: ASP.NET Core Identity with JWT Bearer tokens
- **Authorization**: Permission-based policies with role mappings

### ChargeLK.OCPP (Charger Management Service)
- **Responsibility**: Validates JWT tokens issued by UM service
- **Authentication**: JWT Bearer token validation
- **Authorization**: Permission-based policies for OCPP operations

## Authentication Flow

### 1. User Registration
```
POST /api/auth/register
{
  "email": "user@example.com",
  "password": "SecurePassword123!",
  "firstName": "John",
  "lastName": "Doe"
}
```

### 2. User Login
```
POST /api/auth/login
{
  "email": "user@example.com",
  "password": "SecurePassword123!"
}

Response:
{
  "userData": {
    "id": 1,
    "userName": "user@example.com",
    "email": "user@example.com",
    "firstName": "John",
    "lastName": "Doe"
  },
  "accessToken": "eyJhbGc...",
  "refreshToken": "base64-encoded-token",
  "accessTokenExpiration": "2025-12-15T13:00:00Z"
}
```

### 3. Token Refresh
```
POST /api/auth/refresh
{
  "accessToken": "expired-or-valid-access-token",
  "refreshToken": "valid-refresh-token"
}
```

### 4. Logout
```
POST /api/auth/logout
{
  "accessToken": "current-access-token",
  "refreshToken": "current-refresh-token"
}
```

## JWT Token Structure

### Access Token Claims
```json
{
  "sub": "1",                                    // User ID
  "email": "user@example.com",
  "name": "user@example.com",
  "FirstName": "John",
  "LastName": "Doe",
  "role": ["Admin", "Operator"],                 // User roles
  "permissions": [                               // Auto-assigned based on roles
    "users:view",
    "users:manage",
    "ocpp:chargepoint:view",
    "ocpp:chargepoint:manage"
  ],
  "jti": "unique-token-id",
  "iat": 1734264000,                             // Issued at
  "exp": 1734267600,                             // Expiration
  "iss": "https://api.chargelk.com/um",         // Issuer
  "aud": "chargelk-services"                     // Audience
}
```

## Roles and Permissions

### Predefined Roles

#### SystemAdmin
- Full access to all features
- Permissions: All permissions across all modules

#### Admin
- User and role management
- Full OCPP operations
- Document management
- Permissions:
  - `users:*` (view, create, update, manage)
  - `roles:view`, `roles:assign`
  - `documents:*`
  - `ocpp:*` (all OCPP permissions)
  - `system:audit:view`, `system:health:view`

#### Operator
- Charge point operations
- Transaction management
- Permissions:
  - `users:view`
  - `documents:view`, `documents:upload`, `documents:download`
  - `ocpp:chargepoint:*`
  - `ocpp:tag:*`
  - `ocpp:transaction:*`
  - `ocpp:reservation:*`

#### Support
- View-only access for customer support
- Permissions:
  - `users:view`
  - `documents:view`, `documents:download`
  - `ocpp:chargepoint:view`
  - `ocpp:tag:view`
  - `ocpp:transaction:view`
  - `ocpp:reservation:view`

#### User
- Basic user permissions
- Permissions:
  - `documents:view`, `documents:upload`, `documents:download`
  - `ocpp:chargepoint:view`
  - `ocpp:transaction:view`

### Permission Naming Convention

Format: `resource:action`

Examples:
- `users:view` - View users
- `users:create` - Create users
- `users:manage` - Full user management
- `ocpp:chargepoint:view` - View charge points
- `ocpp:chargepoint:manage` - Manage charge points
- `ocpp:chargepoint:control` - Control charge points (start/stop)

## Authorization Policies

### ChargeLK.UM Policies

```csharp
// User Management
[Authorize(Policy = PolicyNames.ViewUsers)]        // users:view
[Authorize(Policy = PolicyNames.CreateUsers)]      // users:create
[Authorize(Policy = PolicyNames.UpdateUsers)]      // users:update
[Authorize(Policy = PolicyNames.DeleteUsers)]      // users:delete
[Authorize(Policy = PolicyNames.ManageUsers)]      // users:manage

// Role Management
[Authorize(Policy = PolicyNames.ViewRoles)]        // roles:view
[Authorize(Policy = PolicyNames.ManageRoles)]      // roles:manage
[Authorize(Policy = PolicyNames.AssignRoles)]      // roles:assign

// Document Management
[Authorize(Policy = PolicyNames.UploadDocuments)]  // documents:upload
[Authorize(Policy = PolicyNames.ManageDocuments)]  // documents:manage

// Role-based (for simple role checks)
[Authorize(Policy = PolicyNames.RequireAdmin)]     // Admin or SystemAdmin role
[Authorize(Policy = PolicyNames.RequireOperator)]  // Operator, Admin, or SystemAdmin
```

### ChargeLK.OCPP Policies

```csharp
// Charge Point Management
[Authorize(Policy = "CanViewChargePoints")]        // ocpp:chargepoint:view
[Authorize(Policy = "CanManageChargePoints")]      // ocpp:chargepoint:manage
[Authorize(Policy = "CanControlChargePoints")]     // ocpp:chargepoint:control

// Tag Management
[Authorize(Policy = "CanViewTags")]                // ocpp:tag:view
[Authorize(Policy = "CanManageTags")]              // ocpp:tag:manage

// Transaction Management
[Authorize(Policy = "CanViewTransactions")]        // ocpp:transaction:view
[Authorize(Policy = "CanManageTransactions")]      // ocpp:transaction:manage
```

## Security Best Practices Implemented

### 1. Token Security
- ✅ Minimum 256-bit (32 characters) secret key validation
- ✅ Zero clock skew - no tolerance for expired tokens
- ✅ Unique JWT ID (jti) for each token
- ✅ Issued at (iat) timestamp for token freshness
- ✅ Secure refresh token storage with expiration tracking
- ✅ Device and IP tracking for refresh tokens

### 2. Password Security
- ✅ ASP.NET Core Identity password requirements:
  - Minimum 8 characters
  - Requires digit
  - Requires lowercase
  - Requires uppercase
  - Requires non-alphanumeric
- ✅ Account lockout after 5 failed attempts (5 minutes)
- ✅ Password hashing with bcrypt

### 3. Authorization
- ✅ Permission-based authorization with custom handlers
- ✅ Granular permissions per resource and action
- ✅ Role-permission mapping
- ✅ Comprehensive logging of authorization failures

### 4. Token Management
- ✅ Refresh token rotation on use
- ✅ Refresh token revocation on logout
- ✅ Device-based refresh token management
- ✅ IP address tracking

### 5. Cross-Service Communication
- ✅ Shared JWT secret between UM and OCPP services
- ✅ Consistent issuer/audience validation
- ✅ Same permission claim structure

## Configuration

### Using User Secrets (Recommended for Development)

For local development, use .NET User Secrets to store sensitive configuration:

```bash
# UM Service
cd ChargeLK.UM/ChargeLK.API
dotnet user-secrets set "Jwt:SecretKey" "your-super-secret-key-must-be-at-least-32-characters-long-for-production"
dotnet user-secrets set "Jwt:Issuer" "https://api.chargelk.com/um"
dotnet user-secrets set "Jwt:Audience" "chargelk-services"

# OCPP Service (MUST match UM service)
cd ChargeLK.OCPP/ChargeLK.API
dotnet user-secrets set "Jwt:SecretKey" "your-super-secret-key-must-be-at-least-32-characters-long-for-production"
dotnet user-secrets set "Jwt:Issuer" "https://api.chargelk.com/um"
dotnet user-secrets set "Jwt:Audience" "chargelk-services"
```

**Benefits:**
- ✅ Secrets stored outside source control
- ✅ Different settings per developer
- ✅ Easy to manage and update
- ✅ Overrides appsettings.json values

### appsettings.json (UM Service)
```json
{
  "Jwt": {
    "SecretKey": "your-super-secret-key-must-be-at-least-32-characters-long-for-production",
    "Issuer": "https://api.chargelk.com/um",
    "Audience": "chargelk-services",
    "ExpirationInMinutes": 60,
    "RefreshTokenExpirationInDays": 7
  }
}
```

### appsettings.json (OCPP Service)
```json
{
  "Jwt": {
    "SecretKey": "your-super-secret-key-must-be-at-least-32-characters-long-for-production",
    "Issuer": "https://api.chargelk.com/um",
    "Audience": "chargelk-services",
    "ExpirationInMinutes": 60,
    "RefreshTokenExpirationInDays": 7
  }
}
```

**Important**: Both services must use the same `SecretKey` and `Issuer` for token validation to work.

## API Usage Examples

### Accessing Protected Endpoints

```bash
# Include JWT token in Authorization header
curl -H "Authorization: Bearer eyJhbGc..." \
  https://api.chargelk.com/um/api/users

# OCPP service endpoints (uses same token)
curl -H "Authorization: Bearer eyJhbGc..." \
  https://api.chargelk.com/ocpp/api/chargepoints
```

### Assigning Roles to Users

```bash
POST /api/auth/user-role
{
  "userId": 1,
  "roleName": "Operator"
}
```

### Assigning Custom Permissions

```bash
POST /api/auth/user-claim
{
  "userId": 1,
  "claimType": "permissions",
  "claimValue": "ocpp:chargepoint:control"
}
```

## Troubleshooting

### Token Validation Failures

1. **"The token is invalid"**
   - Check that `SecretKey` matches between UM and OCPP services
   - Verify `Issuer` and `Audience` are configured correctly

2. **"The token is expired"**
   - Use the refresh token endpoint to get a new access token
   - Check system clocks are synchronized

3. **"Insufficient permissions"**
   - Verify user has the required role
   - Check permission claims in the token
   - Review authorization policy configuration

### Common Issues

- **401 Unauthorized**: Token is missing, invalid, or expired
- **403 Forbidden**: Token is valid but lacks required permissions

## Migration from Legacy System

If migrating from the old permission system:

1. Update policy names from `AuthorizationPolicies` to `PolicyNames`
2. Replace permission claims:
   - `user_management` → `users:manage`
   - `document_upload` → `documents:upload`
   - `read` → specific resource permissions
3. Update role names if needed (User, Admin → SystemAdmin, Admin, Operator, Support, User)

## Security Recommendations

### Production Deployment

1. ✅ Set `RequireHttpsMetadata = true` in JWT configuration
2. ✅ Use strong, randomly generated secret keys (minimum 64 characters)
3. ✅ Store secrets in Azure Key Vault or similar
4. ✅ Enable HTTPS redirection
5. ✅ Implement rate limiting on auth endpoints
6. ✅ Monitor and alert on failed authentication attempts
7. ✅ Regular security audits and token rotation
8. ✅ Implement token blacklisting for compromised tokens

### Future Enhancements

- [ ] Implement asymmetric JWT (RS256) with public/private keys
- [ ] Add multi-factor authentication (MFA)
- [ ] Implement token blacklisting/revocation list
- [ ] Add permission caching for performance
- [ ] Implement dynamic permission assignment from database
- [ ] Add API key authentication for service-to-service calls
- [ ] Implement OAuth2/OIDC for third-party integrations

