# Message Service — Agreed Architecture (notes)

## Goals

* Async, provider-agnostic message sending over NATS/JetStream.
* Searchable UI/API (list, filter, inspect, resend).
* No distributed transactions (no XA/2PC). Robust to partial failures.
* Clear audit trail and “latest status” that’s always correct.

## First Principles

* **Truth lives on the broker.** All state changes are JetStream **events**.
* **Latest view lives in NATS KV** with **CAS** (never go backwards).
* **Database is a projection** for search/list only. If it lags, that’s fine; we can replay.
* **ACK only after persisting to the broker** (event + KV CAS). DB uptime is not required to send.

---

## Two Independent State Machines (per message)

* **handoff_state** (our pipeline): `created → queued → sending → handed_off | failed`
* **provider_state** (external world): `unknown → accepted → delivered | bounced | dropped | complaint | timed_out`

Keep them **decoupled**. Each has its own events and “latest” KV key.

### State ranks (monotonic writes)

Use ranks to avoid regressions on out-of-order events:

* handoff: `created(0), queued(10), sending(20), handed_off(100), failed(100)`
* provider: `unknown(0), accepted(10), delivered(100), bounced(100), dropped(100), complaint(100), timed_out(90)`

Only apply an update if `incoming.rank >= current.rank` (enforced via KV CAS).

---

## JetStream & KV Shape

### Streams / subjects (example)

* `msg.commands` — high-level commands (`SendRequested`, `ResendRequested`)
* `msg.handoff` — our pipeline events

  * `MessageCreated`, `HandoffAttempted`, `HandoffAccepted`, `HandoffFailed`
* `msg.provider` — external status events

  * `ProviderAccepted`, `ProviderDelivered`, `ProviderBounced`, `ProviderDropped`, `ProviderComplaint`, `ProviderTimedOut`
* `provider.check.<provider_key>` — work-queue for polling (when webhooks aren’t available)

**Dedup:** set `Nats-Msg-Id = sha256(message_id, attempt_no, event_type)`.

### KV buckets (latest)

* `kv/latest/handoff/<message_id>` → `{state, rank, version, at}`
* `kv/latest/provider/<message_id>` → `{state, rank, version, at, provider_msg_id?}`

Use **CAS**: update only if revision matches (or if incoming `rank` is higher).

---

## Services / Workers

* **API**: validates requests, publishes `SendRequested` (no DB writes required).
* **Sender**: consumes commands, renders template, calls provider.

  * Flow per message:

    1. publish `HandoffAttempted(attempt_no, attempt_id)`
    2. call provider with **idempotency key = hash(message_id, attempt_no)**
    3. publish `HandoffAccepted` **or** `HandoffFailed`
    4. KV CAS update `latest/handoff`
    5. **ACK**
    6. if accepted: enqueue `provider.check` OR rely on webhooks
* **Webhook Receiver (preferred)**: verifies provider signature, publishes `Provider*` events, KV CAS `latest/provider`, stops polling.
* **Provider Poller (fallback)**: consumes `provider.check.<provider_key>`, batches status lookups, emits `Provider*` events, re-enqueues with **exponential backoff** until terminal state or timeout (`ProviderTimedOut`).
* **Projector (DB indexer)**: subscribes to both event streams, upserts **projection docs** for UI queries. If DB is down, it catches up via replay later.

---

## Database (projection only; document-friendly)

Use Mongo (or similar) purely for query/search. Keep it rebuildable.

**messages (projection)**

```
_id, tenant_id, created_at, channel, provider_key,
recipient, subject, body_ref, metadata{tags,campaign,...},
provider_msg_id?, last_event_at
// optional derived fields (ok to omit): handoff_state, provider_state
```

**message_events (optional)**

* Either embed recent events in `messages` or keep a separate collection for timelines.

**Indexes (examples)**

* `{tenant_id:1, created_at:-1}`
* `{tenant_id:1, channel:1, created_at:-1}`
* `{tenant_id:1, provider_key:1, created_at:-1}`
* `{tenant_id:1, recipient:1, created_at:-1}`
* `{provider_msg_id:1}`
* Text index on `{subject, body_text}` or use Atlas Search for fuzzy.

> If you dislike derived `status` fields here—drop them. The UI overlays KV “latest” for freshness.

---

## Read Paths (UI/API)

* **List/Search**: query DB projection (fast + indexed).
* **Per-row freshness**: fetch KV `latest/*` for the visible message_ids and overlay current `handoff_state` and `provider_state`.
* **Details view**: DB doc + recent events + KV overlay.

---

## Idempotency & Retries

* **Provider calls**: deterministic `attempt_id` → pass as provider idempotency key; safe on retries.
* **Event publish**: dedupe via `Nats-Msg-Id`.
* **Handlers**: make projector idempotent (e.g., upsert by `(message_id, attempt_no, event_type)`).

---

## Resend

* Publish `ResendRequested{message_id}`.
* Sender increments `attempt_no`, repeats handoff flow with a new `attempt_id`.
* Keep thread under the same `message_id` (or create a new message linked via `original_message_id`—your call).

---

## Failure Semantics (no XA)

* If event publish or KV CAS fails → **do not ACK** → redelivery.
* If provider accepted but crash before ACK → on redelivery you re-call with same idempotency key (no double-send), re-emit same event (dedup), then ACK.
* DB outages do not block sending; projector replays later.

---

## Retention, PII & GDPR

* Bodies/attachments in **object storage** (S3/MinIO) with per-message encryption keys.
* PII erasure = **crypto-shred** the key + tombstone in projection. Events remain opaque.
* JetStream retention for events; DB can prune via TTL/archival policy as needed.

---

## Observability & Correlation

Include in every event/log:

* `message_id`, `attempt_no`, `attempt_id`, `provider_msg_id?`, `tenant_id`, `nats_seq`, `trace_id`.
  Correlate across sender, webhook/poller, projector. Emit metrics by state transitions (rates, latencies, error codes).

---

## Operational Knobs

* JetStream: **explicit ACK**, `AckWait` ~30–60s, `MaxDeliver` with backoff.
* Consumers: **durables** and work-queue (deliver to one).
* KV: CAS writes; optional TTL if you want to bound growth.
* Backoff policy for polling: e.g., 30s → 2m → 10m → 30m → 2h → cap.

---

## Nice-to-haves / Later

* Add **search index** (Atlas Search/OpenSearch) if UI wants fuzzy/full-text.
* Export to a **warehouse** for heavy analytics (BI) without touching OLTP.
* Optional **MCP server** as a thin adapter if/when LLM/agents become clients.

---

## Mini pseudocode (ACK-after-broker-persist)

```python
# outcome = call_provider(...)
evt = make_event(message_id, attempt_no, outcome)
pub_ok = js.publish("msg.handoff", evt, msg_id=dedupe_id(evt))
if not pub_ok:  # broker down?
    return  # don't ACK; redelivery will retry

kv_ok = kv.cas(f"latest/handoff/{message_id}", make_latest(evt))
if not kv_ok:
    # somebody moved it forward; fine. Re-read if needed.
    pass

msg.ack()  # only now
```

---

**Bottom line:**

* **Events (JetStream) = truth**, **KV CAS = fresh latest**, **DB = projection**.
* Two independent states (handoff vs provider), updated by sender + webhook/poller.
* No distributed transactions, no stale truth, easy replay, resilient by design.
