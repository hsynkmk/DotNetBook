# Chapter 12 — Observability — Q & A

---

### Q1. What is observability and the three pillars?

The ability to understand a system's internal state from external outputs (answer what/why in production without a debugger). Three pillars: **logs** (discrete detailed events — what happened), **metrics** (cheap aggregates over time — performance/trends), **traces** (one request's path across services — where time went/failed).

---

### Q2. Monitoring vs observability?

**Monitoring** watches **known** signals for **known** problems (predefined dashboards/alerts). **Observability** lets you ask **arbitrary, new** questions about behavior, including unanticipated ones. Monitoring says *that* something's wrong; observability lets you find *why*.

---

### Q3. How do the three pillars work together?

Metrics **detect** (an alert: p99 spiked), traces **localize** (which span/service is slow), logs **explain** (the specific error with context). Correlate them via a shared **TraceId** so you can pivot between pillars. No single pillar suffices.

---

### Q4. Why structured logging over string interpolation?

Templates with named placeholders (`"...{OrderId}...", id`) capture **structured, queryable fields** (search "OrderId = 42" across services) and skip formatting when filtered out. Interpolation (`$"...{id}..."`) flattens to a string (no fields) and formats regardless. Always use named placeholders + args.

---

### Q5. What do log scopes do for observability?

`BeginScope` attaches structured properties to every log in an operation (flowing across awaits). ASP.NET Core's per-request scope adds the **TraceId/SpanId**, correlating logs with the current trace — so you can pivot from a slow trace to its logs.

---

### Q6. Why `[LoggerMessage]` source generation?

It emits allocation-free, pre-compiled logging methods (no boxing of value-type args, no per-call template parsing, level checked first) — ideal for hot log paths and AOT. The generic `Log*` methods allocate/parse per call.

---

### Q7. Where should logs go in containers?

To **stdout** (console), ideally as structured JSON (`AddJsonConsole`), which the platform collects and the aggregator parses into fields. Don't log to files (ephemeral, uncollected). Prefer exporting via OpenTelemetry for vendor neutrality.

---

### Q8. What does Serilog add over built-in providers?

Rich structured logging (`{@obj}` destructuring), **enrichment** (auto-attach machine/service/thread/**trace id**/per-request context to every event), and a huge **sink** ecosystem. It plugs into `ILogger` and can export via the OpenTelemetry sink.

---

### Q9. Counter vs Histogram, and why not average latency?

**Counter** = monotonic sum (totals: requests, errors). **Histogram** = distribution (latencies, sizes). Use a histogram for latency and look at **percentiles** (p95/p99) — an *average* hides the slow tail (a few very-slow requests), which is what users actually experience.

---

### Q10. What is metric cardinality and why is it the #1 metrics pitfall?

Cardinality = the number of distinct dimension-value combinations; each creates a separate time series. High-cardinality dimensions (user/request/order ids — millions of values) explode into millions of series, overwhelming/bankrupting the backend. Use only **low-cardinality** dimensions (region, status, route template); put ids in logs/traces.

---

### Q11. What are RED and USE?

Frameworks for *what* to measure. **RED** (request services): **R**ate, **E**rrors, **D**uration. **USE** (resources): **U**tilization, **S**aturation, **E**rrors (for queues, pools, CPU/memory). Measure RED for services (SLIs/alerts) and USE for resources (catch saturation early).

---

### Q12. What is a trace and a span?

A **trace** follows one request across services; a **span** (`Activity`) is one step in it, with timing and parent/child links. Spans nest into a trace tree (shared **TraceId**) showing where time went and where it failed.

---

### Q13. How does trace context propagate across services?

Via the **W3C Trace Context** standard — the `traceparent` HTTP header carrying TraceId + parent SpanId. It's **automatic over HTTP** (HttpClient injects it, ASP.NET Core extracts it), so a request's trace flows across services. Over **messaging** you must propagate it in message headers manually.

---

### Q14. Can span tags be high-cardinality? What about metric dimensions?

Span **tags** can be high-cardinality (specific ids are useful for per-request detail). Metric **dimensions** must be low-cardinality (high-cardinality explodes series). Put ids in span tags/logs, not metric dimensions.

---

### Q15. Why is instrumentation "near-free when uncollected"?

`StartActivity` returns **null** when no tracing listener is subscribed (and metrics no-op), so `activity?.` calls do nothing — meaning you can instrument liberally without overhead in environments where collection is off.

---

### Q16. What is OpenTelemetry and why does it matter for .NET?

The vendor-neutral standard unifying logs/metrics/traces. The crucial .NET fact: the BCL primitives (`ILogger`/`Meter`/`ActivitySource`) **are** the OTel APIs — instrument once with standard types, and the OTel SDK exports to any backend (no vendor lock-in).

---

### Q17. What does OTel auto-instrumentation give you for free?

`AddAspNetCoreInstrumentation`/`AddHttpClientInstrumentation`/`AddEntityFrameworkCoreInstrumentation`/`AddRuntimeInstrumentation` give RED metrics per route, runtime metrics (GC/thread pool), and distributed traces (request → HTTP → DB) with automatic W3C context propagation — almost no custom code. You add business telemetry on top.

---

### Q18. Why export OTLP to a Collector instead of directly to a backend?

The **OpenTelemetry Collector** receives OTLP and routes to backends, decoupling your app from backend choices — switch from Jaeger to Tempo or add Datadog by reconfiguring the collector, no app code changes. Central config, swappable backends, no lock-in.

---

### Q19. Why register custom sources/meters with OTel?

Auto-instrumentation covers only framework components (ASP.NET Core/HttpClient/EF/runtime). Your custom `ActivitySource`/`Meter` need `AddSource`/`AddMeter`, or your business spans/metrics aren't exported.

---

### Q20. What built-in telemetry does ASP.NET Core provide?

Request metrics (rate/duration/active, by route template/status — RED for free), Kestrel connection metrics, HttpClient outbound metrics, runtime metrics (GC, thread-pool queue, exceptions), and EF Core query metrics/spans — plus automatic request/outbound/DB trace spans. You add domain (business) telemetry.

---

### Q21. What is sampling and why use it?

Recording a representative fraction of traces (e.g., 10%) to control cost at high volume. **Tail sampling** (decide after the trace completes — keep all errors/slow traces, sample the rest) is ideal but needs a collector; head sampling is simpler. Tune to volume/budget.

---

### Q22. Which diagnostic tool do you reach for first, and then?

**`dotnet-counters`** first — triage whether it's CPU, GC pressure, thread-pool starvation, or exceptions (low overhead, live). Then the matching deep tool: **`dotnet-trace`** (CPU hot stacks), **`dotnet-gcdump`** (memory leaks via retention), **`dotnet-dump`/`dotnet-stack`** (crashes/hangs).

---

### Q23. How do you hunt a production memory leak?

`dotnet-gcdump` snapshots over time → compare to find what's **growing**; in a dump, `dumpheap -stat` (objects by type) + `gcroot <addr>` (what keeps an object alive — the unintended root). The classic production leak investigation.

---

### Q24. Profiling vs observability?

Observability (logs/metrics/traces) is always-on, operation-level (what/where/which service). Profiling/diagnostics are on-demand (or continuous-profiler), code-level (why — CPU/memory/GC/threads). Telemetry detects/localizes; profiling root-causes at the code level. Use both; measure, don't guess.

---

### Q25. Liveness vs readiness health checks (observability context)?

**Liveness** (alive? → restart) checks **only the app** — checking dependencies causes restart storms on a dependency blip. **Readiness** (can serve? → stop routing) checks **dependencies**. They're a coarse instance-level signal for orchestration, distinct from the detailed pillars; wire readiness to graceful shutdown for zero-downtime deploys.

---

### Q26. Is a passing health check enough to know the app is healthy?

No — it means "up/ready," not "performing well." Health checks are a binary orchestration signal; use **metrics/traces/logs** to understand behavior and diagnose performance. They complement, not replace, the pillars.

---

→ Next: [Coding.md](Coding.md)
