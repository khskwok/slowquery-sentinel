# SlowQuery Sentinel

> An AI-powered SQL performance copilot that continuously detects long‑running queries in Elasticsearch, explains their impact, and gives engineers concrete, prioritized fixes — with zero manual log digging.

---

## Overview

SlowQuery Sentinel is an AI agent built with **Elasticsearch Agent Builder** that automates SQL performance triage for SRE and DBA teams.

It:

- Monitors SQL query telemetry stored in Elasticsearch.
- Detects long‑running queries (≥ 1000 ms by default).
- Fingerprints queries into reusable patterns.
- Scores and ranks issues by business impact.
- Generates concrete recommendations (indexes, query rewrites, pagination, caching, etc.).
- Produces structured reports that teams can act on immediately.

This project was created for the **Elasticsearch Agent Builder Hackathon** to showcase how ES|QL tools and multi‑step agents can automate messy observability workflows.

---

## Features

- **Continuous SQL performance monitoring**  
  Reads SQL telemetry from a `test-sql-queries-*` index (or your own), including timestamp, service name, query text, duration, and more.

- **Long‑running query detection**  
  Uses an ES|QL tool (`detectslowqueries`) to find queries slower than 1000 ms over the last 24 hours.

- **Query fingerprinting**  
  Normalizes query text by replacing numeric and string literals with `?` placeholders (e.g., `userid 12345` → `userid ?`) to group similar patterns.

- **Priority scoring**  
  Combines average duration, frequency, and service criticality (e.g., checkout/payment weighted higher) into a `priorityscore`.

- **Actionable recommendations**  
  Suggests specific optimizations such as single‑column/composite indexes, LIMIT/pagination, caching, and materialized views.

- **Structured reporting**  
  Returns a “SlowQuery Sentinel Report” with Critical/High/Medium/Low issues, impact descriptions, and next steps.

- **Ready‑to‑run deployment plan**  
  End‑to‑end instructions to deploy on Elasticsearch Cloud Serverless in about 60 minutes, including test data and validation queries.

---

## Architecture

SlowQuery Sentinel uses a **single‑agent architecture** with one primary ES|QL tool.

- **Data Source**
  - Index: `test-sql-queries-*`
  - Fields: `timestamp`, `service.name`, `environment`, `querytext`, `durationms`, `rowsreturned`, `querytype`, `database.name`, `user.name`.

- **ES|QL Tool: `detectslowqueries`**
  - Input: last 24 hours of data.
  - Filter: `durationms >= 1000`.
  - Aggregations: counts, avg/max/min duration by `querytext` and `service.name`.
  - Derived field: `priorityscore = (avgdurationms / 1000) * querycount`.
  - Limit: 50 rows to keep responses manageable.

- **Agent: `slowquery-sentinel`**
  - Attached tools: `detectslowqueries`.
  - System instructions define a multi‑step workflow:
    1. Detect slow queries with the tool.
    2. Normalize query fingerprints.
    3. Apply service‑based weights.
    4. Classify priorities.
    5. Generate a structured report with recommendations and expected improvements.

Optional indices and tools (fingerprints, alerts, trends) are also described in the deployment plan for future expansion.

---

## Prerequisites

- **Environment**
  - Elasticsearch Cloud (Serverless project).
  - Kibana with access to:
    - Dev Tools → Console.
    - Discover.
    - Agent Builder (Management).
  - Modern browser (Chrome, Firefox, or Edge).

- **Access**
  - Permissions to create index templates, indices, and run ES|QL queries.
  - Permissions to create and manage Agent Builder tools and agents.

---

## Quick Start (60‑Minute Deployment)

The deployment plan is organized into phases with estimated time per phase.

### 1. Phase 1 – Infrastructure Setup (≈ 5 min)

Create index templates using Dev Tools → Console.

- `test-sql-queries-template` for SQL telemetry:
  - Key fields: `timestamp`, `service.name`, `environment`, `querytext` (with keyword subfield), `durationms`, `rowsreturned`, `querytype`, `database.name`, `user.name`.
- `test-sql-query-fingerprints-template` (optional): for storing aggregated fingerprint data.
- `slowquery-alerts-template` (optional): for storing alert documents.

Verify templates with `GET _index_template/<template-name>`.

### 2. Phase 2 – Test Data Creation (≈ 5 min)

Insert **12 synthetic SQL query documents** into a `test-sql-queries-YYYY.MM.DD` index via Dev Tools.

- Mix of services: `checkout-api`, `user-api`, `admin-portal`, `reporting`, `payment-api`, `inventory-api`.
- Durations from ~1100 ms to 4500 ms.
- Queries simulate realistic workloads (e.g., orders by user, users by email, logs, reporting aggregates).

Validate:

- `GET test-sql-queries-*/_count` → `count: 12`.
- `GET test-sql-queries-*/_search` with timestamp aggregations to confirm range (10:00:00–10:11:00 UTC).

### 3. Phase 3 – ES|QL Query Validation (≈ 5 min)

In **Discover**:

1. Set an absolute time range around the test data (e.g., 09:00–12:00 on 2026‑02‑12).
2. Switch to **ES|QL** mode.
3. Run validation queries:
   - Basic retrieval:  
     `FROM test-sql-queries-* WHERE timestamp >= 2026-02-12T09:00:00Z LIMIT 10`.
   - Slow query detection by query text: counts, avg/max durations grouped by `querytext`.
   - Grouping by `service.name` to confirm service‑level patterns.

Ensure results match expected patterns (e.g., `checkout-api` has the most queries, highest averages).

### 4. Phase 4 – Create `detectslowqueries` ES|QL Tool (≈ 10 min)

In **Agent Builder → Tools → New tool (ES|QL)**:

- **Name:** `detectslowqueries`.
- **Description:** Detects slow SQL queries (≥1000 ms) from `test-sql-queries-*` in the last 24 hours, grouped by query pattern with priority scoring.
- **ES|QL query:**  
  See full text in the deployment plan; it:
  - Filters by `timestamp >= now() - 24 hours` and `durationms >= 1000`.
  - Aggregates counts and durations by `querytext`, `service.name`.
  - Computes `priorityscore`.
  - Sorts by `priorityscore DESC`.
  - Limits to 50 rows.

Save and verify the tool appears in the tools list with no syntax errors.

### 5. Phase 5 – Create SlowQuery Sentinel Agent (≈ 15 min)

In **Agent Builder → Agents → New agent**:

- **Agent name:** `slowquery-sentinel`.
- **Description:** Automated SQL query performance monitoring agent that detects slow queries, identifies patterns via fingerprinting, scores them, and generates actionable recommendations.
- **System instructions:**  
  Paste the full instruction block from the deployment plan, which defines:
  - Mission: detect slow queries, fingerprint patterns, analyze impact by service, recommend optimizations.
  - Execution workflow (steps 1–5).
  - Priority classification (Critical/High/Medium/Low).
  - Recommendation rules for WHERE clauses, large result sets, joins, aggregates, etc.
  - Strict constraints (always call the tool first, never invent data, handle empty results gracefully).
  - Output format for the SlowQuery Sentinel Report.
- **Tools:** attach `detectslowqueries` and ensure it is enabled.

Save the agent.

### 6. Phase 6 – Test the Agent (≈ 10 min)

Open the agent chat and run test prompts.

Examples:

- `Run slow query detection`  
  Expect a full structured report using all 12 test documents.
- `What's slowing down checkout-api?`  
  Expect a focused analysis on `checkout-api` only.
- `Show me only critical issues`  
  Expect filtering by final score ≥ critical threshold.
- `Find queries slower than 10 seconds`  
  Expect no results and a clear explanation that the slowest is 4500 ms.

Your deployment is complete when:

- The agent calls the tool correctly.
- Reports include normalized query fingerprints with `?`.
- Priority scores incorporate service weights.
- Recommendations are specific and actionable.
- Edge cases (no results, specific filters) are handled gracefully.

---

## Example Interactions

Some example prompts and expected behaviors encoded in the agent instructions:

- **User:** `Run slow query detection`  
  **Agent:** Calls `detectslowqueries`, analyzes all patterns, returns full report.

- **User:** `What's slowing down checkout-api?`  
  **Agent:** Filters tool results by `service.name = "checkout-api"` and reports only those issues.

- **User:** `Show me only critical issues`  
  **Agent:** Calculates final priority scores and reports only CRITICAL issues.

- **User:** `Find queries slower than 3 seconds`  
  **Agent:** Uses the 1000 ms threshold from the tool but filters results to those with `avgdurationms >= 3000`.

---

## What We Learned

From building SlowQuery Sentinel:

- **ES|QL + Agent Builder is a powerful combo**  
  ES|QL handles heavy data analysis; the agent focuses on orchestration and explanation.

- **Tight, tool‑only instructions reduce hallucinations**  
  Constraining the agent to always use the ES|QL tool and never fabricate data made outputs reliable.

- **Query fingerprinting is crucial**  
  Normalizing queries via hashing/placeholders turned noisy logs into clear patterns that could be prioritized and optimized.

---

## Roadmap

Planned improvements:

- **Anomaly detection**  
  Baseline query patterns over time and alert on deviations.

- **Multi‑DB support**  
  Normalize queries from MySQL, Postgres, SQL Server, etc.

- **Self‑healing workflows**  
  Generate `EXPLAIN ANALYZE`, suggest indexes, and auto‑apply changes in staging environments.

- **Open source assets**  
  Publish agent configuration, sample data generator, and templates for SRE teams.

---

## Repository Structure (Suggested)

```text
.
├── README.md                     # This file
├── agent/
│   ├── slowquery-sentinel-agent-config.yaml   # Agent Builder JSON/YAML export (if available)
│   └── tools/
│       └── detectslowqueries-esql.txt        # ES|QL query for the tool
├── deployment/
│   └── SlowQuery-Sentinel-Deployment-Plan-2.0.md
├── docs/
│   └── SlowQuery-Sentinel-Project-Story.md
└── examples/
    └── sample-requests.md        # Example prompts and sample responses
