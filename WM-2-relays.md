# WM-2-relays

**Status:** Draft (2026-07-13). **Profile:** see WM-0.

**Scope:** OHTTP (RFC 9458) relay/gateway split; operator-diversity rule; rendezvous mailboxes; signed relay directory; payer-gateway role; stated non-goals (no mixnet claim)

## 1. Rendezvous mailboxes (v0-draft — implemented by `owm-rendezvous`)

A rendezvous mailbox is a short-lived, anonymous, two-sided drop box on a
relay. It exists for the flows that happen *before* an end-to-end encrypted
session exists — primarily the SCX PAKE handshake (WM-3) and join-request
handoffs. The design follows the magic-wormhole nameplate model: a **small,
human-speakable integer id** plus a strong secret spoken out of band, where
the id locates the mailbox and the PAKE built on the spoken words provides
the security.

### 1.1 Model

- A mailbox has exactly **two sides**: side `0` (creator) and side `1`
  (claimant). Each side holds a **side capability**: a 256-bit random bearer
  token (base64url, no padding), scoped to that one side of that one mailbox.
- Mailbox ids are the **lowest free integer ≥ 1** — deliberately short so an
  SCX code like `7-panda-mocha-quilt` can be read over a phone call. Ids
  are low-value by construction and are **reused** after expiry.
- Frames are **opaque byte blobs** (ciphertext/envelopes). The relay never
  parses, validates, or interprets them.
- Frames carry a per-mailbox, 1-based, monotonically increasing `seq`
  shared across both sides.
- **A side only ever reads the other side's frames** (wormhole semantics —
  no self-echo, no reflection confusion).

### 1.2 HTTP surface

Normative machine-readable definition: `api/openapi/owm-v0.yaml`. Summary:

| Method & path | Purpose | Success |
|---|---|---|
| `POST /owm/v1/mailbox` | Create; body optionally `{"ttl_s": n}` | `201` `{id, side_cap, ttl_s, expires_at}` |
| `POST /owm/v1/mailbox/{id}/claim` | Claim side 1 — first-come, exactly once | `200` `{side_cap, ttl_s, expires_at}` |
| `PUT /owm/v1/mailbox/{id}` | Append `{"frame": "<base64>"}` with `Authorization: Bearer <side_cap>` | `204` |
| `GET /owm/v1/mailbox/{id}?after=<seq>&wait=<secs>` | Long-poll the other side's frames, Bearer-gated | `200` `{frames: [{seq, side, frame}], next_after}` |

Error semantics:

- `404` — the mailbox does not exist (never created, or expired and
  vanished; the two are indistinguishable by design).
- `403` — missing or wrong side capability on an existing mailbox.
- `409` — second and later claims (`already-claimed`), or the per-mailbox
  frame cap reached (`mailbox-full`).
- `413` — a single frame over the size cap.
- `400` — malformed JSON, invalid base64, or malformed query parameters.
- `429` — rate limit exceeded (per source and per mailbox).
- `503` — the relay is at its live-mailbox capacity.

Error bodies are `{"error": "<kebab-case-code>"}`.

**Claiming requires no secret.** The id alone grants the second seat. This
is intentional: the mailbox is a meeting point, not a security boundary —
the PAKE run through it (WM-3) is what protects the exchange, and a wrong
guesser burns the one claim slot visibly (the legitimate party's claim then
fails, which aborts the exchange loudly rather than silently). Claims are
rate-limited to keep id-space sweeping impractical at scale.

### 1.3 Long-poll semantics

`GET` returns immediately when frames with `seq > after` from the other
side exist. Otherwise the relay holds the request up to `wait` seconds
(**cap 25 s**; larger values are clamped) and returns as soon as the other
side appends, or returns `{"frames": [], "next_after": <after>}` on
timeout. Clients loop, feeding `next_after` back as `after`. If the mailbox
expires while a poll is parked, the poll completes with `404`.

### 1.4 Limits and TTL (v0 defaults)

| Parameter | Default | Bound |
|---|---|---|
| TTL | 15 min | creator-requested `ttl_s` clamped to [1 s, 60 min] |
| Frame size (decoded) | — | 64 KiB per frame |
| Frames per mailbox | — | 32 total across both sides |
| Long-poll `wait` | 0 | 25 s cap |
| Rate limits | per-source and per-mailbox token buckets | operator-tunable |

At expiry the mailbox — frames, capabilities, everything — is deleted
(lazily on next touch and by a periodic sweep), and its id returns to the
free pool. There is no way to extend, resurrect, or enumerate mailboxes.

### 1.5 Privacy properties

- **No identities.** No field anywhere in the mailbox API names a wallet,
  account, inbox, or user. Capabilities are random bearer strings that die
  with the mailbox.
- **Opaque payloads.** Frames are ciphertext to the relay; conforming
  relays MUST NOT parse or filter on frame contents.
- **No retention.** State is in-memory and TTL-bounded; nothing persists
  past the mailbox lifetime, so there is nothing to disclose after the
  fact. Rendezvous relays are stateless by design beyond rate-limit
  counters and live-mailbox TTL state.
- **No source-association records.** Conforming relays MUST NOT log or
  store client address ↔ mailbox id associations, and MUST NOT log
  capabilities or frame contents. (Rate limiting keys on source addresses
  in memory only, without tying them to mailbox ids in any record.)
- **Anonymity against the network path** comes from running the mailbox
  API through the OHTTP relay/gateway split of §2, not from the mailbox
  itself.

### 1.6 Reference implementation

`rust/owm-rendezvous` in the reference repo: in-memory store, axum HTTP
surface, expiry sweep, token-bucket limits. Binds
`OWM_RENDEZVOUS_ADDR` (default `127.0.0.1:8757`; port `0` for ephemeral)
and prints `owm-rendezvous listening on <addr>` on startup.

## 2. OHTTP relay/gateway split — pending

RFC 9458 request/response split, operator-diversity rule (a conforming
OWM-PRIVATE client refuses a relay+gateway pair under one operator), and
the payer-gateway role. Content lands during the phase that implements it
(see repo README).

## 3. Relay directory — pending

Signed JSON document listing relay/gateway endpoints, operator identities,
jurisdictions, OHTTP key configs, and uptime attestation. Content lands
during the phase that implements it.

## 4. Non-goals

WM-2 does not claim resistance to a global passive adversary correlating
traffic timing across relays; it is not a mixnet. The property provided is
removal of IP↔topic (and IP↔mailbox-content) linkage at the infrastructure
the ecosystem operates.
