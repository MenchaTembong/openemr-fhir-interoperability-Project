# 5 · Data element mapping

Before you can argue that two systems will understand each other, you have to be
able to say which field goes where. This mapping sets OpenEMR's internal storage
against the three exchange formats in scope: FHIR R4, HL7 v2 and C-CDA.

It was the analysis deliverable I owned on the interoperability team, and it fed
the future-state recommendation that followed it.

## Patient demographics

| OpenEMR | FHIR R4 | HL7 v2 | C-CDA |
|---|---|---|---|
| `patient_data.pubpid` | `Patient.identifier` | `PID-3` | `patientRole/id` |
| `patient_data.lname` | `Patient.name.family` | `PID-5.1` | `patient/name/family` |
| `patient_data.fname` | `Patient.name.given` | `PID-5.2` | `patient/name/given` |
| `patient_data.DOB` | `Patient.birthDate` | `PID-7` | `patient/birthTime` |
| `patient_data.sex` | `Patient.gender` | `PID-8` | `patient/administrativeGenderCode` |
| `patient_data.street` | `Patient.address.line` | `PID-11.1` | `patientRole/addr/streetAddressLine` |
| `patient_data.city` | `Patient.address.city` | `PID-11.3` | `patientRole/addr/city` |
| `patient_data.state` | `Patient.address.state` | `PID-11.4` | `patientRole/addr/state` |
| `patient_data.postal_code` | `Patient.address.postalCode` | `PID-11.5` | `patientRole/addr/postalCode` |
| `patient_data.phone_cell` | `Patient.telecom` (phone) | `PID-13` | `patientRole/telecom` |
| `patient_data.email` | `Patient.telecom` (email) | `PID-13.4` | `patientRole/telecom` |
| `patient_data.language` | `Patient.communication.language` | `PID-15` | `languageCommunication` |

### A real-world wrinkle

OpenEMR's address model is built around US states. For an Ontario clinic the
UI/UX team replaced the state dropdown with Canadian provinces and territories,
which is a presentation fix. The field underneath is still `state`, and it still
maps to `Patient.address.state`.

That is not a defect, but it is the kind of thing that has to be written down.
A receiving system reading `"state": "ON"` needs to know it is looking at an
Ontario province code and not a malformed US state. Province and postal-code
format are both places a US-origin EMR needs explicit handling before it is
honest to call it Ontario-ready.

## Laboratory orders and results

| OpenEMR | FHIR R4 | HL7 v2 | C-CDA |
|---|---|---|---|
| `procedure_order.procedure_order_id` | `ServiceRequest.identifier` | `ORC-2` / `OBR-2` | order id |
| `procedure_type.procedure_code` | `Observation.code.coding.code` | `OBR-4` / `OBX-3` | observation `code` |
| `procedure_order.date_ordered` | `ServiceRequest.authoredOn` | `ORC-9` | `author/time` |
| `procedure_order.provider_id` | `ServiceRequest.requester` | `ORC-12` | `author/assignedAuthor` |
| `procedure_order.diagnoses` | `ServiceRequest.reasonCode` | `DG1-3` | `entryRelationship` → problem |
| `procedure_order.specimen_type` | `Specimen.type` | `SPM-4` | specimen entry |
| `procedure_result.result` | `Observation.valueQuantity.value` | `OBX-5` | `observation/value` |
| `procedure_result.units` | `Observation.valueQuantity.unit` | `OBX-6` | `value/@unit` |
| `procedure_result.range` | `Observation.referenceRange.text` | `OBX-7` | `referenceRange` |
| `procedure_result.abnormal` | `Observation.interpretation` | `OBX-8` | `interpretationCode` |
| `procedure_result.result_status` | `Observation.status` | `OBX-11` | `statusCode` |
| `procedure_report.date_report` | `DiagnosticReport.issued` | `OBR-22` | `effectiveTime` |
| `procedure_report.report_notes` | `DiagnosticReport.conclusion` | `NTE-3` | `text` |

The LOINC code appears twice on purpose. In HL7 v2 it rides in `OBR-4` on the
order and `OBX-3` on each result, which is what lets a receiving system match a
result back to the test that was requested.

## Vital signs

Vitals map to the US Core vital signs profile without any configuration, which
is the one area where OpenEMR's FHIR output is LOINC-clean out of the box.

| OpenEMR (`form_vitals`) | LOINC | FHIR |
|---|---|---|
| `bps` | `8480-6` | `Observation.component` (systolic) |
| `bpd` | `8462-4` | `Observation.component` (diastolic) |
| — | `85354-9` | parent blood pressure Observation |
| `pulse` | `8867-4` | `Observation.valueQuantity` |
| `respiration` | `9279-1` | `Observation.valueQuantity` |
| `temperature` | `8310-5` | `Observation.valueQuantity` |
| `height` | `8302-2` | `Observation.valueQuantity` |
| `weight` | `29463-7` | `Observation.valueQuantity` |
| `BMI` | `39156-5` | `Observation.valueQuantity` |
| `oxygen_saturation` | `2708-6` / `59408-5` | `Observation.valueQuantity` |

Blood pressure is the structurally interesting one. It is not two observations;
it is one observation carrying two components, each separately coded:

```json
"code": { "coding": [{ "system": "http://loinc.org", "code": "85354-9" }] },
"component": [
  { "code": { "coding": [{ "code": "8480-6" }] },
    "valueQuantity": { "value": 120, "unit": "mm[Hg]" } },
  { "code": { "coding": [{ "code": "8462-4" }] },
    "valueQuantity": { "value": 80,  "unit": "mm[Hg]" } }
]
```

Flattening that into two unrelated observations loses the fact that the two
numbers were taken together, which is the whole clinical point of a blood
pressure reading.

## Encounters and diagnoses

| OpenEMR | FHIR R4 | HL7 v2 | C-CDA |
|---|---|---|---|
| `form_encounter.encounter` | `Encounter.identifier` | `PV1-19` | `encompassingEncounter/id` |
| `form_encounter.date` | `Encounter.period.start` | `PV1-44` | `effectiveTime/low` |
| `form_encounter.facility_id` | `Encounter.location` | `PV1-3` | `location/healthCareFacility` |
| `form_encounter.provider_id` | `Encounter.participant` | `PV1-7` | `performer` |
| `form_encounter.reason` | `Encounter.reasonCode` | `PV2-3` | `reasonCode` |
| `lists` (type `medical_problem`) | `Condition.code` | `DG1-3` | problem observation `value` |
| `lists.begdate` | `Condition.onsetDateTime` | `DG1-5` | `effectiveTime/low` |

Encounter class and type come back SNOMED CT coded, which is covered in the
[FHIR setup notes](02-fhir-oauth2-setup.md#step-4--query).

## What the mapping exposed

Writing it out surfaced three things that a narrative description would have
hidden:

**Specimen is not reachable on every build.** The hosted 7.0.2 instance did not
advertise `Specimen` or `ServiceRequest` in its capability statement, although a
newer local build did. Any mapping row that depends on those resources is
conditional on the deployed version.

**Reference ranges arrive as text, not structure.** FHIR models
`referenceRange.low` and `referenceRange.high` as quantities. OpenEMR stores the
range as a free-text string like `< 5.7`. Anything that needs to compute against
the range has to parse it first, and parsing free text is where interoperability
goes to die.

**Not every mapped field is populated.** A row in this table says a mapping
exists, not that data flows through it. Several observations in the retrieved
bundles carry `dataAbsentReason: unknown` rather than a value, because the
underlying vitals field was left blank at the point of care. The mapping is
correct and the data is still missing, and those are different problems with
different owners.

---

Next: [Integrated testing →](06-integrated-testing.md)
