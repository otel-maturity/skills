---
name: dimension-6-audience-signal-quality
version: 0.0.1
description: Evaluate Dimension 6 (Audience & Signal Quality) of the OTel Support Maturity Model. Assesses whether telemetry is designed for operators and users (logical operations, low noise, production-ready defaults) vs being maintainer-centric and verbose.
argument-hint: "<project-name> <version>"
allowed-tools:
  - Bash
  - Read
  - WebFetch
  - WebSearch
---

# Evaluate Dimension 6: Audience & Signal Quality

## What this dimension measures

Who telemetry is designed for and how usable it is by default. High-quality telemetry communicates meaningful system behavior with minimal noise. Low-quality telemetry exposes implementation internals that require deep project knowledge to interpret.

**Key question:** Can an operator who is unfamiliar with the project's internals use the telemetry to diagnose a production issue without customizing or filtering it heavily?

**Critical distinction:** This is about defaults and design intent, not just technical correctness. A project can emit semantically correct attributes but still score poorly here if span names expose internal function names or if logs drown operators in debug noise.

## Required arguments

- `<project-name>` — the CNCF project being evaluated
- `<version>` — the evaluation run version tag (e.g. `v1`)

## Inputs

Telemetry files at `/tmp/otel-eval-<project-name>/`:
- `traces.jsonl`, `metrics.jsonl`, `logs.jsonl`

## Evaluation steps

### Step 1: Audit span names for logical vs internal naming

This is the clearest signal of audience-awareness. Good span names describe what the system is doing from a user perspective; bad ones expose code structure.

```bash
# All unique span names with frequency
echo "=== Span names (by frequency) ===" && \
  cat /tmp/otel-eval-<project-name>/traces.jsonl | \
  jq -r '.resourceSpans[]?.scopeSpans[]?.spans[]?.name' | \
  sort | uniq -c | sort -rn | head -40
```

For each span name, classify:
- **Good** (logical, user-relevant): `HTTP GET /api/orders`, `process message`, `query users`, `authenticate request`
- **Bad** (internal, implementation detail): `processRequestV3`, `handleIngress_default`, `route_match_success`, `(*handler).ServeHTTP`
- **Neutral**: method names that are somewhat descriptive but not clearly user-facing

### Step 2: Assess log severity distribution and verbosity

```bash
# Log severity distribution
echo "=== Log severity levels ===" && \
  cat /tmp/otel-eval-<project-name>/logs.jsonl | \
  jq -r '.resourceLogs[]?.scopeLogs[]?.logRecords[]?.severityText // "UNSET"' | \
  sort | uniq -c | sort -rn

# Log record count per scope (high counts from one scope = verbosity warning)
echo "=== Log records per scope ===" && \
  cat /tmp/otel-eval-<project-name>/logs.jsonl | \
  jq -r '.resourceLogs[]?.scopeLogs[]? | "\(.scope.name // "unknown"): \(.logRecords | length)"' | \
  sort -t: -k2 -rn | head -20

# Sample log bodies to assess content quality
echo "=== Sample log bodies ===" && \
  cat /tmp/otel-eval-<project-name>/logs.jsonl | \
  jq -r '.resourceLogs[]?.scopeLogs[]?.logRecords[]?.body.stringValue // .resourceLogs[]?.scopeLogs[]?.logRecords[]?.body' | \
  head -20
```

### Step 3: Assess metric signal quality — SLO-relevant vs raw counters

```bash
# All metric names — categorize as SLO-relevant vs internal counters
echo "=== All metrics ===" && \
  cat /tmp/otel-eval-<project-name>/metrics.jsonl | \
  jq -r '.resourceMetrics[]?.scopeMetrics[]?.metrics[]? | "\(.name)"' | sort -u

# Metric attribute cardinality (high-cardinality warning)
echo "=== Metric attribute unique values (sample first metric) ===" && \
  FIRST_METRIC=$(cat /tmp/otel-eval-<project-name>/metrics.jsonl | \
    jq -r '.resourceMetrics[0]?.scopeMetrics[0]?.metrics[0]?.name' | head -1) && \
  echo "Checking cardinality for: $FIRST_METRIC" && \
  cat /tmp/otel-eval-<project-name>/metrics.jsonl | \
  jq -r --arg name "$FIRST_METRIC" \
    '.resourceMetrics[]?.scopeMetrics[]?.metrics[]? | 
     select(.name == $name) | 
     (.sum.dataPoints[]?, .gauge.dataPoints[]?, .histogram.dataPoints[]?) | 
     .attributes | map("\(.key)=\(.value.stringValue // .value.intValue // "?")") | join(", ")' | \
  sort -u | head -20
```

### Step 4: Check for high-cardinality attributes that indicate noise

```bash
# Check for URL paths or IDs in span attributes (potential high cardinality)
echo "=== Potentially high-cardinality span attributes ===" && \
  cat /tmp/otel-eval-<project-name>/traces.jsonl | \
  jq -r '.resourceSpans[]?.scopeSpans[]?.spans[]?.attributes[]? | 
    select(.key | test("url\\.full|http\\.url|request\\.id|user\\.id|session")) | 
    "\(.key): \(.value.stringValue // "?")"' | \
  sort | uniq -c | sort -rn | head -20
```

### Step 5: Check documentation for usability guidance

```
WebSearch: "<project-name> observability monitoring operations guide"
WebSearch: "<project-name> telemetry sampling filtering configuration"
```

Look for:
- Is there an operator guide for using telemetry?
- Are there recommended dashboards or alert templates?
- Is there guidance on sampling, filtering, or reducing noise?
- Are metrics described as SLO-oriented?

## Level determination

Work through each level's checklist. Assign the **highest** level where the project substantially meets the characteristics.

### Level 0 — Instrumented

Telemetry is maintainer-centric and noisy, optimized for internal debugging.

| Question | Answer | Evidence |
|----------|--------|----------|
| Do span names expose internal function/class/component names (e.g. `processRequestV3`, `(*handler).ServeHTTP`)? | | |
| Are logs emitted for every internal step by default (high volume of DEBUG/TRACE)? | | |
| Is there no distinction between debug and operational signals? | | |
| Do users need to heavily filter telemetry before it becomes useful? | | |
| Are high-cardinality attributes emitted indiscriminately (e.g. raw URL with query strings)? | | |

### Level 1 — OpenTelemetry-Aligned

Some effort to improve usability but signals still shaped by internal perspectives.

| Question | Answer | Evidence |
|----------|--------|----------|
| Is obvious noise reduced but defaults remain conservative (still more than needed)? | | |
| Are some spans renamed to logical operations but others remain internal? | | |
| Do operators need domain knowledge to interpret span names? | | |
| Is signal quality inconsistent across traces, metrics, and logs? | | |
| Are logs structured but still overly detailed for operational use? | | |

### Level 2 — OpenTelemetry-Native

Telemetry intentionally designed for its audience with sensible production defaults.

| Question | Answer | Evidence |
|----------|--------|----------|
| Do span names describe logical, user-relevant operations (e.g. `HTTP GET /orders`)? | | |
| Are telemetry defaults usable in production without customization? | | |
| Are logs emitted on state changes or errors — not on every internal step? | | |
| Are metrics focused on SLO-relevant signals rather than raw internal counters? | | |
| Can operators move from symptoms to causes without deep internal knowledge? | | |

### Level 3 — OpenTelemetry-Optimized

Signal quality is actively optimized based on real-world usage and feedback.

| Question | Answer | Evidence |
|----------|--------|----------|
| Are signal volume, cardinality, and cost managed intentionally? | | |
| Is telemetry quality evaluated using objective criteria (e.g. Instrumentation Score checks)? | | |
| Are high-cardinality attributes avoided in favor of trace-driven investigation? | | |
| Are defaults refined based on user feedback? | | |
| Are quality regressions detectable and addressed proactively? | | |

## Output

Write the result to `.otel-eval/<project-name>/dim-6-audience-signal-quality.md`:

```markdown
### 6. Audience & Signal Quality

**Level: <0-3> — <level name>**

#### Evidence

##### Span naming assessment
- **Good (logical/user-relevant)**: <list examples from the data>
- **Bad (internal/implementation)**: <list examples from the data>
- Overall: <mostly logical / mixed / mostly internal>

##### Log verbosity
- Log severity distribution: <table of counts per severity>
- Volume assessment: <high / moderate / low by default>
- Quality: <state-change events / step-by-step internal logging / mixed>

##### Metric quality
- Metrics list: <names with brief assessment>
- SLO-relevant metrics: <list or "none identified">
- High-cardinality concerns: <issues found or "none">

##### Default production usability
<can an operator use this telemetry without heavy customization? specific evidence>

#### Checklist assessment

<completed tables from level determination>

#### Rationale

<why this specific level was chosen>
```
