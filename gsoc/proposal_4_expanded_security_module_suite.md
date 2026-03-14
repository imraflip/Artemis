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

### Motivation
Reviewing the existing modules in `artemis/modules/`, Artemis has strong coverage for:
- CMS vulnerabilities (WordPress, Joomla, Drupal)
- Credential bruting (FTP, SSH, MySQL, PostgreSQL, WordPress admin)
- Code/data exposure (VCS, directory index, bruter for backup files)
- DNS issues (mail_dns_scanner, dangling DNS, domain expiration)
- General scanning (Nuclei integration, port scanner, Shodan)

But several vulnerability classes that appear frequently in real-world CSIRT reports are **not covered**:
- CORS misconfigurations enabling cross-origin data theft
- Missing/misconfigured HTTP security headers (beyond what `humble` does)
- Open redirects used in phishing campaigns
- Subdomain takeover via dangling DNS records pointing to unclaimed services
- API keys and secrets exposed in JavaScript files
- Exposed backup and configuration files (beyond current bruter wordlists)
- TLS/SSL configuration problems (old protocols, weak ciphers, expired certs)
- SSRF vulnerabilities via common injection points

Each of these is a well-understood vulnerability class with clear detection methodology, making them ideal for automated scanning at scale.

---

## Detailed Description

### Module Architecture Pattern

Every Artemis module follows this pattern (studied from `artemis/modules/vcs.py`, `artemis/modules/bruter.py`, etc.):

```python
@load_risk_class.load_risk_class(LoadRiskClass.LOW)  # LOW/MEDIUM/HIGH request volume
class MyScanner(ArtemisBase):
    identity = "my_scanner"                    # unique module ID
    filters = [
        {"type": TaskType.SERVICE.value, "service": Service.HTTP.value},
    ]

    def run(self, current_task: Task) -> None:
        url = get_target_url(current_task)
        # ... scanning logic ...
        self.db.save_task_result(
            task=current_task,
            status=TaskStatus.INTERESTING,     # or TaskStatus.OK
            status_reason="Description of finding",
            data=result_dict
        )

if __name__ == "__main__":
    MyScanner().loop()
```

Each module also needs:
1. **Reporter** in `artemis/reporting/modules/<name>/reporter.py` — converts `TaskResult` into `Report` objects
2. **Email template** in `artemis/reporting/modules/<name>/template_*.jinja2`
3. **Severity mapping** entry in `artemis/reporting/severity.py`
4. **Docker service** entry in `docker-compose.yaml`
5. **Unit tests** in `test/modules/test_<name>.py`
6. **Documentation** in `docs/`

### Module 1: CORS Misconfiguration Detector

**File:** `artemis/modules/cors_scanner.py`
**Binds:** `TaskType.SERVICE` + `Service.HTTP`
**Load Risk:** LOW

**Detection methodology:**
1. Send request with `Origin: https://evil.example.com` header
2. Check if response contains `Access-Control-Allow-Origin: https://evil.example.com` (reflected origin)
3. Send request with `Origin: null` — check for `Access-Control-Allow-Origin: null`
4. Check for `Access-Control-Allow-Credentials: true` combined with wildcard or reflected origin
5. Test common CORS bypass patterns: `Origin: https://target.com.evil.com`, `Origin: https://evil-target.com`

**Result data:**
```python
{
    "reflected_origin": True/False,
    "null_origin_allowed": True/False,
    "credentials_with_wildcard": True/False,
    "tested_origins": ["https://evil.example.com", ...],
    "vulnerable_endpoints": ["/api/v1/users", ...]
}
```

**Report types:** `cors_misconfiguration` (Severity: MEDIUM — can be HIGH with credentials)

### Module 2: HTTP Security Headers Analyzer

**File:** `artemis/modules/security_headers.py`
**Binds:** `TaskType.SERVICE` + `Service.HTTP`
**Load Risk:** LOW

**Note:** Artemis already has a `humble` module that wraps the Humble tool for header analysis. This module provides a **native, more configurable** implementation that checks:

1. **Content-Security-Policy (CSP):** Missing, or dangerously permissive (`unsafe-inline`, `unsafe-eval`, wildcard sources)
2. **Strict-Transport-Security (HSTS):** Missing, or short `max-age`, missing `includeSubDomains`
3. **X-Frame-Options:** Missing (clickjacking risk)
4. **Permissions-Policy:** Missing (allows sensors/features by default)
5. **Referrer-Policy:** Missing or set to `unsafe-url`
6. **X-Content-Type-Options:** Missing `nosniff`
7. **Cross-Origin-Opener-Policy / Cross-Origin-Resource-Policy:** Missing

**Differs from humble:**
- Pure Python (no external tool dependency)
- Reports individual header issues as separate findings with specific remediation
- Configurable via `ModuleRuntimeConfiguration` — choose which headers to check, minimum HSTS age
- Produces machine-readable results suitable for automated triage

**Result data:**
```python
{
    "missing_headers": ["Content-Security-Policy", "Permissions-Policy"],
    "misconfigured_headers": {
        "Strict-Transport-Security": {
            "value": "max-age=86400",
            "issue": "max-age too short (recommended: 31536000)"
        },
        "Content-Security-Policy": {
            "value": "default-src 'self' 'unsafe-inline'",
            "issue": "unsafe-inline allows XSS bypass"
        }
    }
}
```

**Report types:** `missing_security_headers_detailed`, `misconfigured_security_header` (Severity: LOW-MEDIUM)

### Module 3: Open Redirect Scanner

**File:** `artemis/modules/open_redirect.py`
**Binds:** `TaskType.URL` (requires crawled URLs)
**Load Risk:** LOW

**Detection methodology:**
1. Identify URL parameters that may be redirect targets: `url`, `redirect`, `next`, `return`, `returnTo`, `goto`, `continue`, `dest`, `destination`, `redir`, `redirect_uri`, `return_url`
2. For each parameter found in crawled URLs, replace the value with a canary URL (e.g., `https://evil.example.com`)
3. Follow the response — if the `Location` header or meta refresh points to the canary domain, it's an open redirect
4. Also check JavaScript-based redirects in response body (`window.location`, `document.location`)

**Handling false positives:**
- Verify the redirect actually leads to the injected domain (not just reflects it in page content)
- Check multiple canary patterns to avoid WAF-like transformations
- Don't report redirects to same-site URLs

**Result data:**
```python
{
    "vulnerable_urls": [
        {
            "original_url": "https://example.com/login?next=...",
            "parameter": "next",
            "redirect_type": "header",  # or "javascript"
        }
    ]
}
```

**Report types:** `open_redirect` (Severity: MEDIUM)

### Module 4: Subdomain Takeover Detector

**File:** `artemis/modules/subdomain_takeover.py`
**Binds:** `TaskType.DOMAIN`
**Load Risk:** LOW

**Note:** A version exists in `modules-extra` but is not in the core tree. This is a clean-room implementation.

**Detection methodology:**
1. Resolve the domain's CNAME chain
2. Check if the CNAME target belongs to a known cloud service fingerprint set:
   - **GitHub Pages:** CNAME to `*.github.io`, returns 404 with "There isn't a GitHub Pages site here"
   - **AWS S3:** CNAME to `*.s3.amazonaws.com`, returns "NoSuchBucket"
   - **Heroku:** CNAME to `*.herokuapp.com`, returns "No such app"
   - **Azure:** CNAME to `*.azurewebsites.net`, returns 404
   - **Shopify:** CNAME to `*.myshopify.com`, returns "Sorry, this shop is currently unavailable"
   - **Fastly:** CNAME to `*.fastly.net`, returns "Fastly error: unknown domain"
   - **Pantheon, Surge, Fly.io, etc.**
3. Attempt to connect and match the response body/status against known fingerprints
4. If CNAME points to a service that returns a "not configured" response, flag as potential takeover

**Fingerprint database** stored in `artemis/modules/data/subdomain_takeover_fingerprints.json`:
```json
[
    {
        "service": "GitHub Pages",
        "cname_patterns": ["*.github.io"],
        "response_fingerprints": ["There isn't a GitHub Pages site here"],
        "nxdomain": false,
        "severity": "high"
    },
    ...
]
```

**Result data:**
```python
{
    "vulnerable": True,
    "cname_chain": ["sub.example.com", "example.github.io"],
    "service": "GitHub Pages",
    "fingerprint_match": "There isn't a GitHub Pages site here"
}
```

**Report types:** `subdomain_takeover_possible` (Severity: HIGH — already defined in `severity.py`)

### Module 5: JavaScript Secrets Scanner

**File:** `artemis/modules/js_secrets_scanner.py`
**Binds:** `TaskType.URL` (operates on discovered URLs with `.js` extension)
**Load Risk:** LOW

**Detection methodology:**
1. From crawled URLs, identify JavaScript files
2. Fetch each JS file (using `self.http_get`)
3. Scan content against a regex pattern database for:
   - AWS Access Keys: `AKIA[0-9A-Z]{16}`
   - Google API Keys: `AIza[0-9A-Za-z\-_]{35}`
   - Slack Tokens: `xox[baprs]-[0-9a-zA-Z]{10,}`
   - Private Keys: `-----BEGIN (RSA |EC |DSA )?PRIVATE KEY-----`
   - Generic API keys: `api[_-]?key['":\s]*[=:]\s*['"][0-9a-zA-Z]{20,}['"]`
   - JWT tokens: `eyJ[A-Za-z0-9-_=]+\.eyJ[A-Za-z0-9-_=]+\.[A-Za-z0-9-_.+/=]*`
   - Database connection strings: `(mongodb|postgres|mysql)://[^'"}\s]+`
4. Filter false positives by checking entropy (high-entropy strings are more likely real secrets)
5. Exclude known public/placeholder values (e.g., example API keys from documentation)

**Pattern database** stored in `artemis/modules/data/js_secrets_patterns.py`:
```python
SECRET_PATTERNS = [
    SecretPattern(
        name="AWS Access Key ID",
        regex=r"AKIA[0-9A-Z]{16}",
        severity="high",
        min_entropy=3.0,
    ),
    ...
]
```

**Result data:**
```python
{
    "secrets_found": [
        {
            "type": "AWS Access Key ID",
            "file": "https://example.com/assets/app.js",
            "line": 142,
            "match_preview": "AKIA...XXXX",  # partially redacted
            "entropy": 4.2
        }
    ]
}
```

**Report types:** `exposed_js_secret` (Severity: HIGH)

### Module 6: Exposed Backup/Config File Scanner

**File:** `artemis/modules/backup_scanner.py`
**Binds:** `TaskType.SERVICE` + `Service.HTTP`
**Load Risk:** MEDIUM (makes multiple HTTP requests)

**This extends the existing `bruter.py` module** but focuses specifically on backup and configuration files with targeted detection logic (not just checking for 200 status).

**Scan targets:**
```
.env, .env.bak, .env.local, .env.production
config.php~, config.php.bak, config.php.old, config.php.swp
wp-config.php.bak, wp-config.php~, wp-config.php.old
database.yml, database.yml.bak
.htpasswd, .htaccess.bak
web.config.bak, web.config.old
backup.sql, dump.sql, database.sql, db.sql
backup.tar.gz, backup.zip, site.tar.gz, www.tar.gz
application.properties, application.yml
settings.py, settings.py.bak, local_settings.py
```

**Detection logic (beyond status code):**
- For `.env` files: check for `KEY=value` patterns (not just HTML error pages returning 200)
- For `.sql` files: check for SQL keywords like `CREATE TABLE`, `INSERT INTO`
- For archive files: check `Content-Type` header for `application/zip`, `application/gzip`
- For PHP backup files: check for `<?php` in content
- For config files: check for structured content (YAML, JSON, INI patterns)

**Result data:**
```python
{
    "exposed_files": [
        {
            "url": "https://example.com/.env",
            "type": "environment_file",
            "content_preview": "DB_HOST=...",  # first 200 chars, secrets redacted
            "contains_credentials": True
        }
    ]
}
```

**Report types:** `exposed_backup_file`, `exposed_config_file` (Severity: HIGH)

### Module 7: TLS/SSL Configuration Checker

**File:** `artemis/modules/tls_scanner.py`
**Binds:** `TaskType.SERVICE` + `Service.HTTP` (only SSL-enabled services)
**Load Risk:** LOW

**Detection methodology (using Python's `ssl` and `socket` modules):**
1. **Protocol version check:** Attempt connection with TLS 1.0 and TLS 1.1 — report if accepted
2. **Certificate validation:**
   - Check expiration date (report if expired or expiring within 30 days)
   - Check if self-signed
   - Check if certificate names match the domain
3. **Cipher suite analysis:**
   - Enumerate accepted cipher suites
   - Flag weak ciphers (RC4, DES, 3DES, export ciphers, NULL ciphers)
   - Flag missing forward secrecy (non-ECDHE/DHE ciphers)
4. **HSTS preload check:** Query the domain against the HSTS preload list

**Note:** This module intentionally overlaps slightly with Nuclei's TLS checks but provides structured, Artemis-native results that feed into the reporting pipeline with proper severity classification.

**Result data:**
```python
{
    "tls_issues": {
        "old_tls_versions": ["TLSv1.0", "TLSv1.1"],
        "weak_ciphers": ["TLS_RSA_WITH_RC4_128_SHA"],
        "certificate": {
            "expired": False,
            "expires_soon": True,
            "expiry_date": "2026-04-15",
            "self_signed": False,
            "name_mismatch": False
        },
        "missing_forward_secrecy": True
    }
}
```

**Report types:** `weak_tls_configuration`, `expired_ssl_certificate`, `almost_expired_ssl_certificate`, `bad_certificate_names` (Severity: LOW-MEDIUM — some already defined in `severity.py`)

### Module 8: SSRF Detection Module

**File:** `artemis/modules/ssrf_detector.py`
**Binds:** `TaskType.URL` (requires crawled URLs with parameters)
**Load Risk:** LOW

**Detection methodology:**
1. Identify URL parameters that may fetch external resources: `url`, `uri`, `path`, `src`, `href`, `file`, `load`, `fetch`, `proxy`, `page`, `callback`
2. For each candidate parameter, inject:
   - A callback URL pointing to an Artemis-controlled canary service (or DNS-based detection)
   - Internal network addresses: `http://127.0.0.1`, `http://169.254.169.254` (cloud metadata)
3. **DNS-based detection:** Generate a unique subdomain for each test (e.g., `<uuid>.ssrf-test.artemis.local`) and check DNS logs for resolution
4. **Response-based detection:** If injecting `http://127.0.0.1` returns different content than the original, potential SSRF

**Safety considerations:**
- Only test with benign payloads (canary domains, localhost)
- Rate-limited to avoid abuse
- Configurable: disabled by default, requires explicit opt-in via `ModuleRuntimeConfiguration`
- Document that this module actively probes targets and should only be used on authorized systems

**Result data:**
```python
{
    "ssrf_candidates": [
        {
            "url": "https://example.com/proxy?url=...",
            "parameter": "url",
            "detection_method": "dns_callback",  # or "response_diff"
            "payload": "https://<uuid>.ssrf-test.example.com"
        }
    ]
}
```

**Report types:** `ssrf_vulnerability` (Severity: HIGH)

---

## Cross-Cutting Concerns

### Reporting Integration
Each module's reporter follows the exact pattern established by existing reporters (e.g., `VCSReporter`):
- Check `task_result["headers"]["receiver"]` matches the module identity
- Check `task_result["status"] == "INTERESTING"`
- Create `Report` objects with proper `top_level_target`, `target`, `report_type`, `additional_data`
- Provide Jinja2 email template fragments

### Severity Mapping
New entries in `artemis/reporting/severity.py`:
```python
ReportType("cors_misconfiguration"): Severity.MEDIUM,
ReportType("missing_security_headers_detailed"): Severity.LOW,
ReportType("misconfigured_security_header"): Severity.MEDIUM,
ReportType("open_redirect"): Severity.MEDIUM,
ReportType("exposed_js_secret"): Severity.HIGH,
ReportType("exposed_backup_file"): Severity.HIGH,
ReportType("exposed_config_file"): Severity.HIGH,
ReportType("weak_tls_configuration"): Severity.MEDIUM,
ReportType("ssrf_vulnerability"): Severity.HIGH,
```

### Testing Strategy
Each module gets:
1. **Unit tests** in `test/modules/test_<name>.py` using mocked HTTP responses
2. **Test fixtures** with known-vulnerable and known-safe response data
3. **Reporter tests** verifying correct `Report` generation from sample `TaskResult` data

The test pattern follows existing modules (e.g., `test/modules/test_vcs.py`).

---

## Timeline and Deliverables

### Community Bonding Period (May 8 - June 1)
- Deep study of `ArtemisBase`, `TaskType`, `Service` binds, and module lifecycle
- Study existing modules: `vcs.py`, `bruter.py`, `humble.py` for patterns
- Research detection methodologies for each vulnerability class
- Build the subdomain takeover fingerprint database
- Submit a small module bugfix or enhancement as a warmup PR

### Phase 1: Weeks 1-4 (June 2 - June 29)

**Deliverable: Modules 1-3 (CORS, Security Headers, Open Redirect)**

| Week | Tasks |
|------|-------|
| 1 | Implement CORS scanner module + reporter + email template + unit tests |
| 2 | Implement Security Headers analyzer module + reporter + tests |
| 3 | Implement Open Redirect scanner module + reporter + tests |
| 4 | Integration testing of all 3 modules; add severity mappings; Docker entries |

### Midterm Evaluation (June 30 - July 4)

**Expected state:** 3 fully functional modules with reporters, email templates, unit tests, severity mappings, and Docker Compose entries. All follow established patterns and pass existing CI checks.

### Phase 2: Weeks 5-8 (July 5 - August 3)

**Deliverable: Modules 4-6 (Subdomain Takeover, JS Secrets, Backup Scanner)**

| Week | Tasks |
|------|-------|
| 5 | Implement Subdomain Takeover detector; build fingerprint database |
| 6 | Implement JS Secrets scanner; build pattern database with entropy filtering |
| 7 | Implement Backup/Config File scanner with content-type validation |
| 8 | Reporters, email templates, and comprehensive unit tests for modules 4-6 |

### Phase 3: Weeks 9-12 (August 4 - August 31)

**Deliverable: Modules 7-8 (TLS, SSRF), documentation, final polish**

| Week | Tasks |
|------|-------|
| 9 | Implement TLS/SSL checker module; handle edge cases in certificate validation |
| 10 | Implement SSRF detector module (opt-in, with safety controls) |
| 11 | Full integration testing; reporters and tests for modules 7-8; end-to-end scan test |
| 12 | Documentation for all 8 modules in `docs/`; update module list; final code review prep |

### Final Evaluation (September 1 - September 8)

---

## Technical Challenges and Mitigations

| Challenge | Mitigation |
|-----------|------------|
| False positives in CORS detection (CDNs reflect origin by design) | Check `Access-Control-Allow-Credentials` — only report when credentials are exposed |
| JS secrets scanner may flag placeholder keys | Entropy threshold filtering + exclude list of known test/example keys |
| Subdomain takeover fingerprints change over time | Store fingerprints in a data file (not hardcoded); make it easy to update |
| SSRF detection requires callback infrastructure | Use DNS-based detection (check for resolution of unique subdomains); fallback to response-diff |
| TLS testing requires low-level socket operations | Use Python's `ssl` module with custom contexts; handle connection timeouts gracefully |
| Backup scanner may trigger WAF blocks | Use `self.forgiving_http_get()` (existing pattern) for resilience to partial failures |

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
