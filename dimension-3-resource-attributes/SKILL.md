---
name: dimension-3-resource-attributes
version: 0.0.1-dev
description: Evaluate Dimension 3 (Resource Attributes & Configuration) of the OTel Support Maturity Model. Assesses how the project expresses service identity across signals and whether standard OTEL_* environment variables are respected.
argument-hint: "<project-name> <version>"
allowed-tools:
  - Bash
  - Read
  - WebFetch
  - WebSearch
---

# Evaluate Dimension 3: Resource Attributes & Configuration

## What this dimension measures

How stable and consistent service identity is across signals, and whether standard OpenTelemetry configuration mechanisms (`OTEL_*` env vars) work end-to-end.

**Key question:** Is `service.name` stable, consistent across traces/metrics/logs, and overridable via `OTEL_SERVICE_NAME`?

**Critical distinction:** Resource attributes injected by the OTel Collector's `k8sattributes` processor (pod name, namespace, node) are **pipeline-derived**, not native. They do **not** count toward a higher maturity level for this dimension. Focus on what the project emits at the source.

## Required arguments

- `<project-name>` — the CNCF project being evaluated
- `<version>` — the evaluation run version tag (e.g. `v1`)

## Inputs

Telemetry files at `/tmp/otel-eval-<project-name>/`:
- `traces.jsonl`, `metrics.jsonl`, `logs.jsonl`

Research notes: `.otel-eval/<project-name>/RESEARCH.md` (contains pre-enrichment observations)

Collector logs (pre-enrichment view):
```bash
kubectl logs -n opentelemetry -l app.kubernetes.io/instance=otel-collector --tail=200 | head -300
```

## Evaluation steps

### Step 1: Extract resource attributes per signal

```bash
# Resource attributes on traces
echo "=== Trace resource attributes ===" && \
  cat /tmp/otel-eval-<project-name>/traces.jsonl | \
  jq -r '.resourceSpans[]? | .resource.attributes[]? | "\(.key): \(.value.stringValue // .value.intValue // .value.boolValue // "?")"' | \
  sort -u

# Resource attributes on metrics
echo "=== Metric resource attributes ===" && \
  cat /tmp/otel-eval-<project-name>/metrics.jsonl | \
  jq -r '.resourceMetrics[]? | .resource.attributes[]? | "\(.key): \(.value.stringValue // .value.intValue // .value.boolValue // "?")"' | \
  sort -u

# Resource attributes on logs
echo "=== Log resource attributes ===" && \
  cat /tmp/otel-eval-<project-name>/logs.jsonl | \
  jq -r '.resourceLogs[]? | .resource.attributes[]? | "\(.key): \(.value.stringValue // .value.intValue // .value.boolValue // "?")"' | \
  sort -u
```

### Step 2: Check service.name consistency across signals

```bash
echo "=== service.name across signals ===" && \
  echo "Traces:" && \
  cat /tmp/otel-eval-<project-name>/traces.jsonl | \
  jq -r '.resourceSpans[]?.resource.attributes[]? | select(.key == "service.name") | .value.stringValue' | sort -u && \
  echo "Metrics:" && \
  cat /tmp/otel-eval-<project-name>/metrics.jsonl | \
  jq -r '.resourceMetrics[]?.resource.attributes[]? | select(.key == "service.name") | .value.stringValue' | sort -u && \
  echo "Logs:" && \
  cat /tmp/otel-eval-<project-name>/logs.jsonl | \
  jq -r '.resourceLogs[]?.resource.attributes[]? | select(.key == "service.name") | .value.stringValue' | sort -u

echo "=== service.version across signals ===" && \
  echo "Traces:" && \
  cat /tmp/otel-eval-<project-name>/traces.jsonl | \
  jq -r '.resourceSpans[]?.resource.attributes[]? | select(.key == "service.version") | .value.stringValue' | sort -u && \
  echo "Metrics:" && \
  cat /tmp/otel-eval-<project-name>/metrics.jsonl | \
  jq -r '.resourceMetrics[]?.resource.attributes[]? | select(.key == "service.version") | .value.stringValue' | sort -u && \
  echo "Logs:" && \
  cat /tmp/otel-eval-<project-name>/logs.jsonl | \
  jq -r '.resourceLogs[]?.resource.attributes[]? | select(.key == "service.version") | .value.stringValue' | sort -u
```

### Step 3: Distinguish native vs pipeline-derived attributes

The collector's `k8sattributes` processor enriches telemetry with Kubernetes metadata. Compare what arrives at the Collector vs what ends up in the JSONL files.

```bash
# Collector debug logs show what arrives BEFORE enrichment
kubectl logs -n opentelemetry -l app.kubernetes.io/instance=otel-collector --tail=200 2>/dev/null | \
  grep -E '"resource"' | head -10
```

Also read `.otel-eval/<project-name>/RESEARCH.md` for notes on enrichment configuration.

Classify each resource attribute:
- **Native** (project emits it): `service.name`, `service.version`, `telemetry.sdk.*`, `process.*`
- **Pipeline-derived** (added by k8sattributes/enrichment): `k8s.pod.name`, `k8s.namespace.name`, `k8s.node.name`, `k8s.deployment.name`, `k8s.cluster.name`, `container.id`

### Step 4: Check for identity attributes misplaced as span/metric attributes

```bash
# Are service.name or service.version being set as span attributes (wrong scope)?
cat /tmp/otel-eval-<project-name>/traces.jsonl | \
  jq -r '.resourceSpans[]?.scopeSpans[]?.spans[]?.attributes[]? | 
    select(.key | test("^service\\.|^deployment\\.|^cloud\\.")) | .key' | sort -u

# Are environment or version attributes duplicated on spans?
cat /tmp/otel-eval-<project-name>/traces.jsonl | \
  jq -r '.resourceSpans[]?.scopeSpans[]?.spans[]?.attributes[]?.key' | \
  grep -E '^(env|environment|version|region|cluster)$' | sort -u
```

### Step 5: Check for service.instance.id

```bash
cat /tmp/otel-eval-<project-name>/traces.jsonl | \
  jq -r '.resourceSpans[]?.resource.attributes[]? | select(.key == "service.instance.id") | .value.stringValue' | sort -u
```

### Step 6: Check documentation for OTEL_* variable support

```
WebSearch: "<project-name> OTEL_SERVICE_NAME OTEL_RESOURCE_ATTRIBUTES"
WebSearch: "<project-name> opentelemetry environment variables configuration"
```

Look for:
- Are `OTEL_SERVICE_NAME` and `OTEL_RESOURCE_ATTRIBUTES` documented?
- Are they documented as working, or only as "experimental"?
- Is there guidance on configuration precedence (project defaults vs OTEL_* vars)?
- Are there known issues with env var support?

## Level determination

Work through each level's checklist. Assign the **highest** level where the project substantially meets the characteristics.

### Level 0 — Instrumented

Identity is implicit, hard-coded, or inconsistent.

| Question | Answer | Evidence |
|----------|--------|----------|
| Is `service.name` hard-coded or always the same generic value (e.g. always "proxy", "app")? | | |
| Does `service.name` differ between signals (different value in traces vs metrics)? | | |
| Are `service.version` and instance identity absent? | | |
| Are identity attributes placed on spans instead of resources? | | |
| Is `OTEL_RESOURCE_ATTRIBUTES` ignored or overridden? | | |

### Level 1 — OpenTelemetry-Aligned

Some resource attributes exist but behavior is inconsistent.

| Question | Answer | Evidence |
|----------|--------|----------|
| Is `service.name` present and stable but `service.version` missing? | | |
| Is configuration precedence between project config and `OTEL_*` unclear? | | |
| Are Kubernetes/platform attributes only available through Collector enrichment? | | |
| Does identity differ between signals or exporters? | | |
| Does `OTEL_RESOURCE_ATTRIBUTES` work only in some environments? | | |

### Level 2 — OpenTelemetry-Native

Resource attributes are the single source of identity, set consistently at the source.

| Question | Answer | Evidence |
|----------|--------|----------|
| Is `service.name` present, stable, and consistent across traces/metrics/logs? | | |
| Is `service.version` present and consistent? | | |
| Are `OTEL_SERVICE_NAME` and `OTEL_RESOURCE_ATTRIBUTES` respected end-to-end? | | |
| Are identity attributes in resource scope, not duplicated on spans? | | |
| Are Kubernetes attributes available via standard OTel resource detection (even if pipeline-derived)? | | |

### Level 3 — OpenTelemetry-Optimized

Resource modeling and configuration are intentional, governed, and documented.

| Question | Answer | Evidence |
|----------|--------|----------|
| Is resource attribute behavior explicitly documented? | | |
| Is configuration precedence (project defaults vs `OTEL_*`) clearly explained? | | |
| Are identity changes treated as breaking changes? | | |
| Are resource attributes immutable at runtime (no runtime mutation of `service.name`)? | | |
| Does documentation explain identity behavior across shared clusters/multi-tenant deployments? | | |

## Output

Write the result to `.otel-eval/<project-name>/dim-3-resource-attributes.md`:

```markdown
### 3. Resource Attributes & Configuration

**Level: <0-3> — <level name>**

#### Evidence

##### Native resource attributes (emitted by the project)
<list attributes the project emits natively, with values observed>

##### Pipeline-derived resource attributes (added by Collector enrichment)
<list attributes added by k8sattributes or other processors — these do NOT count toward the level>

##### service.name consistency across signals
- Traces: <value>
- Metrics: <value>
- Logs: <value>
- Consistent: <yes/no>

##### service.version presence
<present/absent — value if present>

##### OTEL_* env var support
<documented/undocumented — tested behavior if available in research notes>

##### Identity misplacement
<any identity attributes found on spans instead of resources>

#### Checklist assessment

<completed tables from level determination>

#### Rationale

<why this specific level was chosen>
```
