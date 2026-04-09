# Authentication Quick Reference

## 🚀 Starting Services

```bash
# Terminal 1 - UM Service
cd ChargeLK.UM/ChargeLK.API && dotnet run

# Terminal 2 - OCPP Service  
cd ChargeLK.OCPP/ChargeLK.API && dotnet run
```

## 🔑 Quick Test

```bash
# 1. Login
curl -X POST http://localhost:5001/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "admin@chargelk.com", "password": "Admin@123"}'

# 2. Save token (replace {TOKEN} with accessToken from response)
TOKEN="eyJhbGc..."

# 3. Test UM endpoint
curl -H "Authorization: Bearer $TOKEN" http://localhost:5001/api/users

# 4. Test OCPP endpoint (cross-service auth)
curl -H "Authorization: Bearer $TOKEN" http://localhost:5002/api/chargepoints
```

## ⚙️ Configuration

### Both services need IDENTICAL JWT settings:

```json
{
  "Jwt": {
    "SecretKey": "MUST_BE_SAME",
    "Issuer": "https://api.chargelk.com/um",
    "Audience": "chargelk-services"
  }
}
```

## 🔐 Permission Structure

Format: `{service}:{resource}:{action}`

### UM Service Permissions
- `users:view` - View users
- `users:create` - Create users
- `users:update` - Update users
- `users:delete` - Delete users
- `roles:view` - View roles
- `roles:manage` - Manage roles

### OCPP Service Permissions
- `ocpp:chargepoint:view` - View charge points
- `ocpp:chargepoint:manage` - Manage charge points
- `ocpp:chargepoint:control` - Control charge points
- `ocpp:tag:view` - View RFID tags
- `ocpp:tag:manage` - Manage RFID tags
- `ocpp:transaction:view` - View transactions
- `ocpp:transaction:manage` - Manage transactions

## 👥 Default Roles

### SystemAdmin
Full access to everything

### Admin  
User/role management + OCPP operations

### Operator
OCPP operations + view users

### Support
Read-only access

### User
Basic document and view access

## 🐛 Troubleshooting

### 401 Unauthorized
```bash
# Check JWT config matches
diff <(cat ChargeLK.UM/ChargeLK.API/appsettings.json | jq '.Jwt') \
     <(cat ChargeLK.OCPP/ChargeLK.API/appsettings.json | jq '.Jwt')
```

### 403 Forbidden
```bash
# Check permissions in token
jwt decode $TOKEN | jq '.permissions'

# Grant permission
psql -d chargelk_um -c "
INSERT INTO cl_user_roles (user_id, role_id)
VALUES (
  (SELECT id FROM cl_users WHERE email = 'user@example.com'),
  (SELECT id FROM cl_roles WHERE name = 'Operator')
);"
```

### No Permissions in Token
```sql
-- Check user roles
SELECT r.name FROM cl_roles r
JOIN cl_user_roles ur ON r.id = ur.role_id
JOIN cl_users u ON ur.user_id = u.id
WHERE u.email = 'user@example.com';

-- Assign role if missing
INSERT INTO cl_user_roles (user_id, role_id)
VALUES (
  (SELECT id FROM cl_users WHERE email = 'user@example.com'),
  (SELECT id FROM cl_roles WHERE name = 'Operator')
);
```

## 📝 Code Examples

### Backend - Protect Endpoint
```csharp
[Authorize(Policy = "CanViewChargePoints")]
[HttpGet]
public async Task<IActionResult> GetChargePoints()
{
    // Only users with ocpp:chargepoint:view permission can access
}
```

### Frontend - API Call
```typescript
// Store token after login
localStorage.setItem('accessToken', response.data.accessToken);

// Use token in requests
const response = await fetch('http://localhost:5002/api/chargepoints', {
  headers: {
    'Authorization': `Bearer ${localStorage.getItem('accessToken')}`
  }
});
```

### WebSocket Authentication
```typescript
const token = localStorage.getItem('accessToken');
const ws = new WebSocket(
  `ws://localhost:5002/ocpp/1.6?access_token=${token}`
);
```

## 📚 Full Documentation

- **MICROSERVICES-AUTHENTICATION-IMPLEMENTATION.md** - Complete guide
- **AUTHENTICATION-TESTING-GUIDE.md** - Testing procedures
- **AUTHENTICATION-FIX-SUMMARY.md** - Change details
- **AUTHENTICATION-COMPLETE.md** - Status overview

## ✅ Status

**Authentication is fully implemented and working!**

```
✅ UM service issues JWT tokens
✅ OCPP service validates tokens
✅ Permissions embedded in tokens
✅ Authorization works cross-service
✅ Both services build successfully
```

## 🎯 Key Files Modified

```
ChargeLK.UM/
  ├─ ChargeLK.Domain/Constants/CustomClaimTypes.cs ★NEW
  ├─ ChargeLK.Infrastructure/Authorization/PermissionAuthorizationHandler.cs
  ├─ ChargeLK.Infrastructure/Services/Token/TokenService.cs
  ├─ ChargeLK.Infrastructure/Services/Permission/PermissionClaimsTransformation.cs
  └─ ChargeLK.Infrastructure/Services/User/CurrentUserService.cs

ChargeLK.OCPP/
  ├─ ChargeLK.Domain/Constants/CustomClaimTypes.cs ★NEW
  ├─ ChargeLK.Domain/Constants/AuthenticationSchemes.cs ★NEW
  ├─ ChargeLK.Infrastructure/Authorization/PermissionAuthorizationHandler.cs
  └─ ChargeLK.Infrastructure/Services/User/CurrentUserService.cs
```

---

*Quick Reference v1.0 - December 20, 2025*

