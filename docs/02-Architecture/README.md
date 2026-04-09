# System Architecture

## Architecture Overview

ChargeLK follows a microservices architecture pattern with clear separation of concerns, enabling independent scaling, deployment, and development of different system components.

## Architecture Principles

### 1. **Microservices Architecture**
Each service is independently deployable, scalable, and maintainable:
- **ChargeLK.OCPP**: Handles OCPP protocol and charge point management
- **ChargeLK.UM**: Handles user management and business transactions

### 2. **Clean Architecture**
Both backend services follow Clean Architecture principles:
- **Domain Layer**: Business entities and rules
- **Application Layer**: Use cases and business logic
- **Infrastructure Layer**: External concerns (database, messaging, APIs)
- **Presentation Layer**: API controllers and endpoints

### 3. **Event-Driven Communication**
Services communicate asynchronously using Azure Service Bus:
- Loose coupling between services
- Eventual consistency
- Scalable message processing

### 4. **API Gateway Pattern** (Ready for Implementation)
Single entry point for clients:
- Request routing
- Authentication/Authorization
- Rate limiting
- API composition

### 5. **Database per Service**
Each microservice has its own database:
- Data isolation and autonomy
- Independent scaling
- Technology diversity (PostgreSQL + MongoDB)

## C4 Model Architecture Diagrams

### Level 1: System Context Diagram

```mermaid
C4Context
    title System Context Diagram - ChargeLK EV Charging Solution

    Person(evDriver, "EV Driver", "End user who charges their vehicle")
    Person(csoAdmin, "CSO Administrator", "Manages charging infrastructure")
    Person(sysAdmin, "System Administrator", "Manages entire platform")
    
    System(chargeLK, "ChargeLK Platform", "EV Charging Management System")
    
    System_Ext(chargePoint, "Charge Point", "Physical EV charging station with OCPP 1.6")
    System_Ext(paymentGateway, "Payment Gateway", "Processes payments")
    System_Ext(notificationService, "Notification Service", "Sends emails/SMS")
    
    Rel(evDriver, chargeLK, "Uses", "HTTPS/Mobile App")
    Rel(csoAdmin, chargeLK, "Manages", "HTTPS/Web Portal")
    Rel(sysAdmin, chargeLK, "Administers", "HTTPS/Web Portal")
    
    Rel(chargePoint, chargeLK, "Communicates", "WebSocket/OCPP 1.6")
    Rel(chargeLK, paymentGateway, "Processes payments", "HTTPS")
    Rel(chargeLK, notificationService, "Sends notifications", "HTTPS")
```

### Level 2: Container Diagram

```mermaid
C4Container
    title Container Diagram - ChargeLK Platform

    Person(user, "User", "EV Driver, Admin, or Operator")
    System_Ext(chargePoint, "Charge Point", "OCPP 1.6 Device")

    Container_Boundary(chargeLK, "ChargeLK Platform") {
        Container(mobileApp, "Mobile App", "React Native", "Charging sessions, account management")
        Container(adminUI, "Admin Portal", "React + TypeScript", "System administration and monitoring")
        
        Container(apiGateway, "API Gateway", "Future", "Request routing, auth, rate limiting")
        
        Container(ocppService, "OCPP Service", ".NET 9 Web API", "OCPP protocol, charge point management")
        Container(umService, "UM Service", ".NET 9 Web API", "User management, transactions, billing")
        
        ContainerDb(ocppDb, "OCPP Database", "PostgreSQL", "Charge points, OCPP tags, connector status")
        ContainerDb(ocppMongo, "OCPP Time-Series DB", "MongoDB", "Meter values, transaction logs")
        
        ContainerDb(umDb, "UM Database", "PostgreSQL", "Users, roles, transactions")
        ContainerDb(umMongo, "UM Document DB", "MongoDB", "Documents, audit logs")
        
        Container(serviceBus, "Message Bus", "Azure Service Bus", "Async event communication")
        Container(blobStorage, "Blob Storage", "Azure Blob Storage", "Document storage")
    }

    Rel(user, mobileApp, "Uses", "HTTPS")
    Rel(user, adminUI, "Uses", "HTTPS")
    
    Rel(mobileApp, umService, "API calls", "HTTPS/REST")
    Rel(adminUI, ocppService, "API calls", "HTTPS/REST")
    Rel(adminUI, umService, "API calls", "HTTPS/REST")
    
    Rel(chargePoint, ocppService, "OCPP messages", "WebSocket/JSON")
    
    Rel(ocppService, ocppDb, "Reads/Writes", "Entity Framework")
    Rel(ocppService, ocppMongo, "Writes time-series", "MongoDB Driver")
    Rel(ocppService, serviceBus, "Publishes events", "Azure SDK")
    Rel(ocppService, blobStorage, "Stores files", "Azure SDK")
    
    Rel(umService, umDb, "Reads/Writes", "Entity Framework")
    Rel(umService, umMongo, "Stores documents", "MongoDB Driver")
    Rel(umService, serviceBus, "Publishes/Subscribes", "Azure SDK")
    Rel(umService, blobStorage, "Stores files", "Azure SDK")
```

### Level 3: Component Diagram - OCPP Service

```mermaid
graph TB
    subgraph "ChargeLK.OCPP Service"
        subgraph "API Layer"
            RestAPI[REST API Controllers]
            WsEndpoint[WebSocket Endpoint]
            Middleware[Middleware Pipeline]
        end
        
        subgraph "Application Layer"
            ChargePointHandlers[Charge Point Handlers]
            TransactionHandlers[Transaction Handlers]
            OcppTagHandlers[OCPP Tag Handlers]
            EventPublishers[Event Publishers]
        end
        
        subgraph "Domain Layer"
            ChargePointEntity[Charge Point Entity]
            TransactionEntity[Transaction Entity]
            ConnectorEntity[Connector Entity]
            OcppTagEntity[OCPP Tag Entity]
        end
        
        subgraph "Infrastructure Layer"
            OcppMessageHandler[OCPP Message Handler]
            PostgresRepo[PostgreSQL Repositories]
            MongoRepo[MongoDB Repositories]
            ServiceBusClient[Service Bus Client]
            BlobClient[Blob Storage Client]
            UserServiceClient[User Service Client]
        end
    end
    
    WsEndpoint --> OcppMessageHandler
    RestAPI --> ChargePointHandlers
    RestAPI --> TransactionHandlers
    RestAPI --> OcppTagHandlers
    
    OcppMessageHandler --> ChargePointHandlers
    OcppMessageHandler --> TransactionHandlers
    
    ChargePointHandlers --> ChargePointEntity
    TransactionHandlers --> TransactionEntity
    OcppTagHandlers --> OcppTagEntity
    
    ChargePointHandlers --> PostgresRepo
    TransactionHandlers --> PostgresRepo
    TransactionHandlers --> MongoRepo
    
    EventPublishers --> ServiceBusClient
    ChargePointHandlers --> EventPublishers
    TransactionHandlers --> EventPublishers
```

### Level 3: Component Diagram - UM Service

```mermaid
graph TB
    subgraph "ChargeLK.UM Service"
        subgraph "API Layer"
            AuthAPI[Authentication API]
            UserAPI[User Management API]
            TransactionAPI[Transaction API]
            Middleware2[Middleware Pipeline]
        end
        
        subgraph "Application Layer"
            AuthHandlers[Authentication Handlers]
            UserHandlers[User Management Handlers]
            TransactionHandlers2[Transaction Handlers]
            NotificationHandlers[Notification Handlers]
        end
        
        subgraph "Domain Layer"
            UserEntity[User Entity]
            RoleEntity[Role Entity]
            TransactionEntity2[Transaction Entity]
            WalletEntity[Wallet Entity]
        end
        
        subgraph "Infrastructure Layer"
            IdentityService[ASP.NET Identity]
            PostgresRepo2[PostgreSQL Repositories]
            MongoRepo2[MongoDB Repositories]
            ServiceBusClient2[Service Bus Client]
            BlobClient2[Blob Storage Client]
            TokenService[JWT Token Service]
        end
    end
    
    AuthAPI --> AuthHandlers
    UserAPI --> UserHandlers
    TransactionAPI --> TransactionHandlers2
    
    AuthHandlers --> IdentityService
    AuthHandlers --> TokenService
    
    UserHandlers --> UserEntity
    TransactionHandlers2 --> TransactionEntity2
    TransactionHandlers2 --> WalletEntity
    
    UserHandlers --> PostgresRepo2
    TransactionHandlers2 --> PostgresRepo2
    TransactionHandlers2 --> MongoRepo2
    
    NotificationHandlers --> ServiceBusClient2
```

## Technology Stack

### Backend Services (.NET)

| Component | Technology | Version | Purpose |
|-----------|-----------|---------|---------|
| Framework | .NET | 8.0 | Application runtime |
| Web Framework | ASP.NET Core | 8.0 | Web API and WebSocket |
| ORM | Entity Framework Core | 8.0 | Database access |
| Identity | ASP.NET Identity | 8.0 | User authentication |
| Logging | Serilog | Latest | Structured logging |
| API Documentation | Swagger/OpenAPI | 3.0 | API documentation |
| Testing | xUnit | Latest | Unit and integration tests |

### Frontend (Admin Portal)

| Component | Technology | Version | Purpose |
|-----------|-----------|---------|---------|
| Framework | React | 18 | UI framework |
| Language | TypeScript | Latest | Type-safe development |
| Build Tool | Vite | Latest | Fast builds and HMR |
| UI Library | Material-UI (MUI) | 6.x | Component library |
| State Management | Zustand | 5.x | Lightweight state management |
| Data Fetching | TanStack Query | 5.x | Server state management |
| HTTP Client | Axios | Latest | API communication |
| Routing | React Router | 6.x | Client-side routing |
| Charts | Recharts | 2.x | Data visualization |

### Databases

| Database | Use Case | Data Type |
|----------|----------|-----------|
| PostgreSQL | Primary relational data | Users, charge points, transactions |
| MongoDB | Time-series & documents | Meter values, logs, audit trails |

### Cloud Infrastructure (Azure)

| Service | Purpose |
|---------|---------|
| Azure Service Bus | Message broker for event-driven communication |
| Azure Blob Storage | Document and file storage |
| Azure Container Registry | Docker image storage |
| Azure App Service / AKS | Application hosting |
| Azure Monitor | Application monitoring and logging |

### DevOps & Tools

| Tool | Purpose |
|------|---------|
| Docker | Containerization |
| Docker Compose | Local development orchestration |
| Git | Version control |
| GitHub Actions | CI/CD pipeline |

## Architecture Patterns

### 1. Clean Architecture (Backend Services)

```
┌─────────────────────────────────────────────┐
│          Presentation Layer (API)           │
│  - Controllers                              │
│  - Middleware                               │
│  - WebSocket Handlers                       │
└────────────────┬────────────────────────────┘
                 │
┌────────────────▼────────────────────────────┐
│        Application Layer                    │
│  - Use Cases (Handlers)                     │
│  - DTOs & Mapping                           │
│  - Interfaces                               │
└────────────────┬────────────────────────────┘
                 │
┌────────────────▼────────────────────────────┐
│           Domain Layer                      │
│  - Entities                                 │
│  - Domain Events                            │
│  - Business Rules                           │
└────────────────┬────────────────────────────┘
                 │
┌────────────────▼────────────────────────────┐
│       Infrastructure Layer                  │
│  - Database Context                         │
│  - Repositories                             │
│  - External Services                        │
│  - Message Bus                              │
└─────────────────────────────────────────────┘
```

### 2. CQRS (Command Query Responsibility Segregation)

Implemented using MediatR pattern:
- **Commands**: Write operations (Create, Update, Delete)
- **Queries**: Read operations (Get, List, Search)
- Separate models for read and write operations

### 3. Repository Pattern

Abstracts data access logic:
- Interface-based repositories in Application layer
- Concrete implementations in Infrastructure layer
- Enables testing with mock repositories

### 4. Event-Driven Architecture

Asynchronous communication between services:
- Domain events published to Azure Service Bus
- Event handlers subscribe to topics
- Ensures loose coupling and scalability

## Non-Functional Requirements

### Performance
- API response time: < 200ms (P95)
- WebSocket message processing: < 50ms
- Support 10,000+ concurrent WebSocket connections
- Database query optimization with indexes

### Scalability
- Horizontal scaling of API services
- Stateless service design
- Database read replicas for query scaling
- Message queue for async processing

### Reliability
- 99.9% uptime SLA
- Automatic retry mechanisms
- Circuit breaker pattern for external services
- Health checks and monitoring

### Security
- TLS 1.2+ for all communications
- JWT-based authentication
- Role-based authorization
- Input validation and sanitization
- Rate limiting and DDoS protection

### Maintainability
- Clean code principles
- Comprehensive documentation
- Automated testing (Unit, Integration, E2E)
- CI/CD automation
- Logging and monitoring

## Related Documentation

- [Components Documentation](../03-Components/README.md)
- [Data Architecture](../05-Data-Architecture/README.md)
- [Infrastructure & Deployment](../06-Infrastructure/README.md)

