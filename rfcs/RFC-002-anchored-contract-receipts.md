# RFC-002: Anchored Contract Receipts (ACoR)

**Authors:** Dekimu
**Status:** Draft
**Created:** 2026-05-25
**Envelope version target:** v1.2 (no envelope changes needed)

---

## 1. Motivation

The current 14-member family covers data protection (GDPR), AI governance (AI Act), and trust services (eIDAS). A gap exists for **contract lifecycle events** — execution, amendment, termination, renewal, and breach-of-contract notification.

Contracts are the legal foundation of B2B SaaS relationships. EU regulations increasingly require verifiable evidence of contract execution (eIDAS QES), amendment histories (Directive 2019/771 for digital content), and payment term compliance (Late Payment Directive 2011/7/EU).

ACoR fills this gap with 5 event types under a single `ar.contract.v1` claim type.

## 2. Proposed Changes

### 2.1 New claim type

```
ar.contract.v1
```

Uses one of the 3 parked discriminator slots in REGISTRY.md.

### 2.2 Event types

| Event type | Description | Subject | TSA |
|---|---|---|---|
| `contract.executed` | Contract signed by all parties | `sha256:<hex>` counterparty commit | MANDATORY |
| `contract.amended` | Amendment to existing contract | `sha256:<hex>` counterparty commit | RECOMMENDED |
| `contract.terminated` | Contract terminated | `sha256:<hex>` counterparty commit | RECOMMENDED |
| `contract.renewed` | Contract renewed with updated terms | `sha256:<hex>` counterparty commit | RECOMMENDED |
| `contract.breach_noticed` | Breach-of-contract notification sent | `sha256:<hex>` counterparty commit | RECOMMENDED |

### 2.3 Body schema

```typescript
interface ACoRBody {
  event_type: "contract.executed" | "contract.amended" | "contract.terminated"
    | "contract.renewed" | "contract.breach_noticed";
  contract_id: string;
  contract_type: "saas" | "service" | "license" | "nda" | "dpa" | "other";
  parties: Array<{
    role: "controller" | "processor" | "counterparty" | "witness";
    commit: string; // sha256:<hex>
  }>;
  terms_hash: string; // sha256:<hex> of the contract document
  terms_uri: string | null;
  governing_law: string; // ISO 3166-1 alpha-2 jurisdiction code
  effective_date: string; // ISO 8601
  expiry_date: string | null;
  amendment_ref: string | null; // previous receipt ID for amendments
  breach_description: string | null; // only on breach_noticed
  qts_attestation_hash: string | null;
}
```

### 2.4 Regulatory profiles

| Profile ID | Regulation | Requirements |
|---|---|---|
| `acor.eidas.qes` | eIDAS Art. 25-26 | MANDATORY TSA + QES signing cert chain |
| `acor.directive.2019-771` | EU Directive 2019/771 | Digital content contract conformity |
| `acor.late-payment.2011-7` | EU Late Payment Directive | Payment terms, interest rate reference |
| `acor.gdpr.art28` | GDPR Art. 28 | Data processing agreement specific fields |
| `acor.general` | No specific regulation | General contract receipt |

## 3. Backward Compatibility

No envelope changes. ACoR uses the existing v1.2 envelope with a new `claim_type` value. Existing verifiers will reject `ar.contract.v1` with `claim_type_unknown` — correct behavior for verifiers that predate ACoR.

## 4. Security Considerations

- **Non-repudiation:** `contract.executed` with MANDATORY TSA provides timestamp evidence for dispute resolution. eIDAS QES profile adds qualified signature status.
- **Tampering:** `terms_hash` anchors the contract document content. Any modification to the document is detectable.
- **Party impersonation:** Each party identified via `sha256:<hex>` commit — same salted-commit pattern as ACR/ARR.
- **Replay:** Unique receipt `id` + `contract_id` + `prev` chain prevents replay.

## 5. Test Vectors

Minimum set (deferred to implementation):
- Valid `contract.executed` receipt with eIDAS QES profile
- Valid `contract.amended` receipt with `amendment_ref` pointing to prior
- Invalid: missing `contract_id`
- Invalid: `contract.executed` without TSA under eIDAS QES profile

## 6. Implementation Notes

Estimated scope: ~2 weeks (verifier + issuer-kit + profiles).
Affected packages (new):
- `@dekimuhq/acor-verifier`
- `@dekimuhq/acor-issuer-kit`
- `@dekimuhq/acor-profiles`

FAMILY-CONSISTENCY.md updates required: chip position, TSA matrix, subject presence, Status List consumer, shared-package imports, engineering budget.

## 7. References

- eIDAS Regulation (EU) No 910/2014
- eIDAS 2.0 amendments (Regulation (EU) 2024/1183)
- EU Directive 2019/771 (sale of goods / digital content)
- EU Late Payment Directive 2011/7/EU
- GDPR Art. 28 (processor agreements)
