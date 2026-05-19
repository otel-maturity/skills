---
name: dimension-1-integration-surface
version: 0.0.1-dev
description: Evaluate Dimension 1 (Integration Surface) of the OTel Support Maturity Model. Assesses how the project exposes telemetry to users and how well it integrates into existing OpenTelemetry pipelines without requiring adapters, sidecars, or legacy exporters.
argument-hint: "<project-name> <version>"
allowed-tools:
  - Bash
  - Read
  - WebFetch
  - WebSearch
---

# Evaluate Dimension 1: Integration Surface

## What this dimension measures

How users connect a project to their observability pipelines — whether telemetry is tightly coupled to specific tools/vendors or integrates naturally into OpenTelemetry-native environments.

**Key question:** Is OTLP the default/primary export path, or does the project require adapters, sidecars, or project-specific configuration?

This dimension is less about *what* telemetry is emitted and more about *how* users consume it. A mature integration surface minimizes coupling to specific backends, allows configuration via standard env vars, and fits naturally into OpenTelemetry Collector pipelines.

**Critical rule:** Evaluate the project's own integration surface. Downstream Collector adapters or sidecars do **not** raise the level — they represent integration friction.

## Required arguments

- `<project-name>` — the CNCF project being evaluated
- `<version>` — the evaluation run version tag (e.g. `v1`)

## Inputs

Telemetry files at `/tmp/otel-eval-<project-name>/`:
- `traces.jsonl`, `metrics.jsonl`, `logs.jsonl`

Research notes: `.otel-eval/<project-name>/RESEARCH.md`

## Evaluation steps

### Step 1: Determine which signals are flowing via OTLP

The presence of data in the JSONL files is direct evidence that OTLP export is working for each signal.

```bash
# How many batches per signal
wc -l /tmp/otel-eval-<project-name>/*.jsonl

# Instrumentation scopes (reveals SDK or exporter type)
cat /tmp/otel-eval-<project-name>/traces.jsonl | \
  jq -r '.resourceSpans[]?.scopeSpans[]?.scope | "\(.name // "unknown") v\(.version // "unknown")"' | sort -u

cat /tmp/otel-eval-<project-name>/metrics.jsonl | \
  jq -r '.resourceMetrics[]?.scopeMetrics[]?.scope | "\(.name // "unknown") v\(.version // "unknown")"' | sort -u

cat /tmp/otel-eval-<project-name>/logs.jsonl | \
  jq -r '.resourceLogs[]?.scopeLogs[]?.scope | "\(.name // "unknown") v\(.version // "unknown")"' | sort -u
```

Note: absence of data in a signal file means that signal is not flowing via OTLP — this is a negative indicator for integration surface maturity.

### Step 2: Review installation research notes

Read `.otel-eval/<project-name>/RESEARCH.md` and extract:
- What configuration was needed to get each signal flowing (OTEL_* env vars vs project-specific flags)
- Whether a sidecar, adapter, or Collector component was required as a bridge
- Whether OTLP was enabled by default or required explicit activation

### Step 3: Search official documentation

```
WebSearch: "<project-name> opentelemetry otlp configuration"
WebSearch: "<project-name> observability telemetry documentation"
WebSearch: "<project-name> OTEL_EXPORTER_OTLP_ENDPOINT"
```

For each result, look for:
- Is OTLP documented as the recommended/default export path?
- Are legacy exporters (Jaeger, Zipkin) still prominently documented alongside OTLP?
- Are `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_EXPORTER_OTLP_PROTOCOL`, `OTEL_SERVICE_NAME` documented?
- Is there one clear "connect to your pipeline" guide, or multiple conflicting approaches?

### Step 4: Check for signs of per-signal configuration inconsistency

```bash
# Do different signals use different scope names, suggesting different export paths?
echo "=== Trace scopes ===" && \
  cat /tmp/otel-eval-<project-name>/traces.jsonl | \
  jq -r '.resourceSpans[]?.scopeSpans[]?.scope.name // "unknown"' | sort -u

echo "=== Metric scopes ===" && \
  cat /tmp/otel-eval-<project-name>/metrics.jsonl | \
  jq -r '.resourceMetrics[]?.scopeMetrics[]?.scope.name // "unknown"' | sort -u

echo "=== Log scopes ===" && \
  cat /tmp/otel-eval-<project-name>/logs.jsonl | \
  jq -r '.resourceLogs[]?.scopeLogs[]?.scope.name // "unknown"' | sort -u
```

## Level determination

Work through each level's checklist, bottom-up. The assigned level is the **highest** where the project substantially meets the characteristics. Record YES/NO for each question with your evidence.

### Level 0 — Instrumented

Telemetry exists but OpenTelemetry is not a primary integration concern.

| Question | Answer | Evidence |
|----------|--------|----------|
| Is telemetry exported only via tool-specific or legacy exporters (Jaeger only, Prometheus scrape only)? | | |
| Is OTLP unsupported or available only indirectly via sidecars/adapters? | | |
| Does telemetry configuration rely entirely on project-specific flags (`--enable-tracing`, `telemetry.enabled=true`)? | | |
| Do users need to adapt their observability stack to fit the project's model? | | |
| Is OpenTelemetry absent from docs or treated as an afterthought? | | |

If most are YES → **Level 0**.

### Level 1 — OpenTelemetry-Aligned

OpenTelemetry is supported but not central.

| Question | Answer | Evidence |
|----------|--------|----------|
| Is OTLP supported alongside equally-promoted legacy exporters (documented side-by-side)? | | |
| Are there multiple overlapping ways to configure telemetry (project flags AND OTEL_* variables)? | | |
| Does OTLP require disabling legacy behavior or enabling experimental flags? | | |
| Is OpenTelemetry integration inconsistent across signals (e.g. traces via OTLP, metrics via Prometheus scrape only)? | | |
| Do users need to read multiple docs pages to get a working OTLP integration? | | |

If the project goes beyond Level 0 but still has these traits → **Level 1**.

### Level 2 — OpenTelemetry-Native

OpenTelemetry is the primary integration surface.

| Question | Answer | Evidence |
|----------|--------|----------|
| Is OTLP the default or clearly-recommended export path in docs? | | |
| Are `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_EXPORTER_OTLP_PROTOCOL`, `OTEL_SERVICE_NAME` respected end-to-end? | | |
| Can users connect to an existing OTel Collector without adapters or glue code? | | |
| Are legacy exporters clearly secondary, optional, or deprecated? | | |
| Is telemetry configuration consistent across all signals (not just traces)? | | |

If most are YES → **Level 2**.

### Level 3 — OpenTelemetry-Optimized

The integration surface is intentionally designed, governed, and stable.

| Question | Answer | Evidence |
|----------|--------|----------|
| Is the telemetry integration surface documented as a stable contract? | | |
| Are telemetry integration changes reviewed like API changes? | | |
| Are breaking changes communicated with migration guidance? | | |
| Does the project explicitly support diverse deployment models (local dev, Kubernetes, managed platforms)? | | |
| Are legacy integrations removed or tightly scoped with clear deprecation timelines? | | |

If most are YES → **Level 3**.

## Output

Write the result to `.otel-eval/<project-name>/dim-1-integration-surface.md` using this format:

```markdown
### 1. Integration Surface

**Level: <0-3> — <level name>**

#### Evidence

- **Signals flowing via OTLP**: <traces / metrics / logs — list present ones; note absent ones>
- **Configuration method**: <OTEL_* env vars / project-specific flags / mixed>
- **Documentation stance**: <OTLP primary / OTLP alongside legacy / legacy only / undocumented>
- **Legacy exporter status**: <deprecated / optional / co-equal / primary>
- **Signals requiring adapters/sidecars**: <list or "none">

<specific observations from telemetry files, docs, and research notes>

#### Checklist assessment

<completed table from level determination — include YES/NO and brief evidence for each row>

#### Rationale

<why this specific level was chosen, referencing concrete evidence above>
```
