# Known-Answer Test Vectors (KATs)

Machine-readable test vectors for the shared Anchored Receipts **v=2 envelope**,
minted by the reference implementation. They let an independent verifier
implementation check its signature, canonicalization, temporal, status, and
commitment logic against known answers.

These vectors are **illustrative fixtures**, not a conformance certification and
not a security assurance. See [`../threat-model.md`](../threat-model.md) and
[`../verifier-conformance.md`](../verifier-conformance.md).

## Test keys — throwaway only

All signing material here is **throwaway and deterministic**, generated solely
for these fixtures:

- **Ed25519 seed:** 32 bytes of `0xAA` (hex `aa`×32). Public key is embedded in
  each Ed25519 vector as `issuer_public_key` (base64url raw 32 bytes).
- **Salt fixture:** 32 bytes of `0xBB`, base64url-encoded.
- **ML-DSA-65 keypair:** freshly generated per run; the raw public key is
  embedded. No seed or private key is published.

No vector uses or embeds any real issuer key, DID, or environment secret. Issuer
identity throughout is the placeholder `did:web:example.com`.

## Vector schema

```jsonc
{
  "description": "what this vector tests",
  "sig_alg": "ed25519 | ml-dsa-65",     // present on signed vectors
  "receipt": { /* full v=2 envelope */ }, // present on signed vectors
  "issuer_public_key": "base64url",       // ed25519 raw 32-byte public key
  "issuer_public_key_raw_base64url": "…", // ml-dsa-65 raw public key
  "verify_at": "ISO-8601",                // temporal vectors: the eval instant
  "status_list_fixture": { … },           // status vectors: decoded bitstring
  "input": { … },                         // fixture vectors (JCS / commit)
  "expected": { … }                       // expected verifier outcome
}
```

## Vectors and expected verifier behaviour

| File | Tests | Expected verifier behaviour |
|---|---|---|
| [`01-valid-ed25519.json`](./01-valid-ed25519.json) | Valid `ar.provenance.v1`, Ed25519 over RFC 8785 canonical bytes | Signature verifies. **Still not an accept on its own** — the verifier must proceed to trust/status/anchor checks. |
| [`02-broken-sig.json`](./02-broken-sig.json) | One bit flipped in the signature | **Reject** — `signature_invalid`. |
| [`03-expired.json`](./03-expired.json) | Signature-valid, `exp` before `verify_at` | Signature verifies, **reject on `expired`**. Expiry checked after and independently of the signature. |
| [`04-revoked-status.json`](./04-revoked-status.json) | Signature-valid, `credentialStatus` index set to 1 in the status list | Signature verifies, **reject on `revoked`**. Includes a self-contained `status_list_fixture` (decoded bit array) standing in for the fetched Status List 2021 credential. |
| [`05-null-subject.json`](./05-null-subject.json) | Valid `ar.action.v1` with `subject: null` | Signature verifies. `null` subject is signed-over — **MUST NOT** be treated as an absent field or rejected. |
| [`06-jcs-canonicalization.json`](./06-jcs-canonicalization.json) | RFC 8785 JCS: input object → canonical string → UTF-8 bytes → SHA-256 | A conforming canonicalizer produces **byte-identical** canonical bytes; the `sha256` MUST match. Exercises key sorting, whitespace stripping, and NFC normalisation. |
| [`07-salted-commit.json`](./07-salted-commit.json) | Salted commitment `sha256(salt ‖ utf8(value))` | Recomputing `commit(salt, value)` yields the given `commitment`. Note the residual: hiding depends on value entropy (see threat-model §3.6). |
| [`08-mldsa65.json`](./08-mldsa65.json) | Valid envelope signed with ML-DSA-65 (FIPS 204) via `sig_alg` | Verifier dispatches on `sig_alg`, checks the `iss_key` prefix binding, and verifies against the raw public key. A verifier that cannot evaluate ML-DSA-65 MUST **fail closed** (`unsupported_sig_alg`). **Not byte-reproducible** — ML-DSA signatures are randomized; the vector self-verifies against its embedded public key. |

## How to use

1. Parse the JSON. For signed vectors, extract `receipt` and the public key
   (`issuer_public_key` for Ed25519 raw 32-byte; `issuer_public_key_raw_base64url`
   for ML-DSA-65).
2. For Ed25519, wrap the raw key in SPKI DER before handing it to a standard
   verifier: prefix the 32 raw bytes with `302a300506032b6570032100` (hex).
3. Recompute the canonical pre-signature bytes locally and verify — never trust a
   canonical form shipped with the vector.
4. Assert the `expected` outcome. For invalid/expired/revoked vectors, assert the
   verifier reaches the stated rejection reason, not merely "not accepted".

## License

CC0 1.0 Universal. See the repository `LICENSE`.
