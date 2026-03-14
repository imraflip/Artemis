# GSoC 2026 Proposal: Interactive Analytics Dashboard with Trend Analysis for Artemis

## Contact Information

- **Name:** [Your Full Name]
- **Email:** [your.email@example.com]
- **GitHub:** [github.com/yourusername]
- **Timezone:** [e.g., UTC+2]
- **University / Program:** [Your University, Year, Major]

---

## Project Overview

### Title
Interactive Analytics Dashboard with Trend Analysis

### Synopsis
Artemis's current web interface is a functional but static table-based view. When CSIRT teams manage thousands of scanned domains, they need visual trend analysis, comparative views, and actionable analytics — not just raw data tables. This project replaces the static dashboard with an interactive, chart-based analytics UI backed by efficient SQL aggregation queries, enabling operators to understand vulnerability trends, compare scan runs, and assess organizational risk posture at a glance.

### Current vs. Proposed Dashboard

```
┌───────────────────────────────────────────────────────────────────┐
│  CURRENT: Static table with minimal insight                       │
│                                                                   │
│  ┌───────────────────────────────────────────────────────────┐    │
│  │ Target         │ Tag      │ Created At     │ Actions      │    │
│  │ example.com    │ client_a │ 2026-03-01     │ [View]       │    │
│  │ test.org       │ client_b │ 2026-03-02     │ [View]       │    │
│  │ app.io         │ client_c │ 2026-03-03     │ [View]       │    │
│  │ ...            │ ...      │ ...            │              │    │
│  └───────────────────────────────────────────────────────────┘    │
│  No charts. No trends. No comparisons. Just rows.                 │
└───────────────────────────────────────────────────────────────────┘

                              │
                              ▼

┌───────────────────────────────────────────────────────────────────┐
│  PROPOSED: Interactive analytics with visual insights             │
│                                                                   │
│  ┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐ ┌──────────────────┐   │
│  │ 1,234 │ │ 5,678 │ │  234  │ │  89   │ │ Trend ↗ +12%     │   │
│  │Analyses│ │Findings│ │ High │ │Pending│ │ vs last month    │   │
│  └───────┘ └───────┘ └───────┘ └───────┘ └──────────────────┘   │
│  ┌────────────────────────────┐ ┌────────────────────────────┐   │
│  │  📈 Trend Chart           │ │  🍩 Severity Distribution  │   │
│  │  ▄   ▄▄                   │ │    ██ High: 234 (15%)      │   │
│  │ ██▄▄████▄▄   ▄            │ │   ████ Med: 890 (55%)     │   │
│  │ ████████████▄██▄           │ │  ██████ Low: 480 (30%)    │   │
│  └────────────────────────────┘ └────────────────────────────┘   │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │  📊 Risk by Organization  │ Findings │ Trend │ Score      │   │
│  │  client_a                 │    67    │  ↗ +5 │ ████ 8.2   │   │
│  │  client_b                 │    23    │  ↘ -2 │ ██   4.1   │   │
│  └───────────────────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────────────────┘
```

### Motivation
The current Artemis dashboard (`templates/index.jinja2`) renders a paginated table of analyses with minimal filtering. The task results view (`templates/task_list.jinja2`) offers basic search via PostgreSQL fulltext (`TSVector`). For a tool that scans hundreds of thousands of domains:

- There is **no way to see vulnerability trends over time** — are findings increasing or decreasing?
- There is **no comparison between scan runs** — what's new vs. what persists?
- There is **no per-organization risk breakdown** — which tagged groups need most attention?
- There is **no module performance visibility** — which modules find the most issues?
- The existing `stats.py` in `artemis/reporting/export/` computes statistics only during report export, not for interactive use

These gaps make it difficult for CSIRT teams to prioritize their work and demonstrate the value of their scanning programs.

---

## Detailed Description

### System Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                      ANALYTICS ARCHITECTURE                         │
│                                                                     │
│  ┌───────────┐     ┌──────────────┐     ┌────────────────────────┐ │
│  │ PostgreSQL│     │   FastAPI     │     │   Browser (Client)     │ │
│  │           │     │              │     │                        │ │
│  │ TaskResult├────►│ /api/stats/  ├────►│  Chart.js renders      │ │
│  │ Analysis  │     │ findings-    │     │  interactive charts    │ │
│  │ Tag       │     │ over-time    │     │                        │ │
│  │           │     │              │     │  Jinja2 provides       │ │
│  │ Aggregate │     │ /api/stats/  │     │  page structure        │ │
│  │ Queries   │     │ tags-summary │     │                        │ │
│  │ (GROUP BY,│     │              │     │  Bootstrap handles     │ │
│  │  FILTER)  │     │ /api/stats/  │     │  layout & responsive   │ │
│  │           │     │ modules      │     │                        │ │
│  └─────┬─────┘     │              │     └────────────────────────┘ │
│        │           │ /api/stats/  │                                 │
│        │           │ comparison   │                                 │
│        ▼           └──────┬───────┘                                 │
│  ┌───────────┐           │                                         │
│  │   Redis   │◄──────────┘                                         │
│  │  Cache    │  TTL-based caching                                  │
│  │  (5 min)  │  for expensive queries                              │
│  └───────────┘                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Data Flow for Analytics

```
User opens           Jinja2 renders         JS fetches             Charts
dashboard            page skeleton          API data               render
    │                     │                     │                     │
    ▼                     ▼                     ▼                     ▼
┌────────┐          ┌──────────┐          ┌──────────┐          ┌────────┐
│GET /   │─────────►│ HTML with │─────────►│GET /api/ │─────────►│Chart.js│
│        │          │ empty     │          │stats/*   │          │draws   │
│        │          │ chart     │          │          │          │charts  │
│        │          │ containers│          │Returns   │          │into    │
│        │          │           │          │JSON data │          │<canvas>│
└────────┘          └──────────┘          └────┬─────┘          └────────┘
                                               │
                                    ┌──────────┴──────────┐
                                    │                     │
                               Cache HIT            Cache MISS
                                    │                     │
                                    ▼                     ▼
                              ┌──────────┐          ┌──────────┐
                              │  Redis   │          │PostgreSQL│
                              │  Return  │          │ Execute  │
                              │  cached  │          │ aggregate│
                              │  JSON    │          │ query    │
                              └──────────┘          └────┬─────┘
                                                         │
                                                    Store in Redis
                                                    with TTL=300s
```

### Component 1: Analytics API Endpoints

New routes in a dedicated `artemis/analytics_api.py` module, registered on the FastAPI app:

**API Endpoint Map:**

| Endpoint | Method | Query Params | Returns | Cache TTL |
|----------|--------|-------------|---------|-----------|
| `/api/stats/findings-over-time` | GET | `interval`, `tag`, `severity` | `[{date, new, resolved, total}]` | 5 min |
| `/api/stats/tags-summary` | GET | — | `[{tag, total, high, med, low}]` | 5 min |
| `/api/stats/modules` | GET | — | `[{module, findings, errors}]` | 5 min |
| `/api/stats/comparison` | GET | `tag_a`, `tag_b` or date ranges | `{new, resolved, persistent}` | 2 min |
| `/api/stats/severity-distribution` | GET | `tag` | `[{severity, count}]` | 5 min |
| `/api/stats/top-targets` | GET | `limit`, `tag` | `[{target, count, severity}]` | 5 min |

**SQL Query Design:** These endpoints use PostgreSQL aggregate functions and window functions:

```sql
-- Example: findings over time (weekly)
SELECT
    date_trunc('week', created_at) AS period,
    COUNT(*) FILTER (WHERE status = 'INTERESTING') AS interesting_count,
    COUNT(*) AS total_count
FROM task_result
WHERE tag = :tag AND created_at >= :since
GROUP BY period
ORDER BY period;

-- Example: per-tag severity breakdown
SELECT
    tag,
    COUNT(*) AS total,
    COUNT(*) FILTER (WHERE result->>'severity' = 'high') AS high_count,
    COUNT(*) FILTER (WHERE result->>'severity' = 'medium') AS medium_count,
    COUNT(*) FILTER (WHERE result->>'severity' = 'low') AS low_count,
    MAX(created_at) AS latest_finding
FROM task_result
WHERE status = 'INTERESTING'
GROUP BY tag
ORDER BY total DESC;
```

For performance with large datasets, queries use:
- Indexed columns (`tag`, `status`, `created_at` — already indexed in `TaskResult`)
- `FILTER` clauses instead of subqueries
- Result caching in Redis with configurable TTL

### Component 2: Dashboard Redesign

**Full Dashboard Layout Mockup:**

```
┌──────────────────────────────────────────────────────────────────────┐
│  Artemis Analytics Dashboard                    [+ Add Targets]      │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐    │
│  │  📊 1,234  │  │  🔍 5,678  │  │  🔴  234   │  │  ⏳   89   │    │
│  │  Analyses  │  │  Findings  │  │  Critical  │  │  Pending   │    │
│  │            │  │  Total     │  │  (High)    │  │  Tasks     │    │
│  └────────────┘  └────────────┘  └────────────┘  └────────────┘    │
│                                                                      │
│  ┌─ Findings Over Time ──────────────────────── [1W][1M][3M][1Y] ─┐ │
│  │  250│                                                          │ │
│  │     │    ▄                                                     │ │
│  │  200│   ██    ▄                                                │ │
│  │     │   ██   ██  ▄                                             │ │
│  │  150│   ██   ██  ██         ▄▄                                 │ │
│  │     │   ██▄  ██  ██    ▄   ████                                │ │
│  │  100│   ███▄ ██▄ ██   ██▄ ████▄    ▄                           │ │
│  │     │   ████ ███ ██▄ ████ █████   ██▄                          │ │
│  │   50│   ████ ███ ███ ████ █████▄ ████                          │ │
│  │     │   ████ ███ ███ ████ ██████ ████▄                         │ │
│  │    0│───████─███─███─████─██████─█████──                       │ │
│  │     └──Jan──Feb──Mar──Apr──May──Jun──Jul──                     │ │
│  │                                                                │ │
│  │  Legend: ███ High  ███ Medium  ███ Low                         │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│  ┌─ Severity Distribution ────────┐  ┌─ Top Modules ──────────────┐ │
│  │                                │  │                            │ │
│  │        ╭────────╮              │  │  nuclei      ████████  312 │ │
│  │      ╭─┤ HIGH   ├──╮          │  │  bruter      ██████    245 │ │
│  │    ╭─┤ │  15%   │  ├─╮        │  │  vcs         █████     198 │ │
│  │    │ │ ╰────────╯  │ │        │  │  wp_scanner  ████      156 │ │
│  │    │ │  ╭───────╮  │ │        │  │  sql_inj     ███       112 │ │
│  │    │ ├──┤MEDIUM ├──┤ │        │  │  dns_scan    ██         89 │ │
│  │    │ │  │  55%  │  │ │        │  │  humble      █          45 │ │
│  │    │ │  ╰───────╯  │ │        │  │  port_scan   █          34 │ │
│  │    ╰─┤  ╭──────╮  ├─╯        │  │                            │ │
│  │      ╰──┤ LOW  ├──╯          │  │  [Show All Modules ►]      │ │
│  │         │  30% │              │  │                            │ │
│  │         ╰──────╯              │  │                            │ │
│  └────────────────────────────────┘  └────────────────────────────┘ │
│                                                                      │
│  ┌─ Per-Organization Risk Breakdown ──────────────── [Filter] [↕] ─┐ │
│  │                                                                  │ │
│  │  Tag          Total   🔴High  🟡Med   🔵Low   Trend    Last Scan │ │
│  │  ──────────   ─────   ─────   ────    ────    ──────   ──────── │ │
│  │  client_a       67      25     30      12     ↗ +5     2h ago   │ │
│  │   ▁▂▃▅▇▅▃  (sparkline showing 8-week trend)                    │ │
│  │                                                                  │ │
│  │  client_b       45      12     20      13     ↘ -2     1d ago   │ │
│  │   ▇▅▃▂▁▂▃  (sparkline)                                         │ │
│  │                                                                  │ │
│  │  client_c       23       3      8      12     → 0      3h ago   │ │
│  │   ▃▃▃▃▃▃▃  (sparkline)                                         │ │
│  │                                                                  │ │
│  │  ◄ 1  2  3 ►                             Showing 1-10 of 45     │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│  ┌─ Top Critical Findings ─────────────────────────────────────────┐ │
│  │  🔴 Exposed .git with creds    example.com       vcs    3d ago  │ │
│  │  🔴 SQL injection              app.test.com      nuclei 1d ago  │ │
│  │  🔴 Weak admin password        admin.site.org    bruter 5h ago  │ │
│  │  [View All Findings ►]                                          │ │
│  └─────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘
```

### Component 3: Comparison View

**Comparison Page Mockup:**

```
┌──────────────────────────────────────────────────────────────────────┐
│  Artemis — Compare                                    [Dashboard]    │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Compare: [Tag ▼] client_a  vs.  [Tag ▼] client_b    [Compare]     │
│           OR                                                         │
│           [Date Range] Mar 1-7  vs.  [Date Range] Mar 8-14           │
│                                                                      │
│  ┌─ Comparison Summary ────────────────────────────────────────────┐ │
│  │                                                                  │ │
│  │          client_a              BOTH             client_b         │ │
│  │        ┌──────────┐      ┌──────────┐      ┌──────────┐        │ │
│  │        │          │      │          │      │          │        │ │
│  │        │    23    │      │    34    │      │    11    │        │ │
│  │        │  unique  │      │  shared  │      │  unique  │        │ │
│  │        │ findings │      │ findings │      │ findings │        │ │
│  │        │          │      │          │      │          │        │ │
│  │        └──────────┘      └──────────┘      └──────────┘        │ │
│  │                                                                  │ │
│  │  ┌────────────────────────────┬────────────────────────────┐    │ │
│  │  │ Only in client_a (23)     │ Only in client_b (11)      │    │ │
│  │  ├────────────────────────────┼────────────────────────────┤    │ │
│  │  │ 🔴 exposed_vcs (5)       │ 🔴 sql_injection (2)      │    │ │
│  │  │ 🔴 weak_credentials (3)  │ 🟡 outdated_wp (4)        │    │ │
│  │  │ 🟡 directory_index (8)   │ 🟡 open_redirect (3)      │    │ │
│  │  │ 🔵 missing_headers (7)   │ 🔵 exposed_phpinfo (2)    │    │ │
│  │  └────────────────────────────┴────────────────────────────┘    │ │
│  │                                                                  │ │
│  │  Severity Comparison:                                            │ │
│  │  ┌──────────────────────────────────────────────────────────┐   │ │
│  │  │ HIGH   │ client_a: ████████  8  │ client_b: ████  4     │   │ │
│  │  │ MEDIUM │ client_a: ████████  8  │ client_b: ███████ 7   │   │ │
│  │  │ LOW    │ client_a: ███████  7   │ client_b: ██  2       │   │ │
│  │  └──────────────────────────────────────────────────────────┘   │ │
│  └──────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘
```

The comparison logic reuses the existing `NormalForm` concept from `artemis/reporting/base/normal_form.py` — two findings are "the same" if they share the same normal form.

### Component 4: Saved Filters and Bookmarkable Searches

```
┌─────────────────────────────────────────────────────────────────┐
│  Saved Filters                                      [+ New]     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ 📌 "High-sev client_a"    tag=client_a, severity=high  │    │
│  │    [Load]  [Delete]  [Copy URL]                         │    │
│  │                                                         │    │
│  │ 📌 "VCS findings"         module=vcs, status=INTERESTING│    │
│  │    [Load]  [Delete]  [Copy URL]                         │    │
│  │                                                         │    │
│  │ 📌 "Last week new"        date=7d, severity=all        │    │
│  │    [Load]  [Delete]  [Copy URL]                         │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  URL: artemis.example.com/?tag=client_a&severity=high&module=vcs│
│       ↑ All filters encoded in URL — shareable via link         │
└─────────────────────────────────────────────────────────────────┘
```

**Saved Filters model:**
```python
class SavedFilter(Base):
    __tablename__ = "saved_filter"
    id = Column(Integer, primary_key=True)
    name = Column(String)
    filter_params = Column(JSONB)  # {tag, severity, module, date_range, search_query}
    created_at = Column(DateTime, server_default=text("NOW()"))
```

### Component 5: Performance Optimization Strategy

```
┌─────────────────────────────────────────────────────────────────────┐
│                  PERFORMANCE OPTIMIZATION LAYERS                     │
│                                                                     │
│  Layer 1: Query Optimization                                        │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  Partial Index                    Existing Indexes             │  │
│  │  ┌─────────────────────────┐     ┌─────────────────────┐     │  │
│  │  │ CREATE INDEX ... WHERE  │     │ tag (B-tree)        │     │  │
│  │  │ status = 'INTERESTING'  │     │ status (B-tree)     │     │  │
│  │  │ ON (tag, created_at)    │     │ created_at (B-tree) │     │  │
│  │  └─────────────────────────┘     │ fulltext (GIN)      │     │  │
│  │  ~10x faster for filtered queries└─────────────────────┘     │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  Layer 2: Redis Caching                                             │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  Dashboard stats ─── TTL: 5 min ─── Auto-refresh on request  │  │
│  │  Comparison data ─── TTL: 2 min ─── Invalidate on new scan   │  │
│  │  Module stats    ─── TTL: 5 min ─── Background refresh       │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  Layer 3: Materialized Views (for >1M rows)                         │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  mv_daily_findings_count                                      │  │
│  │  mv_tag_severity_summary    ← Refreshed every 15 minutes     │  │
│  │  mv_module_performance                                        │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  Layer 4: Frontend Optimization                                     │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  Default range: 90 days │ Lazy-load charts │ Loading spinners │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Timeline and Deliverables

### Gantt Chart

```
        May         June           July           August         Sep
       Week: 1 2 3 4 1 2 3 4 5 6 7 8 9 10 11 12  1
             ├─┤                                      Community Bonding
             │CB│
                   ├─────────┤                        Phase 1: API + Charts
                   │SQL│Cache│Chart.js│
                             ├───┤                    ◆ Midterm Eval
                                  ├─────────┤         Phase 2: Compare + Filters
                                  │Comp│Risk│Filters│
                                              ├──────┤Phase 3: Perf + Polish
                                              │Idx│UI│Tests│Doc│
                                                     ├┤ Final Eval

Legend: CB=Community Bonding, ◆=Evaluation, SQL=SQL queries, Comp=Comparison
```

### Phase 1: Weeks 1-4 (June 2 - June 29)

**Deliverable: Analytics API endpoints and basic dashboard charts**

| Week | Tasks | Deliverable | Hours |
|------|-------|-------------|-------|
| 1 | Core SQL queries; `analytics_api.py`; findings-over-time + severity endpoints | 2 working API endpoints | 30 |
| 2 | Tags-summary, modules, top-targets endpoints; Redis caching layer | 6 API endpoints + caching | 30 |
| 3 | Redesign `index.jinja2`; overview cards; trend line chart (Chart.js) | Dashboard with live data | 35 |
| 4 | Severity pie chart; module bar chart; heatmap; responsive layout | Complete chart suite | 30 |

### Midterm Evaluation (June 30 - July 4)

**Expected state:** New analytics API with 6+ endpoints, all backed by efficient SQL queries with Redis caching. Dashboard shows trend charts, severity breakdown, and overview cards. API is documented and has unit tests.

### Phase 2: Weeks 5-8 (July 5 - August 3)

**Deliverable: Comparison view, per-tag risk table, saved filters**

| Week | Tasks | Deliverable | Hours |
|------|-------|-------------|-------|
| 5 | Comparison API (NormalForm matching); comparison page UI | Working compare tool | 30 |
| 6 | Enhanced per-tag table with sparklines; top-findings section | Risk breakdown view | 30 |
| 7 | SavedFilter model; CRUD API; save/load/delete UI | Saved filters feature | 30 |
| 8 | URL-based filter state; bookmarkable views; share via link | Shareable filter URLs | 25 |

### Phase 3: Weeks 9-12 (August 4 - August 31)

**Deliverable: Performance optimization, polish, documentation**

| Week | Tasks | Deliverable | Hours |
|------|-------|-------------|-------|
| 9 | Database indexes; materialized views; benchmark with large datasets | Perf improvements | 30 |
| 10 | UI polish: loading states, errors, empty states, mobile | Production-ready UI | 30 |
| 11 | API unit tests; SQL tests; frontend interaction tests | Test coverage | 30 |
| 12 | User guide; API docs; developer extension guide | Complete documentation | 25 |

### Final Evaluation (September 1 - September 8)

---

## Effort Breakdown

```
                    Effort Distribution (350 hours)

  ┌────────────────────────────────────────────────────────┐
  │                                                        │
  │  SQL Query Design          ████░░░░░░░░░░░░░  40h (11%)│
  │  API Endpoints (FastAPI)   ██████░░░░░░░░░░░  50h (14%)│
  │  Redis Caching Layer       ███░░░░░░░░░░░░░░  25h  (7%)│
  │  Dashboard Charts (JS)     █████████░░░░░░░░  75h (21%)│
  │  Comparison View           █████░░░░░░░░░░░░  45h (13%)│
  │  Saved Filters             ████░░░░░░░░░░░░░  30h  (9%)│
  │  Performance Optimization  ████░░░░░░░░░░░░░  30h  (9%)│
  │  Testing                   ████░░░░░░░░░░░░░  35h (10%)│
  │  Documentation             ██░░░░░░░░░░░░░░░  20h  (6%)│
  │                                                        │
  └────────────────────────────────────────────────────────┘
```

---

## Technical Challenges and Mitigations

| Challenge | Risk | Mitigation |
|-----------|------|------------|
| Aggregation queries on millions of rows may be slow | High | Partial indexes, materialized views, Redis caching with TTL |
| Chart.js charts need to handle variable data ranges | Low | Adaptive time bucketing (auto-select day/week/month based on range) |
| Comparison across tags requires finding matching | Medium | Reuse existing `NormalForm` infrastructure from the reporting system |
| Adding JS to a Jinja2 app without creating maintenance burden | Low | Keep JS minimal, use Chart.js declaratively, avoid custom build pipeline |
| Dashboard must remain usable during long-running queries | Medium | Async API calls with loading spinners; cache aggressively |
| Mobile responsiveness for chart-heavy pages | Low | Chart.js is responsive by default; test on mobile viewports |

---

## Technology Stack

```
┌─────────────────────────────────────────────────────────┐
│                    TECHNOLOGY STACK                       │
│                                                         │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │  Frontend   │  │   Backend    │  │   Database    │  │
│  ├─────────────┤  ├──────────────┤  ├───────────────┤  │
│  │ Chart.js    │  │ FastAPI      │  │ PostgreSQL 16 │  │
│  │ Bootstrap 5 │  │ SQLAlchemy   │  │ (existing)    │  │
│  │ Jinja2      │  │ Python 3.x   │  │               │  │
│  │ Vanilla JS  │  │ Pydantic     │  │ Redis 7.2     │  │
│  │             │  │              │  │ (existing)    │  │
│  └─────────────┘  └──────────────┘  └───────────────┘  │
│                                                         │
│  No new frameworks. No build pipeline. Minimal JS.       │
│  100% compatible with existing Artemis tech stack.       │
└─────────────────────────────────────────────────────────┘
```

---

## Relevant Skills and Experience

- [Describe your Python experience]
- [Describe your experience with web development (HTML/CSS/JS)]
- [Describe your experience with data visualization (Chart.js, D3, etc.)]
- [Describe your experience with SQL and database optimization]
- [Link to relevant projects/contributions]

---

## Why This Project?

The dashboard is the primary interface through which CSIRT operators interact with Artemis daily. The current static tables work for small deployments but become unwieldy at scale. An interactive analytics dashboard transforms Artemis from a tool you query into a tool that proactively surfaces insights. The trend analysis and comparison features directly enable data-driven prioritization of remediation efforts — a critical need for teams managing hundreds of organizations.

This project has high visibility: every Artemis user will see and benefit from the improvements immediately.

---

## Contributions to Artemis (Pre-GSoC)

- [List any PRs, issues, or discussions you've contributed to]
- [If none yet, describe your plan to contribute before the application deadline]

---

## Availability

- I can commit approximately **30-35 hours per week** during the coding period
- [Mention any planned vacations, exams, or other commitments]
- I am available for weekly sync calls with my mentor
