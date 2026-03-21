# Artemis Codebase Bug Audit

**Date:** 2026-03-21
**Scope:** Full codebase audit of all Python source, templates, JS, and migration files

---

## CRITICAL (12 bugs)

### 1. Closure captures loop variable — tasks silently run with wrong input
**File:** `artemis/module_base.py:691`
The lambda `lambda: self.run(task)` inside `for task in task_group` captures `task` by reference. By the time `timeout_decorator` invokes the lambda, `task` may have advanced to the next iteration value. In task groups with >1 task, all iterations execute `self.run()` with the **last** task, silently skipping earlier tasks.

### 2. Lock released without ever being acquired — releases other processes' locks
**File:** `artemis/module_base.py:846, 859`
`check_connection_to_base_url_and_save_error` creates a `ResourceLock` on line 818 but never calls `lock.acquire()`. On WAF detection (line 846) or connection error (line 859), `lock.release()` is called, which deletes the Redis key unconditionally (`REDIS.delete(self.res_name)`). This releases a lock that may be held by **another** process/thread scanning the same destination.

### 3. Redis keys are `bytes` — `str.startswith()` always fails
**File:** `artemis/cleanup.py:29, 37`
`backend.redis.keys()` returns `bytes` keys by default. `key.startswith("karton.task")` compares `bytes` against `str`, which either raises `TypeError` or always returns `False` (depending on redis-py version). The entire cleanup routine silently does nothing.

### 4. `PORTS_SET_SHORT` undefined when custom ports are configured — `NameError` crash
**File:** `artemis/modules/port_scanner.py:71`
`PORTS_SET_SHORT` is only defined inside the `else` branch (when no custom ports). But it's referenced unconditionally on line 189 for IPs with too many open ports. When custom ports are configured, this crashes with `NameError`.

### 5. `process_json_data()` called on a list — `AttributeError` crash
**File:** `artemis/modules/humble.py:135`
When humble returns empty/no findings, `result` is set to `[]` (line 132). `process_json_data(result)` then calls `result.items()` (line 32), which crashes: `AttributeError: 'list' object has no attribute 'items'`.

### 6. Unhandled DNS exceptions crash mail scanner
**File:** `artemis/modules/mail_dns_scanner.py:58, 64`
`dns.resolver.resolve(domain, "MX")` can raise `NXDOMAIN`, `NoNameservers`, or `Timeout`. Only `NoAnswer` is caught. Any of these common DNS errors crashes the scanner.

### 7. `gethostbyaddr()` crash — unhandled `socket.herror`/`gaierror`
**File:** `artemis/modules/reverse_dns_lookup.py:46`
`gethostbyaddr(ip)` raises `socket.herror` or `socket.gaierror` for IPs without PTR records. These are not caught, crashing the module for any IP without reverse DNS.

### 8. Time-based SQLi detection uses truncated integer seconds
**File:** `artemis/modules/sql_injection_detector.py:109`
`datetime.timedelta(seconds=timer() - start).seconds` returns only whole seconds. A 4.9-second response returns `4`, a 0.8-second response returns `0`. This makes the time-based SQL injection detection unreliable (both false positives and false negatives).

### 9. `resolvers.lookup()` raises `NotImplementedError` for AAAA/MX queries
**File:** `artemis/resolvers.py:68`
`_single_resolution_attempt` only handles `query_type` "A" and "NS". It raises `NotImplementedError` for "AAAA" and "MX", which are used by other modules (e.g., `module_base.py:264`).

### 10. Mutable default argument causes cross-invocation data leakage in report export
**File:** `artemis/reporting/export/main.py:169`
`custom_template_arguments: Dict[str, str] = {}` — the same dict is shared across all calls. `build_export_data` mutates it (sets `min_vulnerability_date_str`, `max_vulnerability_date_str`), so subsequent calls see stale date values from previous exports.

### 11. Archiver has no exception handling — any error kills the service
**File:** `artemis/autoarchiver/autoarchiver.py:88-106`
The `main()` infinite loop has no `try/except`. Any exception (DB error, filesystem error, disk full) terminates the archiver permanently.

### 12. Archiver data loss race condition
**File:** `artemis/autoarchiver/autoarchiver.py:40-47`
Items are written to a gzip file, then deleted from the database one-by-one. If the process crashes mid-deletion, some records are lost from the DB while the archive may be incomplete. No transaction wraps the deletions.

---

## MEDIUM (30 bugs)

### 13. `IPv4Address` hardcoded in blocklist — crashes on IPv6
**File:** `artemis/blocklist.py:140, 214`
`ipaddress.IPv4Address(ip)` is used unconditionally. Any IPv6 address causes a `ValueError` crash. The `ip_range` field supports IPv6Network, but the address check does not.

### 14. API returns 200 OK for invalid targets instead of error status
**File:** `artemis/api.py:89`
When validation fails in the `add` endpoint, a `200 OK` response with `{"error": ...}` is returned instead of `HTTPException`. Callers interpret this as success.

### 15. Unvalidated column index from query params — `IndexError`
**File:** `artemis/api.py:377`
`int(request.query_params[column_key])` indexes into `column_names` without bounds checking. A malformed DataTables request crashes the API.

### 16. `session.query().get()` returns None — `AttributeError` crashes
**File:** `artemis/db.py:254, 461, 466, 626`
Multiple functions call `.get(id)` and immediately access attributes on the result without null checks: `mark_analysis_as_stopped`, `delete_analysis`, `delete_task_result`, `delete_report_generation_task`.

### 17. Path traversal in report deletion
**File:** `artemis/db.py:627`
`output_location = "/opt/" + task.output_location` followed by `shutil.rmtree(output_location)`. The assertion guard (line 630) is disabled with `python -O`. A crafted `output_location` with `../` could delete arbitrary directories.

### 18. SSRF via karton dashboard proxy
**File:** `artemis/frontend.py:408`
The `path` parameter is directly concatenated into a proxy URL to `karton-dashboard:5000`. URL manipulation characters could redirect requests.

### 19. `lru_cache` on function with `**kwargs` — `TypeError` crash
**File:** `artemis/reporting/utils.py:86-92`
`@functools.lru_cache` on `cached_get(url, **kwargs)`. Calling with any keyword argument crashes: `TypeError: unhashable type: 'dict'`.

### 20. `lru_cache` key mismatch in DNS lookup
**File:** `artemis/resolvers.py:80-93`
`domain = domain.lower()` runs AFTER the cache key is computed. `lookup("Example.com")` and `lookup("example.com")` cache as separate entries with potentially different results.

### 21. `retrying_resolver.retry()` raises stale exception on falsy valid result
**File:** `artemis/retrying_resolver.py:42-43`
If `result` is a valid but falsy value (e.g., empty set from NXDOMAIN), `all([not result, last_exception])` raises a stale exception from a prior failed attempt.

### 22. Timezone-naive vs timezone-aware datetime crash
**File:** `artemis/modules/domain_expiration_scanner.py:63-65`
`datetime.datetime.now()` is naive. If WHOIS returns a timezone-aware `expiration_date`, subtraction raises `TypeError`.

### 23. Zero days-to-expire treated as falsy — domain expiring today not flagged
**File:** `artemis/modules/domain_expiration_scanner.py:68`
`if days_to_expire and days_to_expire <= ...` — `0` is falsy. A domain expiring today is not reported. Should be `if days_to_expire is not None`.

### 24. Drupal version parsing broken for multi-`=` URLs
**File:** `artemis/modules/drupal_scanner.py:41`
`split("=")[1]` on `?v=9.5.0&extra=foo` returns `9.5.0&extra` instead of `9.5.0`.

### 25. `IPWhois(None)` crash when domain can't be resolved
**File:** `artemis/modules/dangling_dns_detector.py:362-364`
`get_ip_from_domain()` can return `None`. `IPWhois(None)` raises an exception.

### 26. IDNA encoding crash for invalid domain names
**File:** `artemis/modules/classifier.py:61`
`data.encode("idna")` raises `UnicodeError` for certain domains. Not caught.

### 27. MySQL/PostgreSQL connections leaked — never closed
**File:** `artemis/modules/mysql_bruter.py:43`, `artemis/modules/postgresql_bruter.py:43`
Successful `connect()` calls return connection objects that are immediately discarded. Connections are never closed, leaking database connections.

### 28. Race condition on shared user_agents.txt file
**File:** `artemis/modules/humble.py:93-99`
Multiple concurrent tasks can overwrite each other's `user_agents.txt` backup/restore, corrupting user agent strings.

### 29. FTP `prot_p()` called before login — protocol violation
**File:** `artemis/modules/ftp_bruter.py:77`
`prot_p()` sets data channel protection, which requires prior authentication per FTP RFC. Some servers reject this.

### 30. Incorrect regex in URL parameter detection
**File:** `artemis/modules/sql_injection_detector.py:69` (also `artemis/modules/lfi_detector.py:60`)
Regex `"/?/*="` — `*` in regex means "zero or more of preceding". This matches any string containing `=`, not just URLs with query parameters.

### 31. Duplicate `"headers"` key in dict literal
**File:** `artemis/modules/sql_injection_detector.py:418-425, 449-456`
Dictionary has `"headers": headers` twice. The second silently overwrites the first. The first was likely meant to be a different key.

### 32. `all([])` returns True — false positive SQLi on zero retries
**File:** `artemis/modules/sql_injection_detector.py:302, 383, 448`
If `SQL_INJECTION_NUM_RETRIES_TIME_BASED` is 0, `flags` is empty and `all([])` is `True`, falsely reporting SQL injection.

### 33. Port scanner line parsing crashes on IPv6
**File:** `artemis/modules/port_scanner.py:169`
`ip, port_str = line.split(":")` — IPv6 addresses contain multiple colons, causing `ValueError`.

### 34. `eol` field can be `bool` — comparison with `datetime.date` crashes
**File:** `artemis/modules/base/base_newer_version_comparer.py:59`
`release["eol"]` from endoflife.date can be `False`. `False <= datetime.now().date()` raises `TypeError`.

### 35. Nuclei assertion crashes entire batch processing
**File:** `artemis/modules/nuclei.py:710`
`assert found` aborts processing of ALL tasks in a batch if any single finding can't be matched.

### 36. Nuclei DAST template containment check is substring-based instead of list membership
**File:** `artemis/modules/nuclei.py:267`
`template in s` checks substring containment within strings, not list membership. Templates could be incorrectly excluded from the "other" category.

### 37. Malformed robots.txt crashes module
**File:** `artemis/modules/robots.py:65, 69`
`ValueError` raised for `allow`/`disallow` rules before `user-agent:`. Not caught — many real-world robots.txt files are malformed.

### 38. WordPress bruter uses display name instead of login slug
**File:** `artemis/modules/wordpress_bruter.py:33`
`user_entry["name"]` gets display name (e.g., "John Smith"), not login username (`"slug"` field like "jsmith").

### 39. Shodan API error crashes module
**File:** `artemis/modules/shodan_vulns.py:35`
`shodan_client.host(ip)` can raise `shodan.APIError` (e.g., "No info for IP"). Not caught.

### 40. XSS via `|safe` filter on scan results
**File:** `templates/view_export.jinja2:38, 70`
`{{ message|safe }}` and `{{ vulnerability.html|safe }}` render unsanitized HTML. Scan results from attacker infrastructure could contain injected JavaScript.

### 41. File handle leak in `load_previous_reports`
**File:** `artemis/reporting/export/previous_reports.py:16`
`json.load(open(path))` — file opened without `with` or `.close()`. Leaks a file descriptor per JSON file.

### 42. `min()`/`max()` crash on reports with all-None timestamps
**File:** `artemis/reporting/export/export_data.py:80-81`
`min([r.timestamp for r in reports if r.timestamp])` crashes with `ValueError: min() arg is an empty sequence` if all timestamps are None.

---

## LOW (20 bugs)

### 43. `None` href crash in bruter classifier
**File:** `artemis/reporting/modules/bruter/classifier.py:51`
`link.get("href")` can return `None`. Subsequent `"access.log" in href` raises `TypeError`.

### 44. BeautifulSoup missing parser argument (multiple files)
**Files:** `artemis/modules/drupal_scanner.py:36`, `artemis/reporting/export/main.py:79,85`, `artemis/reporting/modules/bruter/classifier.py:47`
`BeautifulSoup(html)` without `"html.parser"` produces `GuessedAtParserWarning` and non-deterministic behavior.

### 45. Global monkey-patching race condition in vhost checker
**File:** `artemis/modules/removed_domain_existing_vhost.py:70-75`
`connection.create_connection` is globally replaced, then restored in `finally`. Concurrent calls clobber each other's patches.

### 46. HTML `<option>` uses `name` instead of `value` attribute
**Files:** `templates/export.jinja2:39`, `templates/add.jinja2:50`
`<option name="...">` is invalid HTML. The `name` attribute has no effect on `<option>` elements. Should be `value`.

### 47. Stale variable in port scanner error message
**File:** `artemis/modules/port_scanner.py:200`
Error log uses `line` (from previous loop) instead of current `ip:port_str`.

### 48. `ssl` module shadowed by local variable
**File:** `artemis/modules/port_scanner.py:208`
`ssl = data["tls"]` shadows the `ssl` module import.

### 49. `rstrip("s")` strips all trailing s characters
**File:** `artemis/modules/classifier.py:237`
`service.rstrip("s")` strips ALL trailing 's' chars, not just one. "address" → "addre".

### 50. Temp directory from `base_newer_version_comparer` never cleaned up
**File:** `artemis/modules/base/base_newer_version_comparer.py:19`
`tempfile.mkdtemp()` creates a directory that persists forever on success.

### 51. Fragile JSON parsing via quote replacement
**File:** `artemis/modules/joomla_extensions.py:49`
`json.loads(line.replace("'", '"'))` corrupts strings containing apostrophes.

### 52. `strip_trailing_zeros` can return empty string for "0.0.0"
**File:** `artemis/modules/wordpress_plugins.py:83-90`
Stripping all trailing zero segments from "0.0.0" produces `""`, a falsy value.

### 53. Credential logging in admin panel bruter
**File:** `artemis/modules/admin_panel_login_bruter.py:135, 224`
Passwords logged in plaintext via `self.log.info`.

### 54. Exception re-wrapping in classifier
**File:** `artemis/modules/classifier.py:35-36`
`except Exception` catches `RIPEAccessException` and wraps it in another `RIPEAccessException`.

### 55. FTP files list accumulates duplicates
**File:** `artemis/modules/ftp_bruter.py:90`
`result.files.extend(ftp.nlst())` called per successful login — duplicates if multiple credentials work.

### 56. Mutable default arguments (multiple locations)
**Files:** `artemis/producer.py:18`, `artemis/db.py:591`, `artemis/modules/nuclei.py:349`
`List/Dict = []` or `= {}` as default arguments. Mutation would affect all subsequent calls.

### 57. Migration creates DB engine at import time
**Files:** `migrations/versions/99b5570a348e_initial_migration.py:24`, `migrations/versions/40355237ae7c_tag_migration.py:21`
`sa.create_engine(DATABASE_URL)` at module level creates unnecessary connections during Alembic version scanning.

### 58. Report `__eq__` without `__hash__` makes Reports unhashable
**File:** `artemis/reporting/base/report.py:96-105`
Python 3 sets `__hash__ = None` when `__eq__` is defined. Reports can't be used in sets or as dict keys.

### 59. `subprocess.call` return code ignored for translation compilation
**File:** `artemis/reporting/export/translations.py:70`
`pybabel compile` failure is silent; downstream `.mo` file access then crashes.

### 60. Nuclei reporter JSON file loaded at import time
**File:** `artemis/reporting/modules/nuclei/reporter.py:40-41`
`json.load()` at class definition time. Missing/malformed file prevents the entire module from importing.

### 61. `None` concatenation crash in removed_domain_existing_vhost reporter
**File:** `artemis/reporting/modules/removed_domain_existing_vhost/reporter.py:32`
`"www." + task_result["payload_persistent"].get("original_domain")` — if `get()` returns `None`, this is `"www." + None` → `TypeError`.

### 62. Filename collision when shortening long target names
**File:** `artemis/reporting/export/main.py:140-148`
Long `top_level_target` strings truncated to same prefix overwrite each other's output files, causing data loss.

---

## Summary

| Severity | Count |
|----------|-------|
| Critical | 12    |
| Medium   | 30    |
| Low      | 20    |
| **Total**| **62**|

### Top Priority Fixes
1. **#1** — Lambda closure bug: tasks silently processed with wrong input
2. **#2** — Lock release without acquire: breaks distributed locking
3. **#3** — Redis bytes/str mismatch: cleanup routine completely broken
4. **#5** — humble module crashes on empty results
5. **#6, #7** — DNS/socket exceptions crash scanners
6. **#8** — SQLi detector timing truncation: unreliable detection
7. **#10** — Mutable default dict: cross-invocation state leakage in exports
