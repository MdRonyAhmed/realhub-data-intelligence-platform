# RealHub Data Intelligence Platform

<div align="center">

![Python](https://img.shields.io/badge/Python-3.11+-3776AB?style=for-the-badge&logo=python&logoColor=white)
![Django](https://img.shields.io/badge/Django-5.x-092E20?style=for-the-badge&logo=django&logoColor=white)
![Celery](https://img.shields.io/badge/Celery-5.x-37814A?style=for-the-badge&logo=celery&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-7.x-DC382D?style=for-the-badge&logo=redis&logoColor=white)
![AWS S3](https://img.shields.io/badge/AWS_S3-Cloud_Storage-FF9900?style=for-the-badge&logo=amazons3&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-CI%2FCD-2088FF?style=for-the-badge&logo=githubactions&logoColor=white)

**A production-grade, distributed real estate data intelligence and web scraping platform targeting major North American markets — Zillow, Realtor.ca, and Realtor.com — built across three engineering evolution phases.**

[Architecture Overview](#️-architecture-overview) · [Tech Stack](#-tech-stack) · [System Evolution](#-system-evolution) · [API Reference](#-api-endpoints) · [DevOps](#️-devops--cicd)

</div>

---

## 📌 Project Overview

**RealHub Data Intelligence Platform** is an enterprise-grade real estate data extraction and intelligence system engineered to reliably scrape structured data from heavily bot-protected real estate platforms across Canada and the United States.

The system was designed and evolved through **three distinct engineering phases**, each addressing compounding challenges in scalability, response-time requirements, anti-bot evasion, and cloud data pipeline reliability.

### What This Platform Extracts

| Platform | Data Targets |
|---|---|
| **Zillow** | Property details, agent profiles, active listings, property image galleries |
| **Realtor.ca** | Agent details, live listing inventories, property search (paginated API calls) |
| **Realtor.com** | Agent profiles, active listings (via XHR network capture), property data |
| **Google Search** | Address-to-URL resolution across all three platforms |

---

## 🏗️ Architecture Overview

This platform evolved across three repositories, each solving increasingly complex infrastructure challenges:

```
┌─────────────────────────────────────────────────────────────────────┐
│                     SYSTEM EVOLUTION TIMELINE                       │
├────────────────────┬──────────────────────┬─────────────────────────┤
│  Phase 1           │  Phase 2             │  Phase 3                │
│  realhub-          │  realhub-instant-    │  realhub-scraper-api    │
│  listings-api-v2   │  scraper             │                         │
├────────────────────┼──────────────────────┼─────────────────────────┤
│  Async Queue       │  Synchronous         │  Anti-Bot Hardened      │
│  Architecture      │  Instant Execution   │  Cloud-Native Sinks     │
│                    │                      │  Push Webhook Delivery  │
└────────────────────┴──────────────────────┴─────────────────────────┘
```

### High-Level Data Flow

```
 API Client (JWT Auth)
        │
        ▼
 Django REST Framework
        │
        ├──────────────────────────────────┐
        │  [Phase 1 & 3 - Async]          │  [Phase 2 - Instant]
        ▼                                  ▼
  task.delay()                      task.apply_async().get()
  → returns task_id                 → blocks & returns result directly
        │
        ▼
  Redis Broker (Message Queue)
        │
        ▼
  Celery Worker Pool (concurrency=20)
        │
        ├── Fetch URL
        │     ├── Direct HTTP + Header Spoofing (http_request.py)
        │     ├── ZenRows Proxy API (premium residential proxies)
        │     │     ├── js_render=true
        │     │     ├── premium_proxy=true
        │     │     └── XHR network capture
        │     ├── Zyte API (browser HTML / HTTP body)
        │     └── Headless Browser (Selenium / DrissionPage / Playwright)
        │           └── Undetected ChromeDriver + proxy auth extension
        │
        ├── Parse & Extract
        │     ├── BeautifulSoup4 (HTML parsing)
        │     ├── __NEXT_DATA__ JSON extraction (Zillow / Realtor.com SSR)
        │     └── Realtor.ca REST API v2 (structured JSON payloads)
        │
        ├── Cache Result → Redis (indefinite TTL for agent profiles)
        │
        ├── Upload to Cloud → AWS S3 (boto3 stream upload)
        │     └── s3://bucket/realhub_listings_api_v2/{platform}/{type}/{uuid}.json
        │
        ├── Persist to DB → CompletedTask (task_id, status, response, source)
        │
        └── Push Result → Client Webhook (multipart/form-data POST)
              └── push_task_status_to_client() × 2 domains (prod + dev)

 Monitoring
        ├── GET /api/monitoring/health/         → Django health check
        └── GET /api/monitoring/celery-health/  → Celery ping (10s timeout)

 Error Telemetry
        └── @task_failure signal → Redis Stream (error_notifications{stream})
```

---

## 🛠️ Tech Stack

| Layer | Technology | Purpose |
|---|---|---|
| **Web Framework** | Django 5.x | REST API, ORM, Admin panel |
| **API Layer** | Django REST Framework + SimpleJWT | JWT-authenticated endpoints |
| **Task Queue** | Celery 5.x | Distributed background job processing |
| **Message Broker** | Redis 7.x (cluster + TLS in prod) | Celery broker & result backend |
| **Caching** | django-redis | Response memoization (agent/property profiles) |
| **HTTP Scraping** | `requests` + session management | Direct platform HTTP calls |
| **Anti-Bot APIs** | ZenRows, Zyte API | Premium residential proxy, JS rendering, XHR capture |
| **Browser Automation** | Selenium, Undetected ChromeDriver | Headless rendering w/ proxy auth extension |
| **Chromium Automation** | DrissionPage 4.x | High-performance browser control |
| **Headless Rendering** | Playwright | Cross-browser JS execution |
| **Virtual Display** | Xvfbwrapper | Headless X11 frame buffer for Chrome on Linux |
| **HTML Parsing** | BeautifulSoup4, lxml | Structured DOM extraction |
| **Cloud Storage** | AWS S3 (boto3) | JSON payload archival & retrieval |
| **CI/CD** | GitHub Actions | Automated SSH deploy pipelines |
| **Process Manager** | Gunicorn + Nginx | Production WSGI serving |
| **Rate Limiting** | django-ratelimit | API endpoint protection |

---

## 🔄 System Evolution

### Phase 1 — Asynchronous Queue Architecture (`realhub-listings-api-v2`)

**Problem:** Long-running scraping jobs (10–60+ seconds per property) caused HTTP gateway timeouts when called synchronously.

**Solution:** Fully decoupled async architecture. Every scraping job is dispatched to a Celery worker via Redis, and the API immediately returns a `task_id` for the client to poll.

```python
# Dispatch a task — returns immediately
task = scrape_zillow_agent_details.delay(profile_url)
return Response({'task_id': task.id, 'status': 'Task started.'})
```

**Key architectural components:**

- **`CompletedTask` model** — Persists all task outcomes (task_id, status, response JSON, source, timestamp) via `@task_postrun` and `@task_failure` Celery signals.
- **`PortBlockHistory` model** — Tracks blocked outbound ports per domain to inform adaptive proxy selection.
- **Redis Stream Error Notifications** — On task failure, error metadata is published to a Redis Stream (`error_notifications{stream}`) for real-time downstream alerting.
- **Celery Worker Configuration:**
  - `worker_concurrency = 20` — 20 simultaneous scrape workers
  - `task_acks_late = True` — Prevents job loss on worker crash
  - `worker_prefetch_multiplier = 1` — No queue flooding per worker
  - `task_track_started = True` — Full lifecycle visibility

```python
# Celery signal — auto-persists every completed task to the database
@task_postrun.connect
def log_task_success(sender=None, task_id=None, state=None, retval=None, **kwargs):
    with transaction.atomic():
        CompletedTask.objects.update_or_create(
            task_id=task_id,
            defaults={"status": state, "response": json.dumps(retval), ...}
        )
```

**Apps in this phase:**
- `zillow` — Agent details, agent property lists
- `realtor_ca` — Agent details, live listing inventories, property search
- `realtor_com` — Agent details, active listings (XHR capture)
- `google_search_manager` — Address-to-property-URL resolution
- `task_log` — Task audit log, filtering UI, admin access
- `monitoring` — `/health/`, `/celery-health/` endpoints

---

### Phase 2 — Synchronous Instant Execution (`realhub-instant-scraper`)

**Problem:** Some downstream consumers required immediate responses without building a polling mechanism.

**Solution:** Tasks are dispatched to Celery workers but the HTTP request **blocks and waits** for the result before returning the response.

```python
# Dispatch and block until result is ready
task = scrape_zillow_property_details.apply_async(args=[url])
result = task.get()  # blocks until the Celery worker returns
return Response({"property_details": result})
```

**New capabilities added in this phase:**
- `scrape_zillow_property_details` — Full property data extraction from `__NEXT_DATA__` JSON blob
- `scrape_zillow_property_images` — High-resolution image gallery extraction
- `scrape_zillow_agent_live_listings` — Paginated live listing feed for agents (by `zuid`)
- `realtor_com` — Extended with property detail scraping
- `realtor` (Realtor.ca) — Expanded endpoint coverage
- **`django-ratelimit`** — API-level rate limiting to protect scraper concurrency limits
- **`s3_bucket.py`** — First introduction of AWS S3 cloud storage sink
- **`nodejs_server.py`** — Node.js subprocess management for JS challenge bypass

**Trade-off acknowledged:** While synchronous blocking is simpler for consumers, it ties up a Gunicorn worker thread for the full scrape duration — acceptable for lightweight, fast-responding endpoints; not for bulk data operations.

---

### Phase 3 — Anti-Bot Hardening & Cloud-Native Sinks (`realhub-scraper-api`)

**Problem:** Major real estate platforms intensified bot detection, causing increasing failure rates. Results also needed to be proactively pushed to downstream systems rather than polled.

**Solution:** Integrated enterprise-grade anti-bot proxy services, multi-layered fallback chains, and webhook-based push delivery.

**Anti-bot stack (multi-tier fallback):**

```
Tier 1: Direct HTTP request with session + spoofed headers
        ↓ (on failure or 403)
Tier 2: ZenRows API (premium residential proxy + JS rendering)
        ↓ (on failure)
Tier 3: Zyte API (browser HTML or HTTP response body)
        ↓ (on failure)
Tier 4: Headless Chrome (Undetected ChromeDriver / DrissionPage)
         with proxy auth extension injected at browser startup
```

**ZenRows integration (three modes):**

```python
# Mode 1: Standard page fetch with wait
params = {'js_render': 'true', 'premium_proxy': 'true', 'wait': '5000'}
response = client.get(url, params=params)

# Mode 2: XHR network capture (intercepts AJAX calls)
params = {'js_render': 'true', 'json_response': 'true', 'wait': '10000'}
# → extracts XHR body matching a target API endpoint URL

# Mode 3: Session-pinned requests (consistent residential IP per session)
params = {'session_id': session_id, 'premium_proxy': 'true', ...}
```

**Zyte API integration (four modes):**

```python
# HTTP body (no browser)
{"url": url, "httpResponseBody": True}

# Browser HTML (full JS rendering)
{"url": url, "browserHtml": True}

# Network capture (intercept specific API calls)
{"networkCapture": [{"filterType": "url", "value": capture_url, ...}]}

# Session-persisted POST request
{"session": {"id": session_id}, "httpRequestMethod": "POST", ...}
```

**Retry & fault tolerance:**
```python
@shared_task(
    bind=True,
    autoretry_for=(Exception,),  # Retry on any exception
    retry_backoff=5,             # Exponential backoff starting at 5s
    max_retries=3                # Max 3 retry attempts
)
def scrape_zillow_property_details(self, url):
    ...
```

**Push webhook delivery (`push_api.py`):**
```python
# Automatically pushes task results to prod AND dev environment simultaneously
def push_task_status_to_client(type, task_id, status, result):
    for domain in [PUSH_API_DOMAIN, "dev.realhub.ai"]:
        # Multipart/form-data POST with server_key auth signature
        conn.request("POST", "/api/save-property-listing", payload, headers)
```

**Redis caching pattern (indefinite TTL for stable profiles):**
```python
cache_key = f"zillow_{url}"
cached = cache.get(cache_key)
if cached:
    return cached  # instant return — no scrape needed

# After successful scrape:
cache.set(cache_key, data, timeout=None)  # cache indefinitely
```

---

## 📡 API Endpoints

> All endpoints require JWT authentication via `Authorization: Bearer <token>`.
> Obtain tokens via `POST /api/token/` with Django user credentials.

### Zillow

| Method | Endpoint | Description | Response |
|---|---|---|---|
| `POST` | `/api/zillow/property-details/` | Full property data extraction | `{task_id}` or `{property_details}` |
| `POST` | `/api/zillow/agent-details/` | Agent profile scrape | `{task_id}` or `{agent_details}` |
| `POST` | `/api/zillow/property-images/` | High-res image gallery | `{task_id}` or `{images}` |
| `POST` | `/api/zillow/agent-listings/` | Agent active listings (paginated) | `{task_id}` or listings payload |

### Realtor.ca

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/realtor_ca/agent-details/` | Full agent profile extraction |
| `POST` | `/api/realtor_ca/agent-listings/` | Paginated live listing inventory |
| `POST` | `/api/realtor_ca/property-search/` | Property search by filters |

### Realtor.com

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/realtor_com/agent-details/` | Agent profile via `__NEXT_DATA__` |
| `POST` | `/api/realtor_com/agent-listings/` | Active listings via XHR capture |

### Google Search / Cross-Platform

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/google_search_manager/resolve/` | Resolves a property address → platform URLs |

### Task Management & Monitoring

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/task_log/` | Paginated task audit log with time-filter |
| `GET` | `/api/task_log/<id>/` | Full task detail with JSON response viewer |
| `GET` | `/api/monitoring/health/` | Django application health check |
| `GET` | `/api/monitoring/celery-health/` | Celery worker ping (10s timeout) |

---

## 🔐 Environment Configuration

Create a `.env` file in the project root before running:

```env
# Django
DJANGO_SECRET_KEY=your-secret-key
DEBUG=False
ALLOWED_HOSTS=yourdomain.com
IS_PRODUCTION=True
ENVIRONMENT=production

# Redis / Celery
REDIS_HOST=your-redis-host
REDIS_PORT=6379
REDIS_PASSWORD=your-redis-password

# Anti-Bot Services
ZENROWS_API_KEY=your-zenrows-api-key
ZYTE_API_KEY=your-zyte-api-key

# Proxy Pools
PROXY_HOST=host:port
PROXY_CREDENTIALS=username:password
SESSION_PROXY_HOST=host:port
SESSION_PROXY_CREDENTIALS=username:password

# AWS S3
AWS_ACCESS_KEY_ID=your-key-id
AWS_SECRET_ACCESS_KEY=your-secret-key
AWS_S3_REGION_NAME=us-east-1
AWS_STORAGE_BUCKET_NAME=your-bucket-name

# Client Push Webhook
PUSH_API_DOMAIN=yourplatform.com
SERVER_KEY=your-server-key

# Webhooks (Phase 1)
WEBHOOK_URL=https://your-webhook-endpoint.com
```

---

## 🚀 Local Development Setup

### Prerequisites

- Python 3.11+
- Redis server running locally
- Google Chrome installed (for browser automation)

### Installation

```bash
# Clone the repository
git clone https://github.com/yourusername/realhub-data-intelligence-platform.git
cd realhub-data-intelligence-platform

# Create virtual environment
python -m venv venv
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Configure environment
cp .env.example .env
# Edit .env with your credentials

# Run database migrations
python manage.py migrate

# Create a superuser
python manage.py createsuperuser
```

### Running the Platform

```bash
# Terminal 1 — Django development server
python manage.py runserver

# Terminal 2 — Celery worker (20 concurrent workers)
celery -A realhub_scraper_api worker --loglevel=info --concurrency=20

# Terminal 3 — (Optional) Celery task monitoring
celery -A realhub_scraper_api flower
```

### Getting an API Token

```bash
curl -X POST http://localhost:8000/api/token/ \
  -H "Content-Type: application/json" \
  -d '{"username": "admin", "password": "yourpassword"}'
```

---

## ☁️ DevOps & CI/CD

### GitHub Actions Automated Deployment

Push to `main` triggers a zero-downtime deployment via SSH:

```yaml
# .github/workflows/deploy.yml
- name: Connect, Pull & Restart Services
  run: |
    ssh ${{ secrets.SSH_USER }}@${{ secrets.SSH_HOST }} "
      cd ${{ secrets.WORK_DIR }} &&
      git pull &&
      sudo systemctl restart gunicorn &&
      sudo systemctl restart celery &&
      sudo systemctl restart nginx &&
      echo '✅ Deployment Finished!'
    "
```

**Required GitHub Secrets:**

| Secret | Description |
|---|---|
| `SSH_PRIVATE_KEY` | Private key for server SSH access |
| `SSH_HOST` | Production server IP/hostname |
| `SSH_USER` | SSH login username |
| `WORK_DIR` | Absolute path to project on server |
| `MAIN_BRANCH` | Branch to deploy (typically `main`) |

### Production Service Architecture

```
[Nginx]  ←── Reverse proxy / SSL termination
   │
[Gunicorn]  ←── Django WSGI server (multiple workers)
   │
[Django App]
   │
[Redis Cluster]  ←── Celery broker + result backend + cache (TLS)
   │
[Celery Workers]  ←── 20 concurrent scraper workers (systemd managed)
```

---

## 📊 Data Models

### `CompletedTask` — Task Audit Log

| Field | Type | Description |
|---|---|---|
| `task_id` | `CharField` (unique) | Celery task UUID |
| `status` | `CharField` | `SUCCESS`, `FAILURE`, `RETRY` |
| `task_name` | `CharField` | Human-readable task name |
| `source` | `CharField` | Platform: `Zillow`, `Realtor.ca`, `Realtor.com` |
| `response` | `JSONField` | Full scraped payload |
| `error` | `TextField` | Exception traceback (on failure) |
| `args` | `CharField` | Task arguments (on failure) |
| `date_completed` | `DateTimeField` | Auto-set timestamp |

### `PortBlockHistory` — Anti-Detection Telemetry

| Field | Type | Description |
|---|---|---|
| `port` | `IntegerField` | Outbound port that was blocked |
| `domain` | `CharField` | Target domain (e.g., `zillow.com`) |
| `timestamp` | `DateTimeField` | Auto-updated on each event |

### `agent_details` — Realtor.ca Agent Profiles (cached)

Stores structured agent profile data: name, address, phone, profile image, corporation, social URLs (Facebook, LinkedIn), organization details.

---

## 🧠 Key Engineering Decisions

### Why Celery over ThreadPoolExecutor?

Celery provides **process isolation** (each task runs in a separate worker process), **fault tolerance** (late acknowledgement prevents task loss on crash), **distributed scalability** (workers can run across multiple machines), and **a mature retry/backoff ecosystem** — none of which are available with threading primitives alone.

### Why Redis as both Broker and Cache?

A single Redis instance handles both Celery's message queue and Django's response cache. This reduces infrastructure cost while keeping the architecture simple. In production, TLS-encrypted Redis Cluster is used (`rediss://`) with connection pooling (`max_connections=100`) and socket timeout protection.

### Why Multi-Tier Anti-Bot Fallback?

Real estate platforms employ layered bot detection (TLS fingerprinting, behavioral analysis, CAPTCHA, IP reputation). A single proxy service introduces a single point of failure. The three-tier fallback (ZenRows → Zyte → headless Chrome) maximizes data extraction success rates while keeping operational costs proportional to the difficulty of the target page.

### Why Push vs. Poll for Results?

Phase 3 introduced **proactive result delivery** via webhook POST immediately upon task completion. This eliminates client-side polling overhead and reduces latency-to-result by removing the need for a separate status-check request cycle.

---

## 📁 Repository Structure

```
realhub-data-intelligence-platform/
│
├── realhub_scraper_api/         # Django project settings, Celery config, URL routing
│   ├── celery.py                # Celery app init, worker config (concurrency=20)
│   ├── settings.py              # Env-driven config (Redis, AWS, proxy, JWT)
│   └── urls.py                  # Global URL routing
│
├── zillow/                      # Zillow scraper app
│   ├── tasks.py                 # Celery tasks: property details, images, agent listings
│   ├── views.py                 # DRF API views (task dispatch / instant response)
│   └── utils.py                 # Extraction helpers, SSR JSON parsing
│
├── realtor/                     # Realtor.ca scraper app
│   ├── tasks.py                 # Agent details, live listings (Realtor.ca REST API)
│   ├── views.py                 # DRF API views
│   └── utils.py                 # Cookie management, payload builders
│
├── realtor_com/                 # Realtor.com scraper app
│   ├── tasks.py                 # Agent details, XHR network capture for listings
│   └── views.py                 # DRF API views
│
├── google_search_manager/       # Cross-platform address resolution
│   └── tasks.py                 # Google search → property URL extraction
│
├── utils_manager/               # Shared infrastructure utilities
│   ├── browsers.py              # Chrome/Selenium/DrissionPage/Playwright launchers
│   ├── zenrows.py               # ZenRows API integration (3 modes)
│   ├── zyte.py                  # Zyte API integration (4 modes)
│   ├── http_request.py          # Session-based HTTP with proxy auth
│   ├── s3_bucket.py             # AWS S3 stream upload (boto3)
│   ├── push_api.py              # Client push webhook delivery
│   ├── proxy.py                 # Proxy pool selection logic
│   └── proxy_auth_plugin/       # Chrome extension for proxy authentication
│
├── task_log/                    # Task audit & admin UI
│   ├── models.py                # CompletedTask, PortBlockHistory
│   ├── signals.py               # @task_postrun / @task_failure auto-logging
│   └── views.py                 # Filterable task log UI (time, task ID, custom range)
│
├── monitoring/                  # Health check endpoints
│   ├── views.py                 # /health/ and /celery-health/ views
│   └── urls.py                  # Endpoint routing
│
├── .github/workflows/
│   └── deploy.yml               # GitHub Actions CI/CD (SSH deploy + service restart)
│
├── requirements.txt             # Python dependencies
└── manage.py                    # Django management entry point
```

---

## 🔬 Scraper Engine Deep Dive

### Zillow — SSR JSON Extraction

Zillow's property pages are server-side rendered via Next.js. All property data is embedded in a `<script id="__NEXT_DATA__">` tag as a deeply nested JSON blob. The extraction pipeline:

1. Fetch page HTML (via proxy API or headless browser)
2. Parse `__NEXT_DATA__` JSON from `<script>` tag
3. Navigate to `props.pageProps.componentProps.gdpClientCache`
4. Extract the first key's `property` object
5. Normalize fields (address, price, tax history, agent contacts, images, geolocation)

### Realtor.ca — Internal REST API

Realtor.ca exposes an internal REST API (`api2.realtor.ca/Listing.svc/PropertySearch_Post`) used by its own frontend. The scraper:

1. Harvests a valid browser session cookie via headless Chrome
2. Reuses that cookie to authenticate paginated API calls
3. Submits structured `x-www-form-urlencoded` POST payloads including `IndividualId`, `RecordsPerPage=100`, currency, and pagination offset
4. Handles full pagination (`while(1)` loop) until all listings are collected

### Realtor.com — XHR Network Capture

Realtor.com's agent listing data is loaded via an AJAX call to `/realestateagents/api/v3/listings`. The scraper uses ZenRows/Zyte network capture mode to intercept this XHR response directly rather than parsing the final DOM — extracting the raw JSON API response.

### Google Search — Address Resolution

When a property URL is unknown, the platform queries Google Search with a targeted operator:
```
{address} (site:realtor.ca OR site:realtor.com OR site:zillow.com) property
```
Results are parsed via ZenRows (to bypass bot detection) and filtered by URL pattern matching for each platform.

---

## 📈 Performance Characteristics

| Metric | Value |
|---|---|
| Worker Concurrency | 20 simultaneous scrape tasks |
| Redis Connection Pool | 100 max connections |
| S3 Retry Policy | 5 attempts (standard mode) |
| JWT Token Lifetime | 60 days access / 30 days refresh |
| Celery Retry Backoff | 5s base (exponential), max 3 retries |
| Cache TTL (profiles) | Indefinite (manually invalidated) |
| Celery Health Timeout | 10 seconds (ping/pong) |

---

## 🔗 Related

This repository showcases the complete architectural evolution of the RealHub platform across three engineering phases. Each phase is independently deployable and represents a distinct solution to real-world data engineering constraints.

---

<div align="center">

**Built with Django · Celery · Redis · ZenRows · Zyte · AWS S3 · GitHub Actions**

*Engineered for production reliability at scale.*

</div>
