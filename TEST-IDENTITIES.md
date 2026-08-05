# OWM reserved test identities

**Status:** Reference (2026-08-04). Normative data: `../api/test-identities.json`.

The networking world reserves `example.com` (RFC 2606) and the `192.0.2.0/24`
TEST-NET blocks (RFC 5737) so documentation never accidentally points at a real
host, and toolchains ship a well-known dev mnemonic (Hardhat's
*"test test … junk"*) so a funded address is never mistaken for a burner. OWM
needs the same hygiene, adapted to the fact that **crypto addresses are hashes of
keys, not ranges you can allocate.**

This document defines OWM's answer: a small, fixed set of **reserved test
identities** whose private keys are **public on purpose**. If an address in this
file shows up anywhere — a conformance vector, a spec example, a demo — it is
*unambiguously* a test identity. No one should ever fund it, and no conformant
tool should ever trust a production signature from it.

## The one rule

> **Every private key here is public. Never use these in production.**
> Conformant tooling SHOULD refuse — or at minimum loudly warn — if a production
> flow signs with, or trusts a signature from, any listed identity. (The same
> guardrail wallets already show for the Hardhat key.)

## Derivation (reproducible)

```
privateKey = keccak256( utf8( "owm-test:" + name ) )        # 32 bytes, hex, no 0x
address    = EIP-55 checksum( keccak256( secp256k1_pubkey )[-20:] )
```

Anyone can regenerate the roster from the names alone and confirm the pinned
addresses byte-for-byte — there is no secret and nothing to trust.

## The roster

| Name | Address | Role in examples |
|---|---|---|
| **alice** | `0x547A30497Eeba91C44bE51d264dEf069d7A2E087` | primary user / record owner (WM-10 store subject) |
| **bob** | `0xcbaF7eEC94C0ED080BC3C93DDf50eb00C4892EEb` | counterparty / friend (WM-3 peer, WM-10 §10b group member) |
| **carol** | `0x1236068191169279083c66F2AA211772caCD354d` | third party / trusted group member |
| **dave** | `0x0F9dFeAfBf0Aa3Bd4719aA2008443213fF66e5f1` | **untrusted** party (WM-10 §10b mixed-trust group) |
| **bank** | `0x5583E2e5714a5cb657956aEa2Cc6FE5C083415aB` | institution: source binding + verified sender (WM-7 §10, WM-9) |
| **issuer** | `0x39485c043084bEe0C6D58A061FCb99c5eB6b70F9` | credential issuer (e.g. `university.edu`) for WM-10 REVEAL/VC |
| **mother** | `0x78dD72452d8A71f366ce5A2ec02C3EF4066E1E62` | recovery delegate — read grant on `recovery` (WM-10 §5/§13) |
| **accountant** | `0x45738a4D1Fd612a2531b3E92cfa3bECA48B2E2Ee` | read-only financial delegate — `{get,list}` on `financial` |

The private keys live in `../api/test-identities.json`.

## Loopback — the `127.0.0.1` of OWM

You can't reserve an address *range* the way IANA reserves `127.0.0.0/8`, because
addresses are hashes, not allocations. The useful loopback in a *messaging* system
isn't a range anyway — it's an **echo endpoint**:

| | Reserved | Purpose |
|---|---|---|
| **echo** | `0x82359cD8017421f39112AfD0a97393188d581428` | a responder that pongs (WM-4 500/501) everything sent to it on the XMTP `dev` network — send-then-receive validation of any client end to end |

The echo **address is reserved now**; the bot itself is a small hosted service to
stand up later (same shape as the notifier). Its key is public like the rest, so
anyone can run their own echo at the same identity.

Two other loopbacks already exist and need no address at all:
- **`./scripts/run_all.sh --offline`** — pure-logic, compute-only loopback (no wire).
- **WM-10 "self"** — the address-is-the-store self-conversation *is* a loopback to
  yourself.

## Sentinels & the zero address

The only address worth reserving as a structured sentinel is the all-zero one —
safe precisely because no one can find a key whose pubkey hashes to it:

> **`0x0000…0000` is never a valid OWM identity.** Reject it as a sender or
> recipient at validation — the `0.0.0.0` of OWM.

Beyond that, structured sentinel *blocks* are unnecessary: a random 160-bit
address colliding with a real funded wallet is astronomically improbable, so
documentation examples don't need a reserved block to stay safe — they need the
*public-key* guarantee this file provides.

## Why no "private range" like RFC 1918

`10.0.0.0/8` is private because routers agree not to route it. Nothing routes
crypto addresses by prefix, and any address can hold real value, so a
"private/test range" has no meaning here. The equivalent guarantee — "this
identity is not, and cannot become, a real user" — comes from **publishing the
private key**, which is exactly what this roster does.
