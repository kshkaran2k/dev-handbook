# Observability

## Outline
1. Introduction
2. What actually changes when adopt OTel
3. The data model
4. Instrumentation strategy
5. My experince with OTel

---

## 1. Introduction

### What is observability and why we need it?

As most of the systems are running in microservices or huge monolithic systems, it is very crucial to identify the unexpected failures or troubleshooting the errors. This makes the troubleshooting slower and affects the service reliability and customer experience.


### Three pillars of observability

- `Logs` captures the informations from the systems transactions and events in the running systems. This includes the errors, warnings and other activities. Analysing logs helps to identify why there is a deviation in the system

- `Metrics` are the KPI based measurement parameters. These are used to create dashboards and visualise performance indicators like CPU utilization, memory utilization, network throughput, etc.

- `Tracing` records and tracks the flow of the request moves end-to-end through various components of the systems


### Monitoring v/s Observability

| Aspect | Monitoring | Observability |
| --- | --- | --- |
| Goals | Identify and alert on known issues and metrics | Understand and explore unknown behaviors |
| Scope | Focuses on upptime and system metrics | Provide a holistic view of the system using it's pillars |
| Data source | Metrics of the system | All the outputs |
| Complexities | Easier to implement using standard metrics | More complex as requires deep integration |
| Examples | Nagios, Prometheus | Grafana, OTel, Jaeger|


---

## 2. What actually changes when adopt OTel

The team has used Zipkin/Jaeger clients, Prometheus client libraries, or a vendor APM agent (Datadog, New Relic, AppDynamics) directly, here's the mental shift:

| Old model | OTel model |
|---|---|
| Instrumentation library *is* tied to a backend (Jaeger client, Prometheus client) | Instrumentation (the API) is backend-agnostic; the **exporter** decides the destination |
| Switching backends means re-instrumenting | Switching backends means changing Collector/exporter config — application code is untouched |
| Vendor agent format is proprietary | **OTLP** is a standard wire protocol (gRPC on :4317 / HTTP on :4318) that most vendors now accept natively |
| Tracing, metrics, and logging are three separate toolchains with three separate (or no) correlation IDs | One SDK, one Resource, and a shared TraceID/SpanID across all three signals |

The payoff worth explaining to an architecture review board isn't "better telemetry" — it's that OTel **decouples your instrumentation investment from your vendor contract**. Can renegotiate or replace observability backend without a re-instrumentation project.

**Where OTel isn't a slam dunk yet — worth saying plainly in any design doc:**
- **Semantic convention stability is mixed.** HTTP conventions are stable; some domain conventions (GenAI, CI/CD) are still evolving and can break between versions. Pin SDK/convention versions per service and track migration guides rather than assuming forward compatibility across major bumps.
- **Collector component stability is mixed**, by the project's own admission — core is stable, but `contrib` components range from alpha to stable. Vet each component individually before it goes into a production pipeline.
- **Auto-instrumentation agents (Java especially) add real startup/CPU overhead** at high span-creation rates. Benchmark before assuming "instrument everything" is free.

---


## 3. The data model

### Trace anatomy


- **A span** is a name, a start/end timestamp, attributes (using semantic-convention keys), events (timestamped points inside the span — e.g. "retry #2", "cache miss" — cheaper than creating a child span for sub-events), a status (unset/OK/error), and **links**. Links exist for fan-out and async cases where causality isn't a clean parent-child edge — for example, a Kafka consumer span linking back to its producer span, or a batch job span linking to the N requests that fed into it.
- **Resource attributes** (`service.name`, `deployment.environment`, `k8s.pod.name`, `cloud.region`) are stamped once per process and apply to everything that process emits. This becomes your primary filter/group-by dimension in whatever backend you use, so it's worth getting right at rollout — retrofitting resource attributes across an already-running fleet is a slog.
- **Context propagation**, in practice, means the **W3C `traceparent` header** gets injected on every outbound call and extracted on every inbound one. The failure mode people actually hit: **any hop that doesn't propagate the header — a queue, a batch job, a scheduled task, a third-party SDK that strips headers — breaks the trace into disconnected fragments.** Auditing header propagation across message brokers (Kafka, SQS, RabbitMQ) is usually the hard part of a rollout, not the SDK instrumentation itself.

### Metrics — instrument choice is an architecture decision

| Instrument | Semantics | What goes wrong if you pick the wrong one |
|---|---|---|
| Counter | Monotonic, only increases | Using it for a value that can decrease silently corrupts any `rate()` query downstream |
| UpDownCounter | Can go up or down | The right choice for gauge-like values that need delta semantics, like active connections |
| Gauge | Point-in-time snapshot | Don't use it for anything you need rate-of-change on over an interval |
| Histogram | Distribution, used to derive p50/p95/p99 | Bucket boundaries are fixed at instrument-creation time and are hard to change retroactively. Get default boundaries reviewed against your actual latency distribution before rollout — not after six months of data has accumulated with the wrong buckets |

**The production risk with metrics is cardinality, not throughput.** Every unique combination of attribute values on a metric creates a new time series. Attaching a high-cardinality attribute (`user_id`, `request_id`, a full URL with query params) to a metric — instead of to a span, where cardinality isn't an issue — is the single most common cause of an observability backend falling over or a bill spiking 10x. This is the same mistake most people have made once already with Prometheus labels; OTel doesn't fix it, it just re-packages it.

### Logs

OTel wraps existing logging libraries (Log4j/SLF4J, Python's `logging`) rather than replacing them, and injects `trace_id`/`span_id` into each log record. The migration work here is usually just mapping your **existing structured-logging field names** onto OTel's conventions — not a rewrite.

---


## 4. Instrumentation strategy

- **Default recommendation for an existing fleet: turn on auto-instrumentation everywhere first.** It gives baseline HTTP/DB/RPC span coverage with zero application code changes, and it's what actually gets an organization from "no traces" to "we have traces" fastest.
- **Reserve manual instrumentation for business-meaningful spans** the automatic layer can't see — a specific pricing calculation, a fraud-check decision path — rather than sprinkling it everywhere. Over-instrumenting manually is a maintenance tax: spans become one more thing that rots when nobody updates them during a refactor, the same failure mode as stale log statements or stale metrics.
- **Auto-instrumentation is zero-code but not zero-cost.** Java's `-javaagent` and Python's `opentelemetry-instrument` launcher both add overhead. Budget for a performance-regression test in the rollout plan, especially for latency-sensitive hot paths, before flipping it on fleet-wide.

---



## 5. My experience with OTel

We were using one of the paid APM agent.
Switching from that to OTel was one of the big task. But after getting to know
about it and switching to OTel, got to know more about it and understand it much
better.

Able to understand the trace, span and other things and whats their use.
Additionally the integration system developed by some other team, but way too
generic and that to for a couple of languages(Python and NodeJS). 

With custom integration, we achieve each log/tracing/metrics at the function
level. This helps to identify the issue and resolve it quickly. Before OTel we
had custom middleware which adds a request is to each API call and binds it to
each logs using context. The challenge were 
- it is difficult to identify the function which is taking more time and is a
  culprit
- difficult to add request id to various other processess like cron tasks, kafka
  message processing
- If the service is calling another service via API, we are not able to trace it in the another
  service
as the request id was configuring at the API request side and there were same
functions used by the cron tasks and in the kafka message processings



