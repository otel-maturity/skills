---
name: dimension-4-trace-modeling
version: 0.0.1
description: Evaluate Dimension 4 (Trace Modeling & Context Propagation) of the OTel Support Maturity Model. Assesses how traces are structured, whether parent-child relationships are intentional, and whether W3C Trace Context propagates correctly through synchronous and asynchronous paths.
argument-hint: "<project-name> <version>"
allowed-tools:
  - Bash
  - Read
  - WebFetch
  - WebSearch
---

# Evaluate Dimension 4: Trace Modeling & Context Propagation

## What this dimension measures

How traces are structured and how trace context flows through the system. This goes beyond "spans exist" — it asks whether traces tell a coherent, user-comprehensible story about system execution.

**Key question:** Do traces represent logical operations the user cares about, with correct parent-child relationships, or do they fragment into disconnected spans that only maintainers can interpret?

**Critical distinction:** Trace structure reflects modeling decisions. Fragmented traces from asynchronous paths indicate Level 0-1 even if some traces look correct.

## Required arguments

- `<project-name>` — the CNCF project being evaluated
- `<version>` — the evaluation run version tag (e.g. `v1`)

## Inputs

Telemetry files at `/tmp/otel-eval-<project-name>/traces.jsonl`

## Evaluation steps

### Step 1: Analyze span structure — root spans, depth, parent-child relationships

```bash
# Count root spans (no parentSpanId) vs child spans
echo "=== Root spans (no parent) ===" && \
  cat /tmp/otel-eval-<project-name>/traces.jsonl | \
  jq -r '.resourceSpans[]?.scopeSpans[]?.spans[]? | 
    select(.parentSpanId == "" or .parentSpanId == null) | .name' | \
  sort | uniq -c | sort -rn | head -20

echo "=== Child spans (have parent) ===" && \
  cat /tmp/otel-eval-<project-name>/traces.jsonl | \
  jq -r '.resourceSpans[]?.scopeSpans[]?.spans[]? | 
    select(.parentSpanId != "" and .parentSpanId != null) | .name' | \
  sort | uniq -c | sort -rn | head -20

# Total spans by parentage
echo "Total root spans:" && \
  cat /tmp/otel-eval-<project-name>/traces.jsonl | \
  jq -r '.resourceSpans[]?.scopeSpans[]?.spans[]? | 
    select(.parentSpanId == "" or .parentSpanId == null) | .traceId' | wc -l

echo "Total child spans:" && \
  cat /tmp/otel-eval-<project-name>/traces.jsonl | \
  jq -r '.resourceSpans[]?.scopeSpans[]?.spans[]? | 
    select(.parentSpanId != "" and .parentSpanId != null) | .traceId' | wc -l
```

### Step 2: Check span kinds

Correct span kinds indicate intentional trace modeling. Entry-point spans should be `SERVER` (2), outgoing calls `CLIENT` (3), async producers `PRODUCER` (4), consumers `CONSUMER` (5).

```bash
# Span kinds for root spans (entry points)
echo "=== Root span kinds ===" && \
  cat /tmp/otel-eval-<project-name>/traces.jsonl | \
  jq -r '.resourceSpans[]?.scopeSpans[]?.spans[]? | 
    select(.parentSpanId == "" or .parentSpanId == null) | 
    "\(.name) | kind=\(.kind // 0)"' | sort | uniq -c | sort -rn | head -20

# All span kinds distribution (0=UNSPECIFIED, 1=INTERNAL, 2=SERVER, 3=CLIENT, 4=PRODUCER, 5=CONSUMER)
echo "=== Span kind distribution ===" && \
  cat /tmp/otel-eval-<project-name>/traces.jsonl | \
  jq -r '.resourceSpans[]?.scopeSpans[]?.spans[]?.kind // 0' | \
  sort | uniq -c | sort -rn
```

### Step 3: Check W3C Trace Context propagation

```bash
# How many distinct trace IDs exist? Many single-span traces = fragmentation
echo "=== Distinct trace IDs ===" && \
  cat /tmp/otel-eval-<project-name>/traces.jsonl | \
  jq -r '.resourceSpans[]?.scopeSpans[]?.spans[]?.traceId' | sort -u | wc -l

echo "=== Spans per trace distribution (top 20 traces by span count) ===" && \
  cat /tmp/otel-eval-<project-name>/traces.jsonl | \
  jq -r '.resourceSpans[]?.scopeSpans[]?.spans[]?.traceId' | \
  sort | uniq -c | sort -rn | head -20

# Check for W3C traceparent header being extracted (look for http.request.header.traceparent or similar)
cat /tmp/otel-eval-<project-name>/traces.jsonl | \
  jq -r '.resourceSpans[]?.scopeSpans[]?.spans[]?.attributes[]?.key' | \
  grep -i 'traceparent\|tracestate\|trace.parent\|b3' | sort -u
```

### Step 4: Check for span links (used for async/fan-out patterns)

```bash
# Spans that use links (intentional cross-trace relationships)
echo "=== Spans with links ===" && \
  cat /tmp/otel-eval-<project-name>/traces.jsonl | \
  jq -r '.resourceSpans[]?.scopeSpans[]?.spans[]? | 
    select(.links != null and (.links | length) > 0) | 
    "\(.name) — \(.links | length) link(s)"' | sort | uniq -c | sort -rn
```

### Step 5: Assess trace coherence — are single-span traces dominant?

```bash
# Traces with only 1 span (potential fragmentation indicator)
echo "=== Single-span traces ===" && \
  cat /tmp/otel-eval-<project-name>/traces.jsonl | \
  jq -r '.resourceSpans[]?.scopeSpans[]?.spans[]?.traceId' | \
  sort | uniq -c | awk '$1 == 1 {print $2}' | wc -l

echo "=== Multi-span traces ===" && \
  cat /tmp/otel-eval-<project-name>/traces.jsonl | \
  jq -r '.resourceSpans[]?.scopeSpans[]?.spans[]?.traceId' | \
  sort | uniq -c | awk '$1 > 1 {print $2}' | wc -l
```

### Step 6: Check span status codes

```bash
# Span status distribution (0=UNSET, 1=OK, 2=ERROR)
echo "=== Span status distribution ===" && \
  cat /tmp/otel-eval-<project-name>/traces.jsonl | \
  jq -r '.resourceSpans[]?.scopeSpans[]?.spans[]?.status.code // 0' | \
  sort | uniq -c | sort -rn

# Error spans — check if they have meaningful status messages
cat /tmp/otel-eval-<project-name>/traces.jsonl | \
  jq -r '.resourceSpans[]?.scopeSpans[]?.spans[]? | 
    select(.status.code == 2) | 
    "\(.name): \(.status.message // "no message")"' | head -10
```

### Step 7: Check documentation for trace modeling design

```
WebSearch: "<project-name> opentelemetry tracing context propagation documentation"
WebSearch: "<project-name> distributed tracing spans"
```

Look for:
- Is W3C Trace Context (`traceparent` header) documented as supported?
- Are there docs on how traces flow through the system?
- Is async/background work propagation addressed?

## Level determination

Work through each level's checklist. Assign the **highest** level where the project substantially meets the characteristics.

### Level 0 — Instrumented

Spans exist but trace structure is accidental or misleading.

| Question | Answer | Evidence |
|----------|--------|----------|
| Do most traces consist of a single isolated span with no parent or children? | | |
| Do requests produce multiple unrelated traces instead of one coherent trace? | | |
| Are root spans created arbitrarily (no consistent SERVER kind at entry points)? | | |
| Is context propagation absent (no incoming traceparent support)? | | |
| Does async/background work create detached traces with new trace IDs? | | |

### Level 1 — OpenTelemetry-Aligned

Tracing works for simple synchronous flows but breaks for complex paths.

| Question | Answer | Evidence |
|----------|--------|----------|
| Do synchronous HTTP request paths produce multi-span coherent traces? | | |
| Does context propagation break for async execution, background jobs, or fan-out? | | |
| Are span links used inconsistently or as a patch for propagation failures? | | |
| Do retries, redirects, or internal forwarding start new traces? | | |
| Is trace behavior undocumented or implicit? | | |

### Level 2 — OpenTelemetry-Native

Trace modeling is intentional and documented.

| Question | Answer | Evidence |
|----------|--------|----------|
| Is W3C Trace Context supported and propagated consistently at entry points? | | |
| Are parent-child vs span links used intentionally (not as a patch)? | | |
| Are entry-point spans consistently `SERVER` kind? | | |
| Do traces represent logical operations rather than internal function calls? | | |
| Is trace topology stable across retries, fan-out, and async execution? | | |

### Level 3 — OpenTelemetry-Optimized

Trace modeling is actively refined, validated, and evolved.

| Question | Answer | Evidence |
|----------|--------|----------|
| Does trace topology support complex async or graph-shaped workflows? | | |
| Are trace modeling decisions reviewed architecturally? | | |
| Are trade-offs between trace completeness, cost, and clarity explicit? | | |
| Is trace behavior tested or validated over time? | | |
| Do span links, events, and attributes enrich understanding intentionally? | | |

## Output

Write the result to `.otel-eval/<project-name>/dim-4-trace-modeling.md` using the `writeFile` tool. This path is **relative to your working directory `/app`**; do **not** write the report under `/tmp/otel-eval-<project-name>/` (that directory holds the input telemetry files, not the output report) or downstream pipeline steps will not find your result.

Use this format:

```markdown
### 4. Trace Modeling & Context Propagation

**Level: <0-3> — <level name>**

#### Evidence

##### Span structure
- Total root spans: <count>
- Total child spans: <count>
- Multi-span traces: <count>
- Single-span traces: <count>
- Span kind distribution: UNSPECIFIED=<n>, INTERNAL=<n>, SERVER=<n>, CLIENT=<n>, PRODUCER=<n>, CONSUMER=<n>

##### Context propagation
<W3C Trace Context support — observed evidence or documented behavior>

##### Async/background work
<does async work continue or break the trace? specific span names observed>

##### Span links usage
<present/absent — used intentionally or as a patch?>

##### Trace coherence assessment
<can a user follow a logical operation through the trace, or must they stitch multiple fragments?>

#### Checklist assessment

<completed tables from level determination>

#### Rationale

<why this specific level was chosen>
```
