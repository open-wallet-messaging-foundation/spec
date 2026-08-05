# First-class primitives: Attestation · Presentation · Transfer

**Status:** Foundational exposition (2026-08-04). This is the *why* document; the
normative wire formats live in the per-primitive specs (proposed WM-13 / WM-14 /
WM-15). The catalogue of all primitives is [PRIMITIVES](PRIMITIVES.md).

## Why "first-class" is a decision worth making

OWM is built so that use-cases **compose** from a small pantry of orthogonal
primitives instead of each getting its own bespoke protocol. Some primitives are
already first-class — a signed statement, an append-only signed log you *fold* into
current state, a capability grant, a sealed-to-N-recipients payload, an MLS group.
When a new feature needs one, it reuses the canonical object; it doesn't reinvent
it.

Three more ideas kept showing up **reinvented per feature**, and that is the tell
that they should be promoted:

- We wrote "an issuer signs a claim about a subject" as verified-sender (515), as
  binding mutual-attestation (528), as KYC, and (the moment we looked at SSH) as a
  certificate authority. → **Attestation.**
- We built "prove a signed object to an audience, replay-proof" *inside* WM-10 as
  REVEAL — then immediately needed the exact same thing for the WM-12 turnstile,
  WM-11 tickets, credential verification, and proof-of-funds. → **Presentation.**
- We specified "hand a bound object from one holder to another under the issuer's
  policy" bespoke in WM-11 (ticket resale), with the same shape latent in binding
  transfer, asset title, and licence reassignment. → **Transfer.**

**The test we applied** (and that any future promotion should pass): (1) it already
appears in ≥3 features, reinvented; (2) it is *orthogonal* — it composes cleanly
with the others rather than overlapping them; (3) promoting it makes the specialised
specs **thinner**, not thicker. All three pass decisively.

**The payoff:** one canonical object per idea — one signing domain, one
verification path, one stated set of ceilings — inherited by every feature, and
handed *for free* to use-cases we haven't imagined. WM-9/10/11/12 stop being
separate pantries and become **recipes** over a shared one.

---

## 1. Attestation — *"someone signs a claim about a subject"*

**What it is.** An **issuer** signs a typed **claim** about a **subject**,
optionally anchored to a real-world identity and recorded in a transparency log:

```
Attestation {
  issuer   : address (who is vouching)
  subject  : address | subjectRef (who/what it is about)
  claim    : { type: registry key, body }   // "degree", "kyc-tier", "paid-access", "ssh-principal", …
  anchor   : optional domain/legal-entity proof (WM-9 §7 Web PKI / KYB)
  exp      : validity window
  sig      : EIP-191 over the canonical, disjoint domain
  txlog    : optional CT-style inclusion proof
}
```

The **self-attestation** case — issuer == subject — is the degenerate one we
already ship: a WM-9 binding is you attesting "this address controls this SMS."
The powerful case is issuer ≠ subject: `university.edu` attests your degree, a
bank attests your KYC tier, a CA attests your SSH principal, a broadcaster attests
your paid access.

**Why it must be first-class.** It is the single most reinvented shape in OWM —
verified-sender (515), binding-attest (528), KYC (200–202), the SSH CA, and WM-11
entitlements are all the same object wearing different clothes. Unifying them means
*one* trust-anchoring story, *one* revocation-and-transparency story, and a verifier
that learns one verification path instead of five.

**Why we need it.** Attestation is OWM-native **Verifiable Credentials** — the
universal "X vouches for Y." It unlocks, from one primitive: credentials (degree,
licence, membership), KYC/AML tiers, reputation and reviews, verified-sender/brand
trust, the **SSH certificate authority**, paid-access entitlements (WM-11), and
product provenance (a brand attests a serial). Anything of the form "someone
authoritative says something about you or your asset" is this.

**What's important.**
- **Issuer/subject separation** — the whole value is that a *third party* vouches;
  self-attestation is just the special case.
- **Trust-anchoring** (WM-9 §7) — a verifier believes an attestation because it can
  tie the issuer's key to a domain (TLS + DNSSEC) or a legal entity (KYB), reusing
  the web's roots rather than a new one.
- **Typed claims** — `claim.type` is a registry, so a verifier knows what it is
  reading and can reject unknown-but-critical claims (strict validation).
- **Revocability + transparency** — short `exp`, a revocation path, and a CT-style
  log so a mis-issued or compromised attestation is *detectable*, not silent.
- **Domain separation** — an attestation signature can never be replayed as an
  auth, grant, presentation, or transfer.

**Honest ceilings.** An attestation asserts a claim; it does **not** make the claim
true — garbage in, signed garbage out. Its trust ceiling is the *issuer's* key: a
compromised or coerced issuer can sign falsely (mitigated by transparency + short
validity + revocation, exactly as the web lives with CA compromise). Low-entropy
subject refs are confirmable-by-hash (WM-9 §5). It proves **who said what**, never
ground truth.

**How it composes.** The subject is often a **binding**; it is *verified* against
**trust anchors**; it is shown via a **Presentation**; it can be handed on via a
**Transfer**; carried in a QR (WM-12); and an **SSH certificate is simply an
attestation projected into SSH's cert format.**

**Prior art.** W3C Verifiable Credentials, X.509 + the CA/Browser Forum, Ethereum
Attestation Service (EAS), Sign Protocol, Certificate Transparency (RFC 6962),
DKIM.

---

## 2. Presentation — *"prove or disclose a signed object to an audience"*

**What it is.** A **holder** discloses (or proves possession of) a signed object —
a record, an attestation, a binding, a ticket — to a specific **audience**, bound
to context and a freshness **challenge**, non-repudiably:

```
Presentation {
  object   : the signed object (or a reference + version pin)
  holder   : address (who is presenting)
  audience : verifier / groupId / audienceHash (to whom)
  challenge: verifier nonce (freshness — defeats replay)
  fields   : which fields are disclosed (selective disclosure)
  exp      : short
  sig      : EIP-191 by the holder, over all of the above
}
```

This is today's WM-10 §10b REVEAL, lifted out of the KV feature because it belongs
to *every* object, not just records.

**Why it must be first-class.** We built it once and immediately needed it four more
times: the WM-12 turnstile, WM-11 ticket admission, credential verification (the VP
in VC→VP), and proof-of-funds. Leaving it inside WM-10 would force every one of
those to re-derive the anti-replay and context-binding rules — the exact rules that
are easy to get subtly, dangerously wrong.

**Why we need it.** It is the **"show it / prove it"** verb. It unlocks turnstiles
and access gates, credential verification, age-gates, proof-of-funds/solvency,
mixed-trust group disclosure, mobile-driving-licence-style identity checks, and any
"scan to verify" flow. Where Attestation is *being* vouched-for, Presentation is
*proving it to someone right now*.

**What's important.**
- **Holder ≠ issuer** — you present an attestation *someone else* made about you;
  the holder's signature proves possession, the issuer's proves the claim.
- **Context-binding** — `audience`/`groupId` means a presentation captured in one
  place can't be replayed in another.
- **Freshness challenge** — a verifier-supplied nonce makes a *screenshot of
  yesterday's proof* fail. This is the anti-replay backbone and is **mandatory for
  anything that gates access**.
- **Version pin** — presenting pins the exact version shown, so later changes don't
  retroactively alter what was proven.
- **Selective disclosure** — which fields are revealed; the privacy frontier
  ("over-18" without the birthdate) is the ZK/BBS+ upgrade path.
- **Non-repudiation** — holder-signed and (in a group) logged, so "they presented X
  here at this time" is provable.

**Honest ceilings.** Disclosure is **irrevocable and memory-bound** — you showed it;
signatures stop alteration and false attribution, never a screenshot or a memory.
Field-level selective disclosure needs ZK/BBS+ (a named future). A whole-audience
presentation is, by definition, visible to that whole audience. Presentation
controls **whom you prove to and defeats replay** — it cannot control what a
legitimate viewer remembers.

**How it composes.** It presents an **Attestation**, a KV record, or a **binding**;
its serialization into a QR *is* WM-12; its challenge comes from **WM-7 auth**; an
SSH login is arguably a presentation of a certificate.

**Prior art.** W3C Verifiable Presentations, ISO 18013-5 mDL, FIDO/WebAuthn
assertions (challenge-bound), SD-JWT, OAuth token presentation.

---

## 3. Transfer — *"hand a bound object to a new holder under issuer policy"*

**What it is.** The current **holder** signs the object over to a **new holder**;
the **issuer's policy** governs whether and how, and state advances by the same
log+fold + hash-chain we use everywhere:

```
Transfer {
  object   : reference to the bound object (ticket, binding, asset, licence)
  from     : current holder      to: new holder
  terms    : optional price / royalty / conditions
  prevId   : chain link (CAS — rejects double-transfer)
  exp      : short
  sig(from): the holder authorises the hand-off
  issuerSig: optional counter-signature, required where policy demands it
}
```

**Why it must be first-class.** It is bespoke in WM-11 (ticket resale) and latent
everywhere something is *owned*: binding transfer, asset title, licence
reassignment, handle/domain transfer. Same shape, same hazards (double-spend,
policy evasion) — worth solving once.

**Why we need it.** It is the **"sell / give / reassign"** verb. It unlocks
artist-controlled ticket resale (the anti-scalping story — cap price, take a
royalty, or forbid), NFT-style asset transfer, software-licence reassignment,
membership hand-off, and product-ownership change on resale. Ownership that can't
change hands safely isn't ownership.

**What's important.**
- **Issuer-policy binding** — the issuer's Attestation sets the rules
  (transferable? price-capped? royalty? soulbound?), and a Transfer that violates
  them is rejected. This is what turns "resale" into a *market the issuer governs*.
- **Atomicity & ordering** — `prevId`/CAS + fold reject a double-transfer or a
  fork, exactly like KV writes.
- **Revocation of the old holder** — for group-gated things (a stream), the old
  holder is removed and MLS forward secrecy ends their access instantly.
- **Provenance chain** — the transfer history *is* the chain of custody, verifiable
  end to end.
- **Payment leg** — a paid transfer composes with settlement (the 5-method core) so
  money and hand-off are one flow.

**Honest ceilings.** A transfer moves the **entitlement/attestation**, not
necessarily the underlying real-world thing — handing over a signed car-title
attestation doesn't move the car; the world still has to honour the claim. A
completed transfer is **final** unless the issuer's policy allows reversal. True
cross-party atomicity (money ↔ object) needs both signatures or an escrow leg.

**How it composes.** It transfers an **Attestation** or a **binding** or a WM-11
entitlement; the **issuer's Attestation** sets the policy it must honour; state
moves via **log+fold**; the payment via **settlement**; old-holder access dies via
**group** forward secrecy.

**Prior art.** ERC-721/1155 transfers + royalties (EIP-2981), SSH certificate
reissuance, domain transfer (EPP auth-info codes), Ticketmaster SafeTix transfer,
UCAN delegation.

---

## How the three lock together

They are not three unrelated additions — they are the missing joints of one motion.
The concert-to-turnstile arc uses all three plus the primitives we already have:

> The broadcaster **attests** your paid access → you **present** it at the gate
> (challenge-bound QR, replay-proof) → later you **transfer** it under the artist's
> policy → the new holder **presents** it. Access itself is a **group**; the money
> is **settlement**; the audit trail is **log+fold**.

And SSH is the same shape in a different suit: the CA **attests** your principal (a
certificate) → your client **presents** it at login → the grant sets which hosts and
for how long.

**Why this maximises the use-cases we haven't thought of.** Almost any protocol
interaction decomposes into six questions: *who may act* (**grant**), *who said what
about whom* (**attestation**), *how do I prove it to you* (**presentation**), *how do
I hand it over* (**transfer**), *who can read it* (**seal / group**), and *what's the
audit trail* (**log + fold**). Make those six orthogonal and clean, and the answer to
"can OWM do «some X we never listed»?" is usually "yes — it's a composition." That is
the entire reason to build the standard this way.

---

## What promoting them changes (practical)

- **New normative specs** (proposed): **WM-13 Attestation**, **WM-14 Presentation**,
  **WM-15 Transfer** — each defines the canonical payload, verification, and
  ceilings sketched above.
- **Refactors toward thin profiles:** WM-9 binding = a *self-Attestation* + a Grant;
  WM-10 REVEAL becomes a *use of* Presentation; WM-11 entitlement = Attestation +
  Transfer + Group; WM-12 QR = a Presentation carrier; verified-sender (515) and
  binding-attest (528) become Attestation instances; the SSH certificate is an
  Attestation.
- **Registry:** reserve themed sub-bands for the three (proposal, not yet written to
  the live registry): e.g. `580–589 attestation`, `590–599 presentation`,
  `600–609 transfer`, continuing the 500–799 sub-band scheme.
- **Companion enrichment (separate, already first-class):** the **Grant** gains a
  caveat/chain grammar (rate-limits, amount caps, one-time, re-delegation) — not a
  new primitive, but the enhancement that lets agents and delegated authority reuse
  the same object. Tracked alongside, not inside, this promotion.
