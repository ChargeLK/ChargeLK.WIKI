# Infrastructure & Deployment

## Overview

ChargeLK infrastructure is designed to run both locally for development and in production on Azure cloud platform.

## Deployment Architectures

### Local Development Architecture

```mermaid
graph TB
    subgraph "Developer Machine"
        subgraph "Docker Compose Services"
            OCPP[OCPP Service<br/>:5001]
            UM[UM Service<br/>:5002]
            AdminUI[Admin UI<br/>:3000]
            
            PG_OCPP[(PostgreSQL OCPP<br/>:5432)]
            PG_UM[(PostgreSQL UM<br/>:5433)]
            MONGO[(MongoDB<br/>:27017)]
            AZURITE[Azurite<br/>Blob Storage Emulator]
            
            subgraph "Service Bus Emulator (Optional)"
                SB_EMU[Service Bus Emulator]
            end
        end
    end
    
    CP[OCPP Simulator<br/>External] -->|WebSocket| OCPP
    Browser[Web Browser] -->|HTTP| AdminUI
    
    AdminUI -->|REST API| OCPP
    AdminUI -->|REST API| UM
    
    OCPP --> PG_OCPP
    OCPP --> MONGO
    OCPP --> AZURITE
    OCPP -.->|Events| SB_EMU
    
    UM --> PG_UM
    UM --> MONGO
    UM --> AZURITE
    UM -.->|Events| SB_EMU
```

### Production Azure Architecture

```mermaid
graph TB
    subgraph "External"
        CP[Charge Points<br/>OCPP 1.6]
        Mobile[Mobile Apps]
        Browser[Web Browsers]
    end
    
    subgraph "Azure Cloud"
        subgraph "Front Door / Application Gateway"
            AFD[Azure Front Door]
            WAF[Web Application Firewall]
        end
        
        subgraph "Azure Kubernetes Service (AKS) / App Services"
            LB[Load Balancer]
            
            subgraph "OCPP Service Pods"
                OCPP1[OCPP Service<br/>Instance 1]
                OCPP2[OCPP Service<br/>Instance 2]
                OCPP3[OCPP Service<br/>Instance N]
            end
            
            subgraph "UM Service Pods"
                UM1[UM Service<br/>Instance 1]
                UM2[UM Service<br/>Instance 2]
                UM3[UM Service<br/>Instance N]
            end
        end
        
        subgraph "Data Layer"
            PG_OCPP[(Azure Database<br/>PostgreSQL<br/>OCPP)]
            PG_UM[(Azure Database<br/>PostgreSQL<br/>UM)]
            COSMOS[(Azure Cosmos DB<br/>MongoDB API)]
        end
        
        subgraph "Integration Layer"
            SB[Azure Service Bus<br/>Premium Tier]
            BLOB[Azure Blob Storage]
            REDIS[Azure Cache<br/>for Redis]
        end
        
        subgraph "Monitoring & Logging"
            MONITOR[Azure Monitor]
            APPINS[Application Insights]
            LOGS[Log Analytics]
        end
        
        subgraph "Static Content"
            CDN[Azure CDN]
            STORAGE[Storage Account<br/>Static Website]
        end
    end
    
    CP -->|WSS| AFD
    Mobile -->|HTTPS| AFD
    Browser -->|HTTPS| CDN
    
    AFD --> WAF
    WAF --> LB
    
    LB --> OCPP1
    LB --> OCPP2
    LB --> OCPP3
    LB --> UM1
    LB --> UM2
    LB --> UM3
    
    OCPP1 --> PG_OCPP
    OCPP2 --> PG_OCPP
    OCPP3 --> PG_OCPP
    
    UM1 --> PG_UM
    UM2 --> PG_UM
    UM3 --> PG_UM
    
    OCPP1 --> COSMOS
    OCPP2 --> COSMOS
    UM1 --> COSMOS
    UM2 --> COSMOS
    
    OCPP1 --> SB
    OCPP2 --> SB
    UM1 --> SB
    UM2 --> SB
    
    OCPP1 --> BLOB
    UM1 --> BLOB
    
    OCPP1 --> REDIS
    OCPP2 --> REDIS
    UM1 --> REDIS
    UM2 --> REDIS
    
    STORAGE --> CDN
    
    OCPP1 --> APPINS
    UM1 --> APPINS
    APPINS --> MONITOR
    APPINS --> LOGS
```

## Docker Configuration

### Development Docker Compose

Create `docker-compose.yml` in the root directory:

```yaml
version: '3.8'

services:
  # PostgreSQL for OCPP Service
  postgres-ocpp:
    image: postgres:16-alpine
    container_name: chargelk-postgres-ocpp
    environment:
      POSTGRES_DB: chargelk_ocpp
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres_dev_password
    ports:
      - "5432:5432"
    volumes:
      - postgres-ocpp-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  # PostgreSQL for UM Service
  postgres-um:
    image: postgres:16-alpine
    container_name: chargelk-postgres-um
    environment:
      POSTGRES_DB: chargelk_um
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres_dev_password
    ports:
      - "5433:5432"
    volumes:
      - postgres-um-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  # MongoDB
  mongodb:
    image: mongo:7
    container_name: chargelk-mongodb
    ports:
      - "27017:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: mongo_dev_password
    volumes:
      - mongodb-data:/data/db
    healthcheck:
      test: echo 'db.runCommand("ping").ok' | mongosh localhost:27017/test --quiet
      interval: 10s
      timeout: 5s
      retries: 5

  # Azurite - Azure Storage Emulator
  azurite:
    image: mcr.microsoft.com/azure-storage/azurite
    container_name: chargelk-azurite
    ports:
      - "10000:10000"  # Blob service
      - "10001:10001"  # Queue service
      - "10002:10002"  # Table service
    volumes:
      - azurite-data:/data
    command: "azurite --blobHost 0.0.0.0 --queueHost 0.0.0.0 --tableHost 0.0.0.0 --loose"

  # Redis (for caching - future enhancement)
  redis:
    image: redis:7-alpine
    container_name: chargelk-redis
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    command: redis-server --appendonly yes

  # OCPP Service
  ocpp-service:
    build:
      context: ./ChargeLK.OCPP
      dockerfile: Dockerfile
    container_name: chargelk-ocpp-service
    ports:
      - "5001:8080"
      - "5000:5000"  # WebSocket
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__DefaultConnection=Host=postgres-ocpp;Database=chargelk_ocpp;Username=postgres;Password=postgres_dev_password
      - MongoDbSettings__ConnectionString=mongodb://admin:mongo_dev_password@mongodb:27017
      - MongoDbSettings__DatabaseName=chargelk_ocpp
      - AzureBlobStorage__ConnectionString=DefaultEndpointsProtocol=http;AccountName=devstoreaccount1;AccountKey=Eby8vdM02xNOcqFlqUwJPLlmEtlCDXJ1OUzFT50uSRZ6IFsuFq2UVErCz4I6tq/K1SZFPTOtr/KBHBeksoGMGw==;BlobEndpoint=http://azurite:10000/devstoreaccount1
      - UserServiceBaseUrl=http://um-service:8080
    depends_on:
      postgres-ocpp:
        condition: service_healthy
      mongodb:
        condition: service_healthy
      azurite:
        condition: service_started
    networks:
      - chargelk-network

  # UM Service
  um-service:
    build:
      context: ./ChargeLK.UM
      dockerfile: Dockerfile
    container_name: chargelk-um-service
    ports:
      - "5002:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__DefaultConnection=Host=postgres-um;Database=chargelk_um;Username=postgres;Password=postgres_dev_password
      - MongoDbSettings__ConnectionString=mongodb://admin:mongo_dev_password@mongodb:27017
      - MongoDbSettings__DatabaseName=chargelk_um
      - AzureBlobStorage__ConnectionString=DefaultEndpointsProtocol=http;AccountName=devstoreaccount1;AccountKey=Eby8vdM02xNOcqFlqUwJPLlmEtlCDXJ1OUzFT50uSRZ6IFsuFq2UVErCz4I6tq/K1SZFPTOtr/KBHBeksoGMGw==;BlobEndpoint=http://azurite:10000/devstoreaccount1
      - OcppServiceBaseUrl=http://ocpp-service:8080
    depends_on:
      postgres-um:
        condition: service_healthy
      mongodb:
        condition: service_healthy
      azurite:
        condition: service_started
    networks:
      - chargelk-network

  # Admin UI
  admin-ui:
    build:
      context: ./ChargeLK.AdminUI
      dockerfile: Dockerfile
    container_name: chargelk-admin-ui
    ports:
      - "3000:80"
    environment:
      - VITE_OCPP_API_URL=http://localhost:5001
      - VITE_UM_API_URL=http://localhost:5002
    depends_on:
      - ocpp-service
      - um-service
    networks:
      - chargelk-network

volumes:
  postgres-ocpp-data:
  postgres-um-data:
  mongodb-data:
  azurite-data:
  redis-data:

networks:
  chargelk-network:
    driver: bridge
```

### Service Dockerfiles

#### ChargeLK.OCPP/Dockerfile

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 8080
EXPOSE 5000

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["ChargeLK.API/ChargeLK.API.csproj", "ChargeLK.API/"]
COPY ["ChargeLK.Application/ChargeLK.Application.csproj", "ChargeLK.Application/"]
COPY ["ChargeLK.Domain/ChargeLK.Domain.csproj", "ChargeLK.Domain/"]
COPY ["ChargeLK.Infrastructure/ChargeLK.Infrastructure.csproj", "ChargeLK.Infrastructure/"]
RUN dotnet restore "ChargeLK.API/ChargeLK.API.csproj"
COPY . .
WORKDIR "/src/ChargeLK.API"
RUN dotnet build "ChargeLK.API.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "ChargeLK.API.csproj" -c Release -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "ChargeLK.API.dll"]
```

#### ChargeLK.AdminUI/Dockerfile

```dockerfile
# Build stage
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Production stage
FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

#### ChargeLK.AdminUI/nginx.conf

```nginx
server {
    listen 80;
    server_name localhost;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    # API proxy for development
    location /api/ocpp {
        proxy_pass http://ocpp-service:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    location /api/um {
        proxy_pass http://um-service:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    # WebSocket proxy for OCPP
    location /ocpp {
        proxy_pass http://ocpp-service:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_set_header Host $host;
    }
}
```

## Azure Infrastructure as Code

### Terraform Configuration

Create `infrastructure/terraform/main.tf`:

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
  backend "azurerm" {
    resource_group_name  = "chargelk-tfstate-rg"
    storage_account_name = "chargelktfstate"
    container_name       = "tfstate"
    key                  = "prod.terraform.tfstate"
  }
}

provider "azurerm" {
  features {}
}

# Resource Group
resource "azurerm_resource_group" "main" {
  name     = "chargelk-${var.environment}-rg"
  location = var.location
  tags     = var.tags
}

# Azure Container Registry
resource "azurerm_container_registry" "acr" {
  name                = "chargelk${var.environment}acr"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  sku                 = "Premium"
  admin_enabled       = true
  tags                = var.tags
}

# Azure Kubernetes Service
resource "azurerm_kubernetes_cluster" "aks" {
  name                = "chargelk-${var.environment}-aks"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  dns_prefix          = "chargelk-${var.environment}"

  default_node_pool {
    name       = "default"
    node_count = var.aks_node_count
    vm_size    = "Standard_D4s_v3"
  }

  identity {
    type = "SystemAssigned"
  }

  tags = var.tags
}

# PostgreSQL Servers
resource "azurerm_postgresql_flexible_server" "ocpp" {
  name                   = "chargelk-${var.environment}-ocpp-psql"
  resource_group_name    = azurerm_resource_group.main.name
  location               = azurerm_resource_group.main.location
  version                = "16"
  administrator_login    = var.postgres_admin_username
  administrator_password = var.postgres_admin_password
  storage_mb             = 32768
  sku_name               = "GP_Standard_D4s_v3"
  
  tags = var.tags
}

resource "azurerm_postgresql_flexible_server" "um" {
  name                   = "chargelk-${var.environment}-um-psql"
  resource_group_name    = azurerm_resource_group.main.name
  location               = azurerm_resource_group.main.location
  version                = "16"
  administrator_login    = var.postgres_admin_username
  administrator_password = var.postgres_admin_password
  storage_mb             = 32768
  sku_name               = "GP_Standard_D2s_v3"
  
  tags = var.tags
}

# Azure Cosmos DB (MongoDB API)
resource "azurerm_cosmosdb_account" "mongodb" {
  name                = "chargelk-${var.environment}-cosmos"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  offer_type          = "Standard"
  kind                = "MongoDB"

  consistency_policy {
    consistency_level = "Session"
  }

  geo_location {
    location          = azurerm_resource_group.main.location
    failover_priority = 0
  }

  capabilities {
    name = "EnableMongo"
  }

  tags = var.tags
}

# Azure Service Bus
resource "azurerm_servicebus_namespace" "main" {
  name                = "chargelk-${var.environment}-sb"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  sku                 = "Premium"
  capacity            = 1

  tags = var.tags
}

# Service Bus Topics
resource "azurerm_servicebus_topic" "transaction_events" {
  name         = "transaction-events"
  namespace_id = azurerm_servicebus_namespace.main.id
  
  enable_partitioning = true
}

resource "azurerm_servicebus_topic" "charging_events" {
  name         = "charging-events"
  namespace_id = azurerm_servicebus_namespace.main.id
  
  enable_partitioning = true
}

# Blob Storage
resource "azurerm_storage_account" "main" {
  name                     = "chargelk${var.environment}storage"
  resource_group_name      = azurerm_resource_group.main.name
  location                 = azurerm_resource_group.main.location
  account_tier             = "Standard"
  account_replication_type = "GRS"
  
  tags = var.tags
}

# Azure Cache for Redis
resource "azurerm_redis_cache" "main" {
  name                = "chargelk-${var.environment}-redis"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  capacity            = 1
  family              = "P"
  sku_name            = "Premium"
  
  tags = var.tags
}

# Application Insights
resource "azurerm_application_insights" "main" {
  name                = "chargelk-${var.environment}-appinsights"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  application_type    = "web"
  
  tags = var.tags
}
```

## Kubernetes Deployment

### OCPP Service Deployment

Create `k8s/ocpp-service-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ocpp-service
  namespace: chargelk
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ocpp-service
  template:
    metadata:
      labels:
        app: ocpp-service
    spec:
      containers:
      - name: ocpp-service
        image: chargelkprodacr.azurecr.io/ocpp-service:latest
        ports:
        - containerPort: 8080
          name: http
        - containerPort: 5000
          name: websocket
        env:
        - name: ASPNETCORE_ENVIRONMENT
          value: "Production"
        - name: ConnectionStrings__DefaultConnection
          valueFrom:
            secretKeyRef:
              name: ocpp-secrets
              key: postgres-connection
        - name: MongoDbSettings__ConnectionString
          valueFrom:
            secretKeyRef:
              name: ocpp-secrets
              key: mongodb-connection
        resources:
          requests:
            memory: "2Gi"
            cpu: "1000m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: ocpp-service
  namespace: chargelk
spec:
  selector:
    app: ocpp-service
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: websocket
    port: 5000
    targetPort: 5000
  type: LoadBalancer
```

## CI/CD Pipeline

### GitHub Actions Workflow

Create `.github/workflows/deploy.yml`:

```yaml
name: Build and Deploy

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  AZURE_CONTAINER_REGISTRY: chargelkprodacr.azurecr.io
  AKS_CLUSTER_NAME: chargelk-prod-aks
  AKS_RESOURCE_GROUP: chargelk-prod-rg

jobs:
  build-ocpp-service:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Login to Azure Container Registry
      uses: docker/login-action@v2
      with:
        registry: ${{ env.AZURE_CONTAINER_REGISTRY }}
        username: ${{ secrets.ACR_USERNAME }}
        password: ${{ secrets.ACR_PASSWORD }}
    
    - name: Build and push OCPP Service
      uses: docker/build-push-action@v4
      with:
        context: ./ChargeLK.OCPP
        push: true
        tags: |
          ${{ env.AZURE_CONTAINER_REGISTRY }}/ocpp-service:${{ github.sha }}
          ${{ env.AZURE_CONTAINER_REGISTRY }}/ocpp-service:latest

  build-um-service:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Login to Azure Container Registry
      uses: docker/login-action@v2
      with:
        registry: ${{ env.AZURE_CONTAINER_REGISTRY }}
        username: ${{ secrets.ACR_USERNAME }}
        password: ${{ secrets.ACR_PASSWORD }}
    
    - name: Build and push UM Service
      uses: docker/build-push-action@v4
      with:
        context: ./ChargeLK.UM
        push: true
        tags: |
          ${{ env.AZURE_CONTAINER_REGISTRY }}/um-service:${{ github.sha }}
          ${{ env.AZURE_CONTAINER_REGISTRY }}/um-service:latest

  deploy-to-aks:
    needs: [build-ocpp-service, build-um-service]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
    - uses: actions/checkout@v3
    
    - name: Azure Login
      uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}
    
    - name: Set AKS context
      uses: azure/aks-set-context@v3
      with:
        cluster-name: ${{ env.AKS_CLUSTER_NAME }}
        resource-group: ${{ env.AKS_RESOURCE_GROUP }}
    
    - name: Deploy to AKS
      run: |
        kubectl apply -f k8s/
        kubectl rollout restart deployment/ocpp-service -n chargelk
        kubectl rollout restart deployment/um-service -n chargelk
```

## Monitoring & Logging

### Application Insights Configuration

Both services send telemetry to Application Insights:

```json
{
  "ApplicationInsights": {
    "InstrumentationKey": "your-instrumentation-key",
    "EnableAdaptiveSampling": true,
    "EnablePerformanceCounterCollectionModule": true
  }
}
```

### Logging Configuration

```json
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "System": "Warning"
      }
    },
    "WriteTo": [
      {
        "Name": "Console"
      },
      {
        "Name": "ApplicationInsights",
        "Args": {
          "telemetryConverter": "Serilog.Sinks.ApplicationInsights.TelemetryConverters.TraceTelemetryConverter, Serilog.Sinks.ApplicationInsights"
        }
      }
    ]
  }
}
```

## Environment Variables Reference

### OCPP Service

| Variable | Description | Example |
|----------|-------------|---------|
| `ASPNETCORE_ENVIRONMENT` | Environment name | `Development`, `Production` |
| `ConnectionStrings__DefaultConnection` | PostgreSQL connection | `Host=...;Database=...` |
| `MongoDbSettings__ConnectionString` | MongoDB connection | `mongodb://...` |
| `AzureServiceBus__ConnectionString` | Service Bus connection | `Endpoint=sb://...` |
| `AzureBlobStorage__ConnectionString` | Blob Storage connection | `DefaultEndpointsProtocol=https;...` |
| `UserServiceBaseUrl` | UM Service URL | `http://um-service:8080` |

### UM Service

| Variable | Description | Example |
|----------|-------------|---------|
| `ASPNETCORE_ENVIRONMENT` | Environment name | `Development`, `Production` |
| `ConnectionStrings__DefaultConnection` | PostgreSQL connection | `Host=...;Database=...` |
| `MongoDbSettings__ConnectionString` | MongoDB connection | `mongodb://...` |
| `AzureServiceBus__ConnectionString` | Service Bus connection | `Endpoint=sb://...` |
| `AzureBlobStorage__ConnectionString` | Blob Storage connection | `DefaultEndpointsProtocol=https;...` |
| `OcppServiceBaseUrl` | OCPP Service URL | `http://ocpp-service:8080` |
| `Jwt__SecretKey` | JWT signing key | `your-secret-key` |

## Related Documentation

- [Development Guide](../07-Development/README.md)
- [Architecture Overview](../02-Architecture/README.md)
- [Operations & Monitoring](../09-Operations/README.md)

