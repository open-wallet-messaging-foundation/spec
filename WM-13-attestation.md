# WM-13-attestation — OWM-ATTEST: a signed claim about a subject

**Status:** DRAFT (2026-08-04). **Profile:** see WM-0. Rationale:
[FIRST_CLASS_PRIMITIVES](FIRST_CLASS_PRIMITIVES.md) §1.

**Scope:** the first-class **attestation** — an *issuer* signs a typed *claim*
about a *subject*, optionally anchored to a real-world identity (WM-9 §7) and
recorded in a transparency log. OWM-native Verifiable Credentials. Generalises
verified-sender (515), binding mutual-attestation (528), KYC, the SSH CA
(WM-16), and WM-11 entitlements — all become instances. Reference:
`packages/owm-core/src/attestation.js`. Kind `owm-attestation` (580).

## 1. The object

```
Attestation {
  v: 1, kind: "owm-attestation",
  issuer      : address — who is vouching
  subject     : address | subjectRef (WM-9 §5) — who/what it is about
  claimType   : registry key ("kyc-tier","degree","paid-access","ssh-principal","verified-sender",…)
  claim       : the claim body (typed by claimType)
  anchor      : optional domain / legal-entity the issuer key is bound to (WM-9 §7)
  iat, exp    : validity window (unix seconds)
  sig         : EIP-191 over the canonical (§2)
  attestationId : sha256(canonical)
  txlog       : optional CT-style inclusion proof (§5)
}
```

**Self-attestation** (issuer == subject) is the degenerate case we already ship: a
WM-9 binding is you attesting "this address controls this SMS." The distinctive
case is issuer ≠ subject — a bank attests your KYC tier, a CA attests your SSH
principal, a venue attests your paid access.

## 2. Canonical payload (normative)

EIP-191 `personal_sign`, disjoint domain, over exactly:

```
"owm-attestation-v1" \n issuer \n subject \n claimType \n sha256(claimBody) \n anchor \n iat \n exp
```

`issuer`/`subject` lower-cased (subject may be a `subjectRef` hash instead of an
address); `anchor` is the domain string or `""`; `claimBody` is the claim object
serialised canonically (sorted keys). Verification: `sig` MUST recover to `issuer`;
the attestation MUST be unexpired; a `verified`-tier consumer MUST additionally
validate `anchor` against the WM-9 §7 trust anchors (a network fetch, out of scope
for the pure verify).

## 3. Trust anchoring (normative, by reference)

An attestation is only as believable as the verifier's reason to trust the issuer's
key. WM-13 reuses WM-9 §7 verbatim: **domain tier** (issuer key proven to control
`anchor` via `.well-known` over TLS + DNSSEC/DANE — DV-equivalent) and
**legal-identity tier** (KYB binding `domain ↔ entity` — EV-equivalent). No new
root. Self-attestations carry no `anchor` and assert only "this key says so."

## 4. Claim registry (normative)

`claimType` is an open registry (like the kind registry); a verifier rejects an
unknown-but-critical claimType rather than guessing. Core types: `verified-sender`,
`kyc-tier`, `paid-access` (WM-11), `ssh-principal` (WM-16), `credential` (generic
VC body), `membership`, `provenance` (a serial ↔ manufacturer). Vendor types carry
a `vnd.` prefix.

## 5. Revocation & transparency (normative)

Short `exp` is the first defence. Beyond it: an issuer publishes revocations
(a signed revocation list, folded per WM-9 §8b) and — for high-assurance issuers —
records issuance in a **Certificate-Transparency-style append-only log** so a
mis-issued or compromised attestation is publicly detectable. A verifier MAY
require an inclusion proof (`txlog`) for `verified` tier.

## 6. Honest ceilings

An attestation asserts a claim; it does **not** make it true (garbage in, signed
garbage out). Its trust ceiling is the **issuer's key** — a compromised or coerced
issuer can sign falsely, mitigated (not eliminated) by transparency + short validity
+ revocation, exactly as the web lives with CA compromise. Low-entropy subject refs
are confirmable-by-hash (WM-9 §5). It proves **who said what**, never ground truth.

## 7. Composability

The subject is often a **binding** (WM-9); it is verified against **trust anchors**
(WM-9 §7); it is shown via a **Presentation** (WM-14); it can be handed on via a
**Transfer** (WM-15); it rides a QR (WM-12); a **WM-11 entitlement** *is* a
`paid-access` attestation; an **SSH certificate** (WM-16) *is* an `ssh-principal`
attestation projected into SSH's cert format. Verified-sender (515) and binding-
attest (528) become attestation instances.

## 8. Prior art

W3C Verifiable Credentials, X.509 + CA/Browser Forum DV/EV, Ethereum Attestation
Service (EAS), Sign Protocol, Certificate Transparency (RFC 6962), DKIM, DANE
(RFC 6698).

## 9. Proposed kinds

`580–589` attestation sub-band: `580 owm-attestation` (this), `581
owm-attestation-revoke` (reserved), `582–589` reserved.
