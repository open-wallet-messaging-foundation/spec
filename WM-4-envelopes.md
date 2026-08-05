# WM-4-envelopes

**Status:** Draft (2026-07-13). **Profile:** see WM-0.

**Scope:** Envelope base + strict validation; wm-ping/pong/knock; room join policies closed/invited/request/open + concierge; verified sender attestation (.well-known/owm.json + DNS TXT); signed announcements; OWM-INVITE (517 wm-invite addressed, 518 wm-invite-response, 519 wm-intro signed warm introduction — implemented in `@open-wallet-messaging/core` envelope.js/intro.js); OWM-STAGE (540 wm-stage-config, 541 wm-playback-sync, 542 wm-stage-cue — no media-key kind by design, keys derive from the room); invite-link join mode `m=stage`; courier matrix (carrier × bearer/addressed/bound × what the carrier learns); OWM-PAY (543 wm-settlement-card signed CAIP-10 capability card, 544 wm-tx-intent over the closed settlement core {transfer, token-transfer, contract-call, typed-sign, batch}, 545 wm-tx-reference — OWM envelope with a documented mapping to XMTP TransactionReference, 546 wm-broadcast-request signed over receive targets with role-bound rendering; implemented in `@open-wallet-messaging/core` envelope.js/pay.js); OWM-PRESENCE (547 wm-call-attestation mutual presence sets — presence.js)

Content lands during the phase that implements it (see repo README).
