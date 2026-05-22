# ACR v1 — privacy review (Phase 0 scaffold)

> **Status — 2026-05-16.** Phase 0 scaffold. Companion to [`THREAT-MODEL.md`](./THREAT-MODEL.md). Expanded during v1 spec authoring; this version is normative for the assertions in §5 ("Conclusion").

This is not legal advice. It is a structured engineering analysis of identifiability under documented disclosure semantics, written so a Data Protection Officer or external reviewer can assess fitness for purpose.

## 1. Scope

The review covers ACR's three disclosure tiers (`metadata_only`, `selective_disclosure`, `public_full`) under the regulations ACR is designed against: GDPR Art. 4–17, the EU AI Act consent and oversight clauses, HIPAA §164.508, KYC/AML record-keeping, MiFID II suitability, and eIDAS 2.0 EUDI-Wallet compatibility.

The review does NOT cover the truthfulness of the underlying consent (a legal question), nor the lawfulness of the controller's processing purpose (a per-deployment question).

## 2. What each tier discloses

Per the ACR design doc §3 and D5:

| Tier | Public anchored content | Auditor-disclosable content | Subject-disclosable content |
|---|---|---|---|
| `metadata_only` | Outer hash + chain position | Body byte hash only | Body (on subject request via controller) |
| `selective_disclosure` | Outer hash + per-field commitments | Per-field reveals (auditor presents salt + plaintext, verifier confirms `sha256(salt‖value)` matches commitment) | Body |
| `public_full` | Full body verbatim | Same as public | Same as public |

The outer anchored root carries the same shape regardless of tier: `{ rootHash, date, kid, sig }` plus the per-day Merkle tree of receipt ids. The anchored root **never** carries plaintext principal data in any tier. This is structurally different from APR's `public_minimal`, which serves a truncated pseudonym in the open.

## 3. Cryptographic commitment analysis

The privacy-load-bearing primitive in ACR is the per-field salted commitment used in `selective_disclosure`:

```
commit(field, value) = sha256(salt || value)
where salt is 16 random bytes per field per receipt
```

### 3.1 Preimage resistance

SHA-256 is preimage-resistant. Given the commitment alone, recovering `value` requires brute-forcing the 128-bit salt × |value| candidate space. For typical principal identifiers (email, opaque user id) the salt makes brute force intractable.

### 3.2 Dictionary attack — the salt is load-bearing

If the salt leaks, the commitment collapses to `sha256(known_salt || value)` and a dictionary attack on `value` becomes trivial — identical to APR's `principalPseudonym` problem but worse, because the original APR pseudonym was openly published while ACR's salts are *meant* to be confidential.

**Required deployer assumption:** salts are confidential to controller + auditor and never logged in cleartext. Salt rotation on suspected leak is operational; the protocol cannot recover privacy after salt disclosure.

### 3.3 Cross-receipt correlation

Two receipts about the same principal use **different** salts per field per receipt. Their commitments differ even for identical plaintext. An observer of the public log cannot link receipts by principal commitment alone — a substantial improvement over APR's stable pseudonym.

Auditors with access to both salts CAN link, which is the intended behaviour (auditor's job is to confirm consent existed for a named subject across the audit period).

### 3.4 Withdrawal-chain linkability

The chained withdrawal mechanism (design D4) requires withdrawals to reference the prior receipt's id. The chain itself is therefore linkable on the public surface — anyone observing the anchored roots can see "receipt X was followed by withdrawal Y was followed by re-grant Z." This is intentional: regulators need to verify the chain. Identifying *whose* chain it is requires either the salts (auditor path) or out-of-band correlation. The latter is the residual risk addressed in §3.5.

### 3.5 Side-channel correlation

Combinations of `policyRef`, `policyHash`, `controller.kid`, `timestamps`, `chainPosition`, and behavioural cadence form a fingerprint. For a controller with one policy and many subjects, this fingerprint is weak (every subject hits the same policy at roughly the same time). For controllers with rare policies or unusual cadences, the fingerprint can narrow the candidate set on its own. ACR cannot eliminate this; it can only avoid making it worse.

## 4. GDPR posture

### 4.1 Lawful basis

ACR receipts are records of processing. The controller chooses the lawful basis for processing the underlying personal data; ACR records that choice in `lawfulBasis` (per Art. 6(1)). The receipt itself is processed on **legitimate interest (Art. 6(1)(f)) in providing tamper-evident audit trails**. Recital 49 supports legitimate-interest processing for purposes including the security of network and information systems; tamper-evident audit records are within that envelope. Per-deployment DPIA still required where Art. 35 thresholds are met.

### 4.2 Data subject rights

- **Right of access (Art. 15).** Controllers retain the salt+plaintext map keyed by subject and can produce a subject's receipts on request.
- **Right to rectification (Art. 16).** N/A for the historical record. A controller who recorded incorrect consent corrects it by issuing a new chain entry, never by mutating the original.
- **Right to erasure (Art. 17).** Same structural tension as APR's PRIVACY-REVIEW.md §4.2. Mitigations specific to ACR: (a) default to `metadata_only` for individual subjects; an erasure request is honoured by ceasing to serve the body and the salt map, leaving only the outer anchored hash, which carries no plaintext; (b) where `selective_disclosure` was used, destroying the salt map renders the commitments preimage-protected to the same level as APR's `public_minimal` minus the truncation weakness — i.e. cryptographically strong against dictionary attack, but the *fact of the receipt* remains anchored.
- **Right to withdraw consent (Art. 7(3)).** First-class. Withdrawal is a receipt type (design D4). The withdrawal MUST be issued within the regulatory deadline (typically Art. 12(3) one-month window). Per the threat model §3.4, antedating is detected as `missing-from-anchor`.
- **Right to data portability (Art. 20).** Receipts ARE the portable artefact. A subject can be handed their full set of receipts plus salts + plaintext as a self-verifying archive.

### 4.3 Pseudonymisation under GDPR

ACR's per-field salted commitments are technical pseudonymisation in the GDPR Art. 4(5) sense when the salt+plaintext map is retained by the controller. They count as an appropriate safeguard under Recital 28. When the controller destroys the salt map (e.g., post-retention-period or post-erasure-request), the commitments become preimage-resistant to dictionary attack — closer to anonymisation, though "anonymisation" remains a contextual GDPR judgement that depends on residual identifiability via other channels.

### 4.4 Cross-border transfer

The anchored root carries no plaintext personal data and can therefore be replicated to verifiers in any jurisdiction without triggering GDPR Chapter V transfer requirements. The body + salt map ARE personal data and travel under the controller's normal transfer rules (SCCs, adequacy, EU-US DPF, etc.).

## 5. Conclusion

ACR's disclosure design is **materially stronger than APR's** for principal privacy, because the public surface carries no plaintext or truncated pseudonyms — only outer hashes and salted commitments. Concrete privacy properties of v1:

1. The public anchored root reveals nothing about principal identity without out-of-band knowledge.
2. Per-receipt salts prevent cross-receipt correlation by public observers.
3. Withdrawal chains are publicly linkable as chains, but not to identified subjects.
4. Salt confidentiality is the single load-bearing operational assumption. Salt leak collapses privacy on that field for that receipt; controllers MUST treat the salt store with the same care as the signing-key store.
5. ACR remains in-scope for GDPR. Pseudonymisation does not move data out of scope.

**Therefore, the spec body will carry this residual-risk statement:**

> ACR's privacy properties depend on the confidentiality of per-field salts. Controllers MUST treat the salt store as a secrets vault. Salt loss reduces the commitment to a preimage-resistant hash of guessable input — see `PRIVACY-REVIEW.md`.

Controllers SHOULD:

- Default to `metadata_only` for receipts about individual data subjects when the receipt does not need an auditor-disclosable body.
- Use `selective_disclosure` only when auditor access to specific fields is required.
- Reserve `public_full` for receipts about organisations, not natural persons.
- Document, in their own privacy notice, that ACR receipts are emitted, anchored daily, and what each tier exposes.
- Generate per-field salts with CSPRNG, never reuse, never log in cleartext, and rotate the salt store on any suspected compromise.

Controllers MUST NOT rely on the salted commitment to satisfy GDPR anonymisation requirements while the salt+plaintext map is retained. After destruction of the salt map AND the plaintext map, the commitment alone is preimage-protected; whether that constitutes "anonymisation" for GDPR purposes is a per-deployment legal judgement.

## 6. Items deferred to v1.x or v2

- **Wider per-field salts (32 bytes).** Increases brute-force margin. Considered for v1; rejected because 16 bytes already exceeds the practical brute-force ceiling. Candidate if quantum-era preimage attacks become a concern.
- **Per-controller secret salts (in addition to per-field).** Materially raises the cost of a salt-store breach: an attacker who compromises one receipt's salts cannot link to other receipts even with the per-field salts. Adds complexity to the controller's key management. Candidate for v2.
- **Encrypted body at rest (zero-knowledge tier).** Out of v1 scope. Controllers can layer their own at-rest encryption on the body store.
- **Differential-privacy aggregates instead of per-receipt disclosure.** Out of v1 scope — changes the entire surface.

## 7. Review trail

| Reviewer | Role | Date | Outcome |
|---|---|---|---|
| Reference engineering pass | spec author | 2026-05-16 | Phase 0 scaffold drafted alongside design doc |
| `business-legal` Mode A | legal | before v1 spec freeze | EUIPO-aligned + GDPR posture sanity-check (deliverable retained internally) |

Subsequent material updates to disclosure semantics MUST regenerate this review and update both the conclusion and the spec-body mirror.
