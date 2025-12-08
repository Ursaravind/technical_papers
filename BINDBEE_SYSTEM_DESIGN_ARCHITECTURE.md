# BindBee System Design Architecture

**Version**: 1.0  
**Last Updated**: December 8, 2025  
**Author**: System Architecture Documentation

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [System Overview](#2-system-overview)
3. [Component Architecture](#3-component-architecture)
4. [Data Flow Architecture](#4-data-flow-architecture)
5. [Infrastructure & Deployment](#5-infrastructure--deployment)
6. [Security Architecture](#6-security-architecture)
7. [Integration Patterns](#7-integration-patterns)
8. [Scalability & Performance](#8-scalability--performance)

---

## 1. Executive Summary

### 1.1 What is BindBee?

BindBee is a **unified API platform** that provides seamless integrations with 50+ HRIS (Human Resource Information Systems), ATS (Applicant Tracking Systems), and LMS (Learning Management Systems). It acts as a middleware layer that abstracts the complexity of multiple third-party integrations into a single, consistent API.

### 1.2 Core Value Proposition

- **Single API, Multiple Integrations**: Customers integrate once with BindBee's unified API instead of building 50+ separate integrations
- **Data Normalization**: Transforms vendor-specific data formats into a standardized schema
- **Real-time Sync**: Automated background workers keep data synchronized
- **Embedded Auth Flow**: White-labeled authentication interface for end-users

### 1.3 Key Metrics

- **Integrations Supported**: 50+ (BambooHR, Workday, ADP, Greenhouse, Lever, etc.)
- **Categories**: HRIS, ATS, LMS
- **Architecture**: Microservices-based event-driven system
- **Tech Stack**: FastAPI (Python), Next.js (TypeScript), PostgreSQL, RabbitMQ

---

## 2. System Overview

### 2.1 High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                      CUSTOMER APPLICATION                         │
│           (Uses BindBee Unified API for HR Data)                 │
└────────────────┬────────────────────────┬──────────────────────┘
                 │                         │
                 │ API Calls               │ Embed Magic Link
                 ▼                         ▼
    ┌─────────────────────┐   ┌────────────────────────┐
    │  unifyx-backend     │   │    link.unifyx         │
    │  (FastAPI)          │   │    (Next.js)           │
    │  Port: 8000         │   │    Port: 3001          │
    │                     │   │                        │
    │  • Unified APIs     │   │  • OAuth Flow UI       │
    │  • Connector Mgmt   │   │  • Integration Setup   │
    │  • Auth & Security  │   │  • White-labeled       │
    └──────────┬──────────┘   └────────────────────────┘
               │
               ▼
    ┌──────────────────────────────────────────────┐
    │           PostgreSQL Database                 │
    │  • Unified Data Schema (hris_*, ats_*, lms_*) │
    │  • Connector Configurations                   │
    │  • Job Queue (connector_job)                  │
    └──────────┬───────────────────────────────────┘
               │
               ▼
    ┌──────────────────────────────────────────────┐
    │              RabbitMQ                         │
    │         Message Queue/Broker                  │
    └──────────┬───────────────────────────────────┘
               │
               ▼
    ┌─────────────────────────────────────────────┐
    │              workers                         │
    │           (Background Sync)                  │
    │                                              │
    │  ┌──────────────┐    ┌──────────────┐      │
    │  │  scheduler   │───▶│    worker    │      │
    │  │ (scheduler.py)│   │  (main.py)   │      │
    │  └──────────────┘    └──────┬───────┘      │
    │                              │              │
    │                              ▼              │
    │         External Services (bamboohr.py)    │
    │         Internal Services (unification)    │
    └──────────────────┬──────────────────────────┘
                       │
                       ▼
           ┌────────────────────────┐
           │   Third-Party APIs     │
           │  • BambooHR            │
           │  • Workday             │
           │  • Greenhouse          │
           │  • 50+ more...         │
           └────────────────────────┘

    ┌─────────────────────────────────────────────┐
    │          app.unifyx (Dashboard)             │
    │            (Next.js - Port: 3000)           │
    │                                             │
    │  • Customer Dashboard                       │
    │  • Analytics & Monitoring                   │
    │  • Connector Management UI                  │
    │  • API Key Management                       │
    └─────────────────────────────────────────────┘
```

### 2.2 Component Summary

| Component | Technology | Purpose | Port |
|-----------|-----------|---------|------|
| **unifyx-backend** | FastAPI (Python) | Main API server, serves unified data | 8000 |
| **workers** | Dramatiq + RabbitMQ | Background sync from third-party systems | N/A |
| **app.unifyx** | Next.js 14 | Customer-facing dashboard | 3000 |
| **link.unifyx** | Next.js 13 | Embedded authentication/setup flow | 3001 |
| **PostgreSQL** | Database | Stores unified data & configurations | 5432 |
| **RabbitMQ** | Message Broker | Job queue for workers | 5672 |

---

## 3. Component Architecture

### 3.1 unifyx-backend (Main API Server)

**Technology**: FastAPI (Python 3.11+)  
**Responsibility**: Serve unified APIs to customers

#### 3.1.1 Layered Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    ROUTER LAYER                          │
│                  app/router/                             │
│  • HTTP endpoints                                        │
│  • Authentication (Depends)                              │
│  • Request validation                                    │
│                                                          │
│  Files:                                                  │
│  - hris/v1/employee.py    → GET /api/hris/v1/employees  │
│  - ats/v1/candidate.py    → GET /api/ats/v1/candidates  │
│  - common/v1/passthrough.py → POST /api/common/v1/*     │
└──────────────────┬──────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────┐
│              INTERNAL SERVICE LAYER                      │
│              app/services/internal/                      │
│  • Business logic                                        │
│  • Data orchestration                                    │
│  • Validation & transformation                           │
│                                                          │
│  Structure:                                              │
│  - hris/hris_employee_service.py                        │
│  - ats/ats_candidate_service.py                         │
│  - common/connector_service.py                          │
│  - common/link_service.py (Magic Link creation)         │
└──────────────────┬──────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────┐
│              REPOSITORY LAYER                            │
│              app/repositories/                           │
│  • Database CRUD operations                              │
│  • Query construction                                    │
│  • No business logic                                     │
│                                                          │
│  Files:                                                  │
│  - hris/hris_employee_repo.py                           │
│  - common/connector_repository.py                       │
│  - base_repository.py (Generic CRUD)                    │
└──────────────────┬──────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────┐
│                    DATABASE                              │
│                  PostgreSQL                              │
│  Tables: hris_employee, connector, org, link            │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│           EXTERNAL SERVICE LAYER                         │
│           app/services/external/                         │
│  • Integration-specific clients (READ-ONLY in backend)   │
│  • Passthrough service                                   │
│  • Used primarily for passthrough API calls              │
│                                                          │
│  Files:                                                  │
│  - passthrough/base_passthrough_service.py              │
│  - hris/bamboohr.py (minimal, mainly in workers)        │
└─────────────────────────────────────────────────────────┘
```

#### 3.1.2 Key Features

**Authentication & Authorization**
- API Key validation (`Authorization: Bearer prod_xxx`)
- Connector token validation (`X-Connector-Token: conn_xxx`)
- Dependency injection via `app/deps.py`

**API Endpoints**
- External APIs: `/api/hris/v1/*`, `/api/ats/v1/*`, `/api/lms/v1/*`
- Internal APIs: `/api/internal/v1/*` (admin operations)
- Embedded APIs: `/api/embedded/v1/*` (for link.unifyx)
- Common APIs: `/api/common/v1/passthrough` (raw API proxy)

**Data Models**
- Pydantic models in `app/models/`
- Request/Response schemas in `app/schemas/`
- Database models map directly to unified schema

---

### 3.2 workers (Background Sync System)

**Technology**: Dramatiq (Actor-based task queue), RabbitMQ  
**Responsibility**: Sync data from third-party systems to BindBee's database

#### 3.2.1 Architecture

```
┌──────────────────────────────────────────────────────────┐
│                    SCHEDULER                              │
│                  scheduler.py                             │
│                                                           │
│  Runs every 30 seconds (configurable)                     │
│                                                           │
│  1. Query pending jobs from connector_job table           │
│     SELECT * FROM connector_job                           │
│     WHERE next_sync_start_time <= NOW()                   │
│     AND status = 'PENDING'                                │
│                                                           │
│  2. For each job:                                         │
│     - Get task from TASK_MAP                              │
│     - Enqueue to RabbitMQ via Dramatiq                    │
│     - Mark status = 'SCHEDULER_PROCESSED'                 │
└───────────────────┬──────────────────────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────────────────────┐
│                   RABBITMQ                                │
│              Message Queue                                │
│  • Queue: default                                         │
│  • Persistence: enabled                                   │
└───────────────────┬──────────────────────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────────────────────┐
│                    WORKER                                 │
│                   main.py                                 │
│                                                           │
│  @dramatiq.actor                                          │
│  def start_bamboohr_sync(job):                            │
│      start_integration_sync(job)                          │
│                                                           │
│  Flow:                                                    │
│  1. Mark job as PICKED_BY_WORKER                          │
│  2. Fire pre-sync webhook                                 │
│  3. Get external service (bamboohr.py)                    │
│  4. Execute sync → fetch data from BambooHR               │
│  5. Unify & store data in database                        │
│  6. Mark job as DONE                                      │
│  7. Fire post-sync webhook                                │
└───────────────────┬──────────────────────────────────────┘
                    │
                    ▼
         ┌──────────────────────┐
         │  EXTERNAL SERVICES   │
         │  services/external/  │
         │                      │
         │  - bamboohr.py       │
         │  - workday.py        │
         │  - greenhouse.py     │
         │  - 50+ integrations  │
         │                      │
         │  Methods:            │
         │  • start_job()       │
         │  • _fetch_employees()│
         │  • _fetch_companies()│
         └──────────┬───────────┘
                    │
                    ▼
         ┌──────────────────────┐
         │  INTERNAL SERVICES   │
         │  services/internal/  │
         │                      │
         │  Unification Logic   │
         │  • Map TP fields →   │
         │    unified schema    │
         │  • Bulk upsert DB    │
         └──────────────────────┘
```

#### 3.2.2 Integration Service Patterns

**External Service** (`services/external/hris/bamboohr.py`)
- Makes HTTP requests to third-party APIs
- Handles OAuth token refresh
- No direct database writes
- Calls internal services for data storage

**Internal Service** (`services/internal/integrations/hris/bamboohr.py`)
- Field mapping configuration
- Data transformation (TP schema → Unified schema)
- Database upserts via repositories

**Example Flow**:
```python
# External: Fetch data from BambooHR
bamboohr_employees = bamboohr_api.get("/v1/reports/custom")

# Internal: Unify and store
HrisEmployeeService.unify_employees(
    data=bamboohr_employees,
    connector_id="conn_123",
    integration="bamboohr"
)
# → Transforms and bulk inserts into hris_employee table
```

---

### 3.3 app.unifyx (Customer Dashboard)

**Technology**: Next.js 14, React 18, TailwindCSS  
**Responsibility**: Customer-facing SaaS dashboard

#### 3.3.1 Features

**Organization Management**
- Create/manage organizations
- Team member invitations
- Production access control

**Integration Management**
- View all available integrations (HRIS/ATS/LMS)
- Create magic links for end-users
- Monitor connector status and sync history

**API Management**
- Generate API keys (test/production)
- API usage analytics
- Webhook configuration

**Monitoring & Analytics**
- Sync success/failure rates
- API request logs
- Data sync timestamps

#### 3.3.2 Architecture

```
app.unifyx/
├── app/                 # Next.js 14 App Router
│   ├── (auth)/         # Authentication pages
│   ├── (dashboard)/    # Protected dashboard pages
│   │   ├── connectors/
│   │   ├── integrations/
│   │   ├── api-keys/
│   │   └── settings/
│   └── api/            # API routes (Next.js backend)
│
├── components/          # React components
│   ├── ui/             # shadcn/ui components
│   ├── connectors/
│   └── integrations/
│
└── lib/                # Utilities
    ├── api-client.ts   # Axios wrapper for unifyx-backend
    ├── auth.ts
    └── utils.ts
```

**State Management**: Redux Toolkit  
**Styling**: TailwindCSS + shadcn/ui  
**HTTP Client**: Axios (calls `unifyx-backend` APIs)

---

### 3.4 link.unifyx (Embedded Auth Flow)

**Technology**: Next.js 13, React 18  
**Responsibility**: White-labeled connector setup interface

#### 3.4.1 User Flow

```
1. Customer App generates magic link via unifyx-backend
   POST /api/internal/v1/link/magic-link
   Response: { "magic_link": "https://link.unifyx.com/abc123..." }

2. End-user clicks magic link
   → Opens link.unifyx embedded in iframe

3. link.unifyx decodes token and initiates connector
   POST /api/embedded/v1/link/initiate
   Response: { connector_id, integration, config }

4. User authenticates with third-party
   - OAuth: Redirect to provider → callback
   - API Key: User enters credentials
   - SFTP: Credentials generated automatically

5. On success, connector status = COMPLETE
   Initial sync job created → workers start syncing

6. link.unifyx shows success message
   Customer app receives webhook notification
```

#### 3.4.2 Features

- **Multi-Auth Support**: OAuth 2.0, API Key, SFTP
- **White-labeled**: Customizable branding
- **Embedded**: Runs in iframe within customer's app
- **Mobile-responsive**: Works on all devices
- **CORS-enabled**: Allows cross-origin embedding

---

## 4. Data Flow Architecture

### 4.1 Complete Lifecycle: BambooHR Integration

```
┌─────────────────────────────────────────────────────────┐
│ PHASE 1: MAGIC LINK CREATION                            │
└─────────────────────────────────────────────────────────┘

Customer App
  │
  │ POST /api/internal/v1/link/magic-link
  │ { "integration_id": "bamboohr-uuid", "category": "HRIS" }
  ▼
unifyx-backend (link_service.py)
  │
  ├─ Create end_user record
  ├─ Generate random token (48 chars)
  ├─ INSERT INTO link (token, integration_id, org_id)
  └─ Return encoded magic link
  
Response: "https://link.unifyx.com/eyJ0b2tlbiI6ImFiYy4uLiJ9"

┌─────────────────────────────────────────────────────────┐
│ PHASE 2: CONNECTOR ESTABLISHMENT                        │
└─────────────────────────────────────────────────────────┘

End User clicks magic link
  │
  ▼
link.unifyx
  │
  │ 1. Decode base64 → extract token
  │ 2. POST /api/embedded/v1/link/initiate
  ▼
unifyx-backend
  │
  ├─ Validate token (active, not expired)
  ├─ CREATE connector (status=INITIATED)
  └─ Return connector details
  
  ▼
link.unifyx redirects to BambooHR OAuth
  │
  │ https://api.bamboohr.com/authorize?client_id=...
  ▼
BambooHR (User grants permission)
  │
  │ Callback: https://link.unifyx.com/callback?code=xyz
  ▼
link.unifyx
  │
  │ POST /api/embedded/v1/bamboohr/oauth-callback
  │ { "code": "xyz", "connector_id": "conn_123" }
  ▼
unifyx-backend
  │
  ├─ Exchange code for access_token
  ├─ Store in connector_oauth table
  ├─ UPDATE connector (status=COMPLETE)
  └─ CREATE connector_job (status=PENDING, initial_sync=true)

Result: Connector ready for sync!

┌─────────────────────────────────────────────────────────┐
│ PHASE 3: INITIAL DATA SYNC (Workers)                    │
└─────────────────────────────────────────────────────────┘

scheduler.py (runs every 30s)
  │
  │ SELECT FROM connector_job WHERE next_sync_start_time <= NOW()
  ▼
RabbitMQ
  │
  │ Queue: start_bamboohr_sync(job)
  ▼
worker (main.py)
  │
  ├─ Mark job PICKED_BY_WORKER
  ├─ Fire pre-sync webhook
  │
  ▼
BambooHRExternalService (bamboohr.py)
  │
  ├─ Load connector config & OAuth token
  ├─ Validate/refresh token if needed
  │
  ├─ Fetch Companies
  │   POST https://acme.bamboohr.com/api/v1/reports/custom
  │   Response: [{ "id": "1", "name": "Acme Corp" }]
  │   → HrisCompanyService.unify_companies(data)
  │       → Bulk INSERT into hris_company
  │
  ├─ Fetch Employees
  │   POST https://acme.bamboohr.com/api/v1/reports/custom
  │   Response: [{ "id": "123", "firstName": "John", ... }]
  │   → HrisEmployeeService.unify_employees(data)
  │       → Map fields: firstName → first_name
  │       → Resolve company_id from lookup dict
  │       → Bulk UPSERT into hris_employee
  │
  ├─ Fetch Time Offs, Employments, Benefits...
  │
  └─ Mark job DONE
     Fire post-sync webhook

Result: All BambooHR data now in BindBee database!

┌─────────────────────────────────────────────────────────┐
│ PHASE 4: CUSTOMER FETCHES UNIFIED DATA                  │
└─────────────────────────────────────────────────────────┘

Customer App
  │
  │ GET /api/hris/v1/employees?employment_status=ACTIVE
  │ Authorization: Bearer prod_api_key
  │ X-Connector-Token: conn_123
  ▼
unifyx-backend
  │
  │ Router: employee.py
  ├─ Validate API key & connector token
  ├─ Parse filters (employment_status=ACTIVE)
  │
  │ Service: hris_employee_service.py
  ├─ Build query filters
  │
  │ Repository: hris_employee_repo.py
  ├─ SELECT * FROM hris_employee
  │   WHERE connector_id = 'conn_123'
  │   AND employment_status = 'ACTIVE'
  │
  └─ Return paginated response

Response:
{
  "cursor": "base64...",
  "page_size": 50,
  "items": [
    {
      "id": "uuid-1",
      "remote_id": "123",  // BambooHR's ID
      "first_name": "John",
      "last_name": "Doe",
      "work_email": "john@acme.com",
      "employment_status": "ACTIVE",
      "company": { "id": "uuid-c1", "name": "Acme Corp" }
    }
  ]
}
```

### 4.2 Incremental Sync Flow

After initial sync, connector_job is updated:
```sql
UPDATE connector_job SET
    next_sync_start_time = NOW() + INTERVAL '24 hours',
    sync_frequency = 24,
    initial_sync = false
```

Scheduler picks it up 24 hours later and runs incremental sync (fetches only changed data).

---

## 5. Infrastructure & Deployment

### 5.1 Technology Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **Backend API** | FastAPI, Python 3.11, Uvicorn/Gunicorn | REST API server |
| **Frontend** | Next.js 13/14, React 18, TypeScript | Web applications |
| **Database** | PostgreSQL 14+ | Primary data store |
| **Cache** | Valkey/Redis | Session cache, rate limiting |
| **Queue** | RabbitMQ, Dramatiq | Background job processing |
| **Monitoring** | Sentry, Prometheus | Error tracking, metrics |
| **Deployment** | Docker, Docker Compose | Containerization |

### 5.2 Database Schema (Key Tables)

```sql
-- Organization & Authentication
org (id, name, production_access, created_at)
person (id, email, org_id, role)
api_key (id, key, org_id, type)

-- Integration Management
integration (id, name, category, auth_type, logo_url)
link (id, token, integration_id, org_id, expire_at)
end_user (id, email, org_name, org_id)

-- Connector & Jobs
connector (id, org_id, integration_id, status, config, access_token)
connector_oauth (connector_id, access_token, refresh_token, expires_at)
connector_job (id, connector_id, status, next_sync_start_time, sync_frequency)

-- Unified HRIS Data
hris_employee (id, connector_id, remote_id, first_name, last_name, ...)
hris_company (id, connector_id, remote_id, name, ...)
hris_time_off (id, connector_id, employee_id, ...)
hris_employment (id, connector_id, employee_id, ...)

-- Unified ATS Data
ats_candidate (id, connector_id, remote_id, first_name, ...)
ats_job (id, connector_id, remote_id, title, ...)
ats_application (id, connector_id, candidate_id, job_id, ...)

-- Unified LMS Data
lms_user (id, connector_id, remote_id, ...)
lms_course (id, connector_id, remote_id, ...)
```

### 5.3 Deployment Architecture

```
┌─────────────────────────────────────────────────────┐
│                  Load Balancer                       │
│                  (Production)                        │
└──────────┬─────────────────────┬────────────────────┘
           │                     │
           ▼                     ▼
┌──────────────────┐   ┌──────────────────┐
│  unifyx-backend  │   │  unifyx-backend  │
│   Instance 1     │   │   Instance 2     │
│   (Docker)       │   │   (Docker)       │
└──────────────────┘   └──────────────────┘
           │                     │
           └──────────┬──────────┘
                      ▼
           ┌──────────────────────┐
           │  PostgreSQL Primary  │
           │  + Read Replicas     │
           └──────────────────────┘

┌─────────────────────────────────────────────────────┐
│              Workers (Scheduler + N Workers)         │
│  • 1 Scheduler instance (cron-like)                 │
│  • Multiple Worker instances (horizontal scaling)   │
└──────────────────┬──────────────────────────────────┘
                   │
                   ▼
        ┌──────────────────┐
        │     RabbitMQ     │
        │   (Clustered)    │
        └──────────────────┘

┌─────────────────────────────────────────────────────┐
│              Frontend Apps                           │
│  • app.unifyx (Static export → CDN/Vercel)          │
│  • link.unifyx (Static export → CDN/Vercel)         │
└─────────────────────────────────────────────────────┘
```

---

## 6. Security Architecture

### 6.1 Authentication Mechanisms

**API Authentication** (`unifyx-backend`)
- API Keys: `Authorization: Bearer {prod|test}_xxx`
- Connector Tokens: `X-Connector-Token: conn_xxx`
- Validation in `app/deps.py`

**Dashboard Authentication** (`app.unifyx`)
- JWT-based session
- Email/password login
- Person → Org relationship

**OAuth Security**
- PKCE flow for OAuth 2.0
- State parameter validation
- Secure token storage (encrypted in DB)

### 6.2 Data Security

**Encryption**
- TLS/HTTPS for all communications
- Encrypted OAuth tokens in database
- Secure credential storage for SFTP

**Rate Limiting**
- Per-organization rate limits
- Configurable in `connector` table
- Middleware: `app/middleware/rate_limit.py`

**CORS Configuration**
- Whitelisted origins (production/staging)
- Open CORS for `/api/embedded/*` (link.unifyx)
- CSP headers

### 6.3 Compliance

- **Sensitive Headers Masking**: Authorization headers masked before logging to `api_request_detail` table
- **Webhook Security**: HMAC signature validation
- **Audit Logs**: All connector operations logged

---

## 7. Integration Patterns

### 7.1 Supported Integration Types

**OAuth 2.0** (Most common)
- Examples: BambooHR, Workday, Greenhouse
- Flow: Authorization code grant
- Token refresh: Automatic via workers

**API Key**
- Examples: Keka, BreatheHR
- Storage: Encrypted in connector config
- Validation: Per-integration logic

**SFTP**
- Examples: Paylocity SFTP, Paychex SFTP
- Credentials: Auto-generated by BindBee
- File parsing: CSV/XML parsers in workers

### 7.2 Integration Development Pattern

Each integration requires:

1. **External Service** (`workers/services/external/{category}/{integration}.py`)
   - HTTP client methods
   - API endpoint definitions
   - Authentication logic
   - Pagination handling

2. **Internal Mapping** (`workers/services/internal/integrations/{category}/{integration}.py`)
   - Field mapping configuration
   - Data transformation functions
   - Enum mappings

3. **Database Entry**
```sql
INSERT INTO integration (name, category, auth_type, base_api_url)
VALUES ('bamboohr', 'HRIS', 'OAUTH', 'https://{domain}.bamboohr.com/api');
```

---

## 8. Scalability & Performance

### 8.1 Horizontal Scaling

**API Server** (`unifyx-backend`)
- Stateless design
- Scale via load balancer
- Connection pooling (asyncpg)

**Workers**
- Multiple worker instances consume from RabbitMQ
- Auto-scaling based on queue depth
- Each worker handles one connector at a time

**Database**
- Read replicas for GET requests
- Write to primary
- Partitioning for large tables (future)

### 8.2 Performance Optimizations

**Caching**
- Valkey/Redis for frequently accessed data
- Connector configs cached
- Integration metadata cached

**Bulk Operations**
- `bulk_upsert()` for employee/candidate data
- Batch size: 1000 records
- Reduces DB round trips

**Pagination**
- Cursor-based pagination (not offset)
- More efficient for large datasets
- Base64-encoded cursor

### 8.3 Monitoring

**Application Monitoring**
- Sentry: Error tracking and performance
- Prometheus: Metrics collection
- Custom metrics: `/metrics` endpoint

**Job Monitoring**
- Connector job status tracking
- Sync success/failure rates
- Slack alerts on failures

**Database Monitoring**
- Query performance analysis
- Connection pool metrics
- Slow query logs

---

## Appendix: Key Files Reference

### unifyx-backend
```
app/
├── main.py                     # FastAPI app initialization
├── deps.py                     # Authentication dependencies
├── router/
│   ├── hris/v1/employee.py    # HRIS employee endpoints
│   └── internal/v1/link.py    # Magic link creation
├── services/
│   ├── internal/
│   │   ├── hris/hris_employee_service.py
│   │   └── common/link_service.py
│   └── external/
│       └── passthrough/base_passthrough_service.py
└── repositories/
    └── hris/hris_employee_repo.py
```

### workers
```
├── scheduler.py               # Job scheduler (APScheduler)
├── main.py                    # Dramatiq worker tasks
├── services/
│   ├── external/
│   │   └── hris/bamboohr.py  # BambooHR API client
│   └── internal/
│       ├── hris/hris_employee_service.py
│       └── integrations/hris/bamboohr.py  # Field mapping
└── repositories/
    └── hris/hris_employee_repo.py
```

### app.unifyx
```
app/
├── (dashboard)/
│   ├── connectors/             # Connector management UI
│   └── integrations/           # Integration catalog
└── api/                        # Next.js API routes
```

### link.unifyx
```
app/
├── [token]/                    # Dynamic route for magic link
└── callback/                   # OAuth callback handler
```

---

**End of Document**
