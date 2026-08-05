# OWM specification documents

All Draft as of 2026-07-13. Working notes live with the reference
implementation; these documents are the normative surface.

| Doc | Scope |
|---|---|
| [WM-0](WM-0-charter.md) | Charter, governance, conformance profiles (CORE / PRIVATE / NOTIFY) |
| [WM-1](WM-1-identity.md) | Identity: CAIP-10 accounts, inbox binding, anonymous HD sub-identities, guest identities, contact cards |
| [WM-2](WM-2-relays.md) | Blind relay network: OHTTP relay/gateway split, rendezvous mailboxes, relay directory |
| [WM-3](WM-3-scx.md) | Secure Contact Exchange: PAKE (SPAKE2), transcript-bound signed cards, SAS |
| [WM-4](WM-4-envelopes.md) | Envelopes & primitives: ping/knock, rooms & join policies, verified senders & signed announcements |
| [WM-5](WM-5-registry.md) | The kind registry (living) — normative copy in ../api/kinds.json |
| [WM-6](WM-6-notifications.md) | Notification bridge: sink model (Class D/C), knock contract, screening interplay |
| [WM-7](WM-7-auth.md) | OWM-AUTH / OWM-GRANT: wallet 2FA + step-up approval (530/531), sign-in and capability grants (532–534), canonical signing strings, OIDC bridge + native session profiles, EIP-4361 interop |
| [WM-8](WM-8-encrypted-files.md) | Encrypted files (OWMF container), sign-to-derive file & identity keys, recipient encryption + `owm1:` messaging over any channel, filename privacy, address→public-key resolution (524–526) |
| [WM-9](WM-9-bindings.md) | Bindings: wallet address ↔ any typed identifier (SMS/email/Telegram/X/bank/card/wallet), capability set, accept posture (Everything/Secure/Verified), Web-PKI trust anchors, delegation/RBAC, `message/{address}/{what}` API (527–528) |
| [WM-10](WM-10-kv.md) | OWM-KV: wallet-native personal record store — address-is-the-store, `SET/GET/DELETE/LIST/REVEAL` with per-verb capabilities, signed hash-chain (non-tamper), delegation, mixed-trust group disclosure (550–555). **Draft for review.** |
| [WM-11](WM-11-cast.md) | OWM-CAST: paid 1:many broadcast / DRM entitlement layer — cast=MLS group, ticket=membership, encryption=group key; forward-secret revocation; signed-transfer resale (anti-scalping); entitlement/key layer, not the CDN (560–569 proposed). **Draft.** |
| [WM-12](WM-12-physical.md) | Physical bridge: barcodes/QR — barcode as a WM-9 subject + QR as a signed presentation (turnstile), anti-replay (challenge/rotating), GS1 Digital Link (570–579 proposed). **Draft.** |
| [WM-13](WM-13-attestation.md) | **OWM-ATTEST** (first-class): an issuer signs a typed claim about a subject, trust-anchored, transparency-logged — OWM-native Verifiable Credentials; generalises verified-sender/binding-attest/KYC/SSH-CA/entitlements (580). **Draft.** |
| [WM-14](WM-14-presentation.md) | **OWM-PRESENT** (first-class): prove a signed object to an audience, challenge-bound (anti-replay) — REVEAL lifted out of KV; the turnstile/credential-verify verb (590). **Draft.** |
| [WM-15](WM-15-transfer.md) | **OWM-XFER** (first-class): hand a bound object to a new holder under issuer policy — CAS/fold provenance; ticket resale, asset title, licence reassignment (600). **Draft.** |
| [WM-16](WM-16-ssh.md) | OWM-SSH: hardware-authenticated SSH via the wallet — CA+short-lived-certs (cert=attestation), sign-to-derive+agent, FIDO `sk-`; no OpenSSH fork (610–619 proposed). **Draft.** |

Reference (non-normative): [PRIMITIVES](PRIMITIVES.md) — the complete list of
composable primitives (the pantry). [FIRST_CLASS_PRIMITIVES](FIRST_CLASS_PRIMITIVES.md) —
why Attestation · Presentation · Transfer are first-class (the composition model).
[TEST-IDENTITIES](TEST-IDENTITIES.md) — reserved test identities (the RFC 2606 /
TEST-NET of OWM), data in `../api/test-identities.json`.
