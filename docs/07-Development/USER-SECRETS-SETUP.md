# .NET User Secrets Setup Guide

## Overview

.NET User Secrets provide a secure way to store sensitive application configuration during development. Secrets are stored outside your project directory and are not committed to source control.

## Why Use User Secrets?

✅ **Security**: Keep sensitive data out of source control  
✅ **Flexibility**: Different configurations per developer  
✅ **Simplicity**: Easy to set up and manage  
✅ **Override**: Automatically overrides appsettings.json values  
✅ **Development Only**: Perfect for local development environments

## System Requirements

- .NET 9.0 SDK
- ChargeLK solution with UserSecretsId configured in .csproj files

## Quick Start

### 1. Verify User Secrets Configuration

Both services already have UserSecretsId configured:

**UM Service** (`ChargeLK.UM/ChargeLK.API/ChargeLK.API.csproj`):
```xml
<UserSecretsId>6923fadb-6223-4d79-bb1c-ffafd7c7c9ff</UserSecretsId>
```

**OCPP Service** (`ChargeLK.OCPP/ChargeLK.API/ChargeLK.API.csproj`):
```xml
<UserSecretsId>642bedcf-406c-4407-8b67-80696d2b549f</UserSecretsId>
```

### 2. Automated Setup (Recommended)

Use the provided setup script for quick configuration:

```bash
cd ChargeLK
chmod +x setup-secrets.sh
./setup-secrets.sh
```

### 3. Manual Setup

If you prefer manual configuration, follow these steps:

#### Step 1: Configure UM Service

```bash
cd ChargeLK.UM/ChargeLK.API

# Database Configuration
dotnet user-secrets set "ConnectionStrings:DefaultConnection" "Host=localhost;Database=chargelk_um;Username=postgres;Password=your_password"

# JWT Configuration (32+ characters required)
dotnet user-secrets set "Jwt:SecretKey" "your-development-secret-key-must-be-minimum-32-characters-long"
dotnet user-secrets set "Jwt:Issuer" "https://localhost:5002"
dotnet user-secrets set "Jwt:Audience" "chargelk-services"
dotnet user-secrets set "Jwt:ExpirationInMinutes" "60"
dotnet user-secrets set "Jwt:RefreshTokenExpirationInDays" "7"

# MongoDB Configuration (if using MongoDB)
dotnet user-secrets set "MongoDbSettings:ConnectionString" "mongodb://localhost:27017"
dotnet user-secrets set "MongoDbSettings:DatabaseName" "chargelk_um"

# Azure Blob Storage (for local development with Azurite)
dotnet user-secrets set "ConnectionStrings:AzureBlobStorage" "UseDevelopmentStorage=true"

# Google OAuth (optional)
dotnet user-secrets set "GoogleAuth:ClientId" "your-google-client-id.apps.googleusercontent.com"
dotnet user-secrets set "GoogleAuth:ClientSecret" "your-google-client-secret"

# Azure Service Bus (for local development)
dotnet user-secrets set "AzureServiceBus:ConnectionString" "Endpoint=sb://localhost;..."
```

#### Step 2: Configure OCPP Service

```bash
cd ../../ChargeLK.OCPP/ChargeLK.API

# Database Configuration
dotnet user-secrets set "ConnectionStrings:DefaultConnection" "Host=localhost;Database=chargelk_ocpp;Username=postgres;Password=your_password"

# JWT Configuration (MUST match UM service exactly)
dotnet user-secrets set "Jwt:SecretKey" "your-development-secret-key-must-be-minimum-32-characters-long"
dotnet user-secrets set "Jwt:Issuer" "https://localhost:5002"
dotnet user-secrets set "Jwt:Audience" "chargelk-services"
dotnet user-secrets set "Jwt:ExpirationInMinutes" "60"
dotnet user-secrets set "Jwt:RefreshTokenExpirationInDays" "7"

# MongoDB Configuration
dotnet user-secrets set "MongoDbSettings:ConnectionString" "mongodb://localhost:27017"
dotnet user-secrets set "MongoDbSettings:DatabaseName" "chargelk_ocpp"

# Azure Blob Storage
dotnet user-secrets set "ConnectionStrings:AzureBlobStorage" "UseDevelopmentStorage=true"

# Azure Service Bus
dotnet user-secrets set "AzureServiceBus:ConnectionString" "Endpoint=sb://localhost;..."
```

## Complete Setup Script

Save as `setup-secrets.sh` in the project root:

```bash
#!/bin/bash

# ChargeLK User Secrets Setup Script
# This script configures user secrets for both UM and OCPP services

set -e

# Colors for output
GREEN='\033[0;32m'
BLUE='\033[0;34m'
YELLOW='\033[1;33m'
RED='\033[0;31m'
NC='\033[0m' # No Color

echo -e "${BLUE}╔════════════════════════════════════════════════════════════╗${NC}"
echo -e "${BLUE}║         ChargeLK User Secrets Setup                       ║${NC}"
echo -e "${BLUE}╚════════════════════════════════════════════════════════════╝${NC}"
echo ""

# Check if .NET SDK is installed
if ! command -v dotnet &> /dev/null; then
    echo -e "${RED}Error: .NET SDK is not installed${NC}"
    echo "Please install .NET 9.0 SDK from: https://dotnet.microsoft.com/download"
    exit 1
fi

echo -e "${BLUE}Step 1: Database Configuration${NC}"
read -sp "Enter PostgreSQL password for development: " DB_PASSWORD
echo ""

echo -e "${BLUE}Step 2: Generate JWT Secret Key${NC}"
# Generate a secure 64-character JWT secret
JWT_SECRET=$(openssl rand -base64 48 | tr -d '\n')
echo -e "${GREEN}✓ Generated secure JWT secret key${NC}"
echo ""

echo -e "${BLUE}Step 3: Configuring UM Service...${NC}"
cd ChargeLK.UM/ChargeLK.API

# Database
dotnet user-secrets set "ConnectionStrings:DefaultConnection" "Host=localhost;Database=chargelk_um;Username=postgres;Password=$DB_PASSWORD"

# JWT
dotnet user-secrets set "Jwt:SecretKey" "$JWT_SECRET"
dotnet user-secrets set "Jwt:Issuer" "https://localhost:5002"
dotnet user-secrets set "Jwt:Audience" "chargelk-services"
dotnet user-secrets set "Jwt:ExpirationInMinutes" "60"
dotnet user-secrets set "Jwt:RefreshTokenExpirationInDays" "7"

# MongoDB
dotnet user-secrets set "MongoDbSettings:ConnectionString" "mongodb://localhost:27017"
dotnet user-secrets set "MongoDbSettings:DatabaseName" "chargelk_um"

# Azure Blob Storage (Azurite for local development)
dotnet user-secrets set "ConnectionStrings:AzureBlobStorage" "UseDevelopmentStorage=true"

echo -e "${GREEN}✓ UM Service secrets configured${NC}"
echo ""

echo -e "${BLUE}Step 4: Configuring OCPP Service...${NC}"
cd ../../ChargeLK.OCPP/ChargeLK.API

# Database
dotnet user-secrets set "ConnectionStrings:DefaultConnection" "Host=localhost;Database=chargelk_ocpp;Username=postgres;Password=$DB_PASSWORD"

# JWT (must match UM service)
dotnet user-secrets set "Jwt:SecretKey" "$JWT_SECRET"
dotnet user-secrets set "Jwt:Issuer" "https://localhost:5002"
dotnet user-secrets set "Jwt:Audience" "chargelk-services"
dotnet user-secrets set "Jwt:ExpirationInMinutes" "60"
dotnet user-secrets set "Jwt:RefreshTokenExpirationInDays" "7"

# MongoDB
dotnet user-secrets set "MongoDbSettings:ConnectionString" "mongodb://localhost:27017"
dotnet user-secrets set "MongoDbSettings:DatabaseName" "chargelk_ocpp"

# Azure Blob Storage
dotnet user-secrets set "ConnectionStrings:AzureBlobStorage" "UseDevelopmentStorage=true"

echo -e "${GREEN}✓ OCPP Service secrets configured${NC}"
echo ""

echo -e "${GREEN}╔════════════════════════════════════════════════════════════╗${NC}"
echo -e "${GREEN}║           User Secrets Setup Complete!                    ║${NC}"
echo -e "${GREEN}╚════════════════════════════════════════════════════════════╝${NC}"
echo ""
echo -e "${YELLOW}Important Information:${NC}"
echo -e "  JWT Secret Key: ${BLUE}${JWT_SECRET:0:20}...${NC} (truncated)"
echo ""
echo -e "${YELLOW}Next Steps:${NC}"
echo "  1. Ensure PostgreSQL is running: sudo systemctl start postgresql"
echo "  2. Ensure MongoDB is running: sudo systemctl start mongod"
echo "  3. Run database migrations:"
echo "     cd ChargeLK.UM/ChargeLK.API && dotnet ef database update"
echo "     cd ChargeLK.OCPP/ChargeLK.API && dotnet ef database update"
echo "  4. Start the services:"
echo "     Terminal 1: cd ChargeLK.UM/ChargeLK.API && dotnet run"
echo "     Terminal 2: cd ChargeLK.OCPP/ChargeLK.API && dotnet run"
echo ""
echo -e "${YELLOW}To view configured secrets:${NC}"
echo "  cd ChargeLK.UM/ChargeLK.API && dotnet user-secrets list"
echo "  cd ChargeLK.OCPP/ChargeLK.API && dotnet user-secrets list"
echo ""
```

## Managing User Secrets

### List All Secrets

```bash
cd ChargeLK.UM/ChargeLK.API
dotnet user-secrets list
```

### Update a Secret

```bash
dotnet user-secrets set "Jwt:SecretKey" "new-secret-key-value"
```

### Remove a Specific Secret

```bash
dotnet user-secrets remove "Jwt:SecretKey"
```

### Clear All Secrets

```bash
dotnet user-secrets clear
```

### View Secrets File Location

```bash
dotnet user-secrets list --json
```

## Secret Storage Locations

User secrets are stored in the following locations:

### Linux/macOS
```
~/.microsoft/usersecrets/<user_secrets_id>/secrets.json
```

**UM Service:**
```
~/.microsoft/usersecrets/6923fadb-6223-4d79-bb1c-ffafd7c7c9ff/secrets.json
```

**OCPP Service:**
```
~/.microsoft/usersecrets/642bedcf-406c-4407-8b67-80696d2b549f/secrets.json
```

### Windows
```
%APPDATA%\Microsoft\UserSecrets\<user_secrets_id>\secrets.json
```

## Configuration Priority

.NET Core configuration providers are loaded in this order (later overrides earlier):

1. appsettings.json
2. appsettings.{Environment}.json
3. User Secrets (Development environment only)
4. Environment Variables
5. Command-line arguments

This means User Secrets will override appsettings.json but not environment variables.

## Important Configuration Values

### JWT Configuration

⚠️ **Critical Requirements:**

1. **SecretKey**: 
   - Minimum 32 characters
   - Recommended: 64+ characters
   - MUST be identical in both UM and OCPP services
   - Use `openssl rand -base64 48` to generate

2. **Issuer**:
   - MUST match in both services
   - Development: `https://localhost:5002`
   - Production: `https://api.chargelk.com/um`

3. **Audience**:
   - MUST match in both services
   - Recommended: `chargelk-services`

### Database Configuration

```bash
# PostgreSQL Connection String Format
"Host=localhost;Database=<database_name>;Username=postgres;Password=<password>"

# Key Components:
# - Host: Database server address
# - Database: Database name (chargelk_um or chargelk_ocpp)
# - Username: Database user
# - Password: Database password
```

### MongoDB Configuration

```bash
# MongoDB Connection String
"mongodb://localhost:27017"

# With authentication:
"mongodb://username:password@localhost:27017"

# Database names:
# - chargelk_um (for UM service)
# - chargelk_ocpp (for OCPP service)
```

## Verification

### 1. Check Secrets Are Loaded

Add logging to verify secrets are being used:

```bash
cd ChargeLK.UM/ChargeLK.API
dotnet run

# Check console output for connection strings
# User Secrets values should be used instead of appsettings.json
```

### 2. Test Database Connection

```bash
# Start the service
dotnet run

# Check if it connects to your local database
# Look for successful database connection in logs
```

### 3. Verify JWT Configuration

```bash
# Login to UM service
curl -X POST http://localhost:5002/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "admin@chargelk.com", "password": "Admin@123"}'

# Use token to access OCPP service
# If JWT configuration matches, token should work
curl -X GET http://localhost:5001/api/chargepoints \
  -H "Authorization: Bearer <token>"
```

## Troubleshooting

### Problem: Secrets Not Being Used

**Symptoms**: Application uses appsettings.json values instead of user secrets

**Solutions**:
1. Verify you're in Development environment:
   ```bash
   echo $ASPNETCORE_ENVIRONMENT  # Should be "Development"
   ```

2. Check UserSecretsId in .csproj:
   ```bash
   grep UserSecretsId ChargeLK.API.csproj
   ```

3. Verify secrets exist:
   ```bash
   dotnet user-secrets list
   ```

### Problem: JWT Token Invalid Between Services

**Symptoms**: Token works in UM but fails in OCPP (401 Unauthorized)

**Solutions**:
1. Compare JWT secrets:
   ```bash
   cd ChargeLK.UM/ChargeLK.API && dotnet user-secrets list | grep "Jwt:"
   cd ../../ChargeLK.OCPP/ChargeLK.API && dotnet user-secrets list | grep "Jwt:"
   ```

2. Ensure all JWT values match:
   - SecretKey
   - Issuer
   - Audience

### Problem: Database Connection Fails

**Symptoms**: Application fails to start with database connection errors

**Solutions**:
1. Verify PostgreSQL is running:
   ```bash
   sudo systemctl status postgresql
   ```

2. Test connection manually:
   ```bash
   psql -h localhost -U postgres -d chargelk_um
   ```

3. Check connection string:
   ```bash
   dotnet user-secrets list | grep "DefaultConnection"
   ```

### Problem: Secrets File Not Found

**Symptoms**: "Could not find user secrets file"

**Solutions**:
1. Initialize user secrets:
   ```bash
   dotnet user-secrets init
   ```

2. Or verify .csproj has UserSecretsId:
   ```xml
   <UserSecretsId>your-guid-here</UserSecretsId>
   ```

## Best Practices

### Development
✅ **DO**:
- Use User Secrets for all sensitive data
- Generate strong, random JWT secrets
- Keep different secrets per developer
- Document required secrets in README
- Use the setup script for consistency

❌ **DON'T**:
- Commit secrets to source control
- Share secrets via email/chat
- Use production secrets in development
- Hardcode sensitive values in code

### Production
✅ **DO**:
- Use Azure Key Vault or similar
- Use environment variables
- Rotate secrets regularly
- Use strong, unique secrets per environment
- Monitor secret access

❌ **DON'T**:
- Use User Secrets in production
- Store secrets in appsettings.json
- Reuse development secrets
- Share secrets across environments

## Security Considerations

1. **Secret Strength**:
   - JWT secrets: 64+ characters
   - Database passwords: 16+ characters with mixed characters
   - Use password managers to generate

2. **Secret Rotation**:
   - Rotate JWT secrets periodically
   - Update database passwords regularly
   - Regenerate OAuth credentials if compromised

3. **Access Control**:
   - Secrets are user-specific on the machine
   - Other users can't access your secrets
   - Protect your user account

4. **Backup**:
   - Document how to regenerate secrets
   - Keep secure backup of production secrets
   - Use secrets management service

## Related Documentation

- [Development Guide](./README.md)
- [Authentication Testing Guide](../08-Security/AUTHENTICATION-TESTING-GUIDE.md)
- [Authentication Implementation](../08-Security/MICROSERVICES-AUTHENTICATION-IMPLEMENTATION.md)
- [Deployment Guide](../09-Operations/DEPLOYMENT-GUIDE.md)

## Additional Resources

- [Safe storage of app secrets in development](https://learn.microsoft.com/en-us/aspnet/core/security/app-secrets)
- [Configuration in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/configuration/)
- [Azure Key Vault Configuration Provider](https://learn.microsoft.com/en-us/aspnet/core/security/key-vault-configuration)

