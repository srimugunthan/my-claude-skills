# ADR-001: Use Triton Inference Server for LightGBM Model Serving in Click Propensity Pipeline

**Status:** Accepted
**Date:** 2026-06-07
**Deciders:** Srimugunthan (AI/ML Tech Lead), Backend Platform Team

---

## Context

The click propensity modeling system (Avazu dataset, LightGBM classifier) requires real-time inference to score ad impression events as they arrive via Kafka. The system targets sub-50ms p99 inference latency at 10k requests/second peak throughput. Two concerns drove this decision: (1) the existing FastAPI + pickle-loaded model approach could not sustain concurrent load beyond ~500 RPS without CPU saturation, and (2) the team needed a serving layer that could eventually support multiple model types (LightGBM, XGBoost, future neural rankers) without rewriting the serving infrastructure for each.

---

## Decision

We will use **NVIDIA Triton Inference Server** with the FIL (Forest Inference Library) backend to serve the LightGBM click propensity model. Triton will be deployed as a Docker container within the existing Docker Compose stack, exposing a gRPC endpoint consumed by the FastAPI feature enrichment layer.

---

## Alternatives Considered

| Option | Pros | Cons |
|--------|------|------|
| FastAPI + LightGBM native predict (current) | Simple, no new infra, fast dev loop | Saturates at ~500 RPS; no batching; model reload requires redeploy |
| BentoML | Good Python-native DX, built-in model registry | Adds registry dependency; less mature FIL support; additional ops surface |
| **Triton + FIL (chosen)** | GPU/CPU batching, dynamic batching, multi-model support, gRPC performance | Steeper learning curve; heavier container; FIL requires model export step |
| TorchServe | Strong PyTorch ecosystem | Overkill for gradient boosted trees; no first-class GBDT support |
| ONNX Runtime via FastAPI | Portable, lightweight | LightGBM → ONNX conversion loses some fidelity on categorical features; manual batching logic |

---

## Consequences

**Positive:**
- Dynamic batching in Triton reduces per-request overhead; p99 latency drops from ~85ms to ~18ms in load tests at 2k RPS
- Multi-model support means future neural rankers can be onboarded without replacing the serving layer
- gRPC reduces serialization overhead vs REST JSON on the hot path
- FIL backend natively supports LightGBM `.model` files with no format conversion required
- Triton's Prometheus metrics endpoint integrates directly into existing monitoring stack

**Negative / Accepted Tradeoffs:**
- FIL requires a model export step (`treelite` compilation) before deployment; adds ~5 min to the model promotion pipeline
- Triton container is ~2GB; increases Docker Compose startup time and resource footprint on dev machines
- Team has limited prior Triton ops experience; initial oncall runbook needs to be authored
- gRPC client code must be maintained in the FastAPI feature layer; adds a thin dependency not present in the current stack

---

## Compliance / Risk Notes

- No PII flows through the Triton serving layer; input features are anonymized impression IDs and engineered signals
- Model version tracking is handled externally (MLflow); Triton model repository versioning must stay in sync — a mismatch is a silent risk requiring a pre-deploy checklist item
- If the system is later extended to financial services use cases, SHAP explainability must be evaluated separately (FIL does not expose feature attributions natively)

---

## Follow-up Actions

- [ ] Author Triton oncall runbook covering model reload, FIL recompile, and gRPC timeout scenarios (Tech Lead, Q3 2026)
- [ ] Add pre-deploy checklist: MLflow version tag == Triton model repository version (Backend, before next release)
- [ ] Benchmark dynamic batching window size (default 100µs) against actual Kafka consumer throughput patterns (ML Eng, Q3 2026)
- [ ] Evaluate Triton model ensemble feature for chaining feature enrichment + scoring in a single gRPC call (deferred, Q4 2026)
