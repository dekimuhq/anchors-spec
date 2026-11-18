# ABR profile `breach.cra.art14`

**Family:** ABR (`ar.breach.v1`) · **Profile id:** `breach.cra.art14` · **Events covered:** `breach.detected`, `breach.dpa_notified` · **Spec licence:** CC0 1.0.

The key words MUST, MUST NOT, SHOULD and MAY are used as in RFC 2119.

## 1. Purpose

This profile lets a manufacturer publish a signed, verifiable record of the timeline of a report made under **Regulation (EU) 2024/2847 (Cyber Resilience Act), Art. 14**: the manufacturer's reporting of (a) an actively exploited vulnerability and (b) a severe incident having an impact on the security of a product with digital elements, through the ENISA Single Reporting Platform (SRP).

It follows the same phased shape as `breach.nis2.art23`. It records *when* the manufacturer became aware and *when* each report phase was made. It does not submit anything to the SRP and does not record the content of a report.

The CRA facts used on this page are limited to those below and are stated per Art. 14; **verify against the primary text** before relying on them:

| Phase | Deadline stated in Art. 14 |
|---|---|
| Early warning | 24 hours |
| Notification | 72 hours |
| Final report, actively exploited vulnerability | 14 days after a corrective or mitigating measure is available |
| Final report, severe incident | one month after the notification |

## 2. Event mapping

| ABR event | Meaning under this profile |
|---|---|
| `breach.detected` | The time the manufacturer became aware. This is the reference time for the 24 h and 72 h clocks. As for every ABR profile, a trusted timestamp on this event is MANDATORY (see the family consistency table). |
| `breach.dpa_notified` | A report phase was made. `notification_phase` says which one (section 3). |

The event name `breach.dpa_notified` is inherited from the family; under this profile the recipient is the SRP channel, not a data protection authority. Other ABR event types are not constrained by this profile beyond the generic ABR rules.

## 3. Invariants

A body under this profile MUST satisfy every row below. A body that fails any row MUST be rejected by a verifier.

| Field | Requirement |
|---|---|
| `profile` | MUST be `breach.cra.art14`. |
| `incident_correlation_id` | MUST be a non-empty string. It ties every receipt of one report together. |
| `notification_phase` | On `breach.dpa_notified`, MUST NOT be `null`. |

Phase values on `breach.dpa_notified`:

| `notification_phase` | Report phase | Art. 14 deadline |
|---|---|---|
| `early_warning` | Early warning | 24 h from awareness |
| `full_notification` | Notification | 72 h from awareness |
| `final_report` | Final report | see section 4 |

## 4. What this profile does not express

The two final-report deadlines differ by what is being reported: 14 days after a corrective or mitigating measure is available (actively exploited vulnerability) versus one month after the notification (severe incident). **v1 has no body field that distinguishes a vulnerability report from a severe-incident report**, and no field for the availability time of a corrective measure.

Consequently:

- Deadline checks for this profile are **out of scope** in v1. The reference verifier skips the `notification_overdue` chain-walk warning for `breach.cra.art14`.
- Issuing applications that need deadline arithmetic MUST compute it themselves from the facts they hold.
- A verifier result under this profile says nothing about whether any deadline was met.

A later version MAY add a field for the report kind; that would be a spec amendment, not a change to this page's v1 invariants.

## 5. Verifier guidance

A successful result is **advisory**. It shows that the receipt is well-formed, its signature and anchor verify, and the invariants in section 3 hold. It does **not** show that a report reached the SRP, that its content was adequate, that the product is in scope of the CRA, or that any obligation was met. This profile makes no claim of conformance with, or recognition under, the CRA or any standard.

## 6. Example body

Placeholder values only. Envelope fields follow the envelope spec; fields not constrained by this profile follow the ABR body and are elided.

```json
{
  "event_type": "breach.dpa_notified",
  "profile": "breach.cra.art14",
  "incident_correlation_id": "inc-2027-0001",
  "notification_phase": "early_warning"
}
```

## 7. Reference implementation

The validator is `validateBreachCraArt14` in [`@dekimuhq/abr-profiles`](https://github.com/dekimuhq/abr-profiles) (dekimuhq/anchors#22). Its error codes are `breach.cra.art14.profile.mismatch`, `breach.cra.art14.incident_correlation_id.empty` and `breach.cra.art14.notification_phase.missing`.

This page describes what the profile checks. It does not give legal advice.
