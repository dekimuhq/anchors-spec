# Proof of Forgetting — `forget.*` companion receipts

Wire kinds: `forget.roster.v1` · `forget.destruction.v1` · `forget.model-disposition.v1`

**Not an `ar.<noun>.v<N>` Anchored Receipts family.** These are standalone Ed25519-signed, by-value receipts that make a GDPR Art. 17 erasure checkable: a signed destruction receipt (the subject's data-encryption key was destroyed, moving the issuer's roster from signed root *R₀* to *R₁*) paired with a Merkle non-inclusion proof against the published, signed roster *R₁* — plus an optional model-disposition receipt for data that may have been used to train a model.

- **Specification:** [SPEC.md](SPEC.md) — wire shapes, canonicalisation, the interval-leaf roster, the verification algorithms, what the public verifier checks, security considerations, changelog.
- **Examples:** [examples/](examples/) — one real fixture per receipt kind, generated and verified by the reference implementation (test keys only).
- **Verify a destruction receipt live:** [verify.dekimu.com](https://verify.dekimu.com) (signature check against the public key catalogue; roster and model-disposition checks are offline, per SPEC §6).
- **Family cross-references:** [`ar.erasure.v1` ↔ `forget.destruction.v1` ↔ `ar.subject_rights.v1`](../CROSS-REFERENCES.md) and the [registry](../registry.md) "Companion receipt specs" section.

## Status

Spec: v1, wire shapes frozen 2026-10-19.
Reference implementation: [`dekimuhq/forget`](https://github.com/dekimuhq/forget) (Apache-2.0) — `@dekimuhq/forget-roster`, `@dekimuhq/forget-keystore`, `@dekimuhq/forget-receipt` on npm. First issuer is Dekimu Hub.

Spec licence: CC0 1.0 Universal. This is a technical specification, not legal advice.
