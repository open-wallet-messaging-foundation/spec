# WM-12-physical — barcodes, QR & the physical-world bridge

**Status:** DRAFT FOR REVIEW (2026-08-04). Nothing frozen; **(open)** marks
decisions still to make. **Profile:** see WM-0.

**Scope:** connect a wallet address to the physical world through **barcodes and
QR codes**, in two distinct roles: (a) a barcode/serial as a **WM-9 subject type**
(bind `address ↔ code`), and (b) a barcode/QR as a **carrier** for a signed OWM
object — an address, a contact card (WM-3), a **ticket/entitlement (WM-11)**, or a
payment request. The turnstile, the product label, the poster, the boarding pass.
Composes WM-9 (subjects), WM-10 (REVEAL as a presentation), WM-3 (invite QR), WM-7
(challenge), WM-11 (tickets). Proposed: WM-9 subject-registry entries + a QR
presentation profile (§ proposed kinds 570–579).

---

## 1. Two roles, one bridge

1. **Barcode as an identifier you bind to** (a WM-9 subject). Your address ↔ a
   product serial, a membership card, a ticket number — signed, revocable, exactly
   like binding to a bank account or a card (WM-9 §3).
2. **Barcode/QR as a transport** for a signed OWM object — scan to *get* the
   address, the contact card, the entitlement, or the payment request into a
   physical touchpoint (a gate, a till, a shelf).

Both are needed; they answer different questions ("what is this code bound to?"
vs "what does scanning this code give me?").

---

## 2. Barcode subject types (WM-9 extension, normative)

Add to the WM-9 subject registry, each `source` or `both` class:

- `barcode` (generic), `qr`, `gs1` / `ean` / `upc` (retail), `serial` (asset).

`subjectRef = sha256(subjectType ‖ ":" ‖ normalise(code))` (WM-9 §5) — the code
value is hashed, not published in the clear; low-entropy codes are confirmable-by-
hash (the WM-9 ceiling). A signed binding says "this address vouches for this
code" — a manufacturer for a product serial, a venue for a ticket number.

---

## 3. Carrier capacity — pointer vs self-contained (normative)

The physics decides the design:

| Carrier | Capacity | Encodes |
|---|---|---|
| **1D barcode** (EAN/UPC/Code-128) | tiny | only an **identifier / URL** that resolves online — the **GS1 Digital Link** pattern (`https://id.example/01/<gtin>` → structured data). A pointer. |
| **QR** | ~KBs | a compact **self-contained signed object** (a REVEAL, a contact card, a payment request) — verifiable **offline**, or a URL+`#t=` fragment (the existing OWM invite pattern). |

So: 1D barcode ⇒ resolve-online pointer; QR ⇒ can stand alone.

---

## 4. QR as a signed presentation — the turnstile (normative + open)

The high-value case: a QR that a scanner **verifies**. This is a **WM-10 REVEAL**
rendered as a QR — a signed presentation of a record (a ticket/entitlement, an
age credential, a membership) bound to context. It connects directly to WM-11: the
stream entitlement you bought is the ticket you present at the gate.

**Anti-replay is the whole game (normative).** A *static* QR/barcode is a
photograph away from replay — a screenshot passed to a friend gets them in. So a
presentation used for access MUST be one of:

- **Challenge-bound** — the scanner emits a nonce; the phone signs a REVEAL over
  it (WM-10 §10b `challenge` + WM-7 auth-challenge). Not replayable; needs
  connectivity or NFC at the gate.
- **Short-TTL / rotating** — the QR refreshes every N seconds from a signed,
  time-boxed token (airline mobile boarding passes, Ticketmaster SafeTix). Works
  offline at the gate; a stale screenshot fails.

A static, long-lived QR is acceptable only for *non-gating* uses (a contact card,
a "learn more" link, a public payment address).

---

## 5. Payment & contact carriers (open)

- **Payment QR** — an OWM payment request (WM settlement intent), with EIP-681
  (`ethereum:`) interop so existing wallets scan it.
- **Contact QR** — a WM-3 signed contact card / SCX invite (`#t=` fragment,
  already OWM's invite pattern). Just the existing invite, printed.

---

## 6. Honest ceilings

- **Static codes are copyable.** Anti-replay (§4) is mandatory for access; without
  it a QR is a bearer token anyone can screenshot.
- **Provenance ≠ physical authenticity.** A signed barcode proves *the brand
  signed this serial* (identifier provenance), not that the atoms in your hand are
  genuine — a clone can carry the same printed code. Binding atoms ↔ code needs a
  physically-unclonable mark (hologram / PUF); crypto alone can't close it.
- **Scanner trust.** Offline verification needs the verifier to hold the trust
  anchors (WM-9 §7); a compromised scanner app is outside OWM's guarantees.
- **1D capacity** forces an online resolve — no offline verification for 1D codes.

---

## 7. Composability (what it consumes / exposes)

**Consumes:** WM-9 (subject types + trust anchors), WM-10 (REVEAL presentation),
WM-3 (invite/contact QR), WM-7 (challenge/auth), WM-11 (ticket entitlements),
WM-8 (pubkey resolution for offline verify).

**Exposes — the reusable seam:** a universal **"scan → verify a signed OWM
object"** capability that any physical touchpoint can adopt: event turnstiles,
retail provenance/anti-counterfeit, building/access control, mDL-style identity
presentation, product registration (bind serial → owner address on purchase),
proof-of-attendance, warranty/repair history bound to a serial. Any place where
atoms meet an address, this is the bridge — including uses we haven't listed,
because the primitive is just "a signed OWM object in a scannable carrier, with
anti-replay when it gates."

---

## 8. Open questions

1. **New kinds or pure profile?** Barcode subjects are WM-9 registry entries; the
   QR presentation may need no new kind (it wraps a REVEAL) — or a small
   `owm-presentation` envelope + a `owm-scan-challenge`. Decide v0 surface.
2. **GS1 Digital Link** — adopt its URL grammar as the 1D-pointer standard?
3. **Rotating-token profile** (§4) — specify the refresh/TTL scheme in v0, or
   reference SafeTix-style and defer?
4. **NFC** — same presentation over NFC tap (mDL/EUDI-style) in scope, or QR-only
   v0?

---

## 9. Prior-art anchors

GS1 Digital Link (barcode/QR → structured web data — the natural partner), ISO
18013-5 mDL & EU Digital Identity Wallet (QR/NFC verifiable presentation, challenge-
bound — *the* incumbent for code↔identity), EIP-681 `ethereum:` payment URIs, Apple
/ Google Wallet passes, IATA BCBP boarding passes, Ticketmaster SafeTix (rotating
barcode), WalletConnect QR (dApp pairing), ISO/IEC 18004 (QR), W3C Verifiable
Presentations (the model REVEAL follows).

---

## 10. Proposed kinds (not yet reserved — held off the live registry)

A `570–579` "physical / presentation" sub-band, if the QR presentation needs
first-class kinds beyond a REVEAL:

```
570  owm-presentation    # a signed OWM object packaged for a scannable carrier
571  owm-scan-challenge   # verifier nonce for challenge-bound scans (§4)
572–579  reserved (physical)
```

Barcode subject types are WM-9 registry entries (no numeric kind). No registry
change until we agree the shape and the tree is clear of the concurrent work.
