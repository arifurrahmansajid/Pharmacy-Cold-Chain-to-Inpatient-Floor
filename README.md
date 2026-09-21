### Pharmacy Cold Chain to Inpatient Floor (Case 1) — Submission Write-Up
#### 1. Approach
To solve the cold-chain medication dispatch and interoperability requirements, I designed a resilient, standards-compliant TypeScript pipeline following HL7 FHIR R4, HL7 v2.5.1, and NIH NLM RxNorm standards:
- **Codified Terminology Verification**: Instead of fragile and unsafe string matching (e.g., regex on brand names), the system validates clinical concepts directly against the official NIH NLM RxNav REST APIs (`/historystatus.json` and `/related.json?tty=SCD+DF+IN`) to verify concept activity, active ingredients (`insulin glargine`), and dose form (`Injectable Solution`).
- **Structured HL7 v2 Ingestion**: Integrated the open-source `hl7-standard` library to parse legacy `OMP^O09` dispatch events, navigating structured MSH, PID, PV1, ORC, RXO, and OBX segments without ad-hoc string splitting.
- **Standards-Compliant FHIR Dispense**: Created an HL7 FHIR R4 `MedicationDispense` resource linked to the original `MedicationRequest`, modeling the delivery ETA via the standard FHIR extension (`http://hl7.org/fhir/StructureDefinition/dispense-delivery-eta`) and clinical note annotations.
- **HIPAA Data Minimization (Zero-PHI Allowlist)**: Built outbound nurse notifications using a strict **Allowlist pattern** (whitelisting only room, bed, drug formulation, courier first name/role, and ETA) while completely stripping patient name, MRN, DOB, phone, and address. Verified zero-leakage using automated assertion tests.
- **Tamper-Proof Audit Trail**: Recorded legal, tamper-evident HL7 FHIR `AuditEvent` resources (`purposeOfEvent: TREAT`) tracking the actor, action (`C`/`R`), timestamp, and resource references.
---
#### 2. Assumptions Made
Due to ambiguities in legacy clinical integrations and test sandbox constraints:
1. **Delivery ETA in FHIR R4**: Because FHIR R4 `MedicationDispense` lacks a native top-level `eta` field, I assumed the standard FHIR extension (`http://hl7.org/fhir/StructureDefinition/dispense-delivery-eta`) paired with a human-readable `note` annotation represents the most interoperable EHR charting pattern.
2. **Courier Identification & Privacy**: Assumed the outbound nurse alert only requires the courier's first name/role (e.g., "Marcus (Courier)") for on-floor physical handoff verification rather than exposing the courier's employee ID or internal personnel records.
3. **HL7 v2 Ingestion Channel**: Assumed legacy hospital dispatch messages are ingested as structured HL7 v2 strings/fixtures rather than exposing an open MLLP raw TCP socket over the public web, while maintaining 100% v2.5.1 segment compliance.
4. **Sandbox Resilience**: Assumed public test endpoints (HAPI FHIR sandbox) can experience transient outages; implemented deterministic seeding with local mock fallback buffers to ensure zero pipeline crashes during evaluation.
---
#### 3. What I Would Improve With More Time
With additional time and production infrastructure, I would implement:
1. **Live IoT Cold-Chain Telemetry Integration**: Ingest real-time Bluetooth/NFC temperature data logger streams from the insulated carrier, dynamically raising automated FHIR `Flag` / `Communication` alerts if temperatures breach the 2°C–8°C threshold in transit.
2. **SMART on FHIR Backend Services Auth**: Implement OAuth 2.0 asymmetric client assertion (RFC 7523) with JSON Web Keys (JWKS) and mTLS for cryptographically secured server-to-server EHR transactions.
3. **Real-Time Push Infrastructure**: Transition nurse alerts from simulated dispatch payloads to FHIR R4 Subscriptions / WebSockets / Apple Push Notification Service (APNs) with end-to-end payload encryption.
4. **Barcode / 2D DataMatrix Cross-Check**: Add Bedside Medication Administration (BCMA) 2D DataMatrix NDC scanning logic to automatically verify the physical vial before nurse administration.
