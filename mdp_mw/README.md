# Django-CID Integration for Middleware Stack



---

## Overview

### Problem
In a microservices or distributed system, tracing a single user request across multiple services, containers, and log aggregation systems is difficult without a unifying identifier.

### Solution
1. **Generate or propagate** a unique `correlation_id` (UUID) at the nginx ingress point
2. **Forward** the correlation ID through HTTP headers (X-Correlation-Id) to the application
3. **Inject** the correlation ID into every log line in Django middleware
4. **Parse and enrich** logs in Fluent-bit to extract the correlation ID
5. **Query** Elasticsearch/Grafana by correlation ID to see the complete request lifecycle

### Scope
-  Infrastructure/configuration changes only
-  Nginx proxy configuration
-  Django middleware for correlation ID propagation
-  Fluent-bit parsing pipelines
-  No changes to `views.py` or application endpoints
-  No custom headers added in API responses (except via django-cid config)

---
## Architecture

```
Client Request
    ↓
Nginx (proxy) ← generates/forwards X-Correlation-Id header
    ↓ (uwsgi_param HTTP_X_CORRELATION_ID)
uWSGI application server
    ↓
Django
    ├─ CidMiddleware (reads/assigns correlation_id)
    ├─ LogRequestResponseMiddleware (logs with correlation_id)
    └─ Application endpoints (no changes)
    ↓
Docker logging driver (fluentd)
    ↓
Fluent-bit (log aggregator)
    ├─ Input: Docker forward protocol
    ├─ Filters: multiline, parser (regex + JSON)
    └─ Output: Elasticsearch
    ↓
Elasticsearch (log storage/indexing)
    ↓
Grafana (visualization/querying)
```
---

An end-to-end request tracing implementation that correlates logs across your entire stack using per-request UUID (`correlation_id`). Every log line—from Django application logs to uWSGI access logs to downstream Elasticsearch documents—can be tied together for unified observability.

---

### Log Flow Example

```
1. Client:        GET /api/v1.0.0/t24/get/branches
2. Nginx:         forwards as uwsgi_param HTTP_X_CORRELATION_ID=<uuid>
3. Django CID:    reads from HTTP header → correlation_id = <uuid>
4. Middleware:    logs ➡️ GET /api/v1.0.0/t24/get/branches (with correlation_id)
5. Application:   processes request
6. Middleware:    logs ⬅️ 200 GET /api/v1.0.0/t24/get/branches (with correlation_id)
7. uWSGI:         access/perf log line (separate channel)
8. Fluent-bit:    parses all three log types, adds correlation_id field
9. Elasticsearch: indexes documents with correlation_id.keyword field
10. Grafana:      user queries correlation_id:"<uuid>" → sees all 3 log entries
```

---

## What Changed

### 1. `requirements.txt`
Add the packages for correlation ID generation and structured logging:

```txt
django-cid         # Correlation ID middleware
```

**Purpose:**
- `django-cid`: Generates/propagates per-request correlation IDs
- `python-json-logger`: Enables structured JSON logging for cleaner Fluent-bit parsing

### 2. `settings.py`

#### Install Django-CID

Add to `INSTALLED_APPS`:
```python
INSTALLED_APPS = [
    # ...
    'cid.apps.CidAppConfig',  # Must be before custom middleware
    # ...
]
```

#### Register Middleware

Add to `MIDDLEWARE` (order matters: after Django's base middleware but before logging middleware):
```python
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    # ... other Django middleware ...
    
    'cid.middleware.CidMiddleware',  # Read/generate correlation_id
]
```

#### CID Configuration

Add to `settings.py`:
```python
# Generate correlation IDs if not provided upstream
CID_GENERATE = True

#comment to generate correlation id for incoming requests and uncomment to use existing correlation id from incoming requests
#CID_HEADER = 'HTTP_X_CORRELATION_ID'

# Header to send in response
CID_RESPONSE_HEADER = 'X-Correlation-Id'
```

#### Logging Configuration

Update your logging formatters to include `correlation_id`:

```python
LOGGING = {
    "version": 1,
    "disable_existing_loggers": False,

    "filters": {
        "correlation_id": {
            "()": "mdpcms.logging_filters.CorrelationIdFilter",
        },
    },

    "formatters": {
        "standard": {
            "format": "[%(asctime)s] %(levelname)s %(name)s: %(message)s [CID: %(correlation_id)s]",
        },
    },

    "handlers": {
        "rotating_file": {
            "level": "INFO",
            "class": "logging.handlers.RotatingFileHandler",
            "filename": log_file_path,
            "maxBytes": 50 * 1024 * 1024,  # 50MB
            "backupCount": 0,
            "formatter": "standard",
            "filters": ["correlation_id"],  # ✅ important
        },
    },

    "loggers": {
        "django.request": {
            "handlers": ["rotating_file"],
            "level": "INFO",
            "propagate": False,
        },

        # ✅ optional but recommended:
        # capture your app logs too (logger = logging.getLogger(__name__))
        "mdpcms": {
            "handlers": ["rotating_file"],
            "level": "INFO",
            "propagate": False,
        },
    },

    # ✅ optional safety net: anything else that logs won't crash either
    "root": {
        "handlers": ["rotating_file"],
        "level": "INFO",
    },
}
```

The formatter calls `cid.middleware.get_cid()` to retrieve the current request's correlation ID.

### 3. `app/logging_filters.py` 

Custom middleware that logs request and response with correlation ID:

```python

# mdpcms/logging_filters.py
from cid.middleware import get_cid

class CorrelationIdFilter:
    def filter(self, record):
        # Always provide correlation_id so formatter never crashes
        try:
            cid = get_cid()
        except Exception:
            cid = None

        record.correlation_id = cid or "-"
        return True

```
### 4. `app/logging_middleware.py` 

Custom middleware that logs request and response with correlation ID:

```python
import logging
from django.utils.deprecation import MiddlewareMixin
from cid.middleware import get_cid

logger = logging.getLogger("django.request")

class LogRequestResponseMiddleware(MiddlewareMixin):
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        cid = get_cid()  # ✅ get your CID for this request

        # Request body
        try:
            request_body = request.body.decode("utf-8")
        except Exception:
            request_body = "<unable to decode request body>"

        logger.info(f"[CID:{cid}] ➡️ {request.method} {request.get_full_path()} BODY: {request_body}")

        response = self.get_response(request)

        # Response body (avoid streaming)
        try:
            if hasattr(response, "content"):
                response_body = response.content.decode("utf-8")
            else:
                response_body = "<streaming or no content>"
        except Exception:
            response_body = "<unable to decode response body>"

        logger.info(f"[CID:{cid}] ⬅️ {response.status_code} {request.method} {request.path} RESPONSE: {response_body}")

        return response
```

**What it does:**
- Logs request line with HTTP method and path
- Logs response line with status code
- Includes `correlation_id` in both logs
- Emits JSON-structured logs for easy Fluent-bit parsing

### 5. `docker-compose.yml`

Update the compose stack to wire up logging infrastructure:

```yaml
services:
  app:
    build:
      context: .
      args:
        - DEV=true
    ports:
      - "8000:8000"
    volumes:
      - ./app:/app
      - ./logs:/app/logs
      # - dev-static-data:/vol/web
    command: >
      sh -c "python manage.py wait_for_db &&
             python manage.py migrate &&
             python manage.py runserver 0.0.0.0:8000"
    environment:
      - DB_HOST=db
      - DB_NAME=devdb
      - DB_USER=devuser
      - DB_PASS=changeme
      - DEBUG=1
      - CID_GENERATE=true # Enable CID generation
    depends_on:
      - db

  db:
    image: postgres:15.3-alpine3.18
    volumes:
      - dev-db-data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=devdb
      - POSTGRES_USER=devuser
      - POSTGRES_PASSWORD=changeme

volumes:
  dev-db-data:
```

### 6. `proxy/default.conf.tpl`

Nginx proxy template that forwards the correlation ID header:

```nginx
upstream django_app {
    server ${APP_HOST}:${APP_PORT};
}

server {
    listen ${LISTEN_PORT};
    server_name _;

    location / {
        uwsgi_pass django_app;
        include /etc/nginx/uwsgi_params;
        
        # Forward correlation ID header to application
        uwsgi_param HTTP_X_CORRELATION_ID $http_x_correlation_id;
    }
}


## Implementation Details

### Correlation ID Flow

1. **Client sends request:**
   ```bash
   curl -H "X-Correlation-Id: <uuid>" http://host/api/v1.0.0/t24/get/branches
   ```

2. **Nginx receives request:**
   - Reads `X-Correlation-Id` header (or allows empty if not present)
   - Forwards as `uwsgi_param HTTP_X_CORRELATION_ID`

3. **Django receives request:**
   - `CidMiddleware` reads `HTTP_X_CORRELATION_ID` from wsgi environ
   - If empty and `CID_GENERATE = True`, generates a new UUID
   - Stores in thread-local storage via `cid.middleware.set_cid(uuid)`

4. **Middleware logging:**
   - `LogRequestResponseMiddleware` calls `get_cid()` to retrieve the correlation ID
   - Logs request/response with `correlation_id` field

5. **Container stdout/stderr:**
   - Docker logging driver (fluentd) captures all logs
   - Tags them as `docker.app`

6. **Fluent-bit processing:**
   - Receives logs via forward protocol
   - Parses JSON to extract `correlation_id`
   - Enriches with metadata (docker labels, etc.)
   - Sends to Elasticsearch

7. **Elasticsearch indexing:**
   - Documents indexed with `correlation_id.keyword` for exact matching
   - Enables sub-millisecond queries

8. **Grafana visualization:**
   - Query logs by: `correlation_id.keyword:"<uuid>"`
   - See complete request lifecycle

### Multiline Log Handling

Python stack traces span multiple lines. Fluent-bit's multiline filter reassembles them:

```ini
[FILTER]
    Name          multiline
    Match         docker.app
    Parser        docker_multiline
    Key_Name      log
```

The parser rule:
```ini
Rule "start_state" "/^\d{4}-\d{2}-\d{2}/" "cont"
Rule "cont" "/^(?!\d{4}-\d{2}-\d{2})/" "cont"
```

This merges lines that don't start with a timestamp back into the previous line.

### Parser Selection

Fluent-bit applies parsers in order. **Only the first matching parser enriches the record:**

```ini
[FILTER]
    Name    parser
    Match   docker.app
    Key_Name log
    Parser  uwsgi_perf_extract     # Try this first
    Parser  json_generic           # If that fails, try this
    Reserve_Data On
```

- If log matches uWSGI regex → extracts fields from uWSGI format
- Else if log is valid JSON → extracts fields from JSON
- Else → passes through unchanged

---

## Validation & Usage

### Test Request

```bash
curl -X GET \
  -H "X-Correlation-Id: test-correlation-123" \
  "http://<HOST>/api/v1.0.0/t24/get/branches"
```

### Query Elasticsearch

```bash
# Get all logs for a correlation ID
curl -X GET "elasticsearch:9200/_search?q=correlation_id:test-correlation-123&pretty"
```

### Query Grafana

1. **Create Elasticsearch datasource** (if not already configured)
   - URL: `http://elasticsearch:9200`
   - Index pattern: `logs-*`

2. **Build query panel:**
   - Metrics: `Count` of documents
   - Filters: `correlation_id.keyword: "<uuid>"`
   - Group by: `@timestamp` (ascending)

3. **Expected results:**
   - 1x middleware request log (➡️)
   - 1x middleware response log (⬅️)
   - 1x uWSGI access/perf log
   - Any additional logs (errors, database, etc.) for the same request

### Log Example Output

**Middleware request log (JSON):**
```json
{
  "timestamp": "2025-01-12T09:37:00Z",
  "level": "INFO",
  "logger": "IPS.logging_middleware",
  "message": "➡️ GET /api/v1.0.0/t24/get/branches",
  "correlation_id": "550e8400-e29b-41d4-a716-446655440000",
  "direction": "➡️",
  "method": "GET",
  "path": "/api/v1.0.0/t24/get/branches",
  "query_string": ""
}
```

**Middleware response log (JSON):**
```json
{
  "timestamp": "2025-01-12T09:37:00.125Z",
  "level": "INFO",
  "logger": "IPS.logging_middleware",
  "message": "⬅️ 200 GET /api/v1.0.0/t24/get/branches",
  "correlation_id": "550e8400-e29b-41d4-a716-446655440000",
  "direction": "⬅️",
  "status_code": 200,
  "method": "GET",
  "path": "/api/v1.0.0/t24/get/branches"
}
```

---
