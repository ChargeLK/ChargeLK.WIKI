# Development Guide

## 📚 Documentation Index

### Quick Start Guides

1. **[User Secrets Setup Guide](./USER-SECRETS-SETUP.md)**
   - Complete guide to .NET User Secrets
   - Automated setup script
   - Configuration management
   - Best practices and security

2. **[Admin Portal Quick Start](./ADMIN-PORTAL-QUICK-START.md)**
   - React Admin Portal setup
   - RTK Query configuration
   - Running the development server
   - API integration

3. **[RTK Implementation Plan](./RTK-IMPLEMENTATION-PLAN.md)**
   - Redux Toolkit Query setup
   - State management architecture
   - API slice configuration
   - Best practices

## Getting Started

This guide will help you set up your local development environment for ChargeLK.

## Prerequisites

### Required Software

| Tool | Version | Purpose |
|------|---------|---------|
| .NET SDK | 9.0+ | Backend development |
| Node.js | 20+ | Frontend development |
| Docker Desktop | Latest | Container orchestration |
| Git | Latest | Version control |
| PostgreSQL Client | 16+ | Database management (optional) |
| MongoDB Compass | Latest | MongoDB GUI (optional) |

### Recommended IDEs

- **Backend**: Visual Studio 2022, JetBrains Rider, or VS Code with C# extension
- **Frontend**: VS Code, WebStorm, or any modern IDE
- **Database**: DBeaver, pgAdmin, or MongoDB Compass

## Local Development Setup

### Option 1: Docker Compose (Recommended)

#### 1. Clone the Repository

```bash
git clone https://github.com/your-org/ChargeLK.git
cd ChargeLK
```

#### 2. Create Environment Files

Create `.env` file in the root directory:

```env
# PostgreSQL
POSTGRES_OCPP_PASSWORD=postgres_dev_password
POSTGRES_UM_PASSWORD=postgres_dev_password

# MongoDB
MONGO_ROOT_PASSWORD=mongo_dev_password

# JWT Configuration
JWT_SECRET_KEY=your-256-bit-secret-key-change-this-in-development

# Service URLs
OCPP_SERVICE_URL=http://localhost:5001
UM_SERVICE_URL=http://localhost:5002
```

#### 3. Start All Services

```bash
# Start infrastructure services (databases, storage)
docker-compose up -d postgres-ocpp postgres-um mongodb azurite redis

# Wait for databases to be ready (about 30 seconds)

# Start backend services
docker-compose up -d ocpp-service um-service

# Start frontend
docker-compose up -d admin-ui
```

#### 4. Verify Services

```bash
# Check running containers
docker-compose ps

# View logs
docker-compose logs -f ocpp-service
docker-compose logs -f um-service

# Check health endpoints
curl http://localhost:5001/health
curl http://localhost:5002/health
```

#### 5. Access Applications

- **Admin UI**: http://localhost:3000
- **OCPP Service API**: http://localhost:5001
- **OCPP Service Swagger**: http://localhost:5001/swagger
- **UM Service API**: http://localhost:5002
- **UM Service Swagger**: http://localhost:5002/swagger
- **OCPP WebSocket**: ws://localhost:5000/ocpp/{chargeBoxId}

---

### Option 2: Run Services Individually

#### Backend Services Setup

##### OCPP Service

```bash
cd ChargeLK.OCPP

# Restore dependencies
dotnet restore

# Update database
cd ChargeLK.API
dotnet ef database update --project ../ChargeLK.Infrastructure

# Run the service
dotnet run --project ChargeLK.API

# Service will start on http://localhost:5001
```

##### UM Service

```bash
cd ChargeLK.UM

# Restore dependencies
dotnet restore

# Update database
cd ChargeLK.API
dotnet ef database update --project ../ChargeLK.Infrastructure

# Run the service
dotnet run --project ChargeLK.API

# Service will start on http://localhost:5002
```

---

### Option 3: Using User Secrets (Recommended for Development)

.NET User Secrets provide a secure way to store sensitive configuration during development without committing them to source control.

#### Why Use User Secrets?

- ✅ Keep sensitive data out of source control
- ✅ Different configurations per developer
- ✅ Easy to manage and update
- ✅ Secure local development

#### Setting Up User Secrets

##### 1. Initialize User Secrets (Already configured in .csproj)

Both projects already have UserSecretsId configured:
- **UM Service**: `6923fadb-6223-4d79-bb1c-ffafd7c7c9ff`
- **OCPP Service**: `642bedcf-406c-4407-8b67-80696d2b549f`

##### 2. Configure UM Service Secrets

```bash
cd ChargeLK.UM/ChargeLK.API

# Set database connection
dotnet user-secrets set "ConnectionStrings:DefaultConnection" "Host=localhost;Database=chargelk_um;Username=postgres;Password=your_local_password"

# Set JWT secret (must be at least 32 characters)
dotnet user-secrets set "Jwt:SecretKey" "your-local-development-secret-key-min-32-chars-change-this"

# Set JWT configuration
dotnet user-secrets set "Jwt:Issuer" "https://localhost:5002"
dotnet user-secrets set "Jwt:Audience" "chargelk-services"

# Set MongoDB connection (optional if not using MongoDB)
dotnet user-secrets set "MongoDbSettings:ConnectionString" "mongodb://localhost:27017"
dotnet user-secrets set "MongoDbSettings:DatabaseName" "chargelk_um"

# Set Azure Blob Storage (optional)
dotnet user-secrets set "ConnectionStrings:AzureBlobStorage" "UseDevelopmentStorage=true"

# Set Google OAuth (optional)
dotnet user-secrets set "GoogleAuth:ClientId" "your-google-client-id"
dotnet user-secrets set "GoogleAuth:ClientSecret" "your-google-client-secret"

# View all secrets
dotnet user-secrets list
```

##### 3. Configure OCPP Service Secrets

```bash
cd ChargeLK.OCPP/ChargeLK.API

# Set database connection
dotnet user-secrets set "ConnectionStrings:DefaultConnection" "Host=localhost;Database=chargelk_ocpp;Username=postgres;Password=your_local_password"

# Set JWT secret (MUST match UM service)
dotnet user-secrets set "Jwt:SecretKey" "your-local-development-secret-key-min-32-chars-change-this"

# Set JWT configuration (MUST match UM service)
dotnet user-secrets set "Jwt:Issuer" "https://localhost:5002"
dotnet user-secrets set "Jwt:Audience" "chargelk-services"

# Set MongoDB connection (optional)
dotnet user-secrets set "MongoDbSettings:ConnectionString" "mongodb://localhost:27017"
dotnet user-secrets set "MongoDbSettings:DatabaseName" "chargelk_ocpp"

# Set Azure Blob Storage (optional)
dotnet user-secrets set "ConnectionStrings:AzureBlobStorage" "UseDevelopmentStorage=true"

# View all secrets
dotnet user-secrets list
```

##### 4. Quick Setup Script

Create a script `setup-secrets.sh` for quick configuration:

```bash
#!/bin/bash

# Colors for output
GREEN='\033[0;32m'
BLUE='\033[0;34m'
NC='\033[0m' # No Color

echo -e "${BLUE}ChargeLK User Secrets Setup${NC}"
echo ""

# Prompt for database password
read -sp "Enter PostgreSQL password for development: " DB_PASSWORD
echo ""

# Generate JWT secret key (64 characters)
JWT_SECRET=$(openssl rand -base64 48)

echo -e "${BLUE}Setting up UM Service secrets...${NC}"
cd ChargeLK.UM/ChargeLK.API
dotnet user-secrets set "ConnectionStrings:DefaultConnection" "Host=localhost;Database=chargelk_um;Username=postgres;Password=$DB_PASSWORD"
dotnet user-secrets set "Jwt:SecretKey" "$JWT_SECRET"
dotnet user-secrets set "Jwt:Issuer" "https://localhost:5002"
dotnet user-secrets set "Jwt:Audience" "chargelk-services"
dotnet user-secrets set "MongoDbSettings:ConnectionString" "mongodb://localhost:27017"
dotnet user-secrets set "MongoDbSettings:DatabaseName" "chargelk_um"

echo -e "${GREEN}✓ UM Service secrets configured${NC}"

echo -e "${BLUE}Setting up OCPP Service secrets...${NC}"
cd ../../ChargeLK.OCPP/ChargeLK.API
dotnet user-secrets set "ConnectionStrings:DefaultConnection" "Host=localhost;Database=chargelk_ocpp;Username=postgres;Password=$DB_PASSWORD"
dotnet user-secrets set "Jwt:SecretKey" "$JWT_SECRET"
dotnet user-secrets set "Jwt:Issuer" "https://localhost:5002"
dotnet user-secrets set "Jwt:Audience" "chargelk-services"
dotnet user-secrets set "MongoDbSettings:ConnectionString" "mongodb://localhost:27017"
dotnet user-secrets set "MongoDbSettings:DatabaseName" "chargelk_ocpp"

echo -e "${GREEN}✓ OCPP Service secrets configured${NC}"
echo ""
echo -e "${GREEN}User secrets setup complete!${NC}"
echo -e "JWT Secret Key: ${BLUE}${JWT_SECRET}${NC}"
echo ""
echo "To view secrets:"
echo "  cd ChargeLK.UM/ChargeLK.API && dotnet user-secrets list"
echo "  cd ChargeLK.OCPP/ChargeLK.API && dotnet user-secrets list"
```

Make it executable:
```bash
chmod +x setup-secrets.sh
./setup-secrets.sh
```

##### 5. Managing User Secrets

```bash
# List all secrets
dotnet user-secrets list

# Remove a specific secret
dotnet user-secrets remove "Jwt:SecretKey"

# Clear all secrets
dotnet user-secrets clear

# Get secret file location
dotnet user-secrets list --json
```

**Secret file locations:**
- **Linux/macOS**: `~/.microsoft/usersecrets/<user_secrets_id>/secrets.json`
- **Windows**: `%APPDATA%\Microsoft\UserSecrets\<user_secrets_id>\secrets.json`

##### 6. Important Notes

⚠️ **Critical Configuration Requirements:**

1. **JWT Secret Key Synchronization**: Both UM and OCPP services MUST use the exact same `Jwt:SecretKey`
2. **JWT Issuer/Audience**: Both services MUST have matching `Jwt:Issuer` and `Jwt:Audience` values
3. **Minimum Key Length**: JWT SecretKey must be at least 32 characters
4. **Database Passwords**: Use strong passwords even in development
5. **Never Commit**: User secrets are stored outside the project directory - never commit them

##### 7. Verify Configuration

After setting up secrets, verify they override appsettings.json:

```bash
cd ChargeLK.UM/ChargeLK.API
dotnet run

# Check logs - should show connection to your specified database
# Connection string from user secrets should be used, not appsettings.json
```

---

#### Frontend Setup

##### Admin UI

```bash
cd ChargeLK.AdminUI

# Install dependencies
npm install

# Start development server
npm run dev

# Application will start on http://localhost:3000
```

---

## Database Setup

### PostgreSQL Setup

#### Using Docker (Recommended)

```bash
# Start PostgreSQL containers
docker-compose up -d postgres-ocpp postgres-um
```

#### Manual Installation

```bash
# Install PostgreSQL 16
sudo apt install postgresql-16  # Ubuntu/Debian
brew install postgresql@16      # macOS

# Create databases
psql -U postgres
CREATE DATABASE chargelk_ocpp;
CREATE DATABASE chargelk_um;
\q
```

### MongoDB Setup

#### Using Docker (Recommended)

```bash
docker-compose up -d mongodb
```

#### Manual Installation

```bash
# Install MongoDB
sudo apt install mongodb-org  # Ubuntu/Debian
brew install mongodb-community # macOS

# Start MongoDB
sudo systemctl start mongod    # Linux
brew services start mongodb    # macOS
```

### Running Migrations

#### Entity Framework Migrations (PostgreSQL)

```bash
# OCPP Service migrations
cd ChargeLK.OCPP/ChargeLK.API
dotnet ef database update --project ../ChargeLK.Infrastructure

# UM Service migrations
cd ChargeLK.UM/ChargeLK.API
dotnet ef database update --project ../ChargeLK.Infrastructure
```

#### Creating New Migrations

```bash
# OCPP Service
cd ChargeLK.OCPP/ChargeLK.API
dotnet ef migrations add MigrationName --project ../ChargeLK.Infrastructure

# UM Service
cd ChargeLK.UM/ChargeLK.API
dotnet ef migrations add MigrationName --project ../ChargeLK.Infrastructure
```

---

## Configuration

### appsettings.Development.json Examples

#### ChargeLK.OCPP/ChargeLK.API/appsettings.Development.json

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "ConnectionStrings": {
    "DefaultConnection": "Host=localhost;Database=chargelk_ocpp;Username=postgres;Password=postgres_dev_password"
  },
  "MongoDbSettings": {
    "ConnectionString": "mongodb://admin:mongo_dev_password@localhost:27017",
    "DatabaseName": "chargelk_ocpp"
  },
  "AzureBlobStorage": {
    "ConnectionString": "UseDevelopmentStorage=true"
  },
  "AzureServiceBus": {
    "ConnectionString": "",
    "Enabled": false
  },
  "UserServiceBaseUrl": "http://localhost:5002",
  "AllowedOrigins": [
    "http://localhost:3000"
  ],
  "Jwt": {
    "SecretKey": "your-development-secret-key-at-least-32-characters-long",
    "Issuer": "https://localhost:5001",
    "Audience": "chargelk-services",
    "ExpirationInMinutes": 60
  }
}
```

#### ChargeLK.UM/ChargeLK.API/appsettings.Development.json

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Microsoft.EntityFrameworkCore": "Information"
    }
  },
  "ConnectionStrings": {
    "DefaultConnection": "Host=localhost;Port=5433;Database=chargelk_um;Username=postgres;Password=postgres_dev_password"
  },
  "MongoDbSettings": {
    "ConnectionString": "mongodb://admin:mongo_dev_password@localhost:27017",
    "DatabaseName": "chargelk_um"
  },
  "AzureBlobStorage": {
    "ConnectionString": "UseDevelopmentStorage=true"
  },
  "AzureServiceBus": {
    "ConnectionString": "",
    "Enabled": false
  },
  "OcppServiceBaseUrl": "http://localhost:5001",
  "AllowedOrigins": [
    "http://localhost:3000"
  ],
  "Jwt": {
    "SecretKey": "your-development-secret-key-at-least-32-characters-long",
    "Issuer": "https://localhost:5002",
    "Audience": "chargelk-services",
    "ExpirationInMinutes": 60,
    "RefreshTokenExpirationInDays": 7
  }
}
```

---

## Testing

### Running Tests

#### Backend Tests

```bash
# Run all tests
cd ChargeLK.OCPP
dotnet test

cd ChargeLK.UM
dotnet test

# Run with coverage
dotnet test /p:CollectCoverage=true /p:CoverletOutputFormat=opencover
```

#### Frontend Tests

```bash
cd ChargeLK.AdminUI

# Run tests
npm test

# Run with coverage
npm run test:coverage
```

### Test Structure

```
ChargeLK.OCPP/
├── ChargeLK.UnitTests/
│   ├── Application/
│   │   ├── Handlers/
│   │   └── Services/
│   └── Domain/
│       └── Entities/
├── ChargeLK.IntegrationTests/
│   ├── Api/
│   │   └── Controllers/
│   └── Infrastructure/
│       └── Repositories/
└── ChargeLK.FunctionalTests/
    └── Scenarios/
```

### Writing Tests

#### Unit Test Example

```csharp
using Xunit;
using Moq;
using ChargeLK.Application.Handlers.ChargePoint;

public class GetChargePointHandlerTests
{
    [Fact]
    public async Task Handle_ValidId_ReturnsChargePoint()
    {
        // Arrange
        var mockRepository = new Mock<IChargePointRepository>();
        var expectedChargePoint = new ChargePoint { Id = Guid.NewGuid() };
        mockRepository.Setup(r => r.GetByIdAsync(It.IsAny<Guid>()))
            .ReturnsAsync(expectedChargePoint);
        
        var handler = new GetChargePointHandler(mockRepository.Object);
        var query = new GetChargePointQuery { Id = expectedChargePoint.Id };
        
        // Act
        var result = await handler.Handle(query, CancellationToken.None);
        
        // Assert
        Assert.NotNull(result);
        Assert.Equal(expectedChargePoint.Id, result.Id);
    }
}
```

---

## Debugging

### Debugging Backend Services

#### Visual Studio / Rider

1. Open solution file (.sln)
2. Set startup project (ChargeLK.API)
3. Press F5 or click Debug

#### VS Code

Create `.vscode/launch.json`:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "OCPP Service",
      "type": "coreclr",
      "request": "launch",
      "preLaunchTask": "build",
      "program": "${workspaceFolder}/ChargeLK.OCPP/ChargeLK.API/bin/Debug/net8.0/ChargeLK.API.dll",
      "args": [],
      "cwd": "${workspaceFolder}/ChargeLK.OCPP/ChargeLK.API",
      "stopAtEntry": false,
      "serverReadyAction": {
        "action": "openExternally",
        "pattern": "\\bNow listening on:\\s+(https?://\\S+)"
      },
      "env": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    },
    {
      "name": "UM Service",
      "type": "coreclr",
      "request": "launch",
      "preLaunchTask": "build",
      "program": "${workspaceFolder}/ChargeLK.UM/ChargeLK.API/bin/Debug/net8.0/ChargeLK.API.dll",
      "args": [],
      "cwd": "${workspaceFolder}/ChargeLK.UM/ChargeLK.API",
      "stopAtEntry": false,
      "env": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    }
  ]
}
```

### Debugging Frontend

1. Start dev server: `npm run dev`
2. Use browser DevTools (F12)
3. Use React DevTools extension
4. VS Code: Install "Debugger for Chrome" extension

---

## OCPP Testing

### Using OCPP Simulator

#### Option 1: SteVe (Free, Open Source)

```bash
# Clone SteVe
git clone https://github.com/steve-community/steve.git
cd steve

# Run with Docker
docker run -p 8080:8080 steve/steve:latest

# Access: http://localhost:8080
# Connect to: ws://localhost:5000/ocpp/TEST_CP_001
```

#### Option 2: Custom Test Client

Create `ocpp-test-client.js`:

```javascript
const WebSocket = require('ws');

const chargeBoxId = 'TEST_CP_001';
const ws = new WebSocket(`ws://localhost:5000/ocpp/${chargeBoxId}`);

ws.on('open', () => {
  console.log('Connected to OCPP server');
  
  // Send BootNotification
  const bootNotification = [
    2,  // Call
    "1",  // MessageId
    "BootNotification",
    {
      "chargePointVendor": "Test Vendor",
      "chargePointModel": "Test Model",
      "chargePointSerialNumber": "TEST001",
      "firmwareVersion": "1.0.0"
    }
  ];
  
  ws.send(JSON.stringify(bootNotification));
});

ws.on('message', (data) => {
  console.log('Received:', data.toString());
});

ws.on('error', (error) => {
  console.error('Error:', error);
});
```

Run the test:

```bash
npm install ws
node ocpp-test-client.js
```

---

## Common Development Tasks

### Adding a New Entity

1. **Create Entity in Domain Layer**

```csharp
// ChargeLK.Domain/Entities/YourEntity/YourEntity.cs
public class YourEntity : BaseEntity
{
    public string Name { get; set; }
    // ... other properties
}
```

2. **Create Repository Interface**

```csharp
// ChargeLK.Application/Interfaces/IYourEntityRepository.cs
public interface IYourEntityRepository
{
    Task<YourEntity> GetByIdAsync(Guid id);
    Task<IEnumerable<YourEntity>> GetAllAsync();
    // ... other methods
}
```

3. **Implement Repository**

```csharp
// ChargeLK.Infrastructure/Repositories/YourEntityRepository.cs
public class YourEntityRepository : IYourEntityRepository
{
    private readonly ApplicationDbContext _context;
    
    public YourEntityRepository(ApplicationDbContext context)
    {
        _context = context;
    }
    
    // Implement methods...
}
```

4. **Add Entity Configuration**

```csharp
// ChargeLK.Infrastructure/EntityConfigurations/YourEntityConfiguration.cs
public class YourEntityConfiguration : IEntityTypeConfiguration<YourEntity>
{
    public void Configure(EntityTypeBuilder<YourEntity> builder)
    {
        builder.ToTable("your_entities");
        builder.HasKey(e => e.Id);
        // ... other configurations
    }
}
```

5. **Create Migration**

```bash
dotnet ef migrations add AddYourEntity --project ChargeLK.Infrastructure
dotnet ef database update --project ChargeLK.Infrastructure
```

### Adding a New API Endpoint

1. **Create Handler**

```csharp
// ChargeLK.Application/Handlers/YourEntity/GetYourEntityHandler.cs
public class GetYourEntityHandler : IRequestHandler<GetYourEntityQuery, YourEntityDto>
{
    // Implementation...
}
```

2. **Create Controller**

```csharp
// ChargeLK.API/Controllers/RestApi/YourEntityController.cs
[ApiController]
[Route("api/[controller]")]
public class YourEntityController : BaseApiController
{
    [HttpGet("{id}")]
    public async Task<ActionResult<YourEntityDto>> Get(Guid id)
    {
        var result = await Mediator.Send(new GetYourEntityQuery { Id = id });
        return Ok(result);
    }
}
```

---

## Coding Standards

### C# Coding Standards

- Follow Microsoft C# Coding Conventions
- Use PascalCase for public members
- Use camelCase for private fields (with _ prefix)
- Use meaningful names
- Keep methods small and focused
- Add XML documentation for public APIs

```csharp
/// <summary>
/// Gets a charge point by its unique identifier.
/// </summary>
/// <param name="id">The charge point identifier.</param>
/// <returns>The charge point details.</returns>
public async Task<ChargePoint> GetChargePointAsync(Guid id)
{
    // Implementation
}
```

### TypeScript Coding Standards

- Use ESLint configuration
- Use PascalCase for components and types
- Use camelCase for variables and functions
- Use meaningful names
- Prefer const over let
- Use async/await over promises

```typescript
// Good
const fetchChargePoints = async (): Promise<ChargePoint[]> => {
  const response = await api.get('/chargepoints');
  return response.data;
};

// Avoid
function getCP() {
  return api.get('/chargepoints').then(r => r.data);
}
```

---

## Troubleshooting

### Common Issues

#### Database Connection Failed

```bash
# Check PostgreSQL is running
docker-compose ps postgres-ocpp
docker-compose ps postgres-um

# Check connection string in appsettings.Development.json
# Verify password matches .env file
```

#### MongoDB Connection Failed

```bash
# Check MongoDB is running
docker-compose ps mongodb

# Test connection
mongosh "mongodb://admin:mongo_dev_password@localhost:27017"
```

#### Port Already in Use

```bash
# Find process using port 5001
lsof -i :5001  # macOS/Linux
netstat -ano | findstr :5001  # Windows

# Kill process or change port in launchSettings.json
```

#### WebSocket Connection Failed

```bash
# Check OCPP service is running
curl http://localhost:5001/health

# Check WebSocket port is exposed
docker-compose ps ocpp-service

# Test WebSocket connection
wscat -c ws://localhost:5000/ocpp/TEST
```

---

## Useful Commands

### Docker

```bash
# View logs
docker-compose logs -f [service-name]

# Restart service
docker-compose restart [service-name]

# Rebuild service
docker-compose up -d --build [service-name]

# Stop all services
docker-compose down

# Remove volumes (⚠️ deletes data)
docker-compose down -v
```

### Database

```bash
# PostgreSQL
docker exec -it chargelk-postgres-ocpp psql -U postgres -d chargelk_ocpp

# MongoDB
docker exec -it chargelk-mongodb mongosh -u admin -p mongo_dev_password

# Backup database
docker exec chargelk-postgres-ocpp pg_dump -U postgres chargelk_ocpp > backup.sql

# Restore database
docker exec -i chargelk-postgres-ocpp psql -U postgres chargelk_ocpp < backup.sql
```

---

## Related Documentation

- [Architecture Overview](../02-Architecture/README.md)
- [API Reference](../04-API-Reference/README.md)
- [Infrastructure](../06-Infrastructure/README.md)
- [Contributing Guidelines](./contributing.md)

