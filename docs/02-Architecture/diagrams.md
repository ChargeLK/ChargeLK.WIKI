# System Architecture Diagrams - Visual Summary

This document provides a comprehensive visual overview of the ChargeLK system architecture.

## Complete System Architecture

```mermaid
graph TB
    subgraph "External Systems"
        CP[Charge Points<br/>OCPP 1.6 Devices]
        MobileUsers[Mobile App Users<br/>EV Drivers]
        WebUsers[Web Users<br/>Admins & Operators]
    end
    
    subgraph "Frontend Layer"
        Mobile[Mobile Application<br/>React Native]
        AdminUI[Admin Portal<br/>React + TypeScript]
    end
    
    subgraph "API Gateway Layer (Future)"
        Gateway[API Gateway<br/>Authentication, Rate Limiting, Routing]
    end
    
    subgraph "Backend Services"
        direction TB
        OCPP[OCPP Service<br/>.NET 8 Web API<br/>OCPP 1.6 Protocol Handler]
        UM[UM Service<br/>.NET 8 Web API<br/>User & Transaction Management]
    end
    
    subgraph "Integration Layer"
        SB[Azure Service Bus<br/>Event-Driven Communication]
    end
    
    subgraph "Data Layer - OCPP"
        OCPP_PG[(PostgreSQL<br/>Charge Points, OCPP Tags,<br/>Transaction Metadata)]
        OCPP_MG[(MongoDB<br/>Meter Values Time-Series,<br/>OCPP Logs)]
    end
    
    subgraph "Data Layer - UM"
        UM_PG[(PostgreSQL<br/>Users, Wallets,<br/>Invoices, Sessions)]
        UM_MG[(MongoDB<br/>Audit Logs,<br/>Documents)]
    end
    
    subgraph "Storage & Cache"
        Blob[Azure Blob Storage<br/>Documents & Files]
        Redis[Redis Cache<br/>Session & Data Cache]
    end
    
    subgraph "Monitoring & Logging"
        AppIns[Application Insights<br/>APM & Telemetry]
        LogAnalytics[Log Analytics<br/>Centralized Logging]
        Monitor[Azure Monitor<br/>Alerts & Dashboards]
    end
    
    %% Connections
    CP -->|WebSocket<br/>OCPP Messages| OCPP
    MobileUsers --> Mobile
    WebUsers --> AdminUI
    
    Mobile -->|REST API| Gateway
    AdminUI -->|REST API| Gateway
    
    Gateway --> OCPP
    Gateway --> UM
    
    OCPP <-->|HTTP Client| UM
    UM <-->|HTTP Client| OCPP
    
    OCPP -->|Publish Events| SB
    UM -->|Publish Events| SB
    SB -->|Subscribe Events| OCPP
    SB -->|Subscribe Events| UM
    
    OCPP --> OCPP_PG
    OCPP --> OCPP_MG
    OCPP --> Blob
    OCPP --> Redis
    
    UM --> UM_PG
    UM --> UM_MG
    UM --> Blob
    UM --> Redis
    
    OCPP --> AppIns
    UM --> AppIns
    AppIns --> LogAnalytics
    AppIns --> Monitor
    
    style CP fill:#e1f5ff
    style Mobile fill:#fff9c4
    style AdminUI fill:#fff9c4
    style OCPP fill:#c8e6c9
    style UM fill:#c8e6c9
    style SB fill:#f8bbd0
    style OCPP_PG fill:#b39ddb
    style UM_PG fill:#b39ddb
    style OCPP_MG fill:#a5d6a7
    style UM_MG fill:#a5d6a7
```

## Technology Stack Overview

```mermaid
graph TB
    subgraph "Frontend Technologies"
        React[React 18]
        TS[TypeScript]
        MUI[Material-UI]
        Vite[Vite]
        TQ[TanStack Query]
        Zustand[Zustand]
    end
    
    subgraph "Backend Technologies"
        NET[.NET 9]
        EF[Entity Framework Core]
        Identity[ASP.NET Identity]
        Serilog[Serilog]
        MediatR[MediatR]
    end
    
    subgraph "Databases"
        PostgreSQL[PostgreSQL 16]
        MongoDB[MongoDB 7]
    end
    
    subgraph "Cloud Services"
        ServiceBus[Azure Service Bus]
        BlobStorage[Azure Blob Storage]
        AppInsights[Application Insights]
        KeyVault[Azure Key Vault]
    end
    
    subgraph "DevOps"
        Docker[Docker]
        K8s[Kubernetes]
        GitHub[GitHub Actions]
        Terraform[Terraform]
    end
    
    style React fill:#61dafb
    style NET fill:#512bd4
    style PostgreSQL fill:#336791
    style MongoDB fill:#47A248
    style ServiceBus fill:#0078d4
```

## Data Flow - Complete Transaction Lifecycle

```mermaid
sequenceDiagram
    autonumber
    participant CP as Charge Point
    participant OCPP as OCPP Service
    participant SB as Service Bus
    participant UM as UM Service
    participant DB_OCPP as OCPP Database
    participant DB_UM as UM Database
    participant User as Mobile App
    
    Note over CP,User: Phase 1: Connection & Registration
    CP->>OCPP: WebSocket Connect
    OCPP->>DB_OCPP: Check Registration
    OCPP-->>CP: Connection Accepted
    CP->>OCPP: BootNotification
    OCPP->>DB_OCPP: Update Charge Point Info
    OCPP->>SB: ChargePointConnected Event
    OCPP-->>CP: BootNotification Response
    
    Note over CP,User: Phase 2: Authorization
    User->>CP: Present RFID Tag
    CP->>OCPP: Authorize Request
    OCPP->>DB_OCPP: Check OCPP Tag
    OCPP->>UM: Verify User & Balance
    UM->>DB_UM: Get User & Wallet
    UM-->>OCPP: User Verified
    OCPP-->>CP: Authorized
    
    Note over CP,User: Phase 3: Start Transaction
    CP->>OCPP: StartTransaction
    OCPP->>DB_OCPP: Create Transaction
    OCPP->>SB: TransactionStarted Event
    OCPP-->>CP: Transaction ID
    SB->>UM: TransactionStarted Event
    UM->>DB_UM: Reserve Balance
    UM->>SB: BalanceReserved Event
    UM->>User: Push Notification
    
    Note over CP,User: Phase 4: Charging (Real-time)
    loop Every 60 seconds
        CP->>OCPP: MeterValues
        OCPP->>DB_OCPP: Store Time-Series Data
        OCPP->>SB: MeterValueReceived Event
        SB->>UM: MeterValueReceived Event
        UM->>UM: Calculate Current Cost
        UM->>User: Real-time Update
    end
    
    Note over CP,User: Phase 5: Stop Transaction
    User->>CP: Remove RFID / Unplug
    CP->>OCPP: StopTransaction
    OCPP->>DB_OCPP: Complete Transaction
    OCPP->>SB: TransactionCompleted Event
    OCPP-->>CP: Stop Confirmed
    
    Note over CP,User: Phase 6: Billing & Settlement
    SB->>UM: TransactionCompleted Event
    UM->>DB_UM: Calculate Final Cost
    UM->>DB_UM: Deduct from Wallet
    UM->>DB_UM: Create Invoice
    UM->>SB: PaymentProcessed Event
    UM->>User: Receipt via Email/Push
```

## Deployment Architecture - Production

```mermaid
graph TB
    subgraph "Internet"
        Users[End Users]
        ChargePoints[Charge Points]
    end
    
    subgraph "Azure Cloud - Production"
        subgraph "Front Door & WAF"
            FD[Azure Front Door<br/>Global Load Balancer]
            WAF[Web Application Firewall]
        end
        
        subgraph "AKS Cluster"
            direction TB
            LB[Load Balancer]
            
            subgraph "OCPP Pods"
                OCPP1[OCPP Pod 1]
                OCPP2[OCPP Pod 2]
                OCPP3[OCPP Pod N]
            end
            
            subgraph "UM Pods"
                UM1[UM Pod 1]
                UM2[UM Pod 2]
                UM3[UM Pod N]
            end
        end
        
        subgraph "Managed Databases"
            PG_OCPP[(Azure PostgreSQL<br/>Flexible Server<br/>OCPP DB)]
            PG_UM[(Azure PostgreSQL<br/>Flexible Server<br/>UM DB)]
            Cosmos[(Azure Cosmos DB<br/>MongoDB API)]
        end
        
        subgraph "Integration Services"
            ASB[Azure Service Bus<br/>Premium Tier]
            Blob[Azure Blob Storage<br/>Hot + Cool Tiers]
            Redis[Azure Cache<br/>for Redis]
            KV[Azure Key Vault<br/>Secrets Management]
        end
        
        subgraph "Monitoring Stack"
            AppIns[Application Insights]
            LA[Log Analytics]
            AM[Azure Monitor]
            Grafana[Grafana Dashboards]
        end
        
        subgraph "Static Content"
            CDN[Azure CDN]
            StaticWeb[Static Web Apps<br/>Admin UI]
        end
    end
    
    Users -->|HTTPS| FD
    ChargePoints -->|WSS| FD
    
    FD --> WAF
    WAF --> LB
    WAF --> CDN
    
    LB --> OCPP1
    LB --> OCPP2
    LB --> OCPP3
    LB --> UM1
    LB --> UM2
    
    CDN --> StaticWeb
    
    OCPP1 --> PG_OCPP
    OCPP1 --> Cosmos
    OCPP1 --> ASB
    OCPP1 --> Blob
    OCPP1 --> Redis
    OCPP1 --> KV
    
    UM1 --> PG_UM
    UM1 --> Cosmos
    UM1 --> ASB
    UM1 --> Blob
    UM1 --> Redis
    UM1 --> KV
    
    OCPP1 --> AppIns
    UM1 --> AppIns
    AppIns --> LA
    LA --> AM
    LA --> Grafana
    
    style FD fill:#0078d4
    style WAF fill:#ff6b6b
    style OCPP1 fill:#c8e6c9
    style UM1 fill:#c8e6c9
    style PG_OCPP fill:#336791
    style PG_UM fill:#336791
    style Cosmos fill:#0078d4
    style ASB fill:#f48fb1
    style CDN fill:#81c784
```

## Security Architecture

```mermaid
graph TB
    subgraph "Security Layers"
        direction TB
        
        subgraph "Network Security"
            WAF[Web Application Firewall<br/>DDoS Protection]
            TLS[TLS 1.2+ Encryption<br/>Certificate Management]
            VNet[Virtual Network<br/>Network Isolation]
        end
        
        subgraph "Application Security"
            Auth[JWT Authentication<br/>Token Validation]
            RBAC[Role-Based Access Control<br/>Authorization Policies]
            Input[Input Validation<br/>XSS & SQL Injection Prevention]
            RateLimit[Rate Limiting<br/>Brute Force Protection]
        end
        
        subgraph "Data Security"
            Encrypt[Data Encryption<br/>At Rest & In Transit]
            KV[Azure Key Vault<br/>Secrets Management]
            MaskData[Data Masking<br/>PII Protection]
            Audit[Audit Logging<br/>Compliance Tracking]
        end
        
        subgraph "OCPP Security"
            WSAuth[WebSocket Authentication<br/>Basic Auth + TLS]
            MsgSign[Message Validation<br/>Schema Verification]
            CPReg[Charge Point Registration<br/>Whitelist Management]
        end
        
        subgraph "Compliance"
            GDPR[GDPR Compliance<br/>Data Privacy]
            PCI[PCI DSS<br/>Payment Security]
            OCPP_Std[OCPP 1.6 Standard<br/>Security Profile]
        end
    end
    
    style WAF fill:#ff6b6b
    style Auth fill:#81c784
    style Encrypt fill:#64b5f6
    style GDPR fill:#ffb74d
```

## Event-Driven Architecture Flow

```mermaid
graph LR
    subgraph "Event Publishers"
        OCPP[OCPP Service]
        UM[UM Service]
    end
    
    subgraph "Azure Service Bus"
        direction TB
        TE[transaction-events Topic]
        CE[charging-events Topic]
        NE[notification-events Topic]
        UE[user-events Topic]
        
        TE_SUB1[ocpp-transaction-processor]
        TE_SUB2[um-transaction-processor]
        TE_SUB3[analytics-processor]
        
        CE_SUB1[um-charging-processor]
        CE_SUB2[mobile-updates]
        
        NE_SUB1[email-processor]
        NE_SUB2[push-processor]
        
        TE --> TE_SUB1
        TE --> TE_SUB2
        TE --> TE_SUB3
        
        CE --> CE_SUB1
        CE --> CE_SUB2
        
        NE --> NE_SUB1
        NE --> NE_SUB2
    end
    
    subgraph "Event Consumers"
        OCPP_C[OCPP Service]
        UM_C[UM Service]
        Analytics[Analytics Service]
        Mobile[Mobile Backend]
        Email[Email Service]
        Push[Push Service]
    end
    
    OCPP -->|TransactionStarted<br/>TransactionCompleted<br/>MeterValueReceived| TE
    OCPP -->|ConnectorStatusChanged<br/>ChargePointConnected| CE
    
    UM -->|NotificationRequested<br/>PaymentProcessed| NE
    UM -->|UserCreated<br/>WalletBalanceChanged| UE
    
    TE_SUB1 --> OCPP_C
    TE_SUB2 --> UM_C
    TE_SUB3 --> Analytics
    
    CE_SUB1 --> UM_C
    CE_SUB2 --> Mobile
    
    NE_SUB1 --> Email
    NE_SUB2 --> Push
    
    style TE fill:#f48fb1
    style CE fill:#ce93d8
    style NE fill:#90caf9
    style UE fill:#a5d6a7
```

## Clean Architecture Layers

```mermaid
graph TB
    subgraph "ChargeLK Services Architecture"
        direction TB
        
        subgraph "Presentation Layer"
            API[API Controllers<br/>REST Endpoints]
            WS[WebSocket Handler<br/>OCPP Protocol]
            MW[Middleware<br/>Authentication, Logging]
        end
        
        subgraph "Application Layer"
            Handlers[Command/Query Handlers<br/>MediatR Pattern]
            DTOs[DTOs & Mapping<br/>AutoMapper]
            Interfaces[Service Interfaces<br/>Contracts]
            Validators[Validators<br/>FluentValidation]
        end
        
        subgraph "Domain Layer"
            Entities[Domain Entities<br/>Business Objects]
            Events[Domain Events<br/>Event Sourcing]
            Rules[Business Rules<br/>Domain Logic]
            ValueObjects[Value Objects<br/>Immutable Types]
        end
        
        subgraph "Infrastructure Layer"
            DB[Database Context<br/>Entity Framework]
            Repos[Repositories<br/>Data Access]
            ExtServices[External Services<br/>HTTP Clients]
            MsgBus[Message Bus<br/>Service Bus Client]
            Storage[Storage Services<br/>Blob Storage]
        end
    end
    
    API --> Handlers
    WS --> Handlers
    MW --> API
    MW --> WS
    
    Handlers --> Interfaces
    Handlers --> DTOs
    Handlers --> Validators
    
    Handlers --> Entities
    Handlers --> Events
    
    Interfaces -.Implemented by.-> Repos
    Interfaces -.Implemented by.-> ExtServices
    Interfaces -.Implemented by.-> MsgBus
    
    Repos --> DB
    Repos --> Entities
    
    ExtServices --> Entities
    MsgBus --> Events
    
    style API fill:#c8e6c9
    style Handlers fill:#fff9c4
    style Entities fill:#f8bbd0
    style Repos fill:#b39ddb
```

---

## Diagram Legend

### Color Coding

| Color | Meaning |
|-------|---------|
| 🟢 Green | Backend Services |
| 🟡 Yellow | Frontend/UI |
| 🔵 Blue | Azure Services |
| 🟣 Purple | Databases |
| 🔴 Red | Security Components |
| 🟠 Orange | Monitoring/Logging |
| 🌸 Pink | Messaging/Events |

### Icons & Symbols

- **Cylinder**: Database
- **Rectangle**: Service/Application
- **Diamond**: Decision Point
- **Circle**: External System
- **Cloud**: Cloud Service
- **Arrows**: Data Flow Direction

---

## Related Documentation

- [Architecture Overview](./02-Architecture/README.md)
- [OCPP Protocol Flow](./02-Architecture/ocpp-protocol-flow.md)
- [Service Communication](./02-Architecture/service-communication.md)
- [Data Architecture](./05-Data-Architecture/README.md)
- [Infrastructure](./06-Infrastructure/README.md)

