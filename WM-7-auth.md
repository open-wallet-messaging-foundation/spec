# WM-7-auth

**Status:** Draft (2026-07-13, content landed with the reference
implementation). **Profile:** see WM-0.

**Scope:** OWM-AUTH — the wallet as a second factor and step-up approver
(kinds 530/531); OWM-GRANT — wallet sign-in and delegated authorization as
signed capabilities (kinds 532–534); canonical signing strings and domain
separation; the match-code (number-entry) ceremony; grant registry and
revocation; the OIDC bridge and native session profiles; EIP-4361 (SIWE)
interoperability.

Reference implementation: `packages/owm-core/src/auth.js` (canonical
strings, sign/verify — pure logic), `src/eth-sign.js` (shared EIP-191
signing), envelope wire formats in `src/envelope.js`; server library
`packages/owm-auth` (`@open-wallet-messaging/auth`). Kind numbers: WM-5 / `api/kinds.json`.

## 1. Problem and design

Every deployed second factor either stores a server-side secret (TOTP
seeds: one database breach breaks every enrolled user) or hands the user a
context-free short code that can be read to a phisher (SMS OTP, TOTP).
OWM-AUTH replaces both with a wallet signature over *this relying party,
this action, this nonce*:

- the server pins only a **public address** at enrollment — a breach
  leaks nothing usable;
- the artifact the user produces is a signature bound to the RP, the
  action text, a single-use nonce, and an expiry — useless anywhere else;
- the challenge travels E2EE over the OWM channel with content-free
  knocks (WM-6): transport providers never learn an auth prompt was sent.

The same ceremony doubles as **step-up transaction approval**: `action`
carries the full human-readable summary ("Release wire #4711: $25,000 to
acct …991") and the thing the user signs IS that summary (WYSIWYS).

## 2. Wire formats (kinds 530–534)

All five kinds validate strictly (WM-4: missing, extra, or type-mismatched
keys are rejected). String fields MUST NOT contain CR or LF — field values
ride newline-delimited canonical signing strings (§3.3, §4.2), so newlines
are banned at the envelope layer to keep the encoding unambiguous.

**Time convention:** `iat`, `exp`, and `ts` in WM-7 kinds are **unix
seconds** (the JWT convention — these fields interoperate with JWT-shaped
verifiers). Note this differs deliberately from the unix-ms `ts` of
WM-3/WM-4 kinds.

| Kind | Code | Fields |
|---|---|---|
| `wm-auth-challenge` | 530 | `v:1`, `rp` (1–128 chars), `action` (1–140 chars, human-readable, rendered verbatim), `challenge` (64 lowercase hex — a 32-byte single-use nonce), `binding?` (1–256 chars, strict mode §6), `iat`, `exp` (MUST be ≤ `iat` + 120) |
| `wm-auth-response` | 531 | `v:1`, `challenge` (64 hex), `match` (1–8 decimal digits — what the user TYPED), `addr` (`0x` + 40 hex), `sig` (130 lowercase hex, r‖s‖v) |
| `wm-grant-request` | 532 | `v:1`, `rp`, `client` (1–128 chars), `scope` (space-separated printable tokens, ≤ 512 chars), `aud` (1–256 chars), `nonce` (64 hex), `iat`, `exp` |
| `wm-grant` | 533 | the 532 fields as approved, plus `addr` (`0x` + 40 hex, the granting sub-identity) and `sig` (130 hex) |
| `wm-grant-revoke` | 534 | `v:1`, `grantId` (64 hex, §4.2), `ts`, `sig` (130 hex) |

The **match code does NOT ride the challenge** (see §3.2). A user decline
is signalled with the terminal `scx-abort` envelope (514, WM-3) carrying
reason `declined` — added to the WM-3 abort-reason enum by this document.
It is not an SCX handshake failure; it reuses the same terminal envelope.

## 3. OWM-AUTH — the ceremony

### 3.1 Enrollment

The service pins the **auth address** once, over an authenticated path:

- an SCX ceremony (WM-3): the contact card's signature is bound to that
  exchange's transcript, so a card from any other exchange is refused; or
- an in-session proof-of-possession: a 530/531 round-trip inside an
  already-authenticated session; the address that validly signs is pinned.

Never a pasted address. Never updatable from an unauthenticated path.
Key-loss recovery = re-enrollment from an authenticated session — never
seed exposure. The enrolled key SHOULD be a dedicated sub-identity (WM-1),
never a treasury/money key: auth-key theft must not be money theft.

### 3.2 Challenge → sign → verify

```
1. user acts on the INITIATING screen (log in / release the wire)
   → that screen displays a 2-digit MATCH CODE
   → server sends wm-auth-challenge (530) E2EE to the enrolled identity
     (content-free knock at the transport layer, WM-6)
2. wallet renders rp + action VERBATIM, requires device unlock,
   requires the user to TYPE the match code from the initiating screen,
   signs, returns wm-auth-response (531)  — or scx-abort(declined)
3. server verifies: signature recovers the enrolled address; nonce is
   live, unused, unexpired; typed match equals the displayed match;
   (strict mode: binding matches). ANY attempt burns the nonce.
   N failures/declines → lock the user + security alert.
```

**Number-ENTRY rationale.** Push-2FA number-*matching* (Duo/Microsoft
style) shows a number on both screens and asks "same?" — a fatigued user
taps yes. Here the code exists ONLY on the initiating screen and the
wallet demands its entry: an unsolicited prompt is unanswerable — the
victim never saw a code. MFA bombing is dead by construction, not by user
diligence. Two digits suffice because the code is not a secret — it is a
channel-binding gesture; the security lives in the signature and nonce.

### 3.3 Canonical auth string (normative)

The 531 `sig` is an EIP-191 `personal_sign` (secp256k1, recovery-based
verification, case-insensitive address compare — the same scheme as WM-3
contact cards) over exactly:

```
"owm-auth-v1" \n rp \n action \n challenge \n match \n (binding | "-") \n exp
```

- literal domain tag `owm-auth-v1` first;
- `binding` is the challenge's binding verbatim, or the single character
  `-` when absent (the literal binding value `-` is therefore reserved);
- `exp` as its decimal integer representation;
- verifiers rebuild this string from **their own state** (what they
  issued and displayed), never from attacker-supplied fields.

Domain separation guarantees: `owm-auth-v1` + the EIP-191 prefix mean an
auth signature can never be a valid transaction, grant, revoke, or contact
card — and vice versa (the three WM-7 domains and the WM-3 card domain are
mutually disjoint; the reference tests prove cross-verification fails).
Wallets MUST refuse raw-hex signing for this flow.

Verification failure reasons are distinct and machine-readable:
`challenge-mismatch`, `expired`, `match-mismatch`, `bad-signature`,
`wrong-address` (valid signature, non-enrolled key).

### 3.4 Attack table

| Attack | Outcome |
|---|---|
| Server DB breach | Public address only — 2FA intact (TOTP: broken) |
| "Read me the code" phish | Nothing to read out: the match code is useless without the wallet signature, and the signature binds rp+action+nonce |
| Replay / brute force | Single-use 32-byte nonce, ≤ 120 s, burned on any attempt (TOTP: 6 digits, 30 s window) |
| MFA bombing | Dead: the victim cannot type a code they never saw; N declines lock + alert |
| Prompt spoofing over messaging | Verified-sender attestation (WM-4, kind 515) + inbound screening gate who can deliver an auth prompt at all |
| Cross-protocol / blind-signing abuse | Domain-separated canonical string; never a valid tx; strict envelopes |
| Auth-key theft ≠ money theft | Dedicated sub-identity; recovery = re-enroll, never seed export |
| Live relay MITM (attacker drives the real site in real time) | Default mode: NOT cryptographically excluded — honest ceiling, same as every non-WebAuthn method; see §6 |

## 4. OWM-GRANT

One primitive, two tracks: **sign-in** (§5) and **delegated
authorization** — OAuth's core job ("app X may access my resource at
service Y within scope S until T") as a wallet-signed capability instead
of a bearer token from a central authorization server.

### 4.1 Issuance

```
1. service sends wm-grant-request (532) E2EE: client, scope, aud,
   requested exp, single-use nonce. The wallet renders it VERBATIM
   (WYSIWYS consent — a payment scope displays the payment terms).
2. user approves (device unlock + §3 hygiene) → wallet signs the
   canonical grant string with the per-RP sub-key → wm-grant (533).
3. issuance acceptance: the grant must answer an outstanding request
   (nonce known and unburned); the nonce burns on success.
```

### 4.2 Canonical grant string and grantId (normative)

```
"owm-grant-v1" \n rp \n client \n scope \n aud \n nonce \n iat \n exp
```

`grantId = SHA-256(canonical grant string)`, lowercase hex. Anyone holding
the grant fields can compute it; no collision-free second grant can share
it.

### 4.3 Verification (offline) and the exp trade-off

The resource server verifies the signature chain **offline** — no token
endpoint, no client secret, no introspection round-trip. Grants are
sender-constrained by construction (the signing key is the constraint —
what OAuth retrofits as DPoP, RFC 9449). Checks: signature recovers
`addr`; `addr` is the pinned per-RP identity; `aud` equals the verifier's
own identifier; `rp` matches; not past `exp`.

**The honest trade-off, stated:** offline verification is bounded by
`exp`. OAuth answered this with short-lived tokens + refresh; OWM-GRANT
answers the same way — grants carry a short `exp` by default (reference:
15 min), and a grant whose lifetime (`exp − iat`) exceeds a configured
threshold (reference: 1 h) **MUST fail verification unless a grant
registry is configured**; instant revocation costs exactly one registry
lookup. Verifiers with a registry MUST check it on every verification.

### 4.4 Revocation

```
"owm-grant-revoke-v1" \n grantId \n ts
```

`wm-grant-revoke` (534) is valid only when signed by the same key that
signed the grant. Revocation is permanent and wins over everything —
including grants that would otherwise still be inside `exp` (the invite
lifecycle rules of WM-4, applied to a new payload: revocation > TTL >
single-use accounting).

### 4.5 Consent hygiene

Unsolicited grant requests cannot reach the user (screening + sender
attestation, WM-4/WM-6); declines are terminal `scx-abort(declined)`; N
declines lock the client. Per-RP sub-keys only — never a money key.

## 5. Sign-in profiles

A login IS the §3 ceremony with `action: "sign in"` — no identity
provider watches logins, no redirect dance to phish (consent phishing,
auth-code interception, and mix-up attacks are artifacts of redirects; an
OWM sign-in has none). Two rules make it structurally better than
federated login, not just different:

- **Per-RP sub-identity by default.** One address across sites is a
  tracking cookie the user cannot clear. Wallets derive one sub-identity
  per RP (reference placeholder: scalar = keccak256(seed ‖ rp), to be
  replaced by BIP-32 hardened derivation per WM-1 before v0 freeze). The
  primary address never appears.
- **Minimal claims.** A bare signature proves key control; anything more
  is a selective credential presentation (WM-4 C-stack), not a profile
  dump.

### 5.1 OIDC bridge profile

For services keeping their OIDC plumbing: a standard **OIDC issuer facade
that authenticates via wallet** (`@open-wallet-messaging/auth` `createOidcIssuer`).
Discovery, JWKS, authorization-code flow with **PKCE S256 required**,
single-use codes, exact `redirect_uri` match, `state` passthrough,
`nonce` claim, ES256 `id_token` with `sub` = CAIP-10 account
(`eip155:<chainId>:<address>`). The wallet ceremony is a caller-supplied
callback — any transport. **v0-dev profile:** no dynamic client
registration, consent persistence, refresh tokens, or userinfo endpoint
yet; static client registration is available and recommended.

### 5.2 Native session profile

For services dropping OIDC entirely (`createWalletSession`): verify
EITHER an OWM-AUTH response with `action: "sign in"` OR an EIP-4361
message (§7), then issue a compact ES256 session JWT
`{ iss: rp, sub: <CAIP-10 address>, iat, exp }`. Cross-service
verification is one JWKS fetch. No redirects, no codes, no IdP.

## 6. Security ceilings (honest)

**Default mode ceiling:** a live relay (evilginx-class) that proxies the
genuine site in real time — and therefore also relays the match code — is
not cryptographically excluded. This is the same ceiling as every
non-WebAuthn method, including push 2FA with number matching; OWM-AUTH
still strictly dominates TOTP (no server secret, no phishable code) and
SMS OTP (no carrier, no SIM swap).

**Strict mode (planned):** the `binding` field carries a hash of the
initiating session, delivered out-of-band (QR from the genuine page) and
signed into the response — a per-attempt risk signal no TOTP deployment
has. Stated honestly: a QR alone is still relayable in real time; full
WebAuthn-parity exclusion needs a proximity check or same-device deep
link with OS-verified origin. Envelope and canonical-string support for
`binding` ships now; the proximity ceremony is future work.

**Grant ceiling:** offline verification is bounded by `exp` (§4.3). Long
lifetimes without a registry fail closed.

## 7. EIP-4361 (SIWE) interop

OWM sign-in coexists with Sign-In with Ethereum rather than competing
with it: `@open-wallet-messaging/auth` ships `OwmSiweMessage`, an API-compatible
replacement for the `siwe` package (byte-exact ABNF `prepareMessage()`,
full parse round-trip, the same `verify()` response/rejection shape and
error strings, `generateNonce()`), verified with the same shared EIP-191
recovery code as every other OWM signature. The native session profile
(§5.2) accepts a verified SIWE message as a first factor interchangeably
with an OWM-AUTH response. EOA signatures verify with zero dependencies;
smart-account (EIP-1271/6492) signatures are an optional, explicitly
configured RPC path — see §8.

Prior-art anchors: RFC 6238 (the baseline replaced), WebAuthn/CTAP (the
origin-binding parity target), Duo/Microsoft number-matching (upgraded to
number-entry), EIP-4361/CAIP-122 (sign-in canonicalization), OpenID
Connect + CIBA (the bridged incumbent; the decoupled-consent shape),
UCAN/ZCAP-LD/Biscuit (capability tokens), RFC 9449 DPoP
(sender-constraint convergence), GNAP (closest standards-track cousin).

## 8. Smart-account signatures (ERC-1271 / ERC-6492) — optional

Default verification is EOA secp256k1 recovery, which excludes users
whose wallet is a contract (Safe, Argent, ERC-4337 accounts): a contract
cannot produce a signature that *recovers to its own address*. Verifiers
MAY additionally support ERC-1271, and if they do, these rules apply:

**Hash.** The ERC-1271 hash is the **EIP-191 digest of the exact
canonical string** the EOA path signs — `canonicalAuthPayload` (§3.3),
`canonicalGrantPayload` (§4.2), `canonicalGrantRevokePayload` (§4.4), or
the EIP-4361 message text. There is no separate smart-account signing
domain: one canonical string, one digest, two ways to check it.

**Order.** (1) EOA recovery first, purely offline — a match MUST NOT
trigger any RPC, and a deployment with no verifier configured MUST behave
exactly as an EOA-only verifier. (2) If the signature ends with the
ERC-6492 magic suffix (32 bytes of `0x6492…6492`), unwrap the
`abi.encode(address factory, bytes factoryCalldata, bytes originalSig)`
envelope and continue with `originalSig`. (3) If `eth_getCode(addr)` is
non-empty, staticcall `isValidSignature(bytes32,bytes)`; accept **only**
the exact magic value `0x1626ba7e` (a bare `bytes4` or the zero-padded
ABI word — any other byte, including nonzero padding, is a failure).

**Fail closed, distinct reasons.** No contract code on a plain signature
(`no-code`), a 6492 wrapper with no code deployed
(`counterfactual-unsupported`), a non-magic return (`bad-magic`), any RPC
error, timeout, or malformed response (`rpc-error`), and any malformed
wrapper (`bad-signature`) are all verification failures; they feed the
same nonce-burning and lockout accounting as EOA failures. The reference
does NOT implement the EIP-6492 deploy-and-simulate universal validator
(the EIP publishes Solidity source, not byte-verifiable bytecode), so
counterfactual signatures fail closed until the account is deployed.

**Trust shift (normative warning).** ERC-1271 verification trusts the RPC
endpoint to report code and execute `isValidSignature` honestly; a
malicious endpoint can forge acceptance. Deployments MUST treat the
endpoint as part of their trust base: run your own node or verify against
a quorum of independent endpoints for high-value actions. This is why the
capability is opt-in, per verifier, never a default.

**Envelope interaction.** The 530s kinds cap `sig` at 65 bytes (§2), so
6492 wrappers and multi-owner signature blobs do not fit OWM envelopes;
over those kinds, smart accounts participate when they validate a 65-byte
owner/session-key signature via ERC-1271 (the common Safe and ERC-4337
shape). The SIWE seam accepts arbitrary-length signatures, wrappers
included. Reference: `packages/owm-auth/src/erc1271.js`
(`createChainVerifier`), threaded through `OwmAuthServer`, `GrantServer`,
`createWalletSession`, and `OwmSiweMessage.verify` as the optional
`verifier`.

## 9. OWM-APPROVAL — M-of-N quorums (kinds 535–537)

The MLS room is the committee: membership is cryptographic, the E2EE
transcript is the ordered audit trail. Reference:
`packages/owm-core/src/approval.js`.

**Wire.** `wm-approval-request` (535): `{v, approvalId (32-byte hex
nonce), rp, action (free text ≤2000, may be multiline — WYSIWYS: the
thing approved IS the thing displayed), policy {m, signers[≤64, distinct,
m ≤ |signers|], exp}, iat, exp}`. `wm-approval-sig` (536): `{v,
approvalId, signer, actionHash, sig}`. `wm-approval-result` (537): `{v,
approvalId, actionHash, sigs: [{signer, sig}], ts}`.

**Canonical approval string (normative).** EIP-191 over

```
owm-approval-v1 \n rp \n approvalId \n sha256(action) \n signer \n exp
```

Signing the *hash* of the action keeps multiline WYSIWYS text out of the
newline-delimited canonical while still binding every byte of it: a
signature cannot be replayed onto a different action, approval, relying
party, or window. The domain tag is disjoint from `owm-auth-v1` and
`owm-grant-v1` — an approval signature can never double as either.

**Verification (offline).** Anyone holding the 535 verifies a 537 with no
server: every listed signature must recover to its declared signer, that
signer must be on the policy roster, signers must be distinct, and the
distinct count must reach `policy.m`. Aggregation below quorum throws — a
partial aggregate must never look like a result. Passing `now` enforces
the signing window at acceptance time; omitting it audits a historical
approval whose window has since closed.

**Safe execution (Phase 4 adapter).** When the action IS an on-chain
Safe transaction, the action text embeds the `safeTxHash`; the execution
adapter has each signer ALSO produce the EIP-712 `SafeTx` owner signature
in the same ceremony — the 536 is the room-side approval, the adapter's
is the chain-side one: two signatures, disjoint domains, one screen.
Policy MAY additionally require OWM-PRESENCE call attestations (547,
WM-4): every 536 must then come from a signer with same-epoch attested
presence.

**Ceilings.** A quorum attests that M keys signed this action hash —
device compromise of M signers defeats it (that is the §6 story, not a
new one); the roster lives in the 535, so verifiers MUST take the policy
from a trusted copy of the request, not from an attacker-supplied one.

## 10. OWM-BIND — the institutional account-binding profile

A customer-held key becomes the authorization root for an institutional
account (bank, exchange, broker). The institution holds only the public
address; every account action requires the customer's signature; the
institution communicates with the customer ONLY over the bound channel.
This is a **profile over existing kinds — it mints nothing**:

| Lifecycle step | Mechanism |
|---|---|
| Enroll | SCX-grade ceremony (WM-3) binds the right key; the binding is a **grant** (533): rp = institution, scope = hashed account reference; the institution countersigns with its 515 sender-attestation |
| Authorize a transaction | 530 challenge whose `action` IS the transaction (amount, payee, reference — dynamic linking); customer's 531 is the authorization; **both sides retain the artifact** |
| Joint / corporate accounts | §9 quorums (535–537) |
| Smart-account customers | §8 (ERC-1271 / ERC-6492) |
| Rotate | new key's binding grant signed by the old key |
| Revoke | 534, via the bound channel + institutional re-verification |
| Recover | guardian quorum (§9) OR institutional re-KYC re-binding that MUST be slow, with a challenge window during which the previously bound key can veto |
| Channel exclusivity | a 515-attested commitment, scoped to the binding: ALL communication for this account rides the bound channel; out-of-band media carry at most content-free, link-free knocks (WM-6) |

**Identity hygiene (normative).** The bound key MUST be a dedicated
sub-identity per institution, never used on-chain: the institution
learns nothing of the customer's other activity; chain observers learn
nothing of the banking relationship.

**Economics (normative rails).** The institution is the sender and the
payer: it funds its own gateway metering and bears carrier costs for
any customer-chosen knock sink. **Receiving, binding, and recovery are
free to the customer — always.** The foundation collects no per-message
rent; any conformant operator may serve any institution.

**Ceilings (normative).** A fiat ledger is the institution's database:
this profile cannot mathematically prevent a malicious institution from
moving entries. It guarantees two things instead: the institution can
never produce an authorization the customer did not sign, and the
customer can prove the absence of authorization — every legitimate
movement has an artifact, so an artifact-less movement is provably
unauthorized. Exclusivity cannot stop spoofed messages from *arriving*
on legacy media; it removes their legitimate cover ("this institution
never texts") and makes an institution's own out-of-band lapse a
breach of its signed commitment.
