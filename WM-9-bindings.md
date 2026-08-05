# WM-9-bindings

**Status:** Draft (2026-08-03, content landed with the reference
implementation). **Profile:** see WM-0.

**Scope:** OWM-BIND (general) — a wallet address as a **switchboard**: bound to
*any* typed identifier (SMS, email, Telegram, X, WhatsApp, Kast, a bank account,
a card, another wallet, …), each binding carrying a **capability set**, a
**direction**, and a per-address **accept posture** (Everything / Secure /
Verified). Self-signed by the owner; optionally **counter-attested** by the
subject, whose key is **trust-anchored to a domain via the existing Web PKI /
DNS** (§7). Kinds `owm-binding` (527), `owm-binding-attest` (528). The
**institutional (bank) profile is WM-7 §10** — a special case of this model.

Reference implementation: `packages/owm-core/src/binding.js`. Registries:
`api/binding-registry.json`. Conformance vector: `api/vectors/bindings.json`.

## 1. Problem and design

Your address is the one handle every counterparty should use to reach you — a
bank, an exchange, a friend, an app, a card issuer. Today each reaches you
through a different, spoofable channel (SMS a scammer forges, an email you can't
verify). WM-9 makes the address a **control panel**: you decide *who is
connected*, *what each may do*, and *what you will accept* — and you can pull any
plug. Every binding is a signed object, so the routing surface is verifiable and
revocable, not a trusted server's opinion.

Design invariants (normative):

- **Extensible by construction.** `subject.type` and `capability` are open
  registries (§3–4); a new channel or payment rail is a registry entry, not a
  protocol change. Vendor entries carry a `vnd.` prefix.
- **Owner-sovereign.** A binding is signed by the owner's key; the routing
  surface for an address is *its bindings*, not a directory's claim.
- **PII-minimising.** Low-entropy subject values (phone, email, PAN, account
  reference) never appear in the clear in the signed payload — only a stable
  reference hash (§5); delivery credentials are encrypted (WM-6 / WM-8).
- **Trust reuses the web's roots** (§7): a verified counterparty is one that
  proves control of a domain via TLS + DNS — the CA hierarchy the world already
  trusts — with an optional legal-identity tier on top. No new global root to
  bootstrap.

## 2. The Binding object

```
Binding {
  v: 1, kind: "owm-binding",
  address     : owner address (a WM-1 sub-identity)
  subjectType : registry key (§3)
  subjectRef  : sha256(subjectType ‖ ":" ‖ normalise(subjectValue))   (§5)
  direction   : "sink" | "source"                                     (§3)
  capabilities: sorted set from the capability registry               (§4)
  accept      : "everything" | "secure" | "verified"                  (§6)
  exp         : unix seconds
  sig         : EIP-191 over the canonical payload (§8)
  bindingId   : sha256(canonical payload)
}
```

## 3. Subject-type registry & direction

`api/binding-registry.json` lists core types; each has a **class**:

- **`sink`** — an outbound reachability channel ("reach me here"): `sms`,
  `telegram`, `whatsapp`, `signal`, `kast`, `x`, `push`, `webhook`. The notify
  sink (WM-6).
- **`source`** — an inbound authority ("this identifier acts through my key"):
  `bank-account`, `card`, `domain`. A `source` binding with
  `approve:mandatory` means **movements are void without the owner's signature**
  (the WM-7 §10 institutional profile, generalised to cards and any account).
- **`both`** — `email` (a `sink`, and the `<addr>@owm.foundation` gateway of
  WM-8), `owm-address` (another wallet, for delegation / RBAC).

## 4. Capability registry

`notify` · `receive` (secure channels only) · `approve:optional` ·
`approve:mandatory` · `verify` (inbound must be signed by the subject) ·
`exclusive` (the ONLY legitimate channel for this relationship — the
channel-exclusivity commitment of WM-7 §10, the smishing-cover kill).
Capabilities are a sorted set in the canonical (§8), so order is immaterial to
the `bindingId`.

## 5. Subject value privacy (normative)

`subjectRef = sha256(subjectType ‖ ":" ‖ normalise(value))`; `normalise` lowers
case and trims (E.164 for phones). Only `subjectRef` is signed and published; the
plaintext stays with the owner, and any **delivery credential** (a Telegram
chat-id, a phone number) is an **encrypted** blob per WM-6 / WM-8, never in the
binding. Honest ceiling: a hash of a low-entropy value is matchable but
brute-forceable by a party that already processes the plaintext — it is at-rest
leak resistance, not anonymity against the matcher.

## 6. Accept posture — the one dial

A single per-address setting (overridable per binding), answering "what will you
accept?":

| Posture | Accepts | Maps to |
|---|---|---|
| **everything** | any inbound, signed or not | WM-4 `open` · notify band 0–3 |
| **secure** | only sender-signed messages | notify band 4–6 |
| **verified** | only **trust-anchored** senders (§7) | WM-4 kind 515 · band 7–9 |

Allow/block lists compose on top of the posture.

## 7. Trust anchors — what "verified" means (normative)

WM-9 does **not** invent a new global root; it **reuses the existing Web PKI**:

- **Domain tier (self-serve, DV-equivalent).** A counterparty's signing key is
  bound to a **domain** by publishing a signed record at
  `https://<domain>/.well-known/owm/sender.json`, fetched **over TLS** — so the
  CA chain the client already trusts vouches that this key controls `<domain>` —
  reinforced by a **DNSSEC/DANE** record. A `mutual` binding carries the
  counterparty's `owm-binding-attest` (528) over `bindingId ‖ domain ‖ ts`; the
  client verifies the signature **and** the domain proof. The badge shows the
  domain — the padlock users already understand.
- **Legal-identity tier (optional, EV-equivalent).** An **OWM verified-sender
  attestation** binds `domain ↔ legal entity` after KYB — the paid institutional
  tier. It upgrades the badge from a domain to a named entity ("ACME Bank Ltd").
- **Transparency.** Verified-sender attestations are recorded in a
  **Certificate-Transparency-style append-only log** — publicly auditable and
  revocable, so a mis-issued or compromised attestation is detectable.

Ceiling (normative, honest): domain control is not legal identity (that is the
legal-identity tier's job); a CA compromise or domain takeover is the trust
ceiling — the same one every TLS user already lives with, plus DNSSEC and CT as
defence-in-depth. WM-9 deliberately inherits the web's trust — and its known
edges — rather than asking the world to trust a new root.

## 8. Canonical binding payload (normative)

EIP-191 `personal_sign` (secp256k1, recovery-based, case-insensitive address
compare — shared with WM-3/WM-7) over exactly:

```
"owm-binding-v1" \n address \n subjectType \n subjectRef \n direction \n capabilities \n accept \n iat \n exp
```

`capabilities` is the sorted set comma-joined; `exp` decimal. The attestation
(528) signs `"owm-binding-attest-v1" \n bindingId \n domain \n ts`. Domain tags
are disjoint from every other OWM signing domain (WM-3/7/8), so a binding
signature can never be replayed as an auth, grant, approval, or contact card.

Verification: the `sig` MUST recover to `binding.address` (self-asserted), the
binding MUST be unexpired, and a `verified`-posture consumer MUST additionally
validate the counterparty attestation + its §7 domain proof.

## 8b. History — append-only and signed (normative)

Every binding change — **create, update** (rotate direction / capabilities /
accept / exp), **or revoke — is a NEW signed event**, never an in-place edit or
delete. The `iat` (issued-at) in the signed payload orders the log and gives
freshness. The **current** binding for a `(address, subjectType, subjectRef)` is
the **fold** of the log: the valid event with the greatest `iat`; **revoke wins**
over everything, including an unexpired binding (the WM-4 / WM-7 §4.4
lifecycle rule: revocation > TTL > single-use). A **revoke** is a signed
binding event with an empty `capabilities` set.

Events MAY chain (`prevId` = the prior event's `bindingId`) into a tamper-evident
hash chain. Because every event is owner-signed, the history is
**non-repudiable** (each state was authorized by a signature) and
**tamper-evident** (an unsigned or altered event is rejected); both the owner and
the counterparty retain the artifacts. Consumers **fold the log**; they never
trust a mutable "current" record from a server.

## 9. The API (bank-easy)

```
POST /owm/v1/message/{address}/{what}      # what ∈ ping | notify | auth-challenge | tx-approval | statement | …
GET  /owm/v1/addresses/{address}/bindings  # the public routing surface a sender needs
POST /owm/v1/addresses/{address}/bindings  # owner-signed create / rotate / revoke
```

`POST message/{address}/{what}` is the whole integration a **bank** needs: it
stores **one field — the customer's address** — plus its own (domain-anchored)
signing key. To get a transaction signed: `POST message/{addr}/tx-approval` (a
WM-7 530 challenge, sent as a verified sender). To notify:
`POST message/{addr}/notify`. The router resolves the address's bindings + accept
posture and delivers per capability — **the bank never learns the customer's
phone or email**; it only ever holds the address. That single-metadata-field
integration is the adoption wedge.

## 10. Delegation / role-based access

Roles are `owm-address` bindings to another party's wallet carrying a scoped
grant (WM-7 §4): **accountant** (`owm.statements.read`, read-only), **analyst**
(scoped site access as the bill payer, no money scope), **bill-payer**
(`owm.pay.approve` capped by amount/velocity). Signed, time-boxed,
least-privilege, instantly revocable — RBAC with no central directory.

## 11. Lifecycle & ceilings

Enroll / rotate (new key signed by old) / revoke / slow-recovery-with-veto follow
WM-7 §10 verbatim. Ceilings: a binding asserts *policy* — enforcement of
`approve:mandatory` still depends on the counterparty honouring it (a rogue
institution can move its own fiat ledger; OWM guarantees it can never produce an
authorization the owner did not sign, and the owner can prove absence — WM-7 §10).
`exclusive` removes legitimate cover for out-of-band contact; it cannot stop a
spoofed SMS from arriving. Subject-value hashes are leak-resistant, not anonymous
against the matcher (§5).

## 11b. Composable form (WM-13/14/15)

A binding is the **binding profile** of the foundational primitives: it is a
**self-attestation** (WM-13, issuer == subject — "this address controls this
subject") carrying a **grant** (WM-7) of capabilities, with its history an
**append-only log + fold** (§8b). The mutual attestation (kind 528) is an instance
of the general **Attestation** primitive (WM-13, kind 580), and the `verified`
posture consumes the same **trust anchors** (§7). Nothing changes on the wire; see
[PRIMITIVES](PRIMITIVES.md).

## 12. Prior-art anchors

Web PKI / X.509 + CA/Browser Forum DV/EV (the trust tiers reused), Certificate
Transparency RFC 6962 (the public attestation log), DANE RFC 6698 + DNSSEC (the
DNS anchor), DKIM RFC 6376 (domain-signed messaging), UCAN / ZCAP-LD / Biscuit
(capability delegation), OAuth scopes (RBAC), EIP-191 (the signing scheme).
