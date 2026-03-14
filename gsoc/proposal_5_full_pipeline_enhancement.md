# GSoC 2026 Proposal: Full-Pipeline Enhancement — Smarter Discovery, Scheduled Scans, Analytics Dashboard, and Real-Time Alerts for Artemis

## Contact Information

- **Name:** [Your Full Name]
- **Email:** [your.email@example.com]
- **GitHub:** [github.com/yourusername]
- **Timezone:** [e.g., UTC+2]
- **University / Program:** [Your University, Year, Major]

---

## Project Overview

### Title
Full-Pipeline Enhancement: Smarter Discovery, Scheduled Scans, Analytics & Alerts

### Synopsis
Artemis is a powerful vulnerability scanner built by CERT Polska, but its current pipeline has blind spots at every stage — **before**, **during**, and **after** scanning. Before scanning, endpoints go undiscovered because crawling is shallow. During scanning, scans must be triggered manually with no scheduling. After scanning, results sit silently in a database with no visual analytics and no real-time alerts. This project enhances every stage of the Artemis pipeline: a structured **attack surface expansion** phase (deep crawling, JS enumeration, deduplication) to find more targets, **scheduled and recurring scans** via UI and API, an **interactive analytics dashboard** with Chart.js for trend visibility, **3 new vulnerability detection modules**, and a **notification system** for real-time alerts to Email, Discord, and webhooks.

### The Full Picture — What's Missing Today

```
┌─────────────────────────────────────────────────────────────────────────┐
│              CURRENT ARTEMIS PIPELINE — GAPS AT EVERY STAGE             │
│                                                                         │
│  BEFORE SCAN            DURING SCAN              AFTER SCAN             │
│  ────────────           ──────────────            ──────────             │
│  ❌ Shallow discovery   ❌ Manual-only triggers   ❌ No visual trends    │
│  ❌ No deep crawling    ❌ No recurring scans     ❌ No real-time alerts │
│  ❌ No JS enumeration   ❌ No scheduling UI       ❌ No dashboards       │
│  ❌ Missed endpoints    ❌ No cron-like API       ❌ Static tables only  │
│                                                                         │
│  ┌──────────┐    ┌──────────────────┐    ┌────────────────────────┐    │
│  │ Submit   │───►│ Classify + Scan  │───►│ Store in DB            │    │
│  │ target   │    │ (whatever we     │    │ (hope someone checks)  │    │
│  │ manually │    │  already found)  │    │                        │    │
│  └──────────┘    └──────────────────┘    └────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘

                              │
                              ▼

┌─────────────────────────────────────────────────────────────────────────┐
│           PROPOSED ENHANCED PIPELINE — GAPS FILLED                       │
│                                                                         │
│  BEFORE SCAN            DURING SCAN              AFTER SCAN             │
│  ────────────           ──────────────            ──────────             │
│  ✅ Deep web crawling   ✅ Scheduled scans (UI)  ✅ Chart.js dashboard  │
│  ✅ JS file parsing     ✅ Recurring cron scans  ✅ Real-time alerts    │
│  ✅ Endpoint dedup      ✅ API scheduling        ✅ Email + Discord     │
│  ✅ Structured staging  ✅ 3 new vuln modules    ✅ Webhook integration │
│                                                                         │
│  ┌────────┐  ┌────────────┐  ┌───────────┐  ┌──────────┐  ┌────────┐ │
│  │Discover│─►│ Deduplicate│─►│ Scan with │─►│Dashboard │─►│ Alert  │ │
│  │& Crawl │  │ & Filter   │  │ scheduled │  │& Trends  │  │via     │ │
│  │deeply  │  │ targets    │  │ modules   │  │(Chart.js)│  │Discord,│ │
│  └────────┘  └────────────┘  └───────────┘  └──────────┘  │Email   │ │
│                                                            └────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

### Motivation

Artemis currently works as a "submit targets → run modules → store results" tool. By enhancing all five stages of the pipeline, it becomes a **continuous, self-improving scanning platform**:

1. **Better Discovery:** The current `crawling.py` only extracts `src` and `href` from a single page. This misses endpoints hidden in JavaScript files, multi-level page links, form actions, and API routes — leading to undiscovered vulnerabilities (as seen with `testphp.vulnweb.com` where SQLi endpoints weren't found because the endpoints themselves were never discovered).

2. **Scheduled Scans:** There is no way to set up recurring scans. CSIRT teams managing hundreds of organizations need weekly or daily automated re-scans, not manual triggers.

3. **Visual Analytics:** The current UI (`templates/index.jinja2`) is a paginated table. At scale (100K+ domains), teams need trend charts, severity breakdowns, and module performance visibility.

4. **More Modules:** Expanding detection coverage with new vulnerability modules directly increases Artemis's value.

5. **Real-Time Alerts:** Findings sit in the database until someone checks. Critical vulnerabilities should trigger immediate notifications to team channels.

---

## Detailed Description

### Enhanced Pipeline Architecture

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                   ENHANCED ARTEMIS PIPELINE ARCHITECTURE                     │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │ PHASE 1: TARGET SUBMISSION & SCHEDULING                                 │ │
│  │                                                                         │ │
│  │  ┌──────────┐   ┌──────────────┐   ┌─────────────────────────────────┐ │ │
│  │  │ Manual   │   │ API          │   │ Scheduler (NEW)                 │ │ │
│  │  │ Submit   │   │ POST /add    │   │                                 │ │ │
│  │  │ (UI)     │   │              │   │ Cron-like recurring scans       │ │ │
│  │  │          │   │ POST /api/   │   │ "Rescan client_a every Monday"  │ │ │
│  │  │          │   │ schedule     │   │ "Daily scan of *.gov.pl"        │ │ │
│  │  └────┬─────┘   └──────┬──────┘   └──────────────┬──────────────────┘ │ │
│  │       │                │                          │                    │ │
│  │       └────────────────┴──────────────────────────┘                    │ │
│  │                        │                                               │ │
│  │                        ▼                                               │ │
│  │              ┌──────────────────┐                                      │ │
│  │              │   create_tasks() │                                      │ │
│  │              │   (producer.py)  │                                      │ │
│  │              └────────┬─────────┘                                      │ │
│  └───────────────────────┼─────────────────────────────────────────────────┘ │
│                          │                                                   │
│  ┌───────────────────────┼─────────────────────────────────────────────────┐ │
│  │ PHASE 2: CLASSIFICATION & DISCOVERY (ENHANCED)                          │ │
│  │                          │                                              │ │
│  │              ┌───────────▼──────────┐                                   │ │
│  │              │    Classifier        │                                   │ │
│  │              │    (existing)        │                                   │ │
│  │              └──┬─────────────┬─────┘                                   │ │
│  │                 │             │                                          │ │
│  │            DOMAIN task    SERVICE+HTTP task                              │ │
│  │                 │             │                                          │ │
│  │                 │             ▼                                          │ │
│  │                 │   ┌──────────────────────┐                            │ │
│  │                 │   │  Deep Web Crawler     │ ◄── NEW                   │ │
│  │                 │   │  (multi-level crawl)  │                           │ │
│  │                 │   └──────────┬───────────┘                            │ │
│  │                 │              │                                         │ │
│  │                 │              ▼                                         │ │
│  │                 │   ┌──────────────────────┐                            │ │
│  │                 │   │ JS Endpoint Extractor │ ◄── NEW                   │ │
│  │                 │   │ (parse .js for routes)│                           │ │
│  │                 │   └──────────┬───────────┘                            │ │
│  │                 │              │                                         │ │
│  │                 │              ▼                                         │ │
│  │                 │   ┌──────────────────────┐                            │ │
│  │                 │   │  Target Deduplicator  │ ◄── NEW                   │ │
│  │                 │   │  & Filter             │                           │ │
│  │                 │   └──────────┬───────────┘                            │ │
│  │                 │              │                                         │ │
│  │                 │     Deduplicated URL tasks                             │ │
│  └─────────────────┼──────────────┼────────────────────────────────────────┘ │
│                    │              │                                           │
│  ┌─────────────────┼──────────────┼────────────────────────────────────────┐ │
│  │ PHASE 3: VULNERABILITY SCANNING                                         │ │
│  │                 │              │                                         │ │
│  │    ┌────────────┴──────────────┴────────────────────────┐               │ │
│  │    │        Existing Modules          New Modules (3)    │               │ │
│  │    │  ┌─────┐ ┌──────┐ ┌──────┐  ┌──────┐ ┌─────────┐ │               │ │
│  │    │  │ vcs │ │nuclei│ │bruter│  │Mod A │ │ Mod B   │ │               │ │
│  │    │  └─────┘ └──────┘ └──────┘  └──────┘ └─────────┘ │               │ │
│  │    │  ┌─────┐ ┌──────┐ ┌──────┐  ┌──────┐             │               │ │
│  │    │  │ sql │ │ lfi  │ │humble│  │Mod C │             │               │ │
│  │    │  └─────┘ └──────┘ └──────┘  └──────┘             │               │ │
│  │    └────────────────────────────────────────────────────┘               │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│                          │                                                   │
│  ┌───────────────────────┼─────────────────────────────────────────────────┐ │
│  │ PHASE 4: RESULTS & VISIBILITY                                           │ │
│  │                       │                                                 │ │
│  │              ┌────────▼────────┐                                        │ │
│  │              │   PostgreSQL    │                                        │ │
│  │              │   TaskResult    │                                        │ │
│  │              └──┬──────────┬───┘                                        │ │
│  │                 │          │                                             │ │
│  │                 ▼          ▼                                             │ │
│  │  ┌──────────────────┐  ┌───────────────────────┐                       │ │
│  │  │ Analytics        │  │ Notification Engine   │ ◄── NEW               │ │
│  │  │ Dashboard (NEW)  │  │ (Email, Discord,      │                       │ │
│  │  │                  │  │  Webhooks)            │                       │ │
│  │  │ Chart.js trends  │  │                       │                       │ │
│  │  │ Severity charts  │  │ Rule-based alerting   │                       │ │
│  │  │ Module stats     │  │ for INTERESTING finds  │                       │ │
│  │  └──────────────────┘  └───────────────────────┘                       │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## Work Package 1: Attack Surface Expansion — Structured Discovery Phase

### Problem: Current Discovery is Shallow

```
┌─────────────────────────────────────────────────────────────────────────┐
│  CURRENT DISCOVERY FLOW — MISSES ENDPOINTS                              │
│                                                                         │
│  target.com ──► port_scanner ──► SERVICE:HTTP                          │
│                                      │                                  │
│                     ┌────────────────┼────────────────────┐             │
│                     │                │                    │             │
│                     ▼                ▼                    ▼             │
│                  robots.py     bruter.py          sql_injection.py      │
│                  (robots.txt   (wordlist of       (crawling.py:         │
│                   paths only)  known files)        single-page          │
│                                                    link extraction)     │
│                                                         │               │
│                                              Only finds links from      │
│                                              ONE page (href, src).      │
│                                              ❌ No recursive crawl      │
│                                              ❌ No JS file parsing      │
│                                              ❌ No form action extract  │
│                                              ❌ Endpoints are MISSED    │
│                                                                         │
│  Example: testphp.vulnweb.com                                           │
│  ├─ /login.php     ← found (direct link)                               │
│  ├─ /artists.php   ← found (direct link)                               │
│  ├─ /listproducts.php?cat=1  ← ❌ MISSED (deeper link)                │
│  ├─ /search.php?q=           ← ❌ MISSED (form action)                 │
│  └─ /api/v1/users            ← ❌ MISSED (in JavaScript file)          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Proposed: Structured 3-Stage Discovery

```
┌─────────────────────────────────────────────────────────────────────────┐
│              ENHANCED DISCOVERY PIPELINE (3 STAGES)                      │
│                                                                         │
│  SERVICE:HTTP target                                                    │
│       │                                                                 │
│       ▼                                                                 │
│  ╔═══════════════════════════════════════════════════════════════════╗  │
│  ║  STAGE 1: Deep Web Crawler                                        ║  │
│  ║  File: artemis/modules/deep_crawler.py                            ║  │
│  ║  Binds: TaskType.SERVICE + Service.HTTP                           ║  │
│  ║                                                                   ║  │
│  ║  ┌─────────────┐     ┌──────────────┐     ┌────────────────┐     ║  │
│  ║  │ Fetch page  │────►│ Extract:     │────►│ Follow links   │     ║  │
│  ║  │ content     │     │ • <a href>   │     │ up to depth N  │     ║  │
│  ║  │             │     │ • <form act> │     │ (configurable, │     ║  │
│  ║  │             │     │ • <script>   │     │  default: 3)   │     ║  │
│  ║  │             │     │ • <link>     │     │                │     ║  │
│  ║  │             │     │ • <iframe>   │     │ Stay on same   │     ║  │
│  ║  │             │     │ • JS imports │     │ domain (scope) │     ║  │
│  ║  └─────────────┘     └──────────────┘     └───────┬────────┘     ║  │
│  ║                                                   │              ║  │
│  ║  Output: discovered_urls[], discovered_js_files[]  │              ║  │
│  ╚═══════════════════════════════════════════════════════╝              │
│                                                   │                    │
│                                                   ▼                    │
│  ╔═══════════════════════════════════════════════════════════════════╗  │
│  ║  STAGE 2: JavaScript Endpoint Extractor                           ║  │
│  ║  File: artemis/modules/js_endpoint_extractor.py                   ║  │
│  ║  Binds: TaskType.URL (filters .js files from crawler)             ║  │
│  ║                                                                   ║  │
│  ║  For each .js file discovered:                                    ║  │
│  ║  ┌──────────────────────────────────────────────────────────┐     ║  │
│  ║  │ Regex patterns to extract:                                │     ║  │
│  ║  │                                                          │     ║  │
│  ║  │ • API routes:   /api/v[0-9]+/\w+                         │     ║  │
│  ║  │ • Fetch calls:  fetch\(["'](.*?)["']\)                   │     ║  │
│  ║  │ • Axios calls:  axios\.(get|post)\(["'](.*?)["']\)       │     ║  │
│  ║  │ • XHR:          \.open\(["'].*?["'],\s*["'](.*?)["']\)   │     ║  │
│  ║  │ • Path strings: ["'](/[a-zA-Z0-9_/.-]+)["']              │     ║  │
│  ║  │ • Form actions: action=["'](.*?)["']                      │     ║  │
│  ║  │                                                          │     ║  │
│  ║  │ Filter out:                                               │     ║  │
│  ║  │ • Static assets (.png, .css, .svg, .woff, etc.)           │     ║  │
│  ║  │ • CDN URLs (different domain)                             │     ║  │
│  ║  │ • Known library paths (/node_modules, /vendor)            │     ║  │
│  ║  └──────────────────────────────────────────────────────────┘     ║  │
│  ║                                                                   ║  │
│  ║  Output: extracted_endpoints[]                                    ║  │
│  ╚═══════════════════════════════════════════════════════════════════╝  │
│                                                   │                    │
│                                                   ▼                    │
│  ╔═══════════════════════════════════════════════════════════════════╗  │
│  ║  STAGE 3: Target Deduplicator & Filter                            ║  │
│  ║  File: artemis/modules/target_deduplicator.py                     ║  │
│  ║                                                                   ║  │
│  ║  Input: All URLs from crawler + JS extractor + robots + bruter    ║  │
│  ║                                                                   ║  │
│  ║  ┌────────────────────────────────────────────────────────┐       ║  │
│  ║  │ Step 1: Normalize URLs                                  │       ║  │
│  ║  │  • Strip trailing slashes, fragments, tracking params   │       ║  │
│  ║  │  • Lowercase scheme and host                            │       ║  │
│  ║  │  • Sort query parameters                                │       ║  │
│  ║  │                                                        │       ║  │
│  ║  │ Step 2: Deduplicate                                     │       ║  │
│  ║  │  • Remove exact duplicates                              │       ║  │
│  ║  │  • Group URLs with same path but different params       │       ║  │
│  ║  │    (keep one representative per path pattern)            │       ║  │
│  ║  │                                                        │       ║  │
│  ║  │ Step 3: Filter                                          │       ║  │
│  ║  │  • Remove static assets (.css, .png, .js, .svg, etc.)  │       ║  │
│  ║  │  • Remove logout/signout URLs (safety)                  │       ║  │
│  ║  │  • Apply scope rules (same domain only)                 │       ║  │
│  ║  │                                                        │       ║  │
│  ║  │ Step 4: Emit as URL tasks for vulnerability modules     │       ║  │
│  ║  └────────────────────────────────────────────────────────┘       ║  │
│  ║                                                                   ║  │
│  ║  Output: Deduplicated, filtered URL tasks → vuln modules          ║  │
│  ╚═══════════════════════════════════════════════════════════════════╝  │
└─────────────────────────────────────────────────────────────────────────┘
```

### Discovery Impact Example

```
┌─────────────────────────────────────────────────────────────────────────┐
│  EXAMPLE: testphp.vulnweb.com                                           │
│                                                                         │
│  Current Discovery (crawling.py):         Enhanced Discovery:           │
│  ─────────────────────────────            ──────────────────             │
│  /index.php                               /index.php                    │
│  /login.php                               /login.php                    │
│  /artists.php                             /artists.php                  │
│  /categories.php                          /categories.php               │
│                                           /listproducts.php?cat=1  ◄NEW │
│                                           /search.php              ◄NEW │
│                                           /comment.php             ◄NEW │
│                                           /userinfo.php            ◄NEW │
│                                           /cart.php                ◄NEW │
│                                           /api/v1/products         ◄NEW │
│                                           /secured/newuser.php     ◄NEW │
│                                                                         │
│  URLs found: 4                            URLs found: 11               │
│  SQLi detected: 0                         SQLi detected: 3+            │
│                                                                         │
│  Coverage increase: ~175% more endpoints discovered                     │
└─────────────────────────────────────────────────────────────────────────┘
```

### Design: Module Structure & Key Logic

```python
# artemis/modules/deep_crawler.py — Module skeleton

@load_risk_class.load_risk_class(load_risk_class.LoadRiskClass.LOW)
class DeepCrawler(ArtemisBase):
    identity = "deep_crawler"
    filters = [{"type": TaskType.SERVICE.value, "service": Service.HTTP.value}]

    def run(self, current_task: Task) -> None:
        base_url = get_target_url(current_task)
        # BFS crawl up to CRAWLER_MAX_DEPTH levels, CRAWLER_MAX_PAGES pages
        # For each page: extract <a href>, <form action>, <script src>, <iframe>
        # Stay on same domain, skip static assets (.css, .png, .svg, ...)
        # Emit URL tasks for discovered pages → vulnerability modules scan them
        # Emit URL tasks for .js files    → js_endpoint_extractor parses them
        self.db.save_task_result(task=current_task, status=TaskStatus.OK, ...)
```

```python
# artemis/modules/js_endpoint_extractor.py — Regex patterns for JS parsing

JS_ENDPOINT_PATTERNS = [
    re.compile(r"""fetch\(\s*["'](\/[^"']+)["']"""),                    # fetch("/api/...")
    re.compile(r"""axios\.(?:get|post|put|delete)\(["'](\/[^"']+)"""),   # axios.get("/...")
    re.compile(r"""\.open\(["'][A-Z]+["'],\s*["'](\/[^"']+)["']"""),     # XHR .open("GET","/...")
    re.compile(r"""["'](\/api\/[a-zA-Z0-9_/.\-]+)["']"""),              # "/api/v1/users"
    re.compile(r"""path\s*:\s*["'](\/[^"']+)["']"""),                    # route path: "/dashboard"
]

# Filter out: .css, .png, .svg, .woff, /node_modules/, /vendor/
# Resolve relative paths → absolute URLs using base domain
# Emit each discovered endpoint as a URL task
```

```python
# artemis/modules/target_deduplicator.py — URL normalization

class TargetDeduplicator(ArtemisBase):
    identity = "target_deduplicator"

    @staticmethod
    def normalize_url(url: str) -> str:
        # 1. Lowercase scheme + host
        # 2. Strip trailing slashes, fragments
        # 3. Remove tracking params (utm_*, fbclid)
        # 4. Sort remaining query parameters alphabetically
        ...

    @staticmethod
    def get_path_pattern(url: str) -> str:
        # /products/123 → /products/{id}  (group numeric path segments)
        # Ensures /products/123 and /products/456 deduplicate to one representative
        ...

    # Also skip logout/signout URLs for safety
    # Emit deduplicated URL tasks → vulnerability scanning modules
```

### Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `CRAWLER_MAX_DEPTH` | `3` | Maximum crawl depth from initial page |
| `CRAWLER_MAX_PAGES` | `100` | Maximum pages to crawl per target |
| `CRAWLER_SAME_DOMAIN_ONLY` | `true` | Restrict crawling to same domain |
| `CRAWLER_EXTRACT_JS_ENDPOINTS` | `true` | Enable JS endpoint extraction |
| `CRAWLER_TIMEOUT_SECONDS` | `30` | Per-request timeout |

---

## Work Package 2: Scan Scheduling System (UI + API)

### Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     SCHEDULING SYSTEM ARCHITECTURE                      │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                        PostgreSQL                                │   │
│  │                                                                  │   │
│  │  ┌──────────────────────────────────┐                           │   │
│  │  │ ScanSchedule                     │                           │   │
│  │  ├──────────────────────────────────┤                           │   │
│  │  │ id (PK)                          │                           │   │
│  │  │ name                             │                           │   │
│  │  │ targets (JSONB)  [list of URIs]  │                           │   │
│  │  │ tag                              │                           │   │
│  │  │ cron_expression  "0 2 * * MON"   │                           │   │
│  │  │ enabled          true/false      │                           │   │
│  │  │ disabled_modules (JSONB)         │                           │   │
│  │  │ priority                         │                           │   │
│  │  │ created_at                       │                           │   │
│  │  │ last_run_at                      │                           │   │
│  │  │ next_run_at                      │                           │   │
│  │  │ run_count                        │                           │   │
│  │  └──────────────────────────────────┘                           │   │
│  │                                                                  │   │
│  │  ┌──────────────────────────────────┐                           │   │
│  │  │ ScanScheduleLog                  │                           │   │
│  │  ├──────────────────────────────────┤                           │   │
│  │  │ id (PK)                          │                           │   │
│  │  │ schedule_id (FK)                 │                           │   │
│  │  │ started_at                       │                           │   │
│  │  │ status (running/completed/error) │                           │   │
│  │  │ targets_submitted                │                           │   │
│  │  │ task_ids (JSONB)                 │                           │   │
│  │  └──────────────────────────────────┘                           │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────┐   ┌──────────────────────────────┐    │
│  │  Scheduler Service (NEW)    │   │  FastAPI Endpoints           │    │
│  │  (Docker container)         │   │                              │    │
│  │                             │   │  POST /api/schedules         │    │
│  │  Runs as a Karton module    │   │  GET  /api/schedules         │    │
│  │  or standalone process      │   │  GET  /api/schedules/{id}    │    │
│  │                             │   │  PUT  /api/schedules/{id}    │    │
│  │  Every 60s:                 │   │  DELETE /api/schedules/{id}  │    │
│  │  1. Query ScanSchedule     │   │  POST /api/schedules/{id}/   │    │
│  │     where enabled=true      │   │       trigger               │    │
│  │     and next_run_at <= now  │   │  GET  /api/schedules/{id}/  │    │
│  │  2. Call create_tasks()     │   │       history               │    │
│  │  3. Update last_run_at     │   │                              │    │
│  │  4. Calculate next_run_at  │   └──────────────────────────────┘    │
│  │     from cron_expression   │                                       │
│  │  5. Log to ScanScheduleLog │                                       │
│  └─────────────────────────────┘                                       │
└─────────────────────────────────────────────────────────────────────────┘
```

### Scheduling Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│  SCHEDULING FLOW — FROM CREATION TO EXECUTION                           │
│                                                                         │
│  Operator creates schedule           Scheduler service checks           │
│  (UI or API)                         every 60 seconds                   │
│       │                                   │                             │
│       ▼                                   ▼                             │
│  ┌──────────────┐                  ┌──────────────────┐                │
│  │ POST /api/   │                  │ SELECT * FROM    │                │
│  │ schedules    │─── stores ──────►│ scan_schedule    │                │
│  │              │                  │ WHERE enabled    │                │
│  │ name: "..."  │                  │ AND next_run_at  │                │
│  │ targets: []  │                  │     <= NOW()     │                │
│  │ cron: "..."  │                  └────────┬─────────┘                │
│  └──────────────┘                           │                          │
│                                    Schedule due?                        │
│                                         │                              │
│                                   YES   │    NO                        │
│                                         │    → sleep(60)               │
│                                         ▼                              │
│                               ┌──────────────────┐                     │
│                               │ create_tasks()   │                     │
│                               │ (producer.py)    │                     │
│                               │                  │                     │
│                               │ Uses existing    │                     │
│                               │ task submission  │                     │
│                               │ infrastructure   │                     │
│                               └────────┬─────────┘                     │
│                                        │                               │
│                                        ▼                               │
│                               ┌──────────────────┐                     │
│                               │ Update schedule: │                     │
│                               │ • last_run_at    │                     │
│                               │ • next_run_at    │                     │
│                               │ • run_count++    │                     │
│                               │ • Log execution  │                     │
│                               └──────────────────┘                     │
└─────────────────────────────────────────────────────────────────────────┘
```

### Scheduling UI Mockup

```
┌──────────────────────────────────────────────────────────────────────────┐
│  Artemis — Scan Schedules                                 [+ New Schedule]│
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌────┬──────────────────┬─────────────┬────────────┬────────┬────────┐ │
│  │ On │ Name             │ Schedule    │ Targets    │Last Run│Actions │ │
│  ├────┼──────────────────┼─────────────┼────────────┼────────┼────────┤ │
│  │ ✅ │ Weekly gov.pl    │ Mon 02:00   │ 342 targets│ 3d ago │ ✏️ ▶ 🗑│ │
│  │ ✅ │ Daily critical   │ Daily 06:00 │ 28 targets │ 18h ago│ ✏️ ▶ 🗑│ │
│  │ ❌ │ Monthly full     │ 1st of month│ 1,204 tgts │ 25d ago│ ✏️ ▶ 🗑│ │
│  └────┴──────────────────┴─────────────┴────────────┴────────┴────────┘ │
│                                                                          │
├──── Create / Edit Schedule ──────────────────────────────────────────────┤
│                                                                          │
│  Name:     [Weekly government scan           ]                           │
│                                                                          │
│  Targets:  [Paste targets, one per line, or select tag ▼]               │
│            ┌─────────────────────────────────────────┐                   │
│            │ example.gov.pl                           │                   │
│            │ portal.gov.pl                            │                   │
│            │ mail.gov.pl                              │                   │
│            └─────────────────────────────────────────┘                   │
│  OR Tag:   [gov_pl ▼]  (rescan all targets with this tag)              │
│                                                                          │
│  Schedule: [Preset ▼: Weekly/Daily/Monthly/Custom]                      │
│  Cron:     [0 2 * * MON      ]  (every Monday at 02:00)                │
│  Preview:  Next runs: Mon Mar 16 02:00, Mon Mar 23 02:00, ...           │
│                                                                          │
│  Priority: [Normal ▼]                                                   │
│  Disabled Modules: [☐ nuclei  ☐ bruter  ☐ shodan  ...]                 │
│                                                                          │
│  [Save Schedule]  [Run Now ▶]  [Cancel]                                 │
└──────────────────────────────────────────────────────────────────────────┘
```

### API Endpoints

| Endpoint | Method | Description | Body/Params |
|----------|--------|-------------|-------------|
| `/api/schedules` | POST | Create schedule | `{name, targets[], tag, cron, disabled_modules[]}` |
| `/api/schedules` | GET | List all schedules | — |
| `/api/schedules/{id}` | GET | Get schedule details | — |
| `/api/schedules/{id}` | PUT | Update schedule | Same as POST |
| `/api/schedules/{id}` | DELETE | Delete schedule | — |
| `/api/schedules/{id}/trigger` | POST | Run schedule now | — |
| `/api/schedules/{id}/history` | GET | Execution history | `?limit=10` |

### Design: Database Schema (SQLAlchemy)

```python
# artemis/db.py — New models (follows existing Analysis/TaskResult pattern)

class ScanSchedule(Base):
    __tablename__ = "scan_schedule"
    id              = Column(Integer, primary_key=True, autoincrement=True)
    name            = Column(String, nullable=False)
    targets         = Column(JSONB, nullable=False)       # ["example.com", "10.0.0.1"]
    tag             = Column(String, index=True)
    cron_expression = Column(String, nullable=False)       # "0 2 * * MON"
    enabled         = Column(Boolean, default=True, index=True)
    disabled_modules= Column(JSONB, default=[])
    created_at      = Column(DateTime, server_default=text("NOW()"))
    last_run_at     = Column(DateTime, nullable=True)
    next_run_at     = Column(DateTime, nullable=False)
    run_count       = Column(Integer, default=0)

class ScanScheduleLog(Base):
    __tablename__ = "scan_schedule_log"
    id                = Column(Integer, primary_key=True, autoincrement=True)
    schedule_id       = Column(Integer, nullable=False, index=True)
    started_at        = Column(DateTime, server_default=text("NOW()"))
    status            = Column(String, default="running")  # running | completed | error
    targets_submitted = Column(Integer)
    task_ids          = Column(JSONB, default=[])
    error_message     = Column(Text, nullable=True)
```

### Design: Scheduler Service Core Loop

```python
# artemis/scheduler.py — Runs as a standalone Docker container

def run_scheduler():
    # Every 60 seconds:
    #   SELECT * FROM scan_schedule WHERE enabled = true AND next_run_at <= NOW()
    for schedule in due_schedules:
        task_ids = create_tasks(           # Reuses existing producer.py
            uris=schedule.targets,
            tag=schedule.tag,
            disabled_modules=schedule.disabled_modules,
        )
        schedule.last_run_at = now
        schedule.next_run_at = croniter(schedule.cron_expression, now).get_next(datetime)
        schedule.run_count += 1
        # Log execution in ScanScheduleLog
```

### Design: Scheduling API Signatures

```python
# artemis/api_schedules.py — FastAPI router

@router.post("/api/schedules")        # Create schedule (validates cron via croniter)
@router.get("/api/schedules")         # List all schedules
@router.get("/api/schedules/{id}")    # Get schedule details + next run preview
@router.put("/api/schedules/{id}")    # Update schedule
@router.delete("/api/schedules/{id}") # Delete schedule
@router.post("/api/schedules/{id}/trigger")  # Run immediately (calls create_tasks())
@router.get("/api/schedules/{id}/history")   # Query ScanScheduleLog
```

```yaml
# docker-compose.yaml — New service entry
  scheduler:
    build: .
    command: python3 -m artemis.scheduler
    depends_on: [postgres, redis]
```

---

## Work Package 3: Analytics Dashboard (Chart.js)

### Dashboard Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      DASHBOARD DATA FLOW                                │
│                                                                         │
│  ┌──────────────┐     ┌──────────────────┐     ┌───────────────────┐   │
│  │  PostgreSQL  │     │  FastAPI          │     │  Browser          │   │
│  │              │     │                  │     │                   │   │
│  │  Aggregate   │────►│  /api/stats/     │────►│  Chart.js renders │   │
│  │  queries     │     │  findings-trend  │     │  into <canvas>    │   │
│  │  (GROUP BY,  │     │  severity-dist   │     │                   │   │
│  │   FILTER,    │     │  modules         │     │  Jinja2 provides  │   │
│  │   date_trunc)│     │  top-targets     │     │  page shell       │   │
│  └──────┬───────┘     └────────┬─────────┘     └───────────────────┘   │
│         │                      │                                        │
│         ▼                      ▼                                        │
│  ┌──────────────┐     Cached in Redis                                  │
│  │  Partial     │     (TTL: 5 min)                                     │
│  │  Index on    │                                                       │
│  │  INTERESTING │                                                       │
│  │  status      │                                                       │
│  └──────────────┘                                                       │
└─────────────────────────────────────────────────────────────────────────┘
```

### Dashboard Layout Mockup

```
┌──────────────────────────────────────────────────────────────────────────┐
│  Artemis Dashboard                          [Schedules] [Alerts] [Add+] │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────────────┐│
│  │  📊 1,234  │  │  🔍 5,678  │  │  🔴  234   │  │  ⏳  89 Pending   ││
│  │  Analyses  │  │  Findings  │  │  High Sev  │  │  🔄  3 Scheduled  ││
│  └────────────┘  └────────────┘  └────────────┘  └────────────────────┘│
│                                                                          │
│  ┌─ Findings Trend ──────────────────────────────── [1W][1M][3M][1Y] ─┐│
│  │  300│                                                               ││
│  │     │         ▄                                                     ││
│  │  200│    ▄   ██▄    ▄▄                                              ││
│  │     │   ██▄ ████▄  ████   ▄▄                                       ││
│  │  100│   ███ █████▄▄████▄ ████▄    ▄                                ││
│  │     │   ███ ████████████ █████▄  ██▄                               ││
│  │    0│───███─████████████─██████──████──                             ││
│  │     └──Jan──Feb──Mar──Apr──May──Jun──Jul──                         ││
│  │                                                                     ││
│  │  ███ High (234)    ███ Medium (1,456)    ███ Low (890)             ││
│  └─────────────────────────────────────────────────────────────────────┘│
│                                                                          │
│  ┌─ Severity Breakdown ───────────┐  ┌─ Top Active Modules ───────────┐│
│  │                                │  │                                ││
│  │       ╭────────╮               │  │  nuclei      ████████████  312 ││
│  │     ╭─┤ HIGH   ├──╮           │  │  bruter      █████████     245 ││
│  │     │ │  9%    │  │           │  │  vcs         ████████      198 ││
│  │     │ ╰────────╯  │           │  │  sql_inj     ███████       156 ││
│  │     ├──┤MEDIUM ├──┤           │  │  deep_crawl  █████         120 ││
│  │     │  │  56%  │  │           │  │  js_extract  ████           95 ││
│  │     ╰──┤ LOW   ├──╯           │  │                                ││
│  │        │  35%  │               │  │  [Show All Modules ►]         ││
│  │        ╰───────╯               │  │                                ││
│  └────────────────────────────────┘  └────────────────────────────────┘│
│                                                                          │
│  ┌─ Per-Organization Risk Table ──────────────── [Filter ▼] [Export] ─┐│
│  │                                                                     ││
│  │  Tag          │ Total │ 🔴High │ 🟡Med │ Trend   │ Last Scan       ││
│  │  ─────────────┼───────┼────────┼───────┼─────────┼─────────        ││
│  │  client_a     │  67   │   25   │  30   │ ↗ +5    │ 2h ago          ││
│  │  client_b     │  45   │   12   │  20   │ ↘ -2    │ 1d ago          ││
│  │  client_c     │  23   │    3   │   8   │ → 0     │ 3h ago          ││
│  │                                                                     ││
│  └─────────────────────────────────────────────────────────────────────┘│
│                                                                          │
│  ┌─ Upcoming Scheduled Scans ──────────────────────────────────────────┐│
│  │  🕐 Mon 02:00 — "Weekly gov.pl" (342 targets)         [Run Now ▶]  ││
│  │  🕐 Tue 06:00 — "Daily critical" (28 targets)         [Run Now ▶]  ││
│  └─────────────────────────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────────────────────┘
```

### Analytics API Endpoints

| Endpoint | Description | Response | Cache |
|----------|-------------|----------|-------|
| `GET /api/stats/findings-trend` | Findings over time | `[{date, high, med, low}]` | 5 min |
| `GET /api/stats/severity-distribution` | Severity pie data | `[{severity, count}]` | 5 min |
| `GET /api/stats/modules` | Module finding counts | `[{module, count}]` | 5 min |
| `GET /api/stats/tags-summary` | Per-tag breakdown | `[{tag, total, high, med}]` | 5 min |
| `GET /api/stats/top-targets` | Most vulnerable targets | `[{target, count}]` | 5 min |

### Design: Analytics SQL Queries

```sql
-- Findings trend (line chart data) — used by GET /api/stats/findings-trend
SELECT
    date_trunc('day', created_at)                               AS date,
    COUNT(*) FILTER (WHERE result->>'severity' = 'high')        AS high,
    COUNT(*) FILTER (WHERE result->>'severity' = 'medium')      AS medium,
    COUNT(*) FILTER (WHERE result->>'severity' = 'low')         AS low
FROM task_result
WHERE status = 'INTERESTING'
  AND created_at >= NOW() - INTERVAL '90 days'
GROUP BY date ORDER BY date;

-- Severity distribution (donut chart) — GET /api/stats/severity-distribution
SELECT COALESCE(result->>'severity', 'unknown') AS severity, COUNT(*) AS count
FROM task_result WHERE status = 'INTERESTING'
GROUP BY severity ORDER BY count DESC;

-- Module performance (bar chart) — GET /api/stats/modules
SELECT receiver AS module, COUNT(*) AS count
FROM task_result WHERE status = 'INTERESTING'
GROUP BY receiver ORDER BY count DESC;

-- Per-organization risk (table) — GET /api/stats/tags-summary
SELECT tag, COUNT(*) AS total,
    COUNT(*) FILTER (WHERE result->>'severity' = 'high')   AS high,
    COUNT(*) FILTER (WHERE result->>'severity' = 'medium') AS medium
FROM task_result
WHERE status = 'INTERESTING' AND tag IS NOT NULL
GROUP BY tag ORDER BY high DESC, total DESC;
```

All results cached in Redis (TTL: 5 min) to avoid repeated heavy aggregations.

### Design: Chart.js Integration Pattern

```javascript
// templates/dashboard.jinja2 — Chart.js renders API data into <canvas> elements

// Findings trend → Line chart (stacked area: high/medium/low)
async function loadTrend(days) {
    const data = await fetch(`/api/stats/findings-trend?days=${days}`).then(r => r.json());
    new Chart(canvas, {
        type: "line",
        data: {
            labels: data.map(d => d.date),
            datasets: [
                {label: "High",   data: data.map(d => d.high),   borderColor: "#dc3545"},
                {label: "Medium", data: data.map(d => d.medium), borderColor: "#ffc107"},
                {label: "Low",    data: data.map(d => d.low),    borderColor: "#198754"},
            ],
        },
    });
}

// Severity breakdown → Doughnut chart
// Module performance → Horizontal bar chart
// Time range buttons: [1W] [1M] [3M] call loadTrend(7/30/90)
```

---

## Work Package 4: New Vulnerability Modules (3 Modules)

### Module Overview

```
┌──────────────────────────────────────────────────────────────────────────┐
│              3 NEW VULNERABILITY DETECTION MODULES                       │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────────┐│
│  │ Module A: [Placeholder — e.g., CORS Misconfiguration Detector]      ││
│  │ File: artemis/modules/module_a.py                                   ││
│  │ Binds: TaskType.SERVICE + Service.HTTP                              ││
│  │ Load Risk: LOW                                                      ││
│  │ Severity: MEDIUM                                                    ││
│  │                                                                     ││
│  │ Detection: [Describe detection methodology]                         ││
│  │ Output: {finding_type, details, vulnerable_endpoints}               ││
│  └──────────────────────────────────────────────────────────────────────┘│
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────────┐│
│  │ Module B: [Placeholder — e.g., Subdomain Takeover Detector]         ││
│  │ File: artemis/modules/module_b.py                                   ││
│  │ Binds: TaskType.DOMAIN                                              ││
│  │ Load Risk: LOW                                                      ││
│  │ Severity: HIGH                                                      ││
│  │                                                                     ││
│  │ Detection: [Describe detection methodology]                         ││
│  │ Output: {vulnerable, cname_chain, service, fingerprint}             ││
│  └──────────────────────────────────────────────────────────────────────┘│
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────────┐│
│  │ Module C: [Placeholder — e.g., Exposed Backup/Config File Scanner]  ││
│  │ File: artemis/modules/module_c.py                                   ││
│  │ Binds: TaskType.SERVICE + Service.HTTP                              ││
│  │ Load Risk: MEDIUM                                                   ││
│  │ Severity: HIGH                                                      ││
│  │                                                                     ││
│  │ Detection: [Describe detection methodology]                         ││
│  │ Output: {exposed_files[], content_preview, contains_credentials}    ││
│  └──────────────────────────────────────────────────────────────────────┘│
│                                                                          │
│  Each module delivers:                                                   │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │  📁 artemis/modules/<name>.py              — Scanner logic       │    │
│  │  📁 artemis/reporting/modules/<name>/                            │    │
│  │     ├── reporter.py                        — TaskResult → Report │    │
│  │     └── template_<type>.jinja2             — Email template      │    │
│  │  📁 artemis/reporting/severity.py          — Severity mapping    │    │
│  │  📁 docker-compose.yaml                    — Container entry     │    │
│  │  📁 test/modules/test_<name>.py            — Unit tests          │    │
│  └──────────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────────┘
```

### Module Integration Into Pipeline

```
┌─────────────────────────────────────────────────────────────────────────┐
│  HOW NEW MODULES FIT INTO THE KARTON PIPELINE                           │
│                                                                         │
│  Classifier ──► DOMAIN tasks ──► Module B (domain-level scanning)      │
│      │                                                                  │
│      └─────► SERVICE:HTTP ──┬──► Existing: vcs, bruter, nuclei, ...    │
│                             ├──► Module A (HTTP-level scanning)         │
│                             ├──► Module C (HTTP-level scanning)         │
│                             │                                           │
│                             └──► deep_crawler ──► js_extractor          │
│                                       │                                 │
│                                       ▼                                 │
│                               URL tasks (enriched) ──► sql_injection   │
│                                                   ──► lfi_detector     │
│                                                   ──► ... (more URLs   │
│                                                         to test!)      │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Work Package 5: Notification & Alerting System

### Notification Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     NOTIFICATION SYSTEM                                  │
│                                                                         │
│  ┌──────────────────┐                                                   │
│  │ New TaskResult    │                                                   │
│  │ (INTERESTING)     │                                                   │
│  └────────┬─────────┘                                                   │
│           │                                                             │
│           ▼                                                             │
│  ┌────────────────────────────────────────┐                             │
│  │     NotificationProcessor              │                             │
│  │     (polls every 60s)                  │                             │
│  │                                        │                             │
│  │  1. Query new INTERESTING results      │                             │
│  │  2. For each enabled AlertRule:        │                             │
│  │     - Does tag match?                  │                             │
│  │     - Does severity meet threshold?    │                             │
│  │     - Already notified? (dedup)        │                             │
│  │  3. Dispatch to matching backends      │                             │
│  │  4. Log delivery in notification_log   │                             │
│  └────────┬───────────────────────────────┘                             │
│           │                                                             │
│     ┌─────┼─────────────────┐                                          │
│     │     │                 │                                          │
│     ▼     ▼                 ▼                                          │
│  ┌──────┐ ┌───────────┐ ┌──────────┐                                  │
│  │Email │ │  Discord   │ │ Webhook  │                                  │
│  │(SMTP)│ │  (Bot API  │ │ (custom  │                                  │
│  │      │ │   or       │ │  URL +   │                                  │
│  │      │ │   Webhook) │ │  JSON)   │                                  │
│  └──────┘ └───────────┘ └──────────┘                                  │
└─────────────────────────────────────────────────────────────────────────┘
```

### Alert Rule Processing

```
┌─────────────────────────────────────────────────────────────────────────┐
│  ALERT RULE EVALUATION FLOW                                             │
│                                                                         │
│  New INTERESTING TaskResult                                             │
│      │                                                                  │
│      ▼                                                                  │
│  ┌─────────────────────┐                                               │
│  │ Load enabled rules  │                                               │
│  └──────────┬──────────┘                                               │
│             │                                                           │
│        For each rule:                                                   │
│             │                                                           │
│  ┌──────────▼──────────┐     ┌──────────┐                              │
│  │ Tag pattern match?  │─NO─►│  Skip    │                              │
│  └──────────┬──────────┘     └──────────┘                              │
│             │ YES                                                       │
│  ┌──────────▼──────────┐     ┌──────────┐                              │
│  │ Severity >= min?    │─NO─►│  Skip    │                              │
│  └──────────┬──────────┘     └──────────┘                              │
│             │ YES                                                       │
│  ┌──────────▼──────────┐     ┌──────────┐                              │
│  │ Already notified?   │─YES►│  Skip    │                              │
│  │ (dedup check)       │     │ (dedup)  │                              │
│  └──────────┬──────────┘     └──────────┘                              │
│             │ NO                                                        │
│             ▼                                                           │
│  ┌──────────────────────┐                                              │
│  │  SEND NOTIFICATION   │                                              │
│  │  via configured      │                                              │
│  │  backend             │                                              │
│  └──────────┬───────────┘                                              │
│             │                                                           │
│  ┌──────────▼──────────┐                                               │
│  │ Log result          │                                               │
│  │ (sent / failed)     │                                               │
│  │ If failed: retry    │                                               │
│  │ (1m, 5m, 30m, 2h)   │                                               │
│  └─────────────────────┘                                               │
└─────────────────────────────────────────────────────────────────────────┘
```

### Notification Message Formats

**Discord Embed:**

```
┌─────────────────────────────────────────────────────────┐
│  🔴  HIGH SEVERITY — Artemis Finding                    │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━             │
│                                                         │
│  **Target:** example.com/.git/config                    │
│  **Type:** Exposed Version Control Folder               │
│  **Module:** vcs                                        │
│  **Tag:** client_production                             │
│  **Found:** 2026-03-14 09:23 UTC                        │
│                                                         │
│  Found exposed .git repository with potential           │
│  credentials in the remote URL.                         │
│                                                         │
│  🔗 View in Artemis                                     │
│  ─────────────────────────────────────────               │
│  Artemis Scanner • Today at 09:23                       │
└─────────────────────────────────────────────────────────┘
```

**Email Alert:**

```
┌─────────────────────────────────────────────────────────┐
│  Subject: [Artemis] 🔴 HIGH — Exposed .git on          │
│           example.com                                    │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  A new HIGH-severity finding was detected:              │
│                                                         │
│  ┌───────────┬──────────────────────────────────────┐   │
│  │ Target    │ https://example.com/.git/config      │   │
│  │ Type      │ Exposed Version Control Folder       │   │
│  │ Module    │ vcs                                  │   │
│  │ Severity  │ HIGH                                 │   │
│  │ Tag       │ client_production                    │   │
│  │ Time      │ 2026-03-14 09:23 UTC                 │   │
│  └───────────┴──────────────────────────────────────┘   │
│                                                         │
│  [View in Artemis →]                                    │
│                                                         │
│  ─────────────────────────────                          │
│  You received this because alert rule                   │
│  "Critical Alerts" matched. Manage your                 │
│  alert rules in Artemis Settings.                       │
└─────────────────────────────────────────────────────────┘
```

### Alert Rules UI Mockup

```
┌──────────────────────────────────────────────────────────────────────────┐
│  Notification Rules                                       [+ New Rule]   │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌────┬───────────────────┬──────────┬──────────┬────────┬─────────────┐│
│  │ On │ Rule Name         │ Channel  │ Min Sev  │ Tags   │ Actions     ││
│  ├────┼───────────────────┼──────────┼──────────┼────────┼─────────────┤│
│  │ ✅ │ Critical Discord  │ 💬Discord│ >= High  │ *      │ ✏️ 🧪 🗑   ││
│  │ ✅ │ Client A email    │ 📧 Email │ >= Med   │client_a│ ✏️ 🧪 🗑   ││
│  │ ✅ │ SIEM webhook      │ 🔗Webhook│ >= High  │ *      │ ✏️ 🧪 🗑   ││
│  └────┴───────────────────┴──────────┴──────────┴────────┴─────────────┘│
│                                                                          │
│  ┌─ Create Rule ────────────────────────────────────────────────────────┐│
│  │ Name:     [Critical Discord Alerts         ]                         ││
│  │ Channel:  (●) Discord  ( ) Email  ( ) Webhook                        ││
│  │                                                                      ││
│  │ Discord Webhook URL: [https://discord.com/api/webhooks/...]          ││
│  │                                                                      ││
│  │ Tag Pattern:    [* (all)           ]                                  ││
│  │ Min Severity:   [High ▼]                                             ││
│  │ Deduplication:  [☑] Don't re-alert same finding (7-day window)       ││
│  │                                                                      ││
│  │ [Save]  [Test 🧪]  [Cancel]                                          ││
│  └──────────────────────────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────────────────────┘
```

### Database Schema

```
┌──────────────────────────┐         ┌──────────────────────────┐
│       AlertRule          │         │     NotificationLog      │
├──────────────────────────┤         ├──────────────────────────┤
│ id (PK)                  │◄───┐    │ id (PK)                  │
│ name                     │    │    │ rule_id (FK) ────────────│───►
│ enabled                  │    │    │ task_result_id (FK)       │
│ channel_type             │    │    │ sent_at                   │
│ (email/discord/webhook)  │    │    │ status (sent/failed)      │
│ channel_config (JSONB)   │    │    │ error_message             │
│ tag_pattern              │    └────│ retry_count               │
│ severity_threshold       │         │ dedup_key                 │
│ deduplicate              │         └──────────────────────────┘
│ dedup_window_days        │
│ created_at               │         ┌──────────────────────────┐
└──────────────────────────┘         │     ScanSchedule         │
                                     ├──────────────────────────┤
┌──────────────────────────┐         │ id (PK)                  │
│     ScanScheduleLog      │         │ name                     │
├──────────────────────────┤         │ targets (JSONB)          │
│ id (PK)                  │         │ tag                      │
│ schedule_id (FK) ────────│────►    │ cron_expression          │
│ started_at               │         │ enabled                  │
│ status                   │         │ disabled_modules (JSONB) │
│ targets_submitted        │         │ priority                 │
│ task_ids (JSONB)         │         │ created_at               │
└──────────────────────────┘         │ last_run_at              │
                                     │ next_run_at              │
                                     │ run_count                │
                                     └──────────────────────────┘
```

### Design: Notification Models

```python
# artemis/db.py — Notification tables

class AlertRule(Base):
    __tablename__ = "alert_rule"
    id                 = Column(Integer, primary_key=True, autoincrement=True)
    name               = Column(String, nullable=False)
    enabled            = Column(Boolean, default=True, index=True)
    channel_type       = Column(String, nullable=False)    # "email" | "discord" | "webhook"
    channel_config     = Column(JSONB, nullable=False)      # backend-specific (webhook_url, smtp_host, etc.)
    tag_pattern        = Column(String, default="*")        # fnmatch pattern ("client_*", "*")
    severity_threshold = Column(String, default="high")     # minimum severity to trigger
    deduplicate        = Column(Boolean, default=True)
    dedup_window_days  = Column(Integer, default=7)

class NotificationLog(Base):
    __tablename__ = "notification_log"
    id              = Column(Integer, primary_key=True, autoincrement=True)
    rule_id         = Column(Integer, index=True)
    task_result_id  = Column(String, index=True)
    sent_at         = Column(DateTime, server_default=text("NOW()"))
    status          = Column(String)    # "sent" | "failed"
    error_message   = Column(Text, nullable=True)
    dedup_key       = Column(String, index=True)  # sha256(rule_id:target:finding_type)
```

### Design: Backend Abstraction & Processor Logic

```python
# artemis/notifications/backends.py — Pluggable notification backends

class NotificationBackend(ABC):
    @abstractmethod
    def send(self, config: Dict, message: Dict) -> None: ...

class DiscordBackend(NotificationBackend):
    # POST config["webhook_url"] with embed (title, color by severity, fields: target/type/module/tag)

class EmailBackend(NotificationBackend):
    # SMTP via config["smtp_host"], subject: "[Artemis] HIGH — finding on target.com"

class WebhookBackend(NotificationBackend):
    # POST config["url"] with JSON payload (full finding data)

BACKENDS = {"discord": DiscordBackend(), "email": EmailBackend(), "webhook": WebhookBackend()}
```

```python
# artemis/notifications/processor.py — Rule evaluation loop (every 60s)

def run_notification_processor():
    # 1. SELECT * FROM task_result WHERE status='INTERESTING' AND created_at > last_check
    # 2. For each new finding, evaluate against all enabled AlertRules:
    #    a. Tag match?      → fnmatch(result.tag, rule.tag_pattern)
    #    b. Severity met?   → severity_order[finding] >= severity_order[rule.threshold]
    #    c. Already sent?   → SELECT FROM notification_log WHERE dedup_key=hash AND sent_at > cutoff
    # 3. Dispatch via BACKENDS[rule.channel_type].send(rule.channel_config, message)
    # 4. Log result in notification_log (sent/failed + retry on failure)
```

### Design: Alert Rule API Signatures

```python
# artemis/api_notifications.py

@router.post("/api/alerts/rules")             # Create alert rule
@router.get("/api/alerts/rules")              # List all rules
@router.put("/api/alerts/rules/{id}")         # Update rule
@router.delete("/api/alerts/rules/{id}")      # Delete rule
@router.post("/api/alerts/rules/{id}/test")   # Send test notification
@router.get("/api/alerts/history")            # Query notification_log
```

```yaml
# docker-compose.yaml — New service
  notification-processor:
    build: .
    command: python3 -m artemis.notifications.processor
    depends_on: [postgres, redis]
```

---

## Timeline and Deliverables

### Gantt Chart

```
        May            June              July              August          Sep
       Week: 1 2 3 4  1  2  3  4  5  6  7  8  9  10  11  12   1
             ├───┤                                                Community Bonding
             │ CB │
                     ├─────────────┤                              Phase 1: Discovery
                     │Crawl│JS Ext│Dedup│
                                   ├────┤                         ◆ Midterm Eval
                                        ├───────────────┤         Phase 2: Sched+Dash
                                        │Sched│Chart.js│API│
                                                        ├────────┤Phase 3: Modules+Alerts
                                                        │M-A│M-B│M-C│Alerts│Test│Doc│
                                                                              ├──┤Final Eval

  Legend: CB=Community Bonding, ◆=Evaluation, M-A/B/C=Modules
```

### Community Bonding Period (May 8 - June 1)

- Deep study of the Karton framework task flow and module lifecycle
- Analyze existing `crawling.py`, `robots.py`, and `bruter.py` for discovery patterns
- Prototype recursive crawling logic and JS regex patterns
- Study `producer.py` and `create_tasks()` to understand scheduling integration points
- Set up Chart.js in a test Jinja2 template
- Submit a small bugfix or enhancement PR as warmup

### Phase 1: Weeks 1-4 — Attack Surface Expansion (June 2 - June 29)

**Deliverable: Deep crawler + JS endpoint extractor + target deduplicator**

| Week | Work Package | Tasks | Deliverable |
|------|-------------|-------|-------------|
| 1 | Discovery | `deep_crawler.py`: multi-level crawling with configurable depth; extract links, forms, script tags | Recursive crawler module |
| 2 | Discovery | `js_endpoint_extractor.py`: fetch JS files, regex patterns for API routes, fetch/axios/XHR calls, filtering | JS parser module |
| 3 | Discovery | `target_deduplicator.py`: URL normalization, dedup by path pattern, static asset filtering, scope enforcement | Dedup + filter module |
| 4 | Discovery | Integration testing with real targets; tune crawl depth and filters; unit tests; Docker entries | Tested discovery pipeline |

### Midterm Evaluation (June 30 - July 4)

**Expected state:** 3 new discovery modules (deep_crawler, js_endpoint_extractor, target_deduplicator) integrated into the Karton pipeline. Demonstrably discovers more endpoints than current crawling.py on test targets. Full unit test coverage.

### Phase 2: Weeks 5-8 — Scheduling + Dashboard (July 5 - August 3)

**Deliverable: Scheduling system + Chart.js analytics dashboard**

| Week | Work Package | Tasks | Deliverable |
|------|-------------|-------|-------------|
| 5 | Scheduling | `ScanSchedule` + `ScanScheduleLog` DB models; Alembic migration; cron parsing with `croniter` | Schema + migration |
| 6 | Scheduling | Scheduler service (Docker container); CRUD API endpoints; integration with `create_tasks()` | Working scheduler |
| 7 | Dashboard | Analytics SQL queries; `/api/stats/*` endpoints; Redis caching; findings trend + severity chart | API + 2 charts |
| 8 | Dashboard | Scheduling UI page; dashboard layout with module stats, risk table, upcoming scans; responsive design | Complete UI |

### Phase 3: Weeks 9-12 — New Modules + Notifications (August 4 - August 31)

**Deliverable: 3 vulnerability modules + notification system**

| Week | Work Package | Tasks | Deliverable |
|------|-------------|-------|-------------|
| 9 | Modules | Implement Module A + reporter + email template + tests | Module A |
| 10 | Modules | Implement Modules B & C + reporters + tests + Docker entries | Modules B & C |
| 11 | Notifications | `AlertRule` + `NotificationLog` DB models; Email + Discord + Webhook backends; rule evaluation engine | Working alerts |
| 12 | Notifications | Alert rules UI; notification history; retry logic; end-to-end testing; documentation | Complete alert system |

### Final Evaluation (September 1 - September 8)

---

## Effort Breakdown

```
                    Effort Distribution (350 hours)

  ┌────────────────────────────────────────────────────────────────┐
  │                                                                │
  │  WP1: Deep Crawler           ██████░░░░░░░░░░░░░  45h  (13%)  │
  │  WP1: JS Endpoint Extractor  █████░░░░░░░░░░░░░░  35h  (10%)  │
  │  WP1: Target Deduplicator    ████░░░░░░░░░░░░░░░  30h   (9%)  │
  │  WP2: Scheduling System      ██████░░░░░░░░░░░░░  50h  (14%)  │
  │  WP3: Analytics Dashboard    ██████░░░░░░░░░░░░░  50h  (14%)  │
  │  WP4: 3 New Vuln Modules     ██████░░░░░░░░░░░░░  55h  (16%)  │
  │  WP5: Notification System    ██████░░░░░░░░░░░░░  50h  (14%)  │
  │  Testing & Integration       ███░░░░░░░░░░░░░░░░  20h   (6%)  │
  │  Documentation               ██░░░░░░░░░░░░░░░░░  15h   (4%)  │
  │                                                                │
  └────────────────────────────────────────────────────────────────┘
```

---

## Technical Challenges and Mitigations

| Challenge | Risk | Mitigation |
|-----------|------|------------|
| Deep crawling may be slow or get stuck in large sites | Medium | `CRAWLER_MAX_PAGES` limit (default 100), per-request timeout, visited URL set |
| JS regex patterns may produce false endpoints | Medium | Filter static assets, validate URL format, test against real JS bundles |
| Crawl loops (page A links to B, B links to A) | Low | Track visited URLs in a set; skip already-seen pages |
| Cron scheduling accuracy (drift over restarts) | Low | Store `next_run_at` in DB; scheduler calculates from cron expression on each check |
| Chart.js performance with large datasets | Low | Aggregate data server-side; limit default time range to 90 days; cache in Redis |
| Discord webhook rate limits | Medium | Queue notifications; respect Discord 30 messages/minute limit; batch if possible |
| Notification storms after large scan completes | High | Deduplication window; configurable cooldown; severity threshold filtering |
| Adding 3 discovery modules increases scan time | Medium | Discovery modules are LOW load risk; configurable max depth and page limits |

---

## Testing Strategy

Each component gets unit tests covering core logic. Key test cases:

```python
# test/modules/test_deep_crawler.py
def test_extract_links_same_domain():       # only same-domain links extracted
def test_static_assets_filtered():          # .css, .png, .svg skipped
def test_fragments_and_mailto_ignored():    # "#section", "mailto:" skipped
def test_max_depth_enforced():              # BFS stops at CRAWLER_MAX_DEPTH

# test/modules/test_js_endpoint_extractor.py
def test_fetch_pattern():                   # fetch("/api/v1/users") → extracted
def test_axios_pattern():                   # axios.post("/api/login") → extracted
def test_route_definition():                # path: "/dashboard" → extracted
def test_static_assets_not_endpoints():     # "/style.css" → rejected

# test/modules/test_target_deduplicator.py
def test_normalize_strips_trailing_slash(): # "/path/" → "/path"
def test_normalize_sorts_params():          # "?b=2&a=1" → "?a=1&b=2"
def test_removes_tracking_params():         # utm_source, fbclid stripped
def test_numeric_path_grouping():           # /products/123 ≡ /products/456
def test_logout_urls_rejected():            # /logout, /sign-out → skipped

# test/test_scheduler.py
def test_cron_next_run():                   # "0 2 * * MON" → next Monday 02:00
def test_invalid_cron_rejected():           # croniter.is_valid("bad") → False

# test/test_notifications.py
def test_severity_threshold():              # high >= medium ✓, low >= high ✗
def test_dedup_key_deterministic():         # same inputs → same sha256 key
def test_tag_pattern_matching():            # fnmatch("client_a", "client_*") ✓
```

---

## Technology Stack

```
┌──────────────────────────────────────────────────────────────────┐
│                    TECHNOLOGY STACK                               │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────┐ │
│  │  Frontend    │  │   Backend    │  │   Infrastructure       │ │
│  ├──────────────┤  ├──────────────┤  ├────────────────────────┤ │
│  │ Chart.js     │  │ FastAPI      │  │ PostgreSQL (existing)  │ │
│  │ Bootstrap 5  │  │ SQLAlchemy   │  │ Redis (existing)       │ │
│  │ Jinja2       │  │ Karton       │  │ Docker Compose         │ │
│  │ Vanilla JS   │  │ Pydantic     │  │ (existing)             │ │
│  │              │  │ croniter     │  │                        │ │
│  │              │  │ (NEW - cron  │  │ Alembic (existing)     │ │
│  │              │  │  parsing)    │  │                        │ │
│  └──────────────┘  └──────────────┘  └────────────────────────┘ │
│                                                                  │
│  New dependencies: Chart.js (CDN), croniter (pip)                │
│  Everything else uses existing Artemis infrastructure.           │
└──────────────────────────────────────────────────────────────────┘
```

---

## Relevant Skills and Experience

- [Describe your Python experience]
- [Describe your experience with web scraping/crawling]
- [Describe your experience with web frameworks (FastAPI/Flask)]
- [Describe your experience with databases (PostgreSQL/SQLAlchemy)]
- [Describe any experience with web security, scanning tools, or OWASP Top 10]
- [Describe any experience with charting/visualization (Chart.js, D3, etc.)]
- [Describe any experience with API integrations (Discord, webhooks, SMTP)]
- [Link to relevant projects/contributions]

---

## Why This Project?

This project takes a **holistic approach** to improving Artemis. Rather than building one isolated feature, it enhances every stage of the scanning pipeline:

1. **Discover more** — The deep crawler and JS endpoint extractor find targets that current shallow discovery misses, directly increasing the number of vulnerabilities detected.
2. **Scan automatically** — Scheduling eliminates manual overhead, letting CSIRT teams set up recurring scans and focus on analysis instead of operations.
3. **See clearly** — The Chart.js dashboard transforms raw data tables into actionable visual insights, enabling data-driven prioritization.
4. **Detect more** — Three new vulnerability modules expand Artemis's detection coverage.
5. **React instantly** — Real-time notifications to Discord, email, and webhooks ensure critical findings trigger immediate action, not delayed discovery.

Each work package is independently valuable and shippable, but together they transform Artemis from a manual "run and read" tool into a **continuous, autonomous security monitoring platform**.

---

## Contributions to Artemis (Pre-GSoC)

- [List any PRs, issues, or discussions you've contributed to]
- [If none yet, describe your plan to contribute before the application deadline]

---

## Availability

- I can commit approximately **30-35 hours per week** during the coding period
- [Mention any planned vacations, exams, or other commitments]
- I am available for weekly sync calls with my mentor
