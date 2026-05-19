---
name: evaluate-otel-maturity
version: 0.0.0-dev
description: Evaluate a CNCF project's OpenTelemetry support maturity by inspecting telemetry data, documentation, and source code. Orchestrates seven dimension skills and assembles a structured per-dimension assessment using the OpenTelemetry Support Maturity Model. Use after install-cncf-project has telemetry flowing.
argument-hint: "<project-name> <version>"
allowed-tools:
   - Bash
   - Read
   - Write
   - Edit
   - Grep
   - Glob
   - Agent
   - AskUserQuestion
   - WebFetch
   - WebSearch
---

# Evaluate OpenTelemetry Support Maturity

You evaluate a CNCF project's OpenTelemetry support using the **OpenTelemetry Support Maturity Model for CNCF Projects**. The full model specification is in `maturity-model-spec.md` in this skill's directory.

Your evaluation is orchestrated: you collect cross-cutting evidence, then invoke seven dimension skills in parallel, then assemble and present the final report.

## Context

The evaluation cluster is running with telemetry written to `/tmp/otel-eval-<project-name>/`:
- `traces.jsonl` — OTLP trace export batches
- `metrics.jsonl` — OTLP metrics export batches
- `logs.jsonl` — OTLP log export batches

Research notes from installation are in `.otel-eval/<project-name>/RESEARCH.md`.

## Required arguments

The user provides two arguments:
- `<project-name>` — the CNCF project to evaluate
- `<version>` — the evaluation run version tag (e.g. `v1`, `v2`), provided by the calling agent

## Skill version stamping

Read the `version:` field from this file's own YAML frontmatter and use that exact string wherever this skill writes the **Skill version** field. The CI workflow rewrites this field to the published version before packaging the OCI artifact.

## Before you start

1. **Read the maturity model spec** — Skim `maturity-model-spec.md` from this skill's directory to understand the model philosophy and the 7 dimensions.
2. **Read the research notes** — Read `.otel-eval/<project-name>/RESEARCH.md` for installation context.
3. **Verify telemetry exists** — Check that the JSONL files have data:
   ```bash
   wc -l /tmp/otel-eval-<project-name>/*.jsonl
   ```

## Phase 1: Collect cross-cutting evidence

Collect the evidence that spans multiple dimensions. Individual dimension skills will perform their own signal-specific analysis; your role here is to gather the shared context.

### Telemetry overview

```bash
# Signal volumes
wc -l /tmp/otel-eval-<project-name>/*.jsonl

# Resource attributes before enrichment — check collector logs
kubectl logs -n opentelemetry -l app.kubernetes.io/instance=otel-collector --tail=100 2>/dev/null | head -200

# Native resource attributes (post-enrichment, from trace file)
cat /tmp/otel-eval-<project-name>/traces.jsonl | \
  jq -r '.resourceSpans[]? | .resource.attributes[]? | "\(.key): \(.value.stringValue // .value.intValue // .value.boolValue // "?")"' | \
  sort -u

# Compare with metric resource attributes to spot inconsistency
cat /tmp/otel-eval-<project-name>/metrics.jsonl | \
  jq -r '.resourceMetrics[]? | .resource.attributes[]? | "\(.key): \(.value.stringValue // .value.intValue // .value.boolValue // "?")"' | \
  sort -u
```

## Phase 2: Collect documentation and source evidence

Use web search and web fetch to check:

1. **Official documentation** — How is OpenTelemetry documented? Is it the recommended path?
2. **Configuration reference** — Are `OTEL_*` env vars documented? Is OTLP the default?
3. **Changelog / release notes** — Are telemetry changes documented? Any breaking changes communicated?
4. **GitHub issues / PRs** — Any open issues about OTel support? Recent improvements?

Take notes on your findings — dimension skills will do their own targeted searches but your cross-cutting notes provide context.

## Phase 3: Run dimension skills in parallel

Invoke all seven dimension skills simultaneously using the Agent tool. Pass `<project-name>` and `<version>` to each.

Each skill reads telemetry directly from `/tmp/otel-eval-<project-name>/` and writes its result to `.otel-eval/<project-name>/dim-N-<name>.md`.

Run these in parallel:

1. **dimension-1-integration-surface** `<project-name> <version>`
2. **dimension-2-semantic-conventions** `<project-name> <version>`
3. **dimension-3-resource-attributes** `<project-name> <version>`
4. **dimension-4-trace-modeling** `<project-name> <version>`
5. **dimension-5-multi-signal** `<project-name> <version>`
6. **dimension-6-audience-signal-quality** `<project-name> <version>`
7. **dimension-7-stability-change-management** `<project-name> <version>`

Wait for all seven to complete before proceeding to Phase 4.

## Phase 4: Assemble the final evaluation report

Read each dimension result file and assemble `EVALUATION.md`:

```bash
cat .otel-eval/<project-name>/dim-1-integration-surface.md
cat .otel-eval/<project-name>/dim-2-semantic-conventions.md
cat .otel-eval/<project-name>/dim-3-resource-attributes.md
cat .otel-eval/<project-name>/dim-4-trace-modeling.md
cat .otel-eval/<project-name>/dim-5-multi-signal.md
cat .otel-eval/<project-name>/dim-6-audience-signal-quality.md
cat .otel-eval/<project-name>/dim-7-stability-change-management.md
```

Write `.otel-eval/<project-name>/EVALUATION.md` with this structure:

```markdown
# OpenTelemetry Support Maturity Evaluation: <project-name>

## Project overview

- **Project**: <name and brief description>
- **Version evaluated**: <version>
- **Evaluation date**: <date>
- **Evaluation run version**: <version-argument e.g. v1>
- **Cluster**: otel-eval-<project-name>
- **Maturity model version**: OpenTelemetry Support Maturity Model for CNCF Projects (draft)
- **Skill version**: evaluate-otel-maturity v<version-from-frontmatter>

## Summary

| Dimension | Level | Summary |
|-----------|-------|---------|
| Integration Surface | <0-3> | <one-line summary> |
| Semantic Conventions | <0-3> | <one-line summary> |
| Resource Attributes & Configuration | <0-3> | <one-line summary> |
| Trace Modeling & Context Propagation | <0-3> | <one-line summary> |
| Multi-Signal Observability | <0-3> | <one-line summary> |
| Audience & Signal Quality | <0-3> | <one-line summary> |
| Stability & Change Management | <0-3> | <one-line summary> |

## Telemetry overview

### Signals observed
- **Traces**: [flowing/not flowing] — [export method]
- **Metrics**: [flowing/not flowing] — [export method]
- **Logs**: [flowing/not flowing] — [export method]

### Resource attributes (native, before collector enrichment)
<list of resource attributes the project emits natively>

### Resource attributes (after collector enrichment)
<list of resource attributes after k8sattributes processing>

## Dimension evaluations

<paste assembled content from all 7 dim-N-*.md files here>

---

## Key findings

### Strengths
<bullet list of what the project does well>

### Areas for improvement
<bullet list of concrete, actionable improvements>

### Notable observations
<anything surprising, unusual, or worth highlighting>

## Methodology notes

- Telemetry was collected using an OpenTelemetry Collector with file export in a local kind cluster
- The k8sattributes processor was used to distinguish native vs enriched resource attributes
- Semantic conventions were checked against the latest stable OpenTelemetry specification
- Documentation and source code were reviewed for context beyond what telemetry data alone reveals
```

## Phase 5: Present findings

After writing the evaluation, present a concise summary to the user with:
1. The summary table
2. Top 3 strengths
3. Top 3 improvement areas
4. Any surprising findings
5. Path to the full evaluation: `.otel-eval/<project-name>/EVALUATION.md`

## Important guidance

- **Thoroughness matters.** This evaluation is meant to be a comprehensive, referenceable document.
- **Always use the latest semantic conventions.** Current HTTP: `http.request.method`, `http.response.status_code`, `url.path`, `url.full`. Flag any deprecated attributes explicitly.
- **Distinguish project-native from collector-derived.** This is the most important analytical distinction. The model evaluates what the project supports natively.
- **Quote actual data.** Every claim in the evaluation should be backed by a specific attribute name, metric name, span name, or documentation quote.
- **Be fair and constructive.** Lower maturity levels often reflect reasonable trade-offs. Acknowledge constraints while still accurately assessing the current state.
