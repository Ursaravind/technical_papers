# Workers Architecture - Complete Documentation

## Executive Summary

The Workers subsystem is a **background job processing system** that handles asynchronous data synchronization from third-party integrations (HRIS, ATS, LMS) to Bindbee's unified database. It uses **Dramatiq** (Python task queue) with **RabbitMQ** as the message broker, **APScheduler** for scheduling, and follows an internal/external service pattern where internal services orchestrate business logic while external adapters communicate with third-party APIs.

**Key Responsibilities:**
- Scheduled data synchronization (every 15 minutes to 24 hours)
- OAuth token refresh and credential management
- Retry handling with exponential backoff
- Webhook notifications (pre-sync, post-sync, error)
- SFTP file processing with PGP decryption
- Request/response logging and observability

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         SCHEDULER                               │
│              (APScheduler - runs every N seconds)               │
│  - Queries database for pending jobs                            │
│  - Enqueues jobs to RabbitMQ                                    │
│  - Updates job status: SCHEDULER_PROCESSED                      │
└────────────────────────┬────────────────────────────────────────┘
                         │ Publishes job to queue
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                       RABBITMQ                                  │
│              (Message Broker - AMQP)                            │
│  - Stores job payloads                                          │
│  - Provides durability and retry semantics                      │
└────────────────────────┬────────────────────────────────────────┘
                         │ Worker polls for jobs
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                        WORKER                                   │
│              (Dramatiq - 2 processes)                           │
│  - Consumes jobs from queue                                     │
│  - Executes sync logic via start_integration_sync()             │
│  - Updates job status: PICKED_BY_WORKER → DONE/FAILED          │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ├──► External Service (BambooHR, Workday, etc.)
                         │    - Fetches data from third-party API
                         │    - OAuth refresh if needed
                         │
                         ├──► Internal Service (Unify & Store)
                         │    - Maps third-party data to unified schema
                         │    - Upserts to PostgreSQL
                         │
                         └──► Webhooks & Alerts
                              - Fire pre/post sync webhooks
                              - Send Slack alerts on errors
```

---

## Sequence Diagram

### ASCII Diagram

```
Scheduler          RabbitMQ        Worker          External API    Database        Webhooks
    |                  |              |                  |             |               |
    |--Query jobs----->|              |                  |             |               |
    |<--Pending jobs---|              |                  |             |               |
    |                  |              |                  |             |               |
    |--Enqueue job---->|              |                  |             |               |
    |--Update status-->|------------->|                  |             |               |
    |                  |              |                  |             |               |
    |                  |              |--Mark PICKED---->|------------>|               |
    |                  |              |                  |             |               |
    |                  |              |--Fire pre-sync webhook-------->|-------------->|
    |                  |              |                  |             |               |
    |                  |              |--Fetch config--->|------------>|               |
    |                  |              |<--Connector info-|<------------|               |
    |                  |              |                  |             |               |
    |                  |              |--GET /employees->|             |               |
    |                  |              |                  |--HTTP req-->|               |
    |                  |              |                  |<--JSON resp-|               |
    |                  |              |<--Employee data--|             |               |
    |                  |              |                  |             |               |
    |                  |              |--Unify & Upsert->|------------>|               |
    |                  |              |                  |             |               |
    |                  |              |--Mark DONE------>|------------>|               |
    |                  |              |                  |             |               |
    |                  |              |--Fire post-sync webhook------->|-------------->|
    |                  |              |                  |             |               |
(ERROR PATH)
    |                  |              |--X Error X------>|------------>|               |
    |                  |              |--Mark FAILED---->|------------>|               |
    |                  |              |--Fire error webhook----------->|-------------->|
    |                  |              |--Send Slack alert------------->|-------------->|
```

---

## Folder Structure

```
workers/
├── main.py                          # 🚀 Worker entrypoint & job handler
├── scheduler.py                     # ⏰ Scheduler that enqueues jobs
├── docker-compose.yaml              # 🐳 RabbitMQ + Scheduler + Worker
├── requirements.txt                 # 📦 Dependencies (Dramatiq, RabbitMQ, etc.)
│
├── drivers/
│   └── db.py                        # 🗄️ PostgreSQL connection pool
│
├── middlewares/
│   └── db_connection_pool.py        # 🔌 Dramatiq middleware for DB lifecycle
│
├── services/
│   ├── external/                    # 🌐 Integration adapters (talks to APIs)
│   │   ├── base_integration.py      # Base class for all integrations
│   │   ├── hris/
│   │   │   ├── bamboohr.py          # BambooHR sync logic
│   │   │   ├── workday.py           # Workday sync logic
│   │   │   └── ... (50+ integrations)
│   │   ├── ats/
│   │   └── external_service_helper.py  # Service locator
│   │
│   └── internal/                    # 🏢 Business logic (orchestration)
│       ├── common/
│       │   └── connector_service.py # Job status management
│       ├── hris/
│       │   ├── hris_employee_service.py  # Unify employees
│       │   ├── hris_company_service.py   # Unify companies
│       │   └── ...
│       └── ats/
│
├── repositories/                    # 🗃️ Database access layer
│   ├── common/
│   │   ├── connector_repository.py
│   │   └── connector_oauth_repository.py
│   ├── hris/
│   └── ats/
│
├── models/                          # 📋 Pydantic/SQLAlchemy models
├── utils/                           # 🔧 Helpers (HTTP client, logging, etc.)
└── README.md                        # 📖 Documentation
```

---

## Complete Example: BambooHR Employee Sync

### 1. Job Payload (JSON)

```json
{
  "job_id": "01932abc-1234-5678-9abc-def012345678",
  "org_id": "01931xyz-abcd-efgh-ijkl-mnop12345678",
  "connector_id": "conn_bamboohr_789",
  "integration_name": "bamboohr",
  "category": "HRIS",
  "initial_sync": true,
  "is_test_account": false,
  "sync_frequency": 24,
  "next_sync_start_time": "2024-12-04T00:00:00Z"
}
```

### 2. Scheduler: Enqueue Job

**File:** `scheduler.py`

```python
def schedulerMainTask():
    # Query database for jobs ready to run
    jobs = getJobsToSchedule()  
    # → SELECT * FROM connector_job WHERE next_sync_start_time <= NOW() AND status = 'PENDING'
    
    for job in jobs:
        enqueued = enqueueJob(job)
        if enqueued:
            markJobAsSchedulerProcessed(job)

def enqueueJob(job: dict):
    integration_name = job.get("i_name")  # "bamboohr"
    category = job.get("category")        # "HRIS"
    
    task = get_task(category=category, name=integration_name)
    # → Returns: start_bamboohr_sync
    
    if task is None:
        return False
    
    task.send(job)  # ← Publishes to RabbitMQ
    return True

def markJobAsSchedulerProcessed(job: dict):
    ConnectorRepository.update_connector_job(
        job_id=job["job_id"],
        data={
            "status": JobStatus.SCHEDULER_PROCESSED,
            "display_status": JobDisplayStatus.SYNCING,
            "last_sync_start_time": datetime.now(timezone.utc)
        }
    )
```

### 3. Worker: Process Job

**File:** `main.py`

```python
@dramatiq.actor(queue_name="default", max_retries=0)
def start_bamboohr_sync(job: dict):
    """Dramatiq actor for BambooHR sync"""
    start_integration_sync(job)

def start_integration_sync(job: dict):
    integration_name = job.get("integration_name")  # "bamboohr"
    category = job.get("category")                  # "HRIS"
    org_id = job.get("org_id")
    connector_id = job.get("connector_id")
    initial_sync = job.get("initial_sync")

    try:
        # ──────────────────────────────────────────────────
        # STEP 1: Mark job as picked by worker
        # ──────────────────────────────────────────────────
        job_start_time = ConnectorService.mark_connector_job_scheduler_processed(job)
        # → UPDATE connector_job SET status = 'PICKED_BY_WORKER' WHERE id = job_id
        
        # ──────────────────────────────────────────────────
        # STEP 2: Fire pre-sync webhook
        # ──────────────────────────────────────────────────
        webhook = WebhookService(org_id=org_id, connector_id=connector_id)
        webhook.fire_pre_sync_webhooks(job=job)
        # → POST https://customer-webhook.com/pre-sync
        
        # ──────────────────────────────────────────────────
        # STEP 3: Get integration service class
        # ──────────────────────────────────────────────────
        external_class = ExternalServiceHelper.fetch_integration_service_by_name(
            name=integration_name, 
            category=category
        )
        # → Returns: BambooHRExternalService class
        
        # ──────────────────────────────────────────────────
        # STEP 4: Execute sync
        # ──────────────────────────────────────────────────
        external_class(org_id=org_id, connector_id=connector_id).start_job(
            initial_sync=initial_sync
        )
        
        # ──────────────────────────────────────────────────
        # STEP 5: Mark job as done
        # ──────────────────────────────────────────────────
        ConnectorService.mark_connector_job_done(job=job, job_start_time=job_start_time)
        # → UPDATE connector_job SET 
        #     status = 'DONE', 
        #     next_sync_start_time = NOW() + sync_frequency hours
        
        # ──────────────────────────────────────────────────
        # STEP 6: Fire post-sync webhook
        # ──────────────────────────────────────────────────
        webhook.fire_post_sync_webhooks(job=job)
        # → POST https://customer-webhook.com/post-sync
        
        logger.info(f"{integration_name} Sync Done for connector_id: {connector_id}")
        
    except Exception as e:
        logger.exception(f"Error in syncing data", e)
        
        # ──────────────────────────────────────────────────
        # ERROR HANDLING
        # ──────────────────────────────────────────────────
        ConnectorService.mark_connector_job_failed(job=job)
        webhook.fire_sync_error_webhook(job=job)
        SlackAlertService.send_error_alert(
            org_id=org_id,
            connector_id=connector_id,
            integration_name=integration_name,
            message=str(e),
            traceback=traceback.format_exc()
        )
```

### 4. External Service: Fetch Data

**File:** `services/external/hris/bamboohr.py`

```python
class BambooHRExternalService(BaseHrisIntegrationService):
    def start_job(self, initial_sync: bool = True):
        self.prepare_initial_setup()      # Load config, OAuth, headers
        self.start_fetching_data(initial_sync)
    
    def prepare_initial_setup(self):
        # ────────────────────────────────────────────
        # Load connector configuration from database
        # ────────────────────────────────────────────
        self._load_configuration()
        # → SELECT config, access_token, refresh_token 
        #   FROM connector WHERE id = connector_id
        
        self._setup_headers()
        # → headers = {"Authorization": "Bearer <access_token>"}
        
        # ────────────────────────────────────────────
        # Validate configuration (test API call)
        # ────────────────────────────────────────────
        if not self._valid_configuration():
            self._update_configuration()  # Refresh OAuth token
    
    def _fetch_employees(self):
        # ────────────────────────────────────────────
        # STEP 1: Fetch employee dataset (pagination)
        # ────────────────────────────────────────────
        employee_dataset_dict = self._fetch_employee_dataset()
        # → POST https://api.bamboohr.com/api/gateway.php/{domain}/v1/reports/custom
        #   Paginated: page=1, page_size=100
        
        # ────────────────────────────────────────────
        # STEP 2: Fetch full employee report
        # ────────────────────────────────────────────
        endpoint = f"{self.base_domain}/v1/reports/custom"
        result = self._make_external_request(
            endpoint=endpoint,
            method=HTTPMethod.POST,
            params={"format": "JSON", "onlyCurrent": "False"},
            data=bamboohr_config.EMPLOYEE_FIELDS_PAYLOAD
        )
        
        if result.success:
            tp_employees = helper.convert_from_json_string(result.response.text)["employees"]
            
            # ────────────────────────────────────────────
            # STEP 3: Unify & store in database
            # ────────────────────────────────────────────
            HrisEmployeeService.unify_employees(
                data=tp_employees,
                connector_id=self.connector_id,
                integration=IntegrationsName.BAMBOOHR,
                companies_dict=self.supplemental_data.get("company", {}),
                group_dict=self.supplemental_data.get("group", {}),
                scope=self.field_scopes.get(ScopeModels.EMPLOYEE, [])
            )
        
        return HrisEmployeeService.get_employees_remote_id_dict(
            connector_id=self.connector_id
        )
```

### 5. Internal Service: Unify & Store

**File:** `services/internal/hris/hris_employee_service.py`

```python
class HrisEmployeeService:
    @classmethod
    def unify_employees(
        cls,
        data: list[dict],
        connector_id: UUID,
        integration: IntegrationsName,
        companies_dict: dict,
        group_dict: dict,
        scope: list[str]
    ):
        """
        Unifies third-party employee data to Bindbee schema and upserts to DB
        
        Transaction: Batched upsert within single transaction
        Idempotency: Uses remote_id as unique key (ON CONFLICT UPDATE)
        """
        unified_employees = []
        
        for tp_employee in data:
            # ────────────────────────────────────────────
            # Map third-party fields → Bindbee schema
            # ────────────────────────────────────────────
            unified_employee = {
                "connector_id": connector_id,
                "remote_id": tp_employee.get("id"),
                "first_name": tp_employee.get("firstName"),
                "last_name": tp_employee.get("lastName"),
                "email": tp_employee.get("workEmail"),
                "mobile_phone_number": tp_employee.get("mobilePhone"),
                "employment_status": cls._map_employment_status(
                    tp_employee.get("employmentStatus")
                ),
                "job_title": tp_employee.get("jobTitle"),
                "company_id": companies_dict.get(tp_employee.get("company")),
                "manager_id": None,  # Will be resolved in second pass
                "raw_data": tp_employee,
                "modified_at": datetime.now(timezone.utc)
            }
            
            # Apply field scoping (only include enabled fields)
            if scope:
                unified_employee = {
                    k: v for k, v in unified_employee.items() 
                    if k in scope or k in ["connector_id", "remote_id"]
                }
            
            unified_employees.append(unified_employee)
        
        # ────────────────────────────────────────────
        # Batch upsert to database
        # ────────────────────────────────────────────
        HrisEmployeeRepository.batch_upsert(
            employees=unified_employees,
            conflict_keys=["connector_id", "remote_id"]
        )
        # → INSERT INTO hris_employee (...) 
        #   VALUES (...) 
        #   ON CONFLICT (connector_id, remote_id) 
        #   DO UPDATE SET first_name = EXCLUDED.first_name, ...
```

---

## Retry & Error Handling

### Current Implementation (No Built-in Retries)

```python
@dramatiq.actor(queue_name="default", max_retries=0)
def start_bamboohr_sync(job: dict):
    start_integration_sync(job)
```

**Current Behavior:**
- `max_retries=0` → No automatic retries
- Errors are logged and Slack alerts sent
- Job marked as `FAILED` in database
- Manual intervention required to reprocess

### Recommended: Exponential Backoff Pattern

```python
import dramatiq
from dramatiq.middleware import Retries

@dramatiq.actor(
    queue_name="default",
    max_retries=5,
    min_backoff=15000,      # 15 seconds
    max_backoff=3600000,    # 1 hour
    throws=(SyncError,)     # Only retry on specific errors
)
def start_bamboohr_sync_v2(job: dict):
    try:
        start_integration_sync(job)
    except RateLimitError as e:
        # Don't retry rate limits immediately
        raise Retries.Retry(delay=e.retry_after_seconds * 1000)
    except TemporaryAPIError as e:
        # Retry with exponential backoff
        raise
    except PermanentError as e:
        # Don't retry, mark as failed immediately
        mark_job_failed(job, error=str(e))
        raise Retries.Stop()
```

**Backoff Schedule:**
- Retry 1: 15 seconds
- Retry 2: 30 seconds
- Retry 3: 60 seconds (1 minute)
- Retry 4: 120 seconds (2 minutes)
- Retry 5: 240 seconds (4 minutes)

### Dead Letter Queue

```python
# In dramatiq broker setup
broker.add_middleware(
    dramatiq.middleware.DLQ(max_attempts=5)
)

# Failed jobs go to: <queue_name>.DLQ
# Example: "default.DLQ"

# Inspect DLQ
from dramatiq import get_broker
broker = get_broker()
dlq = broker.get_declared_queues()["default.DLQ"]
messages = dlq.messages()
```

---

## Observability & Monitoring

### 1. Metrics (Prometheus)

```python
# Expose metrics endpoint
from prometheus_client import Counter, Histogram, Gauge, start_http_server

# Counters
jobs_processed = Counter('worker_jobs_processed_total', 'Total jobs processed', ['integration', 'status'])
jobs_failed = Counter('worker_jobs_failed_total', 'Total jobs failed', ['integration', 'error_type'])

# Histograms
job_duration = Histogram('worker_job_duration_seconds', 'Job processing time', ['integration'])
api_request_duration = Histogram('worker_api_request_duration_seconds', 'API request time', ['integration', 'endpoint'])

# Gauges
queue_size = Gauge('worker_queue_size', 'Number of jobs in queue')

# In worker
def start_integration_sync(job: dict):
    integration = job.get("integration_name")
    start_time = time.time()
    
    try:
        # ... sync logic ...
        jobs_processed.labels(integration=integration, status='success').inc()
    except Exception as e:
        jobs_processed.labels(integration=integration, status='failed').inc()
        jobs_failed.labels(integration=integration, error_type=type(e).__name__).inc()
    finally:
        duration = time.time() - start_time
        job_duration.labels(integration=integration).observe(duration)

# Start Prometheus HTTP server
start_http_server(9191)  # Exposed in docker-compose on port 9191
```

### 2. Structured Logging (Loguru + Loki)

```python
from loguru import logger

logger.add(
    sink=LokiHandler(url="https://loki.example.com/loki/api/v1/push"),
    format="{extra[job_id]} | {extra[connector_id]} | {message}",
    level="INFO"
)

# In worker
def start_integration_sync(job: dict):
    with logger.contextualize(
        job_id=job["job_id"],
        connector_id=job["connector_id"],
        integration=job["integration_name"],
        org_id=job["org_id"]
    ):
        logger.info("Starting sync")
        # ... sync logic ...
        logger.info("Sync completed", employees_synced=count)
```

### 3. Trace IDs (Context Propagation)

```python
import uuid
import contextvars

trace_id_var = contextvars.ContextVar('trace_id', default=None)

def start_integration_sync(job: dict):
    trace_id = str(uuid.uuid4())
    trace_id_var.set(trace_id)
    
    logger.info(f"[{trace_id}] Starting sync for {job['connector_id']}")
    
    # Propagate to external requests
    headers = {
        "X-Trace-ID": trace_id,
        **self.headers
    }
```

### 4. Alerts

```yaml
# Prometheus AlertManager rules
groups:
  - name: worker_alerts
    rules:
      - alert: HighJobFailureRate
        expr: |
          rate(worker_jobs_failed_total[5m]) / rate(worker_jobs_processed_total[5m]) > 0.1
        for: 5m
        annotations:
          summary: "High job failure rate detected"
          
      - alert: QueueBacklog
        expr: worker_queue_size > 100
        for: 10m
        annotations:
          summary: "Queue backlog over 100 jobs"
```

---

## Security & Operations

### 1. Secrets Management

```python
# Environment variables (loaded from .env or secrets manager)
BAMBOOHR_CLIENT_ID = os.getenv("BAMBOOHR_CLIENT_ID")
BAMBOOHR_CLIENT_SECRET = os.getenv("BAMBOOHR_CLIENT_SECRET")
PGP_PRIVATE_KEY = os.getenv("PGP_PRIVATE_KEY")  # Base64 encoded

# OAuth tokens stored in database (encrypted at rest)
# → connector_oauth table with access_token, refresh_token
```

### 2. Rate Limiting

```python
from ratelimit import limits, sleep_and_retry

class BambooHRExternalService:
    @sleep_and_retry
    @limits(calls=100, period=60)  # 100 calls per minute
    def _make_external_request(self, endpoint: str, **kwargs):
        return super()._make_external_request(endpoint, **kwargs)
```

### 3. Graceful Shutdown

```python
# Dramatiq handles SIGTERM gracefully
# In docker-compose:
#   stop_grace_period: 30s

# Workers finish current jobs before shutdown
# Jobs not yet picked up remain in queue
```

---

## Best Practices Checklist

When adding a new worker job:

- [ ] Define Pydantic model for job payload
- [ ] Create Dramatiq actor with appropriate `max_retries`
- [ ] Implement idempotency in database layer (use `ON CONFLICT`)
- [ ] Add structured logging with job context
- [ ] Emit Prometheus metrics (success, failure, duration)
- [ ] Handle OAuth token refresh in external service
- [ ] Fire webhooks (pre-sync, post-sync, error)
- [ ] Add unit tests for unification logic
- [ ] Test locally with Docker Compose
- [ ] Document new integration in `README.md`

---

## Quick Start (Local Testing)

```bash
# 1. Start RabbitMQ, Scheduler, Worker
cd workers
docker-compose up

# 2. Monitor logs
docker logs -f worker
docker logs -f scheduler

# 3. Check RabbitMQ UI
open http://localhost:15672  # guest/guest

# 4. Manually trigger sync (for testing)
python -c "
from scheduler import enqueueJob
job = {'job_id': 'test123', 'integration_name': 'bamboohr', ...}
enqueueJob(job)
"

# 5. Monitor database
psql -h localhost -U postgres -d bindbee
SELECT * FROM connector_job ORDER BY created_at DESC LIMIT 10;
```

---

## Debugging Playbook

### Problem: Job stuck in SCHEDULER_PROCESSED

**Steps:**
1. Check RabbitMQ queue: `http://localhost:15672/#/queues`
2. Verify worker is running: `docker ps | grep worker`
3. Check worker logs: `docker logs worker`
4. Manually requeue: `enqueueJob(job)`

### Problem: Sync fails with 401 Unauthorized

**Steps:**
1. Check connector OAuth status: `SELECT * FROM connector_oauth WHERE connector_id = 'xxx'`
2. Verify token expiry: `access_token`, `expires_at`
3. Manually refresh token via API
4. Re-run sync

### Problem: High memory usage

**Steps:**
1. Check batch size in sync logic
2. Reduce page size for paginated requests
3. Enable streaming for large datasets
4. Monitor with: `docker stats worker`

---

**End of Documentation**
