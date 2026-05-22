# ACR v1 — threat model (Phase 0 scaffold)

> **Status — 2026-05-16.** Phase 0 scaffold. Mirrors APR's threat-model structure (`../apr/THREAT-MODEL.md`) and adds the threats specific to consent and compliance attestations. Expanded during ACR v1 spec authoring; the structural contract here is normative for any v1 conformance claim.

This document enumerates what ACR defends against, what it does NOT defend against, and the assumptions a deployer must satisfy for the defences to hold.

## 1. Assets

The protected assets are:

1. **Authenticity of the receipt.** That an `acr.v1` claim purporting to be signed by controller X under key `kid` was in fact produced by the holder of that key.
2. **Integrity of the receipt.** That every byte of the signed body — including the policy reference, scope, principal commitment, timestamps, and disclosure tier — is the same as when the controller signed it.
3. **Order and existence in time.** That a consent receipt claiming `iat = T` cannot be silently created at `T + Δ` and presented as if consent existed at `T`. Critical for GDPR Art. 7 / Art. 12(3) deadlines.
4. **Membership in a public bag.** That receipts cannot be selectively retracted, replaced, or fabricated post-hoc by a controller with control over only its own private surface.
5. **Withdrawal chain integrity.** That a withdrawal receipt declared at `T'` cannot be silently antedated, replaced, or omitted. The latest receipt in the chain wins by signed `iat`; any controller who serves a receipt that contradicts the chain has misbehaved.

What is explicitly **not** an asset of this protocol:

- The truth of the underlying consent itself (whether the subject genuinely consented).
- The completeness of the policy text referenced by the receipt.
- The behaviour or intent of the controller at signing time.

ACR is a compliance-record protocol. It binds *who attested to what, under which policy, when, and what happened next*. It does not adjudicate whether the attestation was free, specific, informed, or unambiguous in the GDPR Art. 4(11) sense.

## 2. Adversaries

Same six adversary classes as APR (external forger, compromised verifier client, compelled or malicious issuer, anchor-only attacker, side-channel observer, key-compromise insider). Two ACR-specific adversaries:

| Adversary | Capabilities |
|---|---|
| **Withdrawal-denial controller** | Holds valid signing keys; refuses to issue or publish a withdrawal receipt that the subject requested, or delays issuance past the regulatory deadline. |
| **Chain-tamper observer** | Has read access to the published chain; attempts to confuse downstream consumers by serving an out-of-date "latest" receipt while a newer one exists. |

## 3. Defended threats

### 3.1 Receipt forgery, post-hoc tampering, replay against a different scope, anchor backdating, anchor signature forgery, selective retraction

Same defences as APR §3.1–3.6. ACR uses the same primitives (Ed25519 + SHA-256 + RFC 8785 canonicalisation + daily Merkle anchoring + separate root key) under the same governance.

### 3.2 Forged withdrawal

**Threat.** An adversary fabricates a withdrawal receipt to make it appear that a subject withdrew consent when they did not.
**Defence.** Withdrawal receipts are signed by the same controller `kid` (or a successor `kid` declared via `notAfter`) as the original consent receipt. A forged withdrawal fails signature verification.
**Assumption.** Controllers may issue subject-countersigned withdrawals (D3 in the design doc). Where countersignature is present, both signatures verify independently.

### 3.3 Withdrawal-replacement attack

**Threat.** A withdrawal receipt is issued and anchored, then the controller publishes a replacement chain that omits it.
**Defence.** Per APR §3.6: anchored roots are immutable; a second daily root for the same date with different `rootHash` is itself evidence of misbehaviour. ACR consumers SHOULD cache the latest observed chain entry per `policyRef:principalCommitment` pair.

### 3.4 Antedated withdrawal

**Threat.** A controller backdates a withdrawal to claim compliance with a regulatory deadline (e.g., GDPR Art. 12(3) one-month response window).
**Defence.** Withdrawal receipt `iat` resolves to a UTC date; that date's daily root must contain the withdrawal's id. A withdrawal whose id is not in the published daily root for its claimed `iat` date fails verification with `missing-from-anchor`.

### 3.5 Field-level disclosure leak

**Threat.** The outer anchored root exposes principal-identifying material to anyone observing the public log.
**Defence.** Per design D5: the outer hash binds the inner Merkle tree of per-field salted commitments. The anchored root carries no plaintext principal data. Selective disclosure to an auditor is via the inner tree's per-field reveal; the public log stays PII-free.
**Assumption.** Salts are 16 bytes of CSPRNG output per field, never reused, never logged in cleartext. Salt loss collapses the field's privacy guarantee to "preimage-resistant hash of guessable input" — same residual risk as APR §4.2.

## 4. Threats out of scope

### 4.1 Compelled false consent

If a controller is forced (legally, technically, or operationally) to sign a body that records consent which was not actually given, ACR will faithfully record the falsehood. The protocol cannot distinguish a sincere attestation from a compelled-false one. Defence belongs to the legal, organisational, and key-custody layer above ACR.

### 4.2 Side-channel re-identification of pseudonymised principals

ACR's `metadata_only` and `selective_disclosure` tiers commit to principal identifiers via `commit = sha256(salt || principalId)`. With a 16-byte salt this is preimage-resistant against general dictionary attack, but a determined adversary with both a list of plausible `principalId` values AND the salt can verify any candidate. Salts MUST stay confidential to controller + auditor; leak treats the commitment as plaintext.

### 4.3 Policy-text drift

ACR binds `policyRef` (a URI or hash) into the signed body. It does NOT defend against a controller replacing the document at that URI after the fact. Controllers serious about compliance MUST commit to `policyHash = sha256(canonical-policy-bytes)` in the receipt body and publish the policy at a content-addressed URL. Spec authoring will mandate `policyHash` as a v1 SHOULD; profile-level MUST is per-profile.

### 4.4 Key compromise without rotation, verifier-side denial-of-service, active issuer-infrastructure observation, long-term cryptographic erosion

Same as APR §4.3, §4.4, §4.5, §4.6. The defences and limitations are identical.

## 5. Required deployer assumptions

A deployment that wants ACR's defences to hold MUST:

1. Source the controller-key and root-key catalogs from a channel the verifier trusts independently of the receipts being verified.
2. Treat published daily roots as append-only. Cache observed roots and the latest observed receipt per principal-commitment.
3. Rotate signing keys on any compromise suspicion, retaining old `kid`s with `notAfter` set.
4. Use distinct seeds for controller signing, root signing, and (where deployed) subject countersigning.
5. Generate per-field salts with CSPRNG, never reuse, never log in cleartext.
6. Publish `policyHash` alongside `policyRef` wherever the deployment claims regulatory mapping.
7. Honour GDPR Art. 7(3) / Art. 12(3) / equivalent deadlines for withdrawal issuance; withdrawals MUST be minted and anchored within the regulatory window.

If any of these conditions fail, the corresponding defence in §3 fails with them. The protocol is honest about this.

## 6. Non-goals

The following are explicitly NOT goals of ACR v1 and will not be addressed in v1.x:

- Confidentiality of the consent body (use a separate sealing layer for sensitive policy text).
- Anonymous-credential semantics (use a different protocol).
- Real-time revocation push (chained withdrawals are pull-discoverable, not push-broadcast).
- Determining whether consent was *valid* in the GDPR Art. 4(11) sense — that is a legal judgement, not a cryptographic property.
- Replacing the controller's existing consent-management UX. ACR is an output layer per design §1.2.

## 7. Status

This is a Phase 0 scaffold authored 2026-05-16 alongside the ACR design. The v1 spec authoring phase will expand each §3 threat with full test-vector references, formalise the assumptions in §5 as MUST/SHOULD statements, and freeze a per-profile table of mandatory fields. Until then, treat the structural contract here as normative for *what is defended* and the per-clause language as authoritative but pre-conformance.
