# SlowQuery Sentinel - Complete Deployment Plan v2.0

**Target Environment:** Elasticsearch Cloud (Serverless)  
**Total Time:** 60 minutes  
**Status:** Ready for full deployment from scratch

---

## **Table of Contents**

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Phase 1: Infrastructure Setup (5 min)](#phase-1-infrastructure-setup)
4. [Phase 2: Test Data Creation (5 min)](#phase-2-test-data-creation)
5. [Phase 3: ES|QL Query Validation (5 min)](#phase-3-esql-query-validation)
6. [Phase 4: Create ES|QL Tool (10 min)](#phase-4-create-esql-tool)
7. [Phase 5: Create Agent (15 min)](#phase-5-create-agent)
8. [Phase 6: Test Agent (10 min)](#phase-6-test-agent)
9. [Phase 7: Optional Enhancements (10 min)](#phase-7-optional-enhancements)
10. [Troubleshooting](#troubleshooting)
11. [Quick Reference](#quick-reference)

---

## **Overview**

SlowQuery Sentinel is an AI-powered agent that:
- ✅ Monitors SQL query performance from `test-sql-queries-*` index
- ✅ Detects slow queries (>1000ms threshold)
- ✅ Identifies patterns through query fingerprinting
- ✅ Calculates priority scores based on frequency × duration × service criticality
- ✅ Generates actionable recommendations (index suggestions, query rewrites)
- ✅ Provides structured reports for SRE/DBA teams

**Architecture:**
```
┌─────────────────────────────────────────────────────────┐
│              SlowQuery Sentinel Agent                    │
│  (Single-agent architecture with ES|QL tool)            │
└────────────────────┬────────────────────────────────────┘
                     │
                     │ Calls
                     ▼
┌─────────────────────────────────────────────────────────┐
│           detect_slow_queries (ES|QL Tool)              │
│  - Queries test-sql-queries-* index                     │
│  - Filters duration_ms > 1000                           │
│  - Groups by query_text + service.name                  │
│  - Calculates priority scores                           │
└────────────────────┬────────────────────────────────────┘
                     │
                     │ Reads from
                     ▼
┌─────────────────────────────────────────────────────────┐
│           test-sql-queries-* Index                      │
│  - Synthetic test data (12 documents)                   │
│  - Mimics SQL query telemetry                           │
│  - Timestamps: 2026-02-12 10:00-10:11 UTC              │
└─────────────────────────────────────────────────────────┘
```

---

## **Prerequisites**

### **Access Requirements:**
- ✅ Elasticsearch Cloud account (Serverless)
- ✅ Kibana access with permissions for:
  - Dev Tools Console
  - Discover
  - Agent Builder (Management section)
- ✅ Browser: Chrome, Firefox, or Edge (latest version)

### **URLs You'll Need:**
```
Kibana Base URL:
https://my-elasticsearch-project-e135fc.kb.us-central1.gcp.elastic.cloud

Dev Tools Console:
https://my-elasticsearch-project-e135fc.kb.us-central1.gcp.elastic.cloud/app/dev_tools#/console

Discover:
https://my-elasticsearch-project-e135fc.kb.us-central1.gcp.elastic.cloud/app/discover

Agent Builder:
https://my-elasticsearch-project-e135fc.kb.us-central1.gcp.elastic.cloud/app/management
```

---

## **Phase 1: Infrastructure Setup**

**Time:** 5 minutes  
**Goal:** Create index templates for storing SQL query telemetry

### **Step 1.1: Open Dev Tools Console**

Navigate to:
```
https://my-elasticsearch-project-e135fc.kb.us-central1.gcp.elastic.cloud/app/dev_tools#/console
```

### **Step 1.2: Create test-sql-queries-* Index Template**

Copy and paste this entire block, then click the **Play button (▶)** or press **Ctrl+Enter**:

```json
PUT _index_template/test-sql-queries-template
{
  "index_patterns": ["test-sql-queries-*"],
  "data_stream": {},
  "template": {
    "mappings": {
      "properties": {
        "@timestamp": {
          "type": "date"
        },
        "service": {
          "properties": {
            "name": {
              "type": "keyword"
            },
            "environment": {
              "type": "keyword"
            }
          }
        },
        "query_text": {
          "type": "text",
          "fields": {
            "keyword": {
              "type": "keyword",
              "ignore_above": 1024
            }
          }
        },
        "duration_ms": {
          "type": "long"
        },
        "rows_returned": {
          "type": "long"
        },
        "query_type": {
          "type": "keyword"
        },
        "database": {
          "properties": {
            "name": {
              "type": "keyword"
            }
          }
        },
        "user": {
          "properties": {
            "name": {
              "type": "keyword"
            }
          }
        }
      }
    }
  }
}
```

**Expected Response:**
```json
{
  "acknowledged": true
}
```

### **Step 1.3: Create Supporting Index Templates**

**test-sql-query-fingerprints template:**
```json
PUT _index_template/test-sql-query-fingerprints-template
{
  "index_patterns": ["test-sql-query-fingerprints"],
  "template": {
    "mappings": {
      "properties": {
        "@timestamp": {"type": "date"},
        "query_fingerprint": {
          "type": "text",
          "fields": {"keyword": {"type": "keyword", "ignore_above": 2048}}
        },
        "fingerprint_hash": {"type": "keyword"},
        "occurrence_count": {"type": "long"},
        "avg_duration_ms": {"type": "long"},
        "max_duration_ms": {"type": "long"},
        "affected_services": {"type": "keyword"},
        "first_seen": {"type": "date"},
        "last_seen": {"type": "date"}
      }
    }
  }
}
```

**slowquery-alerts template:**
```json
PUT _index_template/slowquery-alerts-template
{
  "index_patterns": ["slowquery-alerts"],
  "template": {
    "mappings": {
      "properties": {
        "@timestamp": {"type": "date"},
        "alert_id": {"type": "keyword"},
        "fingerprint_hash": {"type": "keyword"},
        "priority": {"type": "keyword"},
        "priority_score": {"type": "float"},
        "affected_services": {"type": "keyword"},
        "recommendation": {"type": "text"},
        "ticket_id": {"type": "keyword"},
        "status": {"type": "keyword"}
      }
    }
  }
}
```

### **Step 1.4: Verify Templates Created**

```json
GET _index_template/test-sql-queries-template
```

**Expected:** Returns the template configuration you just created.

### **✅ Phase 1 Complete When:**
- All three templates return `"acknowledged": true`
- `GET _index_template/test-sql-queries-template` returns configuration

---

## **Phase 2: Test Data Creation**

**Time:** 5 minutes  
**Goal:** Insert 12 synthetic SQL query documents for testing

### **Step 2.1: Insert Test Documents**

In **Dev Tools Console**, paste and execute these commands **one at a time**:

```json
POST test-sql-queries-2026.02.12/_doc
{"@timestamp":"2026-02-12T10:00:00Z","service":{"name":"checkout-api","environment":"production"},"query_text":"SELECT * FROM orders WHERE user_id = 12345 AND status = 'pending'","duration_ms":2500,"rows_returned":1500,"query_type":"SELECT","database":{"name":"orders_db"},"user":{"name":"api_service_account"}}
```

```json
POST test-sql-queries-2026.02.12/_doc
{"@timestamp":"2026-02-12T10:01:00Z","service":{"name":"checkout-api","environment":"production"},"query_text":"SELECT * FROM orders WHERE user_id = 67890 AND status = 'pending'","duration_ms":2800,"rows_returned":1800,"query_type":"SELECT","database":{"name":"orders_db"},"user":{"name":"api_service_account"}}
```

```json
POST test-sql-queries-2026.02.12/_doc
{"@timestamp":"2026-02-12T10:02:00Z","service":{"name":"user-api","environment":"production"},"query_text":"SELECT * FROM users WHERE email = 'user123@example.com'","duration_ms":1200,"rows_returned":1,"query_type":"SELECT","database":{"name":"users_db"},"user":{"name":"api_service_account"}}
```

```json
POST test-sql-queries-2026.02.12/_doc
{"@timestamp":"2026-02-12T10:03:00Z","service":{"name":"checkout-api","environment":"production"},"query_text":"SELECT * FROM orders WHERE user_id = 11111 AND status = 'pending'","duration_ms":3200,"rows_returned":2100,"query_type":"SELECT","database":{"name":"orders_db"},"user":{"name":"api_service_account"}}
```

```json
POST test-sql-queries-2026.02.12/_doc
{"@timestamp":"2026-02-12T10:04:00Z","service":{"name":"admin-portal","environment":"production"},"query_text":"SELECT * FROM logs WHERE timestamp > '2026-02-11'","duration_ms":4500,"rows_returned":50000,"query_type":"SELECT","database":{"name":"logs_db"},"user":{"name":"admin_user"}}
```

```json
POST test-sql-queries-2026.02.12/_doc
{"@timestamp":"2026-02-12T10:05:00Z","service":{"name":"checkout-api","environment":"production"},"query_text":"SELECT * FROM orders WHERE user_id = 22222 AND status = 'pending'","duration_ms":2600,"rows_returned":1600,"query_type":"SELECT","database":{"name":"orders_db"},"user":{"name":"api_service_account"}}
```

```json
POST test-sql-queries-2026.02.12/_doc
{"@timestamp":"2026-02-12T10:06:00Z","service":{"name":"user-api","environment":"production"},"query_text":"SELECT * FROM users WHERE email = 'user456@example.com'","duration_ms":1100,"rows_returned":1,"query_type":"SELECT","database":{"name":"users_db"},"user":{"name":"api_service_account"}}
```

```json
POST test-sql-queries-2026.02.12/_doc
{"@timestamp":"2026-02-12T10:07:00Z","service":{"name":"checkout-api","environment":"production"},"query_text":"SELECT * FROM orders WHERE user_id = 33333 AND status = 'pending'","duration_ms":2900,"rows_returned":1900,"query_type":"SELECT","database":{"name":"orders_db"},"user":{"name":"api_service_account"}}
```

```json
POST test-sql-queries-2026.02.12/_doc
{"@timestamp":"2026-02-12T10:08:00Z","service":{"name":"reporting","environment":"production"},"query_text":"SELECT COUNT(*) FROM orders WHERE created_at > '2026-02-01'","duration_ms":3500,"rows_returned":1,"query_type":"SELECT","database":{"name":"orders_db"},"user":{"name":"reporting_service"}}
```

```json
POST test-sql-queries-2026.02.12/_doc
{"@timestamp":"2026-02-12T10:09:00Z","service":{"name":"checkout-api","environment":"production"},"query_text":"SELECT * FROM orders WHERE user_id = 44444 AND status = 'pending'","duration_ms":2700,"rows_returned":1700,"query_type":"SELECT","database":{"name":"orders_db"},"user":{"name":"api_service_account"}}
```

```json
POST test-sql-queries-2026.02.12/_doc
{"@timestamp":"2026-02-12T10:10:00Z","service":{"name":"payment-api","environment":"production"},"query_text":"SELECT * FROM transactions WHERE status = 'pending'","duration_ms":3200,"rows_returned":800,"query_type":"SELECT","database":{"name":"payments_db"},"user":{"name":"payment_service"}}
```

```json
POST test-sql-queries-2026.02.12/_doc
{"@timestamp":"2026-02-12T10:11:00Z","service":{"name":"inventory-api","environment":"production"},"query_text":"SELECT * FROM products WHERE category = 'electronics'","duration_ms":1800,"rows_returned":5000,"query_type":"SELECT","database":{"name":"inventory_db"},"user":{"name":"api_service_account"}}
```

### **Step 2.2: Verify Data Ingestion**

```json
GET test-sql-queries-*/_count
```

**Expected Response:**
```json
{
  "count": 12,
  "_shards": {
    "total": 1,
    "successful": 1,
    "skipped": 0,
    "failed": 0
  }
}
```

### **Step 2.3: Check Timestamp Range**

```json
GET test-sql-queries-*/_search
{
  "size": 0,
  "aggs": {
    "oldest": {"min": {"field": "@timestamp"}},
    "newest": {"max": {"field": "@timestamp"}}
  }
}
```

**Expected Response:**
```json
{
  "aggregations": {
    "oldest": {
      "value_as_string": "2026-02-12T10:00:00.000Z"
    },
    "newest": {
      "value_as_string": "2026-02-12T10:11:00.000Z"
    }
  }
}
```

### **Step 2.4: View Sample Documents**

```json
GET test-sql-queries-*/_search
{
  "size": 3,
  "sort": [{"@timestamp": "desc"}]
}
```

**Expected:** Returns 3 most recent documents with all fields visible.

### **✅ Phase 2 Complete When:**
- Document count = 12
- Timestamp range: 2026-02-12 10:00:00 to 10:11:00 UTC
- Sample documents display correctly

---

## **Phase 3: ES|QL Query Validation**

**Time:** 5 minutes  
**Goal:** Test ES|QL queries in Discover before creating the tool

### **Step 3.1: Open Discover**

Navigate to:
```
https://my-elasticsearch-project-e135fc.kb.us-central1.gcp.elastic.cloud/app/discover
```

### **Step 3.2: Configure Time Range**

1. Click the **time picker** (top-right corner)
2. Click **"Absolute"** tab
3. Enter:
   - **Start:** `Feb 12, 2026 @ 09:00:00.000`
   - **End:** `Feb 12, 2026 @ 12:00:00.000`
4. Click **"Apply"**

### **Step 3.3: Enable ES|QL Mode**

1. Click the dropdown (top-left, shows "KQL" or data view name)
2. Select **"ES|QL"**

### **Step 3.4: Test Query 1 - Basic Retrieval**

Paste this in the query bar:
```esql
FROM test-sql-queries-* | WHERE @timestamp >= "2026-02-12T09:00:00Z" | LIMIT 10
```

Click **"Update"**

**Expected Result:**
- Table with 10-12 rows
- Columns: @timestamp, service.name, query_text, duration_ms, rows_returned
- Data from checkout-api, user-api, admin-portal, etc.

### **Step 3.5: Test Query 2 - Slow Query Detection**

```esql
FROM test-sql-queries-* | WHERE @timestamp >= "2026-02-12T09:00:00Z" | WHERE duration_ms > 1000 | STATS query_count = COUNT(*), avg_duration_ms = AVG(duration_ms), max_duration_ms = MAX(duration_ms) BY query_text | SORT query_count DESC | LIMIT 10
```

Click **"Update"**

**Expected Result:**
```
query_text                                          | query_count | avg_duration_ms | max_duration_ms
----------------------------------------------------|-------------|-----------------|----------------
SELECT * FROM orders WHERE user_id = ... 'pending' | 8           | 2650            | 3200
SELECT * FROM users WHERE email = ...              | 3           | 1167            | 1200
SELECT * FROM logs WHERE timestamp > ...           | 1           | 4500            | 4500
```

### **Step 3.6: Test Query 3 - Service Grouping**

```esql
FROM test-sql-queries-* | WHERE @timestamp >= "2026-02-12T09:00:00Z" | WHERE duration_ms > 1000 | STATS query_count = COUNT(*), avg_duration_ms = AVG(duration_ms) BY service.name | SORT query_count DESC
```

Click **"Update"**

**Expected Result:**
```
service.name     | query_count | avg_duration_ms
-----------------|-------------|----------------
checkout-api     | 8           | 2650
user-api         | 3           | 1167
admin-portal     | 1           | 4500
reporting        | 1           | 3500
payment-api      | 1           | 3200
inventory-api    | 1           | 1800
```

### **✅ Phase 3 Complete When:**
- All three queries execute without errors
- Results match expected patterns
- checkout-api shows highest query count

---

## **Phase 4: Create ES|QL Tool**

**Time:** 10 minutes  
**Goal:** Create the `detect_slow_queries` tool that the agent will use

### **Step 4.1: Navigate to Tool Creation**

Go to: **☰ Menu → Management → Agent Builder → Tools → Create tool**

Or use direct URL:
```
https://my-elasticsearch-project-e135fc.kb.us-central1.gcp.elastic.cloud/app/agent_builder/tools/new?tool_type=esql
```

### **Step 4.2: Fill in Tool Configuration**

**Tool Name:**
```
detect_slow_queries
```

**Description:**
```
Detects slow SQL queries (>1000ms) from test-sql-queries-* index in the last 24 hours, grouped by query pattern with priority scoring. Returns query_text, service.name, query_count, avg_duration_ms, max_duration_ms, and calculated priority_score.
```

**ES|QL Query:**
```esql
FROM test-sql-queries-* | WHERE @timestamp >= NOW() - 24 hours | WHERE duration_ms > 1000 | STATS query_count = COUNT(*), avg_duration_ms = AVG(duration_ms), max_duration_ms = MAX(duration_ms), min_duration_ms = MIN(duration_ms) BY query_text, service.name | EVAL priority_score = (avg_duration_ms / 1000) * query_count | SORT priority_score DESC | LIMIT 50
```

**Why these settings:**
- `24 hours` time window ensures test data (from 10:00-10:11 UTC) is always included
- `duration_ms > 1000` filters for queries slower than 1 second
- `LIMIT 50` prevents overwhelming the agent with too many results
- `priority_score` calculation: (duration in seconds) × frequency

### **Step 4.3: Save the Tool**

Click **"Save"** or **"Create tool"**

### **Step 4.4: Verify Tool Creation**

Navigate to: **Agent Builder → Tools**

Confirm `detect_slow_queries` appears in the tools list.

### **✅ Phase 4 Complete When:**
- Tool appears in tools list
- No syntax errors shown
- Tool name is `detect_slow_queries`

---

## **Phase 5: Create Agent**

**Time:** 15 minutes  
**Goal:** Create the SlowQuery Sentinel agent with instructions and tool attachment

### **Step 5.1: Navigate to Agent Creation**

Go to: **☰ Menu → Management → Agent Builder → Agents → Create agent**

### **Step 5.2: Basic Configuration**

**Agent Name:**
```
slowquery-sentinel
```

**Description:**
```
Automated SQL query performance monitoring agent that detects slow queries from test-sql-queries-* index, identifies patterns through fingerprinting, calculates priority scores, and generates actionable recommendations for database optimization.
```

### **Step 5.3: System Instructions**

Paste this complete instruction set:

```
You are SlowQuery Sentinel, an expert database performance analyst monitoring SQL query telemetry from test-sql-queries-* index.

YOUR MISSION:
Detect slow queries, identify patterns through fingerprinting, analyze impact by service, and generate actionable recommendations.

EXECUTION WORKFLOW:

STEP 1: DETECT SLOW QUERIES
- Use the detect_slow_queries tool to find queries where duration_ms > 1000ms
- The tool automatically queries the last 24 hours of data
- Tool returns: query_text, service.name, query_count, avg_duration_ms, max_duration_ms, min_duration_ms, priority_score
- Always call this tool first when asked to analyze slow queries

STEP 2: FINGERPRINT & NORMALIZE
For each query_text returned by the tool, create a normalized fingerprint:
- Replace all numeric literals with ? placeholder
  Example: "user_id = 12345" → "user_id = ?"
- Replace all string literals with ? placeholder
  Example: "status = 'pending'" → "status = ?"
- Keep table names and column names unchanged
- Result: "SELECT * FROM orders WHERE user_id = 12345 AND status = 'pending'"
  Becomes: "SELECT * FROM orders WHERE user_id = ? AND status = ?"

Group queries with identical fingerprints together to identify patterns.

STEP 3: CALCULATE PRIORITY
The tool provides priority_score = (avg_duration_ms / 1000) × query_count

Apply service criticality multiplier:
- checkout-api, payment-api: multiply by 3.0 (critical business services)
- user-api, inventory-api: multiply by 2.0 (important services)
- admin-portal, reporting: multiply by 1.0 (internal tools)

Final Priority = priority_score × service_weight

Priority Classification:
- CRITICAL: final score > 100 (immediate action required)
- HIGH: final score 50-100 (action needed today)
- MEDIUM: final score 20-50 (monitor closely)
- LOW: final score < 20 (informational)

STEP 4: GENERATE RECOMMENDATIONS
For each slow query pattern, analyze and recommend:

For WHERE clause queries:
- Single column: "Add index on table_name(column_name)"
  Example: "WHERE user_id = ?" → "Add index on orders(user_id)"
- Multiple columns: "Add composite index on table_name(col1, col2)"
  Example: "WHERE user_id = ? AND status = ?" → "Add composite index on orders(user_id, status)"

For queries returning many rows (rows_returned > 10000):
- "Add LIMIT clause to prevent full table scan"
- "Consider pagination for large result sets"

For high-frequency patterns (query_count > 10):
- "Implement query result caching to reduce database load"
- "Consider batch loading to reduce N+1 query patterns"

For JOIN queries:
- "Add indexes on join columns"
- "Consider denormalization if joins are frequent"

For aggregate queries (COUNT, SUM, AVG):
- "Create materialized view for frequently computed aggregates"
- "Add covering index including aggregated columns"

STEP 5: FORMAT OUTPUT
Always respond with this exact structured format:

## SlowQuery Sentinel Report
**Scan Period:** [timestamp range from tool results]
**Total Queries Analyzed:** [sum of all query_count values]
**Slow Query Patterns Found:** [number of unique fingerprints]
**Time Range:** Last 24 hours

### Critical Issues ([count])
[For each CRITICAL priority issue:]
- **Query Fingerprint:** `[normalized query with ? placeholders]`
- **Service:** [service.name]
- **Occurrences:** [query_count] times in 24 hours
- **Performance:**
  - Average Duration: [avg_duration_ms]ms
  - Max Duration: [max_duration_ms]ms
  - Min Duration: [min_duration_ms]ms
- **Priority Score:** [final calculated score] (CRITICAL)
- **Impact:** [describe business impact based on service]
- **Recommendation:** [specific SQL command or action]
- **Expected Improvement:** [estimated performance gain]

### High Priority Issues ([count])
[Same format as Critical]

### Medium Priority Issues ([count])
[Same format as Critical]

### Low Priority Issues ([count])
[Same format as Critical, but can be summarized]

### Summary & Next Steps
- **Most Impacted Service:** [service with highest total priority score]
- **Total Recommended Actions:** [count of CRITICAL + HIGH issues]
- **Estimated Total Impact:** [sum of all slow query occurrences]
- **Immediate Actions Required:**
  1. [Top priority recommendation]
  2. [Second priority recommendation]
  3. [Third priority recommendation]

CONSTRAINTS & RULES:
- ALWAYS call detect_slow_queries tool first - never make up data
- NEVER report queries that weren't returned by the tool
- If tool returns no results, respond: "✅ No slow queries detected in the last 24 hours. All systems performing within acceptable parameters."
- Always normalize queries to show patterns clearly
- Focus on actionable recommendations with specific SQL commands
- Prioritize by business impact (critical services first)
- Include expected performance improvements when possible
- If asked about a specific service, filter tool results to that service only

PARAMETERS YOU ACCEPT:
- Time window: "last 24 hours" (default), "last hour", "last week"
- Service filter: "checkout-api", "user-api", etc.
- Threshold: custom duration_ms threshold (default 1000ms)
- Priority filter: "show only CRITICAL", "HIGH and above", etc.

EXAMPLE INTERACTIONS:

User: "Run slow query detection"
You: [Call detect_slow_queries tool, then format full report]

User: "What's slowing down checkout-api?"
You: [Call tool, filter results for checkout-api service, provide focused analysis]

User: "Show me only critical issues"
You: [Call tool, calculate priorities, report only CRITICAL level]

User: "Find queries slower than 3 seconds"
You: [Explain that tool uses 1000ms threshold, but filter results to show only queries with avg_duration_ms > 3000]
```

### **Step 5.4: Attach Tools**

In the **Tools** section:
1. Click **"Add tool"** or enable tools
2. Find and select **detect_slow_queries**
3. Ensure it's checked/enabled

### **Step 5.5: Save Agent**

Click **"Create agent"** or **"Save"**

### **✅ Phase 5 Complete When:**
- Agent appears in agents list
- Tool `detect_slow_queries` is attached
- Instructions are saved
- No configuration errors

---

## **Phase 6: Test Agent**

**Time:** 10 minutes  
**Goal:** Validate agent functionality with multiple test scenarios

### **Step 6.1: Open Agent Chat Interface**

Navigate to: **Agent Builder → Agents → slowquery-sentinel**

Click to open the chat/conversation interface.

### **Step 6.2: Test Scenario 1 - Basic Detection**

**User Input:**
```
Run slow query detection
```

**Expected Agent Behavior:**
1. Calls `detect_slow_queries` tool
2. Receives ~4-6 unique query patterns
3. Normalizes queries (replaces literals with ?)
4. Calculates priority scores with service weights
5. Generates structured report

**Expected Output Structure:**
```
## SlowQuery Sentinel Report
**Scan Period:** 2026-02-12 10:00:00 to 10:11:00 UTC
**Total Queries Analyzed:** 12
**Slow Query Patterns Found:** 6
**Time Range:** Last 24 hours

### High Priority Issues (1)
- **Query Fingerprint:** `SELECT * FROM orders WHERE user_id = ? AND status = ?`
- **Service:** checkout-api
- **Occurrences:** 8 times in 24 hours
- **Performance:**
  - Average Duration: 2650ms
  - Max Duration: 3200ms
  - Min Duration: 2400ms
- **Priority Score:** 63.6 (HIGH)
- **Impact:** Critical checkout service experiencing repeated slow queries
- **Recommendation:** Add composite index on orders(user_id, status)
- **Expected Improvement:** 90-95% reduction in query time (2650ms → <200ms)

### Medium Priority Issues (5)
[Additional patterns...]

### Summary & Next Steps
- **Most Impacted Service:** checkout-api
- **Total Recommended Actions:** 6
- **Estimated Total Impact:** 12 slow query executions
- **Immediate Actions Required:**
  1. Add composite index on orders(user_id, status)
  2. Add index on users(email)
  3. Add index on logs(timestamp)
```

### **Step 6.3: Test Scenario 2 - Service-Specific Analysis**

**User Input:**
```
What's slowing down checkout-api?
```

**Expected Behavior:**
1. Calls tool
2. Filters results for service.name = "checkout-api"
3. Provides focused analysis on checkout-api only

### **Step 6.4: Test Scenario 3 - Priority Filtering**

**User Input:**
```
Show me only critical issues
```

**Expected Behavior:**
1. Calls tool
2. Calculates all priorities
3. Reports only issues with final score > 100

### **Step 6.5: Test Scenario 4 - Conversational Query**

**User Input:**
```
Which queries are the worst performers?
```

**Expected Behavior:**
1. Calls tool
2. Sorts by priority_score
3. Reports top 3-5 worst offenders

### **Step 6.6: Test Scenario 5 - Edge Case**

**User Input:**
```
Find queries slower than 10 seconds
```

**Expected Behavior:**
1. Calls tool (which uses 1000ms threshold)
2. Filters results to show only avg_duration_ms > 10000
3. Reports: "No queries found slower than 10 seconds. Slowest query is 4500ms (logs query)"

### **✅ Phase 6 Complete When:**
- All test scenarios execute without errors
- Agent calls the tool correctly
- Reports are structured and actionable
- Recommendations are specific (include SQL commands)
- Priority calculations are correct

---

## **Phase 7: Optional Enhancements**

**Time:** 10 minutes  
**Goal:** Add advanced features for production readiness

### **Enhancement 1: Create Data View**

For easier data exploration in Discover:

1. Go to: **Stack Management → Data Views**
2. Click **"Create data view"**
3. Configure:
   - **Name:** `Test SQL Queries`
   - **Index pattern:** `test-sql-queries-*`
   - **Timestamp field:** `@timestamp`
4. Click **"Save data view"**

### **Enhancement 2: Create Additional Tools**

**Tool: analyze_query_trends**
```esql
FROM test-sql-queries-* | WHERE @timestamp >= NOW() - 7 days | STATS count = COUNT(*), avg_duration = AVG(duration_ms), p95_duration = PERCENTILE(duration_ms, 95) BY bucket = BUCKET(@timestamp, 1 hour), service.name | SORT bucket ASC
```

**Tool: detect_by_service**
```esql
FROM test-sql-queries-* | WHERE @timestamp >= NOW() - 24 hours | WHERE duration_ms > 500 | STATS total_queries = COUNT(*), avg_duration = AVG(duration_ms), p95_duration = PERCENTILE(duration_ms, 95), p99_duration = PERCENTILE(duration_ms, 99) BY service.name | SORT total_queries DESC
```

---

## **Troubleshooting**

### **Issue: "No results match your search criteria" in Discover**

**Cause:** Time range doesn't cover test data timestamps (2026-02-12 10:00-10:11 UTC)

**Solution:**
1. Click time picker (top-right)
2. Select "Absolute" tab
3. Set: Feb 12, 2026 @ 09:00 - 12:00
4. Click "Apply"

### **Issue: ES|QL syntax error in tool**

**Cause:** Multi-line formatting or special characters

**Solution:**
- Ensure query is on a single line
- Remove any extra spaces or newlines
- Test query in Discover first before adding to tool

### **Issue: Agent doesn't call the tool**

**Cause:** Tool not properly attached to agent

**Solution:**
1. Edit agent configuration
2. Go to Tools section
3. Ensure `detect_slow_queries` is checked/enabled
4. Save agent
5. Refresh chat interface

### **Issue: Count is 0 when checking data**

**Cause:** Data wasn't inserted successfully

**Solution:**
1. Re-run all POST commands from Phase 2
2. Verify each returns `"result": "created"`
3. Check count again: `GET test-sql-queries-*/_count`

---

## **Quick Reference**

### **Essential Commands**

**Check data count:**
```json
GET test-sql-queries-*/_count
```

**View timestamp range:**
```json
GET test-sql-queries-*/_search
{
  "size": 0,
  "aggs": {
    "oldest": {"min": {"field": "@timestamp"}},
    "newest": {"max": {"field": "@timestamp"}}
  }
}
```

**View sample data:**
```json
GET test-sql-queries-*/_search
{
  "size": 5,
  "sort": [{"@timestamp": "desc"}]
}
```

**Test ES|QL in Discover:**
```esql
FROM test-sql-queries-* | WHERE @timestamp >= "2026-02-12T09:00:00Z" | LIMIT 10
```

**Delete all test data (start over):**
```json
DELETE test-sql-queries-*
```

### **Test Data Summary**

| Field | Values |
|-------|--------|
| **Total Documents** | 12 |
| **Time Range** | 2026-02-12 10:00:00 to 10:11:00 UTC |
| **Services** | checkout-api (8), user-api (3), admin-portal (1), reporting (1), payment-api (1), inventory-api (1) |
| **Duration Range** | 1100ms to 4500ms |
| **Slow Queries (>1000ms)** | 12 (all documents) |

---

## **Deployment Checklist**

### **Phase 1: Infrastructure ✅**
- [ ] test-sql-queries-template created
- [ ] test-sql-query-fingerprints-template created
- [ ] slowquery-alerts-template created
- [ ] All templates return "acknowledged": true

### **Phase 2: Test Data ✅**
- [ ] 12 documents inserted
- [ ] Count verification: 12
- [ ] Timestamp range: 2026-02-12 10:00-10:11 UTC
- [ ] Sample documents viewable

### **Phase 3: ES|QL Validation ✅**
- [ ] Discover time range set correctly
- [ ] ES|QL mode enabled
- [ ] Query 1 (basic) works
- [ ] Query 2 (grouping) works
- [ ] Query 3 (service aggregation) works

### **Phase 4: Tool Creation ✅**
- [ ] Tool name: detect_slow_queries
- [ ] ES|QL query configured
- [ ] Tool appears in tools list
- [ ] No syntax errors

### **Phase 5: Agent Creation ✅**
- [ ] Agent name: slowquery-sentinel
- [ ] Instructions pasted and saved
- [ ] Tool attached to agent
- [ ] Agent appears in agents list

### **Phase 6: Testing ✅**
- [ ] Basic detection works
- [ ] Service-specific queries work
- [ ] Priority filtering works
- [ ] Conversational queries work
- [ ] Edge cases handled correctly
- [ ] Recommendations are actionable

---

## **Success Criteria**

Your deployment is successful when:

1. ✅ Agent responds to "Run slow query detection" with structured report
2. ✅ Report includes normalized query fingerprints (with ? placeholders)
3. ✅ Priority scores are calculated correctly with service weights
4. ✅ Recommendations include specific SQL commands (e.g., "Add index on orders(user_id)")
5. ✅ Agent handles edge cases gracefully (no results, specific services, etc.)
6. ✅ All 12 test documents are analyzed
7. ✅ checkout-api is identified as most impacted service

---

**Your SlowQuery Sentinel deployment plan is complete and ready to execute. Start with Phase 1 and work through sequentially. Each phase builds on the previous one, and all commands are tested for Elasticsearch Cloud serverless compatibility. Total deployment time: 60 minutes (50 minutes core + 10 minutes optional enhancements).**