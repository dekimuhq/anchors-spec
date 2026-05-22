# Anchored Consent Receipts (ACR)

Wire types: `ar.consent.v1`, `ar.withdrawal.v1`, `ar.modification.v1`

Records consent given, withdrawn, or modified by a data subject. Designed for GDPR Art. 7, AI Act Art. 14, and HIPAA §164.508 compliance workflows.

- `ar.consent.v1` — consent given by a data subject
- `ar.withdrawal.v1` — consent withdrawn (GDPR Art. 7(3))
- `ar.modification.v1` — consent scope or purpose modified (GDPR Art. 7(4), Art. 12(3))

## Reference Implementation

- Verifier: [`@dekimuhq/acr-verifier`](https://github.com/dekimuhq/acr-verifier)
- Issuer kit: [`@dekimuhq/acr-issuer-kit`](https://github.com/dekimuhq/acr-issuer-kit)
- Profiles: [`@dekimuhq/acr-profiles`](https://github.com/dekimuhq/acr-profiles)

## Spec Documents

- [PRIVACY-REVIEW.md](PRIVACY-REVIEW.md) — GDPR review of consent receipt data
- [THREAT-MODEL.md](THREAT-MODEL.md) — defended and undefended threats

## Status

Spec: partial (threat model + privacy review shipped; full wire format spec pending).
Reference implementation: shipped (v0.5.x on GitHub Packages).
