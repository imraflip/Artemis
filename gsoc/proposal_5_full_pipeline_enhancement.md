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

### Implementation: Deep Web Crawler

```python
# artemis/modules/deep_crawler.py

import re
import urllib.parse
from typing import List, Set, Tuple

from bs4 import BeautifulSoup
from karton.core import Task

from artemis import http_requests, load_risk_class
from artemis.binds import Service, TaskStatus, TaskType
from artemis.config import Config
from artemis.module_base import ArtemisBase
from artemis.task_utils import get_target_url


@load_risk_class.load_risk_class(load_risk_class.LoadRiskClass.LOW)
class DeepCrawler(ArtemisBase):
    """
    Multi-level web crawler that recursively discovers URLs, form actions,
    and JavaScript files on the same domain. Feeds discovered URLs into the
    pipeline for vulnerability scanning.
    """

    identity = "deep_crawler"
    filters = [
        {"type": TaskType.SERVICE.value, "service": Service.HTTP.value},
    ]

    def _extract_links(self, url: str, html: str) -> Tuple[Set[str], Set[str]]:
        """Extract page links and JS file URLs from HTML content."""
        parsed_base = urllib.parse.urlparse(url)
        soup = BeautifulSoup(html, "html.parser")
        links: Set[str] = set()
        js_files: Set[str] = set()

        # Extract href, src, action attributes
        for tag in soup.find_all(True):
            for attr in ("href", "src", "action"):
                value = tag.get(attr)
                if not value or value.startswith(("#", "javascript:", "mailto:")):
                    continue

                absolute = urllib.parse.urljoin(url, value)
                abs_parsed = urllib.parse.urlparse(absolute)

                # Stay on same domain
                if abs_parsed.hostname != parsed_base.hostname:
                    continue

                clean_url = absolute.split("#")[0]

                if clean_url.endswith(".js"):
                    js_files.add(clean_url)
                elif not self._is_static_asset(clean_url):
                    links.add(clean_url)

        # Extract inline script src imports: import("./chunk.js")
        for script_tag in soup.find_all("script", src=True):
            src = urllib.parse.urljoin(url, script_tag["src"])
            src_parsed = urllib.parse.urlparse(src)
            if src_parsed.hostname == parsed_base.hostname:
                js_files.add(src.split("#")[0])

        return links, js_files

    @staticmethod
    def _is_static_asset(url: str) -> bool:
        """Filter out non-interesting static assets."""
        STATIC_EXTENSIONS = {
            ".css", ".png", ".jpg", ".jpeg", ".gif", ".svg", ".ico",
            ".woff", ".woff2", ".ttf", ".eot", ".mp4", ".mp3", ".pdf",
        }
        path = urllib.parse.urlparse(url).path.lower()
        return any(path.endswith(ext) for ext in STATIC_EXTENSIONS)

    def run(self, current_task: Task) -> None:
        base_url = get_target_url(current_task)
        max_depth = Config.Modules.DeepCrawler.CRAWLER_MAX_DEPTH  # default: 3
        max_pages = Config.Modules.DeepCrawler.CRAWLER_MAX_PAGES  # default: 100

        visited: Set[str] = set()
        all_js_files: Set[str] = set()
        all_discovered_urls: Set[str] = set()

        # BFS crawl
        queue: List[Tuple[str, int]] = [(base_url, 0)]

        while queue and len(visited) < max_pages:
            url, depth = queue.pop(0)

            if url in visited or depth > max_depth:
                continue
            visited.add(url)

            try:
                response = http_requests.get(url, timeout=30)
            except Exception:
                continue

            content_type = response.headers.get("content-type", "")
            if "text/html" not in content_type:
                continue

            links, js_files = self._extract_links(url, response.text)
            all_js_files |= js_files
            all_discovered_urls |= links

            # Enqueue discovered links for deeper crawling
            for link in links:
                if link not in visited:
                    queue.append((link, depth + 1))

        self.log.info(
            f"Crawled {len(visited)} pages, found {len(all_discovered_urls)} URLs "
            f"and {len(all_js_files)} JS files on {base_url}"
        )

        # Emit URL tasks for discovered pages (vulnerability modules will scan them)
        for url in all_discovered_urls:
            new_task = Task({"type": TaskType.URL.value})
            new_task.add_payload("url", url)
            self.add_task(current_task, new_task)

        # Emit URL tasks for JS files (js_endpoint_extractor will parse them)
        for js_url in all_js_files:
            new_task = Task({"type": TaskType.URL.value})
            new_task.add_payload("url", js_url)
            new_task.add_payload("is_js_file", True)
            self.add_task(current_task, new_task)

        self.db.save_task_result(
            task=current_task,
            status=TaskStatus.OK,
            status_reason=f"Crawled {len(visited)} pages, "
                          f"discovered {len(all_discovered_urls)} URLs, "
                          f"{len(all_js_files)} JS files",
            data={
                "discovered_urls": list(all_discovered_urls),
                "js_files": list(all_js_files),
                "pages_crawled": len(visited),
                "max_depth_reached": max_depth,
            },
        )


if __name__ == "__main__":
    DeepCrawler().loop()
```

### Implementation: JavaScript Endpoint Extractor

```python
# artemis/modules/js_endpoint_extractor.py

import re
import urllib.parse
from typing import List, Set

from karton.core import Task

from artemis import http_requests, load_risk_class
from artemis.binds import TaskStatus, TaskType
from artemis.module_base import ArtemisBase


# Regex patterns for extracting API endpoints from JavaScript
JS_ENDPOINT_PATTERNS = [
    # fetch("url") or fetch('url')
    re.compile(r"""fetch\(\s*["'](\/[^"']+)["']"""),
    # axios.get("/url") / axios.post("/url") etc.
    re.compile(r"""axios\.(?:get|post|put|delete|patch)\(\s*["'](\/[^"']+)["']"""),
    # XMLHttpRequest .open("METHOD", "/url")
    re.compile(r"""\.open\(\s*["'][A-Z]+["']\s*,\s*["'](\/[^"']+)["']"""),
    # API path strings: "/api/v1/users"
    re.compile(r"""["'](\/api\/[a-zA-Z0-9_/.\-]+)["']"""),
    # Generic path-like strings: "/admin/settings"
    re.compile(r"""["'](\/[a-zA-Z][a-zA-Z0-9_/.\-]{2,})["']"""),
    # Form actions in dynamically rendered HTML
    re.compile(r"""action\s*[:=]\s*["'](\/[^"']+)["']"""),
    # Route definitions (React Router, Vue Router, etc.)
    re.compile(r"""path\s*:\s*["'](\/[^"']+)["']"""),
]

STATIC_EXTENSIONS = {
    ".css", ".png", ".jpg", ".jpeg", ".gif", ".svg", ".ico", ".woff",
    ".woff2", ".ttf", ".eot", ".map", ".mp4", ".mp3",
}

IGNORED_PATHS = {
    "/node_modules/", "/vendor/", "/bower_components/",
    "/.well-known/", "/favicon", "/robots.txt",
}


@load_risk_class.load_risk_class(load_risk_class.LoadRiskClass.LOW)
class JSEndpointExtractor(ArtemisBase):
    """
    Parses JavaScript files to extract hidden API endpoints, fetch calls,
    and route definitions. Emits discovered endpoints as URL tasks.
    """

    identity = "js_endpoint_extractor"
    filters = [
        {"type": TaskType.URL.value},
    ]

    @staticmethod
    def _is_valid_endpoint(path: str) -> bool:
        """Filter out static assets and known library paths."""
        lower = path.lower()
        if any(lower.endswith(ext) for ext in STATIC_EXTENSIONS):
            return False
        if any(ignored in lower for ignored in IGNORED_PATHS):
            return False
        # Skip very short or obviously invalid paths
        if len(path) < 3 or path.count("/") < 1:
            return False
        return True

    def run(self, current_task: Task) -> None:
        url = current_task.get_payload("url")

        # Only process JS files flagged by the deep_crawler
        if not current_task.get_payload("is_js_file", False):
            return

        try:
            response = http_requests.get(url, timeout=30)
        except Exception:
            return

        js_content = response.text
        discovered: Set[str] = set()

        for pattern in JS_ENDPOINT_PATTERNS:
            for match in pattern.findall(js_content):
                path = match.strip()
                if self._is_valid_endpoint(path):
                    discovered.add(path)

        if not discovered:
            self.db.save_task_result(
                task=current_task,
                status=TaskStatus.OK,
                status_reason=f"No endpoints found in {url}",
                data={},
            )
            return

        # Resolve relative paths to absolute URLs
        parsed_js_url = urllib.parse.urlparse(url)
        base = f"{parsed_js_url.scheme}://{parsed_js_url.hostname}"
        if parsed_js_url.port:
            base += f":{parsed_js_url.port}"

        absolute_urls: List[str] = []
        for path in discovered:
            full_url = base + path
            absolute_urls.append(full_url)

            new_task = Task({"type": TaskType.URL.value})
            new_task.add_payload("url", full_url)
            self.add_task(current_task, new_task)

        self.log.info(f"Extracted {len(absolute_urls)} endpoints from {url}")

        self.db.save_task_result(
            task=current_task,
            status=TaskStatus.OK,
            status_reason=f"Extracted {len(absolute_urls)} endpoints from JS file",
            data={
                "js_file": url,
                "endpoints_found": absolute_urls,
            },
        )


if __name__ == "__main__":
    JSEndpointExtractor().loop()
```

### Implementation: Target Deduplicator

```python
# artemis/modules/target_deduplicator.py

import hashlib
import urllib.parse
from typing import Dict, List, Set

from karton.core import Task

from artemis import load_risk_class
from artemis.binds import TaskStatus, TaskType
from artemis.module_base import ArtemisBase


LOGOUT_PATTERNS = {"logout", "signout", "sign-out", "log-out", "disconnect"}


@load_risk_class.load_risk_class(load_risk_class.LoadRiskClass.LOW)
class TargetDeduplicator(ArtemisBase):
    """
    Receives batches of discovered URLs, normalizes and deduplicates them,
    then emits clean URL tasks for vulnerability scanning modules.
    Prevents scanning the same logical endpoint multiple times.
    """

    identity = "target_deduplicator"
    batch_tasks = True
    task_max_batch_size = 50

    filters = [
        {"type": TaskType.URL.value},
    ]

    @staticmethod
    def normalize_url(url: str) -> str:
        """Normalize URL for deduplication: lowercase host, sort params,
        strip fragments and tracking parameters."""
        parsed = urllib.parse.urlparse(url.lower())

        # Sort query parameters, remove common tracking params
        TRACKING_PARAMS = {"utm_source", "utm_medium", "utm_campaign", "ref", "fbclid"}
        params = urllib.parse.parse_qs(parsed.query)
        filtered = {k: v for k, v in sorted(params.items()) if k not in TRACKING_PARAMS}
        clean_query = urllib.parse.urlencode(filtered, doseq=True)

        # Strip trailing slash
        path = parsed.path.rstrip("/") or "/"

        return urllib.parse.urlunparse((
            parsed.scheme,
            parsed.netloc,
            path,
            "",        # params
            clean_query,
            "",        # fragment
        ))

    @staticmethod
    def get_path_pattern(url: str) -> str:
        """Group URLs by path pattern for dedup.
        /products/123 and /products/456 → /products/{param}"""
        parsed = urllib.parse.urlparse(url)
        segments = parsed.path.split("/")
        pattern_segments = []
        for seg in segments:
            # Replace numeric segments with placeholder
            if seg.isdigit():
                pattern_segments.append("{id}")
            else:
                pattern_segments.append(seg)
        return "/".join(pattern_segments)

    @staticmethod
    def is_safe_url(url: str) -> bool:
        """Skip logout/destructive URLs to avoid side effects."""
        lower = url.lower()
        return not any(pattern in lower for pattern in LOGOUT_PATTERNS)

    def run(self, current_task: Task) -> None:
        # In batch mode, current_task contains multiple URL payloads
        urls = [current_task.get_payload("url")]
        if not urls[0]:
            return

        normalized: Dict[str, str] = {}  # normalized → original
        for url in urls:
            norm = self.normalize_url(url)
            if norm not in normalized and self.is_safe_url(url):
                normalized[norm] = url

        # Group by path pattern, keep one representative per pattern
        pattern_groups: Dict[str, str] = {}
        for norm_url, orig_url in normalized.items():
            pattern = self.get_path_pattern(norm_url)
            if pattern not in pattern_groups:
                pattern_groups[pattern] = orig_url

        deduped = list(pattern_groups.values())

        for url in deduped:
            new_task = Task({"type": TaskType.URL.value})
            new_task.add_payload("url", url)
            new_task.add_payload("deduplicated", True)
            self.add_task(current_task, new_task)

        self.db.save_task_result(
            task=current_task,
            status=TaskStatus.OK,
            status_reason=f"Deduplicated {len(urls)} URLs → {len(deduped)} unique targets",
            data={
                "input_count": len(urls),
                "output_count": len(deduped),
                "deduplicated_urls": deduped,
            },
        )


if __name__ == "__main__":
    TargetDeduplicator().loop()
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

### Implementation: Database Models (SQLAlchemy)

```python
# artemis/db.py — New models added alongside existing Analysis, TaskResult, etc.

import enum
from sqlalchemy import (
    Boolean, Column, DateTime, Integer, String, Text, text
)
from sqlalchemy.dialects.postgresql import JSONB


class ScanSchedule(Base):  # type: ignore
    """A recurring scan schedule with cron expression."""

    __tablename__ = "scan_schedule"

    id = Column(Integer, primary_key=True, autoincrement=True)
    name = Column(String, nullable=False)
    targets = Column(JSONB, nullable=False)          # ["example.com", "10.0.0.1"]
    tag = Column(String, index=True)
    cron_expression = Column(String, nullable=False)  # "0 2 * * MON"
    enabled = Column(Boolean, default=True, index=True)
    disabled_modules = Column(JSONB, default=[])
    priority = Column(String, default="normal")
    created_at = Column(DateTime, server_default=text("NOW()"))
    last_run_at = Column(DateTime, nullable=True)
    next_run_at = Column(DateTime, nullable=False)
    run_count = Column(Integer, default=0)


class ScanScheduleLog(Base):  # type: ignore
    """Execution log for a scan schedule run."""

    __tablename__ = "scan_schedule_log"

    id = Column(Integer, primary_key=True, autoincrement=True)
    schedule_id = Column(Integer, nullable=False, index=True)
    started_at = Column(DateTime, server_default=text("NOW()"))
    status = Column(String, default="running")  # running | completed | error
    targets_submitted = Column(Integer)
    task_ids = Column(JSONB, default=[])
    error_message = Column(Text, nullable=True)
```

### Implementation: Alembic Migration

```python
# alembic/versions/xxxx_add_scan_schedule_tables.py

"""Add scan schedule tables for recurring scans."""

from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects.postgresql import JSONB


def upgrade() -> None:
    op.create_table(
        "scan_schedule",
        sa.Column("id", sa.Integer, primary_key=True, autoincrement=True),
        sa.Column("name", sa.String, nullable=False),
        sa.Column("targets", JSONB, nullable=False),
        sa.Column("tag", sa.String, index=True),
        sa.Column("cron_expression", sa.String, nullable=False),
        sa.Column("enabled", sa.Boolean, default=True, index=True),
        sa.Column("disabled_modules", JSONB, server_default="[]"),
        sa.Column("priority", sa.String, server_default="normal"),
        sa.Column("created_at", sa.DateTime, server_default=sa.text("NOW()")),
        sa.Column("last_run_at", sa.DateTime, nullable=True),
        sa.Column("next_run_at", sa.DateTime, nullable=False),
        sa.Column("run_count", sa.Integer, server_default="0"),
    )

    op.create_table(
        "scan_schedule_log",
        sa.Column("id", sa.Integer, primary_key=True, autoincrement=True),
        sa.Column("schedule_id", sa.Integer, nullable=False, index=True),
        sa.Column("started_at", sa.DateTime, server_default=sa.text("NOW()")),
        sa.Column("status", sa.String, server_default="running"),
        sa.Column("targets_submitted", sa.Integer),
        sa.Column("task_ids", JSONB, server_default="[]"),
        sa.Column("error_message", sa.Text, nullable=True),
    )


def downgrade() -> None:
    op.drop_table("scan_schedule_log")
    op.drop_table("scan_schedule")
```

### Implementation: Scheduler Service

```python
# artemis/scheduler.py — Standalone service (runs as a Docker container)

import logging
import time
from datetime import datetime, timezone

from croniter import croniter
from karton.core.task import TaskPriority
from sqlalchemy import select

from artemis.db import DB, ScanSchedule, ScanScheduleLog
from artemis.producer import create_tasks

logger = logging.getLogger(__name__)
POLL_INTERVAL_SECONDS = 60


def compute_next_run(cron_expression: str, after: datetime) -> datetime:
    """Compute the next run time from a cron expression."""
    cron = croniter(cron_expression, after)
    return cron.get_next(datetime)


def run_scheduler() -> None:
    """Main scheduler loop. Polls DB every 60s for due schedules."""
    db = DB()
    logger.info("Scheduler service started")

    while True:
        now = datetime.now(timezone.utc)

        with db.session_scope() as session:
            due_schedules = session.execute(
                select(ScanSchedule).where(
                    ScanSchedule.enabled == True,
                    ScanSchedule.next_run_at <= now,
                )
            ).scalars().all()

            for schedule in due_schedules:
                logger.info(f"Executing schedule '{schedule.name}' (id={schedule.id})")

                # Create execution log entry
                log_entry = ScanScheduleLog(
                    schedule_id=schedule.id,
                    status="running",
                    targets_submitted=len(schedule.targets),
                )
                session.add(log_entry)
                session.flush()

                try:
                    # Use existing create_tasks() — no reinvention needed
                    task_ids = create_tasks(
                        uris=schedule.targets,
                        tag=schedule.tag,
                        disabled_modules=schedule.disabled_modules or [],
                        priority=TaskPriority(schedule.priority),
                    )

                    # Update schedule metadata
                    schedule.last_run_at = now
                    schedule.next_run_at = compute_next_run(
                        schedule.cron_expression, now
                    )
                    schedule.run_count += 1

                    # Update log entry
                    log_entry.status = "completed"
                    log_entry.task_ids = task_ids

                    logger.info(
                        f"Schedule '{schedule.name}' completed: "
                        f"{len(task_ids)} tasks submitted, "
                        f"next run at {schedule.next_run_at}"
                    )

                except Exception as e:
                    log_entry.status = "error"
                    log_entry.error_message = str(e)
                    logger.error(f"Schedule '{schedule.name}' failed: {e}")

            session.commit()

        time.sleep(POLL_INTERVAL_SECONDS)


if __name__ == "__main__":
    run_scheduler()
```

### Implementation: Scheduling API Endpoints

```python
# artemis/api_schedules.py — FastAPI router for schedule CRUD

from datetime import datetime, timezone
from typing import Any, Dict, List, Optional

from croniter import croniter
from fastapi import APIRouter, Body, Depends, HTTPException
from karton.core.task import TaskPriority
from sqlalchemy import select

from artemis.api import verify_api_token
from artemis.db import DB, ScanSchedule, ScanScheduleLog
from artemis.producer import create_tasks
from artemis.scheduler import compute_next_run

router = APIRouter(prefix="/api/schedules", tags=["schedules"])
db = DB()


@router.post("", dependencies=[Depends(verify_api_token)])
def create_schedule(
    name: str = Body(...),
    targets: List[str] = Body(...),
    cron_expression: str = Body(...),
    tag: Optional[str] = Body(default=None),
    disabled_modules: List[str] = Body(default=[]),
    priority: str = Body(default="normal"),
) -> Dict[str, Any]:
    """Create a new scan schedule."""
    # Validate cron expression
    if not croniter.is_valid(cron_expression):
        raise HTTPException(status_code=400, detail=f"Invalid cron: {cron_expression}")

    now = datetime.now(timezone.utc)
    next_run = compute_next_run(cron_expression, now)

    with db.session_scope() as session:
        schedule = ScanSchedule(
            name=name,
            targets=targets,
            tag=tag,
            cron_expression=cron_expression,
            disabled_modules=disabled_modules,
            priority=priority,
            next_run_at=next_run,
        )
        session.add(schedule)
        session.flush()
        schedule_id = schedule.id
        session.commit()

    return {"ok": True, "id": schedule_id, "next_run_at": next_run.isoformat()}


@router.get("", dependencies=[Depends(verify_api_token)])
def list_schedules() -> List[Dict[str, Any]]:
    """List all scan schedules."""
    with db.session_scope() as session:
        schedules = session.execute(select(ScanSchedule)).scalars().all()
        return [
            {
                "id": s.id,
                "name": s.name,
                "targets_count": len(s.targets),
                "tag": s.tag,
                "cron_expression": s.cron_expression,
                "enabled": s.enabled,
                "last_run_at": s.last_run_at.isoformat() if s.last_run_at else None,
                "next_run_at": s.next_run_at.isoformat(),
                "run_count": s.run_count,
            }
            for s in schedules
        ]


@router.post("/{schedule_id}/trigger", dependencies=[Depends(verify_api_token)])
def trigger_schedule(schedule_id: int) -> Dict[str, Any]:
    """Manually trigger a schedule to run immediately."""
    with db.session_scope() as session:
        schedule = session.get(ScanSchedule, schedule_id)
        if not schedule:
            raise HTTPException(status_code=404, detail="Schedule not found")

        task_ids = create_tasks(
            uris=schedule.targets,
            tag=schedule.tag,
            disabled_modules=schedule.disabled_modules or [],
            priority=TaskPriority(schedule.priority),
        )

        log_entry = ScanScheduleLog(
            schedule_id=schedule.id,
            status="completed",
            targets_submitted=len(schedule.targets),
            task_ids=task_ids,
        )
        session.add(log_entry)
        schedule.last_run_at = datetime.now(timezone.utc)
        schedule.run_count += 1
        session.commit()

    return {"ok": True, "tasks_submitted": len(task_ids), "task_ids": task_ids}


@router.get("/{schedule_id}/history", dependencies=[Depends(verify_api_token)])
def get_schedule_history(schedule_id: int, limit: int = 10) -> List[Dict[str, Any]]:
    """Get execution history for a schedule."""
    with db.session_scope() as session:
        logs = session.execute(
            select(ScanScheduleLog)
            .where(ScanScheduleLog.schedule_id == schedule_id)
            .order_by(ScanScheduleLog.started_at.desc())
            .limit(limit)
        ).scalars().all()

        return [
            {
                "id": log.id,
                "started_at": log.started_at.isoformat(),
                "status": log.status,
                "targets_submitted": log.targets_submitted,
                "error_message": log.error_message,
            }
            for log in logs
        ]
```

### Docker Compose Entry

```yaml
# Added to docker-compose.yaml

  scheduler:
    build: .
    command: python3 -m artemis.scheduler
    restart: always
    depends_on:
      - postgres
      - redis
    env_file: .env
    volumes:
      - ./docker/karton.ini:/etc/karton/karton.ini
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

### Implementation: Analytics SQL Queries & API

```python
# artemis/api_stats.py — FastAPI router for dashboard statistics

import json
from datetime import datetime, timedelta, timezone
from typing import Any, Dict, List, Optional

from fastapi import APIRouter, Depends, Query
from sqlalchemy import func, select, text, case

from artemis.api import verify_api_token
from artemis.db import DB, TaskResult

router = APIRouter(prefix="/api/stats", tags=["stats"])
db = DB()

# Redis cache helper (TTL: 5 minutes)
CACHE_TTL = 300


@router.get("/findings-trend", dependencies=[Depends(verify_api_token)])
def findings_trend(
    days: int = Query(default=90, ge=1, le=365),
) -> List[Dict[str, Any]]:
    """
    Findings over time, grouped by day and severity.

    SQL equivalent:
        SELECT
            date_trunc('day', created_at) AS date,
            COUNT(*) FILTER (WHERE result->>'severity' = 'high')   AS high,
            COUNT(*) FILTER (WHERE result->>'severity' = 'medium') AS medium,
            COUNT(*) FILTER (WHERE result->>'severity' = 'low')    AS low
        FROM task_result
        WHERE status = 'INTERESTING'
          AND created_at >= NOW() - INTERVAL '90 days'
        GROUP BY date
        ORDER BY date;
    """
    cache_key = f"stats:findings-trend:{days}"
    cached = db.redis.get(cache_key)
    if cached:
        return json.loads(cached)

    cutoff = datetime.now(timezone.utc) - timedelta(days=days)

    with db.session_scope() as session:
        rows = session.execute(
            select(
                func.date_trunc("day", TaskResult.created_at).label("date"),
                func.count().filter(
                    TaskResult.result["severity"].as_string() == "high"
                ).label("high"),
                func.count().filter(
                    TaskResult.result["severity"].as_string() == "medium"
                ).label("medium"),
                func.count().filter(
                    TaskResult.result["severity"].as_string() == "low"
                ).label("low"),
            )
            .where(
                TaskResult.status == "INTERESTING",
                TaskResult.created_at >= cutoff,
            )
            .group_by(func.date_trunc("day", TaskResult.created_at))
            .order_by(func.date_trunc("day", TaskResult.created_at))
        ).all()

        result = [
            {
                "date": row.date.isoformat(),
                "high": row.high,
                "medium": row.medium,
                "low": row.low,
            }
            for row in rows
        ]

    db.redis.setex(cache_key, CACHE_TTL, json.dumps(result))
    return result


@router.get("/severity-distribution", dependencies=[Depends(verify_api_token)])
def severity_distribution() -> List[Dict[str, Any]]:
    """
    Severity breakdown for pie/donut chart.

    SQL equivalent:
        SELECT
            COALESCE(result->>'severity', 'unknown') AS severity,
            COUNT(*) AS count
        FROM task_result
        WHERE status = 'INTERESTING'
        GROUP BY severity
        ORDER BY count DESC;
    """
    cache_key = "stats:severity-distribution"
    cached = db.redis.get(cache_key)
    if cached:
        return json.loads(cached)

    with db.session_scope() as session:
        rows = session.execute(
            select(
                func.coalesce(
                    TaskResult.result["severity"].as_string(), "unknown"
                ).label("severity"),
                func.count().label("count"),
            )
            .where(TaskResult.status == "INTERESTING")
            .group_by("severity")
            .order_by(func.count().desc())
        ).all()

        result = [{"severity": row.severity, "count": row.count} for row in rows]

    db.redis.setex(cache_key, CACHE_TTL, json.dumps(result))
    return result


@router.get("/modules", dependencies=[Depends(verify_api_token)])
def module_stats() -> List[Dict[str, Any]]:
    """
    Findings per module (bar chart).

    SQL equivalent:
        SELECT receiver AS module, COUNT(*) AS count
        FROM task_result
        WHERE status = 'INTERESTING'
        GROUP BY receiver
        ORDER BY count DESC;
    """
    cache_key = "stats:modules"
    cached = db.redis.get(cache_key)
    if cached:
        return json.loads(cached)

    with db.session_scope() as session:
        rows = session.execute(
            select(
                TaskResult.receiver.label("module"),
                func.count().label("count"),
            )
            .where(TaskResult.status == "INTERESTING")
            .group_by(TaskResult.receiver)
            .order_by(func.count().desc())
        ).all()

        result = [{"module": row.module, "count": row.count} for row in rows]

    db.redis.setex(cache_key, CACHE_TTL, json.dumps(result))
    return result


@router.get("/tags-summary", dependencies=[Depends(verify_api_token)])
def tags_summary() -> List[Dict[str, Any]]:
    """
    Per-organization (tag) risk breakdown.

    SQL equivalent:
        SELECT
            tag,
            COUNT(*) AS total,
            COUNT(*) FILTER (WHERE result->>'severity' = 'high')   AS high,
            COUNT(*) FILTER (WHERE result->>'severity' = 'medium') AS medium
        FROM task_result
        WHERE status = 'INTERESTING' AND tag IS NOT NULL
        GROUP BY tag
        ORDER BY high DESC, total DESC;
    """
    cache_key = "stats:tags-summary"
    cached = db.redis.get(cache_key)
    if cached:
        return json.loads(cached)

    with db.session_scope() as session:
        rows = session.execute(
            select(
                TaskResult.tag,
                func.count().label("total"),
                func.count().filter(
                    TaskResult.result["severity"].as_string() == "high"
                ).label("high"),
                func.count().filter(
                    TaskResult.result["severity"].as_string() == "medium"
                ).label("medium"),
            )
            .where(
                TaskResult.status == "INTERESTING",
                TaskResult.tag.isnot(None),
            )
            .group_by(TaskResult.tag)
            .order_by(text("high DESC, total DESC"))
        ).all()

        result = [
            {"tag": row.tag, "total": row.total, "high": row.high, "medium": row.medium}
            for row in rows
        ]

    db.redis.setex(cache_key, CACHE_TTL, json.dumps(result))
    return result
```

### Implementation: Chart.js Frontend (Jinja2 + JavaScript)

```html
<!-- templates/dashboard.jinja2 — Analytics dashboard page -->
{% extends "base.jinja2" %}
{% block content %}

<!-- Summary Cards -->
<div class="row mb-4">
  <div class="col-md-3">
    <div class="card text-center">
      <div class="card-body">
        <h3 id="total-analyses">—</h3>
        <p class="text-muted">Analyses</p>
      </div>
    </div>
  </div>
  <div class="col-md-3">
    <div class="card text-center">
      <div class="card-body">
        <h3 id="total-findings">—</h3>
        <p class="text-muted">Findings</p>
      </div>
    </div>
  </div>
  <div class="col-md-3">
    <div class="card text-center border-danger">
      <div class="card-body">
        <h3 id="high-sev" class="text-danger">—</h3>
        <p class="text-muted">High Severity</p>
      </div>
    </div>
  </div>
  <div class="col-md-3">
    <div class="card text-center">
      <div class="card-body">
        <h3 id="pending-tasks">—</h3>
        <p class="text-muted">Pending Tasks</p>
      </div>
    </div>
  </div>
</div>

<!-- Charts Row -->
<div class="row mb-4">
  <!-- Findings Trend (Line Chart) -->
  <div class="col-md-8">
    <div class="card">
      <div class="card-header d-flex justify-content-between">
        <span>Findings Trend</span>
        <div class="btn-group btn-group-sm">
          <button class="btn btn-outline-secondary" onclick="loadTrend(7)">1W</button>
          <button class="btn btn-outline-secondary active" onclick="loadTrend(30)">1M</button>
          <button class="btn btn-outline-secondary" onclick="loadTrend(90)">3M</button>
        </div>
      </div>
      <div class="card-body">
        <canvas id="trendChart" height="250"></canvas>
      </div>
    </div>
  </div>

  <!-- Severity Donut -->
  <div class="col-md-4">
    <div class="card">
      <div class="card-header">Severity Breakdown</div>
      <div class="card-body">
        <canvas id="severityChart" height="250"></canvas>
      </div>
    </div>
  </div>
</div>

<!-- Module Performance Bar Chart -->
<div class="row mb-4">
  <div class="col-md-12">
    <div class="card">
      <div class="card-header">Top Active Modules</div>
      <div class="card-body">
        <canvas id="moduleChart" height="150"></canvas>
      </div>
    </div>
  </div>
</div>

<script src="https://cdn.jsdelivr.net/npm/chart.js@4/dist/chart.umd.min.js"></script>
<script>
const API_HEADERS = {"X-API-Token": "{{ api_token }}"};

// ── Findings Trend Line Chart ──
let trendChart = null;

async function loadTrend(days = 30) {
    const resp = await fetch(`/api/stats/findings-trend?days=${days}`, {headers: API_HEADERS});
    const data = await resp.json();

    const labels = data.map(d => d.date.split("T")[0]);
    const config = {
        type: "line",
        data: {
            labels,
            datasets: [
                {label: "High",   data: data.map(d => d.high),   borderColor: "#dc3545", fill: true, backgroundColor: "rgba(220,53,69,0.1)"},
                {label: "Medium", data: data.map(d => d.medium), borderColor: "#ffc107", fill: true, backgroundColor: "rgba(255,193,7,0.1)"},
                {label: "Low",    data: data.map(d => d.low),    borderColor: "#198754", fill: true, backgroundColor: "rgba(25,135,84,0.1)"},
            ],
        },
        options: {responsive: true, scales: {y: {beginAtZero: true}}},
    };

    if (trendChart) trendChart.destroy();
    trendChart = new Chart(document.getElementById("trendChart"), config);
}

// ── Severity Donut Chart ──
async function loadSeverity() {
    const resp = await fetch("/api/stats/severity-distribution", {headers: API_HEADERS});
    const data = await resp.json();

    const colors = {high: "#dc3545", medium: "#ffc107", low: "#198754", unknown: "#6c757d"};
    new Chart(document.getElementById("severityChart"), {
        type: "doughnut",
        data: {
            labels: data.map(d => d.severity),
            datasets: [{
                data: data.map(d => d.count),
                backgroundColor: data.map(d => colors[d.severity] || "#6c757d"),
            }],
        },
    });

    // Update summary cards
    const total = data.reduce((s, d) => s + d.count, 0);
    const high = data.find(d => d.severity === "high");
    document.getElementById("total-findings").textContent = total.toLocaleString();
    document.getElementById("high-sev").textContent = high ? high.count.toLocaleString() : "0";
}

// ── Module Bar Chart ──
async function loadModules() {
    const resp = await fetch("/api/stats/modules", {headers: API_HEADERS});
    const data = await resp.json();

    new Chart(document.getElementById("moduleChart"), {
        type: "bar",
        data: {
            labels: data.slice(0, 10).map(d => d.module),
            datasets: [{
                label: "Findings",
                data: data.slice(0, 10).map(d => d.count),
                backgroundColor: "#0d6efd",
            }],
        },
        options: {indexAxis: "y", responsive: true},
    });
}

// Load all charts on page load
loadTrend(30);
loadSeverity();
loadModules();
</script>

{% endblock %}
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

### Implementation: Notification Models & Backends

```python
# artemis/db.py — Notification-related models

class AlertRule(Base):  # type: ignore
    """A rule that matches INTERESTING findings and dispatches notifications."""

    __tablename__ = "alert_rule"

    id = Column(Integer, primary_key=True, autoincrement=True)
    name = Column(String, nullable=False)
    enabled = Column(Boolean, default=True, index=True)
    channel_type = Column(String, nullable=False)   # "email", "discord", "webhook"
    channel_config = Column(JSONB, nullable=False)   # backend-specific config
    tag_pattern = Column(String, default="*")        # "*" matches all tags
    severity_threshold = Column(String, default="high")  # "high", "medium", "low"
    deduplicate = Column(Boolean, default=True)
    dedup_window_days = Column(Integer, default=7)
    created_at = Column(DateTime, server_default=text("NOW()"))


class NotificationLog(Base):  # type: ignore
    """Log of sent (or failed) notifications for auditing and dedup."""

    __tablename__ = "notification_log"

    id = Column(Integer, primary_key=True, autoincrement=True)
    rule_id = Column(Integer, nullable=False, index=True)
    task_result_id = Column(String, nullable=False, index=True)
    sent_at = Column(DateTime, server_default=text("NOW()"))
    status = Column(String, default="sent")    # "sent" | "failed"
    error_message = Column(Text, nullable=True)
    retry_count = Column(Integer, default=0)
    dedup_key = Column(String, index=True)     # hash for deduplication lookups
```

### Implementation: Notification Backends

```python
# artemis/notifications/backends.py

import json
import logging
import smtplib
from abc import ABC, abstractmethod
from email.mime.text import MIMEText
from typing import Any, Dict

import requests

logger = logging.getLogger(__name__)


class NotificationBackend(ABC):
    """Base class for notification delivery backends."""

    @abstractmethod
    def send(self, config: Dict[str, Any], message: Dict[str, Any]) -> None:
        """Send a notification. Raises on failure."""
        ...


class DiscordBackend(NotificationBackend):
    """Send notifications via Discord webhook."""

    def send(self, config: Dict[str, Any], message: Dict[str, Any]) -> None:
        webhook_url = config["webhook_url"]

        severity_colors = {"high": 0xDC3545, "medium": 0xFFC107, "low": 0x198754}
        severity = message.get("severity", "medium")

        embed = {
            "title": f"{'🔴' if severity == 'high' else '🟡'} {severity.upper()} — Artemis Finding",
            "color": severity_colors.get(severity, 0x6C757D),
            "fields": [
                {"name": "Target",  "value": message["target"],    "inline": True},
                {"name": "Type",    "value": message["type"],      "inline": True},
                {"name": "Module",  "value": message["module"],    "inline": True},
                {"name": "Tag",     "value": message.get("tag", "—"), "inline": True},
            ],
            "description": message.get("description", ""),
            "timestamp": message["timestamp"],
            "footer": {"text": "Artemis Scanner"},
        }

        resp = requests.post(
            webhook_url,
            json={"embeds": [embed]},
            timeout=10,
        )
        resp.raise_for_status()
        logger.info(f"Discord notification sent for {message['target']}")


class EmailBackend(NotificationBackend):
    """Send notifications via SMTP."""

    def send(self, config: Dict[str, Any], message: Dict[str, Any]) -> None:
        severity = message.get("severity", "medium").upper()
        subject = f"[Artemis] {'🔴' if severity == 'HIGH' else '🟡'} {severity} — {message['type']} on {message['target']}"

        body = (
            f"A new {severity}-severity finding was detected:\n\n"
            f"  Target:   {message['target']}\n"
            f"  Type:     {message['type']}\n"
            f"  Module:   {message['module']}\n"
            f"  Severity: {severity}\n"
            f"  Tag:      {message.get('tag', '—')}\n"
            f"  Time:     {message['timestamp']}\n"
        )

        msg = MIMEText(body)
        msg["Subject"] = subject
        msg["From"] = config["from_address"]
        msg["To"] = config["to_address"]

        with smtplib.SMTP(config["smtp_host"], config.get("smtp_port", 587)) as server:
            if config.get("smtp_tls", True):
                server.starttls()
            if config.get("smtp_user"):
                server.login(config["smtp_user"], config["smtp_password"])
            server.send_message(msg)

        logger.info(f"Email notification sent to {config['to_address']}")


class WebhookBackend(NotificationBackend):
    """Send notifications to a custom webhook URL as JSON."""

    def send(self, config: Dict[str, Any], message: Dict[str, Any]) -> None:
        resp = requests.post(
            config["url"],
            json=message,
            headers=config.get("headers", {}),
            timeout=10,
        )
        resp.raise_for_status()
        logger.info(f"Webhook notification sent to {config['url']}")


# Registry of available backends
BACKENDS: Dict[str, NotificationBackend] = {
    "discord": DiscordBackend(),
    "email": EmailBackend(),
    "webhook": WebhookBackend(),
}
```

### Implementation: Notification Processor Service

```python
# artemis/notifications/processor.py — Polls for new findings, evaluates rules, dispatches

import fnmatch
import hashlib
import logging
import time
from datetime import datetime, timedelta, timezone
from typing import Any, Dict, List

from sqlalchemy import select

from artemis.db import DB, AlertRule, NotificationLog, TaskResult
from artemis.notifications.backends import BACKENDS

logger = logging.getLogger(__name__)

SEVERITY_ORDER = {"low": 1, "medium": 2, "high": 3}
POLL_INTERVAL_SECONDS = 60
RETRY_DELAYS = [60, 300, 1800, 7200]  # 1m, 5m, 30m, 2h


def _severity_meets_threshold(severity: str, threshold: str) -> bool:
    """Check if a finding's severity meets the rule's minimum threshold."""
    return SEVERITY_ORDER.get(severity, 0) >= SEVERITY_ORDER.get(threshold, 0)


def _dedup_key(rule_id: int, target: str, finding_type: str) -> str:
    """Generate a deduplication key for a finding + rule combination."""
    raw = f"{rule_id}:{target}:{finding_type}"
    return hashlib.sha256(raw.encode()).hexdigest()


def _build_message(task_result: TaskResult) -> Dict[str, Any]:
    """Convert a TaskResult into a notification message dict."""
    result_data = task_result.result or {}
    return {
        "target": task_result.target_string,
        "type": task_result.status_reason or "Unknown",
        "module": task_result.receiver,
        "severity": result_data.get("severity", "medium"),
        "tag": task_result.tag,
        "description": result_data.get("description", ""),
        "timestamp": task_result.created_at.isoformat(),
        "task_result_id": task_result.id,
    }


def run_notification_processor() -> None:
    """Main loop: poll for new INTERESTING results, evaluate rules, dispatch."""
    db = DB()
    last_check = datetime.now(timezone.utc) - timedelta(minutes=5)
    logger.info("Notification processor started")

    while True:
        now = datetime.now(timezone.utc)

        with db.session_scope() as session:
            # 1. Fetch new INTERESTING results since last check
            new_results = session.execute(
                select(TaskResult).where(
                    TaskResult.status == "INTERESTING",
                    TaskResult.created_at > last_check,
                )
            ).scalars().all()

            if not new_results:
                last_check = now
                time.sleep(POLL_INTERVAL_SECONDS)
                continue

            # 2. Load enabled alert rules
            rules = session.execute(
                select(AlertRule).where(AlertRule.enabled == True)
            ).scalars().all()

            # 3. Evaluate each result against each rule
            for result in new_results:
                message = _build_message(result)

                for rule in rules:
                    # Tag pattern match (supports wildcards like "client_*")
                    if rule.tag_pattern != "*":
                        if not result.tag or not fnmatch.fnmatch(result.tag, rule.tag_pattern):
                            continue

                    # Severity threshold check
                    if not _severity_meets_threshold(message["severity"], rule.severity_threshold):
                        continue

                    # Deduplication check
                    if rule.deduplicate:
                        key = _dedup_key(rule.id, result.target_string, result.status_reason)
                        cutoff = now - timedelta(days=rule.dedup_window_days)
                        existing = session.execute(
                            select(NotificationLog).where(
                                NotificationLog.dedup_key == key,
                                NotificationLog.sent_at > cutoff,
                                NotificationLog.status == "sent",
                            )
                        ).scalars().first()

                        if existing:
                            continue  # Already notified within window

                    # 4. Dispatch via the appropriate backend
                    backend = BACKENDS.get(rule.channel_type)
                    if not backend:
                        logger.error(f"Unknown backend: {rule.channel_type}")
                        continue

                    log_entry = NotificationLog(
                        rule_id=rule.id,
                        task_result_id=result.id,
                        dedup_key=_dedup_key(rule.id, result.target_string, result.status_reason),
                    )

                    try:
                        backend.send(rule.channel_config, message)
                        log_entry.status = "sent"
                        logger.info(
                            f"Notification sent: rule='{rule.name}' "
                            f"target='{result.target_string}' "
                            f"via={rule.channel_type}"
                        )
                    except Exception as e:
                        log_entry.status = "failed"
                        log_entry.error_message = str(e)
                        logger.error(f"Notification failed for rule '{rule.name}': {e}")

                    session.add(log_entry)

            session.commit()

        last_check = now
        time.sleep(POLL_INTERVAL_SECONDS)


if __name__ == "__main__":
    run_notification_processor()
```

### Implementation: Notification API & Alert Rule CRUD

```python
# artemis/api_notifications.py — FastAPI router for alert rule management

from typing import Any, Dict, List, Optional

from fastapi import APIRouter, Body, Depends, HTTPException
from sqlalchemy import select

from artemis.api import verify_api_token
from artemis.db import DB, AlertRule, NotificationLog
from artemis.notifications.backends import BACKENDS

router = APIRouter(prefix="/api/alerts", tags=["alerts"])
db = DB()


@router.post("/rules", dependencies=[Depends(verify_api_token)])
def create_rule(
    name: str = Body(...),
    channel_type: str = Body(...),
    channel_config: Dict[str, Any] = Body(...),
    tag_pattern: str = Body(default="*"),
    severity_threshold: str = Body(default="high"),
    deduplicate: bool = Body(default=True),
    dedup_window_days: int = Body(default=7),
) -> Dict[str, Any]:
    """Create a new alert rule."""
    if channel_type not in BACKENDS:
        raise HTTPException(
            status_code=400,
            detail=f"Invalid channel_type. Must be one of: {list(BACKENDS.keys())}",
        )

    with db.session_scope() as session:
        rule = AlertRule(
            name=name,
            channel_type=channel_type,
            channel_config=channel_config,
            tag_pattern=tag_pattern,
            severity_threshold=severity_threshold,
            deduplicate=deduplicate,
            dedup_window_days=dedup_window_days,
        )
        session.add(rule)
        session.flush()
        rule_id = rule.id
        session.commit()

    return {"ok": True, "id": rule_id}


@router.get("/rules", dependencies=[Depends(verify_api_token)])
def list_rules() -> List[Dict[str, Any]]:
    """List all alert rules."""
    with db.session_scope() as session:
        rules = session.execute(select(AlertRule)).scalars().all()
        return [
            {
                "id": r.id,
                "name": r.name,
                "enabled": r.enabled,
                "channel_type": r.channel_type,
                "tag_pattern": r.tag_pattern,
                "severity_threshold": r.severity_threshold,
            }
            for r in rules
        ]


@router.post("/rules/{rule_id}/test", dependencies=[Depends(verify_api_token)])
def test_rule(rule_id: int) -> Dict[str, Any]:
    """Send a test notification through a rule's configured backend."""
    with db.session_scope() as session:
        rule = session.get(AlertRule, rule_id)
        if not rule:
            raise HTTPException(status_code=404, detail="Rule not found")

        backend = BACKENDS.get(rule.channel_type)
        if not backend:
            raise HTTPException(status_code=400, detail=f"Unknown backend: {rule.channel_type}")

        test_message = {
            "target": "test.example.com",
            "type": "Test Notification",
            "module": "notification_test",
            "severity": "medium",
            "tag": "test",
            "description": "This is a test notification from Artemis.",
            "timestamp": "2026-03-14T12:00:00Z",
            "task_result_id": "test-0000",
        }

        try:
            backend.send(rule.channel_config, test_message)
            return {"ok": True, "message": "Test notification sent successfully"}
        except Exception as e:
            raise HTTPException(status_code=500, detail=f"Test failed: {str(e)}")
```

### Docker Compose Entries

```yaml
# Added to docker-compose.yaml for notification processing

  notification-processor:
    build: .
    command: python3 -m artemis.notifications.processor
    restart: always
    depends_on:
      - postgres
      - redis
    env_file: .env
    volumes:
      - ./docker/karton.ini:/etc/karton/karton.ini
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

## Example Unit Tests

```python
# test/modules/test_deep_crawler.py

import unittest
from unittest.mock import MagicMock, patch

from artemis.modules.deep_crawler import DeepCrawler


class TestDeepCrawler(unittest.TestCase):
    """Unit tests for the deep web crawler module."""

    def test_extract_links_same_domain(self):
        """Links on the same domain should be extracted."""
        html = """
        <html>
        <body>
            <a href="/page1">Page 1</a>
            <a href="/page2">Page 2</a>
            <a href="https://external.com/nope">External</a>
            <form action="/submit"></form>
            <script src="/static/app.js"></script>
        </body>
        </html>
        """
        crawler = DeepCrawler.__new__(DeepCrawler)
        links, js_files = crawler._extract_links("https://target.com/", html)

        self.assertIn("https://target.com/page1", links)
        self.assertIn("https://target.com/page2", links)
        self.assertIn("https://target.com/submit", links)
        self.assertNotIn("https://external.com/nope", links)
        self.assertIn("https://target.com/static/app.js", js_files)

    def test_static_asset_filtering(self):
        """Static assets like .css, .png should be filtered out."""
        self.assertTrue(DeepCrawler._is_static_asset("https://x.com/style.css"))
        self.assertTrue(DeepCrawler._is_static_asset("https://x.com/logo.png"))
        self.assertFalse(DeepCrawler._is_static_asset("https://x.com/api/users"))
        self.assertFalse(DeepCrawler._is_static_asset("https://x.com/login.php"))

    def test_extract_links_ignores_fragments_and_mailto(self):
        """Fragment-only and mailto links should be skipped."""
        html = """
        <a href="#section">Skip</a>
        <a href="mailto:admin@x.com">Skip</a>
        <a href="/real-page">Keep</a>
        """
        crawler = DeepCrawler.__new__(DeepCrawler)
        links, _ = crawler._extract_links("https://target.com/", html)
        self.assertIn("https://target.com/real-page", links)
        self.assertEqual(len(links), 1)


# test/modules/test_js_endpoint_extractor.py

class TestJSEndpointExtractor(unittest.TestCase):
    """Unit tests for the JS endpoint extraction patterns."""

    def test_fetch_pattern(self):
        """fetch("/api/users") should be extracted."""
        from artemis.modules.js_endpoint_extractor import JS_ENDPOINT_PATTERNS
        js = 'fetch("/api/v1/users")'
        matches = []
        for pattern in JS_ENDPOINT_PATTERNS:
            matches.extend(pattern.findall(js))
        self.assertIn("/api/v1/users", matches)

    def test_axios_pattern(self):
        """axios.post("/api/login") should be extracted."""
        from artemis.modules.js_endpoint_extractor import JS_ENDPOINT_PATTERNS
        js = 'axios.post("/api/login", data)'
        matches = []
        for pattern in JS_ENDPOINT_PATTERNS:
            matches.extend(pattern.findall(js))
        self.assertIn("/api/login", matches)

    def test_route_definition_pattern(self):
        """React Router path: "/dashboard" should be extracted."""
        from artemis.modules.js_endpoint_extractor import JS_ENDPOINT_PATTERNS
        js = '{path: "/dashboard/settings", component: Settings}'
        matches = []
        for pattern in JS_ENDPOINT_PATTERNS:
            matches.extend(pattern.findall(js))
        self.assertIn("/dashboard/settings", matches)

    def test_static_assets_filtered(self):
        """Static assets should not be considered valid endpoints."""
        from artemis.modules.js_endpoint_extractor import JSEndpointExtractor
        self.assertFalse(JSEndpointExtractor._is_valid_endpoint("/style.css"))
        self.assertFalse(JSEndpointExtractor._is_valid_endpoint("/node_modules/lib"))
        self.assertTrue(JSEndpointExtractor._is_valid_endpoint("/api/v2/accounts"))


# test/modules/test_target_deduplicator.py

class TestTargetDeduplicator(unittest.TestCase):
    """Unit tests for URL normalization and deduplication."""

    def test_normalize_strips_trailing_slash(self):
        from artemis.modules.target_deduplicator import TargetDeduplicator
        result = TargetDeduplicator.normalize_url("https://example.com/path/")
        self.assertEqual(result, "https://example.com/path")

    def test_normalize_sorts_query_params(self):
        from artemis.modules.target_deduplicator import TargetDeduplicator
        result = TargetDeduplicator.normalize_url("https://example.com/?b=2&a=1")
        self.assertEqual(result, "https://example.com/?a=1&b=2")

    def test_normalize_removes_tracking_params(self):
        from artemis.modules.target_deduplicator import TargetDeduplicator
        result = TargetDeduplicator.normalize_url(
            "https://example.com/page?id=5&utm_source=google&fbclid=abc"
        )
        self.assertEqual(result, "https://example.com/page?id=5")

    def test_path_pattern_groups_numeric_ids(self):
        from artemis.modules.target_deduplicator import TargetDeduplicator
        p1 = TargetDeduplicator.get_path_pattern("https://x.com/products/123")
        p2 = TargetDeduplicator.get_path_pattern("https://x.com/products/456")
        self.assertEqual(p1, p2)  # Both become /products/{id}

    def test_logout_urls_rejected(self):
        from artemis.modules.target_deduplicator import TargetDeduplicator
        self.assertFalse(TargetDeduplicator.is_safe_url("https://x.com/logout"))
        self.assertFalse(TargetDeduplicator.is_safe_url("https://x.com/sign-out"))
        self.assertTrue(TargetDeduplicator.is_safe_url("https://x.com/dashboard"))


# test/test_scheduler.py

class TestScheduler(unittest.TestCase):
    """Unit tests for the scheduling system."""

    def test_compute_next_run(self):
        """Cron '0 2 * * MON' should give next Monday 02:00."""
        from datetime import datetime, timezone
        from artemis.scheduler import compute_next_run

        # Friday 2026-03-13 10:00 UTC
        after = datetime(2026, 3, 13, 10, 0, tzinfo=timezone.utc)
        next_run = compute_next_run("0 2 * * MON", after)

        # Next Monday is March 16
        self.assertEqual(next_run.weekday(), 0)  # Monday
        self.assertEqual(next_run.hour, 2)
        self.assertEqual(next_run.day, 16)

    def test_invalid_cron_rejected(self):
        """Invalid cron expressions should raise ValueError."""
        from croniter import croniter
        self.assertFalse(croniter.is_valid("invalid cron"))
        self.assertTrue(croniter.is_valid("0 2 * * MON"))
        self.assertTrue(croniter.is_valid("*/5 * * * *"))


# test/test_notifications.py

class TestNotificationProcessor(unittest.TestCase):
    """Unit tests for the notification rule evaluation."""

    def test_severity_threshold(self):
        from artemis.notifications.processor import _severity_meets_threshold
        self.assertTrue(_severity_meets_threshold("high", "high"))
        self.assertTrue(_severity_meets_threshold("high", "medium"))
        self.assertTrue(_severity_meets_threshold("medium", "low"))
        self.assertFalse(_severity_meets_threshold("low", "high"))
        self.assertFalse(_severity_meets_threshold("medium", "high"))

    def test_dedup_key_deterministic(self):
        from artemis.notifications.processor import _dedup_key
        k1 = _dedup_key(1, "example.com", "SQL Injection")
        k2 = _dedup_key(1, "example.com", "SQL Injection")
        k3 = _dedup_key(1, "example.com", "XSS")
        self.assertEqual(k1, k2)  # Same inputs → same key
        self.assertNotEqual(k1, k3)  # Different finding → different key


if __name__ == "__main__":
    unittest.main()
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
