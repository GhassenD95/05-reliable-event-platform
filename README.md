# Reliable Payments Event Platform
**Transactional outbox, idempotent processing, signed webhooks, and safe replay — with SLOs.**

> Real product: web UI + API + workers + DB/broker (not a CLI).  
> Built around payments-grade reliability problems seen in PFE offers: **Singleton** (payment infrastructure), **Liberrex** (integration hub), **ODDO BHF** (SRE/observabilité, GitOps), **Medius** (payments operations).

## Why this project
Money-adjacent flows fail in expensive ways: duplicates, lost events, and unsafe replays. Publishing to a queue is easy; **exactly-once effects over at-least-once delivery, under failure, with proof** is the real engineering bar. This project models payment/invoice domain events end-to-end with that guarantee.

## What you can do
- **Event ingestion & outbox:** emit domain events (e.g. payment/invoice state changes) via a **transactional outbox** — no dual-write gaps.
- **Reliable delivery:** signed webhooks to partner endpoints with retries (backoff/jitter), per-endpoint rate limits, ordering keys, and dead-letter queues.
- **Idempotent processing:** consumers/effects are idempotent so redelivery never double-applies; dedupe by idempotency keys.
- **Operations console (UI):** inspect events/deliveries, endpoint health, DLQ triage, and trigger **bounded, safe replays** (time window / event set) with guardrails and audit.
- **Observability & SLOs:** delivery latency, success/error rates, backlog, and error-budget burn; runbooks for incidents and replay.

## Architecture
```text
[Producers / API (FastAPI)] --tx--> [Postgres: domain tables + OUTBOX]
                                            |
                                      [Relay/worker] --> [Broker: Redis Streams | Redpanda/Kafka]
                                                                |
                                                   [Delivery workers: idempotent,
                                                    retries/DLQ, HMAC-signed webhooks]
                                                                |
[Ops Console (React/Next)] <-> [FastAPI: events, deliveries, replay, SLOs] <-> [Metrics/Logs]
```
- Effects are deduplicated so replay/redelivery is safe; replays are explicit, scoped, and audited.
- HMAC-signed payloads with timestamp/nonce protection.

## Hard engineering problems
- **No lost/duplicate effects:** transactional outbox + idempotent handlers across crashes and retries.
- **Failure semantics:** poison messages to DLQ, backoff/rate limiting, per-endpoint isolation, bounded replay.
- **Ordering & concurrency:** ordering keys where required without global head-of-line blocking.
- **Operability:** end-to-end metrics, traces/logs, actionable alerts, and documented SLOs/runbooks.

## Proof (what “done” means)
| Claim | Evidence |
|---|---|
| No loss, no duplicates | Chaos tests (kill broker/consumer mid-flight) → zero lost/duplicate effects (checksummed) |
| Safe replay | Replay a window: resulting state/checksum unchanged vs apply-once |
| Resilient delivery | Flaky/slow endpoints → correct retries, DLQ routing, rate limiting |
| Performant | Sustained throughput and p99 delivery latency under load; backlog recovery time |
| Operable | Runbook (incident + replay), SLO definitions, metrics dashboard |

## Stack
Python 3.12+, FastAPI, Pydantic v2 · PostgreSQL (outbox, deliveries, dedupe, audit), SQLAlchemy/Alembic · Redis Streams (baseline) with adapter path to Redpanda/Kafka · workers (arq/Celery or custom consumers) · Prometheus metrics + Grafana/structured logging (OpenTelemetry-ready) · React/Next.js · pytest, integration/chaos tests (Testcontainers, fault injection), GitHub Actions · Docker Compose.

## Status
**Design / spec-first.** Milestones: `outbox & relay` → `idempotent delivery + retries/DLQ` → `signed webhooks & console` → `safe replay` → `chaos/load tests & SLOs`.
