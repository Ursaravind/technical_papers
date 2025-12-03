# Bindbee Backend Workflow - Complete Documentation

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [Folder Structure](#folder-structure)
3. [Complete Request Flow](#complete-request-flow)
4. [Example: Fetching Employees from BambooHR](#example-fetching-employees-from-bamboohr)
5. [Layer-by-Layer Breakdown](#layer-by-layer-breakdown)
6. [Internal vs External Services](#internal-vs-external-services)
7. [Data Flow Diagrams](#data-flow-diagrams)

---

## Architecture Overview

Bindbee follows a **layered architecture** pattern with clear separation of concerns:

```
┌─────────────────────────────────────────────────────────────┐
│                        CLIENT                               │
│              (Customer's Application)                       │
└────────────────────────┬────────────────────────────────────┘
                         │ HTTP Request
                         │ POST /api/hris/v1/employees
                         │ Headers: x-connector-token, Authorization
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                    ROUTER LAYER                             │
│              (app/router/hris/v1/)                          │
│  • Routes HTTP requests                                     │
│  • Validates authentication                                 │
│  • Parses query parameters                                  │
└────────────────────────┬────────────────────────────────────┘
                         │ connector_id, filters, pagination
                         ▼
┌─────────────────────────────────────────────────────────────┐
│              INTERNAL SERVICE LAYER                         │
│        (app/services/internal/hris/)                        │
│  • Business logic                                           │
│  • Data validation                                          │
│  • Coordinates between layers                               │
└────────────────────────┬────────────────────────────────────┘
                         │ SQL queries via Repository
                         ▼
┌─────────────────────────────────────────────────────────────┐
│              REPOSITORY LAYER                               │
│          (app/repositories/hris/)                           │
│  • Database operations (CRUD)                               │
│  • Query construction                                       │
│  • Data access abstraction                                  │
└────────────────────────┬────────────────────────────────────┘
                         │ Fetches data from
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                DATABASE (PostgreSQL)                        │
│  • Stores unified data from all integrations                │
│  • Tables: hris_employee, hris_company, etc.                │
└─────────────────────────────────────────────────────────────┘

                    ┌────────────────┐
                    │ WORKERS/JOBS   │
                    │  (Background)  │
                    └────────┬───────┘
                             │ Syncs data periodically
                             ▼
┌─────────────────────────────────────────────────────────────┐
│            EXTERNAL SERVICE LAYER                           │
│      (app/services/external/hris/)                          │
│  • Integration-specific logic (BambooHR, Workday, etc.)     │
│  • Authentication (OAuth, API Key)                          │
│  • API request construction                                 │
│  • Response mapping/normalization                           │
└────────────────────────┬────────────────────────────────────┘
                         │ HTTP requests
                         ▼
┌─────────────────────────────────────────────────────────────┐
│              THIRD-PARTY APIs                               │
│    (BambooHR, Workday, ADP, Greenhouse, etc.)               │
└─────────────────────────────────────────────────────────────┘
```

---

## Folder Structure

### `/app` - Application Root

```
app/
├── router/                    # 🔵 Layer 1: HTTP Routes
│   ├── hris/v1/              # HRIS API endpoints
│   │   ├── employee.py       # GET /employees, POST /employees
│   │   ├── company.py        # Company endpoints
│   │   ├── time_off.py       # Time off endpoints
│   │   └── ...
│   ├── ats/v1/               # ATS API endpoints
│   ├── internal/v1/          # Internal admin endpoints
│   └── common/v1/            # Common endpoints (passthrough)
│
├── services/                  # 🟢 Layer 2: Business Logic
│   ├── internal/             # Internal services (business logic)
│   │   ├── hris/             # HRIS business services
│   │   │   ├── hris_employee_service.py
│   │   │   ├── hris_company_service.py
│   │   │   └── ...
│   │   ├── ats/              # ATS business services
│   │   ├── common/           # Common services (org, connector, etc.)
│   │   └── lms/              # LMS business services
│   │
│   └── external/             # External integration clients
│       ├── hris/             # HRIS integrations
│       │   ├── bamboohr.py   # BambooHR API client
│       │   ├── workday.py    # Workday API client
│       │   ├── adp.py        # ADP API client
│       │   └── ... (50+ integrations)
│       ├── ats/              # ATS integrations
│       │   ├── greenhouse.py
│       │   ├── lever.py
│       │   └── ...
│       ├── passthrough/      # Passthrough service
│       └── external_service_helper.py  # Integration locator
│
├── repositories/              # 🟡 Layer 3: Data Access
│   ├── hris/                 # HRIS repositories
│   │   ├── hris_employee_repo.py
│   │   ├── hris_company_repo.py
│   │   └── ...
│   ├── ats/                  # ATS repositories
│   ├── common/               # Common repositories
│   │   ├── connector_repository.py
│   │   ├── org_repository.py
│   │   └── ...
│   └── base_repository.py    # Base repo with common CRUD
│
├── models/                    # 🔷 Data Models (Pydantic)
│   ├── hris/
│   │   ├── hris_employee.py  # Employee model + response schema
│   │   ├── hris_company.py
│   │   └── ...
│   ├── ats/
│   ├── common/
│   └── ...
│
├── schemas/                   # 📋 Request/Response Schemas
│   ├── request/
│   │   ├── external/         # External API requests
│   │   │   ├── hris/
│   │   │   │   └── hris_employee.py  # Employee create/update schemas
│   │   │   └── ats/
│   │   └── internal/         # Internal API requests
│   │       ├── org.py
│   │       └── person.py
│   │
│   └── response/
│       ├── external/         # External API responses
│       └── internal/         # Internal API responses
│
├── utils/                     # 🔧 Utilities
│   ├── helper.py             # Common functions (JSON, auth, etc.)
│   ├── http_client.py        # HTTP client wrapper
│   ├── enum.py               # Enums (categories, statuses, etc.)
│   ├── constants.py          # Constants
│   └── write_configs/        # Write operation configs
│       ├── bamboohr_write_config.py
│       └── ...
│
├── deps.py                    # 🔒 Dependency injection (auth)
├── exceptions/                # ❌ Custom exceptions
├── config/                    # ⚙️ Configuration
├── drivers/                   # 🗄️ Database/Redis drivers
└── main.py                    # 🚀 FastAPI application entry
```

---

## Complete Request Flow

### High-Level Flow

```
1. CLIENT sends request → 
2. ROUTER validates & routes → 
3. INTERNAL SERVICE processes business logic → 
4. REPOSITORY fetches from database → 
5. Return unified data to client

(Background: WORKERS sync data from EXTERNAL SERVICES → DATABASE)
```

### Detailed Flow with Function Calls

```
┌──────────────────────────────────────────────────────────┐
│ 1. CLIENT REQUEST                                        │
└────────────┬─────────────────────────────────────────────┘
             │
             │ GET /api/hris/v1/employees?page_size=50
             │ Headers:
             │   Authorization: Bearer prod_abc123
             │   X-Connector-Token: conn_xyz789
             │
             ▼
┌──────────────────────────────────────────────────────────┐
│ 2. ROUTER LAYER                                          │
│ File: app/router/hris/v1/employee.py                     │
└────────────┬─────────────────────────────────────────────┘
             │
             │ @router.get("/employees")
             │ async def get_employees(
             │     connector_id = Depends(get_current_connector),
             │     pagination_data = Depends(pagination_params),
             │     ...
             │ )
             │
             ├─► get_current_connector(authorization, x_connector_token)
             │   │ File: app/deps.py
             │   │ Validates tokens, returns connector_id
             │   └─► Returns: "conn_xyz789"
             │
             ├─► pagination_params(page_size=50)
             │   │ File: app/deps.py
             │   └─► Returns: {"page_size": 50, "cursor": None}
             │
             ├─► FilterService.generate_filters(connector_id, ...)
             │   │ Constructs database filters
             │   └─► Returns: [Filter(...), Filter(...)]
             │
             ▼
┌──────────────────────────────────────────────────────────┐
│ 3. INTERNAL SERVICE LAYER                                │
│ File: app/services/internal/hris/hris_employee_service.py│
└────────────┬─────────────────────────────────────────────┘
             │
             │ hris_employee_svc.get_employees(
             │     connector_id="conn_xyz789",
             │     filters=[...],
             │     pagination_data={...},
             │     relation_data=[],
             │     include_raw_data=False,
             │     include_custom_fields=False
             │ )
             │
             ├─► Inherits from BaseHrisService
             │   │ File: app/services/internal/hris/base_hris_service.py
             │   │ Provides: get_entities_v2(), get_entity_by_id_v2()
             │   │
             │   └─► get_entities_v2(...)
             │       │ Generic method to fetch entities with relations
             │       │
             ▼       ▼
┌──────────────────────────────────────────────────────────┐
│ 4. REPOSITORY LAYER                                      │
│ File: app/repositories/hris/hris_employee_repo.py        │
└────────────┬─────────────────────────────────────────────┘
             │
             │ hris_employee_repo.get_entities(
             │     connector_id="conn_xyz789",
             │     filters=[Filter(...)],
             │     pagination_data={...},
             │     projections=["id", "first_name", "last_name", ...]
             │ )
             │
             ├─► Inherits from BaseRepository
             │   │ File: app/repositories/base_repository.py
             │   │ Provides: create(), update(), delete(), get_by_id()
             │   │
             │   └─► Constructs SQL query:
             │       │
             │       │ SELECT id, first_name, last_name, email, ...
             │       │ FROM hris_employee
             │       │ WHERE connector_id = $1
             │       │   AND employment_status = $2
             │       │ ORDER BY created_at DESC
             │       │ LIMIT 50
             │       │
             ▼       ▼
┌──────────────────────────────────────────────────────────┐
│ 5. DATABASE (PostgreSQL)                                 │
│ Table: hris_employee                                     │
└────────────┬─────────────────────────────────────────────┘
             │
             │ Returns rows:
             │ [
             │   {id: uuid, first_name: "John", last_name: "Doe", ...},
             │   {id: uuid, first_name: "Jane", last_name: "Smith", ...},
             │   ...
             │ ]
             │
             ▼
┌──────────────────────────────────────────────────────────┐
│ 6. DATA TRANSFORMATION & RESPONSE                        │
└────────────┬─────────────────────────────────────────────┘
             │
             │ Repository → Models (Pydantic validation)
             │ File: app/models/hris/hris_employee.py
             │
             ├─► HrisEmployeeResponse.model_validate(row)
             │   │ Validates & transforms each row
             │   └─► Returns: HrisEmployeeResponse objects
             │
             │ Service → Add relations, custom processing
             │
             ├─► separate_country_code(phone_number)
             │   │ Extracts: "+1 555-1234" → country_code="+1", number="555-1234"
             │
             │ Service → Pagination
             │
             ├─► helper.create_paginated_response(data, page_size)
             │   │ File: app/utils/helper.py
             │   └─► Returns: {
             │         "cursor": "base64_uuid",
             │         "page_size": 50,
             │         "items": [...]
             │       }
             │
             │ Router → Final response
             │
             └─► helper.handle_response(response)
                 │ Converts ResponseModel to JSON
                 └─► Returns HTTP 200 with JSON body
```

---

## Example: Fetching Employees from BambooHR

### Scenario
Customer wants to fetch employees from their BambooHR account via Bindbee's unified API.

### Step-by-Step Execution

#### **Step 1: Client Makes API Request**

```http
GET /api/hris/v1/employees?page_size=10&employment_status=ACTIVE
Host: api.bindbee.dev
Authorization: Bearer prod_abc123xyz
X-Connector-Token: conn_bamboohr_456
```

---

#### **Step 2: Router Receives Request**

**File**: `app/router/hris/v1/employee.py`

```python
@router.get("/employees", response_class=ORJSONResponse)
async def get_employees(
    request: Request,
    connector_id: str = Depends(get_current_connector),  # ← Dependency injection
    pagination_data: dict = Depends(pagination_params),   # ← Parse page_size
    employment_status: HRISEmploymentStatus = Query(None),
    ...
) -> HrisEmployeePaginatedResponse:
    
    # Build filters from query params
    query_filters = {
        "connector_id": connector_id,
        "employment_status": employment_status
    }
    filters = FilterService.generate_filters(**query_filters)
    
    # Call internal service
    response = await hris_employee_svc.get_employees(
        connector_id=connector_id,
        filters=filters,
        pagination_data=pagination_data,
        relation_data=[],
        include_raw_data=False,
        include_custom_fields=False
    )
    
    return handle_response(response)
```

**What Happens Here:**
1. **Authentication**: `get_current_connector()` validates token, returns `connector_id`
2. **Pagination**: `pagination_params()` parses `page_size=10`
3. **Filters**: Creates database filters from query parameters
4. **Delegates** to internal service layer

---

#### **Step 3: Internal Service Processes Request**

**File**: `app/services/internal/hris/hris_employee_service.py`

```python
class HrisEmployeeService(BaseHrisService):
    def __init__(self):
        super().__init__(
            model_slug=HrisModelSlug.EMPLOYEE,
            entity_class=HrisEmployeeResponse,
            table=PostgresTable.HRIS_EMPLOYEE,
            repo=hris_employee_repo,
        )
    
    async def get_employees(
        self,
        connector_id: UUID,
        filters: list[Filter],
        pagination_data: dict,
        relation_data: list[Relation],
        include_raw_data: bool,
        include_custom_fields: bool
    ) -> ResponseModel:
        # Call base class method for generic entity fetching
        result = await self.get_entities_v2(
            connector_id=connector_id,
            filters=filters,
            pagination_data=pagination_data,
            relation_data=relation_data,
            include_raw_data=include_raw_data,
            include_custom_fields=include_custom_fields
        )
        
        # Post-processing: Extract country code from phone numbers
        if not result.error and result.data:
            for employee in result.data["items"]:
                (
                    employee.country_code,
                    employee.mobile_phone_number,
                ) = self.separate_country_code(employee.mobile_phone_number)
        
        return result
```

**What Happens Here:**
1. **Inherits** from `BaseHrisService` which provides generic CRUD operations
2. **Calls** `get_entities_v2()` to fetch data from repository
3. **Post-processes** data (e.g., separating country codes)
4. **Returns** `ResponseModel` with data or error

---

#### **Step 4: Repository Fetches from Database**

**File**: `app/repositories/hris/hris_employee_repo.py`

```python
class HrisEmployeeRepository(BaseRepository):
    def __init__(self):
        super().__init__(PostgresTable.HRIS_EMPLOYEE, HrisEmployee)
```

**Base Repository Method** (`app/repositories/base_repository.py`):

```python
async def get_entities(
    self,
    connector_id: UUID,
    filters: list[Filter],
    pagination_data: dict,
    projections: list[str],
    relation_data: list[Relation] = []
) -> list[dict]:
    # Build WHERE clause
    where_conditions = ["connector_id = $1"]
    params = [connector_id]
    
    # Add filters
    for i, filter in enumerate(filters, start=2):
        where_conditions.append(f"{filter.column} = ${i}")
        params.append(filter.value)
    
    # Build SELECT columns
    cols = ", ".join(projections)
    
    # Construct query
    query = f"""
        SELECT {cols}
        FROM {self.table}
        WHERE {' AND '.join(where_conditions)}
        ORDER BY created_at DESC
        LIMIT {pagination_data['page_size']}
    """
    
    # Execute query
    result = await db_utils.fetch_query(query, params)
    return result
```

**SQL Query Executed:**
```sql
SELECT 
    id, first_name, last_name, email, mobile_phone_number,
    employment_status, job_title, manager_id, company_id,
    created_at, modified_at
FROM hris_employee
WHERE connector_id = 'conn_bamboohr_456'
  AND employment_status = 'ACTIVE'
ORDER BY created_at DESC
LIMIT 10;
```

**What Happens Here:**
1. **Builds SQL query** dynamically based on filters
2. **Executes query** against PostgreSQL database
3. **Returns** list of dictionaries (raw database rows)

---

#### **Step 5: Database Returns Data**

**PostgreSQL Result**:
```json
[
  {
    "id": "01931edf-04b6-7391-8a5c-93ac4b395316",
    "first_name": "John",
    "last_name": "Doe",
    "email": "john.doe@example.com",
    "mobile_phone_number": "+1 555-123-4567",
    "employment_status": "ACTIVE",
    "job_title": "Software Engineer",
    "manager_id": "01931edf-04c8-7649-a470-d85f6161bd1a",
    "company_id": "01931edf-0567-7234-8b3c-74ac5b396427",
    "remote_id": "123",  // BambooHR's employee ID
    "created_at": "2024-11-01T10:00:00Z",
    "modified_at": "2024-11-15T14:30:00Z"
  },
  // ... 9 more employees
]
```

---

#### **Step 6: Transform to Response Models**

**File**: `app/models/hris/hris_employee.py`

```python
class HrisEmployeeResponse(BaseModel):
    id: UUID
    first_name: str | None
    last_name: str | None
    email: str | None
    mobile_phone_number: str | None
    country_code: str | None
    employment_status: HRISEmploymentStatus | None
    job_title: str | None
    manager: HrisEmployeeResponse | None = None  # Relation
    company: HrisCompanyResponse | None = None   # Relation
    remote_id: str | None
    created_at: datetime | None
    modified_at: datetime | None
    # ... 40+ more fields
```

**Transformation**:
```python
employees = [HrisEmployeeResponse.model_validate(row) for row in db_rows]
```

---

#### **Step 7: Add Pagination & Return Response**

**Service Layer**:
```python
paginated_response = helper.create_paginated_response(
    data=employees,
    page_size=10
)
# Returns:
# {
#   "cursor": "base64_encoded_last_id",
#   "page_size": 10,
#   "items": [...]
# }
```

**Router Layer**:
```python
return handle_response(ResponseModel(data=paginated_response))
```

---

#### **Step 8: Client Receives Response**

```json
{
  "cursor": "MDFhY2MzZGItMDRiNi03MzkxLThhNWMtOTNhYzRiMzk1MzE2",
  "page_size": 10,
  "items": [
    {
      "id": "01931edf-04b6-7391-8a5c-93ac4b395316",
      "first_name": "John",
      "last_name": "Doe",
      "email": "john.doe@example.com",
      "mobile_phone_number": "555-123-4567",
      "country_code": "+1",
      "employment_status": "ACTIVE",
      "job_title": "Software Engineer",
      "remote_id": "123",
      "created_at": "2024-11-01T10:00:00Z",
      "modified_at": "2024-11-15T14:30:00Z"
    },
    // ... 9 more employees
  ]
}
```

---

## Layer-by-Layer Breakdown

### 1. Router Layer (`app/router/`)

**Purpose**: HTTP endpoint definitions, request validation, authentication

**Responsibilities**:
- Define API endpoints (`@router.get("/employees")`)
- Dependency injection (auth, pagination, filters)
- Parse query parameters
- Call internal services
- Return HTTP responses

**Key Files**:
- `hris/v1/employee.py` - Employee endpoints
- `hris/v1/company.py` - Company endpoints
- `ats/v1/candidate.py` - Candidate endpoints
- `common/v1/passthrough.py` - Passthrough endpoint

**Example**:
```python
@router.get("/employees/{id}")
async def get_employee_by_id(
    id: UUID,
    connector_id: str = Depends(get_current_connector),
    include_raw_data: bool = Query(False)
):
    response = await hris_employee_svc.get_employee_by_id(
        employee_id=id,
        connector_id=connector_id,
        include_raw_data=include_raw_data
    )
    return handle_response(response)
```

---

### 2. Internal Service Layer (`app/services/internal/`)

**Purpose**: Business logic, orchestration, validation

**Responsibilities**:
- Implement business rules
- Coordinate between repositories
- Data validation and transformation
- Handle complex operations (create, update)
- Return `ResponseModel` (success/error)

**Folder Structure**:
```
services/internal/
├── hris/
│   ├── base_hris_service.py           # Base class for all HRIS services
│   ├── hris_employee_service.py       # Employee business logic
│   ├── hris_company_service.py        # Company business logic
│   └── ...
├── ats/
│   ├── base_ats_service.py
│   ├── ats_candidate_service.py
│   └── ...
└── common/
    ├── org_service.py                 # Organization management
    ├── connector_service.py           # Connector management
    ├── link_service.py                # Magic Link creation
    └── ...
```

**Example**:
```python
class HrisEmployeeService(BaseHrisService):
    async def create_employee(self, connector_id: UUID, employee: HrisEmployeeCreateRequest):
        # 1. Get integration info
        integration_info = await connector_repo.get_integration_info_by_connector_id(connector_id)
        
        # 2. Resolve UUIDs to remote IDs
        employee_dict = await uuid_resolver.resolve(employee.model_dump(), connector_id)
        
        # 3. Build request body using write config
        request_body, headers = write_config.get_req_body_and_headers_for_create_employee(employee_dict)
        
        # 4. Call external integration service
        response, remote_id = await external_integration_svc.create_employee(
            connector_id=connector_id,
            config=config,
            req_body=request_body,
            headers=headers
        )
        
        # 5. Save to database
        employee_id = await self._save_employee(employee_dict, connector_id, remote_id)
        
        return WriteApiResponseModel(success=True, data=f"Employee created with id = {employee_id}")
```

---

### 3. Repository Layer (`app/repositories/`)

**Purpose**: Data access abstraction, database operations

**Responsibilities**:
- CRUD operations
- Query construction
- Database transactions
- No business logic

**Base Repository** (`base_repository.py`):
```python
class BaseRepository:
    def __init__(self, table: PostgresTable, model: Type[BaseModel]):
        self.table = table
        self.model = model
    
    async def create(self, model_dict: dict) -> UUID:
        # INSERT INTO table ...
    
    async def get_by_id(self, id: UUID, projections: list[str]) -> dict:
        # SELECT ... FROM table WHERE id = $1
    
    async def update(self, id: UUID, model_dict: dict) -> UUID:
        # UPDATE table SET ... WHERE id = $1
    
    async def delete(self, id: UUID) -> None:
        # DELETE FROM table WHERE id = $1
    
    async def get_entities(
        self, connector_id: UUID, filters: list[Filter], ...
    ) -> list[dict]:
        # SELECT ... FROM table WHERE connector_id = $1 AND ...
```

**Specific Repository**:
```python
class HrisEmployeeRepository(BaseRepository):
    def __init__(self):
        super().__init__(PostgresTable.HRIS_EMPLOYEE, HrisEmployee)
    
    # Custom methods specific to employees
    async def batch_get_by_connector_id(self, connector_id: UUID, projections: list[str]):
        query = f"SELECT {','.join(projections)} FROM {self.table} WHERE connector_id = $1"
        return await db_utils.fetch_query(query, [connector_id])
```

---

### 4. External Service Layer (`app/services/external/`)

**Purpose**: Integration-specific API clients

**Responsibilities**:
- Make HTTP requests to third-party APIs
- Handle authentication (OAuth, API Key, Basic Auth)
- Request/response mapping
- Rate limiting
- Error handling

**Folder Structure**:
```
services/external/
├── hris/
│   ├── bamboohr.py              # BambooHR API client
│   ├── workday.py               # Workday API client
│   ├── adp.py                   # ADP API client
│   └── ... (50+ integrations)
├── ats/
│   ├── greenhouse.py            # Greenhouse API client
│   ├── lever.py                 # Lever API client
│   └── ...
├── passthrough/                 # Passthrough service
│   ├── base_passthrough_service.py
│   ├── passthrough_request_processor.py
│   └── passthrough_response_processor.py
└── external_service_helper.py   # Service locator
```

**Example: BambooHR Service** (`bamboohr.py`):

```python
class BambooHrService:
    def __init__(self):
        self.client = http_client
    
    # OAuth methods
    async def get_access_token(self, config: dict) -> str:
        # POST to BambooHR OAuth endpoint
        pass
    
    async def get_new_access_token(self, config: dict) -> str:
        # Refresh OAuth token
        pass
    
    # Validation
    async def check_valid_configuration(
        self, connector_id: UUID, config: dict, oauth_info: dict
    ) -> ResponseModel:
        # Test API connection
        response = await self.client.make_request(
            endpoint=f"{base_url}/v1/employees/directory",
            headers=self.generate_auth_headers(config),
            method="GET"
        )
        return ResponseModel(success=response.success)
    
    # Authentication
    def generate_auth_headers(self, config: dict) -> dict:
        return helper.generate_headers(
            config=config,
            authorization_type=AuthorizationType.BEARER_TOKEN
        )
    
    # Write operations
    async def create_employee(
        self, connector_id: UUID, config: dict, req_body: dict, headers: dict
    ) -> tuple[HttpResponseModel, str]:
        # Build endpoint
        endpoint = self.generate_base_api_url(config["base_api_url"], config)
        endpoint = f"{endpoint}/v1/employees"
        
        # Make request
        response = await self.client.make_request(
            endpoint=endpoint,
            headers={**self.generate_auth_headers(config), **headers},
            method="POST",
            data=req_body,
            integration="bamboohr"
        )
        
        # Extract remote_id from response
        remote_id = response.response.headers.get("Location").split("/")[-1]
        
        return response, remote_id
    
    # URL generation
    def generate_base_api_url(self, url: str, config: dict) -> str:
        # BambooHR uses subdomain: https://api.bamboohr.com/api/gateway.php/{subdomain}
        return f"{url}/{config['domain']}"
```

---

### 5. Models Layer (`app/models/`)

**Purpose**: Data structure definitions (Pydantic models)

**Example**:
```python
class HrisEmployee(BaseModel):
    id: UUID
    connector_id: UUID
    remote_id: str | None
    first_name: str | None
    last_name: str | None
    email: str | None
    mobile_phone_number: str | None
    employment_status: HRISEmploymentStatus | None
    job_title: str | None
    manager_id: UUID | None
    company_id: UUID | None
    created_at: datetime
    modified_at: datetime
    raw_data: dict | None  # Original API response

class HrisEmployeeResponse(HrisEmployee):
    country_code: str | None
    manager: HrisEmployeeResponse | None  # Nested relation
    company: HrisCompanyResponse | None   # Nested relation
```

---

### 6. Schemas Layer (`app/schemas/`)

**Purpose**: Request/response validation schemas

```
schemas/
├── request/
│   ├── external/                      # Customer API requests
│   │   ├── hris/
│   │   │   └── hris_employee.py       # Employee create/update requests
│   │   └── ats/
│   └── internal/                      # Admin API requests
│       ├── org.py
│       └── person.py
└── response/
    ├── external/                      # Customer API responses
    │   ├── hris/
    │   │   └── hris_employee.py       # Paginated employee response
    │   └── base.py                    # Base response models
    └── internal/
        └── common.py                  # ResponseModel, WriteApiResponseModel
```

**Example Request Schema**:
```python
class HrisEmployeeCreateRequest(BaseModel):
    first_name: str
    last_name: str
    email: str
    job_title: str | None
    employment_status: HRISEmploymentStatus
    manager: UUID | None               # UUID of manager in Bindbee DB
    company: UUID | None               # UUID of company in Bindbee DB
    additional_attributes: dict | None  # Integration-specific fields
    custom_fields: dict | None         # Custom fields
```

**Example Response Schema**:
```python
class HrisEmployeePaginatedResponse(BaseModel):
    cursor: str | None
    page_size: int
    items: list[HrisEmployeeResponse]
```

---

## Internal vs External Services

### When to Use Internal Services

**Internal services** (`app/services/internal/`) are used for:

1. **Reading data from database**
   - `GET /employees` → Fetch from `hris_employee` table
   - `GET /companies` → Fetch from `hris_company` table

2. **Business logic**
   - Pagination
   - Filtering
   - Relations (e.g., include manager details)
   - Data transformation

3. **Orchestration**
   - Coordinate multiple repositories
   - Handle complex workflows

**Example Flow (Read)**:
```
Client → Router → Internal Service → Repository → Database → Response
```

---

### When to Use External Services

**External services** (`app/services/external/`) are used for:

1. **Writing data to third-party** (Create/Update/Delete)
   - `POST /employees` → Create in BambooHR API
   - `PUT /employees/{id}` → Update in Workday API

2. **Background sync jobs** (Workers)
   - Fetch latest employees from BambooHR
   - Sync to Bindbee database

3. **Authentication**
   - OAuth token refresh
   - API key validation

4. **Passthrough requests**
   - Custom API calls to third-party

**Example Flow (Write)**:
```
Client → Router → Internal Service → External Service → Third-Party API
                                   ↓
                                Repository → Database (save result)
```

**Example Flow (Background Sync)**:
```
Worker → External Service → Third-Party API → Parse Response → 
Repository → Database
```

---

## Data Flow Diagrams

### READ Operation (GET /employees)

```
┌─────────┐
│ Client  │ GET /api/hris/v1/employees
└────┬────┘
     │
     ▼
┌──────────────────────────────────────────┐
│ Router: employee.py                      │
│ • Validate auth (get_current_connector)  │
│ • Parse pagination                       │
│ • Build filters                          │
└────┬─────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────┐
│ Internal Service: hris_employee_service  │
│ • get_employees()                        │
│ • Calls get_entities_v2()                │
└────┬─────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────┐
│ Repository: hris_employee_repo           │
│ • get_entities()                         │
│ • Builds SQL query                       │
│ • Executes query                         │
└────┬─────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────┐
│ Database: PostgreSQL                     │
│ • hris_employee table                    │
│ • Returns rows                           │
└────┬─────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────┐
│ Transform to Models                      │
│ • HrisEmployeeResponse.model_validate()  │
│ • Post-processing (country code)         │
└────┬─────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────┐
│ Paginate & Return                        │
│ • create_paginated_response()            │
│ • handle_response()                      │
└────┬─────────────────────────────────────┘
     │
     ▼
┌─────────┐
│ Client  │ Receives JSON response
└─────────┘
```

---

### WRITE Operation (POST /employees)

```
┌─────────┐
│ Client  │ POST /api/hris/v1/employees
└────┬────┘  Body: { first_name: "John", ... }
     │
     ▼
┌──────────────────────────────────────────────┐
│ Router: employee.py                          │
│ • Validate auth                              │
│ • Validate request body schema              │
└────┬─────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────────┐
│ Internal Service: hris_employee_service      │
│ • create_employee()                          │
│ • Get integration info (BambooHR)            │
│ • Resolve UUIDs to remote IDs                │
│ • Get write config for BambooHR              │
└────┬─────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────────┐
│ Write Config: bamboohr_write_config.py       │
│ • Maps Bindbee fields → BambooHR fields      │
│ • Builds request body                        │
│ • Returns: {                                 │
│     "firstName": "John",                     │
│     "lastName": "Doe",                       │
│     ...                                      │
│   }                                          │
└────┬─────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────────┐
│ External Service: bamboohr.py                │
│ • create_employee()                          │
│ • Generate auth headers (Bearer token)       │
│ • Build endpoint URL                         │
│ • POST to BambooHR API                       │
└────┬─────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────────┐
│ Third-Party API: BambooHR                    │
│ • Creates employee                           │
│ • Returns: remote_id = "12345"               │
└────┬─────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────────┐
│ Save to Database                             │
│ • hris_employee_repo.create()                │
│ • Saves: {                                   │
│     connector_id, remote_id, first_name,     │
│     last_name, raw_data (original response)  │
│   }                                          │
└────┬─────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────────┐
│ Return Success Response                      │
│ • WriteApiResponseModel(                     │
│     success=True,                            │
│     data="Employee created with id = ..."    │
│   )                                          │
└────┬─────────────────────────────────────────┘
     │
     ▼
┌─────────┐
│ Client  │ Receives: { success: true, data: "..." }
└─────────┘
```

---

### BACKGROUND SYNC (Worker Job)

```
┌──────────────────────────────────────────────┐
│ Worker/Scheduler (Cron Job)                  │
│ • Runs every 15 minutes                      │
│ • Syncs data for all active connectors       │
└────┬─────────────────────────────────────────┘
     │
     │ For each connector:
     │
     ▼
┌──────────────────────────────────────────────┐
│ Get Connector Info                           │
│ • Integration: BambooHR                      │
│ • Config: { domain: "acme", api_key: "..." } │
└────┬─────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────────┐
│ External Service: bamboohr.py                │
│ • Fetch employees from BambooHR API          │
│ • GET /v1/employees/directory                │
└────┬─────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────────┐
│ Third-Party API: BambooHR                    │
│ • Returns: [                                 │
│     { id: "1", firstName: "John", ... },     │
│     { id: "2", firstName: "Jane", ... }      │
│   ]                                          │
└────┬─────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────────┐
│ Parse & Map Response                         │
│ • Map BambooHR fields → Bindbee fields       │
│ • Normalize data (e.g., employment_status)   │
│ • Extract custom fields                      │
└────┬─────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────────┐
│ Upsert to Database                           │
│ • For each employee:                         │
│   - Check if exists (by remote_id)           │
│   - If exists: UPDATE                        │
│   - If not exists: INSERT                    │
│ • hris_employee_repo.upsert()                │
└────┬─────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────────┐
│ Database: PostgreSQL                         │
│ • hris_employee table updated                │
│ • modified_at timestamp updated              │
└──────────────────────────────────────────────┘
```

---

## Summary: Complete Request Lifecycle

### 1. **Authentication**
- Client sends `Authorization: Bearer prod_abc123` and `X-Connector-Token: conn_xyz`
- `get_current_connector()` validates tokens
- Returns `connector_id`

### 2. **Routing**
- FastAPI router matches URL to endpoint
- Dependency injection extracts parameters
- Calls internal service

### 3. **Business Logic**
- Internal service applies business rules
- Builds filters, handles pagination
- Calls repository for data

### 4. **Data Access**
- Repository constructs SQL query
- Executes against PostgreSQL
- Returns raw database rows

### 5. **Transformation**
- Convert rows to Pydantic models
- Post-processing (e.g., phone number parsing)
- Add relations if requested

### 6. **Response**
- Paginate results
- Return JSON response to client

---

## Key Takeaways

1. **Layered Architecture**: Clear separation of concerns (Router → Service → Repository → Database)

2. **Internal vs External**:
   - **Internal services**: Business logic, read from database
   - **External services**: Integration clients, write to third-party APIs

3. **Background Sync**: Workers fetch data from third-party APIs and save to database

4. **Unified API**: All integrations (BambooHR, Workday, ADP) return the same unified schema

5. **Write Operations**: Go through external services → third-party API → save to database

6. **Read Operations**: Fetch directly from database (fast, cached data)

7. **Passthrough**: Allows custom API calls to third-party when unified API doesn't cover specific use cases

---

## Next Steps

To explore further:
- Check `app/services/external/hris/bamboohr.py` for integration implementation
- Review `app/utils/write_configs/` for field mapping examples
- See`workers/` folder for background sync job implementation
- Explore `app/repositories/base_repository.py` for generic CRUD operations
