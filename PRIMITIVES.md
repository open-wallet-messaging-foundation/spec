# OWM composable primitives — the complete list

**Status:** Reference index (2026-08-05). This is the canonical catalogue of every
composable primitive in OWM. The design is deliberately a small **pantry** — every
product and use-case is a *recipe* over these, including ones we haven't listed.

- *Why* the core three (Attestation · Presentation · Transfer) are first-class:
  [FIRST_CLASS_PRIMITIVES](FIRST_CLASS_PRIMITIVES.md).
- Normative wire formats: the per-WM specs linked below.

Status legend: **stable** · **draft** (specified, reference-implemented) ·
**proposed** (reserved, spec drafted, not yet in the live registry).

---

## Tier 1 — the eight foundational primitives

Orthogonal building blocks; almost everything reduces to these.

| # | Primitive | What it is | Doc | Kind(s) | Status |
|---|---|---|---|---|---|
| 1 | **Signed statement** | a domain-tagged EIP-191 canonical the signer commits to — the atom under every OWM object; disjoint domains prevent cross-replay | WM-3/7/8/9/10/13/14/15 | (all signed objects) | draft |
| 2 | **Log + fold** | append-only signed events → current state is the *fold*; revoke wins; optional `prevId` hash-chain makes it tamper-evident | WM-9 §8b, WM-10 §6 | 527, 550–555, 600 | draft |
| 3 | **Grant** | a scoped, time-boxed, revocable capability (`owm.<domain>.<scope>`) — RBAC/delegation | WM-7 | 532–534 | draft |
| 4 | **Attestation** | an issuer signs a typed claim about a subject, trust-anchored — OWM-native Verifiable Credentials | WM-13 | 580 | draft |
| 5 | **Presentation** | a holder proves a signed object to an audience, challenge-bound (anti-replay) | WM-14 | 590 | draft |
| 6 | **Transfer** | hand a bound object to a new holder under issuer policy; CAS/fold chain-of-custody | WM-15 | 600 | draft |
| 7 | **Sealed payload** | AEAD encryption to N recipients — sign-to-derive x25519 identity, ECIES, the OWMF container | WM-8 | 524–526 | draft |
| 8 | **Group** | an MLS group — membership = access, forward secrecy = revocation | WM-1/2 | 400–405 | stable |

---

## Tier 2 — application primitives (recipes over Tier 1)

| Primitive | What it is | Doc | Kind(s) | Built from |
|---|---|---|---|---|
| **Identity & sub-identities** | CAIP-10 address, anonymous HD sub-identities, guest/actor types | WM-1 | — | — |
| **Sign-to-derive** | deterministic `personal_sign` → HKDF → keys, cross-curve | WM-7/8 | 524 | signed statement |
| **Address→pubkey resolution** | recover a pubkey from a signed txid; Solana address = key | WM-8 | — | signed statement |
| **Blind relay / rendezvous** | OHTTP relay/gateway split + rendezvous mailboxes (metadata privacy) | WM-2 | — | group |
| **Ping / knock / notify** | attention + the notification sink model (Class D/C) | WM-4, WM-6 | 500–502, 520–523 | signed stmt + group + relay |
| **Secure Contact Exchange** | PAKE (SPAKE2) + SAS + signed contact cards + invites | WM-3 | 510–519 | signed stmt + sealed |
| **Auth / step-up approval (2FA)** | challenge/response, sign-in, approval, mandatory-approve | WM-7 | 530–537 | signed stmt + grant |
| **Settlement intent** | the closed 5-method commerce core (transfer/token/contract-call/typed-sign/batch) | WM-4 | 543–545 | signed stmt + approval |
| **Bindings / switchboard** | address ↔ typed subject + capabilities + accept posture | WM-9 | 527–528 | self-attestation + grant + log/fold |
| **KV store** | personal record store, per-verb capabilities | WM-10 | 550–555 | log/fold + grant + sealed + group + presentation |
| **Cast / paid broadcast** | 1:many DRM entitlement layer (concert, stream, paywall) | WM-11 | 560–569 *(proposed)* | attestation + transfer + group + settlement |
| **Physical bridge (barcodes/QR)** | code ↔ address subject + QR presentation at a gate | WM-12 | 570–579 *(proposed)* | attestation + presentation + bindings |
| **Stage / live performance** | stage config, playback sync, cue | WM-11 (adj.) | 540–542 | signed stmt + group |
| **Duress / liveness** | safety signals, dead-man's-switch hook | WM-7 (adj.) | 538–539 | signed statement |
| **SSH (hardware auth)** | wallet-authenticated SSH; cert = attestation, principals = grants | WM-16 | 610–619 *(proposed)* | attestation + presentation + grant + auth |
| **Screening / accept posture** | Everything / Secure / Verified inbound policy | WM-4/6/9 | — | grant + trust anchors |
| **Trust anchors** | Web PKI reuse — `.well-known`/TLS + DNSSEC + KYB + CT | WM-9 §7 | — | attestation |

---

## Worked compositions (the whole point)

- **SSH login** = **attestation** (the certificate) + **presentation** (login, challenge-bound) + **grant** (which hosts/principals) + **auth** (the wallet authorises issuance).
- **Concert ticket → turnstile → resale** = **attestation** (paid access) + **group** (decrypt the stream) + **presentation** (the QR at the gate) + **transfer** (artist-governed resale) + **settlement** (the payment).
- **AI agent with bounded authority** = **grant** (scoped, rate-limited, one-time) + **auth** (mandatory-approve above a threshold) + actor type.
- **Verifiable-credential wallet** = **attestation** (issued to you) + **presentation** (shown to a verifier) + **sealed** (stored privately).
- **Anti-counterfeit product** = **attestation** (brand signs the serial) + **physical bridge** (the barcode) + **transfer** (ownership on resale).

If a new use-case can be phrased as *who may act · who says what about whom · how do I prove it · how do I hand it over · who can read it · what's the audit trail*, it is a composition of the eight — that is why the pantry stays small.

---

## Kind sub-bands (500–799 OWM-core)

`550 kv · 560 cast · 570 physical · 580 attestation · 590 presentation · 600 transfer · 610 ssh`.
The 500–549 block is the original sequential assignment; 800–899 is reserved; 900–999 is ack/error; 1000+ is vendor (`vnd.` prefix). Grandfathered ranges (1–399: chat, payments/escrow, factoring, KYC, VoIP) import verbatim before v0 freeze.

See the living registry: `../api/kinds.json` (normative), mirrored in
`../packages/owm-core/src/kinds.js`.
