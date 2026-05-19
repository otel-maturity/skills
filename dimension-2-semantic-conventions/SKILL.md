---
name: dimension-2-semantic-conventions
version: 0.0.1-dev
description: Evaluate Dimension 2 (Semantic Conventions) of the OTel Support Maturity Model. Assesses how consistently telemetry attribute names and meanings align with OpenTelemetry semantic conventions across traces, metrics, and logs.
argument-hint: "<project-name> <version>"
allowed-tools:
  - Bash
  - Read
  - WebFetch
  - WebSearch
---

# Evaluate Dimension 2: Semantic Conventions

## What this dimension measures

How telemetry meaning is defined and shared — whether attribute names follow OpenTelemetry semantic conventions consistently across all signals, or whether users must rely on project-specific knowledge to interpret telemetry.

**Key question:** Can a user apply off-the-shelf OTel dashboards and alerts without normalization or project-specific knowledge?

**Critical rule:** Always check against the **current stable** OpenTelemetry semantic conventions. Deprecated attributes like `http.method`, `http.status_code`, `http.url`, `http.target` must be flagged even if they were once correct.

## Required arguments

- `<project-name>` — the CNCF project being evaluated
- `<version>` — the evaluation run version tag (e.g. `v1`)

## Inputs

Telemetry files at `/tmp/otel-eval-<project-name>/`:
- `traces.jsonl`, `metrics.jsonl`, `logs.jsonl`

## Evaluation steps

### Step 1: Audit span attribute names for deprecated HTTP attributes

The most common semantic convention failure — projects using old HTTP attributes that were replaced in OTel semconv 1.20+.

```bash
# Check for deprecated HTTP attributes on spans
echo "=== DEPRECATED HTTP attributes found ===" && \
  cat /tmp/otel-eval-<project-name>/traces.jsonl | \
  jq -r '.resourceSpans[]?.scopeSpans[]?.spans[]?.attributes[]?.key' | \
  grep -E '^(http\.method|http\.status_code|http\.url|http\.target|http\.host|http\.scheme|http\.flavor|http\.user_agent|net\.peer\.name|net\.peer\.port|net\.host\.name|net\.host\.port)$' | \
  sort | uniq -c | sort -rn

# Check for current HTTP attributes
echo "=== CURRENT HTTP attributes found ===" && \
  cat /tmp/otel-eval-<project-name>/traces.jsonl | \
  jq -r '.resourceSpans[]?.scopeSpans[]?.spans[]?.attributes[]?.key' | \
  grep -E '^(http\.request\.|http\.response\.|url\.|network\.|server\.|client\.)' | \
  sort | uniq -c | sort -rn
```

### Step 2: Extract all unique span attribute keys

```bash
# All unique span attribute keys (sorted)
echo "=== All span attribute keys ===" && \
  cat /tmp/otel-eval-<project-name>/traces.jsonl | \
  jq -r '.resourceSpans[]?.scopeSpans[]?.spans[]?.attributes[]?.key' | \
  sort -u

# Span event attribute keys
cat /tmp/otel-eval-<project-name>/traces.jsonl | \
  jq -r '.resourceSpans[]?.scopeSpans[]?.spans[]?.events[]?.attributes[]?.key' | \
  sort -u
```

### Step 3: Audit metric names and attribute keys

```bash
# All metric names
echo "=== Metric names ===" && \
  cat /tmp/otel-eval-<project-name>/metrics.jsonl | \
  jq -r '.resourceMetrics[]?.scopeMetrics[]?.metrics[]?.name' | sort -u

# All metric attribute keys
echo "=== Metric attribute keys ===" && \
  cat /tmp/otel-eval-<project-name>/metrics.jsonl | \
  jq -r '[.resourceMetrics[]?.scopeMetrics[]?.metrics[]? |
    (.sum.dataPoints[]?, .gauge.dataPoints[]?, .histogram.dataPoints[]?, .summary.dataPoints[]?) |
    .attributes[]?.key] | unique[]' | sort -u

# Check metric types
cat /tmp/otel-eval-<project-name>/metrics.jsonl | \
  jq -r '.resourceMetrics[]?.scopeMetrics[]?.metrics[]? |
    "\(.name): \(if .sum then "sum(monotonic=\(.sum.isMonotonic))" elif .gauge then "gauge" elif .histogram then "histogram" elif .exponentialHistogram then "expHistogram" elif .summary then "summary" else "unknown" end)"' | sort -u
```

### Step 4: Audit log attribute keys and structure

```bash
# Log attribute keys
echo "=== Log attribute keys ===" && \
  cat /tmp/otel-eval-<project-name>/logs.jsonl | \
  jq -r '.resourceLogs[]?.scopeLogs[]?.logRecords[]?.attributes[]?.key' | sort -u

# Check if logs carry trace context (traceId, spanId)
echo "=== Logs with trace context ===" && \
  cat /tmp/otel-eval-<project-name>/logs.jsonl | \
  jq -r '.resourceLogs[]?.scopeLogs[]?.logRecords[]? | 
    select(.traceId != null and .traceId != "" and .traceId != "00000000000000000000000000000000") | 
    "has_trace_context"' | wc -l

# Log body structure (first 5)
cat /tmp/otel-eval-<project-name>/logs.jsonl | \
  jq '.resourceLogs[0]?.scopeLogs[0]?.logRecords[0:5][]?.body' 2>/dev/null | head -20
```

### Step 5: Cross-signal consistency check

Compare how the same concept is named across signals. Inconsistency between traces/metrics/logs is a Level 0-1 indicator.

```bash
# Find keys in traces
TRACE_KEYS=$(cat /tmp/otel-eval-<project-name>/traces.jsonl | \
  jq -r '.resourceSpans[]?.scopeSpans[]?.spans[]?.attributes[]?.key' | sort -u)

# Find keys in metrics  
METRIC_KEYS=$(cat /tmp/otel-eval-<project-name>/metrics.jsonl | \
  jq -r '[.resourceMetrics[]?.scopeMetrics[]?.metrics[]? |
    (.sum.dataPoints[]?, .gauge.dataPoints[]?, .histogram.dataPoints[]?) |
    .attributes[]?.key] | unique[]' | sort -u)

# Find keys in logs
LOG_KEYS=$(cat /tmp/otel-eval-<project-name>/logs.jsonl | \
  jq -r '.resourceLogs[]?.scopeLogs[]?.logRecords[]?.attributes[]?.key' | sort -u)

echo "Keys present in BOTH traces and metrics:"
comm -12 <(echo "$TRACE_KEYS") <(echo "$METRIC_KEYS")

echo "Keys present in BOTH traces and logs:"
comm -12 <(echo "$TRACE_KEYS") <(echo "$LOG_KEYS")
```

### Step 6: Check schema URL (signals intent toward semconv governance)

```bash
cat /tmp/otel-eval-<project-name>/traces.jsonl | jq -r '.resourceSpans[]?.schemaUrl // empty' | sort -u
cat /tmp/otel-eval-<project-name>/metrics.jsonl | jq -r '.resourceMetrics[]?.schemaUrl // empty' | sort -u
cat /tmp/otel-eval-<project-name>/logs.jsonl | jq -r '.resourceLogs[]?.schemaUrl // empty' | sort -u
```

### Step 7: Check docs for semantic convention references

```
WebSearch: "<project-name> opentelemetry semantic conventions attributes"
WebSearch: "<project-name> telemetry attributes documentation"
```

Look for:
- Is there documentation listing what attributes the project emits?
- Are attributes described as following OTel semantic conventions?
- Are any custom/domain-specific attributes defined and documented?

## Level determination

Work through each level's checklist. Assign the **highest** level where the project substantially meets the characteristics.

### Level 0 — Instrumented

Attribute names are ad-hoc, proprietary, or derived from internal code.

| Question | Answer | Evidence |
|----------|--------|----------|
| Are attribute names ad-hoc (e.g. `statusCode`, `resp_code`, `requestPath`)? | | |
| Are deprecated OTel attributes used (`http.method`, `http.status_code`, `http.target`)? | | |
| Is the same concept named differently across signals (traces: `http.method`, metrics: `method`, logs: `request_method`)? | | |
| Is semantic meaning encoded in span names rather than attributes? | | |
| Do users need to consult source code to understand attribute meaning? | | |

### Level 1 — OpenTelemetry-Aligned

OTel conventions partially adopted but inconsistently applied.

| Question | Answer | Evidence |
|----------|--------|----------|
| Are *some* OTel semantic conventions used (not zero)? | | |
| Are deprecated and current OTel attributes mixed (`http.status_code` AND `http.response.status_code`)? | | |
| Are conventions applied to traces but not consistently to metrics/logs? | | |
| Are similar concepts named differently across signals? | | |
| Are attribute types inconsistent (HTTP status as string vs int)? | | |

### Level 2 — OpenTelemetry-Native

Current OTel semantic conventions applied consistently across all signals.

| Question | Answer | Evidence |
|----------|--------|----------|
| Are **current, stable** OTel HTTP attributes used (`http.request.method`, `http.response.status_code`, `url.path`, `url.full`)? | | |
| Are **all** deprecated attributes removed or gated (no `http.method`, `http.status_code`, `http.target`)? | | |
| Are attribute names consistent across traces, metrics, and logs? | | |
| Are attributes placed in the correct scope (request metadata on spans, identity on resources)? | | |
| Can telemetry be interpreted using generic OTel knowledge without project-specific mapping? | | |

### Level 3 — Semantic Extension & Stewardship

Domain-specific semantics defined, documented, and governed.

| Question | Answer | Evidence |
|----------|--------|----------|
| Are domain-specific concepts modeled as explicit attributes (not overloaded span names)? | | |
| Are custom attributes documented with name, type, and semantic meaning? | | |
| Do custom attributes extend OTel conventions rather than replace them? | | |
| Are semantic changes versioned and reviewed? | | |
| If a first-class signal uses a proprietary schema where OTel semconv exists, is it explicitly documented as an extension? | | |

## Output

Write the result to `.otel-eval/<project-name>/dim-2-semantic-conventions.md`:

```markdown
### 2. Semantic Conventions

**Level: <0-3> — <level name>**

#### Evidence

##### Deprecated attributes found on spans
<list any deprecated http.*, net.* attributes — or "none found">

##### Current OTel attributes found on spans
<list current semconv attributes present>

##### Metric names and conventions
<list metric names, note OTel semconv alignment vs proprietary naming>

##### Log attributes
<list log attribute keys, note OTel alignment>

##### Cross-signal consistency
<note whether the same concept uses the same key across traces/metrics/logs>

##### Schema URL
<present / absent — per signal>

#### Checklist assessment

<completed tables from level determination>

#### Rationale

<why this specific level was chosen, referencing attribute names found above>
```
