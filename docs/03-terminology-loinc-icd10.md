# 3 · Terminology: LOINC and ICD-10

## The problem terminology solves

Three laboratories can run the same blood test and report it under three
different names. One calls it HbA1c, one calls it Glycated Hemoglobin, one calls
it A1C. A human reading a fax works it out. A receiving system matching on a
text string does not, and the result either lands in the wrong place or does not
land at all.

LOINC fixes that by giving every laboratory test and clinical observation a
permanent identifier. The code answers one question — *what was measured* — and
the answer is the same in every system that uses it. HbA1c is `4548-4` whoever
is running the analyser.

ICD-10 does the same job for diagnoses. Type 2 diabetes without complications is
`E11.9`.

## LOINC in OpenEMR

OpenEMR supports LOINC as code type `110` in its `code_types` table. Support is
not the same as availability: **the codes are not preloaded**. Unlike ICD-10,
which has a documented install path in the administration interface, LOINC has
to be brought in by an administrator.

Once loaded, LOINC codes become selectable when building procedure definitions,
and they travel with orders and results from there.

Codes used in this project:

| Code | Test |
|---|---|
| `4548-4` | Hemoglobin A1c |
| `718-7` | Hemoglobin |
| `1989-3` | Vitamin D |
| `100898-6` | Lipid panel |
| `2093-3` | Total cholesterol |
| `2085-9` | HDL cholesterol |
| `13457-7` | LDL cholesterol (calculated) |
| `2571-8` | Triglycerides |

Vital signs arrive LOINC-coded without any import, because OpenEMR maps its
internal vitals form to the US Core vital signs profile on the way out:

| Code | Measurement |
|---|---|
| `85353-1` | Vital signs panel |
| `85354-9` | Blood pressure (panel) |
| `8480-6` | Systolic |
| `8462-4` | Diastolic |
| `8867-4` | Heart rate |
| `9279-1` | Respiratory rate |
| `8310-5` | Body temperature |
| `8302-2` | Body height |
| `29463-7` | Body weight |
| `39156-5` | BMI |
| `2708-6` / `59408-5` | Oxygen saturation |

## ICD-10

ICD-10 installs from the administration interface rather than needing a manual
import. Diagnoses recorded against a patient surface through FHIR as `Condition`
resources, and attach to laboratory orders as the clinical reason for the test.

The project used `E11.9` (Type 2 diabetes mellitus without complications) as the
chronic condition driving the HbA1c order, and `D50.9` (iron deficiency anaemia,
unspecified) in the referral case study.

## Where a LOINC code actually lives

The same code appears in a different slot depending on which standard is
carrying it. This is the part worth internalising, because it is what makes the
code portable:

| Standard | Location |
|---|---|
| OpenEMR configuration | `code_types` table, `ct_id` 110 |
| HL7 v2 result message | `OBX-3` observation identifier |
| FHIR R4 | `Observation.code.coding`, with `system` = `http://loinc.org` |
| C-CDA | Results section, observation code element |

In FHIR the shape is:

```json
"code": {
  "coding": [{
    "system":  "http://loinc.org",
    "code":    "4548-4",
    "display": "Hemoglobin A1c"
  }]
}
```

The `system` value is what tells a receiving system that `4548-4` is a LOINC
code and not some other coding system's number that happens to look the same.
Getting that URI right matters as much as getting the code right.

LOINC is the vocabulary. HL7 v2 and FHIR are two different roads it travels on.

## A conformance problem worth knowing about

Codes retrieved from the project instance came back carrying a `LOINC:` prefix
inside the code element itself:

```json
"system": "http://loinc.org",
"code":   "LOINC:2093-3"
```

That is not conformant, and a consumer matching on `2093-3` will not find it.
Written up in [findings](07-findings.md#loinc-codes-carry-a-redundant-prefix).

---

Next: [Lab order workflow →](04-lab-order-workflow.md)
