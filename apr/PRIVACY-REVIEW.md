# APR v1 — privacy review

This document is the v1 written privacy review of APR's `public_minimal` disclosure tier under GDPR, with a residual-risk conclusion that is also mirrored into the spec body (`v1.md` §6.4 references this file). It is a Phase 0 deliverable and is normative for the assertions in §5 ("Conclusion").

This is not legal advice. It is a structured engineering analysis of identifiability under documented disclosure semantics, written so a Data Protection Officer or external reviewer can assess fitness for purpose.

## 1. Scope

The review covers what `public_minimal` actually serves, NOT the underlying signed body. Receipts in tiers `metadata_only` and `public_full` are out of scope:

- `metadata_only` serves only `{ disclosure: "metadata_only", redacted: true }`. There is no personal data to review.
- `public_full` serves the full signed body verbatim. By choosing `public_full`, the issuer is making an explicit decision that every field is intentionally public. That decision belongs to the issuer; APR's role is to faithfully reproduce what was signed.

`public_minimal` is the default public surface and the one most likely to be served at scale. It is therefore the only tier requiring a written review.

## 2. What `public_minimal` discloses

Per `v1.md` §6.2, `public_minimal` serves:

| Field | Content | Direct identifier? | Indirect identifier? |
|---|---|---|---|
| `artifact.type` | enum (closed) | no | low |
| `artifact.id` | string set by the issuer | possibly* | possibly |
| `artifact.hash` | SHA-256 hex | no | possibly** |
| `artifact.hashAlgorithm` | literal `"sha256"` | no | no |
| `artifact.contentType` | optional MIME | no | no |
| `artifact.sizeBytes` | optional integer | no | low |
| `agent.kind` | closed enum | no | low |
| `agent.model` | optional string | no | low |
| `agent.provider` | optional string | no | low |
| `agent.humanRole` | optional closed enum | no | low |
| `authorization.principalPseudonym` | `"u_" + first 8 hex of sha256(principalId)` | no | **yes — the central concern** |
| `inputs[].kind` | closed enum array | no | low |
| `startedAt` / `completedAt` / `durationMs` | unix-second integers | no | medium |
| `disclosure` | literal `"public_minimal"` | no | no |
| `redacted` | literal `true` | no | no |

*`artifact.id` is verifier-blind. If issuers put PII or stable user identifiers in artifact ids, public_minimal will faithfully expose them. Issuers SHOULD treat `artifact.id` as public.

**`artifact.hash` is a one-way digest of the artifact bytes. It is not personal data on its own, but it is a stable correlation key: identical artifacts produce identical hashes across receipts.

What `public_minimal` does NOT serve (relative to the signed body): `authorization.principalId` (raw), `inputs[].hash`, `inputs[].source`, `inputs[].claimId`, `summary`.

## 3. Pseudonym threat analysis

The principal pseudonym is the only field in `public_minimal` that purports to be a privacy protection. The mechanism:

```
principalPseudonym = "u_" + sha256_hex(principalId).slice(0, 8)
```

This is a 32-bit truncation of SHA-256. It is **not anonymisation under GDPR**; it is pseudonymisation, and pseudonymous data remains personal data when the original identifier or a means of re-identification exists ([Recital 26, Recital 28](https://gdpr-info.eu/recitals/no-26/)).

### 3.1 Birthday-bound collisions

Within a 32-bit space, expected collisions appear at ≈65 536 distinct principals. For an issuer whose principal pool is materially larger than that, two distinct principals can share a pseudonym across receipts — degrading the *correlation utility* of the pseudonym from an attacker's perspective. This bounds, but does not eliminate, individual identification.

### 3.2 Dictionary attack

If the adversary holds the principal-id space, computing the pseudonym for every candidate is trivial. SHA-256 is not slow, and 32 bits leaves no headroom. Concretely:

- **`principalId` = email address** (e.g. `alice@example.com`): if the adversary suspects Alice and knows her email, they can confirm to ~32-bit confidence (false-positive rate ≈ 1 in 2³², bounded down by the candidate population). For most realistic candidate pools this is *effectively a confirmation*. The pseudonym does **not** protect Alice's pseudonym from linking back to Alice.
- **`principalId` = opaque internal user id**: dictionary attack requires the adversary to enumerate plausible ids. If the id space is small or guessable (e.g. autoincrement integers), it is trivially exhausted. If it is a 128-bit random id, dictionary attack is intractable, but the pseudonym is then redundant with the id and serves no privacy purpose.
- **`principalId` = privacy-preserving randomised id with no out-of-band linkage**: the pseudonym adds nothing on top, but also reveals nothing.

**Conclusion of dictionary analysis:** the pseudonym does not protect against an adversary who already holds the candidate set. It exists to prevent *casual harvesting* of raw `principalId` values from a public verifier surface, not to prevent re-identification by motivated attackers.

### 3.3 Cross-receipt correlation

Two receipts with the same `principalPseudonym` are linkable. If an issuer uses the same `principalId` shape across multiple artifacts, a public verifier surface lets observers build a per-pseudonym timeline (when did `u_a3f9c1b2` produce things, what kinds of things, with what models, on what cadence). This is the most realistic privacy loss from `public_minimal`: **behaviour-pattern reconstruction**, even without ever resolving the pseudonym back to a real identity.

### 3.4 Side-channel correlation

Combinations of `agent.kind`, `agent.model`, `agent.provider`, `artifact.contentType`, `sizeBytes`, `startedAt`/`completedAt`, and `artifact.hash` form a fingerprint. For most issuers and most principal pools, this fingerprint will not uniquely identify a person, but it raises confidence on a candidate set produced by other means.

## 4. GDPR posture

### 4.1 Lawful basis

APR does not specify a lawful basis. The issuer is the controller of `principalId`. Whether a public surface for `public_minimal` receipts is lawful depends on:

- the relationship between issuer and principal (contract, consent, legitimate interest),
- whether the principal can reasonably foresee public emission of receipts about their actions,
- whether public emission is necessary for the issuer's lawful purpose, and
- the residual re-identification risk in the principal's specific population.

A consumer-facing service that issues `public_minimal` APRs about end-users without explicit consent or a clearly-disclosed legitimate-interest argument is unlikely to satisfy GDPR. An issuer for which the "principal" is the issuer's own organisation or a corporate signer is unlikely to involve personal data at all.

### 4.2 Data subject rights

- **Right to erasure (Art. 17).** Tension. The whole point of anchored receipts is non-repudiation. An issuer who emits a `public_minimal` APR about a data subject and then receives an erasure request CANNOT pull the receipt out of an immutable daily root. Practical mitigations: (a) avoid using personally-identifying `principalId` values for individuals where possible; (b) if individuals are unavoidable, restrict the receipt to `metadata_only` so a future erasure request can be satisfied by ceasing to serve the body, even though the signature record persists; (c) document the anchored-record nature in the issuer's privacy notice.
- **Right of access (Art. 15).** Achievable. Issuers know the principalId → receipts mapping internally and can produce subject's receipts on request.
- **Right to rectification (Art. 16).** N/A. APR records past events; rectification is not meaningful. Erroneous receipts are corrected by issuing a corrective receipt, not by mutating the original.

### 4.3 Pseudonymisation under GDPR

The `principalPseudonym` mechanism is technical pseudonymisation in the GDPR sense (Art. 4(5)). It is therefore subject to all the protections owed to personal data, **and** counts as an appropriate safeguard under Recital 28. It does not move data outside GDPR scope.

## 5. Conclusion

The `public_minimal` disclosure tier is a **low-friction-but-not-anonymous** public surface. Its concrete privacy properties are:

1. It prevents casual harvesting of raw `principalId` strings.
2. It does **not** prevent re-identification by an adversary who already holds the candidate principal-id set.
3. It permits per-pseudonym behaviour reconstruction across receipts that share the same `principalId`.
4. It remains personal data under GDPR.

**Therefore, the spec body (`v1.md` §6.4) carries this residual-risk statement:**

> Pseudonymisation in APR is a non-reversible compression. Re-identification risk depends on the principal-id space; see `PRIVACY-REVIEW.md` for the v1 conclusion and residual-risk guidance.

Issuers SHOULD:

- Treat `public_minimal` as "low public friction," not as anonymisation.
- Avoid placing personally-identifying values into `principalId` for individuals whose receipts will be served publicly. Prefer durable but opaque per-issuer-per-principal random ids.
- Choose `metadata_only` over `public_minimal` for receipts about individual data subjects when the receipt does not need a public body.
- Document, in their own privacy notice, that APRs are emitted, that they are anchored daily, and what `public_minimal` actually exposes.

Issuers MUST NOT rely on `principalPseudonym` to satisfy GDPR anonymisation requirements. Such reliance is unsupported by the protocol design.

## 6. Items deferred to v1.x or v2

The following privacy improvements were considered for v1 and deferred:

- **Wider pseudonym (e.g. 128-bit truncation).** Increases dictionary-attack cost from "trivial" to "trivial within reason." Does not change the fundamental property that any adversary holding the candidate set can re-identify. Deferred to v1.x as an additive optional second pseudonym field; not adopted in v1 to keep the disclosure surface minimal.
- **Salted pseudonyms with per-issuer salts.** Materially changes correlation properties: receipts from different issuers about the same principal stop being linkable. Considered but rejected for v1 because the salt itself becomes a recovery key the issuer must manage; the trade-off is real but adds complexity. Candidate for v2.
- **Per-receipt pseudonyms.** Rejected for v1: defeats the protocol's correlation utility for legitimate consumers.
- **Differentially-private aggregates instead of per-receipt disclosure.** Rejected for v1 as it changes the entire surface; out of APR's scope.

## 7. Review trail

| Reviewer | Role | Date | Outcome |
|---|---|---|---|
| Reference engineering pass | spec author | repo public-flip date | drafted as part of Phase 0; mirrored into `v1.md` §6.4 |
| `business-legal` Mode A | legal | before public flip | EUIPO-aligned + GDPR posture sanity-check (deliverable: signed-off draft retained internally, not in repo) |

Subsequent material updates to disclosure semantics MUST regenerate this review and update both the conclusion and the spec-body mirror.
