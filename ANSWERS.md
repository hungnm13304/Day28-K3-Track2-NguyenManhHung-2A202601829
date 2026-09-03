# Day 28 Track 2 — Submission Answers

**Student:** Nguyễn Mạnh Hưng  
**Student ID:** 2A202601829  
**Submission mode:** Individual

## Technical trade-offs

- Kafka accepts ingestion asynchronously and uses the idempotency key as the
  replay identity. This keeps the HTTP acknowledgement fast, but the client
  must treat `202 Accepted` as queued work rather than a completed Delta write.
- Delta MERGE is the durable deduplication boundary. The merge source keeps the
  newest `(occurred_at, event_id)` for each idempotency key, so Kafka delivery
  order cannot choose the winner.
- Feast is read through its online API while Spark owns offline computation and
  materialization. This separates request latency from lakehouse processing,
  at the cost of accepting freshness delay between a Delta commit and an online
  feature row.
- Qdrant IDs are deterministic UUIDs derived from logical document IDs. Replay
  therefore updates one point instead of growing the collection, but a document
  identifier must remain stable throughout its lifecycle.
- MLflow promotion uses the `champion` alias rather than a deployment-time
  model version. Rollback is immediate and reproducible, while every release
  must retain data, prompt, embedding, and served-model provenance.
- Readiness distinguishes a mandatory dependency failure (`not_ready`) from an
  optional one (`degraded`), allowing the gateway to protect users without
  hiding reduced capability.

## Production gaps and next steps

- This lab uses a single Kafka broker, local volumes, and development-grade
  credentials; production needs replicated brokers, managed durable storage,
  secret management, backup/restore drills, and separate environments.
- Compose proves integration locally but is not a production scheduler.
  Kubernetes manifests and GitOps validation are included; a production rollout
  additionally needs a real cluster, image registry, admission policy, autoscaling,
  resource quotas, and canary/rollback monitoring.
- The compose GPU profile serves public `Qwen/Qwen3-1.7B` vLLM 0.28.0 on the
  local RTX 5060 Ti. Startup required a temporary 0.80 VRAM-utilization
  override because the default 0.92 exceeded free memory; IP07 identity and
  GPU golden-path tests now pass. Any capacity number must still be measured
  on this hardware/model/corpus before production extrapolation.
- Local Jaeger validates trace continuity. With the local-only credential file,
  the LangSmith gate now passes against the existing `day22-lab` project; no
  mock trace or fabricated credential is used.
- The lab's `admin/admin` Grafana credentials and open local ports are for
  classroom use only. Production needs SSO/RBAC, TLS, network policy enforcement,
  audit logs, and secret rotation.

## Individual contribution

Nguyễn Mạnh Hưng completed the work as an individual:

- implemented IP01/IP10 Kafka trace and idempotency headers;
- implemented IP03 replay-safe Delta merge source preparation;
- implemented IP04 Feast online feature request contract;
- implemented IP07/IP08 readiness classification;
- ran the fast validation suite, full-stack integration journeys, operational
  evidence collection, failure/recovery exercise, profiling, and manifest checks;
- prepared the architecture/ownership document and this reflection.

## Failure and recovery note

- During the first golden-path trigger, Airflow reported `polled=0` and no
  Delta tables were created. Kafka offsets and direct consumer reads showed the
  broker was healthy; the cause was an ingestion-consumer startup race on the
  initial DAG attempt, not lost input.
- I triggered one remediation DAG run after the consumer was ready. It drained
  the backlog, emitted the Delta tables, refreshed Feast, and indexed Qdrant;
  the replay/golden-path suite then passed. Delta row counts and transaction
  history provide the no-data-loss proof in `evidence/ip02-airflow-run.json`
  and `evidence/ip03-delta-history.json`.
- A burst through Envoy intentionally produced HTTP 429 responses after the
  configured 10 RPS limit. The signal was the 429 plus `x-request-id`; the
  recovery is client backoff/retry while the gateway remains healthy. The
  complete 200/429 capture is in `evidence/ip08-gateway.json`.

## Validation record

The attached `evidence/` directory is runtime output and is intentionally
git-ignored; submit it alongside this repository. It must contain the exact
integration-matrix evidence filenames and the command outputs for the checks
listed in `SUBMISSION.md`. GPU and LangSmith gates are reported separately so a
skipped external credential gate is never represented as a pass.

LangSmith verification: `1 passed, 71 deselected` with `LANGSMITH_PROJECT=day22-lab`.

Load probe note: 200 readiness requests through Envoy returned the configured
local rate-limit behavior under burst load (8 workers: 14 HTTP 200, 186
rejected/connection-level failures; 16 workers: 3 HTTP 200, 197 rejected),
with observed p50/p95/p99 of 0.76/307.57/360.01 ms and 3.12/5.20/215.96 ms.
