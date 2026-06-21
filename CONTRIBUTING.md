# Contributing to `anchors-spec`

Thank you for the interest. Contributions are welcome across any family member, especially on:

- Conformance vectors that exercise edge cases of v1 canonicalisation, signature, anchor inclusion, or disclosure.
- Threat-model entries that meaningfully extend the defended / undefended boundary.
- Privacy-review entries that surface re-identification risks or document GDPR posture in jurisdictions we haven't covered.
- Editorial fixes (typos, ambiguous prose).
- New family member specs (see `<family>/README.md` placeholders for current status).

Contributions that fall outside this scope (large rewrites, framework-specific issuer code, alternative cryptography) will likely be declined politely — they belong in implementations or in a v2 conversation, not in v1 of the spec.

## Spec-stability rule

v1 of the spec is **frozen**. Concretely:

- v1.x is **additive only**. New optional fields, new test vectors, new prose clarifications, new threat-model entries — yes. Renaming a field, changing canonicalisation, narrowing an enum — no. This additive-only rule is the full versioning policy; each family's own `CHANGELOG.md` records its release history.
- Anything that breaks byte-equality of canonicalisation over previously-valid receipts is, by definition, v2.

If your change might be breaking, open a Discussions thread before you open a PR.

## How to propose a change

1. **For substantive proposals**, open a [Discussion](https://github.com/dekimuhq/anchors-spec/discussions) first. Describe the problem, the proposed change, and which surface it touches (wire format, threat model, privacy, vectors).
2. **For editorial fixes**, open a PR directly.
3. **For new test vectors**, include the regenerated files AND the regeneration tool change (typically in `dekimuhq/apr-verifier`).

## Developer Certificate of Origin (DCO)

All contributions to this project are made under the [Developer Certificate of Origin v1.1](https://developercertificate.org/) — the same DCO used by the Linux kernel.

The DCO is a developer's statement that they have the right to submit the contribution. The full text is:

```
Developer Certificate of Origin
Version 1.1

By making a contribution to this project, I certify that:

(a) The contribution was created in whole or in part by me and I
    have the right to submit it under the open source license
    indicated in the file; or

(b) The contribution is based upon previous work that, to the best
    of my knowledge, is covered under an appropriate open source
    license and I have the right under that license to submit that
    work with modifications, whether created in whole or in part
    by me, under the same open source license (unless I am
    permitted to submit under a different license), as indicated
    in the file; or

(c) The contribution was provided directly to me by some other
    person who certified (a), (b) or (c) and I have not modified it.

(d) I understand and agree that this project and the contribution
    are public and that a record of the contribution (including all
    personal information I submit with it, including my sign-off) is
    maintained indefinitely and may be redistributed consistent with
    this project or the open source license(s) involved.
```

You assert the DCO by signing off each commit:

```bash
git commit --signoff -m "spec: clarify canonicalisation NFC behaviour"
```

This appends a `Signed-off-by: Your Name <you@example.com>` line to the commit message. The line is required; CI will reject PRs whose commits are not signed off.

**There is no CLA.** Contributors retain copyright in their contributions; the project licence (CC0 for spec text, Apache 2.0 for accompanying code) is the only operative agreement.

## Code of conduct

This project follows the [Contributor Covenant 2.1](./CODE_OF_CONDUCT.md). Reports go to **conduct@dekimu.com**.

## Trademark notice

"APR" is unclaimed and free for downstream use, including the phrases "APR", "APR-compatible", and "APR receipt". "Dekimu APR" is a brand mark; see the `README.md` of any repo in this project for the full notice. Using "Dekimu APR" to describe an implementation that does not pass the v1 conformance vectors is not okay.
