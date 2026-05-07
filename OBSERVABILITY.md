# Logging and Observability Subsystem Reproduction Prompt

You are implementing a production-grade logging and observability subsystem for a backend application with one HTTP-serving process and, if the project has asynchronous work, one background worker process. Recreate the subsystem described below in the target repository, adapting domain names where necessary but preserving the same operational properties, topology, and verification standard.

## Objective

Deliver a small, coherent observability stack that makes the application operable without introducing unnecessary platform complexity. The result must give engineers all of the following:

- structured JSON logs from every runtime component;
- request and job correlation across logs;
- built-in health, readiness, and Prometheus-style metrics endpoints;
- metrics from both the HTTP process and the worker process;
- local collection and exploration of metrics and logs through a Docker Compose stack;
- pre-provisioned Grafana datasources and dashboards;
- explicit statement that distributed tracing is deferred unless the target project already has a trace standard.

Do not treat observability as an afterthought. It is part of the application contract. A fresh checkout of the repository must be able to start the app and the monitoring stack locally and immediately answer the questions "is it healthy?", "is it ready?", "what failed?", "how much work is it doing?", and "where do I inspect that?".

## Reference Architecture to Recreate

Model the subsystem after this shape:

1. The main application process emits structured JSON logs to stdout.
2. The main application process exposes:
   - `GET /health` for shallow liveness;
   - `GET /ready` for dependency and model readiness;
   - `GET /metrics` for Prometheus text exposition.
3. If the system has a background worker, the worker emits the same JSON log format and exposes its own `GET /metrics` endpoint on a dedicated port, even if it is not an HTTP app.
4. Metrics are scraped by VictoriaMetrics.
5. Container logs are collected from Docker by Vector, normalized, and forwarded to VictoriaLogs.
6. Grafana is provisioned with one metrics datasource, one logs datasource, and dashboards tailored to the product workflow.
7. Traces are not required in this version. Do not add OpenTelemetry wiring just because the libraries exist. Only add tracing if the target project already depends on it and you can preserve the same minimal-operability standard.

## Logging Contract

Implement a single root logging configuration shared by the API process and the worker process.

Requirements:

- Use the Python standard `logging` module unless the target project already has a stronger standard.
- Replace default text logs with one-line JSON objects written to stdout.
- Every record must include at least:
  - `ts`: UTC ISO-8601 timestamp;
  - `level`: lowercase level name;
  - `logger`: logger name;
  - `msg`: rendered message text.
- Preserve arbitrary extra fields passed by application code. Do not discard structured context.
- Avoid multi-line stack traces unless emitted by the logger as structured exception output.

Design intent:

- infrastructure should treat logs as machine-readable events;
- local development should still be usable via `docker compose logs`;
- downstream collectors should be able to parse without brittle regexes.

## Correlation and Access Logging

Add middleware, hooks, or framework-equivalent instrumentation so every HTTP request gets a request identifier.

Requirements:

- Accept an inbound request ID header if present, otherwise generate one.
- Store the request ID in request-local state so handlers can reuse it.
- Return the request ID to the client in a response header.
- Return end-to-end request latency in a response header.
- Emit a structured access log for every HTTP request.

Minimum fields for access logs:

- `request_id`
- `http_method`
- `path`
- `status_code`
- `latency_ms`

If the project has asynchronous jobs, job logs must also include:

- `job_id`
- queue or executor name;
- final job status;
- backend or implementation path when relevant;
- run time or queue wait time when available.

If the project has streaming or session-style workflows, add a single structured session-summary log at the end of each session with identifiers, durations, sizes, and outcome status.

## Metrics Contract

Expose Prometheus-compatible metrics from application memory. Do not block delivery on introducing a full metrics SDK if a minimal in-process registry is enough.

The reference design used a small custom registry that renders Prometheus text directly. You may use an existing metrics library if it keeps the same visible behavior and does not complicate deployment.

The subsystem must expose these metric categories:

1. HTTP traffic
   - request count by method, route template, and status code;
   - request latency with `_sum` and `_count` series or an equivalent histogram.

2. Background job lifecycle
   - total job events such as created, started, completed, failed, canceled, cancel requested;
   - queue wait time;
   - run time;
   - active in-progress jobs in the worker.

3. Domain workload outcomes
   - successful and failed domain operations;
   - input size or duration processed;
   - processing time;
   - labels for backend mode, output mode, optional feature flags, and final status.

4. Streaming or session workflows, if the product has them
   - session lifecycle events;
   - chunks or message counts;
   - byte volume;
   - active sessions;
   - session duration and processed media duration.

Metric naming guidance:

- keep a stable prefix for the product domain;
- use labels for dimensions, not metric-name suffix explosions;
- prefer a small number of high-signal metrics over many low-signal counters;
- if reproducing this design in another ASR or media project, preserve the same categories even if exact names change.

## Health and Readiness

Implement both shallow health and real readiness.

Requirements:

- `/health` returns success if the process is alive.
- `/ready` returns success only when the service can actually do useful work.
- readiness must reflect expensive startup dependencies such as models, queues, or downstream executors.
- readiness failure responses must include a machine-readable reason string where practical.

Do not collapse liveness and readiness into a single endpoint.

## Worker Observability

If the project has a background worker, do not hide it behind the main API metrics.

Requirements:

- the worker must configure the same JSON logging format as the API;
- the worker must expose its own metrics endpoint on a dedicated host and port;
- worker job lifecycle metrics must be emitted by the worker process itself;
- worker metrics must distinguish queue wait time from execution time;
- failures in internal executor calls must produce both logs and failure metrics.

If the worker is not an HTTP framework process, create a tiny standalone HTTP server that serves `/metrics`. Keep it single-purpose and dependency-light.

## Local Observability Stack

Create a Docker Compose profile for a production-like local environment. The stack should be minimal but complete.

Required services:

- application API;
- worker, if the product has one;
- metrics storage and scraping: VictoriaMetrics;
- log storage: VictoriaLogs;
- log collector/forwarder: Vector;
- dashboard UI: Grafana.

Topology requirements:

- VictoriaMetrics scrapes the API metrics endpoint, the worker metrics endpoint, and optionally the monitoring components themselves.
- Vector reads Docker container logs, filters to the relevant services, parses JSON log lines when possible, normalizes key metadata such as service name and container name, and forwards newline-delimited JSON to VictoriaLogs.
- Grafana is provisioned from files committed to the repository, not by manual UI clicks.

Do not require a separate Prometheus server if VictoriaMetrics can perform scraping directly.

## Grafana Provisioning

Provision Grafana so a new environment already contains:

- a metrics datasource pointing at VictoriaMetrics;
- a logs datasource pointing at VictoriaLogs;
- at least two dashboards:
  - service overview;
  - async jobs and streaming or an equivalent workflow dashboard.

Dashboard content should answer these operator questions:

- What is current request throughput?
- What is the error rate?
- What is average or p95 latency?
- Are background jobs piling up?
- Which statuses are failing?
- Are streaming or session flows active and healthy?
- Can I pivot from a metric spike to matching logs?

## Product-Specific Signal Mapping

Before implementing, inspect the target project and map its business flows onto the observability categories above.

Examples:

- synchronous API call;
- asynchronous job creation and execution;
- long-running background processing;
- real-time streaming session;
- model or backend selection;
- optional processing features;
- degraded-but-acceptable execution paths.

For each flow, decide:

- which final states are meaningful;
- which identifiers should appear in logs;
- which dimensions deserve metric labels;
- which failures should appear in readiness versus only in logs and metrics.

Do not blindly copy ASR-specific names into a different domain. Preserve the structure, not the vocabulary.

## Implementation Constraints

- Keep the change set small and local.
- Prefer additive changes over invasive rewrites.
- Use repository-native configuration patterns.
- Use stdout for application logs inside containers.
- Avoid adding tracing in this phase unless it is already a project standard.
- Avoid vendor lock-in inside application code. The app should emit clean logs and metrics without knowing Grafana or Victoria internals.

## Expected Deliverables

Create or update all of the following classes of artifacts in the target repository:

1. Runtime code
   - shared logging configuration;
   - request ID and access-log instrumentation;
   - metrics registry or metrics wiring;
   - health, readiness, and metrics endpoints;
   - worker metrics exposure if applicable.

2. Deployment files
   - Docker Compose services and health checks;
   - scrape configuration for VictoriaMetrics;
   - Vector configuration for Docker log ingestion and normalization;
   - Grafana provisioning files for datasources and dashboards.

3. Tests
   - unit or integration tests for log formatter behavior if the project tests logging helpers;
   - tests or smoke checks for `/health`, `/ready`, and `/metrics`;
   - tests that confirm key metrics appear after representative requests or job events.

4. Documentation
   - README or ops notes that explain how to start the local stack;
   - URLs for metrics, logs, and dashboards;
   - verification commands.

## Validation Standard

Do not call the task complete until all of the following are demonstrably true:

1. Starting the local stack succeeds.
2. `GET /health` returns success.
3. `GET /ready` returns success only after the application is actually ready.
4. `GET /metrics` on the API exposes product metrics.
5. The worker metrics endpoint exposes worker metrics if a worker exists.
6. Making representative requests or jobs causes both logs and metrics to change.
7. Grafana can query both metrics and logs with provisioned datasources.
8. At least one dashboard shows useful product-specific panels without manual setup.

Use real verification commands in the target repository. Prefer the project package manager and test runner already in use.

## Suggested Execution Sequence

1. Identify the API entrypoint, background worker entrypoint, and deployment files.
2. Add shared JSON logging first.
3. Add request correlation and access logs.
4. Add metrics registry and wire the API metrics endpoint.
5. Add worker metrics and worker lifecycle instrumentation.
6. Add readiness and health behavior if it is incomplete.
7. Add Compose services for VictoriaMetrics, VictoriaLogs, Vector, and Grafana.
8. Provision datasources and dashboards from files.
9. Add or update tests and smoke checks.
10. Run validation end to end and document the exact commands and expected outputs.

## Non-Goals

- Do not add full distributed tracing unless explicitly required.
- Do not build a massive platform abstraction layer.
- Do not couple business logic to Grafana, Vector, or Victoria-specific APIs.
- Do not emit duplicate access logs from multiple middleware layers.
- Do not expose only infrastructure metrics while ignoring domain outcomes.

## Definition of Done

The subsystem is done when another engineer can clone the repository, start the production-like Compose profile, hit the service, and reliably answer these questions without reading source code:

- Is the service alive?
- Is it ready to serve real traffic?
- What requests or jobs happened?
- Which ones failed, and why?
- How much work is the system processing?
- Where can I inspect logs and dashboards locally?

If any of those questions still requires tribal knowledge, the subsystem is incomplete.
