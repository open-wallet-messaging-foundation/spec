# WM-14-presentation — OWM-PRESENT: prove a signed object to an audience

**Status:** DRAFT (2026-08-04). **Profile:** see WM-0. Rationale:
[FIRST_CLASS_PRIMITIVES](FIRST_CLASS_PRIMITIVES.md) §2.

**Scope:** the first-class **presentation** — a *holder* discloses, or proves
possession of, a signed object (an attestation, a binding, a KV record, a ticket)
to a specific *audience*, bound to context and a freshness *challenge*,
non-repudiably. This is WM-10 §10b REVEAL lifted out of KV because it belongs to
every object. Reference: `packages/owm-core/src/presentation.js`. Kind
`owm-presentation` (590).

## 1. The object

```
Presentation {
  v: 1, kind: "owm-presentation",
  holder     : address — who is presenting
  objectKind : what is being presented ("owm-attestation","owm-binding","owm-kv-record",…)
  objectHash : sha256 of the presented object (or its id) — the version pin
  object     : optional inline object (else the audience already holds it)
  audience   : verifier address | groupId | audienceHash — to whom
  challenge  : verifier-supplied nonce — freshness (defeats replay)
  fields     : disclosed field set (sorted) or "*"
  iat, exp   : short window
  sig        : EIP-191 by the holder over the canonical (§2)
  presentationId : sha256(canonical)
}
```

## 2. Canonical payload (normative)

```
"owm-presentation-v1" \n holder \n objectKind \n objectHash \n audience \n challenge \n fields \n iat \n exp
```

`holder`/`audience` lower-cased; `fields` sorted comma-joined or `*`. Verification:
`sig` MUST recover to `holder`; the presentation MUST be unexpired; the verifier
MUST check `challenge` equals the nonce it issued (freshness); and MUST separately
verify the presented object's own signature/attestation (a presentation proves
*possession + context*, not the object's validity).

## 3. Freshness & context binding — anti-replay (normative)

A presentation used to **gate access** MUST be fresh: the verifier issues a
`challenge` nonce and rejects any presentation not bound to it, so a captured or
screenshotted presentation cannot be replayed. `audience`/`groupId` binds it to
*this* verifier/room — replay elsewhere fails the match. Non-gating presentations
(a public "here is my contact card") MAY omit the challenge. This rule is what
makes the WM-12 turnstile safe.

## 4. Selective disclosure (normative + open)

`fields` names exactly what is revealed; a verifier learns nothing about undisclosed
fields beyond their existence. **Field-level predicate disclosure** — proving
"over-18" without revealing the birthdate — requires ZK / BBS+ signatures and is a
named future (WM-14a); v0 discloses whole named fields.

## 5. Relationship to WM-10 REVEAL

WM-10 §10b REVEAL is the **KV profile** of this primitive (it presents a KV record,
with the same challenge/context/version-pin semantics). WM-14 is the general form;
REVEAL converges onto it (a `presentation` with `objectKind:"owm-kv-record"`). The
committed `owm-kv-reveal` (554) stays valid; the convergence is a non-breaking
refactor.

## 6. Honest ceilings

Disclosure is **irrevocable and memory-bound** — you showed it; signatures stop
alteration and false attribution, never a screenshot or a memory. A whole-audience
presentation is visible to that whole audience by definition. Field-level ZK
disclosure is future, not v0. Presentation controls **whom you prove to and defeats
replay**; it cannot control what a legitimate viewer retains.

## 7. Composability

Presents an **Attestation** (WM-13), a **binding** (WM-9), a **KV record** (WM-10),
or a **WM-11 ticket**; its `challenge` is a **WM-7** auth nonce; its serialisation
into a scannable carrier *is* **WM-12**; an **SSH login** (WM-16) is a presentation
of a certificate.

## 8. Prior art

W3C Verifiable Presentations, ISO 18013-5 mDL, FIDO/WebAuthn assertions
(challenge-bound), SD-JWT selective disclosure, OAuth token presentation.

## 9. Proposed kinds

`590–599` presentation sub-band: `590 owm-presentation` (this), `591
owm-present-challenge` (a verifier's nonce request, reserved), `592–599` reserved.
