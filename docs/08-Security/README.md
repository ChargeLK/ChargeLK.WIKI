# Security Documentation

## Overview

This section contains comprehensive security documentation for the ChargeLK system, including authentication, authorization, and permission management.

## 📚 Documentation Index

### Authentication & Authorization

1. **[Authentication & Authorization Implementation Guide](./AUTHENTICATION-AUTHORIZATION.md)**
   - Complete implementation overview
   - JWT token structure and claims
   - Roles and permissions reference
   - Authorization policies
   - API usage examples
   - Production security recommendations

2. **[Authentication Complete Guide](./AUTHENTICATION-COMPLETE.md)**
   - Complete authentication implementation overview
   - JWT token configuration
   - User authentication flow
   - Best practices and troubleshooting

3. **[Authentication Flow Diagrams](./AUTHENTICATION-FLOW-DIAGRAMS.md)**
   - Visual flow diagrams for authentication
   - Login process
   - Token generation and validation
   - Refresh token flow
   - WebSocket authentication

3. **[Microservices Authentication Implementation](./MICROSERVICES-AUTHENTICATION-IMPLEMENTATION.md)**
   - Cross-service authentication architecture
   - JWT token validation between services
   - UM Service (token issuer) configuration
   - OCPP Service (token validator) configuration
   - Implementation details and code examples

4. **[Authentication Testing Guide](./AUTHENTICATION-TESTING-GUIDE.md)**
   - Test scenarios for authentication flows
   - API endpoint testing with Postman/curl
   - Token validation testing
   - Common issues and solutions

5. **[Quick Reference - Authentication](./QUICK-REFERENCE-AUTH.md)**
   - Quick lookup guide for common authentication tasks
   - JWT configuration reference
   - Common API endpoints
   - Troubleshooting tips

### Permission Management

6. **[Permission Management SQL](./PERMISSION-MANAGEMENT-SQL.md)**
   - Database schema for permissions
   - SQL scripts for role and permission management
   - User role assignments
   - Permission queries

7. **[Permission Practical Examples](./PERMISSION-PRACTICAL-EXAMPLES.md)**
   - Real-world permission management scenarios
   - Onboarding new users
   - Role assignments
   - Permission auditing
   - Common permission patterns

### Security Checklist

8. **[Security Implementation Checklist](./SECURITY-IMPLEMENTATION-CHECKLIST.md)**
   - Pre-deployment security checklist
   - Configuration validation
   - Security best practices
   - Production readiness verification

## Key Concepts

### JWT-Based Authentication

ChargeLK uses JSON Web Tokens (JWT) for stateless authentication:

- **Access Token**: Short-lived token (60 minutes) for API requests
- **Refresh Token**: Long-lived token (7 days) for obtaining new access tokens
- **Token Claims**: User identity, roles, and permissions embedded in token

### Microservices Architecture

```
┌─────────┐         ┌──────────┐         ┌──────────┐
│  Admin  │────────>│    UM    │         │   OCPP   │
│   UI    │<────────│  Service │         │ Service  │
└─────────┘         └──────────┘         └──────────┘
                     (Token Issuer)      (Token Validator)
```

- **UM Service**: Issues JWT tokens, manages users and permissions
- **OCPP Service**: Validates JWT tokens, enforces permissions

### Permission-Based Access Control (PBAC)

ChargeLK implements fine-grained permission control:

- **Roles**: Collections of permissions (Admin, Operator, Viewer)
- **Permissions**: Specific actions (chargepoints.read, transactions.write, etc.)
- **Claims**: Permission information included in JWT tokens

## Quick Start

### 1. Basic Authentication Flow

```bash
# Login and get tokens
curl -X POST http://localhost:5001/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "admin@chargelk.com",
    "password": "Admin@123"
  }'
```

### 2. Use Access Token

```bash
# Call protected endpoint
curl -X GET http://localhost:5002/api/chargepoints \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

### 3. Refresh Token

```bash
# Get new access token
curl -X POST http://localhost:5001/api/auth/refresh-token \
  -H "Content-Type: application/json" \
  -d '{
    "refreshToken": "YOUR_REFRESH_TOKEN"
  }'
```

## Security Best Practices

### Production Configuration

1. **Strong Secret Keys**: Use 256-bit cryptographically secure random keys
2. **HTTPS Only**: Never transmit tokens over HTTP in production
3. **Token Expiration**: Keep access tokens short-lived (15-60 minutes)
4. **Refresh Token Rotation**: Rotate refresh tokens on each use
5. **CORS Configuration**: Whitelist only trusted origins
6. **Rate Limiting**: Implement rate limiting on auth endpoints
7. **Audit Logging**: Log all authentication and authorization events

### Common Security Issues

- ❌ Storing tokens in localStorage (XSS vulnerable)
- ❌ Same JWT secret in development and production
- ❌ Long-lived access tokens
- ❌ No token validation in microservices
- ❌ Exposing sensitive data in JWT claims

### Recommended Approach

- ✅ Store tokens in httpOnly cookies (for web apps)
- ✅ Unique, strong JWT secrets per environment
- ✅ Short-lived access tokens (60 minutes)
- ✅ Validate tokens in all microservices
- ✅ Include only necessary data in JWT claims

## Related Documentation

- [Development Guide](../07-Development/README.md)
- [API Reference](../04-API-Reference/README.md)
- [Infrastructure Guide](../06-Infrastructure/README.md)

## Support

For security issues or questions:
- Check [Authentication Testing Guide](./AUTHENTICATION-TESTING-GUIDE.md)
- Review [Quick Reference](./QUICK-REFERENCE-AUTH.md)
- See [Practical Examples](./PERMISSION-PRACTICAL-EXAMPLES.md)

