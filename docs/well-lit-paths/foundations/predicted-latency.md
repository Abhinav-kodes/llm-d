# Predicted Latency-Based Scheduling

llm-d's [optimized baseline guide](./optimized-baseline.md) leverages load signals and prefix-cache affinity to schedule requests, combining the signals together with heuristics.

This path is for operators who want to adopt predicted latency-based scheduling - which uses an XGBoost model trained online - to make scheduling decisions. This strategy is useful when:

- Your workload has **high variance in prompt and completion length**, and queue depth alone is a poor proxy for true load.
- Your clients can express **per-request latency SLOs** (interactive vs. batch) and you want the gateway to enforce them.
- Static weight tuning between cache affinity and load has become **fragile** as traffic shifts.

> [!NOTE]
> Predicted latency is not a fit when the pool is **heterogeneous** — mixed GPU types, model variants (e.g. prefill vs decode), or serving configurations in the same pool will produce inaccurate predictions, because the predictor assumes a single pod shape.

## Deploy

See the [Predicted Latency guide](../../../guides/predicted-latency-routing) for manifests and step-by-step deployment.

## Architecture

<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)">
    <img src="../../assets/latency-predictor.svg" alt="Latency Predictor">
  </picture>
</p>

The setup deploys an EPP with the predicted latency sidecar containers:

- **Training Server** - trains the XGBoost model to predict TPOT and TTFT based on observed traffic
- **Prediction Servers** - predict TPOT and TTFT of the request based on current server state

During the standard request flow:

- Request arrives at the proxy, which forwards the request to the EPP
- EPP queries the prediction server
- EPP (using `latency-scorer`) selects optimal endpoint based on the prediction
- Proxy forwards request to the vLLM endpoint
- vLLM endpoint processes the request, returns response to proxy
- Proxy sends results to the training server, which uses samples to update the model

## Observability

Predicted latency routing makes scheduling decisions from an online-trained XGBoost model, so the signals that matter are the **gap between predicted and observed latency** and **SLO violations** rather than a single aggregate latency. The [Predicted Latency guide's Observability section](../../../guides/predicted-latency-routing/README.md#4-observability--troubleshooting) covers the key metrics and failure modes, backed by the shared [PromQL](../../operations/observability/promql.md#routing--load-balancing) and [metric](../../operations/observability/metrics.md#predicted-latency--slo) references.

## Further Reading

- [Latency Predictor Architecture](../../architecture/advanced/latency-predictor.md) — plugin pipeline, ML model, scaling characteristics, metric reference.
- [llm-d/llm-d-latency-predictor](https://github.com/llm-d/llm-d-latency-predictor) — source for the training and prediction server Python code.
- [Predicted Latency-Based Scheduling for LLMs - Blog](https://llm-d.ai/blog/predicted-latency-based-scheduling-for-llms) — design rationale and benchmark results.
