# Anchored Receipts — Envelope Wire Format

The shared envelope is the outer structure wrapping every Anchored Receipt. Member-specific content lives in the `body` field; the envelope handles signing, anchoring, timestamping, and status.

## Versions

### v=1 (legacy)

Single-protocol shape. All existing v1 receipts verify byte-equal. No v1 paths are modified by v2's introduction.

### v=2 (current)

Shared envelope discriminated by `claim_type`. Reference implementation: [`@dekimuhq/anchors-envelope`](https://github.com/dekimuhq/anchors-envelope).

Top-level fields:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `v` | `2` | yes | Envelope version discriminator |
| `id` | `string` | yes | ULID receipt identifier |
| `claim_type` | `string` | yes | Wire-format discriminator (`ar.<noun>.v<N>`) |
| `field_root` | `string` | yes | Top-level body field name (e.g., `provenance`, `consent`) |
| `iss` | `string` | yes | Issuer DID (`did:web:...`) |
| `iat` | `string` | yes | ISO 8601 issuance timestamp |
| `kid` | `string` | yes | Key identifier for signature verification |
| `sig` | `string` | yes | Ed25519 signature (base64url, no padding) |
| `body` | `object` | yes | Member-specific payload |
| `subject` | `string \| null` | per-member | Data subject pseudonym (`sha256:<hex>`) or `null` |
| `prev` | `string \| null` | no | Previous receipt ID in a chain |
| `tsa` | `object \| null` | no | RFC 3161 timestamp authority evidence |
| `credentialStatus` | `object \| null` | no | W3C Bitstring Status List 2021 pointer |
| `suspensionStatus` | `object \| null` | no | Suspension status pointer (Status List Wave 3) |
| `policy` | `object \| null` | no | JSON Logic policy block |
| `kind` | `string \| null` | no | v1.2: `observation`, `decision`, or `null` |
| `cause` | `string \| null` | no | v1.2: event trigger enum |
| `predicate` | `object \| null` | no | v1.2: JSON Logic AST |
| `iat_ms` | `number \| null` | no | v1.2: millisecond-precision issuance timestamp |

## Cryptographic Primitives

- **Canonicalization:** RFC 8785 (JSON Canonicalization Scheme). Single source implementation.
- **Signature:** Ed25519 over `UTF-8(canonicalize(preSignatureObject))`, base64url-encoded (no padding).
- **Anchoring:** Daily Merkle root (RFC 6962 leaf/node prefixing). Ed25519-signed root envelope.
- **Hashing:** SHA-256 only (`node:crypto`). No third-party crypto dependencies.

## Verifier Dispatch

Verifiers auto-dispatch on the `v` field. Callers do not specify a version. A v=1 receipt is handled by the legacy path; v=2 receipts route to the `claim_type`-specific verifier.

## Cross-Family Invariants

See [family-consistency.md](../family-consistency.md) for chip assignments, TSA upgrade paths, subject presence rules, and package version coordination.
