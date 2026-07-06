# Anchored Receipts — Envelope Threat Model

Scope: the **shared v=2 envelope** (`envelope/README.md`) and the verification
flow common to every Anchored Receipt family. Per-family threat models
(e.g. [`apr/THREAT-MODEL.md`](../apr/THREAT-MODEL.md),
[`acr/THREAT-MODEL.md`](../acr/THREAT-MODEL.md)) refine this for their body
shapes; this document is the layer they share.

The purpose of this document is to **scope** the protocol's defences and their
limits precisely enough that an external cryptographic reviewer can check them.
It describes what the specification *intends* to mitigate and, for each threat,
states the explicit **residual** risk and **out-of-scope** boundary. It makes no
conformance or assurance claim.

> ### Review status
>
> **An external cryptographic review of this protocol has NOT yet been
> performed.** This document exists to define the scope for one — it is a
> statement of design intent and known limits, not a statement of verified
> security. Nothing here should be read as an assurance that the described
> mitigations are complete or correct. Findings from a future review will be
> reconciled back into this file and the per-family models.

---

## 1. Assets

The protocol is designed to protect the following assets. For each, the column
"Protection intent" states what the design *aims* to provide, not what has been
independently verified.

| Asset | Protection intent |
|---|---|
| **Envelope integrity** | Every byte of the signed pre-image — body, timestamps, subject, issuer, `sig_alg`, status pointers — is bound by the issuer signature. A change to any signed field is intended to be detectable as a signature failure. |
| **Issuer keys** | The private signing key is the root of authenticity for a family of receipts. Its confidentiality is an issuer-operational asset the protocol depends on but does not itself protect. |
| **Status lists** | The revocation / suspension bitstrings that let an issuer mark a receipt no longer valid without editing the signed receipt. Their authenticity and freshness are assets. |
| **Transparency log** | The append-only inclusion evidence (daily Merkle roots / log inclusion proofs) that a receipt existed at or before a stated time and cannot be silently retracted. |
| **Salted commitments** | The hiding commitments (`sha256(salt ‖ value)`) used to pseudonymise subjects and selectively-disclosed fields. The asset is the *unlinkability* of the committed value. |

Explicitly **not** assets of this protocol:

- The **truth** of the statements inside `body`. The envelope binds *who signed
  what, when*; it does not adjudicate whether the signed statement is factually
  correct. (See §4.1.)
- The **behaviour or honesty of the issuer** at mint time.
- **Confidentiality of the receipt body** (the envelope is integrity-protected,
  not encrypted; use a separate sealing layer for confidentiality).

---

## 2. Adversaries

| Adversary | Capabilities | Explicitly outside their reach |
|---|---|---|
| **Issuer forger (external)** | Fabricates plausible JSON, replays receipts, presents unsigned or re-signed bodies, attempts hash/signature collisions. | Cannot read issuer private keys. |
| **Replay attacker** | Takes a genuine, validly-signed receipt and presents it in a different scope, audience, or time window. | Cannot alter any signed field without breaking the signature. |
| **Log-equivocation adversary** | An issuer (or log operator) that shows different inclusion evidence to different verifiers — e.g. a receipt in one verifier's view of the log, absent from another's. | Bounded by whatever independent log-observation exists (see §4.2). |
| **Status-list tamperer** | Attempts to forge, roll back, or freshness-attack a revocation/suspension list to make a revoked receipt appear live (or a live one appear revoked). | Cannot alter the signed receipt itself. |
| **Commit brute-forcer** | Holds a salted commitment and a candidate list of plausible pre-images; dictionary-attacks the commitment to re-identify the committed value. | Gains nothing against a high-entropy, well-salted value. |
| **Key-compromise insider** | Acquires an issuer signing key (with or without the issuer's knowledge), including across a key rotation / continuity boundary. | — |

---

## 3. Threats and mitigations

Each threat lists the mitigation **as specified** (with a citation to the
governing spec section) and an explicit **residual / out-of-scope** note.
Mitigation language is deliberately "intends / mitigates", never "prevents".

### 3.1 Issuer forgery (signature forgery)

- **Threat.** An adversary without the issuer key produces a receipt that a
  verifier accepts as issuer-signed.
- **Mitigation (as specified).** Ed25519 (or, via the `sig_alg` agility hook,
  ML-DSA-65) over the RFC 8785 canonical pre-signature bytes
  (`envelope/README.md` § Cryptographic Primitives;
  [`apr/v1.md`](../apr/v1.md) §4). Verifiers MUST recompute the canonical bytes
  locally and verify against a public key sourced from a trusted channel, never
  from the receipt itself ([`apr/v1.md`](../apr/v1.md) §5.2). The `sig_alg`↔
  `iss_key` prefix binding ([RFC-001](../rfcs/RFC-001-sig-alg-discriminator.md))
  is intended to block algorithm-substitution across schemes.
- **Residual / out-of-scope.** Rests entirely on the hardness of the chosen
  signature scheme and on the verifier's key catalog being trustworthy. A
  catalog poisoned upstream, or a future cryptanalytic break of the scheme,
  defeats this mitigation. Not independently reviewed (see review-status box).

### 3.2 Post-hoc tampering

- **Threat.** Any signed field of a published receipt is altered and re-presented
  as valid.
- **Mitigation (as specified).** Every signed-in field is inside the canonical
  pre-image; a one-bit change alters the canonical bytes and the signature
  ceases to verify ([`apr/v1.md`](../apr/v1.md) §4.2). Signed-out fields
  (`anchor`, `sig`, `sig2`, `tsa`, `incl`) are excluded/zeroed from the pre-image
  by rule so that post-mint attachments do not perturb the signature.
- **Residual / out-of-scope.** Depends on canonicalization being byte-identical
  across all conforming implementations. The KATs in
  [`test-vectors/`](./test-vectors/) pin this per-byte; a divergent canonicalizer
  is a real risk this model flags rather than eliminates.

### 3.3 Replay against a different scope

- **Threat.** A receipt issued for one purpose/audience/time is replayed against
  another.
- **Mitigation (as specified).** Scope-binding fields are signed-in: `subject`,
  `exp`, `iat`/`iat_ms`, and family-specific audience/purpose fields. Verifiers
  MUST honour `exp` ([`apr/v1.md`](../apr/v1.md) §5.3) and SHOULD scope on
  `subject`.
- **Residual / out-of-scope.** The envelope carries **no nonce / one-time-use
  flag** — a receipt within its validity window is intentionally reusable (it
  documents a past event, it does not authorise a future action). Consumers
  needing single-use semantics MUST layer that above the envelope.

### 3.4 Log equivocation

- **Threat.** An issuer or log operator presents inconsistent inclusion evidence
  to different verifiers, or retro-actively omits a receipt from the log.
- **Mitigation (as specified).** Daily Merkle roots are published on a
  non-retroactive cadence and are immutable once published; a receipt whose id
  is not in the published root fails with an anchor-inclusion failure
  ([`apr/v1.md`](../apr/v1.md) §7.3). A second root for the same date with a
  different root hash is itself evidence of misbehaviour
  ([`apr/v1.md`](../apr/v1.md) §7.3). Verifiers and consumers SHOULD cache
  observed roots.
- **Residual / OUT OF SCOPE (named limitation).** Detection of equivocation
  currently depends on **at least one honest, independent observer** caching and
  comparing roots. There is **no witness co-signing / gossip mechanism** in the
  specified envelope: until independent witnesses co-sign log heads, a
  sufficiently-resourced operator that controls every verifier's view can
  equivocate undetected. This is an accepted, explicitly-scoped gap — see also
  §4.2.

### 3.5 Status-list tampering

- **Threat.** A revoked receipt is made to appear live (or vice versa) by
  forging or rolling back the status list.
- **Mitigation (as specified).** `credentialStatus` / `suspensionStatus` are
  **signed-in** pointers (W3C Bitstring Status List 2021) — the *pointer* cannot
  be altered without breaking the receipt signature. The status list credential
  itself is a separately-published, issuer-authenticated artefact
  (`family-consistency.md`, status-list rows). Verifiers resolve the list and
  read the bit at `statusListIndex`.
- **Residual / out-of-scope.** The freshness and authenticity of the *fetched
  list* depend on how it is published and cached; a stale or substituted list is
  a live risk unless the verifier authenticates the list credential and bounds
  its age. The envelope binds the pointer, not the list contents at read time.

### 3.6 Commit brute-force / re-identification

- **Threat.** A salted commitment to a pseudonymised subject/field is
  dictionary-attacked to recover the pre-image.
- **Mitigation (as specified).** `commit(salt, value) = sha256(salt_bytes ‖
  utf8(value))` with a 256-bit salt recommended (`commit` primitive; KAT
  [`07-salted-commit.json`](./test-vectors/07-salted-commit.json)).
- **Residual / OUT OF SCOPE (named limitation).** Hiding rests **entirely** on
  the entropy of the committed value plus the salt. A low-entropy value (a known
  email, a small enumerable ID) is dictionary-attackable regardless of salt.
  Truncated pseudonyms (e.g. an 8-hex-char display pseudonym) compress to a small
  space and are re-identifiable by design-tradeoff — the disclosure tier choice
  rests with the issuer (per-family privacy reviews cover this).

### 3.7 Key compromise, rotation, and continuity

- **Threat.** An issuer signing key leaks; an attacker mints indistinguishable
  receipts. The compromise may straddle a key-rotation boundary or an issuer
  continuity (issuer-identity handover) event.
- **Mitigation (as specified).** Key catalogs support multiple `kid`s so rotation
  is non-disruptive; a `notAfter`-style marker retires a compromised key without
  retroactively invalidating honestly-signed historical receipts
  ([`apr/v1.md`](../apr/v1.md) §5.2, §7.4). Root/anchor keys are kept distinct
  from claim-signing keys so compromising one does not compromise the other
  ([`apr/v1.md`](../apr/v1.md) §7.4). Continuity attestations link a new key to a
  prior issuer identity.
- **Residual / OUT OF SCOPE (named limitation).** Revocation of a compromised key
  is **not push-revocation** — a verifier that has not refreshed its catalog, or
  that lacks a fresh status list, may still accept forgeries minted before the
  retirement propagated. The window between compromise and catalog/status
  propagation is unprotected. Detecting *whether a rotation/continuity event was
  itself authorised* reduces to trusting the issuer's own key custody; the
  protocol records the event, it does not prove the event was benign.

---

## 4. Out-of-scope by design (residual trust assumptions)

These are not defended by the protocol and a deployer must satisfy them
out-of-band. They are called out so a reviewer scopes them explicitly.

### 4.1 Issuer honesty at mint

The single largest residual: the envelope faithfully records whatever the issuer
signs. If an issuer is compelled, mistaken, or malicious at signing time, the
protocol binds the false statement exactly as it binds a true one. There is **no
protocol-level defence** — mitigation lives in the legal/organisational/key-
custody layer (multi-party signing, hardware-bound keys, out-of-band
corroboration). Any "trust" derived from a receipt is trust in the *issuer*, not
in the receipt format.

### 4.2 Log equivocation until witness co-signing

As in §3.4: absent an independent witness / gossip layer co-signing log heads,
equivocation detection depends on honest independent observers. This is a known
architectural gap, not an oversight.

### 4.3 Confidentiality

The envelope is integrity-protected, not encrypted. Anything placed in `body` in
cleartext is disclosed to any holder. Confidentiality is a separate sealing layer.

### 4.4 Availability / denial-of-service

The model does not address the availability of any specific verifier, log, or
status endpoint. Independent, self-hostable verifiers are encouraged so the
protocol does not depend on a single operator.

### 4.5 Long-term cryptographic erosion

The default scheme is Ed25519 + SHA-256. The `sig_alg` hook
([RFC-001](../rfcs/RFC-001-sig-alg-discriminator.md)) admits ML-DSA-65 for
post-quantum agility, and a hybrid dual-signature posture exists, but per-receipt
posture enforcement cannot by itself defend a receipt whose classical key has
been broken and which carries no PQC signature. Complete migration defence is an
issuer-level trust-state concern, not a per-receipt one.

---

## 5. Required deployer assumptions

The intended mitigations in §3 hold **only if** a deployment:

1. Sources claim-key and root-key catalogs from a channel trusted independently
   of the receipts being verified.
2. Treats published log roots as append-only and caches observed roots.
3. Authenticates and age-bounds fetched status lists.
4. Rotates signing keys on any suspicion of compromise, retiring old `kid`s
   rather than deleting them.
5. Uses distinct key material for claim signing vs. anchor/root signing.
6. Chooses committed-value entropy appropriate to the re-identification risk it
   is willing to accept.

If any assumption fails, the corresponding mitigation in §3 fails with it. This
document states that dependency plainly rather than papering over it.

---

## 6. What this document is not

- Not an assurance that the mitigations are correctly implemented — see the
  review-status box.
- Not a conformance statement and not a claim of alignment to any external
  regulatory or standards regime.
- Not a substitute for the per-family threat models, which refine the body-level
  threats this shared model does not cover.
