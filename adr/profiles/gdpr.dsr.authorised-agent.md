# ADR profile `gdpr.dsr.authorised-agent`

**Family:** ADR (`ar.delegation.v1`) · **Profile id:** `gdpr.dsr.authorised-agent` · **Event covered:** `grant` (revocation uses the generic `revoke` event) · **Spec licence:** CC0 1.0.

The key words MUST, MUST NOT, SHOULD and MAY are used as in RFC 2119.

## 1. Purpose

This profile lets a legal or natural person who acts as a data subject's **authorised representative** publish a signed, verifiable record of that authority, naming **one controller** and a closed set of GDPR data-subject rights. A controller (or anyone acting for it) can check the record without contacting the issuer.

Grounding stated by the profile: a national civil-law mandate or power of attorney granted by the data subject, together with GDPR Art. 12(2) (the controller facilitates the exercise of rights), Art. 12(6) (the controller may ask for information to confirm identity) and Recital 59 (requests made on the subject's behalf).

This profile is **not** `gdpr.art80.*`. GDPR Art. 80 concerns a mandate to a not-for-profit body with statutory objectives and is not the basis here. It is also not `gdpr.art28.processor`: the representative acts for the data subject, not as the controller's processor. And it is not an AI-agent profile: the representative is identified by a verification key and carries no AI `agent_descriptor`.

## 2. Grant body

A grant under this profile MUST set `profile` to `gdpr.dsr.authorised-agent` and `event` to `grant`. In addition to the generic ADR grant fields, a conforming grant MUST satisfy every row below. A grant that fails any row MUST be rejected by a verifier.

| Field | Requirement |
|---|---|
| `delegable` | MUST be `false`. The mandate cannot be sub-delegated. |
| `agent.agent_descriptor` | MUST be `null`. |
| `agent.agent_pubkey` | MUST be a non-empty string: the representative's verification key. |
| `scope.actions` | MUST be a non-empty array. Every element MUST be one of `gdpr-art-15` (access), `gdpr-art-16` (rectification), `gdpr-art-17` (erasure), `gdpr-art-18` (restriction), `gdpr-art-20` (portability), `gdpr-art-21` (objection). |
| `scope.resources` | MUST be an array with exactly one non-empty string naming the controller. Authority is per controller and is not transferable. |
| `scope.constraints.letter_hash` | MUST match `sha256:<64 lowercase hex>`. It binds the human-readable letter of authority (the document a person reads) to this signed record: the hash is over the exact bytes of that letter. |
| `scope.constraints.representative` | MUST be a non-empty string naming the representative legal or natural person (for example a registered company name). |
| `scope.constraints.mandate_basis` | MUST be a non-empty string stating the legal basis of the mandate: governing law and the GDPR provisions relied on. |

Validation applies to `grant` events only; other event types are not constrained by this profile beyond the generic ADR rules.

`letter_hash` is only as useful as the letter it commits to. An issuer SHOULD publish the letter at a stable location so a controller can recompute the hash.

### 2.1 Validity window

A grant SHOULD carry an expiry, both in the ADR body (`expiry`, Unix seconds or `null`) and in the envelope (`exp`). A verifier MUST NOT treat a grant as in force after its expiry has passed.

## 3. Revocation

The issuer withdraws a grant by publishing an ADR `revoke` event under the same profile id, with:

- envelope `prev` equal to the **id of the grant** being revoked;
- the same `iss` and `iss_key` as the grant;
- the same envelope `subject` commitment as the grant.

A verifier that holds a **trusted** successor of this form (valid signature, same issuer and key, `prev` pointing at the grant) MUST treat the grant as revoked. Verifiers SHOULD look for such a successor at the time of acting, because the absence of a revocation at one moment says nothing about a later one.

A revoke receipt shows that the issuer withdrew the authority from the time it was issued. It does not tell a controller when the withdrawal reached any particular party.

## 4. Verifier guidance (controllers)

The result of verifying a grant under this profile is **advisory**. A successful result shows that:

1. the receipt is well-formed and its signature and anchor verify;
2. a trusted issuer signed it;
3. it satisfies the invariants in section 2;
4. no trusted revoke successor was found and it has not expired;
5. it names a controller and a set of rights.

It does **not** show who the requester is, that the data subject is the person who authorised the letter, or that the underlying mandate is valid under any particular national law. The controller keeps the Art. 12(6) judgement and MAY still ask for further information.

A controller verifying a grant:

- MUST check that `scope.resources[0]` names the controller itself, and MUST NOT act on a grant naming a different controller;
- MUST check that the right being exercised appears in `scope.actions`;
- SHOULD recompute `letter_hash` over the letter presented to it and compare;
- SHOULD check for a revoke successor and for expiry at the time of acting.

## 5. Example grant body

Placeholder values only. Envelope fields (`id`, `iss`, `iss_key`, `subject`, `exp`, `prev`, `sig`, anchor) follow the ADR and envelope specs and are not repeated here.

```json
{
  "event": "grant",
  "profile": "gdpr.dsr.authorised-agent",
  "agent": {
    "agent_id": "did:web:representative.example",
    "agent_pubkey": "<representative verification key>",
    "agent_descriptor": null
  },
  "scope": {
    "actions": ["gdpr-art-17", "gdpr-art-21"],
    "resources": ["broker:example-controller"],
    "constraints": {
      "letter_hash": "sha256:0000000000000000000000000000000000000000000000000000000000000000",
      "representative": "Example Representative Ltd",
      "mandate_basis": "National civil-law mandate / power of attorney (governing law: the data subject's country of habitual residence); GDPR Art. 12(2) and 12(6), Recital 59; rights under Art. 17 and Art. 21. Not GDPR Art. 80."
    }
  },
  "delegable": false,
  "expiry": 1798761600,
  "rationale_uri": "https://representative.example/policy/authorisation",
  "rationale_hash": "sha256:<64 hex>",
  "qts_attestation_hash": null,
  "field_root": "<computed per envelope spec>"
}
```

The matching revocation body:

```json
{
  "event": "revoke",
  "profile": "gdpr.dsr.authorised-agent",
  "qts_attestation_hash": null,
  "field_root": "<computed per envelope spec>"
}
```

It is carried in an envelope whose `prev` is the grant's `id`.

## 6. Reference implementation

The validator is `validateGdprDsrAuthorisedAgent` in [`@dekimuhq/adr-profiles`](https://github.com/dekimuhq/adr-profiles) (profile added in the 0.4.0 release). Its error codes are prefixed `gdpr.dsr.authorised-agent.` (for example `letter_hash_invalid`, `actions_invalid`, `resources_must_name_single_controller`).

This page describes what the profile checks. It does not certify the legal effect of any mandate in any jurisdiction.
