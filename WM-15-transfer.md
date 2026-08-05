# WM-15-transfer — OWM-XFER: hand a bound object to a new holder

**Status:** DRAFT (2026-08-04). **Profile:** see WM-0. Rationale:
[FIRST_CLASS_PRIMITIVES](FIRST_CLASS_PRIMITIVES.md) §3.

**Scope:** the first-class **transfer** — the current *holder* signs a bound object
over to a *new holder*, under the *issuer's policy*, with state advancing by the
same append-only log + fold + hash-chain used across OWM. Generalises WM-11 ticket
resale and the latent transfer in bindings, asset title, and licence reassignment.
Reference: `packages/owm-core/src/transfer.js`. Kind `owm-transfer` (600).

## 1. The object

```
Transfer {
  v: 1, kind: "owm-transfer",
  objectId   : id of the object changing hands (attestationId, bindingId, entitlementId, …)
  objectKind : "owm-attestation" | "owm-binding" | "owm-entitlement" | …
  from       : current holder      to: new holder
  terms      : optional { price?, royalty?, conditions? }
  prevId     : previous accepted transfer's id ("" for the first) — the chain / CAS link
  iat, exp   : window
  sig        : EIP-191 by `from` (authorises the hand-off)
  issuerSig  : optional issuer counter-signature, required where policy demands (§3)
  transferId : sha256(canonical)
}
```

## 2. Canonical payload (normative)

```
"owm-transfer-v1" \n objectId \n objectKind \n from \n to \n sha256(terms) \n prevId \n iat \n exp
```

`from`/`to` lower-cased; `terms` serialised canonically (sorted keys) or `""`.
Verification: `sig` MUST recover to `from`; unexpired; and where the object's issuer
policy requires it (§3), `issuerSig` MUST recover to the issuer.

## 3. Issuer policy (normative)

The object's issuer **Attestation** (WM-13) carries a `transfer` policy in its
claim body: `soulbound` (non-transferable — reject all), `open` (holder-signed
suffices), `mediated` (issuer counter-sign required — `issuerSig`), plus optional
`priceCap` and `royalty`. A transfer that violates policy is **not folded**. This
is what turns resale into a market the issuer governs (WM-11 anti-scalping).

## 4. Fold, CAS & provenance (normative)

Current holder = the `to` of the highest-`iat`, chain-valid, policy-valid transfer;
`prevId` MUST chain to the prior accepted transfer's `transferId` (CAS — a
double-transfer / fork is rejected, exactly like WM-10 writes). The accepted
transfer chain **is** the provenance / chain-of-custody, verifiable end to end.

## 5. Old-holder revocation

For a group-gated object (a WM-11 stream), accepting a transfer removes `from` from
the object's MLS group and admits `to`; **forward secrecy** ends the old holder's
access immediately. For a pure credential, the old holder simply no longer presents
as current holder (the fold says so).

## 6. Payment leg (open)

A paid transfer composes with the WM settlement core (543–545): the money leg and
the hand-off reference each other (`terms.settlementRef`) so a resale is atomic-ish
(both signatures, or an escrow). v0 may treat payment as out-of-band with a
referenced receipt; true atomic swap is a follow-on.

## 7. Honest ceilings

A transfer moves the **entitlement/attestation**, not necessarily the real-world
thing — a transferred car-title attestation doesn't move the car; the world must
still honour the claim. A completed transfer is **final** unless the issuer policy
allows reversal. Cross-party atomicity (money ↔ object) needs both signatures or an
escrow leg — a lone holder signature is a *gift*, not a *sale*.

## 8. Composability

Transfers an **Attestation** (WM-13), a **binding** (WM-9), or a **WM-11
entitlement**; the **issuer Attestation** sets the policy it must honour; state
moves via the **log + fold** (WM-9 §8b / WM-10 §6); payment via **settlement**;
old-holder access dies via **group** forward secrecy.

## 9. Prior art

ERC-721/1155 transfers + royalties (EIP-2981), SSH certificate reissuance, domain
transfer (EPP auth-info codes), Ticketmaster SafeTix transfer, UCAN delegation.

## 10. Proposed kinds

`600–609` transfer sub-band: `600 owm-transfer` (this), `601 owm-transfer-receipt`
(reserved, the settlement-linked confirmation), `602–609` reserved.
