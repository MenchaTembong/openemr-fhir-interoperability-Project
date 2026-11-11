# 7 · Findings

Observations from the retrieved data and the deployed environments. Each is
reproducible from the sample payloads in this repository.

---

## LOINC codes carry a redundant prefix

**Severity: would break a receiving system**

Laboratory observations retrieved from the project instance came back like this:

```json
"code": {
  "coding": [{
    "system":  "http://loinc.org",
    "code":    "LOINC:2093-3",
    "display": "Total Cholesterol"
  }]
}
```

The `code` element should carry the bare LOINC code. The `system` URI is what
identifies the code system, so repeating `LOINC:` inside the value is both
redundant and wrong:

```json
"code": "2093-3"
```

This is not cosmetic. A consumer performing the standard token search

```http
GET /Observation?code=http://loinc.org|2093-3
```

will not match `LOINC:2093-3`. The data looks correct to a human reading the
JSON and is invisible to a machine looking for it, which is the worst
combination of the two.

Affects every laboratory observation and diagnostic report in
[`observation-bundle.json`](../samples/observation-bundle.json) and
[`diagnostic-report-bundle.json`](../samples/diagnostic-report-bundle.json).
Vital signs are unaffected, because those codes are emitted by OpenEMR's own
US Core mapping rather than drawn from the imported code list, and they come
through clean:

```json
"code": { "coding": [{ "system": "http://loinc.org", "code": "8867-4",
                       "display": "Heart rate" }] }
```

The difference points at the import rather than at the FHIR layer: the prefix
was carried in when the codes were loaded, and everything downstream faithfully
reproduced it.

---

## Local codes presented as LOINC

**Severity: misleading**

Some results on the demonstration instance declared `http://loinc.org` as the
system while carrying values that are not LOINC codes at all:

```json
"system":  "http://loinc.org",
"code":    "1001",
"display": "Hemoglobin Result"
```

```json
"system":  "http://loinc.org",
"code":    "100",
"display": "Haemoglobin result"
```

`1001` and `100` are local identifiers. Hemoglobin is `718-7`. Declaring a local
code under the LOINC system URI is worse than declaring no system at all,
because a receiving system has no way to know the claim is false and will treat
the number as a LOINC code that happens not to resolve.

Two spellings of the same test, "Hemoglobin Result" and "Haemoglobin result",
also appear under two different local codes, which is precisely the problem
standard terminology exists to prevent.

Reproducible in
[`demo-observation-bundle.json`](../samples/demo-observation-bundle.json).

---

## A DiagnosticReport with no code

**Severity: minor, but it breaks search**

One report came back coded as unknown:

```json
"code": {
  "coding": [{
    "system":  "http://terminology.hl7.org/CodeSystem/v3-NullFlavor",
    "code":    "UNK",
    "display": "unknown"
  }]
}
```

The report has a subject, an encounter, a performer, a date and linked results.
What it does not have is any statement of what was tested. It will not appear in
a search filtered by `code`, and a clinician receiving it sees results without a
panel name.

Second entry in
[`demo-diagnostic-report-bundle.json`](../samples/demo-diagnostic-report-bundle.json).

---

## Duplicate entries in result bundles

**Severity: minor**

Several `DiagnosticReport.result` arrays reference the same observation twice:

```json
"result": [
  { "reference": "Observation/a248f22c-912a-...", "display": "Hemoglobin Result" },
  { "reference": "Observation/a248f22c-912a-...", "display": "Hemoglobin Result" }
]
```

Same UUID, listed twice. The corresponding observation bundle also returns the
same resource twice under one `fullUrl`. A consumer counting results would
double-count, and a chart rendering them would show the patient two identical
values.

Visible in both
[`demo-diagnostic-report-bundle.json`](../samples/demo-diagnostic-report-bundle.json)
and [`demo-observation-bundle.json`](../samples/demo-observation-bundle.json).

---

## Environment parity

**Severity: affects what you can promise**

The hosted 7.0.2 instance and a newer local build advertised different
capabilities:

| | Hosted 7.0.2 | Newer local build |
|---|---|---|
| US Core profiles | 3.1.1 | 3.1.1, 7.0.0, 8.0.0 |
| `Specimen` | not advertised | supported |
| `ServiceRequest` | not advertised | supported |
| `MedicationDispense` | not advertised | supported |
| `QuestionnaireResponse` | not advertised | supported |
| `ValueSet` | not advertised | supported |
| `_lastUpdated` search | limited | on most resources |

A workflow built and demonstrated locally can therefore depend on resources the
shared environment cannot serve. Any mapping row touching `Specimen` or
`ServiceRequest` is conditional on which build is deployed.

The practical lesson is to read the capability statement of the environment you
are actually going to present from, early, and treat the local build as a
convenience rather than a reference.

---

## Reference ranges are free text

**Severity: design constraint**

FHIR models a reference range as structured quantities:

```json
"referenceRange": [{ "low":  { "value": 4.0, "unit": "%" },
                     "high": { "value": 5.7, "unit": "%" } }]
```

OpenEMR stores it as a string: `< 5.7`. Anything downstream that needs to
evaluate a result against its range has to parse that string first, and the
format is whatever the person configuring the test happened to type.

The abnormal flag works because OpenEMR evaluates the range internally. The
problem appears at the boundary, when another system receives the result and
wants to make the same judgement for itself.

---

## Scope of these findings

All of the above come from a student-configured instance carrying synthetic
data. They are findings about this deployment and its configuration, not claims
about OpenEMR as a product. Several would likely resolve with a corrected
terminology import.

What they do illustrate is the distance between an interface that returns
`200 OK` and an interface another system can actually consume. The endpoint
working is the beginning of the question, not the answer to it.
