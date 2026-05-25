# RFC-001: Signature Algorithm Discriminator (`sig_alg`)

**Authors:** Dekimu
**Status:** Draft
**Created:** 2026-05-25
**Target version:** v1.3 (envelope)

---

## Motivation

The Anchored Receipts envelope currently hardcodes Ed25519 as the sole signature algorithm. The `iss_key` and `countersig_key` fields carry an `ed25519:<base64url>` prefix convention, but the `sig` field itself has no algorithm indicator — verifiers assume Ed25519 unconditionally.

This blocks three adoption paths:

1. **Post-quantum cryptography.** NIST FIPS 204 (ML-DSA-65) is standardized and expected to become a requirement for government and critical-infrastructure issuers. Receipts signed today with Ed25519 alone cannot prove quantum-resistant integrity.

2. **BBS+ selective disclosure.** W3C vc-di-bbs defines BBS+ signature suites for Verifiable Credentials. Adding BBS+ to the envelope requires algorithm negotiation at the envelope level, not inside member bodies.

3. **HSM-backed signing.** eIDAS Qualified Signature Creation Devices (QSCDs) may expose algorithms other than Ed25519. An algorithm discriminator lets issuers sign with whatever their HSM supports without forking the envelope.

A `sig_alg` field solves all three by making the algorithm explicit and extensible.

## Proposed Changes

### Wire format

Add one optional field to the v2 envelope:

```typescript
sig_alg?: "ed25519" | "ml-dsa-65" | "ed25519+ml-dsa-65";
```

| Property | Value |
|----------|-------|
| Field name | `sig_alg` |
| Position | Top-level envelope field, alongside `sig` |
| Type | `string` (constrained enum) or absent |
| Required | No — absent implies `"ed25519"` |
| Signed-in | **Yes** — mutating `sig_alg` invalidates the signature |
| Canonicalization | Included in the pre-signature object per RFC 8785 |

The envelope field inventory moves from 24 to 25 fields.

**Algorithm values:**

| Value | Meaning |
|-------|---------|
| `"ed25519"` | Ed25519 (current default). Equivalent to field absent. |
| `"ml-dsa-65"` | ML-DSA-65 (FIPS 204). `sig` carries a single ML-DSA-65 signature, base64url-encoded. |
| `"ed25519+ml-dsa-65"` | Dual-sign mode. `sig` carries `<ed25519-sig>.<ml-dsa-sig>` as two base64url values separated by a single `.` (dot). Both signatures are computed over the same canonical pre-signature bytes. |

Future algorithm values (e.g., `"bbs+"`) follow the same pattern: add a new enum member in a subsequent RFC.

### Verification

Verifiers MUST:

1. Read `sig_alg`. If absent, treat as `"ed25519"`.
2. If the value is unrecognized, reject with error code **`sig_alg_unknown`**.
3. For `"ed25519"`: verify as today — single Ed25519 signature.
4. For `"ml-dsa-65"`: verify `sig` as a single ML-DSA-65 signature against the issuer's ML-DSA-65 public key.
5. For `"ed25519+ml-dsa-65"`: split `sig` on `.`, verify the first segment as Ed25519 and the second as ML-DSA-65. Both MUST pass; failure of either is a signature failure.

**New error code:**

| Code | Meaning |
|------|---------|
| `sig_alg_unknown` | The `sig_alg` value is not in the verifier's supported set |

Verifiers that do not yet support ML-DSA-65 will correctly reject `"ml-dsa-65"` and `"ed25519+ml-dsa-65"` receipts with `sig_alg_unknown`. This is the intended behavior — verifiers upgrade at their own pace.

### Issuer

Issuers:

- MAY omit `sig_alg` when signing with Ed25519 (backward-compatible default).
- MUST set `sig_alg` when signing with any algorithm other than Ed25519.
- MUST include `sig_alg` in the pre-signature object before canonicalization when the field is present.
- For dual-sign mode, MUST compute both signatures over identical canonical bytes and concatenate as `<ed25519>.<ml-dsa>`.

The `iss_key` prefix convention (`ed25519:<base64url>`) extends naturally: issuers using ML-DSA-65 would use `ml-dsa-65:<base64url>`. Dual-sign issuers publish both keys. Key publication mechanics are out of scope for this RFC.

## Backward Compatibility

- **v1.2 receipts verify unchanged.** The field is optional and absent implies Ed25519 — the current behavior.
- **No canonicalization change.** `sig_alg` is a new signed-in field; its presence in the pre-signature object follows the existing RFC 8785 canonicalization. Receipts without `sig_alg` produce the same canonical bytes as today.
- **Verifier upgrade is not forced.** Verifiers that do not recognize `sig_alg` will either ignore it (still verifying Ed25519 correctly for omitted/`"ed25519"` values) or reject unknown values. The `sig_alg_unknown` error code gives them a clean rejection path.

## Security Considerations

1. **Algorithm downgrade.** Because `sig_alg` is signed-in, an attacker cannot strip or change it without invalidating the signature. A receipt signed with `"ed25519+ml-dsa-65"` cannot be downgraded to `"ed25519"` by removing the field.

2. **Dual-sign binding.** In `"ed25519+ml-dsa-65"` mode, both signatures cover identical canonical bytes including the `sig_alg` field itself. This prevents mix-and-match attacks where an Ed25519 signature from one receipt is paired with an ML-DSA-65 signature from another.

3. **Key-algorithm mismatch.** Verifiers MUST check that `iss_key` prefix matches `sig_alg`. An `ed25519:...` key with `sig_alg: "ml-dsa-65"` is invalid.

4. **ML-DSA-65 signature size.** ML-DSA-65 signatures are ~3,309 bytes (vs. 64 bytes for Ed25519). Implementations should anticipate larger `sig` values. The base64url encoding adds ~33% overhead.

5. **Quantum timeline.** This RFC does not mandate post-quantum signatures. It provides the mechanism so issuers can adopt PQ crypto when their threat model requires it. The dual-sign mode is the recommended migration path: issuers dual-sign during the transition period, then drop Ed25519 once PQ verification is ubiquitous.

## Test Vectors

The following vectors MUST ship in `envelope/conformance/` alongside this change:

1. **`sig_alg` absent** — existing receipt, verifies as Ed25519.
2. **`sig_alg: "ed25519"`** — explicit Ed25519, verifies identically to absent.
3. **`sig_alg: "ml-dsa-65"`** — ML-DSA-65-only signature, verifies with ML-DSA-65.
4. **`sig_alg: "ed25519+ml-dsa-65"`** — dual-sign, both signatures verify.
5. **`sig_alg: "unknown-alg"`** — rejected with `sig_alg_unknown`.
6. **`sig_alg` tampered** — signed-in field mutated post-sign, signature fails.
7. **Key-algorithm mismatch** — `iss_key` prefix does not match `sig_alg`, rejected.

## Implementation Notes

- **Rollout order:** Verifiers first (accept new field), then issuers (start emitting). This is the standard additive-field rollout.
- **Feature detection:** Verifiers can advertise supported algorithms. Out of scope for this RFC but relevant for discovery.
- **ML-DSA-65 library:** Implementations should use a FIPS 204-compliant library. Node.js does not ship ML-DSA in `node:crypto` as of v22; a userland dependency is expected.
- **`countersig` alignment:** If `countersig_key` uses a different algorithm prefix than `iss_key`, the countersignature algorithm is independent of `sig_alg`. A future RFC may add `countersig_alg` if needed; for now, countersignatures follow their key prefix.

## References

- [NIST FIPS 204 — Module-Lattice-Based Digital Signature Standard (ML-DSA)](https://csrc.nist.gov/pubs/fips/204/final)
- [W3C vc-di-bbs — BBS+ Data Integrity Cryptosuites](https://www.w3.org/TR/vc-di-bbs/)
- [eIDAS Regulation — Qualified Signature Creation Devices (QSCD)](https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:32014R0910)
- [RFC 8785 — JSON Canonicalization Scheme (JCS)](https://www.rfc-editor.org/rfc/rfc8785)
- [Anchored Receipts Envelope Wire Format](../envelope/README.md)
- [Family Consistency — Countersignature key field](../family-consistency.md#10-countersignature-key-field-countersig_key)
