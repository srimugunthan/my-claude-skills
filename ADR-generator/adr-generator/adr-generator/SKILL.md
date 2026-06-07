---
name: adr-generator
description: >
  Generate well-structured Architecture Decision Records (ADRs) for software and ML system design decisions.
  Use this skill whenever the user wants to write, draft, create, or document an architectural decision,
  design choice, or technical tradeoff. Trigger on phrases like "write an ADR", "create an ADR",
  "document this decision", "architecture decision record", "document why we chose X", "write up our
  decision to use X", "I need to justify this architecture choice", "help me write up this tech decision",
  or when the user describes a system design decision and wants it documented. Also trigger when the user
  asks to capture a decision made during a design review, RFC process, or technical discussion.
  Output is a markdown file saved to the working directory unless the user requests docx.
---

# ADR Generator

Produces complete, well-reasoned Architecture Decision Records following the Nygard/MADR convention,
adapted for ML systems, agentic pipelines, fintech services, and general software architecture.

---

## Elicitation (always do this first)

Before writing, gather the following. Check the conversation — the user may have already provided some of these:

**Required:**
1. **Decision title** — What was decided? (e.g., "Use Redis for session state in fraud API")
2. **Context** — What problem or constraint forced this decision?
3. **Decision** — What was chosen?
4. **Alternatives considered** — At least 2 options that were on the table

**Optional but strongly preferred:**
5. **Consequences** — Both positive and negative outcomes
6. **Status** — Proposed / Accepted / Deprecated / Superseded (default: Accepted)
7. **Deciders** — Names/roles who made the call
8. **Date** — Default to today if not provided
9. **ADR number** — e.g., ADR-007; ask if not provided, or default to ADR-NNN
10. **Follow-up actions** — Deferred work, open risks, or tracked items
11. **Compliance/risk notes** — Especially relevant for fintech, ML model risk, or data privacy contexts

If the user gives a rough description of the decision, extract what you can and fill in reasonable defaults for the rest. Only ask for clarification on items you cannot infer.

---

## Output Format

Produce a markdown file named `ADR-NNN-<kebab-case-title>.md` saved to `/mnt/user-data/outputs/`.

Use this template:

```
# ADR-NNN: <Title>

**Status:** <Proposed | Accepted | Deprecated | Superseded by ADR-XXX>
**Date:** <YYYY-MM-DD>
**Deciders:** <Names / roles>

---

## Context

<2–4 sentences describing the problem, constraint, or forcing function that made this decision necessary.
Include system scale, latency requirements, regulatory constraints, team constraints, or other relevant factors.>

---

## Decision

<1–3 sentences stating clearly what was decided. Be unambiguous.>

---

## Alternatives Considered

| Option | Pros | Cons |
|--------|------|------|
| <Option A> | <...> | <...> |
| <Option B (chosen)> | <...> | <...> |
| <Option C> | <...> | <...> |

<Optional: 1–2 sentences per alternative if the table needs elaboration.>

---

## Consequences

**Positive:**
- <Outcome 1>
- <Outcome 2>

**Negative / Accepted Tradeoffs:**
- <Tradeoff 1>
- <Tradeoff 2>

---

## Compliance / Risk Notes  *(omit section if not applicable)*

- <Regulatory, data privacy, model risk, or security considerations>

---

## Follow-up Actions  *(omit section if none)*

- [ ] <Action 1> (<owner if known>, <target date if known>)
- [ ] <Action 2>
```

---

## Quality Guidelines

### Context section
- Explain *why* a decision was needed, not just what the system does
- Include quantitative constraints where available (latency SLA, throughput, data volume, team size)
- For ML systems: mention model lifecycle, inference serving, data pipeline, or regulatory context as relevant
- For fintech: mention fraud/AML/compliance constraints if applicable

### Decision section
- State the decision as a complete sentence: "We will use X" not just "X"
- Avoid hedging — the decision is already made; write it that way

### Alternatives table
- Always include the chosen option in the table (mark it clearly)
- Be honest about the chosen option's cons — ADRs are most valuable when they record why you *didn't* pick alternatives, not just that you picked one
- Minimum 2 alternatives; 3 is ideal

### Consequences
- Separate positive from negative/tradeoffs
- Negative consequences are not failures — they are accepted tradeoffs. Write them that way.
- For ML decisions: consider model performance, retraining cost, explainability, drift monitoring as consequences
- For infrastructure decisions: consider operational burden, failure modes, on-call impact

### Follow-up actions
- Convert accepted risks into tracked work items
- Use checkbox format `- [ ]` so the list is actionable

---

## Domain-Specific Guidance

### ML / AI Systems
Include in context: model serving method, inference latency budget, batch vs real-time, training pipeline dependencies.
Common decision areas: model registry choice, feature store, serving framework (Triton vs TorchServe vs vLLM), retraining triggers, shadow deployment strategy, drift detection.

### Financial Services / Fraud / AML
Include compliance notes for: PII handling, audit log retention, model explainability requirements (SHAP/LIME), regulatory review triggers, false positive rate targets.

### Agentic / LLM Systems
Common decision areas: orchestration framework (LangGraph vs custom), memory architecture, tool use strategy, guardrails placement, prompt versioning, evaluation harness.

### Infrastructure / DevOps
Consider: failure modes, SLA impact, operational runbook burden, rollback strategy.

---

## After Writing

1. Save the file as `ADR-NNN-<kebab-title>.md` in `/mnt/user-data/outputs/`
2. Present it to the user with `present_files`
3. Offer: "Want me to adjust the tone, add more detail to any section, or convert this to a Word document?"
