# GSoC 2026 Proposal: Expanded Security Module Suite — Modern Web Attack Vectors for Artemis

## Contact Information

- **Name:** [Your Full Name]
- **Email:** [your.email@example.com]
- **GitHub:** [github.com/yourusername]
- **Timezone:** [e.g., UTC+2]
- **University / Program:** [Your University, Year, Major]

---

## Project Overview

### Title
Expanded Security Module Suite: Modern Web Attack Vectors

### Synopsis
Artemis currently includes 30+ scanning modules covering common vulnerabilities like exposed VCS folders, SQL injection, weak credentials, and CMS-specific issues. However, it lacks detection for several critical modern web vulnerability classes that CSIRT teams regularly encounter: CORS misconfigurations, HTTP security header analysis, open redirects, subdomain takeover, JavaScript secrets exposure, and TLS/SSL configuration issues. This project adds a suite of 8 new, high-value scanning modules — each with corresponding reporting modules, unit tests, and documentation — significantly expanding Artemis's detection capabilities.

### Coverage Gap Analysis

```
┌─────────────────────────────────────────────────────────────────────────┐
│            ARTEMIS VULNERABILITY DETECTION COVERAGE                     │
│                                                                         │
│  ✅ CURRENTLY COVERED               ❌ GAPS (THIS PROJECT FILLS)       │
│  ─────────────────────               ───────────────────────────        │
│  ✅ CMS vulns (WP, Joomla, Drupal)  ❌ CORS Misconfigurations          │
│  ✅ Credential bruting (FTP,SSH,DB)  ❌ HTTP Security Headers (native)  │
│  ✅ Exposed VCS (.git, .svn, .hg)   ❌ Open Redirects                  │
│  ✅ SQL Injection                    ❌ Subdomain Takeover              │
│  ✅ LFI Detection                   ❌ JavaScript Secrets               │
│  ✅ DNS misconfig (SPF, DMARC)      ❌ Backup/Config File Exposure     │
│  ✅ Port scanning                    ❌ TLS/SSL Configuration           │
│  ✅ Nuclei (generic CVEs)           ❌ SSRF Detection                   │
│  ✅ Directory indexing                                                  │
│  ✅ Domain expiration                                                   │
│                                                                         │
│  Coverage:  ██████████░░░░░░░  ~65%  →  ████████████████░  ~90%        │
│             Current                      After this project             │
└─────────────────────────────────────────────────────────────────────────┘
```

### Motivation
Reviewing the existing modules in `artemis/modules/`, Artemis has strong coverage for CMS vulnerabilities, credential bruting, code/data exposure, and DNS issues. But several vulnerability classes that appear frequently in real-world CSIRT reports are **not covered**. Each of the proposed 8 modules addresses a well-understood vulnerability class with clear detection methodology, making them ideal for automated scanning at scale.

---

## Detailed Description

### How Modules Fit Into Artemis's Karton Pipeline

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ARTEMIS SCANNING PIPELINE                            │
│                                                                         │
│  User submits ──► Classifier ──►──┬──► port_scanner ──► SERVICE tasks  │
│  target               │          │                         │           │
│  (NEW task)           │          ├──► subdomain_enum       │           │
│                        │          │                         ▼           │
│                        ▼          │                  ┌────────────┐     │
│                    DOMAIN task    │                  │ Existing   │     │
│                        │          │                  │ modules:   │     │
│                        │          │                  │ vcs,bruter │     │
│                        │          │                  │ nuclei,... │     │
│                        ▼          │                  └────────────┘     │
│               ┌────────────────┐  │                        │           │
│               │ NEW MODULE 4:  │  │                        │           │
│               │ subdomain_     │  │                        ▼           │
│               │ takeover       │  │              ┌──────────────────┐  │
│               └────────────────┘  │              │ NEW MODULES      │  │
│                                   │              │ (bind to SERVICE │  │
│                        ┌──────────┘              │  + HTTP):        │  │
│                        │                         │                  │  │
│                        ▼                         │ 1. cors_scanner  │  │
│                   URL tasks                      │ 2. security_hdrs │  │
│                        │                         │ 6. backup_scan   │  │
│                        ▼                         │ 7. tls_scanner   │  │
│               ┌────────────────┐                 └──────────────────┘  │
│               │ NEW MODULES    │                                       │
│               │ (bind to URL): │                                       │
│               │                │                                       │
│               │ 3. open_redirect│                                      │
│               │ 5. js_secrets  │                                       │
│               │ 8. ssrf_detect │                                       │
│               └────────────────┘                                       │
└─────────────────────────────────────────────────────────────────────────┘
```

### Module Architecture Pattern

Every Artemis module follows this pattern (studied from `artemis/modules/vcs.py`, `artemis/modules/bruter.py`, etc.):

```python
@load_risk_class.load_risk_class(LoadRiskClass.LOW)
class MyScanner(ArtemisBase):
    identity = "my_scanner"
    filters = [{"type": TaskType.SERVICE.value, "service": Service.HTTP.value}]

    def run(self, current_task: Task) -> None:
        url = get_target_url(current_task)
        # ... scanning logic ...
        self.db.save_task_result(
            task=current_task,
            status=TaskStatus.INTERESTING,
            status_reason="Description of finding",
            data=result_dict
        )

if __name__ == "__main__":
    MyScanner().loop()
```

**Each module requires these deliverables:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                PER-MODULE DELIVERABLES CHECKLIST                        │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  📁 artemis/modules/<name>.py          — Scanning logic         │   │
│  │  📁 artemis/reporting/modules/<name>/                           │   │
│  │     ├── reporter.py                    — TaskResult → Report    │   │
│  │     └── template_<type>.jinja2         — Email template         │   │
│  │  📁 artemis/reporting/severity.py      — Add severity mapping   │   │
│  │  📁 docker-compose.yaml                — Add container service  │   │
│  │  📁 test/modules/test_<name>.py        — Unit tests             │   │
│  │  📁 docs/                              — Module documentation   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  Total per module: ~6 files, ~300-500 lines                             │
│  Total for project: ~48 files, ~3000-4000 lines                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### Module Overview Table

| # | Module | File | Binds To | Load Risk | Severity | Detection Method |
|---|--------|------|----------|-----------|----------|------------------|
| 1 | CORS Scanner | `cors_scanner.py` | SERVICE+HTTP | LOW | MEDIUM | Reflected origin, null origin, credentials check |
| 2 | Security Headers | `security_headers.py` | SERVICE+HTTP | LOW | LOW-MED | Missing/misconfigured CSP, HSTS, X-Frame, etc. |
| 3 | Open Redirect | `open_redirect.py` | URL | LOW | MEDIUM | Parameter injection with canary URLs |
| 4 | Subdomain Takeover | `subdomain_takeover.py` | DOMAIN | LOW | HIGH | CNAME chain + cloud service fingerprints |
| 5 | JS Secrets Scanner | `js_secrets_scanner.py` | URL | LOW | HIGH | Regex patterns + entropy filtering |
| 6 | Backup Scanner | `backup_scanner.py` | SERVICE+HTTP | MEDIUM | HIGH | Path probing + content-type validation |
| 7 | TLS/SSL Checker | `tls_scanner.py` | SERVICE+HTTP | LOW | LOW-MED | Protocol version, ciphers, certificate checks |
| 8 | SSRF Detector | `ssrf_detector.py` | URL | LOW | HIGH | DNS callback + response diff analysis |

---

### Module 1: CORS Misconfiguration Detector

```
┌─────────────────────────────────────────────────────────────────────┐
│  CORS SCANNER — DETECTION FLOW                                      │
│                                                                     │
│  Target URL                                                         │
│      │                                                              │
│      ├──► Test 1: Origin: https://evil.example.com                  │
│      │    Check: ACAO header reflects evil.example.com?              │
│      │    ┌──────────────┐                                          │
│      │    │ Response:    │  YES → 🔴 Reflected Origin               │
│      │    │ ACAO: evil.. │  NO  → Continue                          │
│      │    └──────────────┘                                          │
│      │                                                              │
│      ├──► Test 2: Origin: null                                      │
│      │    Check: ACAO: null?                                        │
│      │    YES → 🟡 Null Origin Allowed                              │
│      │                                                              │
│      ├──► Test 3: Check Access-Control-Allow-Credentials: true      │
│      │    Combined with wildcard/reflected?                          │
│      │    YES → 🔴 Credentials + Wildcard (upgrade to HIGH)        │
│      │                                                              │
│      └──► Test 4: Bypass patterns                                   │
│           Origin: https://target.com.evil.com                       │
│           Origin: https://evil-target.com                           │
│           Reflected? → 🔴 CORS Bypass                               │
│                                                                     │
│  Output: {reflected_origin, null_allowed, creds_with_wildcard,      │
│           tested_origins[], vulnerable_endpoints[]}                  │
└─────────────────────────────────────────────────────────────────────┘
```

**Report types:** `cors_misconfiguration` (Severity: MEDIUM, HIGH with credentials)

### Module 2: HTTP Security Headers Analyzer

```
┌──────────────────────────────────────────────────────────────────────┐
│  SECURITY HEADERS — CHECK MATRIX                                     │
│                                                                      │
│  Header                      │ Check                  │ Severity    │
│  ────────────────────────────┼────────────────────────┼────────────  │
│  Content-Security-Policy     │ Missing?               │ MEDIUM      │
│                              │ unsafe-inline?         │ MEDIUM      │
│                              │ unsafe-eval?           │ MEDIUM      │
│                              │ wildcard sources?      │ LOW         │
│  ────────────────────────────┼────────────────────────┼────────────  │
│  Strict-Transport-Security   │ Missing?               │ MEDIUM      │
│                              │ max-age < 31536000?    │ LOW         │
│                              │ No includeSubDomains?  │ LOW         │
│  ────────────────────────────┼────────────────────────┼────────────  │
│  X-Frame-Options             │ Missing?               │ MEDIUM      │
│  Permissions-Policy          │ Missing?               │ LOW         │
│  Referrer-Policy             │ Missing or unsafe-url? │ LOW         │
│  X-Content-Type-Options      │ Missing nosniff?       │ LOW         │
│  Cross-Origin-Opener-Policy  │ Missing?               │ LOW         │
│  Cross-Origin-Resource-Policy│ Missing?               │ LOW         │
│                                                                      │
│  Differs from existing `humble` module:                              │
│  • Pure Python (no external tool dependency)                         │
│  • Individual findings with specific remediation                     │
│  • Configurable via ModuleRuntimeConfiguration                       │
│  • Machine-readable output for automated triage                      │
└──────────────────────────────────────────────────────────────────────┘
```

### Module 3: Open Redirect Scanner

```
┌─────────────────────────────────────────────────────────────────────┐
│  OPEN REDIRECT — DETECTION FLOW                                      │
│                                                                     │
│  Crawled URL: https://example.com/login?next=/dashboard             │
│                                                                     │
│  Step 1: Identify redirect parameters                               │
│  ┌───────────────────────────────────────────────────────┐          │
│  │ Param Names:  url, redirect, next, return, returnTo,  │          │
│  │               goto, continue, dest, destination,       │          │
│  │               redir, redirect_uri, return_url          │          │
│  └───────────────────────────────────────────────────────┘          │
│         │ Found: "next" parameter                                   │
│         ▼                                                           │
│  Step 2: Inject canary URL                                          │
│  ┌───────────────────────────────────────────────────────┐          │
│  │ GET /login?next=https://evil.example.com               │          │
│  └───────────────────────────────────────────────────────┘          │
│         │                                                           │
│         ▼                                                           │
│  Step 3: Check response                                             │
│  ┌────────────────────────┬──────────────────────────────┐          │
│  │ Location header        │ → Redirect to evil.example?  │          │
│  │ Meta refresh           │ → Points to evil.example?    │          │
│  │ JS (window.location)   │ → Sets evil.example?         │          │
│  └────────────────────────┴──────────────────────────────┘          │
│         │                                                           │
│     YES │                         NO                                │
│         ▼                          ▼                                │
│  ┌──────────────┐          ┌──────────────┐                        │
│  │ 🟡 OPEN      │          │    SAFE      │                        │
│  │    REDIRECT   │          │   (skip)     │                        │
│  └──────────────┘          └──────────────┘                        │
│                                                                     │
│  False Positive Mitigations:                                        │
│  • Verify redirect actually reaches injected domain                 │
│  • Exclude same-site redirects                                      │
│  • Test multiple canary patterns for WAF bypass                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Module 4: Subdomain Takeover Detector

```
┌─────────────────────────────────────────────────────────────────────┐
│  SUBDOMAIN TAKEOVER — DETECTION FLOW                                 │
│                                                                     │
│  Input: sub.example.com (DOMAIN task)                               │
│         │                                                           │
│         ▼                                                           │
│  ┌──────────────────────────────┐                                   │
│  │ Step 1: Resolve CNAME chain │                                   │
│  │ sub.example.com              │                                   │
│  │  └─ CNAME: example.github.io│                                   │
│  └──────────────┬───────────────┘                                   │
│                 │                                                    │
│                 ▼                                                    │
│  ┌──────────────────────────────────────────────────────────┐       │
│  │ Step 2: Match against fingerprint database                │       │
│  │                                                          │       │
│  │  Service         │ CNAME Pattern        │ Fingerprint    │       │
│  │  ────────────────┼──────────────────────┼───────────────  │       │
│  │  GitHub Pages    │ *.github.io          │ "There isn't  │       │
│  │                  │                      │  a GitHub..." │       │
│  │  AWS S3          │ *.s3.amazonaws.com   │ "NoSuchBucket"│       │
│  │  Heroku          │ *.herokuapp.com      │ "No such app" │       │
│  │  Azure           │ *.azurewebsites.net  │ 404 page      │       │
│  │  Shopify         │ *.myshopify.com      │ "Sorry, this  │       │
│  │                  │                      │  shop is..."  │       │
│  │  Fastly          │ *.fastly.net         │ "Fastly error"│       │
│  │  Pantheon        │ *.pantheonsite.io    │ 404 + pattern │       │
│  │  Surge.sh        │ *.surge.sh           │ "project not  │       │
│  │                  │                      │  found"       │       │
│  └──────────────────┼──────────────────────┼────────────────┘       │
│                     │ MATCH FOUND          │                        │
│                     ▼                      │                        │
│  ┌──────────────────────────────┐          │                        │
│  │ Step 3: Fetch & confirm     │          │                        │
│  │ Response matches fingerprint│          │                        │
│  └──────────────┬───────────────┘          │                        │
│            YES  │                     NO   │                        │
│                 ▼                          ▼                        │
│          ┌──────────────┐          ┌──────────┐                    │
│          │ 🔴 TAKEOVER  │          │   SAFE   │                    │
│          │    POSSIBLE   │          │          │                    │
│          └──────────────┘          └──────────┘                    │
└─────────────────────────────────────────────────────────────────────┘
```

### Module 5: JavaScript Secrets Scanner

```
┌─────────────────────────────────────────────────────────────────────┐
│  JS SECRETS SCANNER — DETECTION PIPELINE                             │
│                                                                     │
│  Crawled URLs                                                       │
│      │                                                              │
│      ▼                                                              │
│  ┌──────────────────────┐                                           │
│  │ Filter: *.js files   │                                           │
│  │ app.js, bundle.js,   │                                           │
│  │ vendor.js, ...       │                                           │
│  └──────────┬───────────┘                                           │
│             │                                                       │
│             ▼                                                       │
│  ┌──────────────────────────────────────────────────────────┐       │
│  │ Pattern Database (regex + metadata)                       │       │
│  │                                                          │       │
│  │  Pattern                    │ Type          │ Severity   │       │
│  │  ──────────────────────────┼───────────────┼──────────   │       │
│  │  AKIA[0-9A-Z]{16}          │ AWS Access Key│ 🔴 HIGH    │       │
│  │  AIza[0-9A-Za-z\-_]{35}    │ Google API Key│ 🔴 HIGH    │       │
│  │  xox[baprs]-[0-9a-zA-Z]+  │ Slack Token   │ 🔴 HIGH    │       │
│  │  -----BEGIN.*PRIVATE KEY   │ Private Key   │ 🔴 HIGH    │       │
│  │  eyJ[A-Za-z0-9-_=]+\.eyJ  │ JWT Token     │ 🟡 MEDIUM  │       │
│  │  (mongodb|postgres)://     │ DB Conn String│ 🔴 HIGH    │       │
│  │  api[_-]?key.*['"]{20,}   │ Generic API   │ 🟡 MEDIUM  │       │
│  └──────────────────────────────────────────────────────────┘       │
│             │                                                       │
│             ▼                                                       │
│  ┌──────────────────────┐     ┌──────────────────────┐              │
│  │ Entropy Filter       │     │ Exclude List         │              │
│  │                      │     │                      │              │
│  │ Calculate Shannon    │     │ Known test keys:     │              │
│  │ entropy of match     │     │ • AKIAIOSFODNN7...   │              │
│  │                      │     │ • AIzaSyExample...   │              │
│  │ min_entropy=3.0      │     │ • xoxb-example-...   │              │
│  │ (filters out low-    │     │                      │              │
│  │  entropy placeholders│     │ Skip if exact match  │              │
│  │  like "AAAA...")     │     │ to known placeholder │              │
│  └──────────┬───────────┘     └──────────┬───────────┘              │
│             │                            │                          │
│             └──────────┬─────────────────┘                          │
│                        ▼                                            │
│  ┌──────────────────────────────────────┐                           │
│  │ Output: Partially redacted matches   │                           │
│  │ {type, file, line, "AKIA...XXXX"}    │                           │
│  └──────────────────────────────────────┘                           │
└─────────────────────────────────────────────────────────────────────┘
```

### Module 6: Exposed Backup/Config File Scanner

```
┌─────────────────────────────────────────────────────────────────────┐
│  BACKUP SCANNER — INTELLIGENT CONTENT VALIDATION                     │
│                                                                     │
│  Unlike naive 200-status checking, this module validates content:    │
│                                                                     │
│  File Type        │ Probe Path          │ Validation Logic          │
│  ─────────────────┼─────────────────────┼─────────────────────────  │
│  .env             │ /.env, /.env.bak    │ Contains KEY=value        │
│                   │ /.env.local         │ patterns (not HTML)       │
│  ─────────────────┼─────────────────────┼─────────────────────────  │
│  PHP config       │ /config.php~        │ Contains <?php            │
│                   │ /wp-config.php.bak  │                           │
│  ─────────────────┼─────────────────────┼─────────────────────────  │
│  SQL dump         │ /backup.sql         │ Contains CREATE TABLE     │
│                   │ /dump.sql           │ or INSERT INTO            │
│  ─────────────────┼─────────────────────┼─────────────────────────  │
│  Archives         │ /backup.tar.gz      │ Content-Type matches      │
│                   │ /backup.zip         │ application/zip or gzip   │
│  ─────────────────┼─────────────────────┼─────────────────────────  │
│  Config files     │ /application.yml    │ Valid YAML/JSON/INI       │
│                   │ /settings.py        │ structure detected        │
│  ─────────────────┼─────────────────────┼─────────────────────────  │
│  Credentials      │ /.htpasswd          │ username:hash format      │
│                   │ /.htaccess.bak      │                           │
│                                                                     │
│  False positive prevention:                                         │
│  ┌────────────────────────────────────────────────────────┐         │
│  │  200 status ──► Content validation ──► Not HTML error? │         │
│  │  (not enough)    (required)             (required)      │         │
│  └────────────────────────────────────────────────────────┘         │
└─────────────────────────────────────────────────────────────────────┘
```

### Module 7: TLS/SSL Configuration Checker

```
┌─────────────────────────────────────────────────────────────────────┐
│  TLS SCANNER — CHECK CATEGORIES                                      │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Category 1: Protocol Versions                                │    │
│  │ ┌──────────┬──────────┬──────────┬──────────┬──────────┐    │    │
│  │ │ SSLv3    │ TLS 1.0  │ TLS 1.1  │ TLS 1.2  │ TLS 1.3  │    │    │
│  │ │ 🔴 FAIL  │ 🔴 FAIL  │ 🟡 WARN  │ ✅ OK    │ ✅ OK    │    │    │
│  │ └──────────┴──────────┴──────────┴──────────┴──────────┘    │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Category 2: Certificate                                      │    │
│  │                                                              │    │
│  │  Check              │ Result Example        │ Severity       │    │
│  │  ────────────────────┼───────────────────────┼──────────      │    │
│  │  Expired?           │ Expired 2026-01-15    │ 🟡 MEDIUM     │    │
│  │  Expiring soon?     │ Expires in 12 days    │ 🔵 LOW        │    │
│  │  Self-signed?       │ No CA chain           │ 🔵 LOW        │    │
│  │  Name mismatch?     │ CN=other.com          │ 🔵 LOW        │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Category 3: Cipher Suites                                    │    │
│  │                                                              │    │
│  │  ✅ GOOD: TLS_AES_256_GCM_SHA384                            │    │
│  │  ✅ GOOD: TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256             │    │
│  │  🔴 WEAK: TLS_RSA_WITH_RC4_128_SHA                          │    │
│  │  🔴 WEAK: TLS_RSA_WITH_3DES_EDE_CBC_SHA                     │    │
│  │  🟡 NO FORWARD SECRECY: TLS_RSA_WITH_AES_256_CBC_SHA        │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

### Module 8: SSRF Detection Module

```
┌─────────────────────────────────────────────────────────────────────┐
│  SSRF DETECTOR — DETECTION METHODOLOGY                               │
│                                                                     │
│  ⚠️ This module is OPT-IN (disabled by default) — it actively      │
│  probes targets and should only be used on authorized systems.       │
│                                                                     │
│  Crawled URL with parameters                                        │
│      │                                                              │
│      ▼                                                              │
│  ┌──────────────────────────────────────────────┐                   │
│  │ Step 1: Identify candidate parameters         │                   │
│  │ url, uri, path, src, href, file, load, fetch, │                   │
│  │ proxy, page, callback                         │                   │
│  └──────────────────────┬───────────────────────┘                   │
│                         │                                           │
│          ┌──────────────┴──────────────┐                            │
│          │                             │                            │
│          ▼                             ▼                            │
│  ┌───────────────────┐       ┌───────────────────┐                  │
│  │ Method A:          │       │ Method B:          │                  │
│  │ DNS Callback       │       │ Response Diff      │                  │
│  │                    │       │                    │                  │
│  │ Inject:            │       │ Inject:            │                  │
│  │ <uuid>.ssrf-test.  │       │ http://127.0.0.1   │                  │
│  │ artemis.local      │       │                    │                  │
│  │                    │       │ Compare response   │                  │
│  │ Check DNS logs     │       │ with original      │                  │
│  │ for resolution     │       │                    │                  │
│  └────────┬──────────┘       └────────┬──────────┘                  │
│           │                           │                             │
│     DNS query         Response differs                              │
│     observed?         significantly?                                │
│           │                           │                             │
│      YES  │                      YES  │                             │
│           ▼                           ▼                             │
│    ┌──────────────┐            ┌──────────────┐                     │
│    │ 🔴 SSRF      │            │ 🟡 Potential │                     │
│    │    CONFIRMED  │            │    SSRF       │                     │
│    └──────────────┘            └──────────────┘                     │
│                                                                     │
│  Safety Controls:                                                   │
│  ┌─────────────────────────────────────────────────────────┐        │
│  │ • Disabled by default (opt-in via RuntimeConfiguration) │        │
│  │ • Only benign payloads (canary domains, localhost)       │        │
│  │ • Rate-limited via existing REQUESTS_PER_SECOND          │        │
│  │ • Documented as active probing module                    │        │
│  └─────────────────────────────────────────────────────────┘        │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Severity Mapping

New entries in `artemis/reporting/severity.py`:

| Report Type | Severity | Justification |
|-------------|----------|---------------|
| `cors_misconfiguration` | MEDIUM | Data theft via cross-origin requests |
| `cors_misconfiguration_with_credentials` | HIGH | Credential theft possible |
| `missing_security_headers_detailed` | LOW | Defense-in-depth, not direct exploit |
| `misconfigured_security_header` | MEDIUM | CSP bypass enables XSS |
| `open_redirect` | MEDIUM | Used in phishing campaigns |
| `subdomain_takeover_possible` | HIGH | Full domain control (already in severity.py) |
| `exposed_js_secret` | HIGH | Credential exposure |
| `exposed_backup_file` | HIGH | Source code / data exposure |
| `exposed_config_file` | HIGH | Credential exposure |
| `weak_tls_configuration` | MEDIUM | Downgrade attack risk |
| `expired_ssl_certificate` | LOW | Already in severity.py |
| `ssrf_vulnerability` | HIGH | Internal network access |

---

## Timeline and Deliverables

### Gantt Chart — Module Development Schedule

```
        May         June           July           August         Sep
       Week: 1 2 3 4 1 2 3 4 5 6 7 8 9 10 11 12  1
             ├─┤                                      Community Bonding
             │CB│
                   ├─────────┤                        Phase 1: Modules 1-3
                   │M1│M2│M3│Integration│
                             ├───┤                    ◆ Midterm Eval
                                  ├─────────┤         Phase 2: Modules 4-6
                                  │M4│M5│M6│Reporters│
                                              ├──────┤Phase 3: Modules 7-8
                                              │M7│M8│E2E│Docs│
                                                     ├┤ Final Eval

Legend: M1-M8 = Module 1-8, CB = Community Bonding
```

### Development Cadence Per Module

```
┌─────────────────────────────────────────────────────────────────┐
│  TYPICAL MODULE DEVELOPMENT CYCLE (~35-45 hours each)           │
│                                                                 │
│  Day 1-2:  Research & design detection methodology              │
│            ├─ Study vulnerability class                         │
│            └─ Define test cases (vulnerable + safe)             │
│                                                                 │
│  Day 3-5:  Implement scanning module                            │
│            ├─ Write scanner class extending ArtemisBase          │
│            ├─ Implement detection logic                         │
│            └─ Handle edge cases & false positives               │
│                                                                 │
│  Day 6-7:  Implement reporter + email template                  │
│            ├─ TaskResult → Report conversion                    │
│            ├─ Jinja2 email template fragment                    │
│            └─ Severity mapping entry                            │
│                                                                 │
│  Day 8-9:  Testing                                              │
│            ├─ Unit tests with mocked HTTP responses              │
│            ├─ Reporter tests with sample TaskResults             │
│            └─ Docker service entry                              │
│                                                                 │
│  Day 10:   Documentation + code review prep                     │
└─────────────────────────────────────────────────────────────────┘
```

### Phase-by-Phase Breakdown

| Phase | Weeks | Modules | Deliverables | Hours |
|-------|-------|---------|-------------|-------|
| **Community Bonding** | May 8 - Jun 1 | — | Dev env setup, study patterns, build fingerprint DB, warmup PR | 25 |
| **Phase 1** | 1-4 | CORS, SecHeaders, OpenRedirect | 3 modules + reporters + tests + Docker entries | 120 |
| **Midterm** | — | — | 3 fully functional modules passing CI | — |
| **Phase 2** | 5-8 | SubdomainTakeover, JSSecrets, BackupScanner | 3 modules + reporters + tests + fingerprint/pattern DBs | 120 |
| **Phase 3** | 9-12 | TLS, SSRF + documentation | 2 modules + E2E tests + docs for all 8 modules | 85 |
| **Total** | | **8 modules** | ~48 files, ~3500 lines, full test coverage | **350** |

---

## Effort Breakdown

```
                    Effort Distribution (350 hours)

  ┌────────────────────────────────────────────────────────┐
  │                                                        │
  │  Module 1: CORS Scanner      ████░░░░░░░░░░░  35h     │
  │  Module 2: Security Headers  ████░░░░░░░░░░░  40h     │
  │  Module 3: Open Redirect     ████░░░░░░░░░░░  35h     │
  │  Module 4: Subdomain Takeover████░░░░░░░░░░░  45h     │
  │  Module 5: JS Secrets        █████░░░░░░░░░░  45h     │
  │  Module 6: Backup Scanner    ████░░░░░░░░░░░  40h     │
  │  Module 7: TLS/SSL Checker   █████░░░░░░░░░░  45h     │
  │  Module 8: SSRF Detector     ████░░░░░░░░░░░  35h     │
  │  Integration Testing         ██░░░░░░░░░░░░░  15h     │
  │  Documentation               ██░░░░░░░░░░░░░  15h     │
  │                                                        │
  │  Each module includes: scanner + reporter + template    │
  │  + severity mapping + Docker entry + unit tests         │
  └────────────────────────────────────────────────────────┘
```

---

## Technical Challenges and Mitigations

| Challenge | Risk | Mitigation |
|-----------|------|------------|
| False positives in CORS detection (CDNs reflect origin) | Medium | Only report when `Access-Control-Allow-Credentials: true` is present |
| JS secrets scanner flags placeholder keys | Medium | Entropy threshold filtering + exclude list of known test keys |
| Subdomain takeover fingerprints change over time | Low | Store in JSON data file, not hardcoded; easy to update |
| SSRF detection requires callback infrastructure | High | DNS-based detection (no server needed); fallback to response-diff |
| TLS testing requires low-level socket operations | Low | Python `ssl` module with custom contexts; handle timeouts gracefully |
| Backup scanner may trigger WAF blocks | Medium | Use `self.forgiving_http_get()` for resilience to partial failures |
| 8 modules is a lot of code to maintain | Low | Each module is independent; follow existing patterns exactly |
| Testing requires diverse vulnerable targets | Medium | Create test fixtures with mocked HTTP responses in `test/data/` |

---

## Relevant Skills and Experience

- [Describe your Python experience]
- [Describe your experience with web security / penetration testing]
- [Describe your familiarity with vulnerability classes (OWASP Top 10)]
- [Describe any experience writing security tools or scanners]
- [Link to relevant projects/contributions, CTF writeups, etc.]

---

## Why This Project?

Adding new detection modules directly increases the number and types of vulnerabilities Artemis can discover. For CSIRT teams, this translates to better coverage of their constituency's attack surface. Each of the 8 proposed modules addresses a vulnerability class that is:
- Common in real-world deployments
- Well-understood with clear detection methodology
- Valuable for CSIRT teams to find proactively (before attackers do)

The modular architecture of Artemis makes this project well-scoped — each module is independent and can be developed, tested, and released incrementally.

---

## Contributions to Artemis (Pre-GSoC)

- [List any PRs, issues, or discussions you've contributed to]
- [If none yet, describe your plan to contribute before the application deadline]

---

## Availability

- I can commit approximately **30-35 hours per week** during the coding period
- [Mention any planned vacations, exams, or other commitments]
- I am available for weekly sync calls with my mentor
