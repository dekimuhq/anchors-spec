# RFC-003: Input provenance for `ar.action.v1` (`input_provenance`)

**Authors:** Dekimu
**Status:** Draft (PROPOSAL — no wire edit; nothing in `action/v1.md`, the reference implementation or the registry changes until Stewards accept). Revised 2026-11-27 after an adversarial review (see §Review log).
**Created:** 2026-11-27
**Target version:** n/a (profile/family) — additive optional field on `ar.action.v1`, or a sibling receipt (see §Options)

---

## Motivation

An AActR records that an agent *acted* (verb, rule, outcome) and, optionally, under which mandate (`credential_id`, `caveats_consumed`, `delegation_chain`, `agent_id`). It says nothing about **what third-party content the action was derived from**. When a harness enforces an information-flow policy at its own outbound-call boundary (for example `@dekimuhq/agent-taint`: values from email bodies, web pages, uploads or third-party tool results carry an `untrusted` label, and a guard refuses labelled values at privileged arguments named in the policy), the receipt cannot say so. A reader cannot tell whether labelled untrusted content fed the action, or whether a human or policy **declassified** a labelled value to let it through, and where that approval is recorded.

Mandate fields (§4.4–4.6) answer *"who may act and how far"*. This RFC adds the data-flow side: *"which labelled untrusted content fed this action, and who let it through"*.

### Non-goals and the honesty boundary

- It does **not** claim the action was safe, correct, or free of prompt injection.
- **No label ≠ no untrusted influence.** Labels are assigned by harness code at the tool-result boundary. In an ordinary agent loop the model reads the email and then writes the tool arguments itself; those arguments carry no label (agent-taint: "data that is never wrapped is treated as plain", "copies text out of a model reply, the label is gone"). A receipt for an injected action can therefore look label-free. For that reason **this RFC defines no "clean" state** (rule R1).
- It does **not** prove a guard ran. Like `credential_id`, `delegation_chain` and `agent_id`, the field is **issuer-asserted descriptive metadata**; a stateless verifier checks internal consistency only.
- It does **not** standardise a guard, a policy language or a label lattice.

## Proposed Changes

### Wire format

One new OPTIONAL object in `AActRBody`, key absent (never `null`/`undefined`) when not populated:

```typescript
type InputProvenance = {
  /** Sorted, de-duplicated source classes of the labelled untrusted data that fed the action,
   *  taken from (a) labels on the dispatched arguments and (b) labels in the declassification
   *  records consumed by the call. Non-empty. Each class matches ^[a-z][a-z0-9_]{0,31}$.
   *  Suggested registry (registry.md): email, web, upload, tool_result. Unknown classes are
   *  valid; renderers show them as "other". The issuer maps its own label sources to classes;
   *  that mapping is issuer policy, not specified here. Plaintext, non-PII. */
  untrusted_classes: string[];

  /** Salted commit over canonicalize(sorted list of source identifiers), e.g. inbox message
   *  ids or URLs. Identifiers can be personal data, so they are committed, never plaintext.
   *  The salt lives in the issuer's owner-revealable reveal map. */
  sources_commit: string;

  /** Labelled values that were explicitly declassified to reach a privileged argument.
   *  Empty array when none. Each entry binds the approval, not the value. */
  declassifications: Array<{
    /** Privileged-argument path copied VERBATIM from the policy spec string (never the walked
     *  data path, which can contain keys that are personal data). Grammar:
     *  ^[A-Za-z_][A-Za-z0-9_]*(\[[0-9]+\]|\.[A-Za-z_][A-Za-z0-9_]*)*$, max 128 chars. */
    path: string;
    /** sha256:<hex> salted commit over canonicalize(declassification record), the
     *  `declassify().record` shape (labels, approver, reason, value hash). */
    record_commit: string;
    /** OPTIONAL sha256:<hex> pointing at the approving receipt: an ACR `aiact.art14` receipt
     *  (same shape and target as `aiact_art12.oversight_ref`). */
    approval_ref?: string;
  }>;

  /** OPTIONAL. Identifies the flow policy that was evaluated, so a reader sees which arguments
   *  were protected. See §Policy identification. */
  flow_policy?: {
    version: 1;
    /** sha256:<hex> of canonicalize(normalized policy). */
    hash: string;
    /** https URI where the issuer publishes the normalized policy. REQUIRED when `flow_policy` is present. */
    uri: string;
  };
};
```

The policy member is named `flow_policy` (not `policy_hash`) because the envelope already carries `policy_hash` meaning the recipe-catalog hash (§2.1).

### Rules (base verifier, `verifyActionBody`)

The rules go in the base body verifier, not in a profile validator, because the implementation has two profiles (`hub.automation`, `aiact.art12`) and a profile-only rule would skip one. Error codes follow the base style.

| # | Rule | Error code |
|---|---|---|
| R1 | **The object is present only when labelled untrusted data was involved.** `untrusted_classes` MUST be non-empty. There is no clean/`false` state: absence means "no labelled untrusted data was recorded", **never** "no untrusted input". | `input_provenance.empty` |
| R2 | every class matches the class pattern; array sorted and unique | `input_provenance.classes_invalid` |
| R3 | `sources_commit`, every `record_commit`, `flow_policy.hash` and any `approval_ref` match `sha256:<64hex>` | `input_provenance.hash_malformed`, `approval_ref.invalid` |
| R4 | each `path` matches the path grammar | `input_provenance.path_invalid` |
| R5 | `flow_policy`, when present, has `version === 1`, a valid hash and an `https:` `uri` | `input_provenance.flow_policy_invalid` |
| R6 | object is a plain object with only known members | `input_provenance.invalid` |

`input_provenance` does **not** require `credential_id` (policy-authorised automation rules have no mandate yet still consume untrusted email text). AActR's `field_root` is a flat RFC 6962 leaf hash over the canonical whole body (`computeFieldRoot`), so the member is covered automatically.

**Salts.** Every commit in this object (`sources_commit`, each `record_commit`) MUST use a fresh ≥128-bit salt generated per commit per receipt. A reused salt would let `sources_commit` link receipts by sender.

### Policy identification (`flow_policy`)

Reproducibility requires a normal form. The normalized policy is a JSON object `{ "verbs": { "<verb>": { "privileged": [<path>...], "allow_untrusted": <bool> } } }` with verbs sorted, paths sorted, each path in grammar form with `.1`-style segments rewritten to `[1]`, and `"*"` written as the single path `"*"`. The hash is over RFC 8785 canonical JSON. The issuer MUST publish the normalized policy at `uri`; a hash with no published policy is not allowed (an unsalted hash of a small policy is guessable and only links receipts). `@dekimuhq/agent-taint` would need to export a `normalizePolicy` to produce this; that is an implementation follow-up, not part of this RFC.

### Verification

Single-receipt verification gains the rules above. A stateless verifier cannot confirm that the policy at `flow_policy.uri` was the one enforced, that labelling covered every input channel, or that `approval_ref` exists. `valid: true` remains "well-formed and internally consistent", never "authorised" or "safe". Verifier UIs (including `verify.dekimu.com`) MUST render the two states as: **absent** = "input provenance not recorded"; **present** = "labelled untrusted input recorded: <classes>". They MUST NOT render absence as a clean indicator.

### Issuer

`untrusted_classes` and `sources_commit` are computed from the labels on the dispatched arguments **plus the labels in consumed declassify records** (a declassified value no longer carries a label, so reading the arguments alone would miss it and produce a declassification with no taint). This needs a label-collection helper that the current agent-taint guard does not export: its `GuardResult` on `ok:true` carries no labels. Follow-up in agent-taint: export `collectLabels(args)` and `normalizePolicy(policy)`.

## Options considered for where this lives

1. **Additive optional field on `ar.action.v1` (this draft).** Precedent: `delegation_chain` and `agent_id` were added after the freeze under §7. The field sits next to the effect it describes, so a verifier of the action sees it without fetching a second receipt. Cost: amends a frozen public spec; the honesty boundary lives in a new §4.8 rather than a family of its own.
2. **Sibling receipt (AGR `guard.flow` profile or a new family) chained to the action with `prev`.** AGR already says `prev` MAY chain a screening verdict to a downstream action receipt (`guard/v1.md` §2.1), and AScR was built as a new family "so that its honesty boundary is stated in its own text and no frozen public spec is amended" (`screening/v1.md` §1.1). Cost: two receipts per action, and the link is bookkeeping only.
3. **`ar.trace.v2` span attributes.** Rejected for the summary: trace is per-run provenance and DISTINCT from AActR by design; good home for per-span detail only.
4. **Extend `caveats_consumed`.** Rejected: caveats bound authority and are rejected without `credential_id`; data provenance is not authority and applies to mandate-less automation.

Draft position: Option 1, because the convenience of one receipt matters for the verifier UI and §4.6 sets the precedent. **This is a Steward call**; Option 2 is the better fit if the Stewards want the AScR no-amendment precedent to hold.

## Backward Compatibility

Additive and OPTIONAL under `action/v1.md` §7. Receipts minted before this change verify byte-equal. Old verifiers see an unknown body member. Checked 2026-11-27: `verifyActionBody` (`anchors-oss/action/action/src/verify-single.ts`) and the `hub.automation` validator check named members only and do not enumerate keys, so unknown members are ignored. **Not checked:** the `aiact.art12` profile validator and third-party verifiers. Because classes are an open pattern (not a closed enum), later registry additions do not break strict validators.

**Prerequisite spec drift.** The code has two body members that `action/v1.md` does not document (`aiact_art12`, `non_revocation_proof`) and a second profile (`aiact.art12`), while §5 says "One profile". `action/v1.md` §3 also calls `field_root` a "selective-disclosure root", while the code computes a whole-body leaf hash. These should be reconciled in a separate editorial PR before or with this change.

No change to the envelope, to `subject: null`, to `credentialStatus` (non-consumer), to `tsa`, or to the leaf identity.

## Security Considerations

- **False assurance** is the main risk and is addressed structurally by R1 (no clean state) and the rendering rule. Spec and marketing text must say "records which labelled untrusted inputs fed an action and which were declassified"; never "proves no injection", "secure", or "compliant".
- **Issuer-asserted.** An issuer can lie, mislabel, or fail to label. Same trust model as the mandate fields.
- **Privacy.** Source identifiers are committed with a fresh salt. `untrusted_classes` is a low-cardinality token; `path` is a policy string constrained by grammar (never a walked data path, no `@`, no whitespace). `record_commit` covers a record that contains a hash of the declassified value, guessable for short values, hence a salted commit.
- **Fingerprinting.** `flow_policy.hash` is stable across receipts for one issuer and links receipts to a policy version; accepted because the policy is published.
- **Declassification is the weak point by design.** The receipt shows that a human or policy let a labelled value through; it does not say that was wise.

## Test Vectors

New directory `conformance/action/` (none exists for action today). valid: pre-change receipt without the member (byte-equal); one class; declassification with `approval_ref`; without `credential_id`; with and without `flow_policy`. invalid: one per error code above, plus an empty `untrusted_classes`, a `null` value, an unknown member, and a path containing `@`. `field_root` vector: a body carrying the member hashes into `field_root`, and dropping it changes the root.

## Implementation Notes

Rollout if accepted: (1) editorial drift fix + spec §4.8 + vectors; (2) `action-profiles` body type and `verifyActionBody` rules; (3) `mintActionV2` optional input; (4) agent-taint `collectLabels` / `normalizePolicy`; (5) Hub dispatch passes the result through; (6) `verify.dekimu.com` rendering. Only (1)–(3) are public-spec work.

## Review log

Adversarial skeptic review, 2026-11-27 (fresh context): 1 blocker, 6 serious, 1 minor, verdict revise-then-draft. All eight addressed in this revision: clean state dropped (R1, honesty boundary); issuer label-collection gap named (§Issuer); AGR/AScR precedent added (§Options); rules moved to the base verifier with base-style codes; closed enum and `tracked` removed; `policy_hash` renamed, normalized, published; `path` grammar and per-commit salt freshness added; `approval_ref` aligned with `oversight_ref` and given an error code. Family-consistency check performed by hand against `family-consistency.md` (null-subject, Status-List non-consumer, TSA optional, `field_root`, additive-after-freeze precedent, ADR/AgTR/AGR demarcation): no row is changed by Option 1; the AGR/AScR precedent is the open Steward question. The automated `anchors-invariant-check` workflow was not run (no code changes to check).

## References

- `action/v1.md` §4.4–4.6, §6, §7; `family-consistency.md` rows AActR (15), AGR (17), AgTR (19), ADR (4); `guard/v1.md` §2.1; `screening/v1.md` §1.1; `action/aiact-art12-mapping.md` (`oversight_ref`)
- `@dekimuhq/agent-taint` (dekimu-packages#406): label, guard and declassify-record semantics
- Debenedetti et al., *Defeating Prompt Injections by Design* (CaMeL), 2025 — inspiration for the guard, not implemented by it
- Internal idea record: `docs/ideas/2026-11-26-agent-authority-taint-egress.md`
