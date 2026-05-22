# Anchored Delegation Receipts (ADR)

Wire type: `ar.delegation.v1`

Records delegation of authority and scope. Six event types: `delegation.granted`, `delegation.revoked`, `delegation.suspended`, `delegation.reinstated`, `delegation.scope_amended`, `delegation.expired`. Designed for GDPR Art. 28 (processor), AI Act Arts. 14/26 (deployer/overseer).

## Reference Implementation

- Verifier: [`@dekimuhq/adr-verifier`](https://github.com/dekimuhq/adr-verifier)
- Issuer kit: [`@dekimuhq/adr-issuer-kit`](https://github.com/dekimuhq/adr-issuer-kit)
- Profiles: [`@dekimuhq/adr-profiles`](https://github.com/dekimuhq/adr-profiles)

## Status

Spec: placeholder — full spec pending.
Reference implementation: shipped (v0.1.x on GitHub Packages).
