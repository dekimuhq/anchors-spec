# Verifier Conformance Checklist

A checklist for implementers of an Anchored Receipts **verifier**. It enumerates
the checks a verifier is expected to perform and the order in which their
failures should short-circuit. It is a description of the specified verification
flow ([`apr/v1.md`](../apr/v1.md) §8; `envelope/README.md` § Verifier Dispatch),
not a certification: passing this checklist is necessary, not sufficient, and
carries no assurance claim (see [`threat-model.md`](./threat-model.md)
review-status box).

The governing principle throughout: **a verifier MUST NOT accept a receipt on a
valid signature alone.** A valid signature answers only "was this signed by the
key it claims?" — never "is this receipt currently valid and in scope?".

---

## 0. Structural pre-checks (before any crypto)

- [ ] **Version dispatch.** Route on the top-level `v` field. `v=2` uses the
      shared envelope; `v=1` uses the legacy path. Never let the caller pick the
      version.
- [ ] **Unknown-field rejection.** Reject any top-level key outside the allowed
      envelope field set *before* verifying the signature — an unexpected field
      is a shape violation, not a signature question.
- [ ] **`claim_type` is registered.** Reject an unregistered discriminator
      (`claim_type_unknown`) without attempting body interpretation.
- [ ] **`iat_ms` consistency.** If present, `iat_ms` MUST equal
      `Date.parse(iat)`; reject on mismatch.

## 1. Signature

- [ ] **Recompute canonical bytes locally.** Do not trust any canonical form
      supplied with the receipt. Recompute RFC 8785 canonical JSON of the
      pre-signature object (signed-out fields zeroed/excluded per the envelope
      rule) — `envelope/README.md` § Cryptographic Primitives.
- [ ] **Resolve the key from a trusted channel.** The verifying public key comes
      from an independently-trusted catalog, **never** from a field inside the
      receipt being verified ([`apr/v1.md`](../apr/v1.md) §5.2).
- [ ] **`sig_alg` dispatch + binding.** Absent `sig_alg` ≡ `ed25519`. The
      `iss_key` algorithm prefix MUST equal `sig_alg`
      ([RFC-001](../rfcs/RFC-001-sig-alg-discriminator.md)); reject a mismatch.
- [ ] **Unsupported algorithm fails closed.** A `sig_alg` the verifier cannot
      evaluate MUST fail with `unsupported_sig_alg` — never a silent accept and
      never a downgrade.
- [ ] **Own-property scheme lookup.** Resolve the verifier for `sig_alg` by
      own-property lookup so a prototype member name (`toString`, `constructor`,
      `__proto__`, …) cannot masquerade as a registered scheme.
- [ ] **Strict boolean result.** Treat only a strict `true` from the underlying
      verify as success; any throw or truthy-non-boolean is a failure.
- [ ] **Shape-invalid ≡ signature-invalid outcome.** A malformed receipt and a
      signature-mismatch receipt both terminate as *not authentic* — do not leak
      a distinction that lets an attacker probe.

## 2. Canonicalization

- [ ] **Byte-exact canonicalizer.** Keys sorted by UTF-16 code unit, no
      whitespace, strings NFC-normalised, reject NaN/Infinity/undefined and
      circular references. Verify against KAT
      [`06-jcs-canonicalization.json`](./test-vectors/06-jcs-canonicalization.json)
      — the SHA-256 of the canonical bytes MUST match byte-for-byte.
- [ ] **Signed-out field discipline.** `anchor`, `sig`, `sig2`, `tsa`, `incl`
      must be excluded/zeroed identically to the reference rule, or existing
      receipts will fail to reproduce their signed bytes.
- [ ] **`null` is a value, not an absence.** A signed-in field set to `null`
      (e.g. `subject: null` on an autonomous action receipt — KAT
      [`05-null-subject.json`](./test-vectors/05-null-subject.json)) is part of
      the pre-image and MUST verify; do not treat it as a missing field.

## 3. Temporal validity

- [ ] **Honour `exp`.** If `exp` is present and precedes the verification
      instant, reject as `expired` — checked independently of, and after, the
      signature (KAT [`03-expired.json`](./test-vectors/03-expired.json)).
- [ ] **Sane `iat`.** Reject a receipt whose `iat` is implausibly in the future
      relative to the verifier's clock tolerance.

## 4. Status (revocation / suspension)

- [ ] **Resolve `credentialStatus` when present.** Fetch the referenced Status
      List 2021 credential, authenticate it, and read the bit at
      `statusListIndex`. A set revocation bit → reject as `revoked` (KAT
      [`04-revoked-status.json`](./test-vectors/04-revoked-status.json)).
- [ ] **Resolve `suspensionStatus` when present.** Same, for two-way suspension.
- [ ] **Freshness bound.** Bound the age of a cached status list; a stale list is
      a known residual (see [`threat-model.md`](./threat-model.md) §3.5).
- [ ] **Status is independent of signature.** A perfectly-signed receipt can be
      revoked; never let signature validity short-circuit the status check.

## 5. Anchor / transparency proof

- [ ] **Check inclusion, do not assume it.** Verify the RFC 6962 inclusion proof
      binds the receipt id to a published root; refuse to report a "verified"
      anchor state without actually checking inclusion
      ([`apr/v1.md`](../apr/v1.md) §7.3).
- [ ] **Anchor/root key ≠ claim key.** Verify the daily-root envelope against the
      *root* key catalog, distinct from the claim-signing catalog
      ([`apr/v1.md`](../apr/v1.md) §7.4).
- [ ] **Roots are append-only.** Cache observed roots; treat a second, differing
      root for the same date as evidence of misbehaviour, not as an update.
- [ ] **Absence is a failure state, not silence.** A receipt id missing from the
      claimed root fails with an explicit anchor-inclusion failure.

## 6. `trust_outcome` discipline (the load-bearing rule)

- [ ] **Never accept on signature alone.** The final accept/verdict MUST be the
      conjunction of: authentic signature **AND** in temporal validity **AND**
      not revoked/suspended **AND** (where required) anchor-included **AND**
      issuer trusted.
- [ ] **Signature validity ≠ issuer trust.** A valid signature proves the receipt
      is internally consistent (signed by the key it names), not that the key
      belongs to a trusted issuer. Pair every signature check with an issuer-trust
      resolution from an independent trust layer.
- [ ] **Report a structured outcome, not a boolean.** Distinguish
      `authentic`, `tampered`, `expired`, `revoked`, `suspended`,
      `missing-from-anchor`, `untrusted-issuer`, `unsupported_sig_alg` so a
      consumer can apply its own policy — collapsing them to a single boolean
      discards the information the protocol went to trouble to produce.
- [ ] **Issuer-asserted body metadata is not verifier-checkable.** Fields the
      issuer asserts about itself (delegation chains, mandate ids, agent ids) are
      bound by the signature but **not** independently re-derivable by a stateless
      verifier; do not report them as *verified*, only as *signed*.
- [ ] **Fail closed.** On any inability to complete a required check (unresolved
      key, unreachable status list where policy requires it, unsupported
      algorithm), refuse rather than degrade to accept.

---

## Reference vectors

The KATs in [`test-vectors/`](./test-vectors/) exercise the checks above — a
verifier should run them as a regression floor. They are illustrative fixtures,
not a conformance certification.
