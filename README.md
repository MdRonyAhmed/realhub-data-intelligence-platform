# RealHub Data Intelligence Platform

<div align="center">

![Python](https://img.shields.io/badge/Python-3.11+-3776AB?style=for-the-badge&logo=python&logoColor=white)
![Django](https://img.shields.io/badge/Django-5.x-092E20?style=for-the-badge&logo=django&logoColor=white)
![Celery](https://img.shields.io/badge/Celery-5.x-37814A?style=for-the-badge&logo=celery&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-7.x-DC382D?style=for-the-badge&logo=redis&logoColor=white)
![AWS S3](https://img.shields.io/badge/AWS_S3-Cloud_Storage-FF9900?style=for-the-badge&logo=amazons3&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-CI%2FCD-2088FF?style=for-the-badge&logo=githubactions&logoColor=white)

</div>

> **NDA Notice:** The production source code for this platform is protected under a client Non-Disclosure Agreement. This repository is a **technical case study and architectural blueprint** — it documents system design, infrastructure patterns, and engineering decisions without disclosing proprietary business logic or credentials.

---

## What Is This?

A **production-grade, distributed real estate data intelligence platform** built to reliably extract structured data from heavily bot-protected North American markets — Zillow, Realtor.ca, and Realtor.com. The system evolved across **three engineering phases**, solving compounding challenges in async scalability, real-time response delivery, anti-bot evasion, and cloud data pipeline reliability.

👉 Read the full **[System Architecture & Evolution Guide](./ARCHITECTURE.md)**

---

## Core Engineering Highlights

### ⚡ Async Task Queue (Phase 1)
Celery 5 + Redis broker with **20 concurrent workers**, late-acknowledgement fault tolerance, and automatic task lifecycle persistence via Django ORM signals (`@task_postrun`, `@task_failure`). Designed to handle 10–60 second scraping jobs without HTTP timeout failures.

### 🔄 Synchronous Instant Execution (Phase 2)
`apply_async().get()` blocking pattern for consumers requiring immediate responses. Introduced API-level rate limiting (`django-ratelimit`) and first-generation AWS S3 cloud storage sink.

### 🛡️ Anti-Bot Evasion Stack (Phase 3)
Four-tier fallback chain engineered against TLS fingerprinting, behavioral analysis, and IP-reputation blocks:

```
Tier 1 → Direct HTTP + session spoofing
Tier 2 → ZenRows (premium residential proxies, JS rendering, XHR capture)
Tier 3 → Zyte API (browser HTML, network interception, session POST)
Tier 4 → Undetected ChromeDriver / DrissionPage (Chrome extension proxy auth)
```

### ☁️ Cloud-Native Data Sinks
- **AWS S3** — `boto3` stream upload with connection pooling (`max_pool_connections=100`) and retry policy (5 attempts)
- **Redis Cache** — Indefinite-TTL memoization for stable agent/property profiles
- **Push Webhooks** — Proactive `multipart/form-data` result delivery to prod + dev environments on task completion

### 🚀 CI/CD Automation
GitHub Actions SSH pipeline — on every push to `main`, automatically pulls latest code, restarts Gunicorn, Celery, and Nginx via `systemctl` with zero manual intervention.

---

## Target Data Surfaces

| Platform | Extracted Data |
|---|---|
| **Zillow** | Property details, agent profiles, active listings, image galleries |
| **Realtor.ca** | Agent profiles, live listing inventories, paginated property search |
| **Realtor.com** | Agent details, active listings (XHR network interception) |
| **Google Search** | Address → property URL resolution across all three platforms |

---

## Tech Stack at a Glance

`Django 5` · `Celery 5` · `Redis 7` · `Django REST Framework` · `SimpleJWT` · `ZenRows` · `Zyte API` · `Selenium` · `DrissionPage` · `Playwright` · `Undetected ChromeDriver` · `BeautifulSoup4` · `AWS S3 (boto3)` · `GitHub Actions` · `Gunicorn` · `Nginx`

---

👉 **[View Full Architecture, API Docs, Data Models & Scraper Engine Breakdown →](./ARCHITECTURE.md)**
