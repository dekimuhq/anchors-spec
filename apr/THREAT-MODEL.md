# APR v1 — threat model

This document enumerates what APR defends against, what it does NOT defend against, and the assumptions a deployer must satisfy for the defences to hold. It is normative for the threats listed: a verifier or issuer that diverges from these defences is not v1-conforming.

## 1. Assets

The protected assets are:

1. **Authenticity of the receipt.** That an `ar.provenance.v1` claim purporting to be signed by issuer X under key `kid` was in fact produced by the holder of that key.
2. **Integrity of the receipt.** That every byte of the signed body — including artifact hash, agent identity, principal id, timestamps — is the same as when the issuer signed it.
3. **Order and existence in time.** That a receipt claiming `iat = T` cannot be silently created at `T + Δ` and presented as if it existed at `T`.
4. **Membership in a public bag.** That receipts cannot be selectively retracted, replaced, or fabricated post-hoc by an issuer with control over only its own private surface.

What is explicitly **not** an asset of this protocol:

- The truth of the assertions inside the body (whether the agent really did what the body says it did).
- The identity of the principal beyond what the issuer chooses to put in `principalId`.
- The behaviour or intent of the issuer at signing time.

APR is a provenance protocol. It binds *who claims what about whom, when*. It does not adjudicate whether the claim is true.

## 2. Adversaries

| Adversary | Capabilities | Out of scope |
|---|---|---|
| **External forger** | Can fabricate plausible-looking JSON, replay old receipts, present unsigned bodies, attempt collisions on hashes. Cannot read issuer private keys. | — |
| **Compromised verifier client** | Runs untrusted code that consumes APRs; may try to trick issuers into signing arbitrary bodies via inputs the consumer controls. | — |
| **Compelled or malicious issuer** | Holds valid signing keys; can sign anything it wants. May lie about agent kind, principal, or artifact hash at signing time. | — |
| **Anchor-only attacker** | Has access to the daily-root publishing surface but not to claim-signing keys. May attempt to backdate, omit, or duplicate roots. | — |
| **Side-channel observer** | Reads many receipts over time; may attempt traffic analysis, pseudonym dictionary attacks, or correlation across artifacts. | Out of scope: actively watches issuer infrastructure (timing, network). |
| **Key-compromise insider** | Acquires an issuer's signing key (private 32-byte seed) without rotation. | — |

## 3. Defended threats

### 3.1 Receipt forgery

**Threat.** An adversary fabricates a receipt that appears to come from a legitimate issuer.
**Defence.** Ed25519 signature over RFC-8785-lite-canonical bytes of the pre-signature object. Verifiers MUST recompute the canonical bytes locally from the received claim and verify the signature against a public key they trust independently.
**Assumption.** Verifiers source the public-key catalog from a trustworthy channel (env var, signed file, or future `/.well-known` with pinning). Trusting an issuer-supplied catalog inline with the receipt would be self-defeating.

### 3.2 Post-hoc tampering

**Threat.** An adversary modifies any byte of a published receipt — body fields, timestamps, principal id, kid — and represents it as still-valid.
**Defence.** Same as 3.1: any byte change in the pre-signature object changes the canonical bytes and the signature ceases to verify. Verifiers MUST treat shape-invalid receipts and signature-invalid receipts identically (`status: "tampered"`).
**Assumption.** Canonicalisation is deterministic across all conforming implementations. The reference implementation in `apr-verifier` and the published test vectors guarantee this on a per-byte basis.

### 3.3 Replay against a different scope

**Threat.** A receipt issued for one consumer is replayed against another to create the impression of consent or production for the second consumer.
**Defence.** Bound by `subject` (what the receipt is about), optional `audience` (who it was issued to), optional `exp` (when it expires), and `iat` (when it was issued). Verifiers MUST honour `exp`. Verifiers SHOULD use `audience` and `subject` to scope acceptance.
**Limitation.** APR v1 does not include a nonce or one-time-use flag. Receipts within `exp` are reusable by design — they document a past production event, not authorise a future action. Consumers needing one-time-use semantics MUST layer that on top.

### 3.4 Anchor backdating

**Threat.** An issuer with control over its own signing key attempts to retroactively claim a receipt existed earlier than it did.
**Defence.** Daily Merkle anchoring. A receipt's `iat` resolves to a UTC date; the daily root for that date is published once and is itself signed by a separate root key. A receipt whose id is not in the published daily root's leaf set fails verification with `missing-from-anchor`.
**Assumption.** Daily roots are published on a non-retroactive cadence, with each day's root immutable once published. Verifiers MUST refuse to accept a "verified" anchor state without checking inclusion. Roll-forward semantics (publishing tomorrow's root is not enough; today's root must be the one that contains today's claims).

### 3.5 Anchor signature forgery

**Threat.** An adversary publishes a fake daily-root envelope.
**Defence.** Daily-root envelope signed by a root key, verified against a separate root-key catalog. Root keys are intentionally not the same as claim-signing keys; compromising one does not compromise the other.

### 3.6 Selective retraction

**Threat.** An issuer signs and publishes a receipt, then later wants to deny it ever existed by republishing the daily root without that id.
**Defence.** Daily roots, once published, are immutable. Verifiers and consumers MAY (and SHOULD) cache roots they have observed. A second daily root for the same date with different `rootHash` is itself evidence of misbehaviour.
**Assumption.** At least one honest cache of daily roots exists. Phase 1+ deliverables include anchor mirroring as a defence-in-depth.

## 4. Threats out of scope

### 4.1 Compelled false statement at signing

If an issuer is forced (legally, technically, or operationally) to sign a body that is factually wrong, APR will faithfully record the falsehood. The protocol cannot distinguish a sincere claim from a compelled-false one. Defence belongs to the legal, organisational, and key-custody layer above APR — for example, requiring multi-party signatures, hardware-bound keys with attestation, or out-of-band corroboration.

### 4.2 Side-channel re-identification of pseudonymised principals

`public_minimal` discloses `principalPseudonym = "u_" + sha256_hex(principalId).slice(0, 8)`. This compresses to **32 bits**. A determined adversary with a list of plausible `principalId` values can dictionary-attack the pseudonym to high confidence. APR does not defend against this — the choice of `principalId` and disclosure tier rests with the issuer. See [`PRIVACY-REVIEW.md`](./PRIVACY-REVIEW.md).

### 4.3 Key compromise without rotation

If a signing key's 32-byte seed leaks and the issuer does not rotate the `kid`, an attacker can mint indistinguishable forgeries. The defence is operational: issuers MUST rotate keys promptly on any suspicion of compromise, and the verify-keys catalog MUST support multi-kid issuers so rotation is non-disruptive. The `notAfter` field exists precisely to mark a compromised kid as no-longer-valid without removing it (which would also retroactively invalidate honest receipts).

### 4.4 Verifier-side denial-of-service

A verifier serving a public surface (like `verify.dekimu.com`) may be overwhelmed by request volume. APR's threat model does not address availability of any specific verifier — multiple independent verifiers are encouraged exactly so the spec does not depend on a single one. Self-hosting the reference verifier is a documented path; see `apr-verifier`'s self-host guide.

### 4.5 Active issuer-infrastructure observation

An adversary inside the issuer's network can observe inputs to the signer, timing, and pre-publication state. APR does not defend against this; it defends only the published artefact. Issuers responsible for sensitive principals MUST run their signing surface as if it is a target, including HSM-bound keys and minimal logging of pre-signature inputs.

### 4.6 Long-term cryptographic erosion

APR v1 specifies Ed25519 + SHA-256. Both are believed secure today. Cryptographic agility is deferred to v2: there is no algorithm-id field in v1. If Ed25519 or SHA-256 ceases to be safe, v1 receipts produced before the break remain forensically interesting but lose forward defence; new receipts MUST move to v2. The 36-month LTS window for v1 (see `CHANGELOG.md`) is deliberately shorter than the expected horizon of any catastrophic break.

## 5. Required deployer assumptions

A deployment that wants APR's defences to actually hold MUST:

1. Source the claim-key and root-key catalogs from a channel the verifier trusts independently of the receipts being verified.
2. Treat the published daily roots as append-only. Caching observed roots is recommended.
3. Rotate signing keys on any compromise suspicion, retaining old `kid`s with `notAfter` set rather than deleting them.
4. Use distinct seeds for claim signing and root signing.
5. Keep signing-key seeds out of replicated/backed-up storage where possible. HSM-bound is the recommended posture for production issuers.

If any of these conditions fail, the corresponding defence in §3 fails with them. The protocol is honest about this.

## 6. Non-goals

The following are explicitly NOT goals of APR v1 and will not be addressed in v1.x:

- Confidentiality of the receipt body (use a separate sealing layer).
- Anonymous credential semantics (use a different protocol).
- Real-time revocation (closest analogue is `notAfter` on `kid`s, which is not push-revocation).
- Settlement-bearing receipts (planned for a separate spec; APR v1 is observational only).
