# WM-3-scx

**Status:** Draft (2026-07-13, content landed with the Phase 2 reference
implementation). **Profile:** see WM-0.

**Scope:** Secure Contact Exchange (SCX): short pairing codes; SPAKE2
(RFC 9382); transcript-hash-bound EIP-191 contact-card signatures; SAS
confirmation; abort semantics; QR and bearer-link degenerate cases.

Reference implementation: `packages/owm-core/src/scx-code.js`,
`src/spake2.js`, `src/scx.js` (pure logic, no I/O). Envelope wire formats:
`src/envelope.js`; kind numbers: WM-5 / `api/kinds.json`.

## 1. Problem and flow

SCX exchanges wallet addresses (as signed, proof-of-ownership contact
cards) between two parties whose only common channel is insecure — SMS, a
voice call, a chat app. The insecure channel carries **only a short,
low-value pairing code**; addresses never ride it. The real exchange runs
over a rendezvous mailbox (WM-2 §1), protected by a PAKE.

```
Alice                                  insecure channel (SMS / voice / chat)
  1. generate code "7-panda-mocha-quilt" ───────────────────────────► Bob
     (mailbox 7 on the rendezvous relay + 3 wordlist words)
Alice ◄══ SPAKE2 (RFC 9382) via mailbox 7, kinds 510/511/513 ══════► Bob
  2. both derive a strong session key from the weak code; each side
     proves knowledge with a confirmation MAC. A wrong code gets exactly
     ONE online guess and breaks the handshake VISIBLY (scx-abort 514).
  3. exchange wm-contact-card (512), each signed by the wallet key it
     asserts, the signature covering THIS exchange's transcript hash.
  4. both screens show the same SAS: 2 emoji + 4 digits. Humans compare.
     Match → scx-confirm(sas). Mismatch → scx-abort(sas-mismatch). NEVER
     proceed on mismatch.
```

Every step that can fail fails loudly: the session emits `scx-abort` with a
machine-readable reason and hard-stops. There is no silent downgrade.

## 2. Pairing code

Format (normative):

```
<mailboxId>-<word>-<word>-<word>          e.g.  7-panda-mocha-quilt
```

- `mailboxId` — decimal integer ≥ 0, no leading zeros, as issued by the
  rendezvous relay (WM-2 §1.1). It locates the mailbox; it carries **no
  security value**.
- Exactly **3 words**, lowercase, `-`-separated. The word portion
  (`panda-mocha-quilt`) is the PAKE password; the mailbox id is **not**
  part of the password.
- Words are drawn from the OWM SCX wordlist: the **EFF Short Wordlist 1**
  (1,296 words; © Electronic Frontier Foundation, CC-BY 3.0,
  https://www.eff.org/dice), with one adaptation: entry 6652 `yo-yo` is
  replaced by `yolk`, because `-` is the code separator. All other entries
  and their order are verbatim. The adapted table embedded in
  `scx-code.js` is normative.
- Word selection MUST be uniform via a CSPRNG with **rejection sampling**
  (no modulo bias): draw 16 bits, accept only values below
  `65536 - (65536 mod 1296) = 64800`, reduce mod 1296.
- Parsers MUST reject: wrong part count, malformed mailbox id, and any
  word not in the wordlist. Rejecting unknown words catches transcription
  typos *before* they burn the single online PAKE guess.

Entropy: 3 words = 1296³ ≈ 2³¹. This is sufficient **only** because SPAKE2
limits an active attacker to one online guess per handshake (§9); it is
NOT sufficient against offline attack, which is exactly what a PAKE
prevents.

## 3. PAKE: SPAKE2-P256-SHA256-HKDF-HMAC (RFC 9382)

SCX uses the SPAKE2 ciphersuite over P-256 with SHA-256, HKDF, and
HMAC-SHA256, exactly as specified in RFC 9382. CPace remains acceptable as
a future alternative suite; v0 implementations MUST support SPAKE2.

### 3.1 Fixed points

From RFC 9382 §6 (compressed SEC1):

```
M = 02886e2f97ace46e55ba9dd7242579f2993b64e16ef3dcab95afd497333d8fa12f
N = 03d8bbd6c639c62937b04d997f38c3770719c629d7014d49a24b4f98baa1292b49
```

### 3.2 Password scalar w

```
w = int( SHA-256( UTF8(password) || UTF8(context) ) ) mod n
```

where `password` is the word portion of the code, `context` is the ASCII
string `owm-scx-v1`, and `n` is the P-256 group order. If `w = 0`
(probability ≈ 2⁻²⁵⁶), the digest is hashed again. RFC 9382 recommends a
memory-hard function here; v0 deliberately uses plain SHA-256 because SCX
security rests on the online-guess bound, not on offline hashing cost.
This choice is versioned by the `context` string and can be upgraded.

### 3.3 Roles, shares, validation

The party that generated the code is role **A**; the party that received
it is role **B**. Scalars `x`, `y` are uniform in `[1, n)` via rejection
sampling.

```
A:  pA = w*M + x*P        B:  pB = w*N + y*P
A:  K  = x*(pB - w*N)     B:  K  = y*(pA - w*M)      (cofactor h = 1)
```

Shares travel as **uncompressed SEC1** (65 bytes, `04` prefix), lowercase
hex. On receipt a share MUST be rejected unless it is exactly 65 bytes,
`04`-prefixed, decodes to a point on P-256, and is not the identity
element. If `pB - w*N` (resp. `pA - w*M`) is the identity, the handshake
MUST be rejected. Failures here abort with reason `protocol`.

### 3.4 Transcript TT

Exactly RFC 9382 §3.3 — `len(S)` is an 8-byte little-endian byte count:

```
TT = len(A)  || A          A = "owm-scx-a"   (fixed SCX identity, role A)
  || len(B)  || B          B = "owm-scx-b"   (fixed SCX identity, role B)
  || len(pA) || pA         uncompressed SEC1, 65 bytes
  || len(pB) || pB         uncompressed SEC1, 65 bytes
  || len(K)  || K          uncompressed SEC1, 65 bytes
  || len(w)  || w          big-endian, padded to 32 bytes
```

SCX parties are pseudonymous at this stage, so the RFC identities are the
protocol roles themselves. AAD is empty in v0.

### 3.5 Key schedule and confirmation (RFC 9382 §4)

```
Ke || Ka   = SHA-256(TT)                          (16 bytes each)
KcA || KcB = HKDF-SHA256(ikm = Ka, salt = nil,
                         info = "ConfirmationKeys", L = 32)
cA = HMAC-SHA256(KcA, TT)      sent by A in scx-confirm{phase:"mac"}
cB = HMAC-SHA256(KcB, TT)      sent by B in scx-confirm{phase:"mac"}
```

Each side verifies the peer's MAC with a **constant-time comparison**. A
mismatch is the wrong-code / wrong-guess signal: abort with
`bad-confirmation`. `Ke` is the session key; it MUST NOT be exposed to the
application before the peer's confirmation MAC verifies. `Ka` and the
confirmation keys are used for nothing else.

The implementation is validated against all four test vectors of
RFC 9382 Appendix B (`test/spake2.test.js`).

## 4. Transcript hash

The public, card-bindable transcript hash is **domain-separated**:

```
transcriptHash = hex( SHA-256( UTF8("owm-scx-transcript-v1") || TT ) )
```

Rationale (normative): RFC 9382 defines `SHA-256(TT) = Ke || Ka`, so the
*bare* digest of TT **is the key block** — publishing it would disclose
the session keys. The tagged hash is safe to embed in signed cards, logs,
and UI.

## 5. SAS — short authentication string

The SAS is 2 emoji + 4 decimal digits derived from `transcriptHash`
(bytes `th[0..5]`):

```
emoji₁  = EMOJI[ th[0] mod 64 ]
emoji₂  = EMOJI[ th[1] mod 64 ]
digits  = ( u32be(th[2..5]) mod 10000 ), zero-padded to 4 chars
display = "<emoji₁> <emoji₂> <digits>"
```

64 divides 256, so emoji selection is unbiased. Both sides derive the SAS
independently from their own transcript; humans compare out loud or by
eyeball. The 64-entry emoji table is normative **including order**; all
entries are single Unicode codepoints:

| idx | | idx | | idx | | idx | |
|---|---|---|---|---|---|---|---|
| 0 | 🐶 | 16 | 🐞 | 32 | 🌙 | 48 | 🧀 |
| 1 | 🐱 | 17 | 🐠 | 33 | ⭐ | 49 | 🥚 |
| 2 | 🦁 | 18 | 🐬 | 34 | 🌈 | 50 | 🚀 |
| 3 | 🐴 | 19 | 🐳 | 35 | 🔥 | 51 | 🚂 |
| 4 | 🦄 | 20 | 🦈 | 36 | 💧 | 52 | 🚗 |
| 5 | 🐮 | 21 | 🐢 | 37 | ⚡ | 53 | 🚲 |
| 6 | 🐷 | 22 | 🦀 | 38 | 🌊 | 54 | ⛵ |
| 7 | 🐸 | 23 | 🐘 | 39 | 🍇 | 55 | 🚁 |
| 8 | 🐙 | 24 | 🌵 | 40 | 🍎 | 56 | 🎈 |
| 9 | 🐵 | 25 | 🌲 | 41 | 🍋 | 57 | 🎁 |
| 10 | 🦅 | 26 | 🌴 | 42 | 🍉 | 58 | 🎨 |
| 11 | 🦆 | 27 | 🍀 | 43 | 🍓 | 59 | 🎵 |
| 12 | 🦉 | 28 | 🍁 | 44 | 🍒 | 60 | 🔑 |
| 13 | 🐝 | 29 | 🍄 | 45 | 🥕 | 61 | 🔔 |
| 14 | 🦋 | 30 | 🌷 | 46 | 🌽 | 62 | ⚽ |
| 15 | 🐌 | 31 | 🌻 | 47 | 🍞 | 63 | 🎲 |

## 6. Contact card

```json
{ "v": 1, "address": "0x…40 hex…", "inboxId": "…", "displayName": "…?",
  "guest": true?, "ts": 1770000000000, "sig": "…130 hex…" }
```

`sig` is a secp256k1 signature, Ethereum `r || s || v` layout (65 bytes,
`v ∈ {27, 28}`), EIP-191 personal-message style over the canonical payload:

```
digest = keccak256( "\x19Ethereum Signed Message:\n" || len || payload )

payload =
  owm-scx-card-v1        \n
  address:<address, lowercased>   \n
  inboxId:<inboxId>      \n
  displayName:<displayName or "">  \n
  guest:<1 | 0>          \n
  ts:<ts>                \n
  transcript:<transcriptHash>
```

Newlines are banned in `inboxId` and `displayName` (wire-rejected), so the
payload is unambiguous. The payload **includes the transcript hash of this
exchange** — that is the anti cut-and-paste binding (§9).

Verification (`verifyContactCard(card, transcriptHash)`):

1. rebuild the canonical payload from the card's fields and the verifier's
   **own** transcript hash;
2. recover the signer address from `sig` over the EIP-191 digest;
3. require `signer == lowercase(card.address)`.

A genuine card from a *different* exchange rebuilds a different payload,
recovers a different signer, and fails. Success proves control of the
asserted wallet key at the asserted transcript — you receive
*proof-of-ownership*, not a string. Either side MAY present a derived
sub-identity card (WM-1); the signature proves control of the *presented*
key and says nothing about any parent wallet.

## 7. Wire formats

All SCX messages are strict OWM envelopes (WM-4): for a known kind, a
missing, extra, or type-mismatched key is rejected. Kinds are registered
in WM-5 (`api/kinds.json`).

| Kind | Code | Fields |
|---|---|---|
| `scx-pake-a` | 510 | `v:1`, `share`: 130-char lowercase hex, `04`-prefixed |
| `scx-pake-b` | 511 | `v:1`, `share`: as above |
| `wm-contact-card` | 512 | `v:1`, `address` (`0x` + 40 hex), `inboxId` (1–128 chars, no newlines), `displayName?` (≤ 64 chars, no newlines), `guest?` (boolean), `ts` (unix ms), `sig` (130 hex) |
| `scx-confirm` | 513 | `v:1`, `phase`: `"mac"` \| `"sas"`, `mac?`: 64 hex (present iff phase = `mac`) |
| `scx-abort` | 514 | `v:1`, `reason`: see below |

Abort reasons (closed enum in v0):

| Reason | Meaning |
|---|---|
| `bad-confirmation` | PAKE confirmation MAC mismatch — wrong code, or an active wrong guess |
| `bad-card-signature` | contact card not bound to this transcript / signer mismatch |
| `sas-mismatch` | humans compared the SAS and it differed |
| `timeout` | peer never appeared or stalled past the transport deadline |
| `protocol` | malformed, out-of-order, reflected, or off-curve message |
| `declined` | the user declined a WM-7 auth/grant prompt — carried by this same terminal envelope, never emitted by an SCX handshake (WM-7 §2) |

Message sequence (each direction, in order): `scx-pake-*` →
`scx-confirm{mac}` → `wm-contact-card` → `scx-confirm{sas}`, with
`scx-abort` terminal at any point. Transports MUST preserve per-direction
order (the WM-2 mailbox does). A side reads only the other side's frames
(WM-2 §1.1); a session receiving its **own** pake kind aborts with
`protocol` (reflection defence).

## 8. Session state machine

```
init → pake-sent → key-confirmed → card-exchanged → sas-pending
     → confirmed | aborted
```

- **init → pake-sent** — send own share.
- **pake-sent** — on peer share: validate (§3.3), derive keys, send
  `scx-confirm{mac}`. On peer MAC: verify constant-time →
  **key-confirmed**, send own card. MAC mismatch → abort
  `bad-confirmation`.
- **key-confirmed** — on peer card: verify against own transcript hash →
  **card-exchanged** (SAS now displayable). Bad → abort
  `bad-card-signature`.
- **card-exchanged** — local human verdict: accept → send
  `scx-confirm{sas}` → **sas-pending** (or **confirmed** if the peer's SAS
  confirm already arrived); reject → abort `sas-mismatch`.
- **sas-pending** — on peer `scx-confirm{sas}` → **confirmed**.
- **aborted** — terminal. Session key material is wiped; the session
  ignores all further input and emits nothing.

The session key `Ke` is exposed only in states `key-confirmed` and later,
and never after `aborted`. Cards land in the client's address book only
from **confirmed**.

Confidentiality note: this layer produces and consumes envelopes; the
carrying transport (WM-2 mailbox frames) SHOULD encrypt post-confirmation
envelopes under a key derived from `Ke`. The security properties in §9 do
not depend on that encryption: card authenticity is carried by the
signature-over-transcript, not by the channel.

## 9. Security claims

1. **One online guess.** An attacker who intercepts the code's carrier
   channel *after* the fact learns nothing usable offline; an active
   attacker gets exactly one PAKE guess per handshake, and a wrong guess
   deterministically fails confirmation and surfaces as
   `scx-abort(bad-confirmation)`. Wrong guesses are *visible*, never
   silent (RFC 9382 security model + §3.5).
2. **Transcript binding kills cut-and-paste MITM.** A MITM running two
   parallel sessions cannot replay a genuine signed card from one session
   into the other: the transcript hashes differ, the recovered signer no
   longer matches, verification fails (§6), the session aborts.
3. **SAS closes the residual window.** The one permitted online guess, if
   it ever succeeded, would still produce different transcripts on the two
   real endpoints; the 2-emoji + 4-digit SAS comparison (≈ 2²⁵ space)
   detects it. Mismatch MUST abort — implementations MUST NOT offer a
   "proceed anyway" path.
4. **Proof of ownership.** A verified card proves control of the asserted
   key during this exchange — the receiving client can pin it in the
   address book and flag near-miss lookalike addresses afterwards.
5. **Fail-loud invariant.** Every failure path (bad point, bad MAC, bad
   signature, SAS mismatch, timeout, protocol violation) terminates the
   session with an explicit `scx-abort` and wipes key material.

## 10. Degenerate cases

- **QR, in person:** code (and optionally the card) in one frame; the PAKE
  and SAS still run. The QR carrier is authentic-by-proximity, so the SAS
  comparison is trivially satisfied but MUST still be shown.
- **Bearer invite links** (WM-4 rooms): the existing 256-bit
  fragment-carried (`#t=`) invite remains the one-way, no-ceremony option
  where bearer semantics are acceptable — invitations, never
  money-destination exchange. Invite tokens MUST ride URL fragments only;
  a token in a query string is refused at parse.
