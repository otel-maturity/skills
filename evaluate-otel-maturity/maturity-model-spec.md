# **OpenTelemetry Support Maturity Model**

# **OpenTelemetry Support Maturity Model for CNCF Projects**

# Introduction & Purpose

OpenTelemetry has become the de facto standard for producing telemetry in cloud native systems. As adoption has grown, expectations around OpenTelemetry support in CNCF projects have evolved as well. Users increasingly expect projects to integrate cleanly with existing OpenTelemetry pipelines, follow shared semantic conventions, support correlation across signals, and behave predictably across environments.

At the same time, OpenTelemetry support is not binary. Projects evolve over time, often starting with basic instrumentation and gradually moving toward more intentional, user-oriented observability design. Different aspects of OpenTelemetry support mature at different rates, depending on project scope, architecture, and available resources.

This document introduces a *maturity model for OpenTelemetry support in CNCF projects*. The goal is to provide a shared framework for:

* evaluating the current state of OpenTelemetry support  
* discussing trade-offs and improvement areas  
* guiding incremental, intentional evolution over time

The model is inspired by the [*CNCF Platform Engineering Maturity Model*](https://tag-app-delivery.cncf.io/whitepapers/platform-eng-maturity-model/#conclusion) and follows the same philosophy: descriptive rather than prescriptive, multi-dimensional rather than score-based, and focused on progression rather than compliance.

# What This Model Is and What It’s Not

This model *is*:

* a tool for reflection and discussion  
* a way to describe typical evolution patterns  
* a shared vocabulary for maintainers, contributors, users, and CNCF groups

This model *is not*:

* a certification or conformance program  
* a checklist that must be “passed”  
* a ranking or comparison of CNCF projects

This model complements, but does not replace, other community efforts in the OpenTelemetry ecosystem. For example, the [Instrumentation Score](https://github.com/instrumentation-score/spec) specification provides rule-based checks that assess the quality and completeness of emitted telemetry, while ecosystem registries such as the [OpenTelemetry Ecosystem Explorer](https://github.com/open-telemetry/opentelemetry-ecosystem-explorer) catalog available components and integrations.

The maturity model focuses instead on design intent, consistency, and evolution over time \- how OpenTelemetry support is shaped, governed, and maintained \- rather than producing a numeric score or acting as a discovery index.

Projects may be at different maturity levels across different dimensions, and that is expected. Higher maturity is not always necessary or desirable for every project; the value lies in understanding where you are and what matters next.

# How to Use This Model

The model is structured around *dimensions* of OpenTelemetry support. Each dimension is described across *four maturity levels*.

This model evaluates the maturity of a project’s OpenTelemetry support based on what the project emits and supports natively.

Maturity assessments should be based on the structure, semantics, and configuration surface of the telemetry produced by the project itself (for example, its OTLP payloads, supported configuration options, and documented behavior).

Downstream processing \- such as enrichment, transformation, or correlation performed by an OpenTelemetry Collector \- may be noted as a mitigation, but should not be used to award a higher maturity level unless the project explicitly documents that pipeline as part of its supported integration contract.

This distinction helps separate project maturity from pipeline capability and avoids conflating what is possible with what is intentionally supported.

Rule-based quality assessments, such as those defined by the [Instrumentation Score](https://github.com/instrumentation-score/spec) specification, can be useful for validating the quality of emitted telemetry, but they do not determine maturity levels in this model.

You can use the model to:

* assess a project dimension by dimension  
* identify gaps or inconsistencies  
* guide roadmap or design discussions  
* align expectations between maintainers and users

There is intentionally no single overall maturity score. Each dimension stands on its own.

Readers can stop at the overview table for a high-level understanding, or dive deeper into individual dimensions for detailed explanations and examples.

# Global Maturity Levels

The following maturity levels apply consistently across all dimensions.

## Level 0: Instrumented

Telemetry exists, primarily to support internal debugging and development needs. Instrumentation is often incremental and opportunistic. OpenTelemetry is not yet a primary design concern, and observability is treated largely as an implementation detail.

## Level 1: OpenTelemetry-Aligned

OpenTelemetry is explicitly supported, often alongside legacy approaches. OpenTelemetry SDKs, protocols, or exporters are adopted, but legacy assumptions and constraints still influence design. Telemetry generally works for common scenarios.

## Level 2: OpenTelemetry-Native

OpenTelemetry is the primary integration surface for users. Telemetry is designed intentionally, with interoperability, correlation, and user experience in mind. OpenTelemetry concepts shape architecture and configuration choices.

## Level 3: OpenTelemetry-Optimized

OpenTelemetry support is continuously refined based on real-world usage, scale, and feedback. Telemetry is treated as a long-lived product surface, with deliberate evolution, governance, and attention to cost, quality, and usability.

# OpenTelemetry Support Maturity Matrix

The following table provides a high-level overview of the maturity model, showing how OpenTelemetry support typically evolves across each dimension and maturity level. Detailed explanations of each dimension are provided in the sections that follow..

| Dimension | Level 0:  Instrumented | Level 1:OTel-Aligned | Level 2:  OTel-Native | Level 3:  OTel-Optimized |
| :---: | :---: | :---: | :---: | :---: |
| **Integration Surface** | Tool-specific exporters | OTLP alongside legacy | OTel is primary interface | Intentionally evolved integration |
| **Semantic Conventions** | Proprietary / inconsistent | Partial OTel alignment | Consistent OTel semantics | Intentional semantic extensions |
| **Resource Attributes & Configuration** | Hard-coded identity | Inconsistent config | Stable identity, OTEL\_\* respected | Predictable, documented behavior |
| **Trace Modeling & Context Propagation** | Fragmented traces | Common paths coherent | Intentional trace modeling | Complex async workflows supported |
| **Multi-Signal Observability** | Single-signal focus | Loosely connected signals | Correlated traces, metrics, logs | Cross-signal workflows optimized |
| **Audience & Signal Quality** | Maintainer-focused, noisy | Reduced noise | User-oriented defaults | Signal quality actively optimized |
| **Stability & Change Management** | Undocumented changes | Informal communication | Telemetry treated as contract | Planned, reviewed evolution |

## Dimensions Covered

The maturity model evaluates OpenTelemetry support across the following dimensions:

1. **Integration Surface**  
   How users connect a project to their observability pipelines and how strongly telemetry is coupled to specific tools or vendors.  
2. **Semantic Conventions (including extensions)**  
   How consistently telemetry meaning aligns with OpenTelemetry semantic conventions, and how domain-specific meaning is introduced when needed.  
3. **Resource Attributes & Configuration**  
   How identity, scope, and configuration are handled across environments, including correct use of resource attributes and standard OpenTelemetry configuration mechanisms.  
4. **Trace Modeling & Context Propagation**  
   How traces are structured and how context flows through synchronous and asynchronous execution paths.  
5. **Multi-Signal Observability**  
   How traces, metrics, and logs are supported together and correlated to form a coherent observability experience.  
6. **Audience & Signal Quality**  
   Who telemetry is designed for, how noisy it is by default, and how well it communicates meaningful system behavior.  
7. **Stability & Change Management**  
   How telemetry evolves over time and how changes are communicated and managed once users depend on it.

## Visualizing maturity across dimensions

Because OpenTelemetry support often matures unevenly across different dimensions, visual representations can help make trade-offs and patterns easier to understand. One optional way to visualize an assessment is to plot the maturity level of each dimension on a radar chart, with each axis representing a specific aspect of OpenTelemetry support.

Used this way, the chart does not produce a single score or ranking. Instead, it highlights where maturity is concentrated and where gaps or inconsistencies may exist. A project may, for example, show strong trace modeling and context propagation while still relying on legacy approaches for metrics, or offer a clean integration surface while lagging in stability and change management.

### Example: layered maturity profiles

To illustrate how the model can be applied, the following example shows *three fictive projects* plotted on the same radar chart. Each project is evaluated across the same set of dimensions and maturity levels (0–3). The intent is not to compare real systems, but to demonstrate how different maturity profiles can emerge depending on architectural role and design priorities.

**![][image1]**

*Figure: Example radar chart showing layered OpenTelemetry maturity profiles for three fictive projects (for illustration purposes only). Each axis represents a maturity dimension, with values ranging from Level 0 (Instrumented) to Level 3 (OpenTelemetry-Optimized).*

In this fictive example, the differing shapes make trade-offs immediately visible. One project may emphasize trace modeling and context propagation, another may lead in metrics and stability, while a third balances maturity more evenly across dimensions. The chart makes these differences visible without collapsing them into a single score.

### Comparing maturity across categories

When assessments for multiple projects are visualized using the same dimensions, profiles can be layered to explore how different categories of software tend to evolve. Ingress controllers, service meshes, databases, and application frameworks often exhibit distinct maturity patterns, shaped by their role in the architecture and the signals they prioritize first.

These comparisons are intended to be illustrative rather than evaluative. They are most meaningful when used within similar categories and should be read as a way to understand expectations and trade-offs, not as a mechanism for ranking or judgment.

> **Note:** Detailed level-by-level criteria for each dimension are defined in the individual dimension skill files (e.g. `dimension-1-integration-surface/SKILL.md`), not in this document. This spec covers model philosophy, global levels, and the summary matrix only.

