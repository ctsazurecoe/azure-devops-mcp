# Azure Function Deployment Plan for Azure DevOps MCP Server

## Executive Summary

This document outlines the comprehensive plan to deploy the Azure DevOps MCP (Model Context Protocol) Server as an Azure Function, making it accessible to Copilot Studio for agentic operations. The plan addresses architecture, authentication, security, and operational considerations to ensure a secure, scalable, and reliable deployment.

## Table of Contents

1. [Current State Analysis](#current-state-analysis)
2. [Target Architecture](#target-architecture)
3. [Authentication Strategy](#authentication-strategy)
4. [Security Considerations](#security-considerations)
5. [Implementation Phases](#implementation-phases)
6. [Azure Resources Required](#azure-resources-required)
7. [Configuration Management](#configuration-management)
8. [Monitoring and Logging](#monitoring-and-logging)
9. [Cost Considerations](#cost-considerations)
10. [Testing Strategy](#testing-strategy)
11. [Rollback Plan](#rollback-plan)
12. [Post-Deployment Operations](#post-deployment-operations)

---

## Current State Analysis

### Existing Implementation

The Azure DevOps MCP Server is currently designed as a **local STDIO-based server** that:

- Runs as a Node.js application using `@modelcontextprotocol/sdk`
- Uses STDIO transport (`StdioServerTransport`) for communication
- Supports multiple authentication types:
  - Interactive OAuth (default for non-Codespaces environments)
  - Azure CLI credentials
  - Environment variables
  - Managed Identity (via DefaultAzureCredential)
- Implements 10+ domain areas with 100+ tools for Azure DevOps operations
- Requires Node.js 20+ and direct access to Azure DevOps APIs

### Key Components

1. **Entry Point**: `src/index.ts` - Configures and starts the MCP server
2. **Authentication**: `src/auth.ts` - Handles multiple auth types via MSAL and Azure Identity SDK
3. **Tools**: `src/tools/*.ts` - Implements domain-specific operations (core, work-items, repositories, pipelines, wiki, etc.)
4. **Domains Manager**: Controls which tool domains are loaded
5. **Logging**: Winston-based logger for diagnostics

### Current Limitations for Azure Function Deployment

1. **STDIO Transport**: Not compatible with HTTP-based Azure Functions
2. **Interactive Authentication**: Requires browser-based OAuth flow
3. **Stateful Operations**: Caches authentication tokens in memory
4. **Long-Running Operations**: Some Azure DevOps operations may exceed function timeout limits
5. **Direct Client Communication**: Designed for 1:1 client-server model

---

## Target Architecture

### High-Level Architecture

```
┌─────────────────────┐
│  Copilot Studio     │
│  (Agent)            │
└──────────┬──────────┘
           │ HTTPS
           │ (MCP over SSE)
           ▼
┌─────────────────────────────────────────┐
│   Azure API Management (Optional)       │
│   - Rate Limiting                       │
│   - API Key Validation                  │
│   - Request/Response Transformation     │
└──────────────────┬──────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│   Azure Function App                    │
│   ┌───────────────────────────────┐    │
│   │  HTTP Trigger Function        │    │
│   │  - MCP SSE Transport          │    │
│   │  - Session Management         │    │
│   │  - Request Validation         │    │
│   └───────────────┬───────────────┘    │
│                   │                     │
│   ┌───────────────▼───────────────┐    │
│   │  MCP Server Core              │    │
│   │  - Tool Execution             │    │
│   │  - Domain Management          │    │
│   │  - Response Formatting        │    │
│   └───────────────┬───────────────┘    │
│                   │                     │
│   ┌───────────────▼───────────────┐    │
│   │  Authentication Layer         │    │
│   │  - Service Principal          │    │
│   │  - Managed Identity           │    │
│   │  - User Delegation (Optional) │    │
│   └───────────────┬───────────────┘    │
└───────────────────┼─────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│   Azure DevOps REST APIs                │
│   - Work Items                          │
│   - Repositories                        │
│   - Pipelines                           │
│   - Wiki, Test Plans, etc.              │
└─────────────────────────────────────────┘

Supporting Services:
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│ Azure Key Vault  │  │ Application      │  │ Azure Storage    │
│ - Secrets        │  │ Insights         │  │ - Session State  │
│ - Certificates   │  │ - Monitoring     │  │ - Logs           │
└──────────────────┘  └──────────────────┘  └──────────────────┘
```

### MCP Communication Protocol

The function will implement **MCP over Server-Sent Events (SSE)** as per the Azure Functions MCP tutorial:

1. **SSE Endpoint**: `/sse` - Establishes long-lived connection for MCP messages
2. **HTTP POST Endpoint**: `/message` - Receives MCP requests
3. **Session Management**: Azure Table Storage or Redis for session state
4. **Message Format**: JSON-RPC 2.0 over SSE

### Function Configuration

- **Runtime**: Node.js 20 LTS
- **Plan Type**: Premium (EP1 or higher) for:
  - VNET integration
  - Longer execution times (up to 60 minutes)
  - Always-on capability
  - Better performance
- **Region**: Same as Azure DevOps organization (for latency)

---

## Authentication Strategy

### Multi-Layer Authentication

#### Layer 1: API Gateway Authentication (Copilot Studio → Function)

**Primary Approach: Azure AD OAuth 2.0**

```
┌─────────────────┐
│ Copilot Studio  │
└────────┬────────┘
         │ 1. Request with Bearer Token
         │    (Azure AD)
         ▼
┌─────────────────────────────────────┐
│ Azure Function                      │
│ ┌─────────────────────────────────┐│
│ │ Easy Auth / App Service Auth   ││
│ │ - Validates Azure AD token     ││
│ │ - Extracts user claims         ││
│ └─────────────────────────────────┘│
└─────────────────────────────────────┘
```

**Implementation Options:**

1. **Azure AD App Registration** (Recommended)
   - Register Function App in Azure AD
   - Configure Copilot Studio to authenticate with Azure AD
   - Use Easy Auth to validate tokens
   - Benefits:
     - Native Azure integration
     - Identity-based access control
     - Audit trail with user context
     - Works with Conditional Access policies

2. **API Management + Subscription Keys**
   - Deploy APIM in front of Function
   - Issue subscription keys to authorized consumers
   - Benefits:
     - Simpler setup for service-to-service
     - Built-in rate limiting and quotas
     - Transform requests/responses
   - Drawbacks:
     - Additional cost
     - Less granular user tracking

3. **Custom API Key Validation**
   - Store API keys in Key Vault
   - Validate in function code
   - Benefits:
     - Simple implementation
     - Full control
   - Drawbacks:
     - Manual key rotation
     - No built-in rate limiting
     - Limited audit capabilities

#### Layer 2: Azure DevOps API Authentication (Function → Azure DevOps)

**Primary Approach: Managed Identity with Service Principal**

```
┌─────────────────────────────────────┐
│ Azure Function                      │
│ ┌─────────────────────────────────┐│
│ │ Managed Identity Enabled       ││
│ └─────────────────────────────────┘│
└────────┬────────────────────────────┘
         │ 2. Authenticate with MI
         │    or Service Principal
         ▼
┌─────────────────────────────────────┐
│ Azure AD                            │
│ - Issues token for Azure DevOps    │
└────────┬────────────────────────────┘
         │ 3. Token for scope:
         │    499b84ac-1321-427f-aa17-267ca6975798/.default
         ▼
┌─────────────────────────────────────┐
│ Azure DevOps APIs                   │
└─────────────────────────────────────┘
```

**Implementation Options:**

1. **System-Assigned Managed Identity** (Recommended)
   - Enable on Function App
   - Grant Azure DevOps permissions in organization
   - Use `DefaultAzureCredential` in code
   - Benefits:
     - No credential management
     - Automatic rotation
     - Azure-native security
   - Setup:

     ```bash
     # Enable managed identity
     az functionapp identity assign --name <function-app-name> --resource-group <rg>

     # Get principal ID
     PRINCIPAL_ID=$(az functionapp identity show --name <function-app-name> --resource-group <rg> --query principalId -o tsv)

     # Grant Azure DevOps permissions
     # (Manual step in Azure DevOps Organization Settings → Users → Add)
     ```

2. **Service Principal with Certificate** (Alternative)
   - Create service principal
   - Store certificate in Key Vault
   - Benefits:
     - Can be used across multiple services
     - Granular permission control
   - Drawbacks:
     - Manual certificate rotation
     - More complex setup

3. **User Delegation with On-Behalf-Of Flow** (Advanced)
   - Function acts on behalf of authenticated user
   - Requires OAuth consent flow
   - Benefits:
     - User-level permissions
     - Full audit trail
   - Drawbacks:
     - Complex implementation
     - Requires user interaction for token
     - Not suitable for background operations

#### Recommended Strategy: Hybrid Approach

**For Production:**

```typescript
// Pseudo-code for authentication layer
async function authenticate(req: HttpRequest): Promise<AuthContext> {
  // Layer 1: Validate API gateway authentication
  const userToken = await validateAzureADToken(req.headers["authorization"]);

  // Layer 2: Get service credentials for Azure DevOps
  const credential = new DefaultAzureCredential();
  const adoToken = await credential.getToken("499b84ac-1321-427f-aa17-267ca6975798/.default");

  return {
    userId: userToken.claims.oid,
    userName: userToken.claims.preferred_username,
    adoCredential: adoToken,
    organization: req.query.organization || config.defaultOrganization,
  };
}
```

---

## Security Considerations

### Threat Model

#### Threats

1. **Unauthorized Access**
   - Attackers accessing the function without valid credentials
   - Lateral movement after initial access

2. **Credential Theft**
   - Exposure of Azure DevOps tokens or service principal credentials
   - Token replay attacks

3. **Data Exfiltration**
   - Unauthorized access to work items, repositories, or other sensitive data
   - Mass data extraction through API abuse

4. **Denial of Service**
   - Flooding the function with requests
   - Resource exhaustion

5. **Injection Attacks**
   - Malicious input in tool parameters
   - Code injection through work item content

6. **Man-in-the-Middle**
   - Interception of communication between Copilot Studio and Function

### Mitigations

#### 1. Authentication & Authorization

- **Implement Azure AD OAuth 2.0**
  - Enforce multi-factor authentication (MFA)
  - Use Conditional Access policies
  - Implement Just-In-Time (JIT) access for administrators

- **Principle of Least Privilege**
  - Grant minimal Azure DevOps permissions to managed identity
  - Use Azure RBAC for function app access
  - Implement attribute-based access control (ABAC) for specific tools

- **Token Management**
  - Store all secrets in Azure Key Vault
  - Enable Key Vault access via managed identity
  - Rotate keys regularly (90-day policy)
  - Use short-lived tokens (1-hour TTL)

#### 2. Network Security

- **VNET Integration**
  - Deploy function in Azure Virtual Network
  - Use private endpoints for Azure services
  - Implement Network Security Groups (NSGs)

- **IP Whitelisting**
  - Restrict function access to known Copilot Studio IP ranges
  - Use Azure Front Door or APIM for IP filtering

- **TLS/HTTPS Enforcement**
  - Require TLS 1.2 minimum
  - Use Azure-managed certificates
  - Implement HSTS headers

#### 3. Input Validation

- **Schema Validation**
  - Use Zod schemas for all tool inputs (already implemented)
  - Sanitize user input before Azure DevOps API calls
  - Implement length limits and character restrictions

- **Rate Limiting**
  - Implement per-user rate limits (e.g., 100 requests/minute)
  - Use Azure API Management for global rate limiting
  - Implement circuit breaker pattern for Azure DevOps API calls

- **Request Size Limits**
  - Limit request body size (e.g., 1MB max)
  - Implement pagination for large result sets

#### 4. Data Protection

- **Encryption at Rest**
  - Enable Azure Storage encryption for session state
  - Use encrypted Azure Table Storage or Cosmos DB

- **Encryption in Transit**
  - All communication over HTTPS/TLS
  - Encrypted connection to Azure DevOps APIs

- **Data Classification**
  - Tag sensitive operations (e.g., delete operations)
  - Implement additional approval workflows for sensitive actions
  - Log all data access for audit

#### 5. Monitoring & Incident Response

- **Comprehensive Logging**
  - Log all authentication attempts
  - Log all tool executions with user context
  - Send logs to Azure Log Analytics

- **Alerting**
  - Alert on failed authentication attempts (threshold: 5 in 5 minutes)
  - Alert on unusual patterns (e.g., mass data access)
  - Alert on error rate spikes

- **Security Information and Event Management (SIEM)**
  - Integrate with Azure Sentinel
  - Create detection rules for suspicious activity
  - Implement automated response playbooks

#### 6. Code Security

- **Dependency Scanning**
  - Use Dependabot or Snyk for vulnerability scanning
  - Regularly update dependencies
  - Pin dependency versions

- **Static Application Security Testing (SAST)**
  - Integrate CodeQL or SonarQube in CI/CD
  - Scan for common vulnerabilities (OWASP Top 10)

- **Secrets Scanning**
  - Use GitHub Advanced Security or git-secrets
  - Prevent commit of credentials

#### 7. Operational Security

- **Deployment Security**
  - Use Azure DevOps Pipelines with approval gates
  - Implement blue-green or canary deployments
  - Require signed commits

- **Access Control**
  - Use Azure AD PIM for privileged access
  - Implement break-glass procedures
  - Regular access reviews

### Security Checklist

- [ ] Azure AD authentication configured
- [ ] Managed Identity enabled and configured
- [ ] Key Vault integration implemented
- [ ] All secrets removed from code and config
- [ ] TLS 1.2+ enforced
- [ ] IP whitelisting configured
- [ ] Rate limiting implemented
- [ ] Input validation on all endpoints
- [ ] Comprehensive logging enabled
- [ ] Alerts configured for security events
- [ ] Vulnerability scanning in CI/CD
- [ ] Network isolation (VNET) configured
- [ ] Private endpoints for dependencies
- [ ] Regular security testing scheduled
- [ ] Incident response plan documented
- [ ] Access review process established

---

## Implementation Phases

### Phase 1: Infrastructure Setup (Week 1)

**Objectives:**

- Provision Azure resources
- Configure networking and security baseline

**Tasks:**

1. **Azure Resource Provisioning**
   - Create Resource Group
   - Create Azure Function App (Premium Plan)
   - Create Application Insights instance
   - Create Azure Key Vault
   - Create Storage Account for function and session state
   - (Optional) Create Azure API Management instance

2. **Configure Managed Identity**
   - Enable System-Assigned Managed Identity on Function App
   - Grant Key Vault access to Managed Identity
   - Register Managed Identity in Azure DevOps organization

3. **Network Configuration**
   - Create Virtual Network and subnets
   - Configure VNET integration for Function App
   - Set up Private Endpoints for Key Vault and Storage
   - Configure NSG rules

4. **Security Baseline**
   - Enable Azure Defender for Function Apps
   - Configure diagnostic settings
   - Set up Azure Monitor alerts
   - Configure log retention policies

**Deliverables:**

- Infrastructure as Code (Bicep/Terraform templates)
- Network architecture diagram
- Security configuration documentation

**Success Criteria:**

- All resources provisioned successfully
- Managed Identity can access Key Vault
- Network isolation verified
- Monitoring dashboards operational

---

### Phase 2: MCP Server Adaptation (Week 2-3)

**Objectives:**

- Modify MCP server to work with HTTP/SSE transport
- Implement session management
- Adapt authentication layer

**Tasks:**

1. **Create New Transport Layer**
   - Implement SSE transport based on `@modelcontextprotocol/sdk`
   - Create HTTP trigger function for SSE endpoint
   - Implement message POST endpoint
   - Add connection lifecycle management

   ```typescript
   // New file: src/transports/sse-transport.ts
   import { Transport } from "@modelcontextprotocol/sdk/shared/transport.js";

   export class SSEServerTransport implements Transport {
     // Implement SSE-based transport
   }
   ```

2. **Session Management**
   - Create session store using Azure Table Storage
   - Implement session creation, retrieval, and cleanup
   - Add session timeout handling (30-minute idle timeout)

   ```typescript
   // New file: src/session-manager.ts
   export class SessionManager {
     async createSession(userId: string): Promise<Session>;
     async getSession(sessionId: string): Promise<Session>;
     async updateSession(sessionId: string, data: Partial<Session>): Promise<void>;
     async deleteSession(sessionId: string): Promise<void>;
   }
   ```

3. **Adapt Authentication**
   - Replace interactive OAuth with Managed Identity
   - Implement token caching per session
   - Add user context to operations

   ```typescript
   // Modified: src/auth.ts
   export async function createFunctionAuthenticator(sessionId: string, userId: string): Promise<() => Promise<string>> {
     const credential = new DefaultAzureCredential();
     // Implement with token caching
   }
   ```

4. **Create Function Handlers**
   - Create HTTP trigger for SSE endpoint
   - Create HTTP trigger for message endpoint
   - Implement request validation middleware
   - Add error handling and logging

   ```typescript
   // New file: src/functions/mcp-sse.ts
   import { app, HttpRequest, HttpResponseInit, InvocationContext } from "@azure/functions";

   export async function mcpSSE(request: HttpRequest, context: InvocationContext): Promise<HttpResponseInit> {
     // SSE endpoint implementation
   }

   app.http("mcp-sse", {
     methods: ["GET"],
     authLevel: "anonymous", // Using Easy Auth
     handler: mcpSSE,
   });
   ```

5. **Configuration Management**
   - Create environment-based configuration
   - Load settings from App Configuration or environment variables
   - Support per-organization configuration

   ```typescript
   // New file: src/config.ts
   export interface FunctionConfig {
     defaultOrganization: string;
     sessionTimeoutMinutes: number;
     enabledDomains: string[];
     maxConcurrentRequests: number;
     rateLimitPerUser: number;
   }
   ```

6. **Update Package Dependencies**
   - Add `@azure/functions` package
   - Add `@azure/data-tables` for session management
   - Add `@azure/identity` (already present)
   - Update MCP SDK if needed

**Deliverables:**

- Modified source code in `src/`
- New function handlers
- Session management implementation
- Updated authentication layer
- Unit tests for new components

**Success Criteria:**

- Function can establish SSE connection
- Messages can be sent and received
- Sessions are created and managed correctly
- Authentication works with Managed Identity
- All existing tools work in new transport

---

### Phase 3: Security Hardening (Week 3-4)

**Objectives:**

- Implement authentication and authorization
- Add security controls
- Enable monitoring and alerting

**Tasks:**

1. **Configure Azure AD Authentication**
   - Register Function App in Azure AD
   - Configure Easy Auth / App Service Authentication
   - Set up token validation
   - Test authentication flow

2. **Implement Authorization**
   - Create authorization middleware
   - Implement role-based access control (RBAC)
   - Add tool-level permissions (e.g., read-only vs. admin)
   - Create permission configuration

   ```typescript
   // New file: src/middleware/authorization.ts
   export interface ToolPermission {
     toolName: string;
     requiredRole: "reader" | "contributor" | "admin";
   }

   export async function authorizeToolExecution(userId: string, toolName: string): Promise<boolean> {
     // Check user roles and tool permissions
   }
   ```

3. **Input Validation & Sanitization**
   - Add request validation middleware
   - Implement input sanitization for all parameters
   - Add rate limiting per user/session
   - Implement request size limits

4. **Configure IP Restrictions**
   - Set up IP whitelist in Function App settings
   - Document Copilot Studio IP ranges
   - Configure Azure Front Door if using for geo-distribution

5. **Enable Comprehensive Logging**
   - Configure structured logging with correlation IDs
   - Add security event logging
   - Set up log shipping to Log Analytics
   - Create audit log for all tool executions

6. **Set Up Monitoring & Alerting**
   - Create Application Insights queries for anomalies
   - Configure alerts for:
     - Failed authentication (>5 in 5 minutes)
     - High error rates (>10% over 5 minutes)
     - Unusual API call patterns
     - Performance degradation
   - Create dashboard for operations team

**Deliverables:**

- Authentication configuration
- Authorization middleware
- Security monitoring dashboard
- Alert rules documentation
- Incident response runbook

**Success Criteria:**

- Only authenticated requests succeed
- Unauthorized requests are blocked
- All security events are logged
- Alerts trigger on anomalies
- Performance meets SLA (p95 < 2s)

---

### Phase 4: Integration Testing (Week 4)

**Objectives:**

- Test end-to-end integration
- Validate security controls
- Performance testing

**Tasks:**

1. **End-to-End Testing**
   - Test all tool operations from Copilot Studio
   - Verify session management works correctly
   - Test token refresh scenarios
   - Validate error handling

2. **Security Testing**
   - Penetration testing (OWASP Top 10)
   - Authentication bypass attempts
   - Authorization boundary testing
   - Input validation testing (fuzzing)
   - Rate limiting verification

3. **Performance Testing**
   - Load testing (100 concurrent users)
   - Stress testing (identify breaking point)
   - Measure response times (target: p95 < 2s)
   - Test session scaling

4. **Disaster Recovery Testing**
   - Test failover scenarios
   - Verify backup and restore procedures
   - Test monitoring and alerting

**Deliverables:**

- Test results report
- Performance benchmark report
- Security test report
- Identified issues and remediation plan

**Success Criteria:**

- All functional tests pass
- No critical security vulnerabilities
- Performance meets targets
- Disaster recovery procedures validated

---

### Phase 5: Documentation & Deployment (Week 5)

**Objectives:**

- Complete documentation
- Deploy to production
- Train operations team

**Tasks:**

1. **Documentation**
   - Architecture documentation
   - Deployment guide
   - Operations runbook
   - Security configuration guide
   - Troubleshooting guide
   - API documentation for Copilot Studio integration

2. **Production Deployment**
   - Deploy to production environment
   - Configure production monitoring
   - Set up log retention
   - Configure backup policies

3. **Copilot Studio Integration**
   - Configure Copilot Studio to connect to function
   - Test integration from Copilot Studio
   - Document connection configuration
   - Create sample agents/actions

4. **Training & Handoff**
   - Train operations team on monitoring
   - Document incident response procedures
   - Create runbooks for common issues
   - Set up on-call rotation

**Deliverables:**

- Complete documentation set
- Production deployment
- Copilot Studio integration guide
- Operations training materials
- Handoff checklist

**Success Criteria:**

- Function deployed and operational
- Copilot Studio successfully connects
- Operations team trained
- All documentation complete

---

## Azure Resources Required

### Core Resources

1. **Azure Function App**
   - SKU: Premium Plan (EP1 minimum, EP2 recommended)
   - Runtime: Node.js 20 LTS
   - Region: Same as Azure DevOps organization
   - Estimated cost: $146-292/month (EP1-EP2)

2. **Application Insights**
   - For monitoring and diagnostics
   - Log retention: 90 days minimum
   - Estimated cost: $2.30/GB ingested

3. **Azure Key Vault**
   - Standard tier
   - For secret and certificate management
   - Estimated cost: $0.03 per 10,000 operations

4. **Azure Storage Account**
   - Standard tier (LRS or ZRS)
   - For function storage and session state
   - Estimated cost: $20/month (50GB + transactions)

5. **Virtual Network**
   - For network isolation
   - Includes subnets and NSGs
   - Cost: Minimal (data transfer charges only)

### Optional Resources

6. **Azure API Management** (Optional but Recommended)
   - Developer or Standard tier
   - For rate limiting, transformation, and monitoring
   - Estimated cost: $50-630/month

7. **Azure Front Door** (Optional)
   - For global distribution and DDoS protection
   - Estimated cost: $35/month + data transfer

8. **Azure Table Storage / Cosmos DB**
   - For session state (alternative to Storage Account tables)
   - Cosmos DB estimated cost: $24/month (400 RU/s)
   - Table Storage: Included in Storage Account

9. **Log Analytics Workspace**
   - For centralized logging
   - Estimated cost: $2.30/GB ingested

10. **Azure Sentinel** (Optional)
    - For advanced security monitoring
    - Estimated cost: $2.46/GB ingested

### Resource Group Structure

```
rg-mcp-azdevops-prod
├── func-mcp-azdevops-prod (Function App)
├── plan-mcp-azdevops-prod (App Service Plan - Premium)
├── appi-mcp-azdevops-prod (Application Insights)
├── kv-mcp-azdevops-prod (Key Vault)
├── st-mcp-azdevops-prod (Storage Account)
├── vnet-mcp-azdevops-prod (Virtual Network)
│   ├── snet-function (Subnet for Function)
│   └── snet-private-endpoints (Subnet for Private Endpoints)
├── apim-mcp-azdevops-prod (API Management - Optional)
└── law-mcp-azdevops-prod (Log Analytics Workspace)
```

### Infrastructure as Code Template (Bicep)

```bicep
// main.bicep
param location string = resourceGroup().location
param appName string = 'mcp-azdevops'
param environment string = 'prod'

// Function App
resource functionPlan 'Microsoft.Web/serverfarms@2022-03-01' = {
  name: 'plan-${appName}-${environment}'
  location: location
  sku: {
    name: 'EP1'
    tier: 'ElasticPremium'
  }
  kind: 'elastic'
  properties: {
    reserved: true
  }
}

resource functionApp 'Microsoft.Web/sites@2022-03-01' = {
  name: 'func-${appName}-${environment}'
  location: location
  kind: 'functionapp,linux'
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    serverFarmId: functionPlan.id
    siteConfig: {
      linuxFxVersion: 'NODE|20'
      alwaysOn: true
      ftpsState: 'Disabled'
      minTlsVersion: '1.2'
    }
    httpsOnly: true
  }
}

// Application Insights
resource appInsights 'Microsoft.Insights/components@2020-02-02' = {
  name: 'appi-${appName}-${environment}'
  location: location
  kind: 'web'
  properties: {
    Application_Type: 'web'
    RetentionInDays: 90
  }
}

// Key Vault
resource keyVault 'Microsoft.KeyVault/vaults@2022-07-01' = {
  name: 'kv-${appName}-${environment}'
  location: location
  properties: {
    tenantId: subscription().tenantId
    sku: {
      family: 'A'
      name: 'standard'
    }
    enableRbacAuthorization: true
    networkAcls: {
      defaultAction: 'Deny'
      bypass: 'AzureServices'
    }
  }
}

// Grant Function App access to Key Vault
resource keyVaultRoleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(keyVault.id, functionApp.id, 'Key Vault Secrets User')
  scope: keyVault
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '4633458b-17de-408a-b874-0445c86b69e6')
    principalId: functionApp.identity.principalId
    principalType: 'ServicePrincipal'
  }
}

// Storage Account
resource storageAccount 'Microsoft.Storage/storageAccounts@2022-09-01' = {
  name: 'st${appName}${environment}'
  location: location
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
  properties: {
    minimumTlsVersion: 'TLS1_2'
    allowBlobPublicAccess: false
    networkAcls: {
      defaultAction: 'Deny'
      bypass: 'AzureServices'
    }
  }
}

// Virtual Network
resource vnet 'Microsoft.Network/virtualNetworks@2022-07-01' = {
  name: 'vnet-${appName}-${environment}'
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: [
        '10.0.0.0/16'
      ]
    }
    subnets: [
      {
        name: 'snet-function'
        properties: {
          addressPrefix: '10.0.1.0/24'
          delegations: [
            {
              name: 'delegation'
              properties: {
                serviceName: 'Microsoft.Web/serverFarms'
              }
            }
          ]
        }
      }
      {
        name: 'snet-private-endpoints'
        properties: {
          addressPrefix: '10.0.2.0/24'
          privateEndpointNetworkPolicies: 'Disabled'
        }
      }
    ]
  }
}
```

---

## Configuration Management

### Environment Variables

The following environment variables should be configured in the Function App:

```bash
# Azure DevOps Configuration
AZURE_DEVOPS_ORGANIZATION=<default-org-name>
AZURE_DEVOPS_DOMAINS=core,work,work-items,repositories,pipelines,wiki

# Authentication
AZURE_CLIENT_ID=<managed-identity-client-id>  # Auto-populated
AZURE_TENANT_ID=<tenant-id>

# Session Management
SESSION_TIMEOUT_MINUTES=30
SESSION_STORAGE_CONNECTION_STRING=@Microsoft.KeyVault(SecretUri=https://<keyvault>.vault.azure.net/secrets/session-storage-connection/*)

# Security
ALLOWED_ORIGINS=https://copilotstudio.microsoft.com
RATE_LIMIT_PER_USER=100
RATE_LIMIT_WINDOW_MINUTES=1

# Monitoring
APPINSIGHTS_INSTRUMENTATIONKEY=<instrumentation-key>
APPLICATIONINSIGHTS_CONNECTION_STRING=<connection-string>
WEBSITE_NODE_DEFAULT_VERSION=~20

# Logging
LOG_LEVEL=info
LOG_RETENTION_DAYS=90

# Feature Flags
ENABLE_ADVANCED_SECURITY=true
ENABLE_RATE_LIMITING=true
ENABLE_TELEMETRY=true
```

### Application Settings

```json
{
  "version": "1.0",
  "functionAppName": "func-mcp-azdevops-prod",
  "runtime": "node",
  "runtimeVersion": "20",
  "extensionBundle": {
    "id": "Microsoft.Azure.Functions.ExtensionBundle",
    "version": "[4.*, 5.0.0)"
  },
  "managedDependency": {
    "enabled": true
  }
}
```

### Key Vault Secrets

Store the following secrets in Azure Key Vault:

- `session-storage-connection`: Connection string for session storage
- `azure-devops-pat`: Personal Access Token (if not using Managed Identity)
- `api-keys`: JSON array of valid API keys (if using API key auth)

---

## Monitoring and Logging

### Application Insights Metrics

**Key Metrics to Monitor:**

1. **Availability**
   - Uptime percentage (target: 99.9%)
   - Response time (p50, p95, p99)
   - Failed request rate

2. **Performance**
   - Average request duration
   - Function execution time
   - Cold start frequency
   - Memory usage

3. **Capacity**
   - Concurrent executions
   - Queue length
   - Throttling events

4. **Security**
   - Authentication failures
   - Authorization failures
   - Rate limit hits

5. **Business Metrics**
   - Tool execution counts by type
   - Unique users per day
   - Errors by tool type

### Logging Strategy

**Log Levels:**

- **Error**: System errors, unhandled exceptions
- **Warning**: Recoverable errors, rate limits hit
- **Info**: Request lifecycle, tool executions
- **Debug**: Detailed execution flow (dev/test only)

**Structured Logging Format:**

```json
{
  "timestamp": "2024-01-15T10:30:00.000Z",
  "level": "info",
  "correlationId": "abc-123-def",
  "userId": "user@contoso.com",
  "sessionId": "sess-456",
  "operation": "execute_tool",
  "tool": "list_work_items",
  "organization": "contoso",
  "duration": 245,
  "success": true,
  "message": "Tool executed successfully"
}
```

### Dashboards

**Operations Dashboard:**

- Request rate and error rate
- Response time distribution
- Active sessions
- Top users by request volume
- Azure DevOps API health

**Security Dashboard:**

- Failed authentication attempts
- Rate limit violations
- Unusual access patterns
- Geographic distribution of requests

### Alerts

**Critical Alerts (PagerDuty/On-Call):**

- Function app down (> 5 minutes)
- Error rate > 5% (5-minute window)
- Authentication service unavailable

**Warning Alerts (Email):**

- Error rate > 2% (15-minute window)
- p95 response time > 5 seconds
- High rate of rate limiting (indicates potential attack)
- Certificate expiring in < 30 days

---

## Cost Considerations

### Estimated Monthly Costs (Production)

| Resource                    | SKU/Tier     | Estimated Cost | Notes                     |
| --------------------------- | ------------ | -------------- | ------------------------- |
| Function App (EP1)          | Premium Plan | $146           | 210 execution hours/month |
| Application Insights        | Standard     | $50            | ~20GB logs/month          |
| Key Vault                   | Standard     | $5             | ~10K operations/month     |
| Storage Account             | Standard LRS | $20            | 50GB + transactions       |
| Log Analytics               | Standard     | $50            | ~20GB logs/month          |
| Virtual Network             | Standard     | $10            | Data transfer             |
| **Total (Base)**            |              | **$281/month** |                           |
| API Management (Optional)   | Developer    | $50            | Additional layer          |
| Azure Front Door (Optional) | Standard     | $35            | Global distribution       |
| **Total (with optional)**   |              | **$366/month** |                           |

### Cost Optimization Strategies

1. **Right-sizing**
   - Start with EP1, monitor usage
   - Upgrade to EP2 only if needed for performance
   - Consider Consumption plan for low-volume scenarios (not recommended for production)

2. **Log Retention**
   - Use 30-day retention for detailed logs
   - Archive to cheaper storage after 30 days
   - Use sampling for Application Insights (10% sample rate)

3. **Storage Optimization**
   - Use lifecycle management to archive old sessions
   - Clean up expired sessions regularly
   - Use Table Storage instead of Cosmos DB for session state

4. **Network Optimization**
   - Minimize cross-region traffic
   - Use CDN for static content (if any)
   - Batch Azure DevOps API calls when possible

### Budget Alerts

Set up budget alerts at:

- 50% of monthly budget
- 80% of monthly budget
- 100% of monthly budget

---

## Testing Strategy

### Unit Testing

**Scope:**

- Individual tool functions
- Authentication and authorization logic
- Session management
- Input validation

**Framework:** Jest (already in use)

**Coverage Target:** > 80%

**Example Test:**

```typescript
describe("SessionManager", () => {
  it("should create a new session", async () => {
    const manager = new SessionManager(mockTableClient);
    const session = await manager.createSession("user-123");

    expect(session.sessionId).toBeDefined();
    expect(session.userId).toBe("user-123");
    expect(session.createdAt).toBeDefined();
  });

  it("should expire sessions after timeout", async () => {
    const manager = new SessionManager(mockTableClient);
    const session = await manager.createSession("user-123");

    // Simulate timeout
    jest.advanceTimersByTime(31 * 60 * 1000); // 31 minutes

    const retrieved = await manager.getSession(session.sessionId);
    expect(retrieved).toBeNull();
  });
});
```

### Integration Testing

**Scope:**

- End-to-end tool execution
- SSE connection lifecycle
- Authentication flow
- Azure DevOps API integration

**Tools:**

- Postman/Newman for API testing
- Custom test harness for SSE

**Test Scenarios:**

- Happy path: Execute each tool successfully
- Error handling: Invalid inputs, API errors
- Authentication: Valid/invalid tokens, token refresh
- Session management: Create, resume, expire

### Security Testing

**Scope:**

- Authentication bypass attempts
- Authorization boundary testing
- Input validation (injection attacks)
- Rate limiting

**Tools:**

- OWASP ZAP for automated scanning
- Burp Suite for manual testing
- Custom scripts for rate limit testing

**Test Cases:**

- Try to access without authentication
- Try to access with expired token
- Try SQL injection in tool parameters
- Try to exceed rate limits
- Try to access other users' sessions

### Performance Testing

**Scope:**

- Load testing (concurrent users)
- Stress testing (breaking point)
- Endurance testing (sustained load)

**Tools:**

- Apache JMeter or Artillery.io
- Azure Load Testing

**Scenarios:**

1. **Baseline Load**
   - 10 concurrent users
   - Mixed tool operations
   - Duration: 30 minutes
   - Expected: < 1s p95 response time

2. **Peak Load**
   - 100 concurrent users
   - Mixed tool operations
   - Duration: 10 minutes
   - Expected: < 2s p95 response time

3. **Stress Test**
   - Gradually increase from 10 to 500 users
   - Identify breaking point
   - Measure recovery

**Success Criteria:**

- p50 response time < 500ms
- p95 response time < 2s
- p99 response time < 5s
- Error rate < 0.1%
- No memory leaks during endurance test

---

## Rollback Plan

### Rollback Triggers

Execute rollback if:

- Critical security vulnerability discovered
- Error rate > 10% for > 10 minutes
- Complete service outage
- Data integrity issues

### Rollback Procedure

**Automated Rollback (Azure DevOps Pipeline):**

```yaml
# Rollback stage in pipeline
- stage: Rollback
  condition: failed()
  jobs:
    - job: RollbackFunction
      steps:
        - task: AzureFunctionApp@2
          inputs:
            azureSubscription: "Production"
            appType: "functionAppLinux"
            appName: "func-mcp-azdevops-prod"
            package: "$(Pipeline.Workspace)/previous-release/*.zip"
```

**Manual Rollback Steps:**

1. **Identify previous stable version**

   ```bash
   az functionapp deployment list --name func-mcp-azdevops-prod --resource-group rg-mcp-azdevops-prod
   ```

2. **Deploy previous version**

   ```bash
   az functionapp deployment source config-zip \
     --resource-group rg-mcp-azdevops-prod \
     --name func-mcp-azdevops-prod \
     --src previous-version.zip
   ```

3. **Verify rollback**
   - Check Application Insights for error rate
   - Test basic functionality
   - Notify stakeholders

4. **Post-rollback**
   - Document reason for rollback
   - Create incident post-mortem
   - Plan fix for next deployment

### Blue-Green Deployment (Recommended)

Use deployment slots for zero-downtime rollback:

1. Deploy new version to staging slot
2. Test in staging
3. Swap slots (production ↔ staging)
4. If issues, swap back immediately

```bash
# Swap slots
az functionapp deployment slot swap \
  --resource-group rg-mcp-azdevops-prod \
  --name func-mcp-azdevops-prod \
  --slot staging \
  --target-slot production
```

---

## Post-Deployment Operations

### Day 1 Operations

**First 24 Hours:**

- [ ] Monitor error rates every hour
- [ ] Check authentication success rate
- [ ] Verify all tools are functioning
- [ ] Monitor response times
- [ ] Check for any security alerts
- [ ] Verify logging is working
- [ ] Test Copilot Studio integration

### Ongoing Operations

**Daily:**

- Review dashboard for anomalies
- Check error logs
- Monitor cost vs. budget

**Weekly:**

- Review security alerts
- Check certificate expiration dates
- Review performance trends
- Capacity planning review

**Monthly:**

- Access review (remove unused accounts)
- Security patch review
- Performance optimization review
- Cost optimization review
- Update runbooks based on incidents

**Quarterly:**

- Security assessment
- Disaster recovery drill
- Dependency updates
- Architecture review

### Incident Response

**Severity Levels:**

1. **Critical (SEV1)**: Complete outage, data breach
   - Response time: 15 minutes
   - Escalation: Immediate

2. **High (SEV2)**: Degraded service, authentication issues
   - Response time: 1 hour
   - Escalation: 2 hours

3. **Medium (SEV3)**: Individual tool failures
   - Response time: 4 hours
   - Escalation: Next business day

4. **Low (SEV4)**: Minor issues, feature requests
   - Response time: Next business day

**Incident Response Process:**

1. Detect and alert
2. Triage and assess severity
3. Engage appropriate team
4. Investigate and diagnose
5. Implement fix or rollback
6. Verify resolution
7. Post-incident review

### Maintenance Windows

**Schedule:** Monthly, 3rd Saturday, 2-4 AM UTC

**Activities:**

- Apply security patches
- Update dependencies
- Optimize performance
- Clean up old sessions and logs

---

## Integration with Copilot Studio

### Copilot Studio Configuration

**Prerequisites:**

- Copilot Studio Premium license
- Azure AD tenant with Function App

**Setup Steps:**

1. **Register Function as a Connector**
   - In Copilot Studio, go to Settings → Connectors
   - Add Custom Connector
   - Provide OpenAPI specification for function

2. **Configure Authentication**
   - Select OAuth 2.0
   - Provide Azure AD app registration details
   - Configure scopes and permissions

3. **Create Actions**
   - Map MCP tools to Copilot Studio actions
   - Configure input/output schemas
   - Test each action

**Example OpenAPI Spec (Partial):**

```yaml
openapi: 3.0.0
info:
  title: Azure DevOps MCP Server
  version: 1.0.0
servers:
  - url: https://func-mcp-azdevops-prod.azurewebsites.net
paths:
  /api/sse:
    get:
      summary: Establish MCP SSE connection
      security:
        - oauth2: []
      responses:
        "200":
          description: SSE connection established
          content:
            text/event-stream:
              schema:
                type: string
  /api/message:
    post:
      summary: Send MCP message
      security:
        - oauth2: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/MCPRequest"
      responses:
        "200":
          description: Message processed
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/MCPResponse"
components:
  securitySchemes:
    oauth2:
      type: oauth2
      flows:
        implicit:
          authorizationUrl: https://login.microsoftonline.com/common/oauth2/v2.0/authorize
          scopes:
            api://func-mcp-azdevops-prod/.default: Access MCP server
  schemas:
    MCPRequest:
      type: object
      properties:
        jsonrpc:
          type: string
        method:
          type: string
        params:
          type: object
    MCPResponse:
      type: object
      properties:
        jsonrpc:
          type: string
        result:
          type: object
```

### Sample Agent Configuration

**Example: Work Item Assistant**

```yaml
name: Azure DevOps Work Item Assistant
description: Helps users manage work items in Azure DevOps
triggers:
  - "Show my work items"
  - "Create a work item"
  - "Update work item"
actions:
  - name: list_my_work_items
    connector: azure-devops-mcp
    tool: list_my_work_items
    inputs:
      organization: ${organization}
      project: ${project}
  - name: create_work_item
    connector: azure-devops-mcp
    tool: create_work_item
    inputs:
      organization: ${organization}
      project: ${project}
      type: ${type}
      title: ${title}
      description: ${description}
```

---

## Appendices

### Appendix A: Reference Documentation

- [Azure Functions MCP Tutorial](https://learn.microsoft.com/en-us/azure/azure-functions/functions-mcp-tutorial?tabs=mcp-extension&pivots=programming-language-typescript)
- [Model Context Protocol Specification](https://spec.modelcontextprotocol.io/)
- [Azure Functions Best Practices](https://learn.microsoft.com/en-us/azure/azure-functions/functions-best-practices)
- [Azure DevOps REST API Reference](https://learn.microsoft.com/en-us/rest/api/azure/devops/)
- [Azure AD Authentication for Functions](https://learn.microsoft.com/en-us/azure/app-service/configure-authentication-provider-aad)
- [Copilot Studio Custom Connectors](https://learn.microsoft.com/en-us/microsoft-copilot-studio/custom-connectors)

### Appendix B: Glossary

- **MCP**: Model Context Protocol - Protocol for AI agents to access external context
- **SSE**: Server-Sent Events - Protocol for server push notifications
- **Managed Identity**: Azure AD identity managed by Azure platform
- **Easy Auth**: Built-in authentication for Azure App Service/Functions
- **APIM**: Azure API Management
- **VNET**: Virtual Network
- **NSG**: Network Security Group
- **PAT**: Personal Access Token
- **OAuth**: Open Authorization standard
- **RBAC**: Role-Based Access Control

### Appendix C: Support Contacts

- **Azure Support**: [Azure Portal Support](https://portal.azure.com/#blade/Microsoft_Azure_Support/HelpAndSupportBlade)
- **Copilot Studio Support**: [Microsoft Support](https://support.microsoft.com/)
- **Security Incidents**: security@contoso.com (replace with actual)
- **Operations Team**: ops@contoso.com (replace with actual)

### Appendix D: Change Log

| Version | Date       | Author        | Changes                 |
| ------- | ---------- | ------------- | ----------------------- |
| 1.0     | 2024-01-15 | Planning Team | Initial deployment plan |

---

## Summary and Next Steps

This deployment plan provides a comprehensive roadmap for deploying the Azure DevOps MCP Server as an Azure Function accessible to Copilot Studio. The plan prioritizes security through:

- Multi-layer authentication (Azure AD + Managed Identity)
- Network isolation (VNET integration, private endpoints)
- Comprehensive monitoring and alerting
- Defense-in-depth security controls

**Immediate Next Steps:**

1. **Review and Approval**
   - Security team review
   - Architecture review board approval
   - Budget approval

2. **Team Formation**
   - Assign project lead
   - Identify development team
   - Identify operations team
   - Engage security team

3. **Environment Setup**
   - Provision Azure subscription/resource group
   - Set up development environment
   - Configure CI/CD pipelines

4. **Proof of Concept**
   - Implement minimal SSE transport
   - Deploy to development environment
   - Test basic authentication flow
   - Validate one tool end-to-end

5. **Proceed with Phase 1**
   - Begin infrastructure provisioning
   - Follow implementation phases as outlined

**Success Criteria:**

The deployment will be considered successful when:

- Function is accessible from Copilot Studio with Azure AD authentication
- All existing tools work correctly in the new transport
- Security controls are verified and operational
- Performance meets SLA (p95 < 2s)
- Monitoring and alerting are functional
- Documentation is complete and teams are trained

**Estimated Timeline:** 5 weeks from start to production deployment

**Estimated Budget:** $366/month operational costs + one-time implementation costs

---

**Document Version:** 1.0  
**Last Updated:** January 15, 2024  
**Status:** Draft - Awaiting Review  
**Classification:** Internal Use Only
