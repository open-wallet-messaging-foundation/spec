# WM-8-encrypted-files

**Status:** Draft (2026-08-03, content landed with the reference
implementation). **Profile:** see WM-0 (this document is CORE — it needs no
relay, no notifier, no server of any kind).

**Scope:** OWMF — an authenticated, self-describing, crypto-agile container for
client-side file encryption; the **sign-to-derive** OWM key hierarchy (a file
key and an x25519 identity key derived deterministically from a wallet
signature); **recipient encryption** (seal a file/message to someone's OWM
public key) and its offline **`owm1:` envelope** for messaging over any channel;
and address→public-key **resolution**. Kinds `owm-identity-key` (524),
`owmf-msg` (525), `owmf-file` (526).

Reference implementation: `packages/owm-core/src/file-crypto.js` (OWMF, symmetric
self-encryption), `src/recipient-crypto.js` (identity keypair, sealing,
messaging), `src/pubkey-resolver.js` (address→pubkey), `src/eth-sign.js` (shared
EIP-191 signing). Conformance vectors: `api/vectors/owmf.json`.

## 1. Problem and design

The simplest useful thing a signing key can do — encrypt a file so only you (or
a chosen recipient) can open it — has no wallet-native standard. PGP got the
cryptography roughly right and everything around it wrong: key discovery,
forward secrecy, revocation, and a keyring users cannot manage. WM-8 anchors all
of that to the wallet.

Design invariants (normative):

- **100% client-side.** No byte of plaintext, ciphertext, or key material leaves
  the device. A conforming tool performs no upload.
- **Authenticated encryption (AEAD).** Every container is tamper-evident: the
  header is bound as AEAD associated data, so any altered byte fails the tag.
- **Crypto-agility.** The cipher is a versioned `scheme` byte inside the
  container and is **auto-detected** on decrypt; new schemes (including
  post-quantum) are added to the registry without breaking existing files.
- **Quantum posture (honest).** Symmetric AES-256 / ChaCha20 are already
  quantum-resistant (Grover halves the strength → 128-bit, safe). The PQ upgrade
  concerns the **asymmetric** (recipient) path; §4 reserves a hybrid
  `X25519 + ML-KEM` scheme so confidentiality survives "harvest-now,
  decrypt-later".
- **Sign-to-derive over wallet-decrypt.** Standard wallets **sign**; they do not
  expose an ECDH/decrypt primitive. WM-8 therefore derives all key material from
  **signatures**, so encryption round-trips with any wallet whose `personal_sign`
  is deterministic (§4.1). This is why "encrypt to a raw on-chain address" cannot
  round-trip with real wallets, and why recipient encryption uses a **published
  OWM identity key** (§6) rather than the address's on-chain key.

## 2. The OWMF container (self-encryption) — normative byte layout

All multi-byte integers big-endian. Fixed 52-byte header, then ciphertext.

```
off  size  field
0    4     magic          "OWMF"  = 0x4F 57 4D 46
4    1     version        0x01
5    1     scheme         §4 (0x01 = AES-256-GCM)
6    1     mode           0x01 symmetric-self · 0x02 recipient (§6)
7    1     flags          bit0 wallet-signed provenance · bit1 password-protected (PBKDF2, §5.2)
8    32    salt           HKDF salt; for mode 0x01 it is also the value the wallet signs (§5.1)
40   12    nonce          AEAD IV
52   ..    ciphertext‖tag AEAD(scheme) output; tag length per scheme (16 for GCM)
```

The **whole 52-byte header is the AEAD associated data** — `scheme`, `salt`, and
`nonce` cannot be altered without breaking the tag. A reader MUST reject any
input whose first four bytes are not `OWMF`, and MUST select the algorithm from
the `scheme` byte (never from the file extension).

## 3. Payload metadata (normative)

The AEAD *plaintext* is:

```
uvarint(len) ‖ JSON{ "n": name, "t": mimeType, "s": originalSize } ‖ file_bytes
```

`uvarint` is unsigned LEB128. Because the metadata is inside the AEAD, the
**filename and type are encrypted** and are restored on decrypt. See §8 for the
outer-filename privacy rule.

## 4. Scheme registry (crypto-agility)

| id | scheme | class | PQ | status |
|---|---|---|---|---|
| 0x01 | **AES-256-GCM** | symmetric AEAD | yes (256-bit) | reference |
| 0x02 | XChaCha20-Poly1305 | symmetric AEAD | yes | reserved |
| 0x10 | X25519-HKDF-SHA256 + AES-256-GCM | hybrid (recipient) | no | reference (§6) |
| 0x11 | X25519 + ML-KEM-768 + AES-256-GCM | hybrid (recipient) | **yes** | reserved (PQ roadmap) |

A reader that does not know a `scheme` MUST fail closed with a distinct
`unsupported-scheme` reason. New ids are additive; old files keep decrypting.

### 4.1 Determinism requirement (normative)

All WM-8 key derivations feed a **wallet `personal_sign` output** into HKDF. For
a file to decrypt — or a recipient to open a sealed message — the wallet's
`personal_sign` MUST be **deterministic** (RFC 6979). The major wallets
(MetaMask, Ledger, Rabby, Coinbase) are. A tool MUST detect the failure
(the AEAD tag fails) and report it as such, and MAY refuse to encrypt with a
wallet known to randomize. Signatures are compared/consumed case-insensitively
and with an optional leading `0x` stripped.

## 5. Symmetric self-encryption (mode 0x01)

"Encrypt for yourself; the wallet is the key."

### 5.1 File key (normative)

```
salt   ← 32 random bytes                       (stored in the header)
sig    ← personal_sign("owm-file-key-v1" ‖ 0x0A ‖ base64std(salt))
ikm    ← bytes(sig)  [ ‖ PBKDF2-SHA256(password, salt, 210000) → 32 bytes ]   (§5.2)
K      ← HKDF-SHA256(ikm, salt, info = utf8("owmf|" ‖ schemeName), 32)
ct     ← AEAD_enc(K, nonce, plaintext, aad = header)
```

`schemeName` is the registry name (`"AES-256-GCM"`). Decrypt reads `salt` +
`scheme` from the header, re-signs the identical string, re-derives `K`, and the
tag both proves integrity and proves wallet control.

### 5.2 Optional password (second factor)

If a password is supplied, header flag **bit1** is set and the password —
stretched with **PBKDF2-SHA256, 210000 iterations, 32-byte output, salt = the
header salt** — is concatenated after the signature bytes in the HKDF `ikm`. This
defeats a **seized wallet**: an attacker who can make the wallet sign still
cannot decrypt without the password, and PBKDF2 slows offline guessing. Readers
MUST prompt for the password when bit1 is set. Ceiling: a weak password plus a
seized wallet remains brute-forceable — PBKDF2 slows it, it does not rescue a bad
password.

## 6. Recipient encryption (mode 0x02) — the OWM identity key

To let others encrypt *to* you, publish an **OWM identity public key** (kind
`owm-identity-key`, 524), derived once from your wallet:

```
sig  ← personal_sign("owm-identity-key-v1")
sk   ← HKDF-SHA256(bytes(sig), salt = ∅, info = utf8("owm-x25519-identity"), 32)
pk   ← X25519.getPublicKey(sk)          // 32 bytes — this is what you publish
```

Sealing a payload to a recipient's `pk` (scheme 0x10):

```
ephSk ← random X25519 secret ; ephPk ← X25519.getPublicKey(ephSk)
shared← X25519(ephSk, pk)
K     ← HKDF-SHA256(shared, salt = ephPk, info = utf8("owm-seal-v1"), 32)
ct    ← AES-256-GCM(K, nonce, plaintext, aad = ephPk)
sealed← ephPk(32) ‖ nonce(12) ‖ ct‖tag
```

The recipient re-derives `sk` (by signing `owm-identity-key-v1`), computes
`shared = X25519(sk, ephPk)`, and opens it. A **sealed file** (kind `owmf-file`,
526) wraps `uvarint(len)‖JSON{n,t}‖bytes` as the sealed payload. No wallet ECDH
primitive is required — only signing.

## 7. Messaging over insecure channels (`owm1:` envelope)

Because SMS, email, WhatsApp, Telegram and X all carry text, a sealed message is
transported as a printable envelope (kind `owmf-msg`, 525):

```
owm1:<base64std(sealed)>
```

The sender needs only (1) their wallet and (2) the recipient's OWM public key.
The recipient pastes the string back into any OWM tool and their wallet opens it.
**The channel being insecure does not matter** — it only ever carries ciphertext.
Sender authenticity, when required, is a detached WM-7 signature over the
plaintext, verified against the sender's address.

## 8. Filename privacy (normative)

Name and type live inside the AEAD (§3) and never leak from the ciphertext. The
only leak vector is the **outer download name**. A conforming tool MUST default
the encrypted file's outer name to a **non-identifying** value —
`owmf-<first 8 lowercase hex of SHA-256(ciphertext)>.owmf` — and MAY offer an
explicit opt-in to preserve the original (`<name>.owmf`). Decrypt restores the
true name from the metadata regardless.

## 9. Public-key resolution from an address

A convenience for verification and identity display (NOT the recipient
encryption key of §6):

- **Solana** — the address **is** the ed25519 public key (base58-decode; MUST be
  32 bytes).
- **EVM / Tron (secp256k1)** — the address is `keccak(pubkey)[12:]`, so one
  transaction the address **signed** is required. Rebuild the type-specific
  signing hash (legacy/EIP-155, EIP-2930 type 1, EIP-1559 type 2), `ecrecover`
  the public key, and verify `keccak(pub[1:])[12:]` equals the address.

Honest ceiling: a recovered on-chain key supports **verification** and (in
principle) **encrypting to** someone, but standard wallets cannot ECDH-**decrypt**
— which is exactly why round-tripping recipient encryption uses the §6 OWM
identity key, not the on-chain key.

## 10. Security ceilings (honest)

- **Determinism** (§4.1) — non-RFC-6979 wallets break decryption; detected via
  the tag; never silent.
- **Lost wallet = lost files** — sign-to-derive has no recovery path; a tool
  SHOULD offer additionally sealing to a recovery/recipient key (§6).
- **Password** (§5.2) — defeats a seized wallet only as far as the password's
  entropy; PBKDF2 slows, does not save, a weak password.
- **Metadata** — file *size* and *timing* are visible on whatever channel is
  used; content is not.
- **Symmetric is PQ-safe; the recipient path is not until scheme 0x11 lands.**

## 11. Prior-art anchors

OpenPGP / WKD (the discovery+FS+UX problems being fixed), NaCl `crypto_box` /
ECIES (the sealing construction), HPKE RFC 9180 (the KEM-then-AEAD shape),
HKDF RFC 5869, PBKDF2 RFC 2898, AES-GCM (NIST SP 800-38D), X25519 RFC 7748,
ML-KEM FIPS 203 (the PQ target), RFC 6979 (deterministic ECDSA — the property
that makes sign-to-derive possible), EIP-191 (the signing scheme, shared with
WM-3/WM-7).

## 12. Conformance

`api/vectors/owmf.json` pins, for a fixed test key: the derived OWM identity
public key, a self-encrypted OWMF container that MUST decrypt to a known
plaintext, and an `owm1:` message that MUST open to known text. Every
implementation MUST reproduce the identity key and open both containers. The
reference verifies these in `packages/owm-core/test/owmf-vectors.test.js`.
