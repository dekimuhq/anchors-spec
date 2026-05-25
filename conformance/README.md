# Conformance Test Vectors

Machine-readable test vectors for validating Anchored Receipts verifier implementations.

## Directory structure

```
conformance/
  valid/              Correctly signed receipts that MUST verify
    apr-provenance-v1.json
    acr-consent-v1.json
    arr-erasure-v1.json
  invalid/            Intentionally malformed receipts that MUST be rejected
    sig-invalid.json
    claim-type-unknown.json
    iat-ms-mismatch.json
    predicate-operator-not-allowed.json
```

## Vector format

Each JSON file follows this schema:

```jsonc
{
  // Human-readable description of what this vector tests
  "description": "...",

  // The complete v=2 envelope receipt (signed, with anchor stub)
  "receipt": { /* Envelope */ },

  // Base64url-encoded 32-byte Ed25519 public key for signature verification
  "issuer_public_key": "...",

  // For valid vectors — expected verification outcome:
  "expected_result": {
    "signature": "valid",
    "claim_type": "ar.provenance.v1"
  },

  // For invalid vectors — expected error category:
  "expected_error": "sig_invalid"
}
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `description` | string | What the vector tests |
| `receipt` | object | Complete v=2 `Envelope` object |
| `issuer_public_key` | string | Base64url-encoded raw 32-byte Ed25519 public key |
| `expected_result` | object | Present on valid vectors. `signature` is `"valid"`, `claim_type` matches the receipt |
| `expected_error` | string | Present on invalid vectors. Error category (see table below) |

### Error categories

| `expected_error` | Meaning |
|-----------------|---------|
| `sig_invalid` | Ed25519 signature does not verify against the provided public key |
| `claim_type_unknown` | `claim_type` is not a registered discriminator |
| `iat_ms_mismatch` | `iat_ms` does not equal `Date.parse(iat)` |
| `predicate_operator_not_allowed` | Predicate AST contains an operator outside `JSON_LOGIC_ALLOWLIST_V1_2` |

## Issuer key

All vectors use a deterministic Ed25519 key derived from a 32-byte seed of `0xAA`:

- **Seed:** `Buffer.alloc(32, 0xaa)` (hex: `aa` repeated 32 times)
- **Public key (base64url):** included in each vector's `issuer_public_key` field

## How to use these vectors

### 1. Load the vector JSON

Parse the file and extract `receipt`, `issuer_public_key`, and either `expected_result` or `expected_error`.

### 2. Derive the DER-encoded public key

The `issuer_public_key` is a raw 32-byte Ed25519 public key, base64url-encoded. To use it with the `verifyEnvelopeSignature` function, wrap it in SPKI DER:

```typescript
const SPKI_PREFIX = Buffer.from("302a300506032b6570032100", "hex");
const rawKey = base64UrlDecode(vector.issuer_public_key); // 32 bytes
const derKey = new Uint8Array(Buffer.concat([SPKI_PREFIX, Buffer.from(rawKey)]));
```

### 3. Verify

For valid vectors, assert that the signature verifies and the `claim_type` matches.

For invalid vectors, assert that verification fails with the appropriate error.

## Regenerating vectors

From the `anchors/` workspace root:

```bash
node --experimental-strip-types scripts/generate-conformance-vectors.ts
```

The script uses a fixed Ed25519 seed, so output is deterministic. Re-running produces byte-identical files.

## Envelope version

All vectors use the v=2 shared envelope format. The `anchor` field contains a stub proof (not a real Merkle inclusion) since conformance vectors test signature and schema validation, not anchor-log integrity.

## License

CC0 1.0 Universal. See the repository LICENSE file.
