---
name: dimension-7-stability-change-management
version: 0.0.1
description: Evaluate Dimension 7 (Stability & Change Management) of the OTel Support Maturity Model. Assesses whether telemetry is treated as a stable public contract — whether changes are documented, communicated, and governed so users can build long-lived workflows on top of telemetry.
argument-hint: "<project-name> <version>"
allowed-tools:
  - Bash
  - Read
  - WebFetch
  - WebSearch
---

# Evaluate Dimension 7: Stability & Change Management

## What this dimension measures

How telemetry evolves over time once users depend on it. As OTel adoption matures, dashboards, alerts, runbooks, and automation depend on specific spans, attributes, and metrics. Unannounced changes silently break user workflows.

**Key question:** Can a user safely build long-lived operational workflows (dashboards, alerts, SLOs) on top of this project's telemetry without fear of silent breakage on the next upgrade?

**Primary evidence sources:** Unlike other dimensions, this one relies heavily on documentation, changelogs, and GitHub history — not just telemetry data. The telemetry files provide supporting evidence (e.g. schema URL presence), but the main evaluation is documentation-based.

## Required arguments

- `<project-name>` — the CNCF project being evaluated
- `<version>` — the evaluation run version tag (e.g. `v1`)

## Inputs

Telemetry files at `/tmp/otel-eval-<project-name>/`:
- `traces.jsonl`, `metrics.jsonl`, `logs.jsonl`

## Evaluation steps

### Step 1: Check for schema URL in OTLP exports

The presence of a schema URL signals that the project intends to version its semantic conventions — an indicator of stability intent.

```bash
echo "=== Schema URLs in telemetry ===" && \
  echo "Traces:" && \
  cat /tmp/otel-eval-<project-name>/traces.jsonl | \
  jq -r '.resourceSpans[]? | .schemaUrl // empty, .scopeSpans[]?.schemaUrl // empty' | \
  sort -u | grep -v '^$' | head -10 && \
  echo "Metrics:" && \
  cat /tmp/otel-eval-<project-name>/metrics.jsonl | \
  jq -r '.resourceMetrics[]? | .schemaUrl // empty, .scopeMetrics[]?.schemaUrl // empty' | \
  sort -u | grep -v '^$' | head -10 && \
  echo "Logs:" && \
  cat /tmp/otel-eval-<project-name>/logs.jsonl | \
  jq -r '.resourceLogs[]? | .schemaUrl // empty, .scopeLogs[]?.schemaUrl // empty' | \
  sort -u | grep -v '^$' | head -10
```

### Step 2: Search for telemetry documentation

```
WebSearch: "<project-name> telemetry observability documentation reference"
WebSearch: "<project-name> metrics spans attributes reference"
```

Look for:
- Is there a dedicated "telemetry reference" or "observability reference" page?
- Are emitted spans, metrics, and log attributes listed and described?
- Is telemetry behavior described as stable or as subject to change?
- Are any attributes/metrics labeled as "experimental" or "stable"?

### Step 3: Check changelogs and release notes for telemetry change communication

```
WebSearch: "<project-name> changelog release notes telemetry tracing metrics"
WebSearch: "site:github.com <project-name> changelog CHANGELOG"
```

For recent releases (last 3-5), look for:
- Are telemetry changes mentioned in release notes?
- Are breaking telemetry changes called out explicitly?
- Is there migration guidance when spans/metrics are renamed or removed?
- Are telemetry changes treated the same as API changes, or are they buried?

Fetch and read the changelog if available.

### Step 4: Search GitHub for telemetry stability issues

```
WebSearch: "site:github.com <project-name> issues telemetry breaking change"
WebSearch: "site:github.com <project-name> span renamed metric removed"
```

Look for:
- Open issues from users about broken dashboards/alerts after upgrades
- PRs that rename spans/metrics without migration notes
- Maintainer discussions about telemetry stability

### Step 5: Check for experimental/stable labeling in docs

```
WebSearch: "<project-name> experimental telemetry alpha beta stable"
```

Look for:
- Is there any explicit distinction between stable and experimental telemetry?
- Are there versioning commitments for telemetry?

### Step 6: Check telemetry scope/instrumentation library versions

```bash
# Scope versions may reveal versioning intent
cat /tmp/otel-eval-<project-name>/traces.jsonl | \
  jq -r '.resourceSpans[]?.scopeSpans[]?.scope | "\(.name // "unknown") v\(.version // "unknown")"' | sort -u

cat /tmp/otel-eval-<project-name>/metrics.jsonl | \
  jq -r '.resourceMetrics[]?.scopeMetrics[]?.scope | "\(.name // "unknown") v\(.version // "unknown")"' | sort -u
```

Scopes with explicit versions (not "unknown" or empty) suggest more intentional versioning of instrumentation.

## Level determination

Work through each level's checklist. Assign the **highest** level where the project substantially meets the characteristics.

### Level 0 — Instrumented

Telemetry changes are untracked and unmanaged.

| Question | Answer | Evidence |
|----------|--------|----------|
| Do span names, attributes, or metric names change without notice across releases? | | |
| Are users informed of telemetry changes only after breakage? | | |
| Is telemetry treated as an internal debugging aid with no stability expectations? | | |
| Are changes driven by implementation refactors rather than user impact? | | |
| Is there no distinction between stable and experimental telemetry? | | |
| Is schema URL absent from all signals? | | |

### Level 1 — OpenTelemetry-Aligned

Some awareness of stability exists but practices are informal.

| Question | Answer | Evidence |
|----------|--------|----------|
| Are telemetry changes mentioned in release notes inconsistently (sometimes, not always)? | | |
| Are breaking changes discovered reactively (users report broken alerts)? | | |
| Is stability handled differently per signal (traces more stable than metrics)? | | |
| Are users expected to adapt to changes without migration guidance? | | |
| Is there no clear policy for what makes a telemetry change "breaking"? | | |

### Level 2 — OpenTelemetry-Native

Telemetry is treated as part of the public contract.

| Question | Answer | Evidence |
|----------|--------|----------|
| Are telemetry changes documented clearly in release notes? | | |
| Is there a distinction between stable and experimental telemetry? | | |
| Are breaking changes explicitly called out (not buried in general notes)? | | |
| Is migration guidance provided when spans/metrics are renamed or removed? | | |
| Are changes reviewed with downstream user impact in mind? | | |

### Level 3 — OpenTelemetry-Optimized

Telemetry evolution is governed, intentional, and quality-aware.

| Question | Answer | Evidence |
|----------|--------|----------|
| Is there a defined process for reviewing telemetry changes (design proposals, TEPs)? | | |
| Are telemetry changes evaluated for impact on usability, signal quality, and cost? | | |
| Are deprecations planned and communicated with timelines? | | |
| Are migration paths standard practice (deprecated fields retained for multiple releases)? | | |
| Are telemetry regressions detected proactively (not just reactively)? | | |

## Output

Write the result to `.otel-eval/<project-name>/dim-7-stability-change-management.md`:

```markdown
### 7. Stability & Change Management

**Level: <0-3> — <level name>**

#### Evidence

##### Schema URL presence
- Traces: <present with URL / absent>
- Metrics: <present with URL / absent>
- Logs: <present with URL / absent>

##### Telemetry documentation
<is there a reference page for emitted telemetry? link if found>

##### Release note quality for telemetry changes
<quote examples from changelogs — or note absence of telemetry mentions>

##### Stable vs experimental labeling
<present / absent — examples if found>

##### User-reported stability issues
<GitHub issues or community reports about broken dashboards/alerts — or "none found">

##### Instrumentation scope versions
<list scope names and versions from telemetry data>

#### Checklist assessment

<completed tables from level determination>

#### Rationale

<why this specific level was chosen>
```
