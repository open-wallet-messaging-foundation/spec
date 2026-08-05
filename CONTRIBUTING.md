# Contributing to the OWM specification

Thank you — a standard is only as good as the people who read it closely
enough to find its holes.

This repository holds the normative OWM specification documents
(WM-0 … WM-9). Different parts carry different change authority, and this
page says exactly which is which, so you never spend an evening on a PR
that was never mergeable.

## Who may change what

| What you want to change | Who can approve it | How to start |
|---|---|---|
| **Typos, grammar, broken links, unclear wording** | any maintainer | just open a PR |
| **Examples, diagrams, added explanation** that do not change requirements | any maintainer | just open a PR |
| **Normative text** — anything that changes what a conforming implementation MUST/SHOULD do | a spec editor, after working-group discussion | open an **issue** first; land the discussion, then the PR |
| **A new kind (message type) in the OWM range** | spec editors + registry maintainer | issue → spec text → registry PR in [`owm-api`](https://github.com/open-wallet-messaging-foundation/owm-api) |
| **A vendor kind** (1000+, `vnd.` prefix) | self-service | PR straight to `kinds.json` in `owm-api` — no permission needed |
| **WM-0 (charter & governance)** | the foundation's steering group | governance process, not a PR |
| **A new RFC draft** (e.g. a new chain rail) | anyone may propose and edit | PR adding the draft; say in it that you are volunteering as editor |

The rule of thumb: **a change to what implementations must do needs an
editor; a change that makes the same requirement clearer does not.**

## Drafts seeking editors

Several RFCs are drafted and explicitly looking for someone to own them —
non-EVM rails in particular. If you know a chain well, claiming one of
these is the highest-leverage contribution available right now. See the
RFC index on [owm.foundation](https://owm.foundation/#/rfcs).

## House rules that PRs are held to

These are not stylistic preferences; they are the properties the standard
exists to provide. A PR that weakens one will be closed with a link here.

- **Strict validation.** Envelopes reject missing, extra, and
  type-mismatched keys. Unknown *kinds* fall back to readable text and are
  never silently dropped — those are different things and both matter.
- **Invite tokens ride the URL fragment only.** A token in a query string
  reaches servers and access logs. This is refused at parse, by design.
- **State the ceiling.** Every claim in this spec names what it does *not*
  give you. If your PR adds a capability, add its limits in the same
  breath. "We haven't tested this" is a perfectly good sentence in a
  standard; a claim that outruns the evidence is not.
- **No normative dependency on a private document.** If you need to cite
  something, cite a public WM document or write the sentence here.

## Filing a good issue

Say what you were trying to implement, which document and section, what
the text made you do, and what went wrong. An issue that ends with "so
either the text or my reading is wrong, and here is the ambiguity" is
worth more than a patch.

## Who signs for a commit

Every commit in an Open Wallet Messaging repository is authored by an
**identified individual** who takes responsibility for it. Not an anonymous
handle, not a shared organisation mailbox, and not an AI.

Practically, that means each commit carries a Developer Certificate of
Origin sign-off with your real name and a working address:

```
Signed-off-by: Your Name <you@example.com>
```

`git commit -s` adds it for you. By adding it you assert the
[Developer Certificate of Origin](https://developercertificate.org) — in
short, that you wrote the contribution or otherwise have the right to
submit it under this repository's licence.

**On tooling and AI assistance:** use whatever helps you write good code —
compilers, linters, language models. Disclose it if you like; several of us
do, in the commit body. What is not negotiable is that a named human has
read the change, understood it, and is accountable for it. An assistant can
draft a commit; it cannot be responsible for one. In a project whose entire
value is that signatures mean something, authorship that means nothing
would be a strange place to start.

## Security

Do not open a public issue for a vulnerability in the protocol or a
reference implementation. See [SECURITY.md](SECURITY.md).

## Licence

Contributions are accepted under [CC BY 4.0](LICENSE), the licence of this
repository. By opening a PR you confirm you can license your contribution
under it.
