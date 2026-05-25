# RFC-NNN: [Title]

**Authors:** [name(s)]
**Status:** Draft | Under Review | Accepted | Rejected | Withdrawn
**Created:** YYYY-MM-DD
**Target version:** v1.x (envelope) | n/a (profile/family)

---

## Motivation

Why this change is needed. What problem exists today, and who encounters it.

## Proposed Changes

### Wire format

Fields added, modified, or deprecated. Include TypeScript type signatures where applicable.

### Verification

How verifiers handle the new surface. Include error codes if any are introduced.

### Issuer

How issuers populate the new surface. Defaults, constraints, validation rules.

## Backward Compatibility

How existing receipts behave under this change. Explicitly state whether pre-change receipts verify unchanged.

## Security Considerations

New attack surface, mitigations, and threat-model entries this change introduces or modifies.

## Test Vectors

Describe the vectors that MUST ship alongside this change. Vectors land in the relevant `conformance/` directory.

## Implementation Notes

Optional guidance for implementers: migration paths, feature-detection, recommended rollout order.

## References

Links to standards, prior art, related RFCs, and Discussion threads.

---

## Process

1. Copy this template to `rfcs/RFC-NNN-short-title.md` (next available number).
2. Open a PR against `dekimuhq/anchors-spec` with the RFC file.
3. The PR enters a **30-day comment period** from the date it is opened.
4. Stewards review and decide: accept, request changes, or reject.
5. Accepted RFCs are merged. Implementation follows in companion repos.

See [GOVERNANCE.md](../GOVERNANCE.md) for the full RFC process and amendment cadence.
