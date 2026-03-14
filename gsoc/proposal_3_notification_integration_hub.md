# GSoC 2026 Proposal: Notification & Integration Hub for Artemis

## Contact Information

- **Name:** [Your Full Name]
- **Email:** [your.email@example.com]
- **GitHub:** [github.com/yourusername]
- **Timezone:** [e.g., UTC+2]
- **University / Program:** [Your University, Year, Major]

---

## Project Overview

### Title
Notification & Integration Hub (Webhooks, Slack, Email, Jira)

### Synopsis
Artemis discovers vulnerabilities continuously, but there is currently no mechanism to proactively alert teams when critical findings are discovered. CSIRT operators must manually check the dashboard or wait for scheduled report exports. This project builds a configurable notification and integration layer that enables real-time alerts via Slack, Microsoft Teams, Email, generic webhooks, and Jira ticket creation — driven by a flexible rules engine that lets operators define exactly when and how they want to be notified.

### Motivation
Artemis's `autoreporter` system (`artemis/reporting/export/`) generates periodic HTML reports. However:

- **No real-time alerts:** When a critical vulnerability (e.g., exposed `.git` repository with credentials) is found, there's no immediate notification. The finding sits in the database until the next report export cycle.
- **No workflow integration:** CSIRT teams commonly use Slack, Microsoft Teams, or Jira for their day-to-day operations. Without integration, findings exist in a silo.
- **No customizable alerting rules:** Different teams care about different things. A web security team may want all high-severity findings, while a DNS team may only want email misconfigurations.
- **No notification history:** There's no record of what was communicated to whom and when.

The existing `artemis/reporting/export/hooks.py` provides a basic hook mechanism for report export, but it only runs during batch report generation — not in real-time as findings are discovered.

---

## Detailed Description

### Architecture

```
TaskResult saved (status=INTERESTING)
        │
        ▼
  NotificationEngine (new Karton module)
        │
        ├── Evaluates rules against finding
        │
        ├──► SlackBackend      → Slack incoming webhook
        ├──► TeamsBackend      → MS Teams incoming webhook
        ├──► EmailBackend      → SMTP with HTML template
        ├──► WebhookBackend    → Generic HTTP POST
        └──► JiraBackend       → Jira REST API
```

The system integrates at the point where `ArtemisBase.db.save_task_result()` stores findings (in `artemis/module_base.py`). A new Karton module consumes notification events and processes them asynchronously.

### Component 1: Notification Rules Engine

**Data model** — new tables in `artemis/db.py`:

```python
class NotificationRule(Base):
    __tablename__ = "notification_rule"

    id = Column(Integer, primary_key=True)
    name = Column(String)
    enabled = Column(Boolean, default=True)
    created_at = Column(DateTime, server_default=text("NOW()"))

    # Matching criteria
    tag_pattern = Column(String, nullable=True)          # glob pattern, e.g. "client_*"
    severity_threshold = Column(String, nullable=True)   # "high", "medium", "low"
    module_filter = Column(String, nullable=True)        # comma-separated module names
    report_type_filter = Column(String, nullable=True)   # comma-separated report types

    # Delivery configuration
    channel_type = Column(String)    # "slack", "teams", "email", "webhook", "jira"
    channel_config = Column(JSONB)   # channel-specific config (URL, API key, etc.)

    # Behavior
    mode = Column(String, default="immediate")  # "immediate" or "digest"
    digest_schedule = Column(String, nullable=True)  # cron expression for digest mode
    deduplicate = Column(Boolean, default=True)


class NotificationLog(Base):
    __tablename__ = "notification_log"

    id = Column(Integer, primary_key=True)
    rule_id = Column(Integer, ForeignKey("notification_rule.id"), index=True)
    task_result_id = Column(String, ForeignKey("task_result.id"), index=True)
    created_at = Column(DateTime, server_default=text("NOW()"))
    channel_type = Column(String)
    status = Column(String)           # "sent", "failed", "retried"
    error_message = Column(String, nullable=True)
    retry_count = Column(Integer, default=0)

    # Deduplication key — hash of (report_type, target_normal_form)
    dedup_key = Column(String, index=True)
```

**Rule evaluation logic:**

```python
def matches_rule(finding: TaskResult, rule: NotificationRule) -> bool:
    if rule.tag_pattern and not fnmatch(finding.tag, rule.tag_pattern):
        return False
    if rule.severity_threshold:
        finding_severity = get_severity_for_task_result(finding)
        if not severity_meets_threshold(finding_severity, rule.severity_threshold):
            return False
    if rule.module_filter:
        allowed = rule.module_filter.split(",")
        if finding.receiver not in allowed:
            return False
    return True
```

### Component 2: Notification Backends

Each backend implements a common interface:

```python
class NotificationBackend(ABC):
    @abstractmethod
    def send(self, finding: TaskResult, rule: NotificationRule) -> bool:
        """Send a notification. Returns True on success."""
        pass

    @abstractmethod
    def send_digest(self, findings: List[TaskResult], rule: NotificationRule) -> bool:
        """Send a digest notification. Returns True on success."""
        pass
```

**Slack Backend:**
- Uses Slack Incoming Webhooks (no OAuth needed, simple setup)
- Sends a rich Block Kit message with severity color coding, target URL, finding details, and a link back to the Artemis UI
- Digest mode sends a summary message with counts by severity

**Microsoft Teams Backend:**
- Uses Incoming Webhook connector (similar to Slack)
- Sends Adaptive Card format with severity, target, and details
- Digest mode sends a table-format summary

**Email Backend:**
- Uses SMTP (configurable host/port/TLS)
- Reuses Artemis's existing Jinja2 email template infrastructure from `artemis/reporting/base/templating.py`
- HTML email with finding details, styled consistently with existing report emails
- Digest mode sends a daily/weekly summary email

**Generic Webhook Backend:**
- HTTP POST to a user-configured URL
- JSON payload with standardized schema:
  ```json
  {
    "event": "new_finding",
    "timestamp": "2026-...",
    "finding": {
      "id": "...",
      "target": "...",
      "report_type": "...",
      "severity": "high",
      "status_reason": "...",
      "tag": "..."
    },
    "artemis_url": "https://artemis.example.com/task/..."
  }
  ```
- Supports custom headers (for API keys)
- Configurable retry with exponential backoff

**Jira Backend:**
- Creates issues via Jira REST API v2/v3
- Configurable project key, issue type, priority mapping
- Includes finding details in the description
- **Deduplication:** Before creating, searches for existing issues with the same dedup key in a custom field — updates instead of creating duplicates
- When a finding is resolved in Artemis (if lifecycle management is also implemented), optionally transitions the Jira issue

### Component 3: Notification Processing Pipeline

**Karton module:** `NotificationProcessor`

```python
class NotificationProcessor(ArtemisBase):
    identity = "notification_processor"

    # Runs on a timer, not on task binds
    def loop(self):
        while True:
            # Check for new INTERESTING task results since last check
            new_findings = self.get_new_findings_since_last_check()

            for finding in new_findings:
                for rule in self.get_enabled_rules():
                    if matches_rule(finding, rule):
                        self.process_notification(finding, rule)

            # Process digest rules
            self.process_pending_digests()

            time.sleep(Config.Notifications.POLL_INTERVAL_SECONDS)
```

**Alternative approach:** Instead of polling, inject a notification hook directly into `ArtemisBase.db.save_task_result()` that publishes to a Redis pub/sub channel. The `NotificationProcessor` subscribes and processes in near real-time.

**Deduplication:**
- Before sending, compute a dedup key: `hash(report_type + target_normal_form)`
- Check `NotificationLog` for recent entries with the same dedup key and rule
- Skip if already notified within a configurable window (default: 7 days)

**Retry logic:**
- On failure, log the error and schedule a retry
- Exponential backoff: 1 min, 5 min, 30 min, 2 hours
- Maximum 5 retries before marking as permanently failed

### Component 4: Configuration UI

New pages accessible from the main navigation:

**Notification Rules Page (`/notifications/rules`):**
- List of all rules with enable/disable toggle
- Create/edit form:
  - Rule name
  - Tag pattern (with autocomplete from existing tags)
  - Severity threshold (dropdown: High, Medium, Low)
  - Module filter (multi-select from available modules)
  - Channel type (radio: Slack / Teams / Email / Webhook / Jira)
  - Channel-specific configuration (dynamic form based on channel type)
  - Mode: Immediate / Digest (with schedule picker for digest)
  - Test button: sends a test notification

**Notification History Page (`/notifications/history`):**
- Filterable log of all sent notifications
- Shows: timestamp, rule name, finding target, channel, status
- Click to see full notification payload and any error details

### Component 5: Docker Compose Integration

New service in `docker-compose.yaml`:

```yaml
notification-processor:
  <<: *artemis-build-or-image
  <<: *logging
  restart: always
  depends_on:
    - postgres
    - redis
  env_file: .env
  command: python3 -m artemis.notification_processor
```

New configuration variables in `artemis/config.py`:

```python
class Notifications:
    ENABLE_NOTIFICATIONS = get_config("ENABLE_NOTIFICATIONS", default=False, cast=bool)
    POLL_INTERVAL_SECONDS = get_config("NOTIFICATION_POLL_INTERVAL_SECONDS", default=60, cast=int)
    SMTP_HOST = get_config("NOTIFICATION_SMTP_HOST", default="")
    SMTP_PORT = get_config("NOTIFICATION_SMTP_PORT", default=587, cast=int)
    SMTP_USERNAME = get_config("NOTIFICATION_SMTP_USERNAME", default="")
    SMTP_PASSWORD = get_config("NOTIFICATION_SMTP_PASSWORD", default="")
    SMTP_FROM = get_config("NOTIFICATION_SMTP_FROM", default="")
```

---

## Timeline and Deliverables

### Community Bonding Period (May 8 - June 1)
- Study Artemis's Karton task flow and `ArtemisBase` module pattern
- Set up test environments for Slack, Email (MailHog), and Jira (local instance)
- Design the notification payload schema
- Draft the rules engine matching logic and get mentor feedback
- Submit a small PR (e.g., documentation improvement)

### Phase 1: Weeks 1-4 (June 2 - June 29)

**Deliverable: Rules engine, Slack and Email backends, basic UI**

| Week | Tasks |
|------|-------|
| 1 | Implement `NotificationRule` and `NotificationLog` models; Alembic migration; DB methods |
| 2 | Build the rules evaluation engine; implement Slack backend with Block Kit messages |
| 3 | Implement Email backend with HTML templates (reusing existing template infrastructure) |
| 4 | Build notification rules configuration UI; add test notification functionality |

### Midterm Evaluation (June 30 - July 4)

**Expected state:** Working notification system where operators can create rules to get Slack and Email alerts for high-severity findings. Rules UI for configuration and a test button. Notification log tracking delivery status.

### Phase 2: Weeks 5-8 (July 5 - August 3)

**Deliverable: Webhook, Teams, Jira backends; digest mode; deduplication**

| Week | Tasks |
|------|-------|
| 5 | Implement generic Webhook backend; implement Microsoft Teams backend |
| 6 | Implement Jira backend with issue creation, deduplication, and update |
| 7 | Build digest mode: accumulate findings, send on schedule (cron-based) |
| 8 | Implement deduplication logic; retry mechanism with exponential backoff |

### Phase 3: Weeks 9-12 (August 4 - August 31)

**Deliverable: Notification history, Docker integration, testing, documentation**

| Week | Tasks |
|------|-------|
| 9 | Build notification history UI with filtering and search |
| 10 | Docker Compose integration; add configuration to `config.py`; environment variable docs |
| 11 | Comprehensive testing: unit tests with mocked APIs, integration tests with MailHog |
| 12 | Documentation: setup guide for each channel, admin guide, architecture docs |

### Final Evaluation (September 1 - September 8)

---

## Technical Challenges and Mitigations

| Challenge | Mitigation |
|-----------|------------|
| External API rate limits (Slack, Jira) | Implement per-channel rate limiting with token bucket algorithm |
| Notification storms on large scan completion | Deduplication + digest mode + configurable cooldown period |
| Sensitive channel configs (API keys) in DB | Encrypt `channel_config` JSONB values using Fernet symmetric encryption |
| Testing external integrations in CI | Use mock servers (MailHog for SMTP, WireMock for HTTP) in test environment |
| Jira dedup requires searching for existing issues | Use JQL query with custom field; cache results to reduce API calls |

---

## Notification Message Examples

### Slack Message (Block Kit)

```
🔴 High Severity Finding — Artemis
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Target: https://example.com/.git/config
Type: Exposed Version Control Folder
Tag: client_production
Module: vcs

Found exposed .git repository with potential
credentials.

[View in Artemis]  [Mark as False Positive]
```

### Digest Email

```
Subject: [Artemis] Weekly Digest — 12 new findings (3 High)

This week's scan summary for tag "client_production":
• 3 High severity findings (2 new)
• 5 Medium severity findings (1 new)
• 4 Low severity findings (all new)

Top findings:
1. 🔴 Exposed .git with credentials — example.com
2. 🔴 SQL injection detected — app.example.com/api/v1
3. 🟡 Outdated WordPress plugin — blog.example.com
...

View all findings: https://artemis.example.com/?tag=client_production
```

---

## Relevant Skills and Experience

- [Describe your Python experience]
- [Describe your experience with HTTP APIs (REST, webhooks)]
- [Describe your experience with messaging systems (Slack API, email/SMTP)]
- [Describe any experience with async processing or task queues]
- [Link to relevant projects/contributions]

---

## Why This Project?

Real-time notifications are the bridge between vulnerability discovery and remediation action. Without them, critical findings can sit unnoticed for hours or days. By integrating Artemis with the tools CSIRT teams already use (Slack, email, Jira), this project embeds security scanning results directly into existing workflows. The digest mode and deduplication ensure that operators aren't overwhelmed with noise, while the rules engine gives each team precise control over what they see.

This project is also highly extensible — the backend interface makes it easy for future contributors to add new channels (Discord, PagerDuty, MISP, etc.).

---

## Contributions to Artemis (Pre-GSoC)

- [List any PRs, issues, or discussions you've contributed to]
- [If none yet, describe your plan to contribute before the application deadline]

---

## Availability

- I can commit approximately **30-35 hours per week** during the coding period
- [Mention any planned vacations, exams, or other commitments]
- I am available for weekly sync calls with my mentor
