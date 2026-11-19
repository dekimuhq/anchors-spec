# Anchored Breach Receipts (ABR)

Wire type: `ar.breach.v1`

Records breach detection, assessment, and notification lifecycle. Eight event types from detection through DPA notification, subject notification, containment, and closure. Designed for GDPR Arts. 33-34, NIS2 Art. 23, DORA Art. 19, and CRA Art. 14.

## Reference Implementation

- Verifier: [`@dekimuhq/abr-verifier`](https://github.com/dekimuhq/abr-verifier)
- Issuer kit: [`@dekimuhq/abr-issuer-kit`](https://github.com/dekimuhq/abr-issuer-kit)
- Profiles: [`@dekimuhq/abr-profiles`](https://github.com/dekimuhq/abr-profiles)

## Profiles

Six profiles are registered in the family definition (`breach.gdpr.art33`, `breach.gdpr.art34`, `breach.nis2.art23`, `breach.dora.art19`, `breach.cra.art14`, `breach.us_state.attorney_general`); the implementation's `AbrProfile` union additionally lists four `breach.nis2.annex1.*` sectoral sub-profiles (10 members). Only the CRA profile has a profile spec page so far:

- [`breach.cra.art14`](profiles/breach.cra.art14.md) — manufacturer reporting under Regulation (EU) 2024/2847 Art. 14 (early warning, notification, final report).

## Status

Spec: profile `breach.cra.art14` is specified (see above); the rest of the family, including the event types and the other profiles, is a placeholder — full spec pending.
Reference implementation: shipped (v0.3.x on GitHub Packages).
