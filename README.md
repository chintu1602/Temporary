# 🚀 NutriAI Health Portal — Complete Deployment Guide

Complete deployment guide for the NutriAI Health Portal on Microsoft Azure. Covers prerequisites, Azure Portal UI deployment, Terraform deployment, application deployment, monitoring, and operational procedures.

> **Total deployment time: 4-6 hours (Portal UI) or 1-2 hours (Terraform)**

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Architecture Overview](#2-architecture-overview)
3. [Azure Account Setup](#3-azure-account-setup)
4. [Method A: Azure Portal UI Deployment](#4-method-a-azure-portal-ui-deployment)
5. [Method B: Terraform Deployment](#5-method-b-terraform-deployment)
6. [Container Image Build & Push](#6-container-image-build--push)
7. [Database Schema](#7-database-schema)
8. [Email Configuration](#8-email-configuration)
9. [SSL/TLS & Custom Domain](#9-ssltls--custom-domain)
10. [CI/CD Pipeline](#10-cicd-pipeline)
11. [Local Development](#11-local-development)
12. [Post-Deployment Verification](#12-post-deployment-verification)
13. [Troubleshooting](#13-troubleshooting)
14. [Operational Procedures](#14-operational-procedures)
15. [Cost Estimation](#15-cost-estimation)

---

## 1. Prerequisites

### Required Tools

| Tool | Version | Purpose |
|------|---------|---------|
| Azure CLI | ≥ 2.55 | Azure management (for Terraform/CI) |
| Terraform | ≥ 1.5.0 | Infrastructure as Code (Method B) |
| Docker | ≥ 24.0 | Container builds |
| Docker Compose | ≥ 2.20 | Local development |
| Node.js | ≥ 20.0 | Frontend build |
| Python | ≥ 3.11 | Backend development |
| Git | ≥ 2.40 | Version control |

### Required Azure Permissions

- **Subscription**: Contributor role
- **Azure AD**: Application Administrator (for Entra ID app registration)
- **Cognitive Services**: Approved Azure OpenAI access

### Installation Commands

```bash
# Azure CLI
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Terraform
wget https://releases.hashicorp.com/terraform/1.7.0/terraform_1.7.0_linux_amd64.zip
unzip terraform_1.7.0_linux_amd64.zip
sudo mv terraform /usr/local/bin/

# Docker
curl -fsSL https://get.docker.com | sudo sh

# Node.js (via nvm)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
nvm install 20
```

---

## 2. Architecture Overview

### Service Port Map

| Service | Port | Description |
|---------|------|-------------|
| API Gateway | 8000 | Entry point, JWT validation, request routing |
| Auth Service | 8001 | Authentication, registration, Microsoft SSO |
| Document Service | 8002 | File upload, Azure Blob, OCR trigger |
| Diet Service | 8003 | GPT-4 diet generation, PDF, Service Bus |
| Health Service | 8004 | Health metrics logging, chart data |
| Notification Service | 8005 | Service Bus consumer, email sending |
| Profile Service | 8006 | User profiles, allergies, medical info |
| Admin Service | 8007 | Admin dashboard, user management |
| Frontend (Dev) | 3000 | React development server |
| Frontend (Prod) | 80 | Nginx serving React build |
| PostgreSQL | 5432 | Shared database |

### Network Architecture

```
Internet
    ↓
[Azure Front Door + WAF]                ← Global edge, DDoS, caching
    ↓
[Application Gateway + WAF v2]          ← Regional WAF, SSL termination
    ↓
[Frontend App Service]  [Backend App Service]
                            ↓
                  [API Gateway :8000]
                            ↓
            ┌───────────────┼───────────────┐
       [Auth] [Documents] [Diet] [Health] [Notifications] [Profile] [Admin]
                            ↓
           ┌────────────────┼────────────────┐
   [PostgreSQL]    [Blob Storage]    [Service Bus] → Email
                            ↓
                  [Azure OpenAI GPT-4]
                  [Document Intelligence]
                  [Function App OCR]
```

### VNet Subnet Design

| Subnet | CIDR | Purpose | NSG |
|---|---|---|---|
| appgw-subnet | 10.0.1.0/24 | Application Gateway | appgw-nsg |
| appservice-subnet | 10.0.2.0/24 | Backend App Service VNet integration | appservice-nsg |
| function-subnet | 10.0.3.0/24 | Function App VNet integration | function-nsg |
| db-subnet | 10.0.4.0/24 | PostgreSQL private endpoint (delegated) | db-nsg |
| storage-subnet | 10.0.5.0/24 | Storage private endpoint | none |

### Authentication Flow

```
1. User submits login form
2. Auth Service validates credentials, generates JWT
3. JWT set as HttpOnly cookie via Set-Cookie header
4. Subsequent requests include cookie automatically (withCredentials)
5. API Gateway extracts JWT, validates, injects X-User-ID header
6. Downstream services trust X-User-ID from API Gateway
```

### Microsoft SSO Flow

```
1. User clicks "Sign in with Microsoft"
2. Frontend redirects to /auth/microsoft
3. Auth Service generates MSAL auth URL
4. User authenticates with Microsoft
5. Microsoft redirects to /auth/microsoft/callback
6. Auth Service exchanges code for tokens
7. User created/matched in DB, JWT cookie set
8. Redirect to /dashboard
```

---

## 3. Azure Account Setup

### Login and Set Subscription

```bash
az login
az account list --output table
az account set --subscription "YOUR_SUBSCRIPTION_ID"
az account show --output table
```

### Register Required Providers

```bash
az provider register --namespace Microsoft.Web
az provider register --namespace Microsoft.DBforPostgreSQL
az provider register --namespace Microsoft.Storage
az provider register --namespace Microsoft.ContainerRegistry
az provider register --namespace Microsoft.CognitiveServices
az provider register --namespace Microsoft.ServiceBus
az provider register --namespace Microsoft.KeyVault
az provider register --namespace Microsoft.Network
az provider register --namespace Microsoft.Insights
az provider register --namespace Microsoft.OperationalInsights
```

---

## 4. Method A: Azure Portal UI Deployment

Choose this method for first-time deployment or learning. For automated/repeatable deployment, use **Method B (Terraform)** below.

### Step 1: Create Resource Group

**Time: 2 minutes**

1. Go to **portal.azure.com**
2. Search **Resource groups** → click it
3. Click **+ Create**
4. Fill in:
   - **Subscription**: your subscription
   - **Resource group name**: `nutriai-rg`
   - **Region**: `East US`
5. Click **Review + Create** → **Create**

---

### Step 2: Create VNet, Subnets, and NSGs

**Time: 30 minutes**

#### 2.1 Create Virtual Network

1. Search **Virtual networks** → **+ Create**
2. **Basics tab**:
   - **Resource group**: `nutriai-rg`
   - **Name**: `nutriai-vnet`
   - **Region**: `East US`
3. **IP Addresses tab**: delete default, enter `10.0.0.0/16`. Delete any default subnet.
4. Click **Review + Create** → **Create**

#### 2.2 Create 5 Subnets

Open **nutriai-vnet** → **Subnets** → **+ Subnet** for each:

| Name | Address range | Special config |
|---|---|---|
| `appgw-subnet` | `10.0.1.0/24` | none |
| `appservice-subnet` | `10.0.2.0/24` | none |
| `function-subnet` | `10.0.3.0/24` | none |
| `db-subnet` | `10.0.4.0/24` | Delegation: `Microsoft.DBforPostgreSQL/flexibleServers` |
| `storage-subnet` | `10.0.5.0/24` | none |

#### 2.3 Create 4 NSGs with Rules

Search **Network security groups** → **+ Create** for each NSG. Resource group = `nutriai-rg`, region = `East US`.

**NSG: `appgw-nsg`** — Open → Inbound security rules → **+ Add** for each:

| Priority | Name | Source | Source port | Destination port | Protocol | Action |
|---|---|---|---|---|---|---|
| 100 | Allow-HTTP | Any | * | 80 | TCP | Allow |
| 110 | Allow-HTTPS | Any | * | 443 | TCP | Allow |
| 120 | Allow-AppGW-Management | Service Tag: GatewayManager | * | 65200-65535 | TCP | Allow |

**NSG: `appservice-nsg`** — Inbound rule:

| Priority | Name | Source | Source IPs | Destination port | Action |
|---|---|---|---|---|---|
| 100 | Allow-From-AppGW | IP Addresses | 10.0.1.0/24 | * | Allow |

**NSG: `function-nsg`** — Inbound rule:

| Priority | Name | Source | Source IPs | Destination port | Action |
|---|---|---|---|---|---|
| 100 | Allow-From-AppService | IP Addresses | 10.0.2.0/24 | * | Allow |

**NSG: `db-nsg`** — Inbound rules:

| Priority | Name | Source | Source IPs | Destination port | Action |
|---|---|---|---|---|---|
| 100 | Allow-PG-From-AppService | IP Addresses | 10.0.2.0/24 | 5432 | Allow |
| 110 | Allow-PG-From-Function | IP Addresses | 10.0.3.0/24 | 5432 | Allow |

#### 2.4 Attach NSGs to Subnets

Open **nutriai-vnet** → **Subnets** → click each subnet and set NSG:
- `appgw-subnet` → `appgw-nsg`
- `appservice-subnet` → `appservice-nsg`
- `function-subnet` → `function-nsg`
- `db-subnet` → `db-nsg`

---

### Step 3: Create Key Vault with Secrets

**Time: 25 minutes**

#### 3.1 Create Key Vault

1. Search **Key vaults** → **+ Create**
2. **Basics**: Resource group `nutriai-rg`, name `nutriai-keyvault-XXXX` (globally unique), region `East US`, tier `Standard`
3. **Access configuration**: Permission model = **Azure RBAC**
4. **Networking**: Public network access = **Disable**
5. Click **Review + Create** → **Create**

#### 3.2 Grant Yourself Key Vault Administrator

1. Open Key Vault → **Access control (IAM)** → **+ Add** → **Add role assignment**
2. **Role**: `Key Vault Administrator` → **Next**
3. **Members**: search your email → select → **Review + assign**

#### 3.3 Create Private Endpoint

1. Open Key Vault → **Networking** → **Private endpoint connections** → **+ Create**
2. Name: `nutriai-kv-pe`, RG: `nutriai-rg`, region: `East US`
3. Sub-resource: `vault`
4. VNet: `nutriai-vnet`, Subnet: `db-subnet`
5. **Review + Create** → **Create**

#### 3.4 Add All 7 Secrets

Open Key Vault → **Secrets** → **+ Generate/Import** for each:

| Secret Name | Initial Value |
|---|---|
| `DATABASE-URL` | placeholder (update in Step 5) |
| `SECRET-KEY` | 32-character random string |
| `STORAGE-CONNECTION-STRING` | placeholder (update in Step 4) |
| `OPENAI-KEY` | placeholder (update in Step 6) |
| `DOCUMENT-INTELLIGENCE-KEY` | placeholder (update in Step 7) |
| `ENTRA-CLIENT-SECRET` | placeholder (update in Step 14) |
| `SERVICE-BUS-CONNECTION-STRING` | placeholder (update in Step 11) |

---

### Step 4: Create Storage Account

**Time: 15 minutes**

#### 4.1 Create Storage Account

1. Search **Storage accounts** → **+ Create**
2. **Basics**: RG `nutriai-rg`, name `nutriaistoreXXXX`, region `East US`, Performance `Standard`, Redundancy `LRS`
3. **Advanced**: Uncheck **Allow Blob anonymous access**, TLS `1.2`
4. **Networking**: **Disable public access and use private access**
5. **Review + Create** → **Create**

#### 4.2 Create Container

Open storage account → **Containers** → **+ Container**:
- Name: `health-documents`
- Public access: `Private (no anonymous access)`

#### 4.3 Create Private Endpoint

Open storage account → **Networking** → **Private endpoint connections** → **+ Private endpoint**:
- Name: `nutriai-storage-pe`, RG: `nutriai-rg`
- Sub-resource: `blob`
- VNet: `nutriai-vnet`, Subnet: `storage-subnet`

#### 4.4 Update Key Vault

1. Storage account → **Access keys** → copy **Connection string** from key1
2. Key Vault → **Secrets** → **STORAGE-CONNECTION-STRING** → **+ New Version** → paste → **Create**

---

### Step 5: Create PostgreSQL Flexible Server

**Time: 20 minutes**

#### 5.1 Create Server

1. Search **Azure Database for PostgreSQL flexible servers** → **+ Create** → **Flexible server**
2. **Basics**:
   - RG: `nutriai-rg`
   - Server name: `nutriai-postgres-XXXX`
   - Region: `East US`
   - Version: `15`
   - Workload type: `Development`
   - Admin username: `nutriai_admin`
   - Password: strong password (SAVE IT)
3. **Networking**:
   - Connectivity: **Private access (VNet integration)**
   - VNet: `nutriai-vnet`, Subnet: `db-subnet`
4. **Security**: Verify **Require SSL** enabled
5. **Review + Create** → **Create** (5-10 min)

#### 5.2 Create Database

Open server → **Databases** → **+ Add** → Name: `nutriai` → **Save**

#### 5.3 Update Key Vault

Key Vault → **Secrets** → **DATABASE-URL** → **+ New Version** → paste:
```
postgresql://nutriai_admin:YOUR_PASSWORD@nutriai-postgres-XXXX.postgres.database.azure.com/nutriai?sslmode=require
```

---

### Step 6: Create Azure OpenAI

**Time: 15 minutes**

#### 6.1 Create OpenAI Resource

1. Search **Azure OpenAI** → **+ Create**
2. RG: `nutriai-rg`, region: `East US`, name: `nutriai-openai`, tier: `Standard S0`
3. **Network**: All networks
4. **Review + Submit** → **Create**

#### 6.2 Deploy GPT-4 Model

1. Open resource → **Go to Azure OpenAI Studio**
2. **Deployments** → **+ Create new deployment**
3. Model: `gpt-4`, Deployment name: `gpt-4`, Type: `Standard`
4. **Create**

#### 6.3 Update Key Vault

OpenAI → **Keys and Endpoint** → copy **KEY 1** and **Endpoint URL**.
Key Vault → **OPENAI-KEY** → **+ New Version** → paste key → **Create**

---

### Step 7: Create Document Intelligence

**Time: 10 minutes**

1. Search **Document Intelligence** → **+ Create**
2. RG: `nutriai-rg`, region: `East US`, name: `nutriai-docintel`, tier: `Standard S0`
3. **Review + Create** → **Create**
4. Open resource → **Keys and Endpoint** → copy **KEY 1** and **Endpoint URL**
5. Key Vault → **DOCUMENT-INTELLIGENCE-KEY** → **+ New Version** → paste → **Create**

---

### Step 8: Create Container Registry

**Time: 10 minutes**

1. Search **Container registries** → **+ Create**
2. RG: `nutriai-rg`, name: `nutriaiacrXXXX`, location: `East US`, SKU: `Standard`
3. **Review + Create** → **Create**
4. Open registry → **Access keys** → toggle **Admin user** to **Enabled**
5. Save: **Login server**, **Username**, **Password**

> Docker image build and push commands are in [Section 6](#6-container-image-build--push).

---

### Step 9: Create App Service Plan and Two Web Apps

**Time: 30 minutes**

#### 9.1 Create App Service Plan

1. Search **App Service plans** → **+ Create**
2. RG: `nutriai-rg`, name: `nutriai-asp`, OS: `Linux`, region: `East US`, pricing: **Premium V3 P2v3**
3. **Review + Create** → **Create**

#### 9.2 Create Frontend Web App

1. Search **App Services** → **+ Create** → **Web App**
2. **Basics**: RG `nutriai-rg`, name `nutriai-frontend-XXXX`, Publish `Container`, OS `Linux`, region `East US`, Plan `nutriai-asp`
3. **Container**: Source `Azure Container Registry`, Registry `nutriaiacrXXXX`, Image `nutriai-frontend`, Tag `latest`
4. **Review + Create** → **Create**

#### 9.3 Create Backend Web App

Same as 9.2 but name `nutriai-backend-XXXX` and Image `nutriai-backend`.

#### 9.4 Configure Backend VNet Integration

Open backend → **Networking** → **VNet integration** → **+ Add VNet integration**:
- VNet: `nutriai-vnet`, Subnet: `appservice-subnet` → **Connect**

#### 9.5 Enable Backend Managed Identity

Open backend → **Identity** → **System assigned** → **On** → **Save**

#### 9.6 Grant Key Vault Access to Backend Identity

Key Vault → **Access control (IAM)** → **+ Add** → **Add role assignment**:
- Role: `Key Vault Secrets User`
- Members: Managed identity → App Service → `nutriai-backend-XXXX`
- **Review + assign**

#### 9.7 Add Backend App Settings

Open backend → **Environment variables** → **+ Add** for each:

| Name | Value |
|---|---|
| `DATABASE_URL` | `@Microsoft.KeyVault(VaultName=nutriai-keyvault-XXXX;SecretName=DATABASE-URL)` |
| `SECRET_KEY` | `@Microsoft.KeyVault(VaultName=nutriai-keyvault-XXXX;SecretName=SECRET-KEY)` |
| `AZURE_STORAGE_CONNECTION_STRING` | `@Microsoft.KeyVault(VaultName=nutriai-keyvault-XXXX;SecretName=STORAGE-CONNECTION-STRING)` |
| `AZURE_OPENAI_KEY` | `@Microsoft.KeyVault(VaultName=nutriai-keyvault-XXXX;SecretName=OPENAI-KEY)` |
| `AZURE_DOCUMENT_INTELLIGENCE_KEY` | `@Microsoft.KeyVault(VaultName=nutriai-keyvault-XXXX;SecretName=DOCUMENT-INTELLIGENCE-KEY)` |
| `AZURE_SERVICE_BUS_CONNECTION_STRING` | `@Microsoft.KeyVault(VaultName=nutriai-keyvault-XXXX;SecretName=SERVICE-BUS-CONNECTION-STRING)` |
| `AZURE_OPENAI_ENDPOINT` | your OpenAI endpoint URL |
| `AZURE_OPENAI_DEPLOYMENT_NAME` | `gpt-4` |
| `AZURE_OPENAI_API_VERSION` | `2024-02-01` |
| `AZURE_STORAGE_CONTAINER_NAME` | `health-documents` |
| `AZURE_DOCUMENT_INTELLIGENCE_ENDPOINT` | your Document Intelligence endpoint |
| `AZURE_SERVICE_BUS_TOPIC_NAME` | `meal-reminders` |
| `AZURE_SERVICE_BUS_SUBSCRIPTION_NAME` | `email-sender` |
| `ALGORITHM` | `HS256` |
| `ACCESS_TOKEN_EXPIRE_MINUTES` | `60` |
| `WEBSITES_PORT` | `8000` |

Click **Apply** → **Confirm**

#### 9.8 Enable HTTPS Only

For both web apps: **Configuration** → **General settings** → **HTTPS Only** → **On** → **Save**

#### 9.9 Configure Frontend Backend URL

Frontend → **Environment variables** → **+ Add**:
- Name: `VITE_API_URL`
- Value: `https://nutriai-backend-XXXX.azurewebsites.net`

---

### Step 10: Create Function App

**Time: 20 minutes**

#### 10.1 Create Function Storage

Search **Storage accounts** → **+ Create** → RG `nutriai-rg`, name `nutriaifuncstorXXXX`, region `East US`, Standard LRS → **Create**

#### 10.2 Create Function App

1. Search **Function App** → **+ Create** → **Consumption** → **Select**
2. **Basics**: RG `nutriai-rg`, name `nutriai-funcapp-XXXX`, Runtime `Python 3.11`, region `East US`
3. **Storage**: `nutriaifuncstorXXXX`
4. **Review + Create** → **Create**

#### 10.3 VNet Integration

Function App → **Networking** → **VNet integration** → VNet `nutriai-vnet`, Subnet `function-subnet` → **Connect**

#### 10.4 Add App Settings

Function App → **Environment variables** → **+ Add**:

| Name | Value |
|---|---|
| `DATABASE_URL` | `@Microsoft.KeyVault(VaultName=nutriai-keyvault-XXXX;SecretName=DATABASE-URL)` |
| `AZURE_STORAGE_CONNECTION_STRING` | `@Microsoft.KeyVault(VaultName=nutriai-keyvault-XXXX;SecretName=STORAGE-CONNECTION-STRING)` |
| `AZURE_DOCUMENT_INTELLIGENCE_ENDPOINT` | your endpoint |
| `AZURE_DOCUMENT_INTELLIGENCE_KEY` | `@Microsoft.KeyVault(VaultName=nutriai-keyvault-XXXX;SecretName=DOCUMENT-INTELLIGENCE-KEY)` |

#### 10.5 Enable Managed Identity + KV Access

Function App → **Identity** → **System assigned** → **On** → **Save**
Then Key Vault → **IAM** → **+ Add role assignment** → `Key Vault Secrets User` → Managed identity → select Function App → **Assign**

#### 10.6 Deploy Function Code

From local terminal:
```bash
cd function_app/
func azure functionapp publish nutriai-funcapp-XXXX --python
```

---

### Step 11: Create Service Bus

**Time: 15 minutes**

#### 11.1 Create Namespace

Search **Service Bus** → **+ Create** → RG `nutriai-rg`, name `nutriai-servicebus-XXXX`, location `East US`, tier `Standard` → **Create**

#### 11.2 Create Topic

Open namespace → **Topics** → **+ Topic** → Name `meal-reminders`, Max size `1 GB` → **Create**

#### 11.3 Create Subscription

Open topic → **Subscriptions** → **+ Subscription** → Name `email-sender`, Max delivery count `5` → **Create**

#### 11.4 Update Key Vault

Namespace → **Shared access policies** → `RootManageSharedAccessKey` → copy **Primary Connection String**.
Key Vault → **SERVICE-BUS-CONNECTION-STRING** → **+ New Version** → paste → **Create**

---

### Step 12: Create Application Gateway with WAF

**Time: 30 minutes**

#### 12.1 Create Public IP

Search **Public IP addresses** → **+ Create** → RG `nutriai-rg`, name `nutriai-appgw-pip`, SKU `Standard`, Assignment `Static` → **Create**

#### 12.2 Create WAF Policy

1. Search **Web Application Firewall policies (WAF)** → **+ Create**
2. Policy for: `Regional WAF (Application Gateway)`
3. RG: `nutriai-rg`, name: `nutriai-appgw-waf`, location: `East US`
4. **Policy settings**: Mode `Prevention`
5. **Managed rules**: OWASP 3.2 (default)
6. **Review + Create** → **Create**

#### 12.3 Create Application Gateway

1. Search **Application gateways** → **+ Create**
2. **Basics**: RG `nutriai-rg`, name `nutriai-appgw`, region `East US`, tier `WAF V2`, autoscale `Yes` (min 1, max 3), WAF policy `nutriai-appgw-waf`, VNet `nutriai-vnet`, Subnet `appgw-subnet`
3. **Frontends**: Public, IP `nutriai-appgw-pip`
4. **Backends**: **Add a backend pool**:
   - Name: `nutriai-backend-pool`
   - Target type: `App Services` → `nutriai-backend-XXXX`
5. **Configuration**: **+ Add routing rule**:
   - Name: `nutriai-rule`, Priority: `100`
   - **Listener**: name `nutriai-listener`, Frontend `Public`, Port `80`, Protocol `HTTP`
   - **Backend targets**: type `Backend pool` → `nutriai-backend-pool`, **Backend settings**: **Add new**:
     - Name: `nutriai-http-settings`, Protocol `HTTPS`, Port `443`, Override hostname `Pick from backend target`
6. **Review + Create** → **Create** (10-15 min)

---

### Step 13: Create Front Door with WAF

**Time: 30 minutes**

#### 13.1 Create Front Door

1. Search **Front Door and CDN profiles** → **+ Create** → **Azure Front Door** → **Custom create**
2. **Basics**: RG `nutriai-rg`, name `nutriai-frontdoor`, tier `Premium`
3. **Endpoints**: **+ Add endpoint** → name `nutriai-endpoint` → **Add**
4. In endpoint, **+ Add route**:
   - Name: `nutriai-route`
   - Patterns: `/*`
   - Protocols: `HTTP and HTTPS`
   - Redirect: `Redirect to HTTPS`
   - Origin group: **Create new**:
     - Name: `nutriai-origin-group`
     - **+ Add origin**:
       - Name: `appgw-origin`, type `Custom`
       - Host name: Application Gateway public IP (from `nutriai-appgw-pip`)
       - HTTP port `80`, HTTPS port `443`, Priority `1`, Weight `1000`
   - Forwarding protocol: `HTTPS only`
5. **Security**: **+ Add policy**:
   - Name: `nutriai-fd-waf`, Mode: `Prevention`, Domains: select endpoint
6. **Review + Create** → **Create** (10-15 min)

#### 13.2 Add Managed Rule Sets

Search **Web Application Firewall policies** → `nutriai-fd-waf` → **Managed rules** → **+ Assign**:
- `Microsoft_DefaultRuleSet_2.0`
- `Microsoft_BotManagerRuleSet_1.0`

Click **Save**

#### 13.3 Save Front Door URL

Open `nutriai-frontdoor` → **Endpoints** → copy hostname (e.g., `nutriai-endpoint-XXXX.azurefd.net`). Save for Step 14.

---

### Step 14: Register App in Entra ID

**Time: 15 minutes**

#### 14.1 Create App Registration

1. Search **Microsoft Entra ID** → **App registrations** → **+ New registration**
2. Name: `NutriAI Health Portal`
3. Supported accounts: `Single tenant`
4. Redirect URI: Web → `https://nutriai-endpoint-XXXX.azurefd.net/auth/microsoft/callback`
5. **Register**

#### 14.2 Save IDs

From **Overview**, save:
- **Application (client) ID**
- **Directory (tenant) ID**

#### 14.3 Add API Permissions

1. **API permissions** → **+ Add a permission** → **Microsoft Graph** → **Delegated permissions**
2. Add: `User.Read`, `email`, `profile`
3. Click **Grant admin consent for [tenant]** → **Yes**

#### 14.4 Create Client Secret

1. **Certificates & secrets** → **Client secrets** → **+ New client secret**
2. Description: `nutriai-app-secret`, Expires: `24 months`
3. **Add** → **COPY THE VALUE IMMEDIATELY**

#### 14.5 Update Key Vault

Key Vault → **ENTRA-CLIENT-SECRET** → **+ New Version** → paste → **Create**

#### 14.6 Add Entra Settings to Backend

Backend → **Environment variables** → **+ Add**:

| Name | Value |
|---|---|
| `ENTRA_CLIENT_ID` | your Application (client) ID |
| `ENTRA_TENANT_ID` | your Directory (tenant) ID |
| `ENTRA_CLIENT_SECRET` | `@Microsoft.KeyVault(VaultName=nutriai-keyvault-XXXX;SecretName=ENTRA-CLIENT-SECRET)` |
| `ENTRA_REDIRECT_URI` | `https://nutriai-endpoint-XXXX.azurefd.net/auth/microsoft/callback` |

---

### Step 15: Create Monitoring

**Time: 20 minutes**

#### 15.1 Log Analytics Workspace

Search **Log Analytics workspaces** → **+ Create** → RG `nutriai-rg`, name `nutriai-log-workspace`, region `East US` → **Create**

#### 15.2 Application Insights

Search **Application Insights** → **+ Create** → RG `nutriai-rg`, name `nutriai-appinsights`, region `East US`, Mode `Workspace-based`, Workspace `nutriai-log-workspace` → **Create**

#### 15.3 Link to Backend

App Insights → **Overview** → copy **Connection String**.
Backend → **Environment variables** → **+ Add**:
- Name: `APPLICATIONINSIGHTS_CONNECTION_STRING`
- Value: paste connection string

#### 15.4 Diagnostic Settings

For each resource (backend, PostgreSQL, App Gateway, Service Bus), open it → **Diagnostic settings** → **+ Add diagnostic setting**:

| Resource | Logs to Enable |
|---|---|
| Backend App Service | AppServiceHTTPLogs, AppServiceConsoleLogs, AppServiceAppLogs, AllMetrics |
| PostgreSQL | PostgreSQLLogs, AllMetrics |
| Application Gateway | ApplicationGatewayAccessLog, ApplicationGatewayFirewallLog, AllMetrics |
| Service Bus | OperationalLogs, AllMetrics |

Destination: **Log Analytics workspace** → `nutriai-log-workspace`

#### 15.5 Create Alerts

**Create Action Group**: Monitor → **Alerts** → **Action groups** → **+ Create**:
- RG: `nutriai-rg`, name: `nutriai-ag`
- Notification: Email/SMS → your email

**Create 3 Alert Rules**: Monitor → **Alerts** → **+ Create** → **Alert rule** for each:

| Alert Name | Scope | Signal | Threshold |
|---|---|---|---|
| `HighCPU-nutriai` | Backend App Service | CPU Percentage | > 80% avg 5 min |
| `HighMemory-nutriai` | Backend App Service | Memory Percentage | > 85% avg 5 min |
| `HTTP5xx-nutriai` | Backend App Service | Http 5xx | > 10 count in 5 min |

Set Action group: `nutriai-ag`, severity `2 - Warning` → **Create**

---

### Step 16: PostgreSQL Backup Configuration

**Time: 5 minutes**

Open PostgreSQL server → **Backup and restore**:
- **Backup retention period**: `7 days` (default)
- **Backup redundancy**: `Geo-redundant` (recommended)
- **Save** if changed

---

## 5. Method B: Terraform Deployment

Use this for repeatable automation. Faster than UI for production deployments.

### 5.1 Create Terraform State Backend

```bash
az group create --name nutriai-terraform-state --location eastus

az storage account create \
  --name nutriaitfstate \
  --resource-group nutriai-terraform-state \
  --location eastus \
  --sku Standard_LRS \
  --encryption-services blob

ACCOUNT_KEY=$(az storage account keys list \
  --resource-group nutriai-terraform-state \
  --account-name nutriaitfstate \
  --query '[0].value' -o tsv)

az storage container create \
  --name tfstate \
  --account-name nutriaitfstate \
  --account-key $ACCOUNT_KEY
```

### 5.2 Configure Variables

```bash
cd terraform/
cp terraform.tfvars.example terraform.tfvars
```

Edit `terraform.tfvars`:

```hcl
project_name = "nutriai"
environment  = "prod"
location     = "eastus"
admin_email  = "your-admin@email.com"

openai_model_deployment = "gpt-4"

smtp_host     = "smtp.gmail.com"
smtp_port     = 587
smtp_username = "your-email@gmail.com"
smtp_password = "your-app-password"

app_service_sku = "P2v3"
postgres_sku    = "B_Standard_B1ms"
```

### 5.3 Deploy Infrastructure

```bash
terraform init
terraform validate
terraform plan -out=tfplan
terraform apply tfplan
terraform output -json > ../deployment-outputs.json
```

### 5.4 Module Dependency Graph

```
resource_group
    ├── networking (VNet, subnets, NSGs)
    │   ├── database (PostgreSQL in db-subnet)
    │   └── app_service (VNet integration)
    ├── key_vault (private endpoint in db-subnet)
    ├── storage (private endpoint in storage-subnet)
    ├── container_registry
    ├── cognitive_services (OpenAI + Document Intelligence)
    ├── service_bus
    ├── app_service (frontend + backend)
    ├── function_app
    ├── application_gateway
    ├── front_door
    ├── entra_id
    └── monitoring
```

---

## 6. Container Image Build & Push

### 6.1 Login to ACR

```bash
ACR_NAME="nutriaiacrXXXX"
az acr login --name $ACR_NAME
```

### 6.2 Build and Push Backend Image

```bash
docker build -f Dockerfile.backend -t ${ACR_NAME}.azurecr.io/nutriai-backend:latest .
docker push ${ACR_NAME}.azurecr.io/nutriai-backend:latest
```

### 6.3 Build and Push Frontend Image

```bash
cd frontend/
docker build -t ${ACR_NAME}.azurecr.io/nutriai-frontend:latest .
docker push ${ACR_NAME}.azurecr.io/nutriai-frontend:latest
```

### 6.4 Restart App Services to Pull New Images

```bash
az webapp restart --name nutriai-backend-XXXX --resource-group nutriai-rg
az webapp restart --name nutriai-frontend-XXXX --resource-group nutriai-rg
```

---

## 7. Database Schema

Tables are created automatically by SQLAlchemy `Base.metadata.create_all()` on first service startup.

### Tables

| Table | Used By | Description |
|---|---|---|
| `users` | Auth, Profile, Diet, Admin | User accounts |
| `patient_profiles` | Profile | Medical conditions, preferences |
| `food_allergies` | Profile, Diet | Allergen tracking |
| `documents` | Document, Diet, Admin | Uploaded medical documents |
| `diet_plans` | Diet, Admin | AI-generated diet plans |
| `health_logs` | Health, Admin | Daily health metrics |
| `meal_logs` | Health | Meal tracking |
| `notifications` | Notification | In-app notifications |

### Verify Schema (Optional)

Connect via psql from Cloud Shell or jumpbox in VNet:

```bash
psql "postgresql://nutriai_admin:PASSWORD@nutriai-postgres-XXXX.postgres.database.azure.com:5432/nutriai?sslmode=require"
\dt
\d users
\d diet_plans
```

---

## 8. Email Configuration

### Option A: Gmail SMTP

1. Enable 2-Factor Authentication on your Gmail account
2. Generate App Password: Google Account → Security → App passwords
3. Add to Backend App Service environment variables:

| Name | Value |
|---|---|
| `EMAIL_PROVIDER` | `smtp` |
| `SMTP_HOST` | `smtp.gmail.com` |
| `SMTP_PORT` | `587` |
| `SMTP_USERNAME` | `your-email@gmail.com` |
| `SMTP_PASSWORD` | your-app-password |
| `SMTP_FROM_EMAIL` | `your-email@gmail.com` |

### Option B: SendGrid

1. Create SendGrid account → generate API key
2. Add to Backend App Service environment variables:

| Name | Value |
|---|---|
| `EMAIL_PROVIDER` | `sendgrid` |
| `SENDGRID_API_KEY` | `SG.xxxxxxxxx` |
| `SENDGRID_FROM_EMAIL` | `noreply@nutriai-health.com` |

### Email Template

Notification service sends styled HTML emails with:
- Branded gradient header
- Green table: Foods to eat (with portions and timing)
- Red table: Foods to avoid (with reasons and risk levels)
- CTA button linking to portal

---

## 9. SSL/TLS & Custom Domain

### 9.1 Add Custom Domain to Front Door

1. Open `nutriai-frontdoor` → **Domains** → **+ Add**
2. Domain type: Custom domain → enter `app.yourdomain.com`
3. **DNS management**: Follow shown instructions to add CNAME record at your DNS provider
4. **Certificate type**: `AFD managed` (auto TLS)
5. **Add** — certificate provisions automatically once DNS validates

### 9.2 DNS Records

| Type | Name | Value |
|---|---|---|
| CNAME | app | `nutriai-endpoint-XXXX.azurefd.net` |
| TXT | _dnsauth.app | (verification token shown by Front Door) |

---

## 10. CI/CD Pipeline

Create `.github/workflows/deploy.yml`:

```yaml
name: Deploy NutriAI

on:
  push:
    branches: [main]

env:
  ACR_NAME: nutriaiacrXXXX
  BACKEND_APP: nutriai-backend-XXXX
  FRONTEND_APP: nutriai-frontend-XXXX

jobs:
  build-backend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Login to ACR
        uses: azure/docker-login@v1
        with:
          login-server: ${{ env.ACR_NAME }}.azurecr.io
          username: ${{ secrets.ACR_USERNAME }}
          password: ${{ secrets.ACR_PASSWORD }}
      - name: Build and Push
        run: |
          docker build -f Dockerfile.backend -t ${{ env.ACR_NAME }}.azurecr.io/nutriai-backend:${{ github.sha }} .
          docker push ${{ env.ACR_NAME }}.azurecr.io/nutriai-backend:${{ github.sha }}
      - name: Deploy to App Service
        uses: azure/webapps-deploy@v3
        with:
          app-name: ${{ env.BACKEND_APP }}
          images: ${{ env.ACR_NAME }}.azurecr.io/nutriai-backend:${{ github.sha }}

  build-frontend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - name: Build and Push
        run: |
          cd frontend
          docker build -t ${{ env.ACR_NAME }}.azurecr.io/nutriai-frontend:${{ github.sha }} .
          docker push ${{ env.ACR_NAME }}.azurecr.io/nutriai-frontend:${{ github.sha }}
      - name: Deploy to App Service
        uses: azure/webapps-deploy@v3
        with:
          app-name: ${{ env.FRONTEND_APP }}
          images: ${{ env.ACR_NAME }}.azurecr.io/nutriai-frontend:${{ github.sha }}
```

### Required GitHub Secrets

| Secret | Description |
|---|---|
| `ACR_USERNAME` | ACR admin username |
| `ACR_PASSWORD` | ACR admin password |

---

## 11. Local Development

### Run Full Stack with Docker Compose

```bash
docker compose up --build
```

Access:
- Frontend: http://localhost:3000
- API Gateway: http://localhost:8000

### Stop / Clean

```bash
docker compose down              # stop
docker compose down -v           # stop + remove volumes
```

### Run Single Service

```bash
cd services/diet-service
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
python main.py
```

### Sample .env for Local

```bash
DATABASE_URL=postgresql://nutriai_user:nutriai_password@localhost:5432/nutriai
SECRET_KEY=super-secret-jwt-key-for-development
AZURE_OPENAI_ENDPOINT=https://your-endpoint.openai.azure.com
AZURE_OPENAI_KEY=your-key
AZURE_OPENAI_DEPLOYMENT_NAME=gpt-4
AZURE_OPENAI_API_VERSION=2024-02-01
AZURE_STORAGE_CONNECTION_STRING=DefaultEndpointsProtocol=https;...
AZURE_STORAGE_CONTAINER_NAME=health-documents
AZURE_SERVICE_BUS_CONNECTION_STRING=Endpoint=sb://...
AZURE_SERVICE_BUS_TOPIC_NAME=meal-reminders
AZURE_SERVICE_BUS_SUBSCRIPTION_NAME=email-sender
ENTRA_CLIENT_ID=your-client-id
ENTRA_CLIENT_SECRET=your-client-secret
ENTRA_TENANT_ID=your-tenant-id
ENTRA_REDIRECT_URI=http://localhost:8000/auth/microsoft/callback
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USERNAME=your-email@gmail.com
SMTP_PASSWORD=your-app-password
```

---

## 12. Post-Deployment Verification

### Verify URLs and Functionality

1. Visit `https://nutriai-endpoint-XXXX.azurefd.net` → landing page loads
2. Visit `https://nutriai-backend-XXXX.azurewebsites.net/health/all` → JSON shows all services healthy
3. Test login with email/password
4. Test Microsoft SSO login
5. Upload a document → verify OCR processes within 1-2 minutes
6. Generate a diet plan → verify response includes `foods_to_eat` AND `foods_to_avoid`
7. Check Service Bus → Topics → `meal-reminders` → verify messages enqueued
8. Wait until next meal time → verify reminder email received

### Security Verification Checklist

- [ ] All 7 secrets exist in Key Vault
- [ ] Backend App Service has Managed Identity enabled
- [ ] Backend App Service has Key Vault Secrets User role
- [ ] PostgreSQL public access disabled
- [ ] Storage Account public access disabled
- [ ] Key Vault public access disabled
- [ ] 3 private endpoints created and approved (KV, Storage, PG)
- [ ] WAF on Application Gateway in Prevention mode
- [ ] WAF on Front Door with managed rules in Prevention mode
- [ ] 4 NSGs attached to correct subnets
- [ ] HTTPS Only enabled on both App Services
- [ ] PostgreSQL SSL required
- [ ] Backend VNet integrated with appservice-subnet
- [ ] Function App VNet integrated with function-subnet
- [ ] Application Insights connected to backend
- [ ] Diagnostic settings on all critical resources
- [ ] Action group created with admin email
- [ ] 3 metric alerts created

---

## 13. Troubleshooting

| Symptom | Cause | Resolution |
|---|---|---|
| Key Vault reference shows `@Microsoft.KeyVault(...)` literal text | Managed Identity missing role | Verify Backend has `Key Vault Secrets User` role on KV |
| App Service cannot reach PostgreSQL | VNet integration or NSG rule | Verify VNet integration connected; verify `db-nsg` allows 5432 from `10.0.2.0/24` |
| Front Door returns 502 | AppGW backend unhealthy | Verify AppGW backend pool shows healthy probes; check backend App Service running |
| OCR stuck in `processing` | Function App misconfig | Check Function App logs; verify `AZURE_DOCUMENT_INTELLIGENCE_KEY` and endpoint settings |
| Service Bus messages not delivered | Subscription missing or consumer stopped | Verify `email-sender` subscription under `meal-reminders` topic; check Notification service logs |
| Entra ID login fails with redirect_uri_mismatch | URI mismatch | Verify redirect URI in Entra App Registration exactly matches Front Door URL with `/auth/microsoft/callback` |
| Docker image pull fails | ACR admin disabled or credentials wrong | Verify ACR admin user enabled; verify App Service deployment credentials |
| WAF blocks legitimate request | Aggressive rule | Check WAF logs in Log Analytics; add custom rule exclusion in WAF policy |
| Emails not sending | SMTP/SendGrid misconfig | Verify Gmail App Password (not regular password); test SMTP creds locally |
| Backend `502 Bad Gateway` | Backend container crashed | Check `WEBSITES_PORT=8000` setting; check container logs via App Service → Log stream |
| Diet plan generation hangs | OpenAI quota or model name | Verify GPT-4 deployment name = `gpt-4`; check OpenAI quota usage |
| Health check `/health/all` shows service down | Microservice crashed inside supervisord | SSH into backend container → `supervisorctl status` |

### Useful Commands

```bash
# Stream backend logs
az webapp log tail --name nutriai-backend-XXXX --resource-group nutriai-rg

# Check backend app settings
az webapp config appsettings list --name nutriai-backend-XXXX --resource-group nutriai-rg

# Check Service Bus message counts
az servicebus topic subscription show \
  --name email-sender --topic-name meal-reminders \
  --namespace-name nutriai-servicebus-XXXX --resource-group nutriai-rg \
  --query countDetails
```

---

## 14. Operational Procedures

### Backup Database

```bash
# Manual backup
pg_dump "$DATABASE_URL" > backup_$(date +%Y%m%d_%H%M%S).sql

# Restore
psql "$DATABASE_URL" < backup_20260608_120000.sql
```

> Automated backups: 7 days retention configured in Step 16.

### Update Application

```bash
# Build new version
docker build -f Dockerfile.backend -t ${ACR_NAME}.azurecr.io/nutriai-backend:v2 .
docker push ${ACR_NAME}.azurecr.io/nutriai-backend:v2

# Update App Service
az webapp config container set \
  --name nutriai-backend-XXXX \
  --resource-group nutriai-rg \
  --docker-custom-image-name ${ACR_NAME}.azurecr.io/nutriai-backend:v2

az webapp restart --name nutriai-backend-XXXX --resource-group nutriai-rg
```

### Rollback

```bash
az webapp config container set \
  --name nutriai-backend-XXXX \
  --resource-group nutriai-rg \
  --docker-custom-image-name ${ACR_NAME}.azurecr.io/nutriai-backend:v1

az webapp restart --name nutriai-backend-XXXX --resource-group nutriai-rg
```

### Rotate Secrets

```bash
NEW_SECRET=$(openssl rand -base64 48)

az keyvault secret set \
  --vault-name nutriai-keyvault-XXXX \
  --name SECRET-KEY \
  --value "$NEW_SECRET"

az webapp restart --name nutriai-backend-XXXX --resource-group nutriai-rg
```

### Scale Up / Out

```bash
# Scale up (more powerful)
az appservice plan update --name nutriai-asp --resource-group nutriai-rg --sku P3v3

# Scale out (more instances)
az webapp update --name nutriai-backend-XXXX --resource-group nutriai-rg \
  --set siteConfig.numberOfWorkers=3
```

### Destroy Infrastructure

```bash
cd terraform/
terraform plan -destroy
terraform destroy
az group delete --name nutriai-terraform-state --yes
```

---

## 15. Cost Estimation

### Monthly Cost Breakdown (Production - East US)

| Service | SKU | Monthly USD |
|---|---|---|
| App Service Plan | P2v3 Linux | $146 |
| Frontend Web App | Included | $0 |
| Backend Web App | Included | $0 |
| PostgreSQL Flexible | Burstable B1ms | $25 |
| Storage Account | Standard LRS | $5 |
| Function App | Consumption | $5 |
| Azure OpenAI (GPT-4) | Pay-per-token | $50-200 |
| Document Intelligence | S0 | $30 |
| Service Bus | Standard | $10 |
| Application Gateway | WAF v2 | $250 |
| Front Door | Premium | $330 |
| Key Vault | Standard | $5 |
| Application Insights | Pay-as-you-go | $20 |
| Log Analytics | Pay-as-you-go | $15 |
| Container Registry | Standard | $20 |
| Private Endpoints (3) | - | $22 |
| Public IP (Static) | Standard | $4 |
| **Total** | | **~$937-1087/mo** |

### Cost Optimization Tips

1. **Dev/Staging**: Use B2 App Service + B_Standard_B1ms PostgreSQL (~$150/mo total)
2. **OpenAI**: Use `gpt-4o-mini` instead of GPT-4 for 90% cost reduction
3. **Front Door**: Use `Standard` tier instead of `Premium` if managed rules not needed (saves $250/mo)
4. **App Gateway**: Use `Standard_v2` instead of `WAF_v2` if WAF not needed (saves $150/mo)
5. **Log Analytics**: Reduce retention to 7 days in non-prod
6. **Reserved Instances**: 1-year reservation saves 30-40% on App Service and PostgreSQL

---

<div align="center">
  <b>🏥 NutriAI Health Portal — Deployment Guide</b><br>
  <sub>Complete Azure Portal UI + Terraform deployment reference</sub>
</div>
