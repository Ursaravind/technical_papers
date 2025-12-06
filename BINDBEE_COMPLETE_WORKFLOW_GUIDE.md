# BindBee Backend Complete Workflow Guide

**Last Updated**: December 5, 2025  
**Version**: 1.0

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Magic Link Creation Flow](#2-magic-link-creation-flow)
3. [Connector Establishment](#3-connector-establishment)
4. [Workers Job Picking](#4-workers-job-picking-by-connector)
5. [Third-Party API Integration](#5-third-party-hris-system-integration)
6. [Data Unification Process](#6-data-unification-process)
7. [Pulling Unified Data](#7-pulling-unified-data-back)
8. [BambooHR Integration Example](#8-bamboohr-integration-example)
9. [Internal vs External Services](#9-internal-vs-external-service-differentiation)

---

## 1. Architecture Overview

BindBee consists of two main components:

### A. unifyx-backend (Main API Server)
- **Purpose**: Serves unified API to customers
- **Stack**: FastAPI + PostgreSQL
- **Responsibilities**:
  - Magic link creation & connector management
  - Serving unified data via REST APIs
  - Authentication & authorization
  - Managing organizations, end-users, integrations

### B. workers (Background Sync System)
- **Purpose**: Syncs data from third-party systems
- **Stack**: Dramatiq + RabbitMQ + PostgreSQL
- **Responsibilities**:
  - Fetching data from HRIS/ATS systems
  - Transforming third-party data to unified schema
  - Storing unified data in PostgreSQL

### System Architecture Diagram

```
┌─────────────────────────────────────────────────────┐
│              CUSTOMER APPLICATION                   │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│              UNIFYX-BACKEND (FastAPI)               │
│  • Magic Link Creation                              │
│  • Connector Management                             │
│  • Unified API Endpoints                            │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│                 POSTGRESQL                          │
│  Tables: connector, connector_job, hris_employee    │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│                  RABBITMQ                           │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│                  WORKERS                            │
│  ┌──────────────┐      ┌──────────────┐            │
│  │  SCHEDULER   │─────▶│    WORKER    │            │
│  │ (scheduler.py)│      │  (main.py)   │            │
│  └──────────────┘      └──────┬───────┘            │
│                                │                     │
│         ┌──────────────────────▼──────────────┐    │
│         │   EXTERNAL SERVICES                 │    │
│         │   (bamboohr.py, workday.py)         │    │
│         │   • HTTP requests to third-party    │    │
│         │   • OAuth token refresh             │    │
│         └──────────────┬──────────────────────┘    │
│                        │                            │
│         ┌──────────────▼──────────────────────┐    │
│         │   INTERNAL SERVICES                 │    │
│         │   (hris_employee_service.py)        │    │
│         │   • Data unification                │    │
│         │   • Schema mapping                  │    │
│         │   • Database upserts                │    │
│         └─────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘
                     │
                     ▼
          ┌─────────────────────┐
          │  THIRD-PARTY APIS   │
          │  (BambooHR, etc.)   │
          └─────────────────────┘
```

---

## 2. Magic Link Creation Flow

**API Endpoint**: `POST /api/internal/v1/link/magic-link`

**File**: `unifyx-backend/app/services/internal/common/link_service.py`

### Request Example

```json
{
  "integration_id": "bamboohr-integration-uuid",
  "category": "HRIS",
  "end_user_data": {
    "org_name": "Acme Corp",
    "email": "admin@acme.com"
  }
}
```

### Process Flow

```python
# LinkService.create_magic_link()

# 1. Validate organization has production access
has_production_access = current_person.org.production_access

# 2. Validate integration exists
valid_integration_id = await integration_svc.validate_integration_id(integration_id)

# 3. Create or get end user
end_user_id = await end_user_svc.upsert_end_user(end_user_data)

# 4. Generate random token (48 characters)
token = generate_random_token(length=48)

# 5. Store link in database
await link_repo.create({
    "token": token,
    "category": "HRIS",
    "end_user_id": end_user_id,
    "integration_id": integration_id,
    "expire_at": NOW() + 7 days,
    "org_id": org_id
})

# 6. Encode magic link
magic_link_data = {
    "server": "https://api.bindbee.dev",
    "token": token,
    "is_test_connector": False
}
encoded = base64.urlsafe_b64encode(json.dumps(magic_link_data))

# 7. Return magic link
return {
    "link_token": token,
    "magic_link": f"https://link.bindbee.dev/{encoded}"
}
```

### Database State

```sql
-- link table
INSERT INTO link (id, token, category, integration_id, org_id, active, expire_at)
VALUES (
    'link-uuid',
    'abc123...',
    'HRIS',
    'bamboo-integration-uuid',
    'org-uuid',
    true,
    NOW() + INTERVAL '7 days'
);
```

---

## 3. Connector Establishment

**File**: `unifyx-backend/app/services/internal/common/connector_service.py`

### User Flow

```
1. End user clicks magic link
   ↓
2. Frontend decodes link and extracts token
   ↓
3. Frontend POSTs to /api/embedded/v1/link/initiate
   ↓
4. Backend validates token and creates connector
   ↓
5. User authenticates with third-party (OAuth)
   ↓
6. Tokens stored, initial sync job created
```

### Backend Process

```python
# link_service.py - initiate_magic_link()

async def initiate_magic_link(request_data: MagicLinkInitiate):
    # 1. Validate link token
    link = await get_link_by_token(token=request_data.link_token)
    # Check: active=true, expire_at > NOW()
    
    # 2. Create connector
    connector = await connector_svc.create_connector_from_link(link)
    
    # 3. For SFTP integrations, generate credentials
    if integration.authtype == AuthType.SFTP:
        connector_config = await ext_svc.initial_setup(connector_id)
        # Creates SFTP username/password
    
    # 4. Return configuration to frontend
    return {
        "connector_id": connector_id,
        "organization": org_data,
        "integration": integration_data,
        "config": connector_config
    }
```

### OAuth Flow (BambooHR Example)

```
1. Frontend redirects to BambooHR:
   https://api.bamboohr.com/authorize?
     client_id=...&
     redirect_uri=https://link.bindbee.dev/callback&
     response_type=code

2. User grants permission

3. BambooHR redirects back:
   https://link.bindbee.dev/callback?code=auth_code&state=connector_id

4. Backend exchanges code for tokens:
   POST https://api.bamboohr.com/token
   {
     "code": "auth_code",
     "client_id": "...",
     "client_secret": "...",
     "grant_type": "authorization_code"
   }

5. Store tokens:
   INSERT INTO connector_oauth (
       connector_id, access_token, refresh_token, expires_at
   )
```

### Connector Status Updates

```sql
-- Initial creation
INSERT INTO connector (id, org_id, integration_id, status)
VALUES ('conn-uuid', 'org-uuid', 'integration-uuid', 'INITIATED');

-- After OAuth completion
UPDATE connector 
SET status = 'COMPLETE', 
    config = '{"domain": "acme"}',
    modified_at = NOW()
WHERE id = 'conn-uuid';

-- Create initial sync job
INSERT INTO connector_job (
    connector_id, org_id, integration_name, category,
    initial_sync, status, next_sync_start_time, sync_frequency
) VALUES (
    'conn-uuid', 'org-uuid', 'bamboohr', 'HRIS',
    true, 'PENDING', NOW(), 24
);
```

---

## 4. Workers Job Picking by Connector

**Files**: 
- `workers/scheduler.py` - Picks jobs from database
- `workers/main.py` - Processes jobs with Dramatiq

### Scheduler Process

```python
# scheduler.py - Runs every 30 seconds (configurable)

def schedulerMainTask():
    # 1. Query pending jobs
    jobs = getJobsToSchedule()
    # SQL: SELECT * FROM connector_job 
    #      WHERE next_sync_start_time <= NOW() 
    #      AND status = 'PENDING'
    
    # 2. Enqueue each job to RabbitMQ
    for job in jobs:
        integration_name = job['i_name']  # 'bamboohr'
        category = job['category']         # 'HRIS'
        
        # Get Dramatiq task
        task = get_task(category, integration_name)
        # Returns: start_bamboohr_sync
        
        # Send to RabbitMQ
        task.send(job)
        
        # Mark as processed
        markJobAsSchedulerProcessed(job)
```

### Task Mapping

```python
# scheduler.py

HRIS_TASK_MAP = {
    IntegrationsName.BAMBOOHR: start_bamboohr_sync,
    IntegrationsName.WORKDAY: start_workday_sync,
    IntegrationsName.ADP: start_adp_sync,
    # ... 50+ integrations
}

ATS_TASK_MAP = {
    IntegrationsName.GREENHOUSE: start_greenhouse_sync,
    IntegrationsName.LEVER: start_lever_sync,
    # ...
}

def get_task(name, category):
    if category == 'HRIS':
        return HRIS_TASK_MAP[name]
    elif category == 'ATS':
        return ATS_TASK_MAP[name]
```

### Worker Process

```python
# main.py

@dramatiq.actor(queue_name="default", max_retries=0)
def start_bamboohr_sync(job: dict):
    start_integration_sync(job)

def start_integration_sync(job: dict):
    connector_id = job['connector_id']
    integration_name = job['integration_name']
    
    try:
        # 1. Update job status
        ConnectorService.mark_connector_job_scheduler_processed(job)
        # UPDATE connector_job SET status = 'PICKED_BY_WORKER'
        
        # 2. Fire pre-sync webhook
        webhook = WebhookService(org_id, connector_id)
        webhook.fire_pre_sync_webhooks(job)
        
        # 3. Get integration service
        external_class = ExternalServiceHelper.fetch_integration_service_by_name(
            name=integration_name, category=category
        )
        
        # 4. Execute sync
        external_class(org_id, connector_id).start_job(initial_sync=True)
        
        # 5. Mark as done
        ConnectorService.mark_connector_job_done(job, job_start_time)
        
        # 6. Fire post-sync webhook
        webhook.fire_post_sync_webhooks(job)
        
    except Exception as e:
        # Error handling
        ConnectorService.mark_connector_job_failed(job)
        webhook.fire_sync_error_webhook(job)
        SlackAlertService.send_error_alert(...)
```

---

## 5. Third-Party HRIS System Integration

**File**: `workers/services/external/hris/bamboohr.py`

### External Service Responsibilities

- Make HTTP requests to third-party APIs
- Handle authentication (OAuth, API Key)
- Manage token refresh
- Pagination
- Call internal services to store data
- **NO direct database writes**

### BambooHR Sync Flow

```python
class BambooHRExternalService(BaseHrisIntegrationService):
    
    def start_job(self, initial_sync: bool = True):
        # 1. Setup
        self.prepare_initial_setup()
        
        # 2. Fetch data
        self.start_fetching_data(initial_sync)
    
    def prepare_initial_setup(self):
        # Load configuration
        self._load_configuration()
        # SELECT config, access_token FROM connector
        
        # Setup headers
        self._setup_headers()
        # headers = {"Authorization": "Bearer access_token"}
        
        # Validate token
        if not self._valid_configuration():
            self._update_configuration()  # Refresh token
    
    def start_fetching_data(self, initial_sync: bool):
        if initial_sync:
            self._run_initial_sync()
        else:
            self._run_incremental_sync()
    
    def _run_initial_sync(self):
        # Fetch in dependency order
        self.supplemental_data['company'] = self._fetch_companies()
        self.supplemental_data['group'] = self._fetch_groups()
        self.supplemental_data['employee'] = self._fetch_employees()
        
        # Fetch related entities
        self._fetch_employments()
        self._fetch_compensations()
        self._fetch_time_offs()
        self._fetch_benefits()
        self._fetch_managers()
```

### Fetching Employees Example

```python
def _fetch_employees(self):
    # 1. Make HTTP request to BambooHR
    endpoint = f"{self.base_domain}/v1/reports/custom"
    result = self._make_external_request(
        endpoint=endpoint,
        method=HTTPMethod.POST,
        params={"format": "JSON", "onlyCurrent": "False"},
        data=bamboohr_config.EMPLOYEE_FIELDS_PAYLOAD
    )
    
    if result.success:
        # 2. Parse response
        tp_employees = json.loads(result.response.text)['employees']
        
        # 3. Call INTERNAL service to unify and store
        HrisEmployeeService.unify_employees(
            data=tp_employees,
            connector_id=self.connector_id,
            integration=IntegrationsName.BAMBOOHR,
            companies_dict=self.supplemental_data['company'],
            group_dict=self.supplemental_data['group']
        )
    
    return HrisEmployeeService.get_employees_remote_id_dict(connector_id)
```

### HTTP Request Example

```http
POST https://acme.bamboohr.com/api/v1/reports/custom?format=JSON
Authorization: Bearer access_token_xyz
Content-Type: application/json

{
  "fields": [
    "id", "employeeNumber", "firstName", "lastName",
    "workEmail", "jobTitle", "department", "hireDate"
  ]
}
```

**Response from BambooHR**:
```json
{
  "employees": [
    {
      "id": "123",
      "employeeNumber": "EMP001",
      "firstName": "John",
      "lastName": "Doe",
      "workEmail": "john.doe@acme.com",
      "jobTitle": "Software Engineer",
      "department": "Engineering",
      "hireDate": "2023-01-15",
      "status": "Active"
    }
  ]
}
```

---

## 6. Data Unification Process

**Files**:
- `workers/services/internal/hris/hris_employee_service.py` (Orchestration)
- `workers/services/internal/integrations/hris/bamboohr.py` (Mapping)

### Unification Flow

```
Third-Party Data → Mapping Service → Unified Schema → Database
   (BambooHR)        (bamboohr.py)     (PostgreSQL)
```

### Domain Service (Orchestration)

```python
# hris_employee_service.py

class HrisEmployeeService:
    @classmethod
    def unify_employees(cls, data: list, connector_id: str, 
                       integration: str, **kwargs):
        # 1. Get integration-specific mapping service
        mapping_service = IntegrationServiceHelper.fetch_integration_service_by_name(
            name=integration,  # 'bamboohr'
            category='HRIS'
        )
        # Returns: BambooHRInternalService
        
        # 2. Transform data
        unified_employees = mapping_service.unify_employees(
            data=data,
            connector_id=connector_id,
            **kwargs
        )
        
        # 3. Batch upsert
        if unified_employees:
            cls.bulk_upsert(employees=unified_employees)
```

### Integration Mapping Service

```python
# workers/services/internal/integrations/hris/bamboohr.py

class BambooHRInternalService(BaseInternalHrisService):
    
    # Field mapping configuration
    EMPLOYEE_MAPPING = {
        "remote_id": "id",
        HRISEmployee.EMPLOYEE_NUMBER: "employeeNumber",
        HRISEmployee.FIRST_NAME: "firstName",
        HRISEmployee.LAST_NAME: "lastName",
        HRISEmployee.WORK_EMAIL: "workEmail",
        HRISEmployee.DESIGNATION: "jobTitle",
        HRISEmployee.DEPARTMENT: "department",
        
        # Date transformation
        HRISEmployee.START_DATE: lambda x: _get_iso_date_string(
            x.get("hireDate")
        ),
        
        # Lookup mapping
        HRISEmployee.COMPANY: ("company", "companies_dict"),
        
        # Enum mapping
        HRISEmployee.EMPLOYMENT_STATUS: lambda x: _get_unified_employment_status(
            x.get("status")
        ),
        
        # Complex mapping
        HRISEmployee.GROUPS: (
            lambda x: [
                generate_group_id(x.get("department")),
                generate_group_id(x.get("division"))
            ],
            "group_dict",
            lambda x: [item for item in x if item]
        ),
        
        # Store raw data
        "raw_data": lambda x: json.dumps(x)
    }
    
    # Enum mappings
    EMPLOYMENT_STATUS = {
        "Active": HRISEmploymentStatus.ACTIVE,
        "Inactive": HRISEmploymentStatus.INACTIVE
    }
    
    GENDER_MAP = {
        "Male": HRISGender.MALE,
        "Female": HRISGender.FEMALE
    }
```

### Transformation Example

**Input** (BambooHR data):
```python
tp_employee = {
    "id": "123",
    "employeeNumber": "EMP001",
    "firstName": "John",
    "lastName": "Doe",
    "workEmail": "john.doe@acme.com",
    "jobTitle": "Software Engineer",
    "department": "Engineering",
    "hireDate": "2023-01-15",
    "status": "Active",
    "company": "Acme Corp"
}

companies_dict = {"Acme Corp": "company-uuid-123"}
```

**Output** (Unified schema):
```python
unified_employee = {
    "connector_id": "conn-uuid",
    "remote_id": "123",
    "employee_number": "EMP001",
    "first_name": "John",
    "last_name": "Doe",
    "work_email": "john.doe@acme.com",
    "designation": "Software Engineer",
    "department": "Engineering",
    "start_date": "2023-01-15T00:00:00Z",
    "employment_status": "ACTIVE",  # Mapped from "Active"
    "company_id": "company-uuid-123",  # Looked up from dict
    "raw_data": "{\"id\": \"123\", \"firstName\": \"John\", ...}"
}
```

### Database Upsert

```sql
INSERT INTO hris_employee (
    connector_id, remote_id, employee_number, first_name, 
    last_name, work_email, designation, department, 
    start_date, employment_status, company_id, raw_data
) VALUES (
    'conn-uuid', '123', 'EMP001', 'John',
    'Doe', 'john.doe@acme.com', 'Software Engineer', 'Engineering',
    '2023-01-15T00:00:00Z', 'ACTIVE', 'company-uuid-123', '{...}'
)
ON CONFLICT (connector_id, remote_id) 
DO UPDATE SET
    first_name = EXCLUDED.first_name,
    last_name = EXCLUDED.last_name,
    work_email = EXCLUDED.work_email,
    designation = EXCLUDED.designation,
    employment_status = EXCLUDED.employment_status,
    modified_at = NOW();
```

---

## 7. Pulling Unified Data Back

**File**: `unifyx-backend/app/router/hris/v1/employee.py`

### Customer API Request

```http
GET /api/hris/v1/employees?employment_status=ACTIVE&page_size=50
Host: api.bindbee.dev
Authorization: Bearer prod_api_key_xyz
X-Connector-Token: conn-uuid
```

### Backend Flow

```
Router → Internal Service → Repository → Database → Response
```

### 1. Router Layer

```python
# employee.py

@router.get("/employees")
async def get_employees(
    connector_id: str = Depends(get_current_connector),
    pagination_data: dict = Depends(pagination_params),
    employment_status: HRISEmploymentStatus = Query(None)
):
    # Build filters
    filters = FilterService.generate_filters(
        connector_id=connector_id,
        employment_status=employment_status
    )
    
    # Call service
    response = await hris_employee_svc.get_employees(
        connector_id=connector_id,
        filters=filters,
        pagination_data=pagination_data
    )
    
    return handle_response(response)
```

### 2. Service Layer

```python
# hris_employee_service.py

class HrisEmployeeService(BaseHrisService):
    async def get_employees(self, connector_id, filters, pagination_data):
        # Fetch data from repository
        result = await self.get_entities_v2(
            connector_id=connector_id,
            filters=filters,
            pagination_data=pagination_data
        )
        
        # Post-processing
        if result.data:
            for employee in result.data['items']:
                # Extract country code from phone
                employee.country_code, employee.mobile_phone_number = \
                    separate_country_code(employee.mobile_phone_number)
        
        return result
```

### 3. Repository Layer

```python
# hris_employee_repo.py

class HrisEmployeeRepository(BaseRepository):
    async def get_entities(self, connector_id, filters, pagination_data):
        # Build WHERE clause
        where_conditions = ["connector_id = $1"]
        params = [connector_id]
        
        for i, filter in enumerate(filters, start=2):
            where_conditions.append(f"{filter.column} = ${i}")
            params.append(filter.value)
        
        # Build query
        query = f"""
            SELECT id, first_name, last_name, work_email,
                   employment_status, designation, department
            FROM hris_employee
            WHERE {' AND '.join(where_conditions)}
            ORDER BY created_at DESC
            LIMIT {pagination_data['page_size']}
        """
        
        # Execute
        result = await db_utils.fetch_query(query, params)
        return result
```

### 4. Response

```json
{
  "cursor": "base64_encoded_cursor",
  "page_size": 50,
  "items": [
    {
      "id": "uuid-001",
      "remote_id": "123",
      "first_name": "John",
      "last_name": "Doe",
      "work_email": "john.doe@acme.com",
      "employment_status": "ACTIVE",
      "designation": "Software Engineer",
      "department": "Engineering"
    }
  ]
}
```

---

## 8. BambooHR Integration Example

### Complete Sync Timeline

```
T+0s:   Connector created (status=COMPLETE)
T+0s:   Initial sync job created (status=PENDING)
T+30s:  Scheduler picks job → RabbitMQ
T+31s:  Worker processes job
T+32s:  Fetch companies from BambooHR
T+35s:  Fetch groups from BambooHR
T+40s:  Fetch employees from BambooHR (500 employees)
T+45s:  Unify & store 500 employees
T+50s:  Fetch employments
T+55s:  Fetch compensations
T+60s:  Fetch time-offs
T+65s:  Fetch benefits
T+70s:  Job marked as DONE
T+71s:  Post-sync webhook fired
```

### Detailed Example

**1. Fetch Companies**
```python
result = self._make_external_request(
    endpoint=f"{self.base_domain}/v1/company"
)
# Response: {"displayName": "Acme Corp", "legalName": "Acme Corporation"}

HrisCompanyService.unify_companies(data=[company_response])
# Transforms and inserts into hris_company table
```

**2. Fetch Employees**
```python
result = self._make_external_request(
    endpoint=f"{self.base_domain}/v1/reports/custom",
    data={"fields": ["id", "firstName", "lastName", ...]}
)
# Response: {"employees": [{...}, {...}]}

HrisEmployeeService.unify_employees(
    data=tp_employees,
    companies_dict={"Acme Corp": "company-uuid"}
)
```

**3. Database State**
```sql
-- hris_employee table
SELECT id, first_name, last_name, company_id 
FROM hris_employee 
WHERE connector_id = 'conn-uuid';

-- Results:
-- uuid-001 | John | Doe   | company-uuid
-- uuid-002 | Jane | Smith | company-uuid
```

---

## 9. Internal vs External Service Differentiation

### External Services (`workers/services/external/`)

**Purpose**: Integration-specific API clients

**Location**: `workers/services/external/hris/bamboohr.py`

**Responsibilities**:
- ✅ Make HTTP requests to third-party APIs
- ✅ Handle authentication (OAuth, API keys)
- ✅ Token refresh on 401 errors
- ✅ Pagination logic
- ✅ Call internal services to store data
- ❌ **NO direct database writes**
- ❌ **NO data transformation**

**Example**:
```python
class BambooHRExternalService(BaseHrisIntegrationService):
    
    def _fetch_employees(self):
        # 1. Make HTTP request
        result = self._make_external_request(
            endpoint=f"{self.base_domain}/v1/reports/custom"
        )
        
        # 2. Call INTERNAL service
        HrisEmployeeService.unify_employees(
            data=tp_employees,
            connector_id=self.connector_id
        )
    
    def _update_configuration(self):
        # Refresh OAuth token
        result = self._get_new_access_token()
        
        # Store via internal service
        ConnectorService.update_connector_oauth_data(...)
```

### Internal Services (`workers/services/internal/`)

**Purpose**: Business logic & data orchestration

**Two Types**:

#### A. Domain Services
**Location**: `workers/services/internal/hris/hris_employee_service.py`

**Responsibilities**:
- ✅ Orchestrate data operations
- ✅ Call mapping services
- ✅ Database operations via repositories
- ❌ **NO HTTP requests to third-party APIs**

**Example**:
```python
class HrisEmployeeService:
    @classmethod
    def unify_employees(cls, data, integration, **kwargs):
        # 1. Get mapping service
        mapping_svc = fetch_integration_service(integration)
        
        # 2. Transform data
        unified = mapping_svc.unify_employees(data, **kwargs)
        
        # 3. Store in database
        cls.bulk_upsert(unified)
```

#### B. Integration Mapping Services
**Location**: `workers/services/internal/integrations/hris/bamboohr.py`

**Responsibilities**:
- ✅ Define field mappings
- ✅ Transform third-party data to unified schema
- ❌ **NO HTTP requests**
- ❌ **NO database operations**

**Example**:
```python
class BambooHRInternalService(BaseInternalHrisService):
    
    EMPLOYEE_MAPPING = {
        "remote_id": "id",
        HRISEmployee.FIRST_NAME: "firstName",
        HRISEmployee.EMPLOYMENT_STATUS: lambda x: map_status(x)
    }
    
    @classmethod
    def unify_employees(cls, data, **kwargs):
        return [cls.apply_mapping(emp) for emp in data]
```

### Folder Structure

```
workers/services/
│
├── external/                    # Third-party API clients
│   ├── hris/
│   │   ├── bamboohr.py         # BambooHR HTTP client
│   │   ├── workday.py          # Workday HTTP client
│   │   └── adp.py              # ADP HTTP client
│   └── base_integration.py
│
├── internal/                    # Business logic
│   ├── hris/                   # Domain services
│   │   ├── hris_employee_service.py
│   │   └── hris_company_service.py
│   │
│   └── integrations/           # Mapping services
│       └── hris/
│           ├── bamboohr.py    # BambooHR mappings
│           ├── workday.py     # Workday mappings
│           └── adp.py         # ADP mappings
```

---

## Complete Flow Summary

```
┌──────────────────────────────────────────────────────┐
│  1. Magic Link Creation (unifyx-backend)            │
└──────────────────────────────────────────────────────┘
POST /api/internal/v1/link/magic-link
  ↓
LinkService.create_magic_link()
  ↓
INSERT INTO link (token, integration_id)
  ↓
Return magic_link URL

┌──────────────────────────────────────────────────────┐
│  2. Connector Establishment (unifyx-backend)         │
└──────────────────────────────────────────────────────┘
User clicks magic link
  ↓
POST /api/embedded/v1/link/initiate
  ↓
INSERT INTO connector (status='INITIATED')
  ↓
User completes OAuth
  ↓
INSERT INTO connector_oauth (access_token)
  ↓
UPDATE connector SET status='COMPLETE'
  ↓
INSERT INTO connector_job (status='PENDING')

┌──────────────────────────────────────────────────────┐
│  3. Scheduler Picks Job (workers/scheduler.py)      │
└──────────────────────────────────────────────────────┘
SELECT * FROM connector_job WHERE status='PENDING'
  ↓
start_bamboohr_sync.send(job) → RabbitMQ
  ↓
UPDATE connector_job SET status='SCHEDULER_PROCESSED'

┌──────────────────────────────────────────────────────┐
│  4. Worker Processes Job (workers/main.py)          │
└──────────────────────────────────────────────────────┘
@dramatiq.actor start_bamboohr_sync(job)
  ↓
BambooHRExternalService(org_id, connector_id).start_job()

┌──────────────────────────────────────────────────────┐
│  5. External Service Fetches Data                   │
└──────────────────────────────────────────────────────┘
POST https://acme.bamboohr.com/api/v1/employees
  ↓
Response: [{"id": "123", "firstName": "John"}, ...]
  ↓
HrisEmployeeService.unify_employees(tp_data)

┌──────────────────────────────────────────────────────┐
│  6. Internal Service Unifies Data                   │
└──────────────────────────────────────────────────────┘
BambooHRInternalService.unify_employees(tp_data)
  ↓
Apply EMPLOYEE_MAPPING transformations
  ↓
INSERT INTO hris_employee ON CONFLICT UPDATE

┌──────────────────────────────────────────────────────┐
│  7. Job Completion                                   │
└──────────────────────────────────────────────────────┘
UPDATE connector_job SET status='DONE'
  ↓
Fire post-sync webhook

┌──────────────────────────────────────────────────────┐
│  8. Customer Pulls Data (unifyx-backend)            │
└──────────────────────────────────────────────────────┘
GET /api/hris/v1/employees
  ↓
SELECT * FROM hris_employee WHERE connector_id='...'
  ↓
Return unified data as JSON
```

---

## Key Takeaways

1. **Separation of Concerns**:
   - `unifyx-backend`: Customer-facing API
   - `workers`: Background data synchronization

2. **External vs Internal Services**:
   - **External**: Talks to third-party APIs, NO database access
   - **Internal**: Business logic, data transformation, database operations

3. **Data Flow**:
   - Third-party API → External Service → Internal Service → Database → Customer API

4. **Connector Lifecycle**:
   - Magic Link → Connector → OAuth → Job → Sync → Data Available

5. **Job Processing**:
   - Scheduler → RabbitMQ → Worker → External Service → Internal Service → Database

---

**End of Documentation**
