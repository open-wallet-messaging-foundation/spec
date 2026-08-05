# WM-6-notifications

**Status:** Draft (2026-07-13), written from the v0 reference implementation
(`rust/owm-notifier`). **Profile:** see WM-0.

**Scope:** Sink driver manifests; trust classes D/C; the knock-only
guarantee; blinded subscriptions; append-only storage and idempotent
dispatch; the v0 reference drivers (Telegram, Web Push); content-over-C
rules (self-hosted or scoped preview keys only — not implemented in v0).

---

## 1. Overview and the knock-only guarantee

A **notifier node** bridges OWM traffic to out-of-band delivery media —
push, chat apps, SMS, email, anything — so a wallet owner notices new
messages even with no wallet app open. The medium list is **open**: the
standard specifies the manifest shape and the knock contract, not an
approved-media list. Anyone can write a driver.

**The knock-only guarantee (normative for v0):** the notifier never sees
message content, sender identity, or any wallet identity. It handles only:

- **blinded topics** — e.g. `base64url(HMAC(topic, user_blind_key))`,
  meaningless to the notifier and unjoinable to wallets or XMTP topics;
- **priority floors / priorities** — small integers (0–255) used purely for
  filtering;
- **opaque credential blobs** — base64, decoded only inside the dispatch
  path and handed to the driver;
- **knock ids** — sender-supplied idempotency keys.

What a knock delivers to the end medium is a **fixed, content-free
template** (reference text: *"You have new wallet messages waiting. Open
your wallet app to read them."*). It is a compile-time constant with no
interpolation: it cannot carry topic, sender, count, or content by
construction.

**What the notifier still learns (stated honestly):** that *some
subscriber* groups N blinded topics onto one delivery credential, and the
traffic timing on them. It cannot read content (E2EE end to end), cannot
map topics to wallets, and MUST NOT log source associations. This residual
linkage (credential ↔ topic-set ↔ timing) is *the* reason knock-only is the
default and Class D sinks are preferred. Rotating blind keys per MLS epoch
further shears linkage.

**No wallet identity fields exist anywhere in the Notifier API.** All
request bodies are validated strictly with unknown-field rejection: any
attempt to send an extra field (wallet address, inbox id, anything) is a
`400`.

## 2. Trust classes

- **Class D — decentralised.** Dispatch/decryption/policy run in the user's
  own secure environment (handset app, browser service worker, desktop
  agent). The push pipe carries at most ciphertext. *Honest footnote: the
  OS vendor's push pipe (APNs/FCM/a browser push service) still sees that a
  push happened — "Class D" refers to where decryption and policy run, not
  to the pipe.*
- **Class C — centralised.** The notifier node holds a channel credential
  and calls a provider (a chat-bot API, SMS gateway, SMTP…). Reaches a user
  with no app installed. Payload: **knock-only by default**.

Content over Class C is an explicit opt-in with exactly two conforming
mechanisms — (a) a self-hosted notifier (you are your own provider), or
(b) a conversation-scoped preview key explicitly exported by the user,
scoped and revocable, with a permanent UI disclosure. A notifier MUST NOT
ever hold inbox-level keys. **Neither mechanism is implemented in v0; every
v0 manifest carries `content_capable: false`.**

## 3. Sink driver manifests — `GET /owm/v1/sinks`

A **sink** is a delivery endpoint described by a driver manifest. The
notifier advertises its drivers as a JSON array:

```jsonc
[
  { "driver": "telegram", "class": "C", "content_capable": false,
    "credential_kind": "chat_id",             "latency": "seconds", "cost": "free" },
  { "driver": "webpush",  "class": "D", "content_capable": false,
    "credential_kind": "webpush_subscription", "latency": "seconds", "cost": "free" }
]
```

| Field | Meaning |
|---|---|
| `driver` | Registry key; also the value clients pass when subscribing. |
| `class` | `"D"` decentralised \| `"C"` centralised (§2). |
| `content_capable` | Whether the sink could carry E2EE payload for local decrypt. Always `false` in v0. |
| `credential_kind` | What the sink needs to deliver (`chat_id`, `webpush_subscription`, `phone_e164`, …). |
| `latency` | Coarse expectation (`instant` \| `seconds` \| `minutes`). |
| `cost` | `free` \| `per-message` \| `subscription`. |

## 4. Subscription model — `POST /owm/v1/subscriptions`

The client's policy engine runs **on the user's device — always**. For
Class C sinks the device compiles its policy into blinded subscriptions:
it registers only the topics that could ever trigger that sink, tagged with
a priority threshold. The notifier applies filters without knowing what
they mean.

Request (strict; unknown fields → `400`):

```jsonc
{
  "blinded_topic":  "bt_9f2c…",   // 1..=128 chars of [A-Za-z0-9._:-]
  "driver":         "telegram",   // must appear in GET /owm/v1/sinks
  "credential":     "Y3JlZA==",   // opaque blob, standard base64, 1..=4096 decoded bytes
  "priority_floor": 10,           // 0..=255; knocks below this never fire
  "ttl_s":          3600          // 60..=31536000
}
```

`201 → {"id": "<uuid>"}`. Validation failures — malformed JSON, unknown
driver, invalid base64, out-of-range `ttl_s`, extra fields — are `400` and
append nothing.

`DELETE /owm/v1/subscriptions/{id}` → `204`. Removal **appends** a
`subscription_removed` event; nothing is ever deleted. Deleting an
already-removed id is `204` again (idempotent); an id that never existed is
`404`. A removed subscription never fires again, including for knocks that
arrived before removal but were not yet dispatched.

Expiry: a subscription is live while `now < created_at + ttl_s`. Expired
subscriptions never fire; there is no refresh in v0 — re-subscribe.

The credential blob is opaque to the API surface and decoded only inside
the dispatch path. (The v0 reference node stores it as registered;
at-rest encryption under a KMS-held key is a deployment concern layered on
the same contract.)

## 5. Knock ingest — `POST /owm/v1/knock` (v0-dev)

**Honest status:** in the full standard, knocks originate from an XMTP
topic watcher (kind 522 `notify-knock`, see WM-5 / `api/kinds.json`
520–523). The v0 reference node has no watcher yet, so it exposes knock
ingest as an **authenticated-by-nothing dev endpoint**. Do not deploy v0
knock ingest on an open network; it exists so the dispatch path is real and
testable end to end.

```jsonc
{ "blinded_topic": "bt_9f2c…", "knock_id": "k-01H…", "priority": 50 }
```

`202` always (accepted, including replays); strict validation as above.
`knock_id` is the **sender-supplied idempotency key**: replaying a
`knock_id` is accepted and dispatches exactly nothing new.

## 6. Storage: an append-only event log

The store interface has exactly two operations: **append** and **scan**.
No update, no delete — the trait does not expose them, and the reference
Postgres schema (`rust/owm-notifier/schema.sql`) installs a trigger that
raises on any `UPDATE` or `DELETE`. Current state is always a fold over
events:

| Event | Meaning |
|---|---|
| `subscription_created` | id, blinded_topic, driver, credential (opaque, base64), priority_floor, expires_at |
| `subscription_removed` | unsubscribe — an append, not a delete |
| `knock_received` | knock_id, blinded_topic, priority, ts (deduplicated by knock_id at fold time) |
| `dispatch_attempted` | knock_id, subscription_id, driver, outcome (`ok` \| `failed{reason}`) |

Two reference implementations: an in-memory log (tests, dev) and Postgres
(cargo feature `postgres`; single `events` table — `id bigserial, ts, kind,
body jsonb`). The event bodies contain no wallet address and no inbox id —
there is no variant that could carry one.

## 7. Idempotent dispatch — exactly-once-ever per (knock_id, subscription_id)

For each knock, the dispatcher folds the log and selects live
subscriptions where:

1. `subscription.blinded_topic == knock.blinded_topic`, and
2. `knock.priority >= subscription.priority_floor`, and
3. the subscription is neither expired nor removed, and
4. **no `dispatch_attempted` event exists for `(knock_id,
   subscription_id)`** — any outcome.

For each selected pair it executes the driver send, then appends
`dispatch_attempted` with the outcome. Idempotency lives entirely in the
log, never in memory: a fresh dispatcher process over the same store
performs **zero** driver actions on a second run. This is the
**second-run-zero-actions** property, and it is enforced by a test of that
name (`rust/owm-notifier/tests/dispatch.rs`) plus a Postgres variant.
Two subscriptions on the same topic each fire exactly once per knock.

**Failure semantics (v0):** a failed driver send is recorded as
`outcome: failed{reason}` and is **not auto-retried** — the attempt row,
whatever its outcome, means "never send again". A retry, when specified,
will be an explicit new event (a deliberate re-arm), never an implicit
re-scan; this is what keeps a webhook retransmit or firehose replay from
double-delivering. **Crash caveat (stated honestly):** the send executes
before the attempt row is appended, so a crash exactly between the two can
produce one duplicate on restart — dispatch is exactly-once-ever in the
log and at-least-once across crashes.

## 8. v0 reference drivers

Drivers implement `manifest()` + `send_knock(credential, knock)`. Real
drivers are never exercised by tests; only their request *construction* is
unit-tested, and a recording mock driver is used everywhere else.

### 8.1 `telegram` (Class C)

Bot API `sendMessage`: `POST
https://api.telegram.org/bot<token>/sendMessage` with body
`{"chat_id": <credential>, "text": <fixed knock template>}` — exactly those
two fields. Bot token from `OWM_TELEGRAM_BOT_TOKEN` at send time; the
credential blob decodes to the chat id (UTF-8).

### 8.2 `webpush` (Class D)

RFC 8030 (HTTP push, `TTL`/`Urgency`), RFC 8291 (`aes128gcm` message
encryption, single record, padding delimiter `0x02`), RFC 8292 (VAPID ES256
JWT: `Authorization: vapid t=<jwt>, k=<uncompressed P-256 public key,
base64url>`). Implemented directly on p256 + hkdf + aes-gcm (no openssl
dependency); the encryption round-trip is verified in tests against a
simulated user agent. The credential blob decodes to the standard browser
`PushSubscription` triple `{endpoint, p256dh, auth}` (https endpoints
only; unknown fields rejected). VAPID keys from `OWM_VAPID_PRIVATE_KEY`
(base64url raw 32-byte P-256 scalar) and `OWM_VAPID_SUBJECT`
(e.g. `mailto:ops@example.org`). `Urgency: high` when knock priority ≥ 128,
else `normal`. Even though the payload is a content-free knock, it is
encrypted per RFC 8291 like any push payload.

## 9. Reference node

`rust/owm-notifier`: binds `OWM_NOTIFIER_ADDR` (default `127.0.0.1:8758`),
startup line `owm-notifier listening on <addr>`. In-memory event log by
default; `--features postgres` + `DATABASE_URL` for the durable log.
`cargo test -p owm-notifier` passes with zero external services; the
Postgres integration test runs only when `DATABASE_URL` is set and skips
loudly otherwise.

## 10. Related kinds (WM-5)

| Kind | Wire | v0 mapping |
|---|---|---|
| 520 | `notify-subscribe` | `POST /owm/v1/subscriptions` (HTTP in v0; in-band envelope form later) |
| 521 | `notify-unsubscribe` | `DELETE /owm/v1/subscriptions/{id}` |
| 522 | `notify-knock` | `POST /owm/v1/knock` (v0-dev; XMTP watcher later) |
| 523 | `notify-receipt` | the `dispatch_attempted` event row |
