# Authentication & Authorization Testing Guide

## Pre-requisites

Before testing, ensure both services are running:

```bash
# Terminal 1 - Start UM Service
cd ChargeLK.UM/ChargeLK.API
dotnet run

# Terminal 2 - Start OCPP Service
cd ChargeLK.OCPP/ChargeLK.API
dotnet run
```

## Configuration Setup

### Option 1: Using User Secrets (Recommended for Development)

User Secrets keep sensitive configuration secure and out of source control.

#### Configure UM Service Secrets

```bash
cd ChargeLK.UM/ChargeLK.API

# Set database connection
dotnet user-secrets set "ConnectionStrings:DefaultConnection" "Host=localhost;Database=chargelk_um;Username=postgres;Password=your_password"

# Set JWT secret (at least 32 characters - MUST match OCPP service)
dotnet user-secrets set "Jwt:SecretKey" "your-super-secret-key-must-be-at-least-32-characters-long-for-production"

# Set JWT issuer and audience
dotnet user-secrets set "Jwt:Issuer" "https://api.chargelk.com/um"
dotnet user-secrets set "Jwt:Audience" "chargelk-services"

# Verify secrets
dotnet user-secrets list
```

#### Configure OCPP Service Secrets

```bash
cd ChargeLK.OCPP/ChargeLK.API

# Set database connection
dotnet user-secrets set "ConnectionStrings:DefaultConnection" "Host=localhost;Database=chargelk_ocpp;Username=postgres;Password=your_password"

# Set JWT secret (MUST be identical to UM service)
dotnet user-secrets set "Jwt:SecretKey" "your-super-secret-key-must-be-at-least-32-characters-long-for-production"

# Set JWT issuer and audience (MUST match UM service)
dotnet user-secrets set "Jwt:Issuer" "https://api.chargelk.com/um"
dotnet user-secrets set "Jwt:Audience" "chargelk-services"

# Verify secrets
dotnet user-secrets list
```

⚠️ **CRITICAL**: The `SecretKey`, `Issuer`, and `Audience` in user secrets MUST be identical in both services!

### Option 2: Using appsettings.json (Not Recommended)

## Configuration Verification

### 1. Verify JWT Configuration Match

Both services must have identical JWT settings:

**UM Service** (`ChargeLK.UM/ChargeLK.API/appsettings.json`):
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

**OCPP Service** (`ChargeLK.OCPP/ChargeLK.API/appsettings.json`):
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

⚠️ **CRITICAL**: The `SecretKey`, `Issuer`, and `Audience` MUST be identical in both services!

## Test Scenarios

### Test 1: User Authentication (UM Service)

#### 1.1 Login with Valid Credentials

```bash
curl -X POST http://localhost:5001/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "admin@chargelk.com",
    "password": "Admin@123"
  }'
```

**Expected Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "base64-encoded-refresh-token...",
    "expiresAt": "2025-12-20T13:00:00Z",
    "refreshExpiresAt": "2025-12-27T12:00:00Z"
  },
  "message": "Login successful"
}
```

#### 1.2 Decode and Verify Token Claims

Copy the `accessToken` from the response and decode it at https://jwt.io

**Expected Claims:**
```json
{
  "sub": "1",
  "email": "admin@chargelk.com",
  "name": "admin",
  "FirstName": "Admin",
  "LastName": "User",
  "http://schemas.microsoft.com/ws/2008/06/identity/claims/role": "SystemAdmin",
  "permissions": [
    "users:view",
    "users:create",
    "users:update",
    "users:delete",
    "users:manage",
    "roles:view",
    "roles:create",
    "roles:update",
    "roles:delete",
    "roles:manage",
    "roles:assign",
    "ocpp:chargepoint:view",
    "ocpp:chargepoint:manage",
    "ocpp:chargepoint:control",
    "ocpp:tag:view",
    "ocpp:tag:manage",
    "ocpp:transaction:view",
    "ocpp:transaction:manage",
    "ocpp:reservation:view",
    "ocpp:reservation:manage"
  ],
  "jti": "unique-token-id",
  "iat": 1703070000,
  "iss": "https://api.chargelk.com/um",
  "aud": "chargelk-services",
  "exp": 1703073600
}
```

#### 1.3 Login with Invalid Credentials

```bash
curl -X POST http://localhost:5001/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "admin@chargelk.com",
    "password": "WrongPassword"
  }'
```

**Expected Response (401 Unauthorized):**
```json
{
  "success": false,
  "message": "Invalid credentials"
}
```

### Test 2: Token Validation (UM Service)

#### 2.1 Access Protected Endpoint with Valid Token

```bash
# Replace {TOKEN} with the accessToken from login
curl -X GET http://localhost:5001/api/users \
  -H "Authorization: Bearer {TOKEN}"
```

**Expected Response (200 OK):**
```json
{
  "success": true,
  "data": [...],
  "message": "Users retrieved successfully"
}
```

#### 2.2 Access Protected Endpoint without Token

```bash
curl -X GET http://localhost:5001/api/users
```

**Expected Response (401 Unauthorized):**
```json
{
  "type": "https://tools.ietf.org/html/rfc7235#section-3.1",
  "title": "Unauthorized",
  "status": 401
}
```

#### 2.3 Access Protected Endpoint with Expired Token

```bash
# Use an old/expired token
curl -X GET http://localhost:5001/api/users \
  -H "Authorization: Bearer {EXPIRED_TOKEN}"
```

**Expected Response (401 Unauthorized):**
```json
{
  "type": "https://tools.ietf.org/html/rfc7235#section-3.1",
  "title": "Unauthorized",
  "status": 401
}
```

### Test 3: Cross-Service Authentication (OCPP Service)

#### 3.1 Access OCPP Endpoint with UM-Issued Token

```bash
# Use the token from UM service login
curl -X GET http://localhost:5002/api/chargepoints \
  -H "Authorization: Bearer {TOKEN}"
```

**Expected Response (200 OK):**
```json
{
  "success": true,
  "data": [...],
  "message": "Charge points retrieved successfully"
}
```

This proves that:
- ✅ OCPP service validates tokens issued by UM service
- ✅ JWT secret key matches between services
- ✅ Issuer/Audience validation works correctly

#### 3.2 Access OCPP Endpoint without Permission

```bash
# Login as a User with limited permissions
curl -X POST http://localhost:5001/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@chargelk.com",
    "password": "User@123"
  }'

# Try to manage charge points (requires ocpp:chargepoint:manage)
curl -X PUT http://localhost:5002/api/chargepoints/{id} \
  -H "Authorization: Bearer {USER_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Updated Name"
  }'
```

**Expected Response (403 Forbidden):**
```json
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.5.3",
  "title": "Forbidden",
  "status": 403
}
```

### Test 4: Permission-Based Authorization

#### 4.1 SystemAdmin Access (Full Permissions)

```bash
# Login as SystemAdmin
TOKEN=$(curl -s -X POST http://localhost:5001/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "admin@chargelk.com", "password": "Admin@123"}' \
  | jq -r '.data.accessToken')

# Test UM permissions
curl -X GET http://localhost:5001/api/users \
  -H "Authorization: Bearer $TOKEN"
# Expected: 200 OK

curl -X POST http://localhost:5001/api/users \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"email": "newuser@test.com", ...}'
# Expected: 200 OK or 201 Created

# Test OCPP permissions
curl -X GET http://localhost:5002/api/chargepoints \
  -H "Authorization: Bearer $TOKEN"
# Expected: 200 OK

curl -X POST http://localhost:5002/api/chargepoints \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "New Charger", ...}'
# Expected: 200 OK or 201 Created
```

#### 4.2 Operator Access (Limited Permissions)

```bash
# Login as Operator
TOKEN=$(curl -s -X POST http://localhost:5001/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "operator@chargelk.com", "password": "Operator@123"}' \
  | jq -r '.data.accessToken')

# Can view users (has users:view)
curl -X GET http://localhost:5001/api/users \
  -H "Authorization: Bearer $TOKEN"
# Expected: 200 OK

# Cannot create users (no users:create)
curl -X POST http://localhost:5001/api/users \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"email": "newuser@test.com", ...}'
# Expected: 403 Forbidden

# Can manage charge points (has ocpp:chargepoint:manage)
curl -X GET http://localhost:5002/api/chargepoints \
  -H "Authorization: Bearer $TOKEN"
# Expected: 200 OK

curl -X PUT http://localhost:5002/api/chargepoints/1 \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "Updated", ...}'
# Expected: 200 OK
```

#### 4.3 Support Access (Read-Only)

```bash
# Login as Support
TOKEN=$(curl -s -X POST http://localhost:5001/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "support@chargelk.com", "password": "Support@123"}' \
  | jq -r '.data.accessToken')

# Can view (has users:view, ocpp:chargepoint:view)
curl -X GET http://localhost:5001/api/users \
  -H "Authorization: Bearer $TOKEN"
# Expected: 200 OK

curl -X GET http://localhost:5002/api/chargepoints \
  -H "Authorization: Bearer $TOKEN"
# Expected: 200 OK

# Cannot modify (no manage permissions)
curl -X PUT http://localhost:5002/api/chargepoints/1 \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "Updated", ...}'
# Expected: 403 Forbidden
```

### Test 5: Token Refresh Flow

#### 5.1 Refresh Access Token

```bash
# Login first
LOGIN_RESPONSE=$(curl -s -X POST http://localhost:5001/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "admin@chargelk.com", "password": "Admin@123"}')

ACCESS_TOKEN=$(echo $LOGIN_RESPONSE | jq -r '.data.accessToken')
REFRESH_TOKEN=$(echo $LOGIN_RESPONSE | jq -r '.data.refreshToken')

# Wait for access token to expire (or use an expired one)
# Then refresh
curl -X POST http://localhost:5001/api/auth/refresh \
  -H "Content-Type: application/json" \
  -d "{
    \"accessToken\": \"$ACCESS_TOKEN\",
    \"refreshToken\": \"$REFRESH_TOKEN\"
  }"
```

**Expected Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "accessToken": "new-access-token...",
    "refreshToken": "new-refresh-token...",
    "expiresAt": "...",
    "refreshExpiresAt": "..."
  },
  "message": "Token refreshed successfully"
}
```

### Test 6: Logout (Token Revocation)

#### 6.1 Revoke Tokens

```bash
curl -X POST http://localhost:5001/api/auth/logout \
  -H "Content-Type: application/json" \
  -d "{
    \"accessToken\": \"$ACCESS_TOKEN\",
    \"refreshToken\": \"$REFRESH_TOKEN\"
  }"
```

**Expected Response (200 OK):**
```json
{
  "success": true,
  "message": "Logged out successfully"
}
```

#### 6.2 Verify Token is Revoked

```bash
# Try to use the same token
curl -X GET http://localhost:5001/api/users \
  -H "Authorization: Bearer $ACCESS_TOKEN"
# Expected: 401 Unauthorized (token still valid until expiry)

# Try to refresh with revoked refresh token
curl -X POST http://localhost:5001/api/auth/refresh \
  -H "Content-Type: application/json" \
  -d "{
    \"accessToken\": \"$ACCESS_TOKEN\",
    \"refreshToken\": \"$REFRESH_TOKEN\"
  }"
# Expected: 400 Bad Request or 401 Unauthorized
```

### Test 7: WebSocket Authentication (OCPP)

```javascript
// JavaScript/TypeScript example for Admin UI
const token = localStorage.getItem('accessToken');

// Connect to OCPP WebSocket with token in query string
const ws = new WebSocket(
  `ws://localhost:5002/ocpp/1.6?access_token=${token}`
);

ws.onopen = () => {
  console.log('WebSocket connected with JWT authentication');
};

ws.onerror = (error) => {
  console.error('WebSocket authentication failed:', error);
};
```

### Test 8: Claim Verification

#### 8.1 Verify Permission Claims in Token

```bash
# Install jwt-cli: cargo install jwt-cli (or use jwt.io)

# Decode token
jwt decode $ACCESS_TOKEN

# Check for permissions claim
jwt decode $ACCESS_TOKEN | jq '.permissions'

# Expected output:
# [
#   "users:view",
#   "users:create",
#   "ocpp:chargepoint:view",
#   ...
# ]
```

#### 8.2 Verify Claims are Used in Authorization

Check OCPP service logs when accessing protected endpoints:

```bash
# Make a request
curl -X GET http://localhost:5002/api/chargepoints \
  -H "Authorization: Bearer $TOKEN"

# Check OCPP service logs (Terminal 2)
# Should see:
# [Debug] OCPP User admin authorized with permission ocpp:chargepoint:view
```

If permission is missing:
```bash
# Should see:
# [Warning] OCPP User username does not have required permission ocpp:chargepoint:view. 
# Available permissions: users:view, users:create, ...
```

## Troubleshooting

### Issue 1: 401 Unauthorized from OCPP Service

**Symptoms:**
- UM service works fine
- OCPP service returns 401 with UM-issued token

**Diagnosis:**
```bash
# Check if JWT configs match
diff ChargeLK.UM/ChargeLK.API/appsettings.json \
     ChargeLK.OCPP/ChargeLK.API/appsettings.json

# Verify token claims
jwt decode $TOKEN | jq '{iss, aud, exp}'
```

**Common Causes:**
1. ✅ `SecretKey` mismatch between services
2. ✅ `Issuer` mismatch
3. ✅ `Audience` mismatch
4. ✅ Token expired (check `exp` claim)
5. ✅ Clock skew between services

**Solution:**
Ensure both services have identical JWT configuration.

### Issue 2: 403 Forbidden

**Symptoms:**
- Token validates (401 → 200)
- But access is denied (403)

**Diagnosis:**
```bash
# Decode token and check permissions
jwt decode $TOKEN | jq '.permissions'

# Check what permission is required
# Look at controller endpoint authorization policy
```

**Common Causes:**
1. ✅ User doesn't have required permission
2. ✅ Permission claim type mismatch ("permission" vs "permissions")
3. ✅ Permission name mismatch ("ocpp:chargepoint:view" vs "ocpp:chargepoint:read")

**Solution:**
1. Verify permission is in token claims
2. Grant missing permission to user's role
3. Check PermissionAuthorizationHandler logs

### Issue 3: Empty Permission Claims

**Symptoms:**
- Token has user info but no permissions
- All endpoints return 403

**Diagnosis:**
```bash
jwt decode $TOKEN | jq '.permissions'
# Returns: null or []
```

**Common Causes:**
1. ✅ Role not assigned to user
2. ✅ Role permissions not configured
3. ✅ `TokenService.GetPermissionsForRole()` returning empty list

**Solution:**
```sql
-- Check user's roles
SELECT r.name FROM cl_roles r
JOIN cl_user_roles ur ON r.id = ur.role_id
JOIN cl_users u ON ur.user_id = u.id
WHERE u.email = 'user@example.com';

-- If no roles, assign one
INSERT INTO cl_user_roles (user_id, role_id)
VALUES (
  (SELECT id FROM cl_users WHERE email = 'user@example.com'),
  (SELECT id FROM cl_roles WHERE name = 'Operator')
);
```

### Issue 4: Token Refresh Fails

**Symptoms:**
- Login works
- Refresh returns 400/401

**Diagnosis:**
```bash
# Check refresh token in database
# Connect to PostgreSQL
psql -h localhost -U postgres -d chargelk_um

SELECT * FROM cl_user_tokens
WHERE user_id = (SELECT id FROM cl_users WHERE email = 'user@example.com');
```

**Common Causes:**
1. ✅ Refresh token doesn't exist in database
2. ✅ Refresh token expired
3. ✅ Refresh token already used (single-use tokens)
4. ✅ Access token and refresh token don't match

**Solution:**
Login again to get fresh tokens.

## Automated Testing Script

Save as `test-auth.sh`:

```bash
#!/bin/bash

# Configuration
UM_URL="http://localhost:5001"
OCPP_URL="http://localhost:5002"

# Colors
GREEN='\033[0;32m'
RED='\033[0;31m'
NC='\033[0m' # No Color

# Test 1: Login
echo "Test 1: Login to UM Service"
LOGIN_RESPONSE=$(curl -s -X POST $UM_URL/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "admin@chargelk.com", "password": "Admin@123"}')

if echo $LOGIN_RESPONSE | jq -e '.success' > /dev/null 2>&1; then
  echo -e "${GREEN}✓ Login successful${NC}"
  TOKEN=$(echo $LOGIN_RESPONSE | jq -r '.data.accessToken')
else
  echo -e "${RED}✗ Login failed${NC}"
  exit 1
fi

# Test 2: Access UM Protected Endpoint
echo "Test 2: Access UM Protected Endpoint"
UM_RESPONSE=$(curl -s -w "%{http_code}" -o /dev/null \
  -H "Authorization: Bearer $TOKEN" \
  $UM_URL/api/users)

if [ "$UM_RESPONSE" == "200" ]; then
  echo -e "${GREEN}✓ UM authentication works${NC}"
else
  echo -e "${RED}✗ UM authentication failed (HTTP $UM_RESPONSE)${NC}"
fi

# Test 3: Access OCPP Protected Endpoint (Cross-Service)
echo "Test 3: Access OCPP Protected Endpoint with UM Token"
OCPP_RESPONSE=$(curl -s -w "%{http_code}" -o /dev/null \
  -H "Authorization: Bearer $TOKEN" \
  $OCPP_URL/api/chargepoints)

if [ "$OCPP_RESPONSE" == "200" ]; then
  echo -e "${GREEN}✓ Cross-service authentication works${NC}"
else
  echo -e "${RED}✗ Cross-service authentication failed (HTTP $OCPP_RESPONSE)${NC}"
fi

# Test 4: Verify Token Claims
echo "Test 4: Verify Token Claims"
if command -v jwt &> /dev/null; then
  PERMISSIONS=$(jwt decode $TOKEN | jq -r '.permissions | length')
  if [ "$PERMISSIONS" -gt 0 ]; then
    echo -e "${GREEN}✓ Token has $PERMISSIONS permissions${NC}"
  else
    echo -e "${RED}✗ Token has no permissions${NC}"
  fi
else
  echo "⚠ jwt-cli not installed, skipping claim verification"
fi

echo ""
echo "All tests completed!"
```

Run it:
```bash
chmod +x test-auth.sh
./test-auth.sh
```

## Summary

✅ **What We've Tested:**
1. User authentication via UM service
2. JWT token generation with permissions
3. Token validation in UM service
4. Cross-service authentication (UM → OCPP)
5. Permission-based authorization
6. Token refresh flow
7. Token revocation (logout)
8. WebSocket authentication
9. Claim verification

✅ **What Works:**
- UM service issues JWT tokens with user info and permissions
- OCPP service validates JWT tokens issued by UM
- Permission-based authorization works across both services
- Roles determine which permissions users have
- WebSocket connections can authenticate with JWT tokens

✅ **Key Principles:**
- UM service is the **authentication authority** (issues tokens)
- OCPP service is a **resource service** (validates tokens)
- Both services share the same JWT secret and configuration
- Permissions are embedded in JWT tokens as claims
- Authorization decisions are made based on permission claims

