# Governance — `anchors-spec`

This document describes how the Anchored Receipts specification family evolves. It is short on purpose: the project is run by a small group today and the governance should grow only as fast as the contributor base does.

## Scope

This governance covers only `dekimuhq/anchors-spec`. Companion repos (`apr-verifier`, `apr-issuer-kit`) have their own contributor policies, but cannot land changes that would put them out of conformance with the spec under this governance.

## Roles

### Spec Stewards

A small group with merge rights on `dekimuhq/anchors-spec`. Today this is **Dekimu** (the project's originating org). Stewards are accountable for:

- Maintaining v1's frozen state — refusing breaking changes that would land under v1.x.
- Reviewing and approving spec PRs.
- Deciding when v1.x clarifications, v1.x additive features, or a future v2 ship.
- Liaising on security disclosures (see `SECURITY.md`).

The role is named, not anonymous. Current Stewards will be listed in a `STEWARDS.md` once that file exists; before then, all Steward actions are taken under the Dekimu brand.

### Contributors

Anyone who opens a Discussion, files an issue, or sends a PR. No special status required. DCO sign-off (see `CONTRIBUTING.md`) is the only formal commitment a contributor makes.

### Implementers

Maintainers of verifiers, issuers, and downstream tools. Implementers have no governance role *over* the spec, but Stewards SHOULD consult major implementers before any change that would force a multi-implementation upgrade.

## Decision-making

### Editorial changes (typo fixes, clarifying prose, link updates)

Stewards merge on their own judgement. No discussion required.

### Test vectors

New vectors that exercise existing v1 behaviour land via PR. Vectors that document corner cases of canonicalisation, signature, anchor inclusion, or disclosure are encouraged. Vectors that require a behaviour change in the verifier are not editorial — they go through "v1.x additive" review.

### v1.x additive changes (new optional fields, new test vectors that exercise new behaviour, new reason strings)

1. Open a Discussion describing the change, the motivation, and the affected surface.
2. Allow at least **14 days** for comment. Earlier merge possible if no objection and the change is small.
3. A Steward decides. If two or more Stewards exist, decisions are by lazy consensus: silence = consent; an explicit "block" from any Steward holds the change until resolved.
4. Land via PR. Update `CHANGELOG.md`.

### v1.x clarifications that change observed verifier behaviour

These are the hardest decisions: text-only spec changes that re-interpret what conforming verifiers do. They go through the same Discussion → 14 days → Steward consensus path, with one additional constraint: at least one v1-conforming verifier other than `apr-verifier` should weigh in if such a verifier exists.

### v2

A v2 conversation starts with a Discussion thread, not a PR. The thread documents:

- What v1 cannot express, and why retrofitting v1.x is insufficient.
- A migration path: how v1 receipts continue to verify under v2, and for how long.
- Which Stewards are willing to maintain v1 LTS during the overlap window.

There is no fixed timeline. v2 lands when consensus exists; until then v1 is the spec.

## Code of conduct

Governance discussions are subject to the [Contributor Covenant 2.1](./CODE_OF_CONDUCT.md). Code-of-conduct reports do not go through this governance — they go to **conduct@dekimu.com** and are handled separately.

## Conflict of interest

Stewards employed by, contracted to, or financially aligned with an organisation building on APR SHOULD disclose the relationship when reviewing a PR or Discussion that materially affects that organisation. The disclosure happens in-thread; no recusal is required unless the conflict is severe enough that other Stewards request it.

This is honest record-keeping, not a corporate process.

## Steward succession

A Steward may step down at any time by editing `STEWARDS.md` (once that file exists). New Stewards are added by unanimous consent of existing Stewards.

If at any point there are no active Stewards for **90 consecutive days** (no merges, no Discussion replies, no security responses), the project is considered dormant. Any prior Steward — or any contributor who has merged at least three substantive PRs — may step in as a Steward by opening a Discussion and waiting 14 days for objections. This is the explicit fallback so the spec does not die quietly if Dekimu disappears.

## RFC process

Substantial spec changes — new envelope fields, new cryptographic primitives, new error codes — are proposed as numbered RFCs in the [`rfcs/`](./rfcs/) directory.

### Filing an RFC

1. Copy [`rfcs/RFC-TEMPLATE.md`](./rfcs/RFC-TEMPLATE.md) to `rfcs/RFC-NNN-short-title.md` using the next available number.
2. Open a PR against `dekimuhq/anchors-spec` with the RFC file.
3. The PR enters a **30-day comment period** from the date it is opened. Stewards may shorten this to 14 days for changes with narrow scope and no objection.
4. During the comment period, anyone may review: Contributors, Implementers, and the public.
5. After the comment period, Stewards decide: **accept**, **request changes**, or **reject**. Lazy consensus applies (see Decision-making above).
6. Accepted RFCs are merged. The RFC status field is updated to "Accepted" and the change is scheduled for the next envelope amendment (see Quarterly amendment cadence below).

RFCs that affect only profiles or new family members do not need the RFC process — they follow the existing v1.x additive path. The RFC process is for envelope-level changes that touch the shared wire format, canonicalization, signature verification, or error-code vocabulary.

### Quarterly amendment cadence

Envelope amendments batch on a quarterly cycle:

| Quarter | Amendment | Example scope |
|---------|-----------|---------------|
| Q3 2026 | v1.3 | `sig_alg` discriminator (RFC-001) |
| Q4 2026 | v1.4 | TBD |
| Q1 2027 | v1.5 | TBD |

**Profiles and new family members ship continuously** — they are not gated by the quarterly cycle. Only envelope-level wire-format changes (new fields, new error codes, canonicalization amendments) batch quarterly.

The quarterly cadence exists to give implementers a predictable upgrade rhythm. Stewards may issue out-of-cycle amendments for security-critical fixes.

## Evolution of this document

This governance file evolves with the project. Material changes go through the same v1.x process described above. Editorial fixes land freely.

The intent is that this file gets larger only as the contributor base requires it to — small projects deserve small governance.
