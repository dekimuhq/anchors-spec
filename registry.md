# Anchored Receipts — Family REGISTRY

**Authority:** This registry is the single source of truth for `claim_type` discriminators across the Anchored Receipts family. The shared envelope (`@dekimuhq/anchors-envelope`) enforces this union — any new `claim_type` must be added here, with a corresponding spec, before being accepted in production verifiers. **Standalone attestations** (non-receipt signed artifacts that share the `ar.<noun>.v<N>` namespace but carry no `claim_type` and no v=2 envelope) are registered separately in the [Standalone attestations](#standalone-attestations-non-receipt-signed-artifacts) section below; see [family-consistency.md §8](family-consistency.md#8-standalone-attestations-non-receipt-signed-artifacts) for the cross-family invariants.

**Wire format convention:** All members use canonical `ar.<noun>.v<N>` form (e.g., `ar.provenance.v1`, `ar.consent.v1`, `ar.delegation.v1`, `ar.lineage.v1`). The `ar.` namespace identifies the family at the wire level; member acronyms (APR, ACR, ARR, …) survive only as doc shorthand. This eliminates external acronym collisions (ADR ↔ Architecture Decision Record, ABR ↔ Adaptive Bitrate, ASR ↔ Automatic Speech Recognition) at the protocol layer. See addendum at the end of 2026-05-16-anchors-family-closure-brainstorm.md for the decision history (one mid-day reversal to member-prefix form, itself re-reversed before any code shipped). Pre-Phase-A rename swept all shipped APR/ACR/ARR code + tests + vectors + specs; no receipt has ever left the dev environment in member-prefix form.

**Plan source:** Dekimu Labs/docs/plans/2026-05-16-anchors-v1-1-hardening-plan.md — Delta E (Phase 0 scaffolding, Task H1).

**v1.2 amendment (2026-05-17):** Envelope adds optional `kind` ∈ `{observation, decision, null}`, `cause` (closed enum), `predicate` (JSON Logic AST), `iat_ms`. v1.2 `kind` adoption per member is listed below. Spec: Dekimu Labs/docs/specs/2026-05-17-anchors-v1-2-envelope-amendment.md. Decision: Dekimu Labs/docs/decisions/2026-05-17-anchors-v1-2-l1-l9-approved.md.

---

## v1.2 `kind` adoption per member

Per spec §6 matrix. `kind` is OPTIONAL on every member; the values below are the defaults v1.2 issuers SHOULD emit. `predicate` adoption is informational — not enforced by the verifier (recommended for ACR consent, ADR delegation, ATR transfer, APuR purpose, AER evaluation; forbidden on `ar.provenance.v1` and `ar.batch.v1`).

| claim_type | `kind` | `predicate` |
|---|---|---|
| `ar.provenance.v1` | `null` | **Forbidden** |
| `ar.consent.v1` | `decision` | Recommended (lawful basis / age-gate / purpose) |
| `ar.withdrawal.v1` | `decision` | Optional |
| `ar.modification.v1` | `decision` | Optional |
| `ar.erasure.v1` | `decision` | Optional |
| `ar.expiry.v1` | `decision` | Optional |
| `ar.hold.v1` | `decision` | Optional |
| `ar.anonymization.v1` | `decision` | Optional |
| `ar.archive.v1` | `decision` | Optional |
| `ar.batch.v1` | `null` | **Forbidden** (predicates live on members) |
| `ar.delegation.v1` | `decision` | Recommended (delegation predicate) |
| `ar.transfer.v1` | `decision` | Recommended (transfer-allowed predicate) |
| `ar.purpose.v1` | `decision` | Recommended (purpose-allowed predicate) |
| `ar.evaluation.v1` | `observation` | Recommended (evaluation rule) |
| `ar.attestation.v1` | `decision` | Optional |
| `ar.lineage.v1` | per-event | Optional |
| `ar.notice.v1` | `decision` | Optional |
| `ar.breach.v1` | `decision` | Optional |
| `ar.subject_rights.v1` | `decision` | Optional |
| `ar.impact.v1` | `decision` | Optional |
| `ar.tokenization.v1` | `decision` | Optional |
| `ar.action.v1` | `decision` | Optional |
| `ar.quality.v1` | `observation` | Optional |
| `ar.guard.v1` | `observation` | Optional |
| `ar.grounding.v1` | `observation` | Optional |
| `ar.trace.v1` | `observation` | Optional |
| `ar.redaction.v1` | `observation` | Optional |

---

## Shipped Members (v1.0)

| claim_type | Purpose | Spec | Packages | Status |
|---|---|---|---|---|
| `ar.provenance.v1` | Records the actor, timestamp, and method of artifact creation (AI Act Art. 53 GPAI provenance) | Code-only (shipped 2026-05-11) | `@dekimuhq/apr-verifier` + `@dekimuhq/apr-issuer-kit` | shipped |
| `ar.consent.v1` | Records consent given by a data subject (GDPR Art. 7, AI Act Art. 14, HIPAA §164.508) | 2026-05-15-acr-anchored-consent-receipts-design.md | `@dekimuhq/acr-verifier` + `@dekimuhq/acr-issuer-kit` + `@dekimuhq/acr-profiles` | shipped |
| `ar.withdrawal.v1` | Records consent withdrawn by a data subject (GDPR Art. 7(3)) | 2026-05-15-acr-anchored-consent-receipts-design.md | `@dekimuhq/acr-verifier` + `@dekimuhq/acr-issuer-kit` + `@dekimuhq/acr-profiles` | shipped |
| `ar.modification.v1` | Records a consent scope or purpose modified by a data subject (GDPR Art. 7(4), Art. 12(3)) | 2026-05-15-acr-anchored-consent-receipts-design.md | `@dekimuhq/acr-verifier` + `@dekimuhq/acr-issuer-kit` + `@dekimuhq/acr-profiles` | shipped |
| `ar.erasure.v1` | Records the execution of a data subject's right to erasure (GDPR Art. 17, Art. 12(3)) | 2026-05-16-arr-anchored-retention-receipts-design.md | `@dekimuhq/arr-verifier` + `@dekimuhq/arr-issuer-kit` + `@dekimuhq/arr-profiles` | shipped |
| `ar.expiry.v1` | Records scheduled deletion of data after a retention period expires (GDPR Art. 5(1)(e)) | 2026-05-16-arr-anchored-retention-receipts-design.md | `@dekimuhq/arr-verifier` + `@dekimuhq/arr-issuer-kit` + `@dekimuhq/arr-profiles` | shipped |
| `ar.hold.v1` | Records a legal hold preventing deletion (compliance, litigation, regulatory) | 2026-05-16-arr-anchored-retention-receipts-design.md | `@dekimuhq/arr-verifier` + `@dekimuhq/arr-issuer-kit` + `@dekimuhq/arr-profiles` | shipped |
| `ar.anonymization.v1` | Records anonymization of personal data in place of deletion (GDPR Art. 11, Art. 17(2)) | 2026-05-16-arr-anchored-retention-receipts-design.md | `@dekimuhq/arr-verifier` + `@dekimuhq/arr-issuer-kit` + `@dekimuhq/arr-profiles` | shipped |
| `ar.archive.v1` | Records transition of data to long-term archive (GDPR Art. 5(1)(e), Art. 12(3)) | 2026-05-16-arr-anchored-retention-receipts-design.md | `@dekimuhq/arr-verifier` + `@dekimuhq/arr-issuer-kit` + `@dekimuhq/arr-profiles` | shipped |
| `ar.batch.v1` | Records batch processing of multiple records in a single transaction (Merkle tree batching) | 2026-05-16-arr-anchored-retention-receipts-design.md | `@dekimuhq/arr-verifier` + `@dekimuhq/arr-issuer-kit` + `@dekimuhq/arr-profiles` | shipped |
| `ar.lineage.v1` | Records data-derivation lineage and disclosure chain (GDPR Art. 13 transparency, AI Act Art. 53 GPAI training data) — 4 event types (`derived_from`, `forked_from`, `superseded_by`, `disclosed`) + 5 regulatory profiles | 2026-05-16-alr-anchored-lineage-receipts-design.md | `@dekimuhq/alr-verifier` + `@dekimuhq/alr-issuer-kit` + `@dekimuhq/alr-profiles` | shipped |
| `ar.notice.v1` | Records privacy notice delivery and acknowledgement (GDPR Arts. 12–14, Art. 26 essence, AI Act Art. 50) — 6 event types (`notice.presented`, `notice.re_presented`, `notice.acknowledged`, `notice.withdrawn`, `notice.translated`, `notice.joint_controller_essence`) + 5 regulatory profiles. ACR↔ANR composition (mandatory on v1.2+ ACR issuers) + optional ALR translation provenance | 2026-05-17-anr-anchored-notice-receipts-design.md | `@dekimuhq/anr-verifier` + `@dekimuhq/anr-issuer-kit` + `@dekimuhq/anr-profiles` | shipped |
| `ar.breach.v1` | Records breach detection, assessment, and notification lifecycle (GDPR Arts. 33–34, NIS2 Art. 23, DORA Art. 19) — 8 event types (`breach.detected`, `breach.assessed`, `breach.dpa_notified`, `breach.dpa_delayed`, `breach.subject_notified`, `breach.subject_notification_exempted`, `breach.contained`, `breach.closed`) + 5 regulatory profiles. MANDATORY TSA on `breach.detected` (the 72-hour Art. 33 clock starts here, D10-ABR-2). 5-field composition surface (ARR/ATR/AER/ANR/AAR) per audit-fix C5 | 2026-05-17-abr-anchored-breach-receipts-design.md | `@dekimuhq/abr-verifier` + `@dekimuhq/abr-issuer-kit` + `@dekimuhq/abr-profiles` | shipped |
| `ar.subject_rights.v1` | Records data-subject rights requests across the GDPR Arts. 15/16/17/18/20/21 lifecycle — 9 event types (`request.received`, `request.identity_verified`, `request.scope_clarified`, `request.extended`, `request.partially_honoured`, `request.fulfilled`, `request.refused`, `request.escalated`, `request.recipient_notified`) + 6 regulatory profiles. D10-ASR-3 MANDATORY ARR cross-link on `request.fulfilled` under the Art. 17 profile; uniform `sha256:<hex>` subject across all events per D10-ASR-5. First family minted via factory-from-day-1 through `defineFamily()` (Phase 3 of the A.2-full plan). Composition surface (Art. 17 ↔ ARR, Art. 19 ↔ ALR, Art. 20 ↔ ATR, Art. 21 ↔ APuR) deferred to Phase 3 §G | 2026-05-17-asr-anchored-subject-rights-receipts-design.md | `@dekimuhq/asr-verifier` + `@dekimuhq/asr-issuer-kit` + `@dekimuhq/asr-profiles` | shipped |
| `ar.impact.v1` | Records DPIA and prior-consultation lifecycle (GDPR Arts. 35–36; AI Act FRIA via Wave 3 profile extension; AI Act conformity assessment proper handled by `ar.evaluation.v1`) — 11 wire event types (`dpia.threshold_triggered`, `dpia.scope_locked`, `dpia.dpo_advised`, `dpia.stakeholders_consulted`, `dpia.completed`, `dpia.reviewed`, `dpia.prior_consultation_initiated`, `dpia.prior_consultation_resolved`, terminal trio `dpia.processing_authorised` / `dpia.processing_blocked` / `dpia.terminated`) + 5 regulatory profiles. RECOMMENDED-TSA matrix on the three deadline-bearing events (`dpia.completed`, `dpia.prior_consultation_initiated`, `dpia.prior_consultation_resolved`); AIR has no MANDATORY-TSA event in v1. Subject is per-event-split (FAMILY-CONSISTENCY §3 + D10-AIR-6): `null` on every event EXCEPT `dpia.stakeholders_consulted` where `sha256:<hex>` MAY anchor a representative-org / NGO-review `sub_commit` ONLY when `consultation_method ∈ {representative_org, ngo_review}`. Status List 2021 consumer position #6. 5-field composition surface (AER/ATR/ALR/APuR/ABR) — verifier accepts cross-issuer ids as opaque; cross-family walks deferred until AER/ATR/APuR land. Renamed 2026-05-17 (audit fix C8) from `ar.dpia.v1` (no receipts ever minted under old name). | 2026-05-17-air-anchored-impact-assessment-receipts-design.md | `@dekimuhq/air-verifier` + `@dekimuhq/air-issuer-kit` + `@dekimuhq/air-profiles` | shipped |
| `ar.purpose.v1` | Records purpose binding and secondary-use controls (GDPR Art. 5(1)(b), Art. 30 RoPA) — 3 event types (`purpose.binding`, `purpose.repurpose`, `purpose.unbinding`) + 4 regulatory profiles | 2026-05-16-apur-anchored-purpose-receipts-design.md | `@dekimuhq/apur-verifier` + `@dekimuhq/apur-issuer-kit` + `@dekimuhq/apur-profiles` | shipped |
| `ar.evaluation.v1` | Records AI conformity assessment and transparency obligations (AI Act Arts. 43, 50; ISO 42001) — 2 event types (`conformity`, `transparency`) under 9 regulatory profiles (AI Act Arts. 43/50/52 + NIST AI RMF). Per-event subject split: null on conformity (system-level), sha256 on transparency (end-user). TSA RECOMMENDED on conformity. Conformity chains via prev; transparency receipts are leaf nodes referencing conformity by hash. | 2026-05-16-aer-anchored-evaluation-receipts-design.md | `@dekimuhq/aer-verifier` 0.1.0 + `@dekimuhq/aer-issuer-kit` 0.1.0 + `@dekimuhq/aer-profiles` 0.1.0 | shipped |
| `ar.tokenization.v1` | Records PII tokenization lifecycle (token issued, redeemed, revoked, rotated) — salted commit only, no plaintext PII. Scope-binding (audience + purpose + validity). PCI HVT/LVT vocabulary mapping in FAMILY-CONSISTENCY §9. | 2026-05-21-atokr-anchored-tokenization-receipts-design.md | `@dekimuhq/atokr-verifier` 0.1.0 + `@dekimuhq/atokr-issuer-kit` 0.1.0 + `@dekimuhq/atokr-profiles` 0.1.0 | shipped |
| `ar.delegation.v1` | Records delegation of authority and scope (GDPR Art. 28 processor, AI Act Arts. 14/26 deployer/overseer) — 6 event types (`delegation.granted`, `delegation.revoked`, `delegation.suspended`, `delegation.reinstated`, `delegation.scope_amended`, `delegation.expired`) + 5 regulatory profiles (GDPR Art. 28, AI Act Art. 14, AI Act Art. 26, corporate-officer, agentops-scoped). Scope-subset enforcement: each delegation verifies it is within the parent's scope. Sub-delegation tree tracked via `prev` chain. | 2026-05-16-adr-anchored-delegation-receipts-design.md | `@dekimuhq/adr-verifier` 0.1.0 + `@dekimuhq/adr-issuer-kit` 0.1.0 + `@dekimuhq/adr-profiles` 0.1.0 | shipped (2026-05-22) |
| `ar.transfer.v1` | Records cross-border transfer of personal data (GDPR Chapter V, Schrems II) — 4 event types (`arrangement`, `transfer`, `tia`, `suspension`), 8 GDPR/Schrems II profiles. TSA RECOMMENDED on `arrangement` + `suspension`; OPTIONAL on `transfer` + `tia`. Per-event subject split: `transfer` carries `sha256:<hex>`, others `null`. Transfer validity computation (spec §7.2) verifies arrangement in-force, TIA validity, suspension gate. | 2026-05-16-atr-anchored-transfer-receipts-design.md | `@dekimuhq/atr-verifier` 0.1.0 + `@dekimuhq/atr-issuer-kit` 0.1.0 + `@dekimuhq/atr-profiles` 0.1.0 | shipped (2026-05-22) |
| `ar.attestation.v1` | Records qualified trust service issuance (eIDAS Arts. 22, 25, 32–34; ETSI standards) — 2 event types (`issuance`, `revocation`), 8 eIDAS profiles, TSL snapshot evidence. TSA MANDATORY on `issuance` with `service_type: qts`; RECOMMENDED on all others. Subject must-be-present (`sha256:<hex>`) on both events. | 2026-05-16-aar-anchored-attestation-receipts-design.md | `@dekimuhq/aar-verifier` 0.1.0 + `@dekimuhq/aar-issuer-kit` 0.1.0 + `@dekimuhq/aar-profiles` 0.1.0 | shipped (2026-05-21) |
| `ar.action.v1` | Records an autonomous action an automation engine executed on a workspace's behalf — single shape, 1 profile (`hub.automation`). Null-subject (workspace-level); plaintext `verb`/`rule_id`/`recipe_id`/`outcome`/`executed_at` + 3 salted target commits (`decision_commit`/`params_commit`/`request_id_commit`). OPTIONAL mandate binding (`credential_id` + `caveats_consumed`, plus `delegation_chain` — `{depth, leaf_key, root_credential_id}` — when a sub-agent acted under an attenuated sub-mandate) when the action ran under a scoped agent credential; absent for policy-authorized auto actions. Status-List non-consumer (non-revocable event); TSA OPTIONAL (no regulatory clock). | [action/v1.md](action/v1.md) | `@dekimuhq/action` + `@dekimuhq/action-profiles` | shipped |
| `ar.quality.v1` | Records output/version quality-eval provenance (provenance not truth) — 2 profiles (`quality.output`, `quality.release`). Null-subject (must-be-null); Status-List non-consumer; TSA OPTIONAL. DISTINCT from AER (`ar.evaluation.v1`, AI Act conformity). | [quality/v1.md](quality/v1.md) | `@dekimuhq/quality` + `@dekimuhq/quality-profiles` | shipped |
| `ar.guard.v1` | Records a deterministic prompt-injection / input-screening verdict (provenance of screening, NOT a safety guarantee) — verdict `pass`/`flagged`/`blocked`; golden-pinned `ruleset_hash`. Null-subject (must-be-null); Status-List non-consumer; TSA OPTIONAL. DISTINCT from AER and AQR. | [guard/v1.md](guard/v1.md) | `@dekimuhq/guard` + `@dekimuhq/guard-profiles` | shipped |
| `ar.grounding.v1` | Records RAG citation-binding + grounding-support provenance for generated answers (binding authoritative, score advisory; provenance not truth). Null-subject (must-be-null); Status-List non-consumer; TSA OPTIONAL. | [grounding/v1.md](grounding/v1.md) | `@dekimuhq/grounding` + `@dekimuhq/grounding-profiles` | shipped |
| `ar.trace.v1` | Records per-run agent-execution provenance (provenance not behavior) — 1 profile (`agent.trace`). Null-subject (must-be-null); Status-List non-consumer; TSA OPTIONAL. DISTINCT from AActR (single action vs whole run). | [trace/v1.md](trace/v1.md) | `@dekimuhq/trace` + `@dekimuhq/trace-profiles` | shipped |
| `ar.redaction.v1` | Records which PII-redaction pass ran over content before model send (provenance of redaction, NOT a PII-free guarantee) — 1 profile (`redaction.prompt`); per-category counts + dual salted commits (input/output) + `ruleset_hash`; reversible map+salt are controller-only, never anchored. Null-subject (must-be-null); Status-List non-consumer; TSA OPTIONAL. DISTINCT from ATokR and `ar.anonymization.v1`. | [redaction/v1.md](redaction/v1.md) | `@dekimuhq/redaction` + `@dekimuhq/redaction-profiles` | shipped |

> **Note:** `ar.continuity.v1` previously appeared as a row above. It is a **standalone attestation**, not a `claim_type` — registered in the [Standalone attestations](#standalone-attestations-non-receipt-signed-artifacts) section below per the FAMILY-CONSISTENCY §8 generalisation (2026-05-19, cross-issuer interop doc-only gate).

---

## Specced Members (reserved slots, v1.0 baseline design complete)

_All specced members have shipped. This section is intentionally retained for completeness._

> **AIR (`ar.impact.v1`) shipped 2026-05-19** — moved to Shipped Members table above.
> **ATR (`ar.transfer.v1`) + AAR (`ar.attestation.v1`) shipped 2026-05-22** — moved to Shipped Members table above. Specced section now empty.

---

## Parked Members (family-closure brainstorm, awaiting founder review)

All 4 wave-2 closure-set candidates have now moved to **specced** above (audit fix C8, 2026-05-17). This section is intentionally retained to surface that the 13-member v1 closure set is fully specced at design level.

_No parked members remaining as of 2026-05-17. Family v1 closure set spec-complete at 13 members. ATokR (14th member) adopted 2026-05-21 from the 4 reserved discriminator slots; 3 parked slots remain._

---

## Standalone attestations (non-receipt signed artifacts)

Standalone attestations share the canonical `ar.<noun>.v<N>` namespace with member receipts but carry **no `claim_type`, no `field_root`, no `subject`, and no v=2 envelope wrapper**. They certify operational state (key continuity, trust scope, issuer self-description) rather than per-subject events. Cross-family invariants — JCS canonicalisation, ed25519, `kind` discriminator, owned by an `@dekimuhq/anchors-envelope/<subpath>` export — live in [family-consistency.md §8](family-consistency.md#8-standalone-attestations-non-receipt-signed-artifacts).

Standalone attestations are **not** part of the envelope `claim_type` union and are not subject to the v1.2 `kind`/`predicate` matrix above.

| Discriminator (`kind`) | Scope | Purpose | Spec / plan | Packages | Status |
|---|---|---|---|---|---|
| `ar.continuity.v1` | per-family | Issuer key-rotation attestation; each family member's issuer-kit re-exports the helpers and pins its own `family` slug at issue time. Promoted from per-family `<family>.continuity.v1` (`alr` / `arr` / `anr` issuer-kits) to the canonical `ar.<noun>.v<N>` wire form in envelope 0.7.0. Hard-cut migration applied 2026-05-19 (pre-launch); no legacy `<family>.continuity.v1` parser shipped. | 2026-05-18-anchors-continuity-v1-unification.md — see FAMILY-CONSISTENCY §8.1 | `@dekimuhq/anchors-envelope/continuity` (single source) — re-exported by `@dekimuhq/alr-issuer-kit ≥ 0.2.0` + `@dekimuhq/arr-issuer-kit ≥ 0.5.0` + `@dekimuhq/anr-issuer-kit ≥ 0.2.0` (APR / ACR / ATR / APuR / AER / AAR / ADR / ASR / ABR / AIR adoption deferred to per-family spec) | shipped |
| `ar.trusted_issuers.v1` | family-wide | Signed allowlist manifest; one signed list governs trust dispatch for every family member's verifier. Published at `verify.dekimu.com/.well-known/dekimu-trusted-issuers.json`; append-only mirror at `dekimuhq/anchors-trusted-issuers-chain`. Body: `{ issuers, generated_at, sequence, prev_root \| null }` (sequence + prev_root form a hash chain). Signing key: pinned `BOOTSTRAP_PUBLISHER_KEY` (Dekimu Labs SL family key in v1; 1-year rotation with 90-day overlap). Monthly cron re-sign + republish. Consumed by `verifyIssuerKey` via the default `TrustedIssuersSource`. | 2026-05-19-anchors-cross-issuer-interop.md §2 — see FAMILY-CONSISTENCY §8.2 | `@dekimuhq/anchors-trusted-issuers` (Phase B); signing helper expected at `@dekimuhq/anchors-envelope/trust-manifests` for shape consistency with `/continuity` | specced (Phase B) |
| `ar.issuer_manifest.v1` | family-wide (per-issuer instance) | Per-issuer self-published manifest: each `did:web` issuer publishes ONE `dekimu-issuer.json` at their own `/.well-known/` path advertising which family memberships they emit, which `claim_type`s they support, which profiles they implement, and a pointer to their `dekimu-keys.json`. Body: `{ iss, families: FamilySlug[], claim_types: string[], profiles: Record<claim_type, profile[]>, keys_url, generated_at, expires_at }`. Self-signed with the issuer's own family key (the one referenced by `keys_url`); cross-verified via did:web resolution. Verifier surfaces a `profile_scope_mismatch` warning when receipts in the wild advertise profiles not declared in this manifest (does not auto-reject). `expires_at` advisory; the trusted-issuers allowlist (`ar.trusted_issuers.v1`) is the authoritative revocation surface. | 2026-05-19-anchors-cross-issuer-interop.md §3 — see FAMILY-CONSISTENCY §8.3 | `@dekimuhq/anchors-federation` (Phase C) for fetch + verify (`fetchIssuerManifest` + `runHandshake`); signing helper colocated with `ar.trusted_issuers.v1` under `@dekimuhq/anchors-envelope/trust-manifests` | specced (Phase C) |

**FamilySlug reuse:** the closed `FamilySlug` union (`apr | acr | arr | alr | atr | apur | aer | aar | adr | anr | asr | abr | air | atokr`) is exported once from `@dekimuhq/anchors-envelope/continuity` and reused by `ar.trusted_issuers.v1` (per-issuer `families` arrays) and `ar.issuer_manifest.v1` (`families` field). Single source of truth for the continuity-family taxonomy. **AActR (`ar.action.v1`) is intentionally NOT in this union** — it is a Status-List/continuity non-consumer (an autonomous action is a non-revocable event), so it is the 15th *receipt* family (chip 15) without being a member of the continuity `FamilySlug` taxonomy.

---

## Maintenance

When adding a new `claim_type`:

1. Author or update the spec document in `Dekimu Labs/docs/specs/YYYY-MM-DD-<member>-anchored-<name>-receipts-design.md`.
2. Add a row to this REGISTRY with the discriminator, purpose, spec path, packages, and status (`specced` → `shipped` on code merge).
3. Update the shared envelope union in `@dekimuhq/anchors-envelope` to accept the new discriminator.
4. Update `anchors/meta/CROSS-REFERENCES.md` with any chain relationships (e.g., `ar.subject_rights.v1` → `ar.erasure.v1` for Art. 17 erasure requests).
5. Reference `anchors/templates/spec-template.md` for new spec structure.

When adding a new **standalone attestation** (non-receipt signed artifact under `ar.<noun>.v<N>`):

1. Author the spec or interop-plan section that defines the attestation (per-family or family-wide scope, body shape, signing key, consumer surface).
2. Add a row to the [Standalone attestations](#standalone-attestations-non-receipt-signed-artifacts) section with the `kind` discriminator, scope, purpose, spec path, packages, and status — do **not** add to the Shipped/Specced Members tables.
3. Confirm the attestation honours FAMILY-CONSISTENCY §8 invariants (JCS + ed25519, no envelope, no `claim_type`/`field_root`/`subject`, `@dekimuhq/anchors-envelope/<subpath>` ownership).
4. Add a §8.N subsection to `meta/family-consistency.md` describing per-family or family-wide scope, the body shape table, and wire-format notes.
5. Do **not** add a row to the v1.2 `kind` adoption table — standalone attestations are outside the envelope `claim_type` union.

**Decision linkage:**
- Family shape (13-member closure set): D-anchors-1 — closure brainstorm in 2026-05-16-anchors-family-closure-brainstorm.md §12–13.
- ARR full retention lifecycle (5 events + batch): D-anchors-3.
- Shared envelope home (`@dekimuhq/anchors-envelope`): D-anchors-6.
- Cross-issuer interop plan (introduces `ar.trusted_issuers.v1` Phase B + `ar.issuer_manifest.v1` Phase C): 2026-05-19-anchors-cross-issuer-interop.md.
