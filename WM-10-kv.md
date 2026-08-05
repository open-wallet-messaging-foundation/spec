# WM-10-kv — OWM-KV: the wallet-native personal record store

**Status:** DRAFT FOR REVIEW (2026-08-03). Nothing here is frozen; sections
marked **(open)** are decisions we still need to make. **Profile:** see WM-0.

**Scope:** a wallet address as a **personal, encrypted, delegable data store** —
signed key→value records the owner (or a delegate) can `SET`, `GET`, `DELETE`,
`LIST`, with an **independent permission per verb**, over channels that are
**untrusted for confidentiality and integrity** because every record is signed
and (by default) encrypted. The address's own XMTP/MLS inbox is the primary store
(WM-1); external content-addressed storage is an optional tier for oversized or
archival values. Reuses WM-7 grants (532–534) for delegation and WM-8 sealing
(524) for encryption. Proposed kinds `owm-kv-*` (550–559, see §16).

Reference implementation: `packages/owm-core/src/kv.js` (v0 core — signing,
per-verb authorisation, and the fold). Conformance vector: `api/vectors/kv.json`.
Example identities: `api/test-identities.json`.

---

## 0. v0 scope — decided (2026-08-04)

Keep v0 **simple and correct**; everything richer is a named follow-on we may add
later. The sections below still describe the full vision, but only these ship in
v0 (deferred sections are tagged **(post-v0)** at their heading):

**In v0 (the tight core):**
- Address-is-the-store — self-scoped MLS conversation (§3), pending the self-group
  conform check on `dev`.
- Four storage verbs — `SET / GET / DELETE / LIST` (§4).
- `REVEAL` **of your own records, whole-group mode only** (§10b) — the group-facing
  payoff, minus the complexity. No subset-sealing, and because you can only reveal
  records you own, **no `redisclose` caveat is needed**.
- Per-verb capabilities — flat, owner-issued, time-boxed grants over
  `{set,get,delete,list,reveal} × namespace` (§5). No chains, no rich caveats.
- Non-tamper — per-op signature + `prevId` hash-chain + authorised-author fold (§6).
- Encryption — base tier only (per-namespace self/MLS seal, §7).
- Versioning / audit — free from the append-only log (§12).
- Concurrency — single-writer-per-record via CAS/`prevId` (§11).
- Storage — live tier + oversized WM-8 pointer (§14).
- Kinds `550–555` (§16) + the 500–799 rebanding.

**Deferred (revisit later, not committed):** granular per-key encryption &
fine read-splitting (§7), `REVEAL` subset-seal + audience privacy (§10b mode 2),
delegation chains / attenuation / caveats (§5), reactive subscriptions (§10 →
WM-10a), schemas / typed namespaces (§9), inheritance / estate release (§13),
multi-writer CRDT merge (§11), public unencrypted records (§8 — the cheap first
follow-on: an ENS-text replacement).

---

## 1. What this actually is (and isn't)

The one-line demo is `SET firstname=saxon`, `GET firstname → saxon`. But the
primitive underneath is bigger, and naming it honestly is the point of this
draft: **a wallet-native, self-sovereign data store** — the thing Solid PODs,
Ceramic/ComposeDB, and Spruce Kepler are, but built from the primitives OWM
already ships (one wallet, sign-to-derive, domain-tagged signing, the grant
model, the fold-the-log pattern) and living in the inbox the address already has.

From that one primitive you get, without new machinery:

- **A portable profile** any app can read with your grant (`profile.email`).
- **A personal secret vault** (seed backups, API keys, recovery kits) that your
  mother or accountant can be granted a *slice* of.
- **A signed, versioned audit trail** — every value's full history is retained
  and provable ("what was my mailing address on 2024-01-01?", "what consent did I
  hold when?").
- **A live, reactive store** — because it rides a message stream, a grantee can
  *subscribe* to a key and be pushed the new value on `SET` (§10). This is the
  capability the IPFS design can't offer.
- **Social data recovery and inheritance** — a delegate can retrieve your store
  if you lose your wallet; an `estate` namespace can be released to heirs on
  proven inactivity (§13).

What it is **not**: a durability guarantee (that's a storage-tier choice, §14), a
public database (records are encrypted by default), or a way to *un-share* what a
delegate already read (revocation is forward-only, §17).

---

## 2. The Record model

```
Record (logical) {
  owner       : address (a WM-1 sub-identity)
  namespace   : string   — a permission + schema boundary ("profile","financial",…)
  key         : string   — the record name within the namespace
  value       : bytes    — encrypted by default (§7); or public+signed (§8)
  visibility  : "encrypted" | "public"
  schemaRef   : optional — the namespace's type (§9)
  version log : the append-only signed op log; current value = fold (§6)
}
```

A record is never edited in place. Its state is the **fold** of an append-only
log of signed **ops** (§4). `recordId = sha256("owm-kv-v1" ‖ owner ‖ namespace ‖
key)` names the logical slot; low-entropy keys are confirmable-by-hash, same
ceiling as WM-9 §5.

---

## 3. The address IS the store (normative)

The primary store for an address is a **self-scoped MLS conversation** — a
"notes-to-self" group the owner controls (WM-1). A `SET` publishes a signed op
into it; `GET`/`LIST` are a **local fold of the owner's already-synced inbox** —
no server round-trip in the base case. Consequences:

- **Encryption for the self-case is MLS** — the op is sealed to the owner's
  installations by the transport; the WM-8 envelope (§7) is only needed when a
  value must leave the group (a delegate, public storage).
- **The log is native** — an MLS conversation is already an ordered, append-only,
  replicated log. The fold pattern (WM-9 §8b) applies verbatim.
- **Delegation is membership** — adding a delegate to a namespace's group gives
  them the decryptable stream; removing them is MLS member-removal, whose
  **forward secrecy is the revocation** (§17).
- **Recovery is "connect any client"** — the store re-syncs from the network to
  any installation the wallet authorises.

**(open)** Self-conform check: this depends on creating a single-member / self
group on the target XMTP MLS build. To verify on the `dev` network before we
freeze §3; fallback is a reserved 2-member keep-alive group or a dedicated topic.

---

## 4. The four verbs (normative)

Each verb is a domain-tagged EIP-191 message (secp256k1, recovery-based,
case-insensitive address compare — shared with WM-3/7/8/9). Domains are disjoint
from every other OWM signing domain, so a KV op can never be replayed as an auth,
grant, approval, binding, or contact card.

**Writes** carry the value hash (binding the signature to exact ciphertext) and a
`prevId` (the previous accepted op's id in this record's chain, `""` for the
first — §6 hash-chain):

```
SET     "owm-kv-set-v1"  \n owner \n namespace \n key \n valueHash \n prevId \n iat \n exp
DELETE  "owm-kv-del-v1"  \n owner \n namespace \n key \n prevId \n iat
```

**Reads** are signed *requests* (they matter when a delegate asks a node, or a
gated store enforces retrieval; the owner's own base-case read needs no message).
A `nonce` makes them single-use and unreplayable:

```
GET     "owm-kv-get-v1"  \n owner \n namespace \n key \n nonce \n iat
LIST    "owm-kv-list-v1" \n owner \n namespace \n nonce \n iat
```

A `DELETE` is a signed **tombstone**, not an erasure. Ops are self-describing
objects `{ v:1, kind, owner, namespace, key?, valueHash?, prevId?, nonce?, iat,
exp?, sig }`; strict envelope validation (missing/extra/type-mismatched → reject)
per the house rule.

---

## 5. Per-verb capabilities & delegation (normative — the headline)

Permission is **per verb, per namespace** (optionally per key-pattern). A grant
authorises a `grantee` for a subset of `{set, get, delete, list, reveal}` on a
namespace, time-boxed and revocable. `reveal` (§10b) is deliberately distinct from
`get`: reading a value privately is not the authority to disclose it to others.
This is a WM-7 grant (kind 533) whose scope is:

```
owm.kv.<namespace>.<verb>            e.g. owm.kv.financial.get
owm.kv.<namespace>.*                 all verbs on a namespace
owm.kv.<namespace>.<verb>?key=<glob> attenuated to matching keys   (open, §caveats)
```

Worked roles (exactly your examples):

| Delegate | Grant | Can | Cannot |
|---|---|---|---|
| **Mother** (recovery) | `owm.kv.recovery.{get,list}` | read/enumerate the recovery namespace | write or delete anything |
| **Accountant** | `owm.kv.financial.{get,list}` | read financial records | touch `personal`, or write |
| **Bill-payer app** | `owm.kv.payees.{set,get}` | add/read payees | **delete** payees, read `personal` |
| **Co-owner** | `owm.kv.*.*` | everything | — |

**Attenuation & chains (open).** UCAN/ZCAP allow a delegate to *re-delegate* a
**narrower** capability (mother → her lawyer, `get` only, shorter expiry, never
wider). Do we allow delegation chains? If yes, the fold must verify the whole
chain roots in the owner and only narrows at each hop. **Caveats** (key-glob,
rate-limit N reads/day, app-bound, one-time, and `redisclose` — may a read be
re-presented to others, §10b) ride the grant. Decision needed: chains yes/no, and
which caveats are in v0 vs deferred.

---

## 6. Non-tamper: what "tamper-proof" means here (normative)

Three independent layers, none trusting a server:

1. **Per-op signature.** Every op is signed over its canonical (§4), which
   includes `valueHash`. Alter the value, the key, the namespace, or any field
   and the recovered signer no longer matches — the op is rejected.
2. **Hash-chain.** Within a record, each write's `prevId` = the previous accepted
   op's `opId` (`opId = sha256(canonical)`). A dropped, reordered, or inserted op
   breaks the chain and is **detectable** — tamper-*evident* ordering, the WM-9
   §8b pattern strengthened into an explicit chain.
3. **Authorised-author fold.** The fold accepts a `SET`/`DELETE` only if its
   signer is the owner **or** a grantee holding a matching verb-cap valid at the
   op's `iat`. An unauthorised write is **not folded into state** — every party
   who replays the log reaches the same current value, so a delegate cannot forge
   state they weren't granted. **This is why write-permissions are enforceable
   without a trusted server.**

Current value of a record = the highest-`iat`, chain-valid, authorised op;
**a tombstone wins** over a `SET` at equal-or-greater `iat` (the WM-4 / WM-7 §4.4
lifecycle rule: revocation > TTL > single-use). MLS adds transport integrity,
forward secrecy, and post-compromise security beneath all of this.

---

## 7. Encryption & read-granularity tiers (normative + open)

Confidentiality is layered, and **read-permission is only as fine as the
encryption granularity** — the honest crux of GET/LIST splitting:

- **Base tier — one group per namespace.** Membership = decrypt everything in the
  namespace. Cheap. **Enforceable:** `set`/`delete` (via §6) and *whether you
  receive the stream at all*. **Not enforceable:** a `get`-only member can still
  decrypt values they receive; "LIST keys but hide values" is policy, not crypto.
- **Granular tier (optional) — per-record envelope encryption.** A random content
  key (CEK) encrypts each value once (WM-8 AEAD); the CEK is **wrapped** (sealed,
  WM-8 §recipient) to the owner's identity key (524) and to each authorised
  reader's key. Now `get` is per-key grantable, `list` (key names) separates from
  `get` (values), and adding a reader later = one extra wrapped-CEK entry, **no
  re-encryption of the payload** (age/GPG multi-recipient style). Revoke a reader
  = drop their wrap and **rotate the CEK on the next write** (they keep prior
  copies — §17).

**Key rotation (open).** Sign-to-derive ties the identity key to the wallet key;
rotating the wallet (WM-7 §10) changes the derived key. Old records were sealed to
the old key. Options: (a) re-wrap CEKs to the new key at rotation (granular tier
makes this a metadata rewrite, not a re-encrypt); (b) keep a signed key-history so
old keys stay available for decrypt-only. Needs a decision; leaning (a)+(b).

---

## 8. Public vs encrypted records (open)

Not everything should be secret. A `visibility:"public"` record is **signed but
not encrypted** — a portable, gasless, self-hosted **ENS-text-record replacement**
(`profile.avatar`, a verification badge, a public key announcement). Same op log,
same signatures, same fold; just no sealing. Decision: is `public` in v0, or a
follow-on? (It's cheap and it makes OWM-KV a credible ENS-text alternative, which
is a strong adoption story.)

---

## 9. Schemas / typed namespaces — interop (open)

A namespace MAY declare a **schema** (`schemaRef`) so any app knows how to read
it — Ceramic/ComposeDB's model idea. `profile` → `{firstName,lastName,email,…}`;
`credentials` → W3C Verifiable Credentials. Typed, published schemas are what turn
a private KV into an **interoperable** data layer apps can build against without
bilateral agreements. Decision: ship a small **core schema registry** (profile,
contact, credentials) in v0, or leave schemas free-form and standardise later?

---

## 10. Reactive reads — the live-store superpower (open)

Because the store is a message stream, a grantee can **subscribe** to a
`(namespace,key)` and receive the new value **pushed** on `SET`, gated by their
`get` cap (this is exactly WM-6 notify over the KV stream). "Update `profile.email`
once, every app you've granted `profile.get` re-syncs." No polling, no webhooks to
register per app. Decision: specify subscription semantics (delivery, backfill,
unsubscribe) in v0 or as WM-10a.

---

## 10b. Disclosure into a conversation — REVEAL & mixed-trust groups (open)

Your scenario — *a group with trusted and untrusted parties, where a speaker pulls
a key and shows the content to the group* — is a **presentation**, not a capability
grant, and that distinction is the whole design. Holding a record privately (the
store) vs disclosing a value into an audience (a reveal) is exactly the
**Verifiable Credential vs Verifiable Presentation** split (W3C VC): you don't add
the audience to your namespace group to show them one value once — that would give
them ongoing decryptable access — you present a signed, contextual, one-shot
snapshot.

**`REVEAL` — a fifth verb.** It presents the current (or as-of) value of one record
into a conversation, signed and bound to that context:

```
REVEAL "owm-kv-reveal-v1" \n owner \n namespace \n key \n valueHash \n opId \n groupId \n audienceHash \n challenge \n iat
```

- `opId` pins the exact version disclosed — a snapshot, not a live subscription;
  changing the value later doesn't retroactively alter what was shown.
- `groupId` + `audienceHash` bind the presentation to *this* conversation and
  audience; replayed into another group it fails to match (replay-resistant).
- `challenge` is an optional verifier-supplied nonce proving **freshness** (this is
  the *current* value now, not a replayed old presentation) — the VC
  challenge/domain mechanic; use it when the reveal proves a live fact
  (proof-of-funds, "still employed").
- `reveal` is a **distinct capability** from `get` (§5): reading privately is not
  authority to broadcast; least-privilege splits them.

**Trust in a mixed group comes from crypto, not the social graph.** Every member,
trusted or not, verifies a reveal the same way: the signature recovers to `owner`
(really their record, unaltered), and if the record is itself an issuer-signed
credential, the issuer's key checks against the WM-9 §7 trust anchors (Web PKI /
KYB). "Alice's degree" is believable because `university.edu` signed it and Alice
presented it — **not** because anyone trusts Alice or the group. An untrusted party
can verify but cannot forge, re-attribute, or silently alter it. That is how
untrusted parties fit: they need not be trusted in order to *verify*.

**Two disclosure modes** (an MLS group encrypts to *all* members):

1. **Reveal to the whole group.** Post the signed value; MLS shows it to every
   member, including the untrusted ones — the intent. They verify; they also,
   unavoidably, *see* it.
2. **Reveal to a trusted subset.** Seal the value's content key to that `audience`
   subset's identity keys (524) and post the wrapped blob into the group. Untrusted
   non-audience members see *that* a disclosure happened, to whom, and when — but
   **not the content**. Metadata leaks; the value does not.

**Onward-disclosure control (open).** Revealing a record you *own* is your call.
Revealing one you were only *granted read on* (Bob's data) is re-disclosure — by
default **denied**; it requires a `redisclose` caveat on Bob's grant (§5). A
`get`-without-`reveal` grant means "read this, don't broadcast it to third
parties."

**Non-repudiation upside.** A signed reveal in the group's append-only MLS log is a
**provable, timestamped disclosure** — the presenter can't later deny it, and the
room can prove it happened. Valuable for deal rooms, KYC presentations, agreements.

## 11. Concurrency & multi-writer (open)

`prevId` gives **compare-and-set**: a writer sets `prevId = current head`; two
racing writes to the same head → the second is rejected (stale), preventing lost
updates when you *and* a bill-payer app both write `payees`. For genuinely
concurrent multi-writer namespaces (a family's shared records, a company config),
the fold needs a **merge rule** for sibling ops — CRDT territory (Automerge). v0
proposal: **single logical writer per record via CAS**; concurrent-merge
namespaces deferred to a WM-10b with an explicit CRDT profile. Confirm.

---

## 12. Versioning, point-in-time, audit (normative)

Append-only ⇒ history is free. `fold(log, {asOf: t})` returns the value as of any
past `iat`. Every historical value is owner- (or grantee-) signed, so the trail is
**non-repudiable and independently verifiable** — consent records, address
history, config changes, all provable after the fact. Consumers fold the log; they
never trust a mutable "current" record from a server.

---

## 13. Recovery, inheritance, dead-man's-switch (open — high value)

Your own phrasing — *"get it back… someone they've delegated to, e.g. their
mother"* — is **social recovery of data**, distinct from wallet-key recovery:

- **Delegated recovery.** A `recovery` namespace holds your encrypted seed/kit;
  your mother's `owm.kv.recovery.get` grant lets her retrieve it if you lose the
  wallet. Composes with WM-7 §10 slow-recovery-with-veto.
- **Inheritance / estate release (open).** An `estate` namespace released to heirs
  on **proven inactivity** — tie to WM liveness-checkin (539) and duress (538):
  no check-in for N days → heirs' `get` grants activate. A revocable, self-custodial
  **crypto will**. This is a genuinely novel, high-value use-case worth designing
  carefully (and stating its ceilings loudly: liveness oracle trust, forward-only).

---

## 14. Storage tiers & permanence (normative + open)

Address-is-the-store is the **live** tier; it is **not** archival. Tiers:

| Tier | Backing | Durability | Use |
|---|---|---|---|
| **Live (default)** | XMTP/MLS inbox | = network retention + your installations | small structured records, secrets, profile |
| **Archival (opt)** | IPFS/Filecoin/Arweave | pinned / pay-once-permanent | long-term, compliance |
| **Oversized** | any content store; KV holds a **WM-8 pointer** | backing-dependent | files, media, scans |

A value over the message-size cap is stored externally and the KV *value* becomes
a signed **pointer** (CID/URL + content hash + unwrap key). **(open)** Do we set a
recommended inline size ceiling, and is a self-hosted archival pin part of the
standard or a deployment concern? Metering/treasury: a `10c/GB`-style opt-in
donation on archival writes (mirrors WM-8's model)?

---

## 15. The API (app-easy)

```
GET    /owm/v1/kv/{owner}/{namespace}/{key}     # signed GET; returns record or 404
PUT    /owm/v1/kv/{owner}/{namespace}/{key}     # signed SET (owner or set-granted)
DELETE /owm/v1/kv/{owner}/{namespace}/{key}     # signed tombstone
GET    /owm/v1/kv/{owner}/{namespace}           # signed LIST
POST   /owm/v1/kv/{owner}/subscribe             # reactive read (§10)
```

An app integrates by holding **one field — the owner's address** — plus a grant
(WM-9 §9 wedge). The router checks the op signature + the grant's verb-cap +
posture, then serves or writes. It never holds plaintext it wasn't sealed into.

---

## 16. Kind reservation & the 500–799 rebanding (proposed)

Rebanding (your call — "go wider"): **OWM-core = 500–799** (300 slots). 500–549
stays the original sequential block; **550+ opens as themed sub-bands** so related
primitives cluster; 800–899 is a reserved buffer before 900–999 ack/error.

```
550  owm-kv-set
551  owm-kv-get
552  owm-kv-delete
553  owm-kv-list
554  owm-kv-reveal       # present a record into a conversation (§10b)
555  owm-kv-pointer      # signed external-storage pointer (§14)
556  owm-kv-subscribe    # reactive read (§10)   (open — only if §10 lands in v0)
557  owm-kv-schema       # namespace schema decl (§9) (open)
558–559  reserved (kv)
```

Delegation reuses grants (532–534) with `owm.kv.*` scopes — **no new grant kind**.
Sub-band sketch for later: 560–569 storage/data, 570–599 reserved, 600–699 /
700–799 future core (allocated on demand, not pre-committed).

---

## 17. Honest ceilings (consolidated — house rule)

- **Revocation is forward-only.** A revoked reader keeps values they already
  decrypted; revoke = no *future* reads (rotate CEK on next write). Every
  share-then-revoke system (Lit, Lighthouse, GPG) shares this.
- **Durability ≠ permanence.** Live tier lasts as long as XMTP retention + your
  installations; permanence is an explicit archival-tier choice.
- **DELETE is a tombstone, not erasure.** On replicated/append-only backing you
  make a record undecryptable-forward and unfindable; an archived ciphertext held
  by an ex-reader is residual.
- **Disclosure is irrevocable & memory-bound.** A REVEAL (§10b) to an untrusted
  party cannot be taken back — signatures stop alteration and false attribution,
  never a screenshot or a memory. A whole-group reveal is visible to every member,
  trusted or not, by definition. Field-level selective disclosure ("over-18"
  without the birthdate) needs BBS+/ZK signatures — future (WM-10c), not v0.
- **Read-split is bounded by encryption granularity** (§7): base tier's `get` vs
  `list` split is policy, not crypto, until the granular tier.
- **Low-entropy keys are confirmable by hash** (§2, WM-9 §5) — at-rest leak
  resistance, not anonymity against a party that knows the plaintext key.
- **The store is trusted for availability & non-censorship**, never for
  confidentiality or integrity (signatures + encryption handle those). Access
  patterns (who reads what, when) leak to the store unless mixed.
- **Recovery/inheritance trust an inactivity oracle** (§13) — state the liveness
  assumption explicitly wherever release is automated.

---

## 18. Open questions for our review

**Resolved 2026-08-04 (see §0):** keep v0 simple — items 1, 2 (granular tier), 3,
5, 6, 7, and 7b are **deferred** post-v0. Remaining live v0 questions: the §3
self-group conform check on `dev`, and naming (9). The rest below are recorded for
when we pick the follow-ons back up.

1. **Delegation chains** — allow attenuated re-delegation (UCAN-style), or
   owner-only grants in v0? Which **caveats** (key-glob, rate-limit, app-bound,
   one-time) are v0?
2. **Read granularity** — ship both tiers (§7), or base tier in v0 with granular
   as a profile? Key-rotation strategy (re-wrap vs key-history vs both)?
3. **Public records** (§8) in v0? (Cheap; unlocks ENS-text replacement.)
4. **Schemas** (§9) — core registry in v0, or free-form now, standardise later?
5. **Reactive subscriptions** (§10) — v0 or WM-10a?
6. **Multi-writer merge** (§11) — CAS-only v0, CRDT profile later?
7. **Inheritance/estate** (§13) — design now (it's a headline use-case) or note &
   defer?
7b. **Disclosure / REVEAL** (§10b) — is `reveal` a v0 verb-cap? Ship both modes
   (whole-group, subset-sealed)? Is `redisclose` a v0 caveat? Field-level
   selective disclosure (BBS+/ZK) as WM-10c?
8. **Storage/metering** (§14) — inline size ceiling; archival + treasury donation
   in the standard or deployment?
9. **Naming** — is "OWM-KV" right, or does the ambition (self-sovereign data
   store) deserve a better name (OWM-Vault? OWM-Store? OWM-Data)?

---

## 18b. Composable form (WM-13/14/15)

OWM-KV is a **profile** over the foundation: records are an **append-only signed
log + fold** (§6) gated by **grants** (§5); values are **sealed payloads** (WM-8)
carried in a **group** (§3); and **REVEAL** (§10b) is the KV instance of the
general **Presentation** primitive (WM-14, kind 590) — same challenge / context /
version-pin semantics. The committed `owm-kv-reveal` (554) stays valid; converging
it onto WM-14 is a non-breaking refactor. See [PRIMITIVES](PRIMITIVES.md).

## 19. Prior-art anchors

Solid PODs (personal datastore you grant apps slices of — the closest vision),
Ceramic / ComposeDB + IDX (KV-by-DID, signed append-only streams, schemas),
Spruce Kepler + SIWE (wallet-owned encrypted orbits, UCAN/ZCAP delegation), Lit
Protocol (programmable decrypt access — contrast: no threshold network here),
Lighthouse/Fileverse (wallet-address encrypted sharing), UCAN / ZCAP-LD / Biscuit
(capability delegation, attenuation, caveats, chains), CRDT / Automerge
(multi-writer merge), W3C Verifiable Credentials + DIDComm (the `credentials`
namespace), ENS text records (the public-record contrast), age / GPG
(multi-recipient key-wrapping), HashiCorp Vault / AWS Secrets Manager (the
centralised incumbent), EIP-191 (signing), MLS RFC 9420 (the transport, forward
secrecy = revocation).
