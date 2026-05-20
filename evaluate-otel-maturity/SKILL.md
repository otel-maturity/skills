---
name: evaluate-otel-maturity
version: 0.0.0-dev
description: Gather cross-cutting context for a CNCF project's OpenTelemetry evaluation — inspect telemetry data, documentation, and source code to produce the project overview, telemetry overview, and key contextual findings. Used alongside dimension agents which handle per-dimension scoring.
argument-hint: "<project-name> <version>"
allowed-tools:
   - Bash
   - Read
   - Write
   - Edit
   - Grep
   - Glob
   - WebFetch
   - WebSearch
---

# Evaluate OpenTelemetry Support Maturity — Context Gathering

You gather the cross-cutting context for a CNCF project's OpenTelemetry maturity evaluation. The seven dimension agents handle per-dimension scoring independently; your role is to produce the project overview, telemetry overview, and key contextual findings that frame the full evaluation.

The full model specification is in `maturity-model-spec.md` in this skill's directory.

## Context

The evaluation cluster is running with telemetry written to `/tmp/otel-eval-<project-name>/`:
- `traces.jsonl` — OTLP trace export batches
- `metrics.jsonl` — OTLP metrics export batches
- `logs.jsonl` — OTLP log export batches

Research notes from installation are in `.otel-eval/<project-name>/RESEARCH.md`.

## Required arguments

- `<project-name>` — the CNCF project to evaluate
- `<version>` — the evaluation run version tag (e.g. `v1`, `v2`), provided by the calling agent

## Skill version stamping

Read the `version:` field from this file's own YAML frontmatter and use that exact string wherever this skill writes the **Skill version** field.

## Before you start

1. **Read the maturity model spec** — Skim `maturity-model-spec.md` to understand the model philosophy, global levels, and the summary matrix.
2. **Read the research notes** — Read `.otel-eval/<project-name>/RESEARCH.md` for installation context.
3. **Verify telemetry exists** — Check that the JSONL files have data:
   ```bash
   wc -l /tmp/otel-eval-<project-name>/*.jsonl
   ```

## Phase 1: Collect cross-cutting telemetry evidence

Collect the evidence that spans multiple dimensions. Dimension agents perform their own signal-specific analysis; your role here is to capture the shared context.

### Signal volumes

```bash
wc -l /tmp/otel-eval-<project-name>/*.jsonl
```

### Resource attributes (native, before enrichment)

```bash
# From traces
cat /tmp/otel-eval-<project-name>/traces.jsonl | \
  jq -r '.resourceSpans[]? | .resource.attributes[]? | "\(.key): \(.value.stringValue // .value.intValue // .value.boolValue // "?")"' | \
  sort -u

# From metrics — compare to spot inconsistency
cat /tmp/otel-eval-<project-name>/metrics.jsonl | \
  jq -r '.resourceMetrics[]? | .resource.attributes[]? | "\(.key): \(.value.stringValue // .value.intValue // .value.boolValue // "?")"' | \
  sort -u
```

### Resource attributes after collector enrichment

```bash
kubectl logs -n opentelemetry -l app.kubernetes.io/instance=otel-collector --tail=100 2>/dev/null | head -200
```

### Signals flowing

For each signal (traces, metrics, logs), note:
- Whether it is flowing (file has data)
- Export method (OTLP/gRPC, OTLP/HTTP, or other)
- Approximate volume (line count as a proxy)

## Phase 2: Collect documentation and source evidence

Use web search and web fetch to check:

1. **Official documentation** — How is OpenTelemetry documented? Is it the recommended observability path?
2. **Configuration reference** — Are `OTEL_*` env vars documented? Is OTLP the default export protocol?
3. **Changelog / release notes** — Are telemetry changes documented? Any breaking changes communicated?
4. **GitHub issues / PRs** — Any open issues about OTel support? Recent improvements or regressions?

Take notes — dimension agents may do their own targeted searches, but your cross-cutting notes provide framing context.

## Output: Contextual sections for EVALUATION.md

Using your findings from Phases 1 and 2, produce the following sections. These will be incorporated into the final `EVALUATION.md` by the calling agent.

### Project overview block

```markdown
## Project overview

- **Project**: <name and one-line description>
- **Repository**: <project URL>
- **Version evaluated**: <version argument>
- **Evaluation date**: <today's date>
- **Cluster**: otel-eval-<project-name>
- **Maturity model version**: OpenTelemetry Support Maturity Model for CNCF Projects (draft)
- **Skill version**: evaluate-otel-maturity v<version-from-frontmatter>
```

### Telemetry overview block

```markdown
## Telemetry overview

### Signals observed
- **Traces**: [flowing / not flowing] — [export method and approximate volume]
- **Metrics**: [flowing / not flowing] — [export method and approximate volume]
- **Logs**: [flowing / not flowing] — [export method and approximate volume]

### Resource attributes (native, before collector enrichment)
<list of key: value pairs the project emits natively>

### Resource attributes (after collector enrichment)
<list of key: value pairs added by the k8sattributes processor>
```

### Installation context summary

A brief prose paragraph (3–5 sentences) summarising the installation experience, any notable configuration steps, and whether the project required non-standard setup to get telemetry flowing. Draw from `RESEARCH.md` and your own observations.

## Important guidance

- **Distinguish project-native from collector-derived.** This is the most important analytical distinction in the model. Evaluate what the project supports natively, not what the Collector adds downstream.
- **Quote actual data.** Every claim should be backed by a specific attribute name, metric name, span name, or documentation quote.
- **Be fair and constructive.** Acknowledge constraints while accurately assessing the current state.
