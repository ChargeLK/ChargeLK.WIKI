# Microservices Authentication Implementation Guide

## Architecture Overview

ChargeLK uses a microservices architecture with two main services:

1. **ChargeLK.UM (User Management)** - Authentication Authority
   - Issues JWT tokens
   - Manages users, roles, and permissions
   - Serves as the central authentication service

2. **ChargeLK.OCPP (Charge Point Management)** - Resource Service
   - Validates JWT tokens issued by UM
   - Manages charge points, transactions, tags
   - Does NOT issue tokens - only validates them

## Authentication Flow

```
┌─────────┐         ┌──────────┐         ┌──────────┐
│  Admin  │────1───>│    UM    │         │   OCPP   │
│   UI    │<───2────│  Service │         │ Service  │
└─────────┘         └──────────┘         └──────────┘
     │                                          │
     └──────────────3─────────────────────────>│
                    (JWT Token)                │
     <──────────────4───────────────────────────┘
              (Protected Resources)
```

### Flow Steps:

1. **Login Request**: Admin UI sends credentials to UM service
2. **Token Issued**: UM validates credentials and issues JWT token (with permissions)
3. **API Request**: Admin UI calls OCPP service with JWT token in Authorization header
4. **Token Validation**: OCPP validates JWT token and checks permissions
5. **Response**: OCPP returns protected resource data

## Current Issues Fixed

### Issue 1: Permission Claim Type Inconsistency ✅ FIXED

**Problem**: 
- `TokenService` generates claims with type `"permissions"` (plural)
- `PermissionClaimsTransformation` adds claims with type `"permission"` (singular)
- UM `PermissionAuthorizationHandler` checks for `"permission"` (singular)
- OCPP `PermissionAuthorizationHandler` checks for `"permissions"` (plural)

**Solution**: Standardize on `"permissions"` (plural) across all services

### Issue 2: JWT Configuration Synchronization ✅ FIXED

**Problem**: JWT settings must be identical between UM (issuer) and OCPP (validator)

**Solution**: Both services now use identical JWT configuration:
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

### Issue 3: CurrentUserService Consistency ✅ FIXED

**Problem**: Different implementations between UM and OCPP

**Solution**: Aligned both services to extract claims consistently

### Issue 4: WebSocket Authentication ✅ FIXED

**Problem**: OCPP WebSocket connections need JWT token authentication

**Solution**: Added `OnMessageReceived` event handler in OCPP to extract token from query string

## Implementation Details

### 1. UM Service (Token Issuer)

#### Token Generation (`TokenService.cs`)
```csharp
private async Task<List<Claim>> BuildUserClaimsAsync(ApplicationUser user)
{
    var claims = new List<Claim>
    {
        // Standard JWT claims
        new(ClaimTypes.NameIdentifier, user.Id.ToString()),
        new("sub", user.Id.ToString()),
        new(ClaimTypes.Email, user.Email ?? string.Empty),
        new("email", user.Email ?? string.Empty),
        new(ClaimTypes.Name, user.UserName ?? string.Empty),
        new("name", user.UserName ?? string.Empty),
        new("FirstName", user.FirstName),
        new("LastName", user.LastName),
        new("jti", Guid.NewGuid().ToString()),
        new("iat", DateTimeOffset.UtcNow.ToUnixTimeSeconds().ToString())
    };

    // Add role claims
    var roles = await _userManager.GetRolesAsync(user);
    claims.AddRange(roles.Select(role => new Claim(ClaimTypes.Role, role)));

    // Add user-specific claims (custom permissions)
    var userClaims = await _userManager.GetClaimsAsync(user);
    claims.AddRange(userClaims);

    // Add role-based permissions (using "permissions" claim type)
    foreach (var role in roles)
    {
        var rolePermissions = GetPermissionsForRole(role);
        claims.AddRange(rolePermissions.Select(p => new Claim("permissions", p)));
    }

    return claims;
}
```

#### Permission Authorization Handler
```csharp
protected override Task HandleRequirementAsync(
    AuthorizationHandlerContext context, 
    PermissionRequirement requirement)
{
    if (context.User?.Identity?.IsAuthenticated != true)
    {
        return Task.CompletedTask;
    }

    // Check for "permissions" claim type (plural)
    var hasPermission = context.User.Claims.Any(c =>
        c.Type == "permissions" && c.Value == requirement.Permission);

    if (hasPermission)
    {
        context.Succeed(requirement);
    }

    return Task.CompletedTask;
}
```

### 2. OCPP Service (Token Validator)

#### JWT Configuration (`DependencyInjection.cs`)
```csharp
.AddJwtBearer(options =>
{
    options.SaveToken = true;
    options.RequireHttpsMetadata = true; // Set to false for development
    options.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuerSigningKey = true,
        IssuerSigningKey = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes(jwtSettings.SecretKey)),
        
        // Must match UM service issuer
        ValidateIssuer = true,
        ValidIssuer = jwtSettings.Issuer,
        
        // Service identifier
        ValidateAudience = true,
        ValidAudience = jwtSettings.Audience,
        
        ValidateLifetime = true,
        ClockSkew = TimeSpan.Zero,
        
        // Map claims properly
        NameClaimType = "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name",
        RoleClaimType = "http://schemas.microsoft.com/ws/2008/06/identity/claims/role"
    };

    // WebSocket support
    options.Events = new JwtBearerEvents
    {
        OnMessageReceived = context =>
        {
            var accessToken = context.Request.Query["access_token"];
            var path = context.HttpContext.Request.Path;
            
            if (!string.IsNullOrEmpty(accessToken) && 
                path.StartsWithSegments("/ocpp"))
            {
                context.Token = accessToken;
            }
            
            return Task.CompletedTask;
        }
    };
});
```

#### Permission Authorization Handler
```csharp
protected override Task HandleRequirementAsync(
    AuthorizationHandlerContext context,
    PermissionRequirement requirement)
{
    if (context.User?.Identity?.IsAuthenticated != true)
    {
        return Task.CompletedTask;
    }

    // Check for "permissions" claim type (must match UM)
    var hasPermission = context.User.Claims.Any(c =>
        c.Type == "permissions" && c.Value == requirement.Permission);

    if (hasPermission)
    {
        context.Succeed(requirement);
    }

    return Task.CompletedTask;
}
```

## Permission Structure

### Permission Naming Convention
```
{service}:{resource}:{action}

Examples:
- users:view
- users:create
- users:update
- users:delete
- ocpp:chargepoint:view
- ocpp:chargepoint:manage
- ocpp:chargepoint:control
- ocpp:tag:view
- ocpp:tag:manage
- ocpp:transaction:view
- ocpp:transaction:manage
```

### Role-Based Permissions

#### SystemAdmin
- Full access to all permissions across both services
- Users: view, create, update, delete, manage
- Roles: view, create, update, delete, manage, assign
- OCPP: All chargepoint, tag, transaction, reservation permissions
- System: audit:view, settings:manage, health:view

#### Admin
- User and role management (no delete)
- Document management
- OCPP: All operational permissions
- System: audit:view, health:view

#### Operator
- View users
- Document access
- OCPP: Full chargepoint, tag, transaction, reservation management

#### Support
- Read-only access
- View users, documents, OCPP resources

#### User
- Basic document access
- View chargepoints and transactions

## Configuration Checklist

### Recommended: User Secrets (Development)

For local development, use .NET User Secrets to keep configuration secure:

```bash
# UM Service Configuration
cd ChargeLK.UM/ChargeLK.API
dotnet user-secrets set "Jwt:SecretKey" "your-super-secret-key-must-be-at-least-32-characters-long"
dotnet user-secrets set "Jwt:Issuer" "https://api.chargelk.com/um"
dotnet user-secrets set "Jwt:Audience" "chargelk-services"
dotnet user-secrets set "Jwt:ExpirationInMinutes" "60"
dotnet user-secrets set "Jwt:RefreshTokenExpirationInDays" "7"

# OCPP Service Configuration (MUST match UM service)
cd ../../ChargeLK.OCPP/ChargeLK.API
dotnet user-secrets set "Jwt:SecretKey" "your-super-secret-key-must-be-at-least-32-characters-long"
dotnet user-secrets set "Jwt:Issuer" "https://api.chargelk.com/um"
dotnet user-secrets set "Jwt:Audience" "chargelk-services"
dotnet user-secrets set "Jwt:ExpirationInMinutes" "60"
dotnet user-secrets set "Jwt:RefreshTokenExpirationInDays" "7"
```

**Critical:** Both services MUST have identical `SecretKey`, `Issuer`, and `Audience` values!

### Alternative: appsettings.json (Production)

### UM Service (appsettings.json)
```json
{
  "Jwt": {
    "SecretKey": "MUST_BE_SAME_IN_BOTH_SERVICES",
    "Issuer": "https://api.chargelk.com/um",
    "Audience": "chargelk-services",
    "ExpirationInMinutes": 60,
    "RefreshTokenExpirationInDays": 7
  }
}
```

### OCPP Service (appsettings.json)
```json
{
  "Jwt": {
    "__comment": "JWT validation settings - must match UM service",
    "SecretKey": "MUST_BE_SAME_IN_BOTH_SERVICES",
    "Issuer": "https://api.chargelk.com/um",
    "Audience": "chargelk-services",
    "ExpirationInMinutes": 60,
    "RefreshTokenExpirationInDays": 7
  }
}
```

## Testing Authentication

### 1. Test Token Generation (UM Service)

```bash
# Login and get token
curl -X POST http://localhost:5001/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "admin@chargelk.com",
    "password": "Admin@123"
  }'

# Response:
# {
#   "accessToken": "eyJhbGc...",
#   "refreshToken": "...",
#   "expiresAt": "2025-12-20T12:00:00Z"
# }
```

### 2. Test Token Validation (OCPP Service)

```bash
# Use token to access OCPP service
curl -X GET http://localhost:5002/api/chargepoints \
  -H "Authorization: Bearer eyJhbGc..."

# Should return charge points if user has "ocpp:chargepoint:view" permission
```

### 3. Test WebSocket Authentication (OCPP)

```javascript
// Connect to WebSocket with token
const ws = new WebSocket(
  'ws://localhost:5002/ocpp/1.6?access_token=eyJhbGc...'
);
```

### 4. Verify Claims in Token

```bash
# Decode JWT token (use jwt.io or jwt-cli)
jwt decode eyJhbGc...

# Expected claims:
# {
#   "sub": "1",
#   "email": "admin@chargelk.com",
#   "name": "admin",
#   "role": ["SystemAdmin"],
#   "permissions": [
#     "users:view",
#     "users:create",
#     "ocpp:chargepoint:view",
#     "ocpp:chargepoint:manage",
#     ...
#   ],
#   "iss": "https://api.chargelk.com/um",
#   "aud": "chargelk-services",
#   "exp": 1703073600,
#   "iat": 1703070000
# }
```

## Security Best Practices

### 1. Secret Key Management
- ✅ Use strong secret key (minimum 32 characters)
- ✅ Store in environment variables or Azure Key Vault in production
- ✅ Never commit secrets to source control
- ✅ Rotate keys periodically

### 2. HTTPS/TLS
- ✅ Set `RequireHttpsMetadata = true` in production
- ✅ Use HTTPS for all API communication
- ✅ Use WSS (WebSocket Secure) for OCPP WebSocket connections

### 3. Token Expiration
- ✅ Access tokens: 60 minutes (short-lived)
- ✅ Refresh tokens: 7 days
- ✅ Implement token refresh flow in Admin UI
- ✅ Revoke refresh tokens on logout

### 4. CORS Configuration
- ✅ Restrict `AllowedOrigins` to known domains
- ✅ Don't use `*` in production
- ✅ Configure separately for each environment

### 5. Logging and Monitoring
- ✅ Log authentication failures
- ✅ Log authorization denials
- ✅ Monitor for suspicious token usage
- ✅ Set up alerts for high failure rates

## Troubleshooting

### Issue: 401 Unauthorized

**Possible Causes:**
1. Token expired
2. Token not included in request
3. Wrong `Authorization` header format (should be `Bearer {token}`)
4. JWT secret key mismatch between services

**Solution:**
```bash
# Check if token is valid
curl -X GET http://localhost:5001/api/auth/validate \
  -H "Authorization: Bearer {token}"

# Check token expiration
jwt decode {token} | grep exp

# Refresh token if expired
curl -X POST http://localhost:5001/api/auth/refresh \
  -H "Content-Type: application/json" \
  -d '{
    "accessToken": "{old_token}",
    "refreshToken": "{refresh_token}"
  }'
```

### Issue: 403 Forbidden

**Possible Causes:**
1. User doesn't have required permission
2. Permission claim type mismatch
3. Permission name mismatch

**Solution:**
```bash
# Decode token and check permissions
jwt decode {token}

# Verify user has required permission
# Check claims for "permissions" (plural) claim type
# Check if permission value matches exactly (e.g., "ocpp:chargepoint:view")
```

### Issue: Token Validation Fails

**Possible Causes:**
1. Issuer mismatch
2. Audience mismatch
3. Secret key mismatch
4. Token signature invalid

**Solution:**
1. Verify JWT configuration matches in both services
2. Check `Issuer` and `Audience` values
3. Ensure `SecretKey` is identical in both services
4. Check for typos or whitespace in configuration

## Future Enhancements

### 1. Database-Driven Permissions
- Move role permissions from code to database
- Use `cl_role_claims` table
- Dynamic permission assignment via Admin UI

### 2. API Gateway
- Consider adding an API Gateway (e.g., Ocelot, YARP)
- Centralized authentication/authorization
- Rate limiting and request routing

### 3. OAuth 2.0 / OpenID Connect
- Implement full OAuth 2.0 authorization server
- Support for multiple client applications
- OpenID Connect for standardized authentication

### 4. Multi-Tenancy
- Tenant isolation in JWT claims
- Tenant-specific permissions
- Cross-tenant access controls

### 5. Audit Trail
- Log all permission grants/revokes
- Track token usage per user/device
- Anomaly detection

## Summary

This implementation provides:
- ✅ Centralized authentication in UM service
- ✅ Distributed authorization across microservices
- ✅ JWT-based stateless authentication
- ✅ Fine-grained permission-based access control
- ✅ WebSocket authentication support
- ✅ Consistent claim structure
- ✅ Proper security configuration
- ✅ Comprehensive logging
- ✅ Production-ready setup

All services now work collaboratively with the UM service as the authentication authority and OCPP as a resource service that validates tokens.

