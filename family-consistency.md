# Anchored Receipts — Family-wide Consistency Tables

**Authority:** This document is the **single source of truth** for cross-family invariants that would otherwise drift across member specs: chip positions on `verify.dekimu.com/check`, TSA mandate matrix, subject-presence matrix, Status List 2021 consumer ordering, and shared package references.

**Companion to:** [`registry.md`](registry.md) (claim_type registry) + [`CROSS-REFERENCES.md`](CROSS-REFERENCES.md) (composition relationships).

**Created:** 2026-05-17 (closure consistency-audit fix-pass, addressing audit findings H4/H5/H7/H10/H11/H12 per `Dekimu Labs/docs/qa/2026-05-17-anchors-family-consistency-audit.md`).

**Maintenance rule:** Adding a new family member requires updating every table below. The tables ship to the spec template at `anchors/templates/spec-template.md` as a "consistency-checklist" snippet that new spec drafts MUST fill in.

---

## 1. `verify.dekimu.com/check` chip-position matrix

Canonical chip ordering. Each chip represents one family member's filter on the `/check` UI; ordering is locked to avoid mid-roadmap reshuffles. Pre-canonical chips 1–3 are shipped; 4–13 are reserved for the order below.

| Chip # | Family | `claim_type` | Status | Notes |
|---|---|---|---|---|
| 1 | APR | `ar.provenance.v1` | shipped | First shipped family member |
| 2 | ACR | `ar.consent.v1` (+ `ar.modification.v1`, `ar.withdrawal.v1`) | shipped | Single chip covers ACR's 3 claim_types |
| 3 | ARR | `ar.erasure.v1` (+ `ar.expiry.v1`, `ar.hold.v1`, `ar.anonymization.v1`, `ar.archive.v1`, `ar.batch.v1`) | shipped | Single chip covers ARR's 6 claim_types |
| 4 | ADR | `ar.delegation.v1` | shipped | verifier 0.1.0 + issuer-kit 0.1.0 + profiles 0.1.0; 6 event types + 5 regulatory profiles (GDPR Art. 28, AI Act Arts. 14/26, corporate, agentops) |
| 5 | ATR | `ar.transfer.v1` | shipped | 4 event types (arrangement, transfer, tia, suspension); 8 profiles; per-event subject split; transfer validity computation §7.2 |
| 6 | APuR | `ar.purpose.v1` | shipped | 3 event types (binding, repurpose, unbinding); 4 regulatory profiles |
| 7 | AER | `ar.evaluation.v1` | shipped | 2 event types (conformity + transparency), 9 profiles |
| 8 | AAR | `ar.attestation.v1` | specced | |
| 9 | ALR | `ar.lineage.v1` | specced | |
| 10 | ANR | `ar.notice.v1` | specced (closure wave 1) | |
| 11 | ABR | `ar.breach.v1` | shipped | 8 event types under 5 regulatory profiles (GDPR Art. 33/34, NIS2 Art. 23, DORA Art. 19, US-AG) |
| 12 | ASR | `ar.subject_rights.v1` | shipped | 9 event types under 6 regulatory profiles (GDPR Arts. 15/16/17/18/20/21); first family minted via factory-from-day-1 through `defineFamily()` |
| 13 | AIR | `ar.impact.v1` | shipped | 11 wire event types (9 logical + terminal trio) under 5 regulatory profiles (GDPR Art. 35(3)(a-c), Art. 35(4) DPA-list, voluntary); RECOMMENDED-TSA matrix on 3 deadline-bearing events (completed, prior_consultation_initiated, prior_consultation_resolved) — no MANDATORY-TSA event in v1; per-event subject carve-out on `dpia.stakeholders_consulted` (D10-AIR-6) |
| 14 | ATokR | `ar.tokenization.v1` | locked (2026-05-21) | 4 event types (`token.issued`, `token.redeemed`, `token.revoked`, `token.rotated`); PII-free — salted commit only; scope-binding (audience + purpose + validity); format_class discriminator (`pan`, `email`, `iban`, `phone`, `freeform`). Decision: 2026-05-20-atokr-anchored-tokenization-receipts.md |
| 15 | AActR | `ar.action.v1` | shipped | Autonomous Hub-automation action receipt. Single-shape, null-subject (workspace-level); 1 profile (`hub.automation`); plaintext `verb`/`rule_id`/`recipe_id`/`outcome`/`executed_at` + 3 salted target commits (`decision_commit`/`params_commit`/`request_id_commit`); OPTIONAL mandate binding (`credential_id` + `caveats_consumed`); Status-List non-consumer; TSA OPTIONAL. Spec: [action/v1.md](action/v1.md) |

> **Audit fix H4 (2026-05-17):** ANR/ABR/ASR/AIR specs had previously claimed chip positions 6/7/8/9, which collided with AER (positions 4–9 unassigned). This table replaces all per-spec chip-position claims. Member specs SHOULD NOT redeclare chip position; they reference this table instead.

---

## 2. TSA mandate matrix

Per-event TSA (eIDAS RFC 3161 trusted timestamp) requirement. The v1.1 envelope §6 carries `tsa` as a top-level signed-out block. The mandate level is per-event-type, not per-claim-type, because deadline-clock-start events warrant stricter time anchoring than informational events.

**Mandate levels:**

| Level | Meaning | Verifier surface when absent |
|---|---|---|
| **MANDATORY** | Receipt is invalid without `tsa`. | HARD-invalid (`tsa_required`) |
| **RECOMMENDED** | Receipt valid without `tsa`, but verifier surfaces a warning that downgrades verifier confidence for deadline-sensitive queries. | `tsa_recommended` (warning) |
| **OPTIONAL** | No verifier surface. | none |

**Family-wide rule (derived from audit-finding H7 review):** events whose `iat` starts a regulatory clock are **MANDATORY**. Events whose `iat` is the basis for a verifier-side deadline check but where the clock-start is anchored elsewhere are **RECOMMENDED**. All other events are **OPTIONAL**.

| Family | Event | Mandate | Rationale |
|---|---|---|---|
| APR | (any) | OPTIONAL | Provenance receipts; no regulatory clock |
| ACR | `ar.consent.v1` issuance (default profiles) | OPTIONAL | Consent collection; clock is the policy-version anchor |
| ACR | `acr.esig.accept@qes` profile | MANDATORY (incl. `requires_tsa_listed`, `requires_tsa_policy_oid`) | Per v1.1 §6.8 profile-mandate; QES acceptance is legally-defining |
| ARR | `ar.erasure.v1` under `arr.gdpr.art17.erasure-finalized` profile | MANDATORY | Per v1.1 §6.8; erasure execution is legally-defining |
| ARR | other events | RECOMMENDED | Retention decisions cross-checked but not regulatory-clock-start |
| ADR | (any) | OPTIONAL | Delegation; no regulatory clock |
| ATR | `body.event_type: arrangement` | RECOMMENDED | Arrangement clock relevant for transfer-validity checks |
| ATR | `body.event_type: suspension` | RECOMMENDED | Suspension clock relevant for post-suspension transfer invalidation |
| ATR | other events | OPTIONAL | |
| APuR | (any) | OPTIONAL | Purpose binding; no regulatory clock |
| AER | `body.event_type: conformity` | RECOMMENDED | Conformity has `valid_until` clock |
| AER | `body.event_type: transparency` | OPTIONAL | |
| AAR | `body.event_type: issuance` with `service_type: qts` | MANDATORY | QTS issuance is the timestamp itself |
| AAR | `body.event_type: issuance` other service types | RECOMMENDED | |
| AAR | `body.event_type: revocation` | RECOMMENDED | |
| ALR | (any) | OPTIONAL | Lineage; no regulatory clock |
| ANR | `notice.aiact.art50_transparency` profile | MANDATORY + QTS OID `0.4.0.2023.1.1` | AI Act evidence; profile-side eIDAS QTS pinning (EUTL-listed + BTSP OID) |
| ANR | other profiles | RECOMMENDED | Notice presentation deadlines |
| ABR | `body.event_type: detected` | **MANDATORY** (per spec D10-ABR-2) | The 72-hour clock starts here; no exception |
| ABR | other events | RECOMMENDED | All breach-lifecycle events are deadline-sensitive |
| ASR | `body.event_type: received` | RECOMMENDED | 30-day clock starts here; raise to MANDATORY in v1.1 if EUDI Wallet pilots demand stronger anchoring |
| ASR | other events | OPTIONAL | |
| AIR | `body.event_type: completed` | RECOMMENDED | DPIA completion sequence vs processing-start |
| AIR | `body.event_type: prior_consultation_initiated` | RECOMMENDED | Art. 36 clock |
| AIR | `body.event_type: prior_consultation_resolved` | RECOMMENDED | DPA-advice receipt clock |
| AIR | other events | OPTIONAL | |
| ATokR | (any) | OPTIONAL | No regulatory clock starts on a tokenization event in current EU regimes. Re-evaluate if EUDI Wallet ARF or NIS2 sectoral acts impose tokenization deadlines. |
| AActR | (any) | OPTIONAL | No regulatory clock starts on an autonomous action; trusted timestamp is opt-in (rationale as APR/ADR). |

> **Audit fix H7 (2026-05-17):** prior to this table, TSA mandate language varied across closure specs (ABR mandated on `breach.detected`, ASR recommended on `request.received`, AIR recommended on `dpia.completed` — without a principled rule). This table applies a single rule (regulatory-clock-start ⇒ MANDATORY; deadline-checks ⇒ RECOMMENDED; else OPTIONAL). Member specs reference this table; per-spec mandate language MUST match.

---

## 3. Subject-presence matrix

Each family member declares whether the `subject` envelope field is `null`, populated as a salted commit (`"sha256:<hex>"` per v1.1 §5.4), or carries a free-form provenance subject. This table consolidates the per-spec rules so cross-spec invariants (e.g. "ASR↔ARR subject parity") can be enforced uniformly.

| Family | Event | `subject` value | Rationale |
|---|---|---|---|
| APR | (any) | free-form `<anything except "sha256:" prefix>` (artifact URL/id) | Provenance; v1.1 §5.4 |
| ACR | (any) | `"sha256:<hex>"` (HMAC-salted subject commit) | Required per v1.1 §5.4 |
| ARR | (any) | `"sha256:<hex>"` | Required per v1.1 §5.4. Body carries `target_kind` + `target_commit` for finer-grain target discrimination (subject vs system vs object) |
| ADR | (any) | `"sha256:<hex>"` of delegator commit; processor/deployer references in body | |
| ATR | `body.event_type: arrangement` | `null` (arrangement is data-flow-level, not subject-level) | |
| ATR | `body.event_type: transfer` | `"sha256:<hex>"` batch root or per-subject commit | Per-transfer subjects identified via batch leaves |
| ATR | other events | `null` | |
| APuR | (any) | `null` (purpose is processing-activity-level, not subject-level) | |
| AER | `body.event_type: conformity` | `null` (system-level) | |
| AER | `body.event_type: transparency` | `"sha256:<hex>"` (end-user subject) | |
| AAR | `body.event_type: issuance` | `"sha256:<hex>"` of signatory commit | |
| AAR | `body.event_type: revocation` | `"sha256:<hex>"` matching the revoked issuance's subject | |
| ALR | (any) | `null` (lineage is dataset-level) | Body carries source/derivation refs |
| ANR | `body.event_type: notice.acknowledged` | `"sha256:<hex>"` | Acknowledgement is per-subject |
| ANR | other events | `"sha256:<hex>"` (default) OR `null` for jurisdiction-level notice events that have no subject context | |
| ABR | incident-level events (detected, assessed, dpa_notified, dpa_delayed, contained, closed) | `null` | Incident-level, not subject-level |
| ABR | `body.event_type: subject_notified` | `"sha256:<hex>"` batch root | Batch over affected subjects |
| ABR | `body.event_type: subject_notification_exempted` | `null` (the exemption is incident-level, not per-subject) | |
| ASR | (every event) | `"sha256:<hex>"` (MANDATORY per D10-ASR-5) | Every event is subject-scoped |
| AIR | (every event) | `null` (DPIA is processing-activity-level) | EXCEPT `body.event_type: stakeholders_consulted` where representative-org commit MAY appear |
| ATokR | (every event) | `"sha256:<hex>"` (salted commit of the tokenized plaintext) | Every tokenization event is subject-scoped — the receipt anchors the surrogate↔plaintext mapping proof |
| AActR | (any) | `null` (MUST be null) | Workspace/activity-level; the action's target is a body commit (`decision_commit`/`params_commit`), never the envelope subject |

> **Audit fix H10 (2026-05-17):** consolidates subject-presence rules. Member-spec D10 deltas SHOULD reference this table rather than re-declaring rules. Subject-parity invariants between specs (e.g. ASR↔ARR Art. 17) cross-check this table's salted-commit form (`"sha256:<hex>"`).

---

## 4. Status List 2021 consumer ordering

W3C Status List 2021 was introduced in v1.1 hardening (§8). Each family member that consumes Status List 2021 declares an ordering position. The order is anchored to **first-consumer date**, not "Nth family member" (the latter drifts as new members are added).

| Order | Family | Consumer-of-status-list-since | Notes |
|---|---|---|---|
| 1 | ALR | v1.0 (designed-in) | Reference implementation; spec-source for the family-wide consumer pattern |
| 2 | ACR | v1.1 retrofit | First retrofit on shipped code |
| 3 | ARR | v1.1 retrofit | Second retrofit on shipped code |
| 4 | ANR | v1.0 of ANR (= post-v1.1 envelope) | First spec authored with status-list-consumer at design time |
| 5 | ABR | v1.0 of ABR | |
| 6 | ASR | v1.0 of ASR | |
| 7 | AIR | v1.0 of AIR | |
| 8 | ADR | v1.0 of ADR | |
| 9 | ATR | v1.0 of ATR | |
| 10 | APuR | v1.0 of APuR | |
| 11 | AER | v1.0 of AER | |
| 12 | AAR | v1.0 of AAR | |
| 13 | ATokR | v1.0 of ATokR | `token.revoked` flips bit; `token.rotated` flips predecessor bit |

APR currently does NOT consume Status List 2021 (provenance receipts are typically not revocable; immutability is part of their value). Adding APR to Status List 2021 would require a v1.1 APR amendment.

AActR (`ar.action.v1`) likewise does NOT consume Status List 2021 (`status_list_slot: null`): an autonomous action is a non-revocable event that *happened* — there is no suspension/revocation surface in v1. Same posture as APR.

> **Audit fix H5 (2026-05-17):** consolidates per-spec ordering claims that conflicted (multiple specs claimed "Nth"). This table is the canonical sequence; member specs reference it instead of declaring their own ordinal.

### 4.1 Suspension List support

Wave 3 (envelope 0.14.0) added `suspensionStatus` — a second Status List 2021 pointer with `statusPurpose: "suspension"` (two-way, unlike revocation which is permanent). Issuer-kits expose `suspensionList?: SuspensionListConfig` on their mint input.

All 13 families that consume revocation also support suspension. APR does not (non-revocable by design). The `family-mint-pipeline` (`buildMintV2`) supports suspension for any family routed through `defineFamily()` (currently ASR).

---

## 5. Shared-package references (mandatory imports)

Every member spec's anchor flow §7 and verifier package SHOULD reference these shared packages by name. Failing to import implies the spec re-defines envelope fields, leading to drift on next envelope version.

| Package | Purpose | Required for |
|---|---|---|
| `@dekimuhq/anchors-envelope` | v1.1+ shared envelope, canonical-JSON helpers, signing primitives, `JSON_LOGIC_ALLOWLIST_V1_2` constant | Every member's verifier + issuer-kit |
| `@dekimuhq/did-web-resolver` (v0.1.0, shipped 2026-05-17; 0.2.0 adds `verifyIssuerKey` + trust-tier hooks under Phase A of the cross-issuer interop plan) | did:web resolution + InMemoryKeyCache + `trustedLogs` allowlist option (cross-issuer interop) | Every member's verifier that accepts non-Dekimu issuers; closure-set members SHOULD declare cross-issuer interop support in v1 |
| `@dekimuhq/anchors-trusted-issuers` (Phase B of the cross-issuer interop plan, 2026-05-19) | Default `TrustedIssuersSource` implementation: signed allowlist + revocation surface, mirrored at `verify.dekimu.com/.well-known/dekimu-trusted-issuers.json` and hash-chained at `dekimuhq/anchors-trusted-issuers-chain` | Every member's verifier whose default trust dispatch matches Dekimu's policy; OPTIONAL for verifiers that ship their own `TrustedIssuersSource` |
| `@dekimuhq/anchors-federation` (Phase C of the cross-issuer interop plan) | Federated discovery: fetch + verify per-issuer `dekimu-issuer.json` manifests (advertised family memberships, claim_types, profiles); cross-issuer handshake CLI | OPTIONAL — only verifiers and tools that surface cross-issuer discovery (e.g. `verify.dekimu.com`, audit pipelines). Member specs SHOULD list the family memberships and profiles they expect to advertise in their own `dekimu-issuer.json` |

> **Audit fix H11 + H12 (2026-05-17):** consolidates package-reference rule. Pre-canonical specs (ATR/APuR/AER/AAR/ALR) had no envelope-package mention; closure specs (ANR/ABR/ASR/AIR) had no did-web-resolver mention. This table establishes both as family-wide mandatory imports.
>
> **Cross-issuer interop additions (2026-05-19, plan-approved):** `@dekimuhq/anchors-trusted-issuers` and `@dekimuhq/anchors-federation` added as opt-in shared infrastructure for the trust surface. They do NOT touch wire format. Member specs that integrate `verifyIssuerKey` (mandatory once 0.2.0 lands in Phase A) SHOULD reference the default `TrustedIssuersSource` shipped by `@dekimuhq/anchors-trusted-issuers` unless the deployment overrides it.

---

## 6. Engineering budget table (canonical)

| Family | Status | Tests promised | Engineering budget (1-eng-eq) | Closeable by |
|---|---|---|---|---|
| APR | shipped | 211 | shipped | shipped |
| ACR | shipped | (subset of 579 total) | shipped | shipped |
| ARR | shipped | (subset of 579 total) | shipped | shipped |
| ADR | shipped | 99+ | shipped | — |
| ATR | shipped | 173 (verifier 75 + profiles 77 + issuer-kit 21) | shipped | — |
| APuR | shipped | 170 | shipped | — |
| AER | shipped | 90+ | ~3.5 weeks | shipped |
| AAR | specced | 100+ | ~4 weeks (TSL snapshot mechanism adds complexity) | post-v1.2 envelope + EUTL snapshot data |
| ALR | specced | 99+ | ~2 weeks (smallest — already partially canonical) | post-v1.2 envelope |
| ANR | specced (closure wave 1) | 80+ | ~4 weeks | post-v1.2 envelope |
| ABR | shipped | 373 (verifier 166 + issuer-kit 104 + profiles 103) | shipped | shipped |
| ASR | shipped | 213 (verifier 93 + issuer-kit 3 + profiles 117) | shipped | shipped |
| AIR | shipped | 406 (verifier 192 + issuer-kit 103 + profiles 111) | shipped | shipped |
| ATokR | shipped | 136 (verifier 56 + issuer-kit 20 + profiles 32 + integration 28) | 0.1.0 | — |

**Total unimplemented (10 members):** ~34.5 weeks 1-eng-eq.
**Family-wide test gate:** ≥250 tests beyond current 579 baseline (sum of per-spec promises exceeds gate by ~3×; suggests gate can be raised to ≥800 once all 13 ship).

> **Audit fix H2 (2026-05-17):** pre-canonical specs (ATR/APuR/AER/AAR/ADR) had no budget or test-count promise. Consolidated here. ALR weeks added.

---

## 7. Consistency-checklist for new family members

When proposing a new claim_type (any time post-2026-05-17), the spec MUST update this file's tables before merging:

1. Add chip-position row (§1).
2. Add TSA mandate row(s) per event (§2).
3. Add subject-presence row(s) per event (§3).
4. Add status-list consumer position (§4) — or declare APR-style "not consumed".
5. Add shared-package import declarations (§5).
6. Add engineering budget row (§6).

Spec authors: copy the consistency-checklist into your draft's §3.0 ("family integration") section and fill in. Auditor pre-merge: confirm all six tables updated.

---

## 8. Standalone attestations (non-receipt signed artifacts)

Standalone attestations are signed artifacts that share the canonical `ar.<noun>.v<N>` discriminator family with member receipts but carry **no `claim_type`, no `field_root`, no `subject`, and no v=2 envelope wrapper**. They certify operational state (key continuity, trust scope, issuer self-description) rather than a per-subject event. All standalone attestations:

- Sign the JCS-canonicalised attestation body with `sig: null` (RFC 8785 canonical JSON; ed25519 over the canonical bytes).
- Are owned by an `@dekimuhq/anchors-envelope/<subpath>` export so issuer-kits across the family share one primitive surface.
- Use a `kind` discriminator (or shape-named field) keyed to the `ar.<noun>.v<N>` taxonomy — mutating any field (including the discriminator) invalidates the signature.
- Are registered in [`registry.md`](registry.md) under the "standalone-attestation" subsection (separate from the `claim_type` registry).

Two scopes are distinguished:

| Scope | Meaning | Members |
|---|---|---|
| **Per-family** | One attestation per family member, owned by that family's issuer-kit (pins a `family` slug at issue time). | `ar.continuity.v1` (§8.1) |
| **Family-wide** | One attestation surface shared across all 14 members; not pinned to any one family. Carries no `family` slug. | `ar.trusted_issuers.v1` (§8.2), `ar.issuer_manifest.v1` (§8.3) |

### §8.1 — `ar.continuity.v1` (per-family issuer key-rotation attestation)

Promoted from per-family `<family>.continuity.v1` (ALR / ARR / ANR) to the canonical `ar.<noun>.v<N>` wire form in envelope 0.7.0. Owned by `@dekimuhq/anchors-envelope/continuity` subpath export; each family's issuer-kit re-exports the helpers and pins its own `family` slug at issue time. Spec: `2026-05-18-anchors-continuity-v1-unification.md`.

| Family | Slug (body `family` field) | Continuity flow | Issuer-kit re-export | Notes |
|---|---|---|---|---|
| APR | `apr` | **shipped** | `@dekimuhq/apr-issuer-kit` | Re-exports from envelope/continuity. |
| ACR | `acr` | **shipped** | `@dekimuhq/acr-issuer-kit` | Re-exports from envelope/continuity. |
| ARR | `arr` | **shipped** | `@dekimuhq/arr-issuer-kit ≥ 0.5.0` | Re-exports `issueContinuityAttestation` + `verifyContinuityAttestation` from envelope subpath. |
| ALR | `alr` | **shipped** | `@dekimuhq/alr-issuer-kit ≥ 0.2.0` | Re-exports. |
| ANR | `anr` | **shipped** | `@dekimuhq/anr-issuer-kit ≥ 0.2.0` | Re-exports. |
| ASR | `asr` | **shipped** | `@dekimuhq/asr-issuer-kit` | Re-exports from envelope/continuity. |
| ABR | `abr` | **shipped** | `@dekimuhq/abr-issuer-kit` | Re-exports. |
| AIR | `air` | **shipped** | `@dekimuhq/air-issuer-kit` | Re-exports. |
| ATokR | `atokr` | **shipped** | `@dekimuhq/atokr-issuer-kit` | Re-exports. |
| APuR | `apur` | **shipped** | `@dekimuhq/apur-issuer-kit` | Re-exports from envelope/continuity. |
| ATR | `atr` | **shipped** | `@dekimuhq/atr-issuer-kit` | Re-exports from envelope/continuity. |
| AER | `aer` | **shipped** | `@dekimuhq/aer-issuer-kit` | Re-exports from envelope/continuity. |
| AAR / ADR | per slug | defer to per-family spec | n/a | Default position: ship at family launch. |

**Wire-format notes:**

- The closed `FamilySlug` union (`apr | acr | arr | alr | atr | apur | aer | aar | adr | anr | asr | abr | air | atokr`) is exported from `@dekimuhq/anchors-envelope/continuity` and matches the 14-member taxonomy locked in `anchors/CLAUDE.md`.
- Hard-cut migration applied 2026-05-19 (pre-launch): no legacy `<family>.continuity.v1` parser shipped. The verifier rejects any non-`ar.continuity.v1` `kind` with `kind.invalid`.

### §8.2 — `ar.trusted_issuers.v1` (family-wide signed allowlist manifest)

**Status:** specced for Phase B of the cross-issuer interop plan, `2026-05-19-anchors-cross-issuer-interop.md` §2. Not yet implemented. Owned by `@dekimuhq/anchors-trusted-issuers` (signing helper expected to live at `@dekimuhq/anchors-envelope/trust-manifests` for shape consistency with continuity).

Family-wide. No `family` slug — one signed list governs trust dispatch for every family member's verifier. Published at `verify.dekimu.com/.well-known/dekimu-trusted-issuers.json`; append-only mirror at `dekimuhq/anchors-trusted-issuers-chain`.

| Surface | Value |
|---|---|
| Discriminator | `kind: "ar.trusted_issuers.v1"` |
| Body shape | `{ issuers: TrustedIssuerRecord[], generated_at, sequence, prev_root \| null }` (sequence + prev_root form an append-only hash chain) |
| Signing key | Pinned `BOOTSTRAP_PUBLISHER_KEY` (Dekimu Labs SL family key in v1); 1-year rotation with 90-day overlap (Q5 decision) |
| Cadence | Monthly cron re-sign + republish if no manual update in 30 days (`dekimuhq/anchors-trusted-issuers-chain` action) |
| Consumed by | `verifyIssuerKey` (§5 row 2) via the default `TrustedIssuersSource` implementation |

**Wire-format notes:**

- Standalone attestation per §8 invariants (no envelope; JCS + sig: null; ed25519).
- The `BOOTSTRAP_PUBLISHER_KEY` SPKI is baked into `@dekimuhq/anchors-trusted-issuers` and `@dekimuhq/did-web-resolver`. Rotating it requires a package version bump on every consumer.
- The federation governance Q4 (rotating chair / multi-sig 2-of-3) is deferred. v1 ships unilateral Dekimu signing; the manifest README links the federation roadmap.

### §8.3 — `ar.issuer_manifest.v1` (per-issuer self-published manifest)

**Status:** specced for Phase C of the cross-issuer interop plan §3. Not yet implemented. Owned by `@dekimuhq/anchors-federation` for fetch + verify; signing helper colocated with §8.2 under `@dekimuhq/anchors-envelope/trust-manifests`.

Family-wide. One manifest per issuer (not per family member): each `did:web` issuer publishes ONE `dekimu-issuer.json` at their own `/.well-known/` path advertising which family memberships they emit, which `claim_type`s they support, which profiles they implement, and a pointer to their `dekimu-keys.json`. Consumed by `verify.dekimu.com` and by cross-issuer handshake tooling.

| Surface | Value |
|---|---|
| Discriminator | `kind: "ar.issuer_manifest.v1"` |
| Body shape | `{ iss, families: FamilySlug[], claim_types: string[], profiles: Record<claim_type, profile[]>, keys_url, generated_at, expires_at }` |
| Signing key | The issuer's own family key (the one referenced by `keys_url`). Self-attestation; cross-verified via did:web resolution. |
| Consumed by | `@dekimuhq/anchors-federation`'s `fetchIssuerManifest` + `runHandshake`; `verify.dekimu.com` discovery surface |
| Drift surface | `profile_scope_mismatch` warning when receipts in the wild advertise profiles not declared in this manifest (verifier surfaces; does not auto-reject) |

**Wire-format notes:**

- Standalone attestation per §8 invariants. Same JCS + ed25519 signing primitive as §8.1 / §8.2.
- The `FamilySlug` union is reused verbatim from `@dekimuhq/anchors-envelope/continuity` — single source of truth for the 14-member taxonomy.
- `expires_at` is verifier-side advisory; the trusted-issuers allowlist (§8.2) is the authoritative revocation surface. A manifest past `expires_at` MAY be refreshed; a revoked issuer is never trusted regardless of manifest freshness.

> **Cross-issuer interop addition (2026-05-19, plan-approved):** §8 generalised from "`ar.continuity.v1` — family-shared issuer key-rotation attestation" to "Standalone attestations (non-receipt signed artifacts)" to seat `ar.trusted_issuers.v1` (Phase B) and `ar.issuer_manifest.v1` (Phase C) alongside the existing continuity attestation. No wire-format changes; this is the doc-only prerequisite gate cleared before Phase A of the cross-issuer interop plan touches code.

---

## 9. Standards-vocabulary mapping — PCI/EMVCo tokenization ↔ envelope payload modes

The anchors family's `public_full` vs `committed` payload modes (envelope v1.1 §5.4) map cleanly onto the **High-Value Token (HVT) / Low-Value Token (LVT)** distinction codified in the PCI tokenization literature and EMVCo Payment Tokenisation (March 2014). This section is non-normative — it surfaces the lineage so security reviewers fluent in tokenization vocabulary can locate our design without re-deriving it.

| Anchors term | PCI/EMVCo equivalent | Property |
|---|---|---|
| `public_full` payload | High-Value Token (HVT) | Surrogate carries enough structure to act on without round-tripping to the issuer (e.g. a fully-published consent receipt that downstream systems can verify standalone). |
| `committed` payload (`withheld_fields` + salted SHA-256 commits per v1.1 §5.4) | Low-Value Token (LVT) | Surrogate is proof-only; useless without the issuer's mapping (the salt + plaintext). Verifier confirms a commit matches a known plaintext only when the issuer reveals it. |
| Issuer's HMAC/SHA-256 salt (per-receipt or per-issuer) | Tokenization vault secret | Single irreversibility root. Compromise yields offline bruteforce over the receipt's withheld-field space. |
| `scope` envelope block (audience + purpose + validity) | EMV scope-binding cryptogram (device + merchant + transaction context) | Replay/scope-creep mitigation. Verifiers enforce mismatch as a HARD invalidation. |

**Implication for reviewers:** the family is structurally a stateless, format-flexible tokenization system in the PCI sense — vaultless (no central token table), irreversibility via cryptographic commit rather than mapping-table lookup, with scope-binding enforced verifier-side rather than vault-side. The 14-member family extends the pattern from "PAN ↔ token" to "GDPR/EU-AI-Act/eIDAS event ↔ verifiable receipt."

**Not a new normative rule.** No member spec is required to update behaviour because of this section. The mapping is documentation only — added 2026-05-20 after a literature-review pass mapped the anchors design space onto PCI tokenization vocabulary (see `Dekimu Labs/docs/decisions/2026-05-20-atokr-anchored-tokenization-receipts.md`). **ATokR (14th member, LOCKED 2026-05-21)** is the family member that directly instantiates this vocabulary mapping — its 4 event types (`token.issued/redeemed/revoked/rotated`) are the receipt-level counterparts of the HVT/LVT lifecycle described here.

---

## 10. Countersignature key field (`countersig_key`)

Added in envelope 0.13.0 (2026-05-21). Decision: `Dekimu Labs/docs/decisions/2026-05-21-countersig-key-envelope-delta.md`.

**Cross-family invariants:**

- `countersig_key` is **signed-in**. The issuer commits to the countersigner identity at issuance time. Mutating it post-sign invalidates the issuer's Ed25519 signature.
- `countersig` remains **signed-out**. The countersigner signs the same `preSigCanonical` bytes as the issuer (signed-out fields `anchor`, `sig`, `countersig`, `tsa` zeroed to `null`).
- The field follows the same `ed25519:<base64url>` prefix convention as `iss_key`, providing algorithm agility for future post-quantum (ML-DSA) migration.
- Optional on every family member — absent on all pre-0.13.0 receipts. No family member mandates countersignatures in v1.

| State | `countersig_key` | `countersig` | Verifier outcome |
|---|---|---|---|
| No countersig expected | absent | absent | — |
| Legacy/pre-delta receipt | absent | present | Format check only (64-byte Ed25519 length) |
| Awaiting countersig | present | absent | "Awaiting countersignature from identified key" |
| Full verification | present | present | Ed25519 cryptographic verification of countersig against `countersig_key` |

**Planned consumer:** EUDI Wallet integration (subject countersigns consent/notice receipts). No family member ships EUDI support in v1; the field is the forward-compatible prerequisite.

---

**This document is normative for cross-family invariants. When a member spec contradicts this file, this file wins until the spec is updated to match.**
