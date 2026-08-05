# WM-11-cast — OWM-CAST: paid 1:many broadcast & the entitlement layer

**Status:** DRAFT FOR REVIEW (2026-08-04). Nothing frozen; **(open)** marks
decisions still to make. **Profile:** see WM-0.

**Scope:** a wallet address broadcasting encrypted content to *many* addresses
where **only paying addresses can decrypt** — a live performance, a stream, a
paywalled feed, a webinar, a paid API firehose. OWM is the **entitlement + key +
settlement layer** (the decentralised, payment-native equivalent of a DRM
*license server*); it does **not** carry the media bytes — a CDN does that, and
OWM gates the keys. Built on MLS groups + WM-4 broadcast (`ticket`, 546) + stage
(540–542) + settlement (543–545) + WM-8 sealing + WM-10 grants + a WM-9 ticket
binding. Proposed kinds `owm-cast-*` (560–569 sub-band, §12).

Reference implementation: planned. Conformance vector: planned.

---

## 1. What this is (and is not)

The demo is Taylor Swift going live and only ticket-holders hearing it. The
primitive is **paid group broadcast**: encrypt once, let exactly the paid set
decrypt, revoke instantly, and settle payment in the same signed flow.

**The elegant core:** the cast *is* an **MLS group**; the ticket *is*
**membership**; the encryption *is* the **group key**. A confirmed payment is what
adds an address. MLS gives efficient group rekeying and — the property that makes
this work — **forward secrecy**, so removing a member (chargeback, leak, expiry)
denies them *all future frames* immediately.

**What it is NOT — the one architectural honesty:** OWM is not a video pipe. It is
the layer that (a) binds *payment → decryption entitlement*, (b) distributes and
rotates *content keys* to entitled addresses, and (c) *revokes*. A media transport
(WebRTC/HLS/Livepeer) moves the encrypted bytes; OWM hands out the keys that open
them. This is exactly what a Widevine/FairPlay license server does today — OWM
makes it decentralised, self-sovereign, and payment-native. Pretending OWM
messaging streams 4K would be dishonest and is out of scope.

---

## 2. The model

```
Cast        { castId, broadcaster address, title, kind: live|stream|feed, keyEpoch policy }
Entitlement { holder address, castId, tier, window (event | subscription), settlementRef, sig(broadcaster) }
Membership  = the MLS group of the cast; presence ⇔ current decryption right
ContentKey  = the MLS group key for the epoch (media segments encrypted under keys sealed to it)
```

A **cast** is a broadcast context. An **entitlement** is formally a **WM-13
attestation** (`claimType: paid-access`) issued by the broadcaster, binding a
holder's address to the cast for a window and referencing the payment that bought
it; **resale is a WM-15 transfer** (§5). **Membership** of the cast's MLS group is
the *live* decryption right; the entitlement is the *proof* that earns membership
and the artifact presented at a physical gate (WM-12 — a WM-14 presentation).

---

## 3. Access = membership; encryption = the group key (normative)

- The broadcaster owns the cast's MLS group. Admitting an address = adding a
  member; the content key is the group key, so one encryption reaches every paid
  member and **no one else**.
- **Payment → membership.** A confirmed settlement (543/544, or an external
  on-chain tx referenced by WM-8 pointer) triggers issuance of an entitlement and
  admission to the group. The broadcaster (or a ticketing delegate holding a
  WM-10 `set`-scoped grant on the cast) performs the add.
- **Revocation = MLS member-removal.** Forward secrecy is the enforcement: a
  removed address cannot decrypt any frame after removal — chargeback, credential
  leak, or window expiry all resolve to "remove from group."
- **Epochs.** Long streams rotate the group key on an epoch schedule; a key
  compromise exposes only its epoch.

---

## 4. Payment binding & non-custodial issuance (normative + open)

The buyer proves payment; the broadcaster signs the entitlement; neither holds the
other's keys. `settlementRef` binds the entitlement to a specific WM settlement or
on-chain tx (resolved via WM-8 / pubkey-resolver). The broadcaster never learns
more than the buyer's address (WM-9 wedge). **(open)** exact settlement proof:
a WM 543 settlement-card, an on-chain receipt, or a payment-channel voucher for
per-second streaming.

---

## 5. Resale & transfer — anti-scalping as a feature (open, headline)

Because an entitlement is a wallet-bound signed object, **resale is a signed
transfer**, and the broadcaster sets policy: **allow / forbid / price-cap /
royalty**. A transfer is a **WM-15 transfer** (kind 600) signed by the current
holder, counter-admitted by the broadcaster per the issuer policy on the
entitlement attestation; the old holder is removed from the
group (forward secrecy), the new holder added. The artist owns their secondary
market — cap prices, take a cut, or make tickets non-transferable — the thing
Ticketmaster monetises and fans resent. This may be the strongest reason to adopt.

---

## 6. Streaming / subscription profile (open)

Netflix-shape: an entitlement with a rolling `subscription` window; while active
the address stays in the group and receives epoch keys; on lapse it is removed.
Per-title casts or one account-cast with per-title namespaces (compose WM-10).

---

## 7. Honest ceilings (house rule — DRM is never perfect)

| Ceiling | Reality | Mitigation (not a fix) |
|---|---|---|
| **Analog hole** | A *paying* viewer can screen-record and re-broadcast. No crypto stops this; Widevine doesn't either. | Per-recipient **watermarking / traitor-tracing** makes a leak traceable to the leaker — deters, never prevents. |
| **Key / seat sharing** | A payer can hand keys to a freeloader. | Bind access to the **funded wallet** ("your ticket is your wallet" — sharing risks your money) + traitor tracing. |
| **Scale** | MLS groups handle *thousands*, not the millions of a global tour. | Shard into many groups, or a key-server distributing per-segment keys to verified-paid addresses (broadcast-encryption layer). |
| **Latency** | Messaging is not a low-latency media transport. | OWM distributes keys/entitlements; a real CDN carries bytes. |

OWM's honest promise: it can guarantee **only paying addresses receive the keys**,
and can **revoke future access instantly** — it cannot stop a paying viewer from
re-filming their own screen. That boundary is stated up front, not buried.

---

## 8. Composability (what it consumes / exposes)

**Consumes:** **Attestation** (WM-13 — the entitlement is a `paid-access`
attestation), **Transfer** (WM-15 — resale), **Presentation** (WM-14 — the
turnstile), **Group** (MLS, WM-1/2 — decryption), settlement (543–545), WM-8
sealing (524–526), grants (WM-7 — delegated ticketing), WM-4 broadcast `ticket`
(546) + stage (540–542), WM-12 (the QR at the gate).

**Exposes — the reusable seam:** a signed, revocable, transferable **entitlement**
+ **paid-group** primitive that generalises *far* beyond concerts:
paywalled journalism, online courses (pay → cohort group), webinars, DAO/token-
member content, paid data/API firehoses, family media sharing, even software
licences (an entitlement = a licence key that revokes). Anything that is
"exactly these addresses may decrypt this 1:many stream, because they paid /
qualify" is this primitive. Designing the *entitlement object* cleanly is what
lets use-cases we haven't imagined reuse it.

---

## 9. Open questions

1. **Cast vs KV:** is a cast a first-class object, or a WM-10 namespace with a
   membership group? (Leaning first-class for the media/epoch semantics.)
2. **Settlement proof** (§4) — which forms in v0 (543 card / on-chain / channel)?
3. **Transfer policy** (§5) — allow/forbid/cap/royalty in v0, or allow/forbid only?
4. **Scale** — v0 caps at MLS-group size (thousands) with a stated ceiling; the
   sharded/broadcast-encryption tier as WM-11a?
5. **Traitor-tracing / watermarking** — spec a hook now (per-recipient key or
   watermark id) or note & defer?
6. **Key transport for CDN media** — standardise segment-key sealing to the group,
   or leave to the media profile?

---

## 10. Prior-art anchors

Widevine / PlayReady / FairPlay + EME/CENC (the centralised license servers OWM
decentralises), Unlock Protocol & Lit Protocol token-gating (NFT-gated
access/decrypt), Livepeer / Theta (the decentralised CDN layer), Ticketmaster
SafeTix (rotating-barcode tickets), POAP / NFT tickets, MLS RFC 9420 (group key +
forward secrecy = revocation), broadcast encryption (Naor–Naor–Lotspiech subset-
difference), traitor tracing (Boneh–Franklin), payment channels (per-second
streaming settlement).

---

## 12. Proposed kinds (not yet reserved — held off the live registry)

A `560–569` "cast" sub-band. The **entitlement is a WM-13 attestation** (kind 580,
`claimType: paid-access`) and **resale is a WM-15 transfer** (kind 600) — WM-11
mints **no** bespoke entitlement or transfer kinds; it adds only cast-specific
envelopes:

```
560  owm-cast-config   # a cast's public descriptor
563  owm-cast-key      # sealed epoch/segment key distribution
564  owm-cast-revoke   # explicit revoke (beyond MLS removal)
561-562, 565-569  reserved (cast)
```

Delegated ticketing reuses WM-7 grants (533 / `owm.cast.*`). No registry change
until we agree the shape.
