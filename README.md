# Anchored Receipts — Specification

Specification documents for the **Anchored Receipts** protocol family: 14 signed, Merkle-anchored receipt formats for provenance, consent, retention, lineage, and compliance events.

**Spec licence:** CC0 1.0 Universal (public domain). Use it, fork it, embed it.
**Reference implementations:** Apache 2.0, under [`dekimuhq`](https://github.com/dekimuhq) on GitHub.

## Family Members

| Acronym | Wire type | Name | Spec status |
|---------|-----------|------|-------------|
| APR | `ar.provenance.v1` | Agent Provenance Receipts | [Full spec](apr/v1.md) |
| ACR | `ar.consent.v1` / `ar.withdrawal.v1` / `ar.modification.v1` | Anchored Consent Receipts | [Partial](acr/) |
| ARR | `ar.erasure.v1` / `ar.expiry.v1` / `ar.hold.v1` / `ar.anonymization.v1` / `ar.archive.v1` / `ar.batch.v1` | Anchored Retention Receipts | [Placeholder](arr/) |
| ALR | `ar.lineage.v1` | Anchored Lineage Receipts | [Placeholder](alr/) |
| ANR | `ar.notice.v1` | Anchored Notice Receipts | [Placeholder](anr/) |
| ASR | `ar.subject_rights.v1` | Anchored Subject-Rights Receipts | [Placeholder](asr/) |
| ABR | `ar.breach.v1` | Anchored Breach Receipts | [Placeholder](abr/) |
| AIR | `ar.impact.v1` | Anchored Impact Receipts | [Placeholder](air/) |
| ATR | `ar.transfer.v1` | Anchored Transfer Receipts | [Placeholder](atr/) |
| APuR | `ar.purpose.v1` | Anchored Purpose Receipts | [Placeholder](apur/) |
| AER | `ar.evaluation.v1` | Anchored Evaluation Receipts | [Placeholder](aer/) |
| AAR | `ar.attestation.v1` | Anchored Attestation Receipts | [Placeholder](aar/) |
| ADR | `ar.delegation.v1` | Anchored Delegation Receipts | [Placeholder](adr/) |
| ATokR | `ar.tokenization.v1` | Anchored Tokenization Receipts | [Placeholder](atokr/) |

## Cross-Family Documents

- [Registry](registry.md) — canonical `claim_type` discriminator registry
- [Family Consistency](family-consistency.md) — cross-family invariants (chip assignments, TSA upgrade paths, subject presence, package versions)
- [Envelope](envelope/) — shared wire format (v1 + v2)

## Wire Format

All members use canonical `ar.<noun>.v<N>` form (e.g., `ar.provenance.v1`, `ar.consent.v1`). The `ar.` namespace identifies the family at the wire level; member acronyms (APR, ACR, ARR, etc.) are documentation shorthand only.

New fields are additive and optional. Breaking changes require a new `v<N+1>` discriminator.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for how to propose spec changes, the DCO requirement, and the v1 stability rule.

## Governance

See [GOVERNANCE.md](GOVERNANCE.md) for decision-making, the Steward role, and the amendment process.

## Security

To report a vulnerability in the spec or any reference implementation, see [SECURITY.md](SECURITY.md).

## License

All specification documents in this repository are released under [CC0 1.0 Universal](LICENSE).
Reference implementations are licensed separately under Apache 2.0.
