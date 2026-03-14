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

### The Gap

```
┌─────────────────────────────────────────────────────────────────────┐
│  CURRENT: Finding sits silently in DB until manual check            │
│                                                                     │
│  Module finds vuln ──► DB stores TaskResult ──► ??? ──► Operator   │
│                                                   │      checks    │
│                                               Hours or             │
│                                               days later           │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│  PROPOSED: Instant alert through team's existing tools              │
│                                                                     │
│  Module finds vuln ──► DB stores TaskResult ──► Rules Engine ──►   │
│                                                     │              │
│                                     ┌───────────────┼───────────┐  │
│                                     ▼               ▼           ▼  │
│                                  📱 Slack      📧 Email     🎫 Jira│
│                                  (seconds)    (minutes)    (auto)  │
└─────────────────────────────────────────────────────────────────────┘
```

### Motivation
Artemis's `autoreporter` system (`artemis/reporting/export/`) generates periodic HTML reports. However:

- **No real-time alerts:** When a critical vulnerability (e.g., exposed `.git` repository with credentials) is found, there's no immediate notification. The finding sits in the database until the next report export cycle.
- **No workflow integration:** CSIRT teams commonly use Slack, Microsoft Teams, or Jira for their day-to-day operations. Without integration, findings exist in a silo.
- **No customizable alerting rules:** Different teams care about different things. A web security team may want all high-severity findings, while a DNS team may only want email misconfigurations.
- **No notification history:** There's no record of what was communicated to whom and when.

The existing `artemis/reporting/export/hooks.py` provides a basic hook mechanism for report export, but it only runs during batch report generation — not in real-time as findings are discovered.

---

## Detailed Description

### End-to-End Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     NOTIFICATION SYSTEM ARCHITECTURE                    │
│                                                                         │
│  ┌──────────────┐                                                       │
│  │  Scanning     │                                                       │
│  │  Modules      │                                                       │
│  │  (Karton)     │                                                       │
│  │               │                                                       │
│  │  vcs ─────┐   │                                                       │
│  │  nuclei ──┤   │                                                       │
│  │  bruter ──┤   │                                                       │
│  │  ...      │   │                                                       │
│  └───────────┤───┘                                                       │
│              │                                                           │
│              ▼ save_task_result(status=INTERESTING)                       │
│  ┌──────────────────┐                                                    │
│  │   PostgreSQL      │                                                    │
│  │   ┌────────────┐  │      ┌────────────────────────────────────────┐   │
│  │   │ TaskResult  │  │      │      NotificationProcessor (Karton)   │   │
│  │   │   (new!)    │──┼─────►│                                        │   │
│  │   └────────────┘  │ poll │  1. Query new INTERESTING results      │   │
│  │                    │      │  2. Load enabled NotificationRules     │   │
│  │   ┌────────────┐  │      │  3. Evaluate rules against findings    │   │
│  │   │Notification│  │◄─────│  4. Dispatch to backends               │   │
│  │   │   Rule     │  │      │  5. Log results                        │   │
│  │   └────────────┘  │      │                                        │   │
│  │                    │      └───────────┬────────────────────────────┘   │
│  │   ┌────────────┐  │                  │                                │
│  │   │Notification│  │◄─────────────────┘ log delivery                   │
│  │   │   Log      │  │                  │                                │
│  │   └────────────┘  │      ┌───────────┴────────────────────────────┐   │
│  └──────────────────┘      │          BACKEND DISPATCHERS            │   │
│                             │                                        │   │
│                             │  ┌─────────┐  ┌─────────┐  ┌────────┐ │   │
│                             │  │  Slack   │  │  Teams  │  │ Email  │ │   │
│                             │  │ Backend  │  │ Backend │  │Backend │ │   │
│                             │  └────┬────┘  └────┬────┘  └───┬────┘ │   │
│                             │       │            │            │      │   │
│                             │  ┌────┴────┐  ┌────┴────┐            │   │
│                             │  │ Webhook │  │  Jira   │            │   │
│                             │  │ Backend │  │ Backend │            │   │
│                             │  └────┬────┘  └────┬────┘            │   │
│                             └───────┼────────────┼──────────────────┘   │
│                                     │            │                      │
└─────────────────────────────────────┼────────────┼──────────────────────┘
                                      ▼            ▼
                              External Services (Slack, Teams, SMTP, Jira, Webhooks)
```

### Notification Processing Pipeline

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    NOTIFICATION PROCESSING FLOW                         │
│                                                                         │
│                        ┌─────────────────┐                              │
│                        │  New TaskResult  │                              │
│                        │  (INTERESTING)   │                              │
│                        └────────┬────────┘                              │
│                                 │                                       │
│                                 ▼                                       │
│                    ┌────────────────────────┐                           │
│                    │  Load Enabled Rules    │                           │
│                    │  from notification_rule│                           │
│                    └────────────┬───────────┘                           │
│                                 │                                       │
│                    ┌────────────▼───────────┐                           │
│              ┌─────┤  For each rule:        │                           │
│              │     │  Does finding match?   │                           │
│              │     └────────────────────────┘                           │
│              │                                                          │
│         ┌────▼────┐      ┌──────────┐                                  │
│         │  Tag    │ NO   │  Skip    │                                  │
│         │ match?  ├─────►│  rule    │                                  │
│         └────┬────┘      └──────────┘                                  │
│              │ YES                                                      │
│         ┌────▼─────┐     ┌──────────┐                                  │
│         │ Severity │ NO  │  Skip    │                                  │
│         │ meets    ├────►│  rule    │                                  │
│         │threshold?│     └──────────┘                                  │
│         └────┬─────┘                                                   │
│              │ YES                                                      │
│         ┌────▼─────┐     ┌──────────┐                                  │
│         │ Module   │ NO  │  Skip    │                                  │
│         │ in       ├────►│  rule    │                                  │
│         │ filter?  │     └──────────┘                                  │
│         └────┬─────┘                                                   │
│              │ YES                                                      │
│         ┌────▼──────────┐                                              │
│         │  Check dedup  │                                              │
│         │  key in log   │                                              │
│         └────┬──────────┘                                              │
│              │                                                          │
│      ┌───────┴───────┐                                                  │
│      │               │                                                  │
│   Already         New finding                                           │
│   notified            │                                                 │
│      │          ┌─────▼─────────┐                                       │
│      ▼          │  Mode?        │                                       │
│   [Skip]        └──┬────────┬──┘                                       │
│                    │        │                                            │
│              Immediate   Digest                                         │
│                    │        │                                            │
│                    ▼        ▼                                            │
│              ┌─────────┐ ┌──────────┐                                   │
│              │  Send   │ │ Queue for│                                   │
│              │  NOW    │ │ digest   │                                   │
│              │         │ │ batch    │                                   │
│              └────┬────┘ └──────────┘                                   │
│                   │                                                     │
│              ┌────▼──────────┐                                          │
│              │  Log result   │                                          │
│              │  (sent/failed)│                                          │
│              └────┬──────────┘                                          │
│                   │                                                     │
│            ┌──────┴──────┐                                              │
│            │             │                                              │
│          Success      Failed                                            │
│            │             │                                              │
│            ▼             ▼                                              │
│      [Log: sent]   [Schedule retry]                                     │
│                    1m → 5m → 30m → 2h                                   │
│                    (max 5 retries)                                       │
└─────────────────────────────────────────────────────────────────────────┘
```

### Component 1: Database Schema

**Entity Relationship Diagram:**

```
┌──────────────────────┐         ┌──────────────────────┐
│   NotificationRule   │         │   NotificationLog    │
├──────────────────────┤         ├──────────────────────┤
│ id (PK)              │◄───┐    │ id (PK)              │
│ name                 │    │    │ rule_id (FK) ────────│───►
│ enabled              │    │    │ task_result_id (FK)   │───► TaskResult
│ created_at           │    │    │ created_at            │
│                      │    │    │ channel_type          │
│ tag_pattern          │    │    │ status                │
│ severity_threshold   │    └────│ (sent/failed/retried) │
│ module_filter        │         │ error_message         │
│ report_type_filter   │         │ retry_count           │
│                      │         │ dedup_key             │
│ channel_type         │         └──────────────────────┘
│ channel_config (JSONB)│
│                      │         ┌──────────────────────┐
│ mode                 │         │   DigestQueue        │
│ (immediate/digest)   │         ├──────────────────────┤
│ digest_schedule      │         │ id (PK)              │
│ deduplicate          │         │ rule_id (FK) ────────│───►
└──────────────────────┘         │ task_result_id (FK)  │
                                 │ queued_at            │
                                 │ processed            │
                                 └──────────────────────┘
```

### Component 2: Backend Interface and Implementations

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    NOTIFICATION BACKEND INTERFACE                        │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  NotificationBackend (ABC)                                        │  │
│  │  ├── send(finding, rule) → bool                                   │  │
│  │  └── send_digest(findings[], rule) → bool                         │  │
│  └───────────────────────────┬───────────────────────────────────────┘  │
│                              │                                          │
│         ┌────────────────────┼────────────────────┐                     │
│         │                    │                    │                     │
│         ▼                    ▼                    ▼                     │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐               │
│  │ SlackBackend │   │ TeamsBackend │   │ EmailBackend │               │
│  ├──────────────┤   ├──────────────┤   ├──────────────┤               │
│  │ Incoming     │   │ Incoming     │   │ SMTP         │               │
│  │ Webhook URL  │   │ Webhook URL  │   │ host/port/TLS│               │
│  │              │   │              │   │              │               │
│  │ Block Kit    │   │ Adaptive     │   │ HTML Jinja2  │               │
│  │ messages     │   │ Cards        │   │ templates    │               │
│  │              │   │              │   │ (reuse       │               │
│  │ Color-coded  │   │ Severity     │   │  existing)   │               │
│  │ by severity  │   │ icons        │   │              │               │
│  └──────────────┘   └──────────────┘   └──────────────┘               │
│                                                                         │
│         ┌────────────────────┬────────────────────┐                     │
│         ▼                    ▼                                          │
│  ┌──────────────┐   ┌──────────────┐                                   │
│  │WebhookBackend│   │ JiraBackend  │                                   │
│  ├──────────────┤   ├──────────────┤                                   │
│  │ Custom URL   │   │ Jira REST API│                                   │
│  │ Custom hdrs  │   │ v2/v3        │                                   │
│  │              │   │              │                                   │
│  │ JSON payload │   │ Auto-create  │                                   │
│  │ with schema  │   │ issues       │                                   │
│  │              │   │              │                                   │
│  │ Retry with   │   │ Dedup via    │                                   │
│  │ exp backoff  │   │ JQL search   │                                   │
│  └──────────────┘   └──────────────┘                                   │
└─────────────────────────────────────────────────────────────────────────┘
```

### Component 3: Notification Message Formats

**Slack Message (Block Kit):**

```
┌─────────────────────────────────────────────────────────┐
│ 🔴  HIGH SEVERITY FINDING                     Artemis   │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Target:   https://example.com/.git/config              │
│  Type:     Exposed Version Control Folder               │
│  Module:   vcs                                          │
│  Tag:      client_production                            │
│  Found:    2026-03-14 09:23 UTC                         │
│                                                         │
│  Found exposed .git repository with potential           │
│  credentials in the remote URL.                         │
│                                                         │
│  ┌─────────────────┐  ┌─────────────────────────┐      │
│  │ View in Artemis │  │ Mark as False Positive  │      │
│  └─────────────────┘  └─────────────────────────┘      │
└─────────────────────────────────────────────────────────┘
```

**Digest Email:**

```
┌─────────────────────────────────────────────────────────┐
│ Subject: [Artemis] Weekly Digest — 12 new (3 Critical)  │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Weekly Scan Summary for tag "client_production"        │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │ Severity   │ Count  │ New This Week │  Trend    │    │
│  ├────────────┼────────┼───────────────┼───────────┤    │
│  │ 🔴 High   │   8    │     3         │   ↗ +2    │    │
│  │ 🟡 Medium │  15    │     5         │   ↗ +1    │    │
│  │ 🔵 Low    │  22    │     4         │   → 0     │    │
│  └─────────────────────────────────────────────────┘    │
│                                                         │
│  Top New Findings:                                      │
│  1. 🔴 Exposed .git with creds — example.com           │
│  2. 🔴 SQL injection — app.example.com/api/v1           │
│  3. 🔴 Weak admin password — admin.site.org             │
│  4. 🟡 Outdated WordPress — blog.example.com            │
│  5. 🟡 CORS misconfiguration — api.example.com          │
│                                                         │
│  [View All Findings in Artemis ►]                       │
└─────────────────────────────────────────────────────────┘
```

**Jira Issue:**

```
┌─────────────────────────────────────────────────────────┐
│  PROJECT: SEC  │  Type: Bug  │  Priority: 🔴 Critical  │
├─────────────────────────────────────────────────────────┤
│  Title: [Artemis] Exposed .git — example.com            │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Description:                                           │
│  Artemis vulnerability scanner detected:                │
│                                                         │
│  | Field    | Value                              |      │
│  |----------|-------------------------------------|     │
│  | Target   | https://example.com/.git/config    |      │
│  | Type     | Exposed Version Control Folder     |      │
│  | Severity | HIGH                               |      │
│  | Module   | vcs                                |      │
│  | Tag      | client_production                  |      │
│  | Found    | 2026-03-14 09:23 UTC               |      │
│                                                         │
│  [Artemis Link: https://artemis.example.com/task/...]   │
│                                                         │
│  Labels: artemis, security, auto-generated              │
│  Custom Field (dedup_key): a1b2c3d4e5...                │
└─────────────────────────────────────────────────────────┘
```

**Webhook JSON Payload:**

```json
{
  "event": "new_finding",
  "timestamp": "2026-03-14T09:23:45Z",
  "finding": {
    "id": "task-uuid-here",
    "target": "https://example.com/.git/config",
    "report_type": "exposed_version_control_folder",
    "severity": "high",
    "status_reason": "Found exposed .git repository",
    "module": "vcs",
    "tag": "client_production"
  },
  "artemis_url": "https://artemis.example.com/task/task-uuid-here"
}
```

### Component 4: Configuration UI

**Rules Management Page:**

```
┌──────────────────────────────────────────────────────────────────────┐
│  Notification Rules                                    [+ New Rule]  │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌────┬──────────────────────┬────────┬──────────┬────────┬───────┐ │
│  │ On │ Rule Name            │Channel │ Severity │ Tag    │Actions│ │
│  ├────┼──────────────────────┼────────┼──────────┼────────┼───────┤ │
│  │ ✅ │ Critical alerts      │ Slack  │ >= High  │ *      │ ✏️ 🗑️│ │
│  │ ✅ │ Client A weekly      │ Email  │ >= Low   │client_a│ ✏️ 🗑️│ │
│  │ ❌ │ Security team Jira   │ Jira   │ >= Med   │ *      │ ✏️ 🗑️│ │
│  │ ✅ │ SIEM integration     │Webhook │ >= High  │ *      │ ✏️ 🗑️│ │
│  │ ✅ │ DNS team alerts      │ Teams  │ >= Med   │ dns_*  │ ✏️ 🗑️│ │
│  └────┴──────────────────────┴────────┴──────────┴────────┴───────┘ │
│                                                                      │
├──── Create / Edit Rule ─────────────────────────────────────────────┤
│                                                                      │
│  Rule Name: [Critical Slack Alerts          ]                        │
│                                                                      │
│  ┌─ Matching Criteria ──────────────────────────────────────────┐   │
│  │ Tag Pattern:     [client_*     ] (glob, leave empty for all) │   │
│  │ Min Severity:    [High      ▼]                               │   │
│  │ Module Filter:   [☑ vcs ☑ nuclei ☐ bruter ☐ humble ...]     │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌─ Delivery ──────────────────────────────────────────────────┐    │
│  │ Channel:  (●) Slack  ( ) Teams  ( ) Email  ( ) Webhook      │    │
│  │                                                              │    │
│  │ Webhook URL: [https://hooks.slack.com/services/T.../B...]    │    │
│  │                                                              │    │
│  │ Mode: (●) Immediate    ( ) Digest                            │    │
│  │       Deduplication: [☑] (7 day window)                      │    │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  [Save Rule]  [Test Notification]  [Cancel]                          │
└──────────────────────────────────────────────────────────────────────┘
```

**Notification History Page:**

```
┌──────────────────────────────────────────────────────────────────────┐
│  Notification History                [Filter: All ▼] [Search ____]   │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────┬───────────────────┬─────────┬────────┬────────┬─────┐ │
│  │ Time     │ Target            │ Channel │ Rule   │ Status │ ↻   │ │
│  ├──────────┼───────────────────┼─────────┼────────┼────────┼─────┤ │
│  │ 09:23:47 │ example.com/.git  │ 📱Slack │ Crit.  │ ✅ Sent│     │ │
│  │ 09:23:48 │ example.com/.git  │ 📧Email │ Weekly │ ✅ Sent│     │ │
│  │ 09:15:12 │ app.test.com/api  │ 🎫Jira  │ SecTeam│ ❌ Fail│ [↻] │ │
│  │ 08:45:00 │ blog.org/wp-admin │ 📱Slack │ Crit.  │ ✅ Sent│     │ │
│  │ 08:30:15 │ cdn.site.com      │ 🔗Hook  │ SIEM   │ ✅ Sent│     │ │
│  └──────────┴───────────────────┴─────────┴────────┴────────┴─────┘ │
│                                                                      │
│  ◄ 1  2  3  4  5 ►                         Showing 1-25 of 1,203    │
│                                                                      │
│  Summary: 1,180 sent (98.1%) │ 18 failed (1.5%) │ 5 retrying (0.4%) │
└──────────────────────────────────────────────────────────────────────┘
```

### Component 5: Docker & Configuration Integration

**Docker Compose Addition:**

```
┌─────────────────────────────────────────────────────────────────┐
│  docker-compose.yaml — new service                              │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ notification-processor:                                  │    │
│  │   image: certpl/artemis:latest                           │    │
│  │   restart: always                                        │    │
│  │   depends_on: [postgres, redis]                          │    │
│  │   command: python3 -m artemis.notification_processor     │    │
│  │   env_file: .env                                         │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  Existing Services:                                             │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌─────────────┐             │
│  │postgres│ │ redis  │ │  web   │ │autoreporter │             │
│  └────────┘ └────────┘ └────────┘ └─────────────┘             │
│       ▲          ▲          ▲                                   │
│       │          │          │                                   │
│       └──────────┴──────────┴──────────┐                       │
│                                        │                       │
│  ┌─────────────────────────────────────┴───┐                   │
│  │     notification-processor (NEW)        │                   │
│  └─────────────────────────────────────────┘                   │
└─────────────────────────────────────────────────────────────────┘
```

**Configuration Variables:**

| Variable | Default | Description |
|----------|---------|-------------|
| `ENABLE_NOTIFICATIONS` | `false` | Master switch for notification system |
| `NOTIFICATION_POLL_INTERVAL_SECONDS` | `60` | How often to check for new findings |
| `NOTIFICATION_SMTP_HOST` | `""` | SMTP server for email notifications |
| `NOTIFICATION_SMTP_PORT` | `587` | SMTP port |
| `NOTIFICATION_SMTP_USERNAME` | `""` | SMTP authentication |
| `NOTIFICATION_SMTP_PASSWORD` | `""` | SMTP password |
| `NOTIFICATION_SMTP_FROM` | `""` | Sender email address |
| `NOTIFICATION_DEDUP_WINDOW_DAYS` | `7` | Deduplication window |
| `NOTIFICATION_MAX_RETRIES` | `5` | Maximum retry attempts |
| `NOTIFICATION_ENCRYPTION_KEY` | `""` | Fernet key for channel_config encryption |

---

## Timeline and Deliverables

### Gantt Chart

```
        May         June           July           August         Sep
       Week: 1 2 3 4 1 2 3 4 5 6 7 8 9 10 11 12  1
             ├─┤                                      Community Bonding
             │CB│
                   ├─────────┤                        Phase 1: Core + Slack/Email
                   │DB│Rules│Slack│Email│UI│
                             ├───┤                    ◆ Midterm Eval
                                  ├─────────┤         Phase 2: More backends
                                  │Hook│Teams│Jira│Digest│Dedup│
                                              ├──────┤Phase 3: History + Polish
                                              │Hist│Docker│Test│Doc│
                                                     ├┤ Final Eval

Legend: CB=Community Bonding, ◆=Evaluation
```

### Phase 1: Weeks 1-4 (June 2 - June 29)

**Deliverable: Rules engine, Slack and Email backends, basic UI**

| Week | Tasks | Deliverable | Hours |
|------|-------|-------------|-------|
| 1 | `NotificationRule` + `NotificationLog` models; Alembic migration; DB CRUD | Schema + migration | 30 |
| 2 | Rules evaluation engine; Slack backend with Block Kit | Working Slack alerts | 30 |
| 3 | Email backend with Jinja2 templates (reuse existing infra) | Working email alerts | 30 |
| 4 | Rules config UI; test notification button | Management interface | 35 |

### Midterm Evaluation (June 30 - July 4)

### Phase 2: Weeks 5-8 (July 5 - August 3)

**Deliverable: Webhook, Teams, Jira backends; digest mode; deduplication**

| Week | Tasks | Deliverable | Hours |
|------|-------|-------------|-------|
| 5 | Generic Webhook backend; Microsoft Teams backend | 2 more channels | 30 |
| 6 | Jira backend with issue creation + dedup via JQL | Jira integration | 35 |
| 7 | Digest mode: accumulate findings, cron-based batch send | Digest notifications | 30 |
| 8 | Deduplication logic; retry with exponential backoff | Robust delivery | 30 |

### Phase 3: Weeks 9-12 (August 4 - August 31)

| Week | Tasks | Deliverable | Hours |
|------|-------|-------------|-------|
| 9 | Notification history UI with filtering | History dashboard | 30 |
| 10 | Docker Compose integration; config vars; encryption | Production-ready | 25 |
| 11 | Unit tests (mocked APIs); integration tests (MailHog) | Test coverage | 30 |
| 12 | Setup guides per channel; admin guide; architecture docs | Documentation | 25 |

---

## Effort Breakdown

```
                    Effort Distribution (350 hours)

  ┌────────────────────────────────────────────────────────┐
  │                                                        │
  │  DB Schema & Rules Engine  █████░░░░░░░░░░░  45h (13%)│
  │  Slack Backend             ████░░░░░░░░░░░░░  30h  (9%)│
  │  Email Backend             ████░░░░░░░░░░░░░  30h  (9%)│
  │  Webhook Backend           ███░░░░░░░░░░░░░░  20h  (6%)│
  │  Teams Backend             ███░░░░░░░░░░░░░░  20h  (6%)│
  │  Jira Backend              █████░░░░░░░░░░░░  35h (10%)│
  │  Digest + Deduplication    █████░░░░░░░░░░░░  40h (11%)│
  │  Configuration UI          █████░░░░░░░░░░░░  40h (11%)│
  │  Notification History UI   ████░░░░░░░░░░░░░  30h  (9%)│
  │  Docker + Config           ███░░░░░░░░░░░░░░  20h  (6%)│
  │  Testing                   ████░░░░░░░░░░░░░  25h  (7%)│
  │  Documentation             ██░░░░░░░░░░░░░░░  15h  (4%)│
  │                                                        │
  └────────────────────────────────────────────────────────┘
```

---

## Technical Challenges and Mitigations

| Challenge | Risk | Mitigation |
|-----------|------|------------|
| External API rate limits (Slack, Jira) | Medium | Per-channel rate limiting with token bucket algorithm |
| Notification storms on large scan completion | High | Deduplication + digest mode + configurable cooldown period |
| Sensitive channel configs (API keys) in DB | Medium | Encrypt `channel_config` JSONB using Fernet symmetric encryption |
| Testing external integrations in CI | Medium | Mock servers (MailHog for SMTP, WireMock for HTTP) in test environment |
| Jira dedup requires searching for existing issues | Low | JQL query with custom field; cache results to reduce API calls |
| Polling-based may miss findings during restart | Low | Track last-processed timestamp in Redis; catch up on restart |

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
