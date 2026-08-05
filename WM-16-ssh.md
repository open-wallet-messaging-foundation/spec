# WM-16-ssh — OWM-SSH: hardware-authenticated SSH via the wallet

**Status:** DRAFT (2026-08-04). **Profile:** see WM-0. A *profile*, not a new
primitive — it binds OWM to SSH's existing extension points, adding no new
cryptographic core beyond an SSH-shaped attestation.

**Scope:** let a user open an SSH connection **authenticated by their wallet** —
the wallet being (ideally) hardware, so a touch-to-sign is the presence check a
Yubikey gives. Works with **stock OpenSSH everywhere** (Linux/macOS/BSD/Windows),
not a fork. Composes WM-7 (auth challenge), WM-13 (the certificate = an
attestation), WM-9 grants (principals/scopes), WM-14 (login = a presentation),
sign-to-derive (WM-8).

## 1. The core problem: secp256k1 ≠ SSH crypto (normative)

An EVM wallet signs **secp256k1**. OpenSSH speaks **ed25519 / ecdsa-nistp256/384/
521 / rsa / `sk-*`** — *not* secp256k1. So the wallet **cannot be the SSH key
directly.** OWM-SSH bridges this: the wallet is the **authorising authority**, and
an SSH-native credential (ed25519) is what actually authenticates. Three mechanisms,
below, all honour this.

## 2. Mechanisms

| Mechanism | Wallet's role | Works with stock OpenSSH | Hardware isolation |
|---|---|---|---|
| **A. CA + short-lived certs** (recommended) | Wallet signs a WM-7 challenge → an OWM **SSH CA** verifies it and mints a 60-second ed25519 **user certificate** (a WM-13 `ssh-principal` attestation) | ✓ `TrustedUserCAKeys` | ✓ hardware touch **per login** |
| **B. Sign-to-derive + agent** (simplest) | `personal_sign` a fixed domain → HKDF → a stable ed25519 SSH key; an **OWM ssh-agent** signs each auth | ✓ `authorized_keys` + `SSH_AUTH_SOCK` | ⚠ derived key materialised in software at sign time |
| **C. Wallet-as-FIDO2 `sk-` key** (most Yubikey-literal) | Wallet acts as a CTAP2 authenticator behind OpenSSH `sk-ssh-ed25519` | ✓ 8.2+ `SecurityKeyProvider` | ✓ key never leaves hardware — needs the wallet to be a FIDO authenticator |

**Recommended: A.** Touch-sign a WM-7 challenge on your (hardware) wallet → the CA
issues a short-lived cert → you are in, on any OpenSSH server, with RBAC via
principals, instant revocation, per-login audit, and zero key management.

## 3. Approach to the OpenSSH source (normative): do not fork it

OpenSSH is security-critical and already extensible. We touch **zero** OpenSSH
source and integrate at the sanctioned surfaces:

- **Client** — an **OWM `ssh-agent`** answering the agent protocol over
  `SSH_AUTH_SOCK` (reference home: `rust/owm-ssh-agent`, reusing OWM signing);
  and/or hand the user a certificate via `ssh -i`.
- **Server** — config only: **`TrustedUserCAKeys`** (trust the OWM CA) +
  **`AuthorizedPrincipalsCommand` / `AuthorizedKeysCommand`** to gate a connection
  on live OWM bindings/grants at connect time. No `sshd` changes.
- **FIDO path (C)** — a **`SecurityKeyProvider`** middleware implementing OpenSSH's
  sk-api.

Forking would fragment compatibility and break the "with anyone" promise — the
whole point.

## 4. Mapping to OWM primitives

- **SSH certificate = a WM-13 attestation** (`claimType: ssh-principal`): the CA
  (issuer) attests that a key may log in as principals P until T. Projected into
  OpenSSH's `PROTOCOL.certkeys` format for the wire.
- **SSH login = a WM-14 presentation** of that certificate, challenge-bound by
  SSH's own auth exchange.
- **Principals / hosts / TTL = WM-9 grant scopes** — RBAC over SSH is projected
  grants (`owm.ssh.<host>`), revocable and time-boxed.
- **Issuance authority = WM-7 auth** — the wallet signs the challenge that
  authorises the CA to mint the cert; a hardware wallet's touch is the presence
  check.

## 5. Reference architecture

`rust/owm-ssh-agent` (client daemon) + an **OWM SSH CA** service (verifies a WM-7
wallet auth, mints a short-lived cert, logs issuance to a transparency log per
WM-13 §5). Server operators add two `sshd_config` lines. An individual with no CA
uses mechanism B (agent + derived key) — no service at all.

## 6. Honest ceilings

- secp256k1↔SSH means the wallet is the **authoriser/CA-authority**, never the raw
  SSH key.
- Sign-to-derive (B) materialises the ed25519 key in software at sign time — the
  *authorisation to derive* is hardware-gated, the key is not hardware-isolated
  during the SSH signature. A/C are the stronger hardware stories.
- The CA (A) is a trust root the server must trust — the same posture as Teleport /
  Vault-SSH; short-lived certs + a transparency log (WM-13 §5) mitigate.

## 7. Prior art

OpenSSH `sk-*` keys (8.2+, FIDO2/U2F), PKCS#11 + the ssh-agent & certificate
protocols; SSH-CA systems — Teleport, Smallstep, HashiCorp Vault-SSH, Netflix
BLESS, GitHub SSH certificates; Secretive (Secure-Enclave SSH — the Apple
mechanism, with Touch ID); Ledger's FIDO app as an SSH security key.

## 8. Proposed kinds

`610–619` ssh sub-band (mostly reuses WM-13/14): `610 owm-ssh-cert-request` (a
wallet-signed request to the CA), `611 owm-ssh-cert` (the issued cert = an
attestation profile), `612–619` reserved. The cert itself is a WM-13 attestation;
these kinds are the request/response envelope.
