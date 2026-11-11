# 8 · Connecting a real laboratory

The external transmission in this project was simulated. This document covers
what a real connection would take, where OpenEMR already supports it, and the
one requirement that puts a live Ontario connection out of reach for a student
project.

Being precise about that boundary is the point. A demonstration that implies a
working provincial integration is making a false claim.

## What a live lab interface is

Laboratory interoperability runs on HL7 v2 and a pair of message types:

| Message | Direction | Carries |
|---|---|---|
| `ORM^O01` | EMR → laboratory | The order: patient, test, diagnosis, ordering provider |
| `ORU^R01` | laboratory → EMR | The result: value, units, reference range, abnormal flag |

The order goes out, the specimen is collected and analysed, the result comes
back and files itself against the original order. The link between the two is
the placer order number, and the test identity on both ends is the LOINC code.

### An order, in outline

```
MSH|^~\&|OPENEMR|MAPLE_GROVE|LABCORP|LAB|20260725103000||ORM^O01|MSG001|P|2.5.1
PID|1||P03^^^MAPLE^MR||Lee^George||19681109|M
ORC|NW|ORD4821||||||||| |1234^Maple^Alyssa
OBR|1|ORD4821||4548-4^Hemoglobin A1c^LN|||20260725103000
DG1|1||E11.9^Type 2 diabetes mellitus without complications^I10
```

### The result coming back

```
MSH|^~\&|LABCORP|LAB|OPENEMR|MAPLE_GROVE|20260726081500||ORU^R01|MSG002|P|2.5.1
PID|1||P03^^^MAPLE^MR||Lee^George||19681109|M
OBR|1|ORD4821||4548-4^Hemoglobin A1c^LN|||20260725103000|||||||||F
OBX|1|NM|4548-4^Hemoglobin A1c^LN||7.8|%|<5.7|H|||F
```

Read `OBX` left to right: numeric result, LOINC `4548-4`, value `7.8`, units `%`,
reference range `<5.7`, abnormal flag `H`, status `F` for final.

The `LN` in `4548-4^Hemoglobin A1c^LN` is the HL7 identifier for LOINC. It plays
exactly the role that `system: "http://loinc.org"` plays in FHIR. Same job,
different notation, same consequence if you get it wrong.

Note that the reference range arrives as the string `<5.7`, which is where
[the free-text range problem](07-findings.md#reference-ranges-are-free-text)
originates. HL7 v2 models `OBX-7` as text, and OpenEMR stores what it is given.

## Where this is configured in OpenEMR

`Administration → Practice → Procedure Providers`

A procedure provider is a laboratory. Each one carries:

| Field | Meaning |
|---|---|
| Name | The laboratory, as it appears on orders |
| Identifier | Its sending/receiving application code in `MSH` |
| Protocol | How messages move: file drop, SFTP, or a direct connection |
| Direction | Whether this provider sends results, receives orders, or both |
| Remote host, login | Credentials for the transport |

Once a provider exists, it becomes selectable as the destination on a procedure
order, and OpenEMR will generate the `ORM` for it. Inbound results are matched
back by placer order number.

This part is real and works. OpenEMR ships with a laboratory exchange module and
a results inbox; what the project simulated was the network hop, not the
message handling.

## Transport

Most laboratory interfaces are not REST. In descending order of how often you
will meet them:

**SFTP file exchange.** The EMR writes `ORM` files to an outbound directory and
polls an inbound directory for `ORU` files. Unglamorous, widely deployed, and
easy to audit because every message is a file you can open.

**MLLP over TCP.** The classic HL7 transport, framing messages on a socket.
Lower latency, and requires an always-on listener and a VPN or private circuit.

**An interface engine** such as Mirth Connect sitting between the two, doing
transformation, routing and retry. This is what most real deployments use, and
it is where the mapping work in [document 05](05-data-element-mapping.md) earns
its keep: the engine needs to be told which field becomes which.

**FHIR.** Increasingly offered for querying results, rarely yet for placing
orders. The direction of travel, not the current default.

## The Ontario reality

Ontario laboratory results are held in **OLIS**, the provincial Laboratory
Information System. It carries results from the major community laboratories and
many hospitals, and it is the route to a patient's full laboratory history
rather than only the results your own clinic ordered.

An EMR does not simply connect to OLIS. Access runs through an interface that is
**certified against the OntarioMD EMR Specification**, and the vendor configures
that certified interface. It is a governed, credentialed connection, not a
generic HL7 feed you can point at a hostname.

The query model is worth knowing:

- a **practitioner query** runs periodically, perhaps every thirty minutes,
  pulling new or updated results into the EMR
- a **preload query** pulls historic results on demand for a patient

**OpenEMR is not an OntarioMD-certified EMR.** It therefore has no OLIS
connector, and no student project can stand one up. That is a certification
boundary, not a technical gap to be engineered around.

What remains true, and is the reason this work still connects to the Ontario
context: OLIS depends on HL7 v2 for transport and LOINC for test identification.
Those are precisely the standards configured and demonstrated here. The
vocabulary is the same vocabulary. Only the certified pipe is missing.

## What a real deployment would need beyond this

The project proved the clinical workflow and the coding. Production would add:

**A sending facility agreement.** Laboratories do not accept orders from
unknown systems. There is an onboarding process, test messages, and a
conformance review before anything goes live.

**Acknowledgements and retries.** HL7 defines `ACK` messages. A real interface
has to handle a laboratory that does not answer, a message that fails to parse,
and a result that arrives twice.

**Reconciliation.** Orders that never return a result have to surface somewhere
a human will look, or a patient quietly falls through.

**Audit and privacy controls.** Every message carries personal health
information. PHIPA obligations around logging, retention and access apply to
the interface, not only to the chart.

**Monitoring.** Interfaces fail silently. The failure mode that matters is not
an error page; it is results that stop arriving while everything appears normal.

Those five are the gap between a configured workflow and a clinical system
people can depend on, and they are the substance of the future-state
recommendation this project closed with.

---

Back to [the findings](07-findings.md) · [repository root](../README.md)
