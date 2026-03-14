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

### Architecture

The implementation stays within Artemis's existing technology choices:
- **Backend:** FastAPI with new `/api/stats/*` endpoints
- **Database:** PostgreSQL with SQLAlchemy — leveraging SQL aggregation queries
- **Frontend:** Jinja2 templates + Bootstrap + **Chart.js** (lightweight, no heavy framework needed)
- **Caching:** Redis for expensive aggregation query results (using existing `RedisCache` from `artemis/redis_cache.py`)

### Component 1: Analytics API Endpoints

New routes in a dedicated `artemis/analytics_api.py` module, registered on the FastAPI app:

```python
# Trend data
GET /api/stats/findings-over-time
    ?interval=day|week|month
    &tag=optional_tag
    &severity=optional_severity
    → Returns: [{date, count_new, count_resolved, count_total}]

# Per-tag breakdown
GET /api/stats/tags-summary
    → Returns: [{tag, total_findings, high_count, medium_count, low_count,
                  newest_finding_date, oldest_unresolved_date}]

# Module performance
GET /api/stats/modules
    → Returns: [{module_name, total_findings, interesting_count, error_count,
                  avg_processing_time}]

# Comparison between two time ranges or tags
GET /api/stats/comparison
    ?tag_a=tag1&tag_b=tag2
    OR ?date_from_a=...&date_to_a=...&date_from_b=...&date_to_b=...
    → Returns: {new_findings, resolved_findings, persistent_findings,
                findings_only_in_a, findings_only_in_b}

# Severity distribution
GET /api/stats/severity-distribution
    ?tag=optional_tag
    → Returns: [{severity, count}]

# Top vulnerable targets
GET /api/stats/top-targets
    ?limit=10&tag=optional_tag
    → Returns: [{target, finding_count, highest_severity}]
```

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
```

For performance with large datasets, queries use:
- Indexed columns (`tag`, `status`, `created_at` — already indexed in `TaskResult`)
- `FILTER` clauses instead of subqueries
- Result caching in Redis with configurable TTL

### Component 2: Dashboard Redesign

The main dashboard at `/` (`templates/index.jinja2`) is redesigned with multiple sections:

**Section 1: Overview Cards**
- Total active analyses
- Total findings (INTERESTING status)
- Findings by severity (High/Medium/Low counts)
- Pending scan tasks (reusing existing `get_num_pending_tasks`)

**Section 2: Trend Chart**
- Line chart showing findings discovered per week over the past N months
- Stacked by severity (High = red, Medium = orange, Low = yellow)
- Interactive: hover for details, click to filter

**Section 3: Module Activity Heatmap**
- Calendar heatmap showing which modules are active and producing findings
- Quick identification of scanning coverage gaps

**Section 4: Per-Tag Risk Table**
- Enhanced version of the current analyses table
- Added columns: finding count, highest severity, last scan date
- Inline mini-sparklines showing finding trend per tag
- Sortable by any column

**Section 5: Top Findings**
- Quick-view of the most critical unresolved findings across all tags

**Implementation:** Using Chart.js loaded from CDN (or bundled), initialized from data fetched via the new API endpoints. Charts are rendered client-side with JavaScript calling the API endpoints. This approach:
- Keeps the Jinja2 template pattern (server renders page shell, JS fetches data)
- Avoids introducing a heavy JS framework (React/Vue)
- Allows progressive enhancement — the page works without JS (shows tables), JS adds charts

### Component 3: Comparison View

A new page at `/compare` that allows operators to:

1. **Select two tags** (e.g., "client_A" vs "client_B") or two time ranges
2. **See a side-by-side diff** showing:
   - Findings present in A but not B (new findings)
   - Findings present in B but not A (resolved findings)
   - Findings present in both (persistent findings)
3. **Venn diagram visualization** using Chart.js

The comparison logic reuses the existing `NormalForm` concept from `artemis/reporting/base/normal_form.py` — two findings are "the same" if they share the same normal form.

### Component 4: Saved Filters and Bookmarkable Searches

- **URL-based state:** All filter parameters are encoded in the URL query string, making any filtered view shareable via link
- **Saved Filters:** Stored in a new `SavedFilter` DB model:
  ```python
  class SavedFilter(Base):
      __tablename__ = "saved_filter"
      id = Column(Integer, primary_key=True)
      name = Column(String)
      filter_params = Column(JSONB)  # {tag, severity, module, date_range, search_query}
      created_at = Column(DateTime, server_default=text("NOW()"))
  ```
- UI for managing saved filters: save current filter, load saved filter, delete

### Component 5: Performance Optimization for Analytics

Analytics queries on large datasets can be slow. Mitigations:

1. **Materialized views** for frequently-accessed aggregations (daily/weekly stats) — refreshed on a schedule
2. **Redis caching** with TTL (e.g., 5 minutes for dashboard data) using the existing `RedisCache` class
3. **Date range limits** — UI defaults to last 90 days, preventing unbounded queries
4. **New database indexes** — partial indexes on `status = 'INTERESTING'` for faster filtered queries:
   ```sql
   CREATE INDEX idx_task_result_interesting
   ON task_result(tag, created_at)
   WHERE status = 'INTERESTING';
   ```

---

## Timeline and Deliverables

### Community Bonding Period (May 8 - June 1)
- Study Artemis's existing frontend patterns and Bootstrap usage
- Profile existing database queries to understand performance baseline
- Prototype Chart.js integration in a minimal Jinja2 template
- Design the analytics API schema with mentor input
- Submit a small PR to familiarize with the contribution process

### Phase 1: Weeks 1-4 (June 2 - June 29)

**Deliverable: Analytics API endpoints and basic dashboard charts**

| Week | Tasks |
|------|-------|
| 1 | Implement core analytics SQL queries; create `analytics_api.py` with findings-over-time and severity-distribution endpoints |
| 2 | Add tags-summary, modules, and top-targets endpoints; implement Redis caching layer |
| 3 | Redesign `index.jinja2` dashboard with overview cards and trend line chart using Chart.js |
| 4 | Add severity distribution pie chart, module activity heatmap; ensure responsive design |

### Midterm Evaluation (June 30 - July 4)

**Expected state:** New analytics API with 6+ endpoints, all backed by efficient SQL queries with Redis caching. Dashboard shows trend charts, severity breakdown, and overview cards. API is documented and has unit tests.

### Phase 2: Weeks 5-8 (July 5 - August 3)

**Deliverable: Comparison view, per-tag risk table, saved filters**

| Week | Tasks |
|------|-------|
| 5 | Build comparison API endpoint using NormalForm matching; implement comparison page UI |
| 6 | Enhance per-tag risk table with inline sparklines and sorting; add top-findings section |
| 7 | Implement saved filters: DB model, API endpoints, UI for save/load/delete |
| 8 | URL-based state for all filter parameters; ensure all views are bookmarkable |

### Phase 3: Weeks 9-12 (August 4 - August 31)

**Deliverable: Performance optimization, polish, documentation**

| Week | Tasks |
|------|-------|
| 9 | Add database indexes and materialized views for analytics; benchmark with large datasets |
| 10 | UI polish: loading states, error handling, empty states, mobile responsiveness |
| 11 | Write comprehensive tests: API unit tests, SQL query tests, frontend interaction tests |
| 12 | Documentation: user guide for dashboard features, API docs, developer guide for extending |

### Final Evaluation (September 1 - September 8)

---

## Technical Challenges and Mitigations

| Challenge | Mitigation |
|-----------|------------|
| Aggregation queries on millions of rows may be slow | Use partial indexes, materialized views, and Redis caching with TTL |
| Chart.js charts need to handle variable data ranges | Implement adaptive time bucketing (auto-select day/week/month based on range) |
| Comparison across tags requires finding matching | Reuse existing `NormalForm` infrastructure from the reporting system |
| Adding JS to a Jinja2 app without creating maintenance burden | Keep JS minimal, use Chart.js declaratively, avoid custom build pipeline |
| Dashboard must remain usable during long-running queries | Use async API calls with loading spinners; cache aggressively |

---

## Mockup: Dashboard Layout

```
┌──────────────────────────────────────────────────────────────┐
│  Artemis Dashboard                                    [+Add] │
├──────────┬──────────┬──────────┬──────────┬──────────────────┤
│ Analyses │ Findings │ High Sev │ Med Sev  │ Pending Tasks    │
│   1,234  │   5,678  │   234    │  1,456   │     89           │
├──────────┴──────────┴──────────┴──────────┴──────────────────┤
│                                                              │
│  Findings Over Time                          [1W][1M][3M][1Y]│
│  ┌─────────────────────────────────────────────────────┐     │
│  │  ▄                                                  │     │
│  │  █▄    ▄                                            │     │
│  │  ██▄  ▄█▄    ▄       ▄▄                             │     │
│  │  ███▄▄███▄  ▄█▄   ▄▄██▄▄    ▄                      │     │
│  │  ████████████████▄████████▄▄██▄                     │     │
│  └─────────────────────────────────────────────────────┘     │
│  Jan  Feb  Mar  Apr  May  Jun  Jul  Aug  Sep               │
│                                                              │
├─────────────────────────────┬────────────────────────────────┤
│  Severity Distribution      │  Module Activity               │
│   ┌──────────────┐          │  ┌────────────────────────┐    │
│   │   ██ High    │          │  │ nuclei     ████████ 45 │    │
│   │  ████ Med    │          │  │ bruter     ██████   34 │    │
│   │ ██████ Low   │          │  │ vcs        ████     22 │    │
│   └──────────────┘          │  │ wp_scanner ███      18 │    │
│                             │  └────────────────────────┘    │
├─────────────────────────────┴────────────────────────────────┤
│  Per-Organization Breakdown                    [Filter] [⬇]  │
│  ┌──────────┬────────┬──────┬───────┬────────┬───────────┐   │
│  │ Tag      │ Total  │ High │ Med   │ Trend  │ Last Scan │   │
│  ├──────────┼────────┼──────┼───────┼────────┼───────────┤   │
│  │ client_a │ 45     │ 12   │ 20    │ ↗ +5   │ 2h ago    │   │
│  │ client_b │ 23     │ 3    │ 8     │ ↘ -2   │ 1d ago    │   │
│  │ client_c │ 67     │ 25   │ 30    │ → 0    │ 3h ago    │   │
│  └──────────┴────────┴──────┴───────┴────────┴───────────┘   │
└──────────────────────────────────────────────────────────────┘
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
