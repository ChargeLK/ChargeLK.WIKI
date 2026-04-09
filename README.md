# ChargeLK System Documentation

Welcome to the ChargeLK system documentation repository. This wiki serves as the comprehensive knowledge base for the ChargeLK EV Charging Solution.

## 📚 Documentation Index

### 1. [System Overview](./docs/01-Overview/README.md)
- Introduction to ChargeLK
- Business Context
- Key Features
- System Capabilities
- Stakeholders

### 2. [System Architecture](./docs/02-Architecture/README.md)
- Architecture Overview
- C4 Model Diagrams
- Architecture Principles
- Technology Stack
- [OCPP Protocol Flow](./docs/02-Architecture/ocpp-protocol-flow.md)
- [Service Communication](./docs/02-Architecture/service-communication.md)

### 3. [Components Documentation](./docs/03-Components/README.md)
- ChargeLK.OCPP Microservice
- ChargeLK.UM Microservice
- ChargeLK.AdminUI
- Mobile Application

### 4. [API Reference](./docs/04-API-Reference/README.md)
- REST API Endpoints
- OCPP WebSocket Protocol
- Authentication & Authorization
- API Usage Examples

### 5. [Data Architecture](./docs/05-Data-Architecture/README.md)
- Database Schemas
- Data Models
- Data Flow Diagrams
- Storage Strategy

### 6. [Infrastructure](./docs/06-Infrastructure/README.md)
- Cloud Infrastructure
- Container Configuration
- Service Discovery
- Caching Strategy

### 7. [Development Guide](./docs/07-Development/README.md)
- **[User Secrets Setup Guide](./docs/07-Development/USER-SECRETS-SETUP.md)** - .NET User Secrets configuration
- **[Admin Portal Quick Start](./docs/07-Development/ADMIN-PORTAL-QUICK-START.md)** - React Admin Portal setup
- **[RTK Implementation Plan](./docs/07-Development/RTK-IMPLEMENTATION-PLAN.md)** - Redux Toolkit Query
- Local Development Setup
- Coding Standards
- Testing Strategy
- Contributing Guidelines

### 8. [Security](./docs/08-Security/README.md)
- **[Authentication & Authorization Guide](./docs/08-Security/AUTHENTICATION-AUTHORIZATION.md)** - Complete implementation guide
- **[Authentication Complete Guide](./docs/08-Security/AUTHENTICATION-COMPLETE.md)**
- **[Authentication Flow Diagrams](./docs/08-Security/AUTHENTICATION-FLOW-DIAGRAMS.md)**
- **[Microservices Authentication](./docs/08-Security/MICROSERVICES-AUTHENTICATION-IMPLEMENTATION.md)**
- **[Authentication Testing Guide](./docs/08-Security/AUTHENTICATION-TESTING-GUIDE.md)**
- **[Permission Management SQL](./docs/08-Security/PERMISSION-MANAGEMENT-SQL.md)**
- **[Permission Practical Examples](./docs/08-Security/PERMISSION-PRACTICAL-EXAMPLES.md)**
- **[Quick Reference - Auth](./docs/08-Security/QUICK-REFERENCE-AUTH.md)**
- Security Best Practices
- Security Implementation Checklist

### 9. [Operations & Deployment](./docs/09-Operations/README.md)
- **[Deployment Guide](./docs/09-Operations/DEPLOYMENT-GUIDE.md)** - Complete production deployment
- **[Deployment Quickstart](./docs/09-Operations/DEPLOYMENT-QUICKSTART.md)** - Fast deployment reference
- **[Deployment Cheatsheet](./docs/09-Operations/DEPLOYMENT-CHEATSHEET.md)** - Quick command reference
- Logging & Monitoring
- Health Checks
- Troubleshooting
- Performance Optimization

## 🚀 Quick Start Guides

### For Developers
1. **[Development Guide](./docs/07-Development/README.md)** - Set up your local environment
2. **[User Secrets Setup](./docs/07-Development/USER-SECRETS-SETUP.md)** - Configure development secrets
3. **[Admin Portal Quick Start](./docs/07-Development/ADMIN-PORTAL-QUICK-START.md)** - React development
4. **[Authentication Testing](./docs/08-Security/AUTHENTICATION-TESTING-GUIDE.md)** - Test authentication flows

### For DevOps
1. **[Deployment Quickstart](./docs/09-Operations/DEPLOYMENT-QUICKSTART.md)** - Deploy to production fast
2. **[Deployment Cheatsheet](./docs/09-Operations/DEPLOYMENT-CHEATSHEET.md)** - Common commands
3. **[Security Checklist](./docs/08-Security/SECURITY-IMPLEMENTATION-CHECKLIST.md)** - Pre-deployment security

### For Understanding the System
1. **[System Overview](./docs/01-Overview/README.md)** - What is ChargeLK?
2. **[Architecture Overview](./docs/02-Architecture/README.md)** - How it works
3. **[OCPP Protocol Flow](./docs/02-Architecture/ocpp-protocol-flow.md)** - Charging station communication

## 🔑 Key Documentation

### Authentication & Security
- [Authentication Flow Diagrams](./docs/08-Security/AUTHENTICATION-FLOW-DIAGRAMS.md) - Visual authentication flows
- [Microservices Authentication](./docs/08-Security/MICROSERVICES-AUTHENTICATION-IMPLEMENTATION.md) - Cross-service auth
- [Permission Practical Examples](./docs/08-Security/PERMISSION-PRACTICAL-EXAMPLES.md) - Real-world scenarios
- [Quick Reference - Auth](./docs/08-Security/QUICK-REFERENCE-AUTH.md) - Fast lookup guide

### Deployment & Operations
- [Deployment Guide](./docs/09-Operations/DEPLOYMENT-GUIDE.md) - Comprehensive deployment instructions
- [Deployment Quickstart](./docs/09-Operations/DEPLOYMENT-QUICKSTART.md) - Get to production fast
- [Deployment Cheatsheet](./docs/09-Operations/DEPLOYMENT-CHEATSHEET.md) - Essential commands

## 📖 Documentation Standards

- All diagrams use Mermaid syntax for version control and easy rendering
- API documentation follows OpenAPI 3.0 specification
- Architecture documentation follows C4 model principles
- Code examples are provided in both C# and TypeScript where applicable

## 🎯 Documentation Structure

```
ChargeLK.WIKI/
├── README.md (this file)
├── docs/
│   ├── 01-Overview/          # System introduction and capabilities
│   ├── 02-Architecture/      # System design and architecture
│   ├── 03-Components/        # Individual component documentation
│   ├── 04-API-Reference/     # API specifications
│   ├── 05-Data-Architecture/ # Database and data models
│   ├── 06-Infrastructure/    # Cloud and container setup
│   ├── 07-Development/       # Developer guides and setup
│   ├── 08-Security/          # Authentication, authorization, security
│   └── 09-Operations/        # Deployment and operations
```

## 🔄 Last Updated

**Version:** 1.1.0  
**Date:** December 22, 2025  
**Maintainers:** ChargeLK Development Team

## 📝 Recent Changes

- ✅ Reorganized documentation structure for better navigation
- ✅ Consolidated authentication documentation in Security section
- ✅ Added comprehensive deployment guides in Operations section
- ✅ Removed temporary bug fix documentation
- ✅ Enhanced quick start guides for different user roles
- ✅ Added quick reference links for commonly accessed documents

## 🤝 Contributing to Documentation

To contribute to this documentation:
1. Follow the existing folder structure
2. Use Markdown for all documentation
3. Include code examples where applicable
4. Update the relevant README when adding new documents
5. Keep documentation concise and actionable

