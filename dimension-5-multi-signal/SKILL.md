---
name: dimension-5-multi-signal
version: 0.0.1
description: Evaluate Dimension 5 (Multi-Signal Observability) of the OTel Support Maturity Model. Assesses whether traces, metrics, and logs are all first-class signals, and whether they are correlated to form a coherent investigative workflow.
argument-hint: "<project-name> <version>"
allowed-tools:
  - Bash
  - Read
  - WebFetch
  - WebSearch
---

# Evaluate Dimension 5: Multi-Signal Observability

## What this dimension measures

Whether the project supports traces, metrics, and logs together — and whether those signals are correlated so users can move fluently between them during investigation (metric → trace → log) without manual bridging.

**Key question:** Can a user start from a metric anomaly, pivot to a trace, and inspect correlated logs, all without manual correlation?

**Critical rule:** "Flowing via OTLP" is Level 1 evidence. Actual correlation (log records carrying `traceId`/`spanId`, metrics sharing attribute keys with traces) is required for Level 2.

## Required arguments

- `<project-name>` — the CNCF project being evaluated
- `<version>` — the evaluation run version tag (e.g. `v1`)

## Inputs

Telemetry files at `/tmp/otel-eval-<project-name>/`:
- `traces.jsonl`, `metrics.jsonl`, `logs.jsonl`

## Evaluation steps

### Step 1: Determine which signals are present and their volumes

```bash
echo "=== Signal presence and volume ===" && \
  echo "Traces:" && wc -l /tmp/otel-eval-<project-name>/traces.jsonl && \
  echo "Metrics:" && wc -l /tmp/otel-eval-<project-name>/metrics.jsonl && \
  echo "Logs:" && wc -l /tmp/otel-eval-<project-name>/logs.jsonl

# Count actual records per signal
echo "=== Span count ===" && \
  cat /tmp/otel-eval-<project-name>/traces.jsonl | \
  jq -r '.resourceSpans[]?.scopeSpans[]?.spans[]?.traceId' | wc -l

echo "=== Metric series count ===" && \
  cat /tmp/otel-eval-<project-name>/metrics.jsonl | \
  jq -r '.resourceMetrics[]?.scopeMetrics[]?.metrics[]?.name' | wc -l

echo "=== Log record count ===" && \
  cat /tmp/otel-eval-<project-name>/logs.jsonl | \
  jq -r '.resourceLogs[]?.scopeLogs[]?.logRecords[]?.body' | wc -l
```

### Step 2: Check trace-to-log correlation — do logs carry trace context?

This is the most important cross-signal correlation check.

```bash
# Total log records
TOTAL_LOGS=$(cat /tmp/otel-eval-<project-name>/logs.jsonl | \
  jq -r '.resourceLogs[]?.scopeLogs[]?.logRecords[]?.severityText' | wc -l)
echo "Total log records: $TOTAL_LOGS"

# Logs with non-zero traceId
LOGS_WITH_TRACE=$(cat /tmp/otel-eval-<project-name>/logs.jsonl | \
  jq -r '.resourceLogs[]?.scopeLogs[]?.logRecords[]? | 
    select(.traceId != null and .traceId != "" and .traceId != "00000000000000000000000000000000") | 
    .traceId' | wc -l)
echo "Log records with trace context: $LOGS_WITH_TRACE"

# Logs with spanId
LOGS_WITH_SPAN=$(cat /tmp/otel-eval-<project-name>/logs.jsonl | \
  jq -r '.resourceLogs[]?.scopeLogs[]?.logRecords[]? | 
    select(.spanId != null and .spanId != "") | 
    .spanId' | wc -l)
echo "Log records with span context: $LOGS_WITH_SPAN"

# Sample a correlated log record
cat /tmp/otel-eval-<project-name>/logs.jsonl | \
  jq '.resourceLogs[]?.scopeLogs[]?.logRecords[]? | 
    select(.traceId != null and .traceId != "" and .traceId != "00000000000000000000000000000000") | 
    {traceId, spanId, severityText, body}' | head -20
```

### Step 3: Check cross-signal attribute alignment — metrics vs traces

For Level 2, metrics and traces should share dimension/attribute keys for the same concepts.

```bash
# Metric attribute keys
METRIC_ATTR_KEYS=$(cat /tmp/otel-eval-<project-name>/metrics.jsonl | \
  jq -r '[.resourceMetrics[]?.scopeMetrics[]?.metrics[]? |
    (.sum.dataPoints[]?, .gauge.dataPoints[]?, .histogram.dataPoints[]?) |
    .attributes[]?.key] | unique[]' | sort -u)

# Span attribute keys  
SPAN_ATTR_KEYS=$(cat /tmp/otel-eval-<project-name>/traces.jsonl | \
  jq -r '.resourceSpans[]?.scopeSpans[]?.spans[]?.attributes[]?.key' | sort -u)

echo "=== Attribute keys shared between traces and metrics ===" && \
  comm -12 <(echo "$SPAN_ATTR_KEYS") <(echo "$METRIC_ATTR_KEYS")

echo "=== Metric-only attribute keys ===" && \
  comm -23 <(echo "$METRIC_ATTR_KEYS") <(echo "$SPAN_ATTR_KEYS")
```

### Step 4: Assess collection model per signal

```bash
# What export mechanisms are in use for each signal?
# SDK/exporter names appear in scope names

echo "=== Trace scopes (may reveal SDK/exporter) ===" && \
  cat /tmp/otel-eval-<project-name>/traces.jsonl | \
  jq -r '.resourceSpans[]?.scopeSpans[]?.scope | "\(.name // "unknown") v\(.version // "?")"' | sort -u

echo "=== Metric scopes ===" && \
  cat /tmp/otel-eval-<project-name>/metrics.jsonl | \
  jq -r '.resourceMetrics[]?.scopeMetrics[]?.scope | "\(.name // "unknown") v\(.version // "?")"' | sort -u

echo "=== Log scopes ===" && \
  cat /tmp/otel-eval-<project-name>/logs.jsonl | \
  jq -r '.resourceLogs[]?.scopeLogs[]?.scope | "\(.name // "unknown") v\(.version // "?")"' | sort -u
```

### Step 5: Check documentation for multi-signal observability guidance

```
WebSearch: "<project-name> opentelemetry traces metrics logs observability"
WebSearch: "<project-name> observability documentation monitoring"
```

Look for:
- Are all three signals documented as supported?
- Is there guidance on how signals relate to each other?
- Is there an "investigating issues" or "debugging" guide that uses multiple signals?

## Level determination

Work through each level's checklist. Assign the **highest** level where the project substantially meets the characteristics.

### Level 0 — Instrumented

Only one signal is effectively usable.

| Question | Answer | Evidence |
|----------|--------|----------|
| Is only one signal treated as first-class (typically metrics only)? | | |
| Are traces or logs absent, experimental, or completely undocumented? | | |
| Is there no shared context between any signals? | | |
| Do users need to manually correlate timestamps across unrelated tools? | | |

### Level 1 — OpenTelemetry-Aligned

Multiple signals exist but are largely independent.

| Question | Answer | Evidence |
|----------|--------|----------|
| Are traces, metrics, AND logs all present (all three JSONL files have data)? | | |
| Do some signals include correlation identifiers while others do not? | | |
| Do logs lack `traceId`/`spanId` even when traces exist? | | |
| Are signals produced by different pipelines or configurations (inconsistent)? | | |
| Must users manually bridge signals (e.g. copy a trace ID into a log search)? | | |

Note: To progress beyond Level 1, all three signals must be available as first-class supported outputs **and** correlation must not depend on ad-hoc parsing.

### Level 2 — OpenTelemetry-Native

Signals are intentionally correlated and designed to work together.

| Question | Answer | Evidence |
|----------|--------|----------|
| Are traces, metrics, AND logs all first-class (present and documented)? | | |
| Do log records automatically include `traceId` and `spanId`? | | |
| Do metrics share attribute keys with traces for the same concepts? | | |
| Can users pivot from metric → trace → log without manual correlation? | | |
| Do signals complement rather than duplicate each other? | | |

### Level 3 — OpenTelemetry-Optimized

Multi-signal observability is shaped around real investigative workflows.

| Question | Answer | Evidence |
|----------|--------|----------|
| Are signal volume and cardinality managed intentionally across all signals? | | |
| Are high-cardinality metrics avoided in favor of trace-driven analysis? | | |
| Are signals shaped for common investigative paths (documented workflows)? | | |
| Is there guidance on when to use which signal? | | |
| Is the balance between cost, clarity, and depth explicit? | | |

## Output

Write the result to `.otel-eval/<project-name>/dim-5-multi-signal.md` using the `writeFile` tool. This path is **relative to your working directory `/app`**; do **not** write the report under `/tmp/otel-eval-<project-name>/` (that directory holds the input telemetry files, not the output report) or downstream pipeline steps will not find your result.

Use this format:

```markdown
### 5. Multi-Signal Observability

**Level: <0-3> — <level name>**

#### Evidence

##### Signal availability
- Traces: <flowing / not flowing> — <export mechanism>
- Metrics: <flowing / not flowing> — <export mechanism>  
- Logs: <flowing / not flowing> — <export mechanism>

##### Cross-signal correlation
- Log records with traceId: <count> of <total> (<percentage>%)
- Log records with spanId: <count>
- Shared attribute keys (traces ∩ metrics): <list or "none">

##### Collection model per signal
<describe how each signal is exported — OTLP push / Prometheus scrape / log file collection / etc.>

##### Investigative workflow assessment
<can a user follow metric → trace → log, or must they bridge manually?>

#### Checklist assessment

<completed tables from level determination>

#### Rationale

<why this specific level was chosen>
```
