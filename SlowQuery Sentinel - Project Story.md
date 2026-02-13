## Inspiration
Slow queries silently kill database performance and waste SRE hours hunting invisible bottlenecks. SlowQuery Sentinel was inspired by real-world observability pain—teams drowning in query logs but missing proactive detection. We built it for the Elasticsearch Agent Builder Hackathon to showcase how AI agents can automate messy database triage using ES|QL tools and multi-step reasoning.

## What it does
SlowQuery Sentinel continuously monitors SQL telemetry in Elasticsearch, detects long-running queries (>1s by default), fingerprints them for patterns, analyzes impact by service/user, and triggers actions like Slack alerts or Jira tickets. Ask it "What's slowing prod today?" and it delivers prioritized fixes with explanations—no manual SQL digging required.

## How we built it
Using Elastic Agent Builder:
- Indexed SQL telemetry (duration_ms, query_text, timestamp) from APM/logs into sql-queries-*.
- Created ES|QL tool: row timestamp, service, duration_ms, query_text | where duration_ms > ?threshold | sort @timestamp desc.
- Built agent with custom instructions: "Detect → fingerprint → recommend indexes/rewrites → action via workflow."
- Added Search tool for historical context and Workflow for ticketing.
- Scheduled via A2A endpoint + cron for continuous monitoring.

## Challenges we ran into
- Parameterizing ES|QL for dynamic thresholds while preventing huge result sets (solved with LIMIT + stats bucketing).
- Agent hallucination on query explanations—fixed by strict tool-only instructions and query fingerprint normalization.
- Real-time vs batch detection: A2A latency for scheduled runs needed tuning for sub-1min alerts.

Accomplishments that we're proud of
- End-to-end automation: From raw logs → detection → human-readable report + auto-ticket in under 30s.
- Multi-step reasoning: Agent chains ES|QL (detect), Search (context), Workflow (act) flawlessly.
- Zero-config deploy: Paste agent config into Agent Builder, point at your indices, done.

## What we learned
- Agent Builder shines for observability—ES|QL tools handle heavy lifting, letting LLM focus on reasoning/orchestration. 
- Tight instructions prevent tool misuse. A2A endpoints unlock cron-powered "long-running agents" beyond chat UIs. 
- Fingerprinting queries via hashing was key for pattern detection.

## What's next for SlowQuery Sentinel
- Anomaly detection: Baseline query patterns, alert on deviations.
- Multi-DB: MySQL/Postgres/SQL Server query normalization.
- Self-healing: Generate EXPLAIN ANALYZE, suggest indexes, auto-apply in staging.
- Open source the agent YAML + sample data generator for SRE teams.