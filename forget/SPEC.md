# Proof of Forgetting — `forget.*` receipts (v1)

**Status:** **v1 — wire shapes FROZEN** as of 2026-10-19 (the day this specification went public). New fields are additive and OPTIONAL; a breaking change needs a `.v2` kind.
**Spec licence:** CC0 1.0 Universal (public domain).
**Reference implementation:** [`dekimuhq/forget`](https://github.com/dekimuhq/forget) (Apache-2.0). Everything an independent verifier needs is in this document plus Ed25519, SHA-256, and an RFC 8785 canonicaliser. The reference implementation was used to generate and verify every example in [`examples/`](examples/).
**Brand:** "Dekimu Proof of Forgetting" is the brand mark. The wire kinds, the phrase "Proof of Forgetting", and "forget-receipt-compatible" are unclaimed and free for downstream use.
**Wire kinds:** `forget.roster.v1` · `forget.destruction.v1` · `forget.model-disposition.v1`.

> **This is not an `ar.<noun>.v<N>` Anchored Receipts family.** `forget.*` receipts are *companion receipts*: standalone Ed25519-signed JSON objects with their own `kind` discriminator, no `claim_type`, no v=2 envelope, no chip, no entry in the envelope `claim_type` union, and no Merkle anchoring into the daily family log. They are verified **by value** (the holder presents the receipt and the verifier recomputes everything) rather than by reference. The relationship to the family — in particular to `ar.erasure.v1` and `ar.subject_rights.v1` — is in §8 and in [`CROSS-REFERENCES.md`](../CROSS-REFERENCES.md).

---

## 1. What Proof of Forgetting is

GDPR Article 17 gives every person a right to erasure. Everyone claims deletion; almost nobody can evidence it. Proof of Forgetting makes an erasure **checkable** by pairing two signed claims that an auditor can verify offline, against a trusted issuer key, without trusting the issuer's database or its delete query:

| Half | Receipt | What it says |
|---|---|---|
| **Positive evidence** | `forget.destruction.v1` | "The data-encryption key for subject *S* was destroyed at time *T* under key authority *K*, moving the roster from signed root *R₀* (which contained *S*) to signed root *R₁* (which does not)." |
| **Negative evidence** | `forget.roster.v1` + a non-inclusion proof | "*S* is absent from the published, signed roster *R₁* of the subjects the issuer still holds — and *R₁* chains to *R₀*, in which *S* was present." |

Neither half is honest on its own. A receipt alone only records that the issuer *said* it deleted; an absence proof alone carries no account of when, or by whose authority. The destruction receipt commits to **both** roster roots, which is the join that turns two claims into one checkable story.

A third, optional receipt — `forget.model-disposition.v1` — records how the subject's data was handled in the **model dimension** (never trained on · retrained without it · machine-unlearned · excluded from training by a documented filter), bound to the destruction receipt so it cannot be re-pointed at a different subject's erasure.

### 1.1 The honest claim (load-bearing)

A valid bundle evidences, **against the issuer's signing key**:

- an erasure event occurred at a specific time, under a named key authority (signed destruction receipt);
- the subject is absent from the signed roster the receipt names as "after" (non-inclusion proof against that root);
- the subject was present in the signed roster the receipt names as "before" (membership proof against that root);
- the two rosters chain (`prevRoot`), and each one's leaf count is signed in, so a roster cannot be silently truncated or rolled back without the signature failing.

It does **not** evidence:

- that every ciphertext copy was physically located and deleted — crypto-shred (§1.2) makes copies unreconstructable, it does not hunt them down;
- that data the issuer never held, or that left through a third party, was erased — the roster covers what the issuer manages;
- more than the issuer's retention discipline allows: the construction is only as strong as the wrapped key being the single copy, held under a short or no backup window (§7.3);
- that the signer is the authority you expect. Signature validity is not trust; the key must be resolved out of band (§6.1, §7.1).

The receipts are **tamper-evident by construction** (every signed field is recomputed at verification; any change invalidates the signature) and **independently verifiable** (no call to the issuer is needed). The format is **designed to support** a GDPR Art. 17 erasure record; whether a given receipt satisfies a particular legal obligation is a question for the relevant authority. This document is not legal advice.

### 1.2 Erasure mechanism (context, out of scope for the wire format)

The reference issuer erases by **key destruction, not by a delete query**: each subject's personal data is encrypted under its own random 256-bit key (AES-256-GCM); only the small wrapped data-encryption key sits in a hardened, retention-tight registry, itself wrapped by a workspace key-encryption key (KEK) that is never destroyed and never appears in a receipt (only its id does). Erasing a subject deletes that one registry row; the ciphertext — wherever copies live — becomes permanently unreconstructable. A conforming **verifier** needs none of this; it needs only §2–§6. The mechanism is described so the receipts' claims are read at the right strength.

### 1.3 Demarcation from the Anchored Receipts family

| Artifact | Answers |
|---|---|
| `ar.subject_rights.v1` (ASR) | "A data-subject *request* was received / fulfilled / refused — the Art. 15–21 lifecycle." |
| `ar.erasure.v1` (ARR) | "An erasure *event* was logged, anchored into the daily Merkle log, chained per target." |
| **`forget.destruction.v1` + `forget.roster.v1`** | **"The subject's key was destroyed, and the subject is shown absent from the issuer's current signed roster — checkable offline against two signed roots."** |
| **`forget.model-disposition.v1`** | **"And this is how the subject's data was handled in any model trained on it."** |

An anchored `ar.erasure.v1` receipt says an erasure was *logged*; a `forget.*` bundle lets a third party check the *state* the issuer now publishes. They compose (§8) and neither replaces the other.

---

## 2. Cryptographic primitives and encoding

| Primitive | Choice |
|---|---|
| Hash | SHA-256 |
| Signature | Ed25519 (RFC 8032); raw 32-byte public keys, 64-byte signatures |
| Signature encoding | base64url **without** padding (RFC 4648 §5) |
| Canonical JSON | RFC 8785 (JCS) — see §2.1 |
| Digests and roots in JSON | lowercase hex, exactly 64 characters |
| Timestamps | ISO 8601 / RFC 3339 strings, UTC recommended; **not** independently authenticated (§7.6) |

### 2.1 Canonicalisation

The byte string that is signed is `UTF-8( canonicalize( signedView ) )`, where `canonicalize` is RFC 8785 over the receipt's value space: object keys sorted by UTF-16 code unit order, no insignificant whitespace, standard JSON string escaping, finite numbers serialised as JSON. Two rules matter for interoperability:

1. **Keys whose value is `undefined` are omitted** from the canonical form. This is how the optional `trainingRootBefore` / `trainingRootAfter` fields of `forget.model-disposition.v1` drop out consistently at both signing and verification. A field that is *present with `null`* (`prevRoot: null` on a chain-head roster) is serialised as `null` and **is** signed.
2. The only numeric field in any `forget.*` signed view is `leafCount` (a non-negative integer). Non-finite numbers are a canonicalisation error.

### 2.2 Domain-separated hashing

All three are SHA-256; `bytes(x)` is the raw 32-byte decoding of a 64-character lowercase-hex string.

```
subjectValue(id)      = SHA-256( 0x02 ‖ UTF-8(id) )                     // a subject's leaf value
intervalLeaf(lo, hi)  = SHA-256( 0x00 ‖ bytes(lo) ‖ bytes(hi) )          // a sorted-interval leaf
node(left, right)     = SHA-256( 0x01 ‖ bytes(left) ‖ bytes(right) )     // an internal Merkle node
```

**Fixed-length operand rule (normative).** `intervalLeaf` and `node` MUST reject any operand that is not exactly 64 lowercase hex characters. Variable-length operands would make `H(a ‖ b)` ambiguous — `intervalLeaf("aabb…", "ccdd…")` could equal `intervalLeaf("aa…", "bbccdd…")` — and an attacker could re-split a genuine leaf's preimage into a different `(lo, hi)` pair that still hashes to a leaf in the tree. The reference implementation throws on a malformed operand and its verifiers catch that throw and return `false` (fail closed, never throw to the caller). The prefix bytes `0x00` / `0x01` / `0x02` prevent leaf ↔ node ↔ value collisions.

`subjectValue(id)` is the value carried as `subjectKeyIdHash` in every receipt. `id` is the issuer's **subject key id** — an opaque identifier of the subject's key-registry row, never the person's name or email (§7.5).

---

## 3. `forget.roster.v1` — signed roster root

### 3.1 Construction (interval-leaf range Merkle tree)

A roster is the set of subjects the issuer currently holds, represented as a Merkle tree whose leaves are **intervals** over the sorted subject values, bounded by two sentinels:

```
MIN      = "00" × 32            MAX = "ff" × 32
values   = sort( dedupe( subjectValue(id) for each held subject ) )     // ascending, strict hex order
boundary = [ MIN, v₀, v₁, …, vₙ₋₁, MAX ]
leafᵢ    = intervalLeaf( boundary[i], boundary[i+1] )                   // i = 0 … n
leafCount = n + 1                                                        // = subjectCount + 1
root     = merkleRoot( [ leaf₀ … leafₙ ] )
```

`merkleRoot` pairs nodes left to right at each level with `node(left, right)`; an unpaired last node at any level **carries up unchanged** (RFC 6962 style — never duplicated, which removes duplicate-leaf malleability). A roster always has at least one leaf: the empty roster is the single interval `(MIN, MAX)`.

Making each leaf an interval is what makes non-inclusion a single ordinary inclusion proof: adjacency is intrinsic to the leaf content, so the verifier never has to trust a leaf index.

### 3.2 Signed view and fields

```
signedView = { kind: "forget.roster.v1", alg: "Ed25519", root, leafCount, generatedAt, prevRoot }
signature  = base64url( Ed25519-sign( UTF-8( canonicalize( signedView ) ) ) )
```

| Field | Type | Signed | Description |
|---|---|---|---|
| `kind` | `"forget.roster.v1"` | yes | Discriminator. |
| `alg` | `"Ed25519"` | yes | Signature algorithm. v1 admits only this value. |
| `root` | hex64 | yes | Merkle root per §3.1. |
| `leafCount` | integer ≥ 1 | yes | Number of interval leaves (`subjectCount + 1`). Signed in so a verifier cannot be shown a truncated tree (§7.4). |
| `generatedAt` | ISO 8601 | yes | When the roster was built. |
| `prevRoot` | hex64 \| `null` | yes | Root of the roster this one supersedes; `null` for a chain head. Signed in so roster history is tamper-evident and a rolled-back roster is detectable (§7.4). |
| `keyId` | string | no | Identifier of the Ed25519 key that signed this roster (resolved by the verifier's trust anchor, §6.1). |
| `signature` | base64url | no | Signature over the canonical signed view. |

Note the key field is **`keyId`** on a roster and **`signerKeyId`** on the two receipt kinds (§11, changelog). Both resolve through the same trust anchor.

Worked canonical string for [`examples/roster-root.json`](examples/roster-root.json):

```
{"alg":"Ed25519","generatedAt":"2026-06-06T01:00:00Z","kind":"forget.roster.v1","leafCount":3,"prevRoot":"ff9ee517f7213e2d34a6898ff7a50f531740c2bc518169821d224edeea8644f4","root":"e8d0dd9a547903a9d9912151f5a450b7293d58b35ec94c58e58dcc2d046e2b61"}
```

### 3.3 Proofs

An **inclusion proof** is the path from a leaf to the root:

```jsonc
{ "siblings": [ { "hash": "<hex64>", "side": "L" | "R" }, … ] }   // ordered leaf → root
```

`side` is the **sibling's** position: `"L"` means `acc = node(sibling, acc)`, `"R"` means `acc = node(acc, sibling)`. A level at which the proven node was carried up unchanged contributes **no** entry.

A **membership proof** for subject value `v` (present) and a **non-inclusion proof** for value `t` (absent) share one shape — an interval plus the inclusion proof of its leaf:

```jsonc
{ "lo": "<hex64>", "hi": "<hex64>", "inclusion": { "siblings": [ … ] } }
```

- Membership: `lo == v` (the interval that *starts* at the subject's value).
- Non-inclusion: `lo < t < hi` (strict hex comparison; the interval that *brackets* the value). Boundary cases need no special handling: a value below the smallest real value lands in `(MIN, v₀)`, above the largest in `(vₙ₋₁, MAX)`, and anything at all in the empty roster's `(MIN, MAX)`.

### 3.4 Verification algorithms

**`verifyRoot(signedRoot, resolveKey) → bool`**

1. `kind == "forget.roster.v1"` and `alg == "Ed25519"`, else `false`.
2. Rebuild `signedView` from the typed fields (§3.2) and canonicalise it. Never trust a stored payload string.
3. `pub = resolveKey(keyId)` — the trust anchor (§6.1). If it throws (untrusted key), return `false`.
4. Return `Ed25519-verify(pub, canonical bytes, base64url-decode(signature))`.

**`verifyInclusion(leafHash, proof, root) → bool`** — fold the siblings leaf → root as in §3.3; return `acc == root`. Any malformed sibling (not hex64) → `false`.

**`verifyMembership(v, proof, root) → bool`**

1. `proof.lo == v`, else `false`.
2. Return `verifyInclusion( intervalLeaf(proof.lo, proof.hi), proof.inclusion, root )`.

**`verifyNonInclusion(t, proof, root, leafCount) → bool`**

1. `leafCount` is an integer ≥ 1, else `false`.
2. `proof.lo < t` and `t < proof.hi` (strict hex compare), else `false` — the interval must genuinely bracket the value. Reusing an absent value's interval to "prove" a present value absent fails here.
3. Return `verifyInclusion( intervalLeaf(proof.lo, proof.hi), proof.inclusion, root )`.

**Signed wrappers (RECOMMENDED entry points).** `verifyNonInclusionSigned(t, proof, signedRoot, resolveKey)` = `verifyRoot(signedRoot, resolveKey) && verifyNonInclusion(t, proof, signedRoot.root, signedRoot.leafCount)`; `verifyMembershipSigned` likewise. They authenticate the root **before** evaluating the proof against it, so they cannot be misused by skipping the signature check. A bare-root check is only meaningful when the root was already authenticated by a signed root or by a signed destruction receipt that commits to it.

**On `leafCount`.** The bare non-inclusion check validates only that `leafCount` is a positive integer; the protection against a truncated tree comes from `leafCount` being **inside the signed view** — a prover who shows a smaller tree must present a root whose signed `leafCount` disagrees, and the signature fails. Verifiers that rebuild a full roster (e.g. an auditor with the subject list) SHOULD additionally check `leafCount == subjectCount + 1`.

---

## 4. `forget.destruction.v1` — destruction receipt

### 4.1 Signed view and fields

```
signedView = { kind: "forget.destruction.v1", alg: "Ed25519",
               subjectKeyIdHash, kekKeyId, destroyedAt, priorRosterRoot, newRosterRoot }
signature  = base64url( Ed25519-sign( UTF-8( canonicalize( signedView ) ) ) )
```

| Field | Type | Signed | Description |
|---|---|---|---|
| `kind` | `"forget.destruction.v1"` | yes | Discriminator. |
| `alg` | `"Ed25519"` | yes | v1 admits only this value. |
| `subjectKeyIdHash` | hex64 | yes | `subjectValue(subjectKeyId)` — the subject's roster leaf value. Never a raw identifier (§7.5). |
| `kekKeyId` | string | yes | Identifier of the key-encryption key under whose authority the subject's data-encryption key was destroyed. **Never** the destroyed key itself, and never the signing key. |
| `destroyedAt` | ISO 8601 | yes | When the key was destroyed. |
| `priorRosterRoot` | hex64 | yes | Root of the signed roster that still contained the subject. |
| `newRosterRoot` | hex64 | yes | Root of the signed roster that no longer contains the subject. |
| `signerKeyId` | string | no | Identifier of the Ed25519 key that signed the receipt (resolved via §6.1). Distinct from `kekKeyId`. |
| `signature` | base64url | no | Signature over the canonical signed view. |

Committing to **both** roots binds the positive claim (a destruction happened) to the negative claim (the subject moved from present to absent between two specific signed rosters). A receipt therefore cannot be replayed against a different roster, and swapping the two roots invalidates the signature.

**Claim id.** A destruction receipt is identified as `"forget:" + subjectKeyIdHash + ":" + newRosterRoot`. This string is what a `forget.model-disposition.v1` receipt binds to (§5).

Worked canonical string for [`examples/destruction-receipt.json`](examples/destruction-receipt.json):

```
{"alg":"Ed25519","destroyedAt":"2026-06-06T01:00:00Z","kekKeyId":"kek-prod","kind":"forget.destruction.v1","newRosterRoot":"e8d0dd9a547903a9d9912151f5a450b7293d58b35ec94c58e58dcc2d046e2b61","priorRosterRoot":"ff9ee517f7213e2d34a6898ff7a50f531740c2bc518169821d224edeea8644f4","subjectKeyIdHash":"99839d0fd18093817a625092e811ef9197f85c6f2bd4dc8f471d654925271795"}
```

### 4.2 The bundle a holder receives

A holder who wants to verify offline needs the receipt plus the two signed rosters and the two proofs. This specification fixes the member shapes; the **container** is informative (the reference implementation stores the members separately and the public verifier accepts a bare receipt). [`examples/destruction-bundle.json`](examples/destruction-bundle.json) uses:

```jsonc
{
  "receipt":     { /* forget.destruction.v1 */ },
  "priorRoster": { /* forget.roster.v1 — root == receipt.priorRosterRoot */ },
  "newRoster":   { /* forget.roster.v1 — root == receipt.newRosterRoot, prevRoot == priorRoster.root */ },
  "absence":     { /* non-inclusion proof of receipt.subjectKeyIdHash against newRoster.root */ },
  "presence":    { /* membership proof of receipt.subjectKeyIdHash against priorRoster.root */ },
  "signerPublicKeys": { "<keyId>": "<hex64 raw Ed25519 public key>" }   // CONVENIENCE ONLY — never a trust anchor (§7.1)
}
```

---

## 5. `forget.model-disposition.v1` — model-dimension disposition

Record deletion can be insufficient where personal data was used to train a model: the model may embed it, and honouring Art. 17 may call for retraining or machine unlearning. This receipt lets a controller state, signed and bound to the destruction receipt, how the subject's data was handled in the model dimension. It ships the **format and verifier only** — it never runs any training or unlearning.

### 5.1 Dispositions

| `disposition` | Meaning | Merkle evidence |
|---|---|---|
| `not_trained` | The data never entered any training or fine-tuning pipeline. | **Forbidden** — no training roots. |
| `scoped_exclusion` | The data was held but excluded from training by a documented filter. | **Forbidden** — no training roots. |
| `retrained` | The model was retrained from a set with the subject removed. | **Required** — `trainingRootBefore` + `trainingRootAfter` signed in; non-inclusion evidence at verification. |
| `unlearned` | Machine unlearning was applied (approximate removal). | **Required** — same as `retrained`. |

For `not_trained` / `scoped_exclusion` there is nothing to prove in a tree: the claim rests on the signed `rationale` and the signer's trust anchor. The **shape invariant** — evidence roots present *iff* the disposition is `retrained` or `unlearned` — is enforced at minting and again at verification.

### 5.2 Signed view and fields

```
signedView = { kind: "forget.model-disposition.v1", alg: "Ed25519",
               subjectKeyIdHash, destructionClaimId, disposition, rationale,
               trainingRootBefore?, trainingRootAfter?, disposedAt }
```

| Field | Type | Signed | Description |
|---|---|---|---|
| `kind` | `"forget.model-disposition.v1"` | yes | Discriminator. |
| `alg` | `"Ed25519"` | yes | v1 admits only this value. |
| `subjectKeyIdHash` | hex64 | yes | Same value as in the bound destruction receipt. |
| `destructionClaimId` | string | yes | `"forget:" + subjectKeyIdHash + ":" + newRosterRoot` of the destruction receipt this disposition belongs to (§4.1). Minting rejects a value that does not start with `forget:`. |
| `disposition` | enum (§5.1) | yes | Closed set in v1. |
| `rationale` | string | yes | Free text, 1–2000 characters after normalisation (§5.3). |
| `trainingRootBefore` | hex64 | yes, when present | Signed roster root of the training set that **included** the subject. Present only for `retrained` / `unlearned`. |
| `trainingRootAfter` | hex64 | yes, when present | Signed roster root of the training set **after** removal. Present only for `retrained` / `unlearned`. |
| `disposedAt` | ISO 8601 | yes | When the disposition took effect. |
| `signerKeyId` | string | no | Signing key identifier (§6.1). |
| `signature` | base64url | no | Signature over the canonical signed view. |

Absent optional roots are **absent keys** (omitted by canonicalisation, §2.1) — never `null`.

### 5.3 `rationale` text rules

Because `rationale` is free text that gets signed and later displayed, it is normalised and guarded at minting: Unicode NFC normalisation; non-empty; at most 2000 characters; and **rejection** of C0/C1 control characters (except tab, LF, CR), DEL, zero-width characters (U+200B–U+200F), bidirectional embeddings/overrides (U+202A–U+202E), bidirectional isolates (U+2066–U+2069), and U+FEFF. This blocks invisible-text and bidi-injection payloads from being signed into a receipt. Verifiers MAY re-apply the same filter and reject a receipt that fails it.

### 5.4 Evidence object (`retrained` / `unlearned` only; not signed)

```jsonc
{
  "trainingAfterSignedRoot":  { /* forget.roster.v1 with root == receipt.trainingRootAfter */ },
  "nonInclusionAfter":        { /* non-inclusion proof of subjectKeyIdHash against it */ },
  "trainingBeforeSignedRoot": { /* OPTIONAL: forget.roster.v1 with root == receipt.trainingRootBefore */ },
  "membershipBefore":         { /* OPTIONAL: membership proof of subjectKeyIdHash against it */ }
}
```

The training-set rosters use exactly the §3 construction — one proof system covers "absent from the active set" and "absent from the training set".

---

## 6. Verification

### 6.1 The trust anchor

Every verification function takes `resolveKey(keyId) → Ed25519 public key`. **This function is the trust anchor.** It MUST map a key id to a public key only for keys the verifier trusts (a pinned key, a trusted-issuer registry, a transparency log) and MUST throw or fail for any other id. A `resolveKey` that trusts whatever key the receipt or bundle supplies defeats the construction entirely (§7.1). All verifiers fail closed: an untrusted key, a malformed field, or an internal error yields `false`, never a throw to the caller.

### 6.2 Destruction receipt alone

**`verifyDestructionReceipt(receipt, resolveKey) → bool`**

1. `kind == "forget.destruction.v1"` and `alg == "Ed25519"`, else `false`.
2. Rebuild the signed view from the typed fields (§4.1), canonicalise, UTF-8 encode.
3. `pub = resolveKey(signerKeyId)`; throw → `false`.
4. Return `Ed25519-verify(pub, bytes, base64url-decode(signature))`.

This establishes that the issuer signed *this* claim. It does not, by itself, establish absence — that needs the roster half (§6.3).

### 6.3 Full bundle — all six MUST pass

Given the bundle (§4.2) and `resolveKey`:

| # | Check | On failure |
|---|---|---|
| 1 | `verifyDestructionReceipt(receipt, resolveKey)` | bad or untrusted receipt signature |
| 2 | `verifyRoot(priorRoster, resolveKey)` **and** `verifyRoot(newRoster, resolveKey)` | bad or untrusted roster signature |
| 3 | `receipt.priorRosterRoot == priorRoster.root` **and** `receipt.newRosterRoot == newRoster.root` | roster mismatch — the receipt commits to different rosters |
| 4 | `newRoster.prevRoot == priorRoster.root` | rosters do not chain — rollback or substitution |
| 5 | `verifyNonInclusion(receipt.subjectKeyIdHash, absence, newRoster.root, newRoster.leafCount)` | subject not shown absent from the new roster |
| 6 | `verifyMembership(receipt.subjectKeyIdHash, presence, priorRoster.root)` | subject not shown present in the prior roster |

Every hex field MUST be validated as 64 lowercase hex characters before comparison; comparisons are exact string equality. The roster and receipt signers MAY be different keys (the example bundle uses `roster-1` and `sign-1`); both resolve through the same trust anchor.

### 6.4 Model disposition

**`verifyModelDisposition(receipt, { evidence?, destruction? }, resolveKey) → bool`** — in order, failing closed:

1. `kind == "forget.model-disposition.v1"`, `alg == "Ed25519"`, `disposition` is a §5.1 member.
2. Signature over the recomputed signed view (§5.2) verifies under `resolveKey(signerKeyId)`.
3. Shape invariant: `retrained` / `unlearned` MUST carry **both** training roots; the other two MUST carry **neither**.
4. For `retrained` / `unlearned`: `evidence` MUST be supplied; `evidence.trainingAfterSignedRoot.root == receipt.trainingRootAfter`; `verifyNonInclusionSigned(subjectKeyIdHash, nonInclusionAfter, trainingAfterSignedRoot, resolveKey)`. If `trainingBeforeSignedRoot` and `membershipBefore` are both supplied, `trainingBeforeSignedRoot.root == receipt.trainingRootBefore` and `verifyMembershipSigned(…)` MUST also pass.
5. Binding, when a `destruction` receipt is supplied: `destruction.subjectKeyIdHash == receipt.subjectKeyIdHash` **and** `receipt.destructionClaimId == "forget:" + destruction.subjectKeyIdHash + ":" + destruction.newRosterRoot`.

Without a `destruction` receipt the binding is not checked (the claim id's structure is enforced at minting only); verifiers that want the end-to-end verdict use §6.5.

### 6.5 End to end

**`verifyForgottenEndToEnd(destruction, disposition, { evidence? }, resolveKey)`** = `verifyDestructionReceipt(destruction, resolveKey)` **and** `verifyModelDisposition(disposition, { evidence, destruction }, resolveKey)`. True only when the records are evidenced gone **and** the model dimension is addressed for the same subject and the same erasure.

### 6.6 What the public verifier checks

[verify.dekimu.com](https://verify.dekimu.com) accepts `forget.destruction.v1` by value (paste the receipt; a `{ "receipt": … }` wrapper is tolerated) and runs §6.2 in the browser — the receipt never leaves the page. Its trust anchor is the public issuer-key catalogue at `verify.dekimu.com/.well-known/dekimu-claim-keys.json`. Verdicts:

| Verdict | Meaning |
|---|---|
| `verified` | All required fields present; `signerKeyId` found in the catalogue; signature valid over the recomputed signed view. |
| `unknown_key` | Receipt well-formed, but `signerKeyId` is not in the public catalogue (e.g. an issuer-held key), so the signature could not be checked. Not a failure of the receipt. |
| `invalid_signature` | Key found; signature does not verify (a signed field was altered, or `alg` is not `Ed25519`). |
| `malformed` | A required field is missing or the receipt could not be canonicalised. |
| `unrecognized` | `kind` is not one the public verifier handles. |

The public verifier does **not** currently evaluate roster roots, non-inclusion proofs, or `forget.model-disposition.v1` — those are §6.3–§6.5, run offline with an independent implementation of this document. A `verified` verdict there means "this issuer signed this destruction claim", not "the subject is absent".

---

## 7. Security considerations

### 7.1 Self-certifying is not self-trusting

§6 establishes that a bundle is cryptographically intact and internally consistent **relative to the key the verifier resolved**. A malicious issuer can produce a perfectly consistent bundle signed by a key it controls. A conforming verifier MUST obtain the public key out of band — pin it, resolve `keyId` / `signerKeyId` against a trusted-issuer registry, or check a transparency log — and only then run §6. Treating a `signerPublicKeys` convenience map inside a bundle as trusted is a verification error.

### 7.2 What a valid bundle does and does not say

See §1.1. In particular, non-inclusion shows absence from **the published signed roster**, not from "the issuer's current data" in some broader sense. The roster is honest only to the extent that the issuer builds it from the real set of held subjects; the construction makes the *roster* tamper-evident, not the issuer's inventory process.

### 7.3 The retention invariant (operational, not cryptographic)

Crypto-shred is only as strong as **key-registry-row retention discipline**. The wrapped data-encryption key row must be the single, non-resurrectable copy, held under a short or no backup window; bulky ciphertext may be backed up freely. A receipt cannot attest to backup policy. Unwrapped keys may also linger transiently in process memory until garbage collection; the deletion guarantee is about the authoritative wrapped row, and the claim should not be read wider than that.

### 7.4 Roster forgery classes a verifier MUST reject

| Attack | Defence |
|---|---|
| Truncated tree presented as the roster | `leafCount` is inside the signed view (§3.2). |
| Relabelled bracket — reuse an interval proof for a different value | Strict `lo < t < hi` check, and `intervalLeaf` recomputed from the claimed bounds (§3.4). |
| Re-split a genuine leaf preimage into different-length bounds | Fixed-length operand rule (§2.2). |
| Rolled-back roster — an older root presented as current | `prevRoot` chain is signed (§3.2); §6.3 step 4. Detecting that a *later* roster exists still needs the verifier to know the issuer's latest published root out of band. |
| Receipt replayed against a different roster | The receipt commits to both specific roots; §6.3 steps 3–4. |
| Any signed field altered | Recompute-on-verify + signature; no stored payload is ever trusted. |
| Disposition re-pointed at another subject's erasure | `destructionClaimId` binding, §6.4 step 5. |
| Evidence-free `retrained` / `unlearned` claim | §6.4 steps 3–4 fail closed. |

### 7.5 Personal data in the receipts

`subjectKeyIdHash` is a hash of the issuer's opaque **subject key id**, so a receipt about forgetting someone does not re-introduce their identity. This is a pseudonym, not an anonymisation: if an issuer used a guessable identifier (an email address, a sequential id) as the subject key id, anyone holding the receipt could confirm a guess with one SHA-256. Issuers SHOULD use high-entropy, opaque subject key ids. `rationale` is free text; issuers MUST NOT write personal data into it.

### 7.6 Timestamps and clocks

`generatedAt`, `destroyedAt`, and `disposedAt` are issuer-asserted and signed, not independently timestamped. An issuer can backdate. Where a regulatory clock matters (the Art. 12(3) one-month window), pair the bundle with an anchored, chained `ar.erasure.v1` / `ar.subject_rights.v1` receipt (§8) or an RFC 3161 timestamp; this specification adds no timestamping primitive.

### 7.7 Key separation

`kekKeyId` names the key-encryption key (destruction authority); `signerKeyId` / `keyId` name the Ed25519 signing key. They MUST be different keys, and the KEK itself never appears anywhere. The roster signer and the receipt signer MAY differ; both must resolve through the trust anchor.

---

## 8. Relationship to the Anchored Receipts family

`forget.*` receipts do not chain through the envelope `prev` field and have no `claim_type`. They compose with the family by value:

| Family receipt | Composition |
|---|---|
| `ar.erasure.v1` (ARR) | An anchored erasure receipt MAY carry the destruction **claim id** (`forget:<subjectKeyIdHash>:<newRosterRoot>`) or the full `forget.destruction.v1` object in its body, so the anchored log entry points at the checkable state proof. The anchored receipt supplies the trusted-time and chain properties `forget.*` lacks (§7.6); the `forget.*` bundle supplies the absence evidence the anchored receipt lacks. |
| `ar.subject_rights.v1` (ASR) | The Art. 17 request lifecycle. A fulfilled erasure request SHOULD reference the execution (`ar.erasure.v1`) which in turn references the destruction claim id — request → logged event → checkable state. |
| `ar.anonymization.v1` (ARR) | Not composed. Crypto-shred is erasure, not anonymisation; the two are different terminal events. |

The verifier dispatch order in [`CROSS-REFERENCES.md`](../CROSS-REFERENCES.md) §4 is unchanged; a `forget.*` object referenced from a family body is resolved after the envelope-level checks, by value, with this document's §6.

The reference issuer also exports a signed `ropa.register.v1` artifact that the public verifier understands; it is a separate by-value receipt kind and is out of scope here.

---

## 9. Legal note

These formats are designed to support a record of erasure under GDPR Art. 17 and the accountability principle of Art. 5(2). They do not provide legal advice and do not certify compliance. Whether a particular receipt satisfies a particular legal obligation is a question for the controller and the competent authority.

---

## 10. Examples

Every file in [`examples/`](examples/) was generated by the reference implementation from the inputs of its test suite and verified with the reference verifiers (§6.2–§6.5) before being committed. Subject key ids are the test strings `alice`, `bob`, `carol`, `subject-X`, `keep-1`, `keep-2`; signing seeds are fixed test seeds. **None of these keys is a production key — do not add them to any trust anchor.**

| File | Contents |
|---|---|
| [`destruction-receipt.json`](examples/destruction-receipt.json) | Bare `forget.destruction.v1` for the erasure of `bob` from `{alice, bob, carol}`. Pasteable into the public verifier (resolves as `unknown_key` there — test key). |
| [`roster-root.json`](examples/roster-root.json) | The `forget.roster.v1` "after" roster (`{alice, carol}`, `leafCount: 3`, chained to the "before" root). |
| [`destruction-bundle.json`](examples/destruction-bundle.json) | Receipt + both signed rosters + non-inclusion proof against the new root + membership proof against the prior root + the two test public keys. Passes §6.3 end to end. |
| [`model-disposition-not-trained.json`](examples/model-disposition-not-trained.json) | `not_trained` disposition bound to the destruction receipt above. Passes §6.4 with that receipt and §6.5 together with it. |
| [`model-disposition-retrained.json`](examples/model-disposition-retrained.json) | `retrained` disposition for `subject-X` removed from a training set `{subject-X, keep-1, keep-2}`, with the full evidence object (both signed training roots, non-inclusion after, membership before). Passes §6.4 step 4. |

---

## 11. Versioning and changelog

`forget.*.v1` wire shapes are frozen. New fields are additive and OPTIONAL with graceful degradation (absent, never `null`); anything that changes the canonical bytes of an existing valid receipt is a `.v2` kind.

### 1.0 — 2026-10-19 — first public release

Specification written from the shipped reference implementation and its test suite. Where the earlier internal design record and the shipped code differed, **the code wins** and the difference is recorded here so readers of either do not stumble:

- **Field names.** The design record named the destruction receipt's authority field `keyId` and its signing-key field `keyId_signer`; the shipped wire uses **`kekKeyId`** and **`signerKeyId`**. The signed roster root carries its signing key as **`keyId`** (the design record left it unnamed). This asymmetry — `keyId` on rosters, `signerKeyId` on receipts — is in the frozen wire and is preserved as-is.
- **Inclusion proof encoding.** Not fixed by the design record; the wire is `{ "siblings": [ { "hash", "side": "L" | "R" } ] }` (§3.3), with `side` naming the sibling's position and carried-up levels contributing no entry.
- **`leafCount` at verification.** The design record says the bare non-inclusion check "binds the count"; the shipped bare check validates only that it is a positive integer. The binding is in the roster **signature**, which the signed wrappers (§3.4) authenticate first. Stated honestly in §3.4.
- **Fixed-length operand rule** (§2.2) was added in the implementation after an adversarial review found the re-split forgery; the design record does not mention it.
- **Model disposition.** Matches the design record, with two precisions from the code: minting rejects a `destructionClaimId` that does not start with `forget:`; and `retrained` / `unlearned` require **both** training roots in the signed view even though membership-before evidence stays optional at verification.
- **Public verifier scope.** The design record's "offline verification" lists four steps; the public verifier at verify.dekimu.com implements the destruction-receipt signature check only (§6.6). The full bundle and model-disposition checks are offline-only in v1.
- **Optional anchoring hook** in the design record (passing the receipt into the anchored timeline) is not part of the wire format and is described only as a composition in §8.
