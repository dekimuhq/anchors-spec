# Anchored Delegation Receipts (ADR)

Wire type: `ar.delegation.v1`

Records delegation of authority and scope. Six event types: `delegation.granted`, `delegation.revoked`, `delegation.suspended`, `delegation.reinstated`, `delegation.scope_amended`, `delegation.expired`. Designed for GDPR Art. 28 (processor), AI Act Arts. 14/26 (deployer/overseer).

## Reference Implementation

- Verifier: [`@dekimuhq/adr-verifier`](https://github.com/dekimuhq/adr-verifier)
- Issuer kit: [`@dekimuhq/adr-issuer-kit`](https://github.com/dekimuhq/adr-issuer-kit)
- Profiles: [`@dekimuhq/adr-profiles`](https://github.com/dekimuhq/adr-profiles)

## Profile specs

- [`gdpr.dsr.authorised-agent`](profiles/gdpr.dsr.authorised-agent.md) — a representative acting for a data subject against one named controller.

## Status

Spec: profile `gdpr.dsr.authorised-agent` is specified (see above); the rest of the family, including the six event types and the other profiles, is a placeholder — full spec pending.
Reference implementation: shipped (v0.1.x on GitHub Packages).
