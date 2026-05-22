# APR conformance — Phase 0 placeholder

This directory holds the **published test vectors** that any conforming `apr-verifier` MUST reproduce byte-for-byte.

A future Phase 1+ deliverable will extend this directory into a runnable **conformance harness**: a small CLI that takes a candidate verifier, drives it through the vector set, and reports pass/fail. The harness is NOT a Phase 0 deliverable — Phase 0 ships only the vectors and the prose around them.

## What lives here at v1.0.0

- [`test-vectors/`](./test-vectors/) — five vector groups, one subdirectory each (planned, not yet populated):
  - `01-valid-anchored/` — a valid signed claim with a `verified` anchor.
  - `02-tampered-body/` — same claim with one mutated body field; expected `tampered` / `signature_invalid`.
  - `03-missing-from-anchor/` — claim whose id is not in the published daily root for its anchor date; expected `tampered` / `missing-from-anchor`.
  - `04-rotated-kid/` — claim signed with an older `kid` whose key is still in the catalog; expected `verified` (rotation does not invalidate older receipts).
  - `05-unknown-kid/` — claim whose `kid` is not in the catalog; expected `tampered` / `unknown_kid`.

Each vector subdirectory will contain:

- `claim.json` — the serialised APR claim under test.
- `anchor.json` — the relevant daily-root envelope (or absent, where the case demands it).
- `verify-keys.json` — the claim-key catalog input the resolver should be loaded with.
- `root-keys.json` — the root-key catalog input.
- `expected.json` — the exact `{ status, reasons }` a conforming verifier must produce.
- `notes.md` — short prose explaining what the vector exercises.

## How vectors are generated

Vectors are deterministic. Test-only Ed25519 keypairs (NOT production keys) are committed alongside the public halves. Vector claims use **fixed `iat` timestamps and fixed ULIDs** so canonical bytes — and therefore signatures — reproduce exactly.

Generation tooling will live in [`apr-verifier`](https://github.com/dekimuhq/apr-verifier) under `scripts/regen-vectors.ts`. Generated vectors are then committed here. The vectors are the source of truth; the regeneration script exists only so a future canonicalisation patch can be re-validated.

## How to run vectors against a candidate verifier

A reference runner ships with `apr-verifier`:

```bash
cd apr-verifier
pnpm test:conformance --vectors ../apr-spec/conformance/test-vectors
```

Third-party verifiers SHOULD provide a similar runner that consumes the same on-disk format.

## Versioning

Vectors are pinned to the spec version they target. Vectors targeting v1.x live under this directory tree directly. A future v2 spec will introduce `conformance/v2/test-vectors/` alongside, leaving v1 vectors untouched and verifiable indefinitely.

## Conformance claim

A verifier conforms to APR v1.x when it:

1. produces the exact `expected.json` `status` for every vector in this directory tree,
2. produces a `reasons[]` superset of `expected.json.reasons` for failing vectors (additional informational reasons are allowed; missing reasons are not),
3. takes no network access during the run.

Implementations MAY publish their own conformance reports. There is no Dekimu-administered registry of conforming verifiers in v1; that decision is deferred to Phase 3.
