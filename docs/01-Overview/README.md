# System Overview

## Introduction

ChargeLK is a comprehensive EV (Electric Vehicle) charging management solution designed to manage charging infrastructure, user accounts, transactions, and real-time communication with charging stations using the OCPP (Open Charge Point Protocol) standard.

## Business Context

The ChargeLK platform enables:
- **Charging Station Operators (CSOs)**: Manage and monitor their charging infrastructure
- **EV Drivers**: Locate chargers, initiate charging sessions, and manage their accounts
- **System Administrators**: Oversee the entire ecosystem, manage users, and monitor system health
- **Business Stakeholders**: Track revenue, usage analytics, and operational metrics

## System Vision

ChargeLK aims to provide a scalable, reliable, and standards-compliant EV charging platform that supports:
- Multi-tenant architecture for different operators
- Real-time monitoring and control of charging stations
- Seamless payment and billing integration
- Mobile-first user experience
- Cloud-native infrastructure with high availability

## Key Features

### For Charging Station Operators
- **Real-time Monitoring**: Live status of all charging points
- **Remote Management**: Control charging sessions, update configurations
- **OCPP 1.6 Compliance**: Standard protocol for interoperability
- **Transaction History**: Complete audit trail of all charging sessions
- **Analytics Dashboard**: Usage patterns, revenue reports, performance metrics

### For EV Drivers
- **Mobile Application**: Find nearby chargers, start/stop charging sessions
- **Account Management**: Top-up wallet, view charging history
- **RFID Card Support**: Contactless authentication at charging stations
- **Real-time Updates**: Session progress, cost calculation, notifications
- **Payment Integration**: Multiple payment methods support

### For System Administrators
- **Multi-service Management**: Control both OCPP and user management services
- **User & Role Management**: Fine-grained access control
- **Configuration Management**: System-wide settings and parameters
- **Health Monitoring**: Service health checks, performance metrics
- **Audit Logs**: Complete system activity tracking

## System Capabilities

### OCPP Protocol Support (ChargeLK.OCPP)
- WebSocket-based communication with charge points
- OCPP 1.6 JSON specification compliance
- Support for core OCPP operations:
  - BootNotification
  - Heartbeat
  - Authorize
  - StartTransaction / StopTransaction
  - MeterValues
  - StatusNotification
  - RemoteStartTransaction / RemoteStopTransaction
  - ChangeConfiguration
  - DataTransfer

### User & Transaction Management (ChargeLK.UM)
- ASP.NET Identity-based authentication
- JWT token-based authorization
- User registration and profile management
- Role-based access control (RBAC)
- Transaction processing and billing
- Payment gateway integration ready
- Notification services

### Administrative Portal (ChargeLK.AdminUI)
- Modern React-based web interface
- Real-time data updates with React Query
- Responsive Material-UI design
- Dashboard with key metrics
- Charge point management interface
- Transaction monitoring
- User administration

## Stakeholders

### Primary Users
1. **System Administrators**: Full system access, manage operators and configurations
2. **CSO Administrators**: Manage their organization's charging infrastructure
3. **CSO Operators**: Monitor and control charging stations
4. **EV Drivers**: End users who charge their vehicles

### System Integrators
1. **Charge Point Manufacturers**: Integrate their hardware with OCPP protocol
2. **Payment Gateway Providers**: Process financial transactions
3. **Third-party Services**: Map services, notification providers, analytics platforms

### Business Stakeholders
1. **Product Owners**: Define features and priorities
2. **DevOps Team**: Deploy and maintain infrastructure
3. **Support Team**: Handle user issues and system monitoring
4. **Compliance Officers**: Ensure regulatory compliance

## System Constraints

### Technical Constraints
- Must comply with OCPP 1.6 JSON specification
- Real-time WebSocket connections must support up to 10,000 concurrent charge points
- API response time must be < 200ms for 95th percentile
- System must support multi-tenancy with data isolation

### Business Constraints
- Must support local currency and payment methods
- Data retention policies must comply with GDPR and local regulations
- System must be available 99.9% of the time
- Must support horizontal scaling for growing infrastructure

### Security Constraints
- All communications must be encrypted (TLS 1.2+)
- Authentication tokens must expire and support refresh
- PCI DSS compliance for payment processing
- Role-based access control for all operations

## Success Metrics

1. **Operational Metrics**
   - System uptime: 99.9%
   - Average API response time: < 150ms
   - Concurrent charge point connections: 10,000+
   
2. **Business Metrics**
   - Number of active charging stations
   - Total charging sessions per month
   - Revenue per charging session
   - User growth rate

3. **User Experience Metrics**
   - Mobile app session start time: < 30 seconds
   - Admin portal load time: < 2 seconds
   - Transaction success rate: > 99.5%

## Related Documentation

- [System Architecture](../02-Architecture/README.md)
- [Components Documentation](../03-Components/README.md)
- [Development Guide](../07-Development/README.md)

