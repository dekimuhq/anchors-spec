# Changelog — `apr-spec`

All notable changes to the APR specification are recorded here.

This repo follows **SemVer** for the spec itself: `MAJOR.MINOR.PATCH`.

- **MAJOR** (`v2`, `v3`, …) — breaking change. Triggers: any change that breaks byte-equality of the canonicalisation, changes the signature scheme, removes fields, narrows an enum, restructures `APRBody`, alters the disclosure-tier semantics, or otherwise prevents a v(N-1)-compliant verifier from producing the v(N-1)-correct answer on a v(N-1)-conforming receipt.
- **MINOR** (`v1.1`, `v1.2`, …) — additive change to the wire format. Permitted: new optional fields on `APRBody`, new values on **closed-enum sets only when the verifier behaviour for unknown values is explicitly specified**, new optional `inputs` metadata, new test vectors, new verification reasons (existing reason strings are stable). MUST be safe for an old verifier to ignore unknown additive fields.
- **PATCH** (`v1.0.1`, `v1.0.2`, …) — clarifications to spec text, typo fixes, additional non-normative examples, additional test vectors that exercise existing behaviour. No wire-format changes.

## LTS policy

- **v1 is LTS.** Once frozen, v1 receives security and clarification patches for **at least 36 months** from freeze date. Verifiers built against v1 will continue to verify v1 receipts for that entire window.
- Any future MAJOR version (e.g. v2) overlaps v1 LTS for at least **12 months** before v1 enters maintenance-only mode.
- "Maintenance-only" means: security fixes only, no spec clarifications, no new test vectors. Verifiers built against v1 continue to verify v1 receipts indefinitely; the spec just stops accepting non-security PRs.

## Backwards-compatibility rule (v1.x)

> **v1.x is additive only.**

Any change that an old v1.0.0 verifier cannot safely process by ignoring is, by definition, a breaking change and ships in v2.

Concretely, a v1.0.0 verifier confronted with a v1.x receipt MUST:

- ignore unknown additive fields on `APRBody` and on `inputs[]` entries,
- reach the same `status` (`verified` / `tampered` / `unverified`) the v1.x verifier would,
- be permitted to surface a different `reasons` set only insofar as new reason strings are unknown to it; existing reason strings MUST continue to mean what v1.0.0 said they meant.

If any of these would fail, the change is not v1.x-eligible.

## Releases

### v1.0.0 — *FROZEN on public flip*

Initial frozen release of APR.

- Wire format: `ar.provenance.v1` Claim envelope + `APRBody` payload.
- Canonicalisation: RFC 8785-lite (UTF-16-sorted keys, no whitespace, NFC strings, no `NaN`/`Infinity`/`undefined`/`bigint`).
- Signature: Ed25519 over `UTF-8(canonicalize(preSignatureObject))`, base64url-encoded (no padding).
- Disclosure tiers: `public_full`, `public_minimal`, `metadata_only`.
- Pseudonymisation: `principalPseudonym = "u_" + sha256_hex(principalId).slice(0, 8)`.
- Daily anchoring: Ed25519-signed `{v:1, date, rootHash}` envelope; signed payload is the canonical encoding of those three fields only.
- Verification result: closed three-value `status` (`verified` / `tampered` / `unverified`) plus open-set `reasons[]`.
- Conformance: published test vectors in `conformance/test-vectors/` covering valid, tampered, missing-from-anchor, rotated-key, and unknown-kid cases.

Pre-freeze clarification (2026-05-11): §2.1 now states explicitly that unknown top-level claim fields are forbidden and produce `tampered` / `schema_invalid`. This tightens v1.0.0 from "ambiguous" to "closed-set"; any future additive fields land via the v1.x additive rule (§10), at which point the known-key set grows accordingly.

The freeze date is the day this repository becomes public on `github.com/dekimuhq/apr-spec`. That date is recorded in the v1.0.0 git tag annotation.
