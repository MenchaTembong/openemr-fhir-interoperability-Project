# Healthcare Interoperability with OpenEMR, FHIR R4 and LOINC

Standing up and testing a working FHIR R4 interface on OpenEMR, loading clinical
terminology into it, and proving that a laboratory order and result can travel
end to end through a configured clinical workflow.

**Mencha Tembong** — Health Informatics (T402), George Brown College
COMP 4090 Work Integrated Project, Spring/Summer 2026

[Portfolio page](https://menchatembong.github.io/openemr-fhir-interoperability-Project/) ·
[LinkedIn](https://www.linkedin.com/in/menchatembong) ·
[Video walkthroughs](https://www.youtube.com/@menchaemmanuel3762) ·
<Mencha.tembong@georgebrown.ca>

---

## Why the laboratory

I worked in a hospital laboratory in Yaoundé before I came to health informatics.
I have run the tests this project codes for, calibrated the analysers, kept the
quality control, and handed results to clinicians who had to act on them that
afternoon.

That is why the laboratory side of interoperability is where I am most useful.
When a reference range is stored as free text, or a result arrives under a local
code wearing a LOINC label, I know what it costs at the far end: a repeated
draw, a delayed diagnosis, a clinician deciding on a number that does not mean
what the chart says it means.

My responsibility used to end once an accurate result existed. This project
showed me how far that result still has to travel, and how much of the journey
is settled by configuration choices made long before anyone draws a sample.

## What this project was

HIT4EMR is a work-integrated project run through George Brown College. A student
cohort took OpenEMR, an open-source electronic medical record system, and
configured it into a simulated Ontario primary-care clinic called **Maple Grove
Family Health Centre**. Eight subteams covered clinical workflow, infrastructure,
security, UI/UX, interoperability, analytics, QA and documentation.

This repository covers my work on the **Interoperability** subteam, where the
focus was laboratory interoperability: getting clinical data in and out of
OpenEMR using FHIR R4, HL7 v2 and standard terminologies.

The clinic is fictional and all patient data is synthetic. This was an
educational simulation, not a production healthcare deployment and not a
certified system.

## What I did

- Enabled and secured the FHIR R4 API on a cloud-hosted OpenEMR 7.0.2 instance,
  including OAuth2 client registration and scope configuration
- Retrieved live `Patient`, `Observation`, `Encounter` and `DiagnosticReport`
  resources over authenticated requests in Postman
- Imported and validated the LOINC and ICD-10 code sets into OpenEMR
- Configured a Hemoglobin A1c laboratory order against LOINC `4548-4` with units
  and reference range, so it could be ordered and resulted
- Defined the data element mapping between OpenEMR's internal fields and FHIR
  R4, HL7 v2 and C-CDA
- Produced a future-state recommendation for where the lab interoperability work
  should go next
- Served as Laboratory / Interoperability tester during integrated testing and
  the go-live simulation

## Stack

| Layer | What was used |
|---|---|
| EMR | OpenEMR 7.0.2 |
| Host | AWS EC2, Docker, MariaDB, Linux |
| Exchange | FHIR R4 (4.0.1), HL7 v2, C-CDA |
| Terminology | LOINC, ICD-10, SNOMED CT |
| Authorization | OAuth2, SMART on FHIR, OpenID Connect |
| Tooling | Postman, Swagger UI, bash |

## Walkthrough

| Document | Covers |
|---|---|
| [01 · AWS deployment](docs/01-aws-deployment.md) | The hosted instance, HTTPS, and the two environments |
| [02 · FHIR R4 and OAuth2](docs/02-fhir-oauth2-setup.md) | Enabling the API, registering a client, scopes, getting a token |
| [03 · Terminology](docs/03-terminology-loinc-icd10.md) | Importing LOINC and ICD-10, and where the codes actually live |
| [04 · Lab order workflow](docs/04-lab-order-workflow.md) | Building an orderable HbA1c test and resulting it |
| [05 · Data element mapping](docs/05-data-element-mapping.md) | OpenEMR fields mapped across FHIR R4, HL7 v2 and C-CDA |
| [06 · Integrated testing](docs/06-integrated-testing.md) | My role in the scripted end-to-end test |
| [07 · Findings](docs/07-findings.md) | Conformance problems found in the retrieved data |
| [08 · Connecting a laboratory](docs/08-connecting-a-laboratory.md) | HL7 v2 orders and results, transport, and the OLIS certification boundary |

## Run it yourself

None of this has to be taken on trust. **[TESTING.md](TESTING.md)** walks through
standing up a local OpenEMR and pointing the included Postman collection at it.

```bash
cd testing && docker compose up -d     # OpenEMR at https://localhost:9300
```

| File | What it does |
|---|---|
| [`docker-compose.yml`](testing/docker-compose.yml) | A local OpenEMR 7.0.2 and MariaDB |
| [`openemr-fhir.postman_collection.json`](testing/openemr-fhir.postman_collection.json) | OAuth2 token request plus ten FHIR queries |
| [`openemr-fhir.postman_environment.json`](testing/openemr-fhir.postman_environment.json) | Variables to fill in locally |

Three of the requests assert rather than just fetch. One fails if a code under
the LOINC system URI is not a well-formed LOINC code, one shows that a
malformed code cannot be found by token search, and one warns on reports with no
test code or duplicated results. They are the findings below, written as checks
you can run against any instance.

No credential appears in any of these files.

## Sample payloads

Real responses from the project instance, with hostnames replaced and staff
records removed. Only the fictional patient cohort remains.

| File | Contents |
|---|---|
| [`capability-statement.json`](samples/capability-statement.json) | What the server advertises: FHIR 4.0.1, supported resources, SMART endpoints |
| [`patient-bundle.json`](samples/patient-bundle.json) | 16 synthetic patients |
| [`observation-bundle.json`](samples/observation-bundle.json) | 8 LOINC-coded lipid panel results |
| [`diagnostic-report-bundle.json`](samples/diagnostic-report-bundle.json) | 2 reports linking to those observations |
| [`demo-observation-bundle.json`](samples/demo-observation-bundle.json) | 25 observations from the demonstration instance |
| [`demo-diagnostic-report-bundle.json`](samples/demo-diagnostic-report-bundle.json) | 2 reports from the demonstration instance |

The last two are the evidence behind
[the conformance findings](docs/07-findings.md).

## Evidence

Screenshots from the work. Credentials, client identifiers, session tokens and
bearer tokens are blacked out.

| | |
|---|---|
| [Enabling the FHIR API](images/openemr-fhir-enable.png) | Connector settings with the FHIR REST API and OAuth2 options turned on |
| [Connector configuration](images/openemr-connectors-config.png) | US Core version and OAuth2 approval settings |
| [Client registration](images/oauth2-client-registration.png) | A registered API client and the scopes granted to it |
| [Token request](images/oauth2-token-request.png) | OAuth2 token exchange returning 200 OK |
| [Capability statement](images/fhir-capability-statement.png) | `GET /fhir/metadata`, confirming FHIR 4.0.1 |
| [Patient query](images/fhir-patient-request.png) | Authenticated `GET /fhir/Patient` |
| [Observation query](images/fhir-observation-request.png) | `GET /fhir/Observation` returning a result bundle |
| [Encounter query](images/fhir-encounter-request.png) | `GET /fhir/Encounter` with SNOMED CT class coding |
| [Procedures menu](images/openemr-procedures-menu.png) | Lab menu items, inactive until a patient chart is open |

## Elsewhere

- Cohort project showcase: <https://mgfhc.github.io/>
- Video walkthroughs: <https://www.youtube.com/@menchaemmanuel3762>

---

### A note on the data

Every patient in this repository is invented. The lab values are made up. The
clinic does not exist. Nothing here came from a real patient record, and nothing
here should be read as a working configuration for production use.
