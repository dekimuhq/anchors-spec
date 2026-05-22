# Anchored Tokenization Receipts (ATokR)

Wire type: `ar.tokenization.v1`

Records PII tokenization lifecycle: token issued, redeemed, revoked, rotated. Salted commit only — no plaintext PII. Scope-binding with audience, purpose, and validity constraints. PCI HVT/LVT vocabulary mapping.

## Reference Implementation

- Verifier: [`@dekimuhq/atokr-verifier`](https://github.com/dekimuhq/atokr-verifier)
- Issuer kit: [`@dekimuhq/atokr-issuer-kit`](https://github.com/dekimuhq/atokr-issuer-kit)
- Profiles: [`@dekimuhq/atokr-profiles`](https://github.com/dekimuhq/atokr-profiles)

## Status

Spec: placeholder — full spec pending.
Reference implementation: shipped (v0.1.x on GitHub Packages).
