# 1 · Cloud deployment and environments

## Why it was hosted

OpenEMR runs happily in Docker on a laptop, and that is where most of the cohort
started. The problem with a laptop instance is that it is yours alone. Nobody
else can place an order in it, nobody else can pick that order up, and an
interoperability demonstration where both ends of the exchange run on the same
machine under the same login does not prove much.

Moving to a shared cloud instance was what made the rest of the work possible.
Once the EMR was reachable over the network, a physician account and a lab
account could be two different people in two different places, and an API client
could authenticate from outside the host entirely.

## What was running

| | |
|---|---|
| EMR | OpenEMR 7.0.2 |
| Host | AWS EC2 |
| Runtime | Docker |
| Database | MariaDB |
| Access | HTTPS over a public hostname |

Confirmed by the server's own capability statement:

```json
"software":       { "name": "OpenEMR", "version": "7.0.2" },
"implementation": { "description": "OpenEMR FHIR API" },
"fhirVersion":    "4.0.1"
```

See [`samples/capability-statement.json`](../samples/capability-statement.json).

## Three environments, and why that mattered

Work happened across three instances, and they were not identical.

**Local Docker.** Fast to break and rebuild. Where configuration was tried
first, before touching anything shared.

**Shared practice instance.** A clone of the demonstration environment, carrying
18 synthetic patients. Used to rehearse the workflow without risking the
environment everyone else was about to present from.

**Demonstration instance.** The one used for integrated testing and the go-live
simulation, carrying 45 patients, 25 observations and 2 diagnostic reports by
the time testing ran.

Keeping a practice instance separate from the demonstration instance is the part
I would repeat. Configuration work is destructive by nature, and the week before
a formal test is the wrong week to find out that a shared environment moved
underneath you.

The three instances also drifted. The local build and the hosted build advertised
different sets of FHIR resources, which is covered in
[findings](07-findings.md#environment-parity).

## HTTPS was not optional

FHIR endpoints in OpenEMR require TLS. On a local instance with a self-signed
certificate this shows up immediately as a certificate error, and the usual
workaround is to tell the client to skip verification. That is fine on a laptop
and wrong anywhere else, so the hosted instance was given a real hostname and a
valid certificate instead.

The capability statement exposes the SMART endpoints over that same hostname:

```json
"token":        "https://fhir.example.org/oauth2/default/token",
"authorize":    "https://fhir.example.org/oauth2/default/authorize",
"registration": "https://fhir.example.org/oauth2/default/registration",
"introspect":   "https://fhir.example.org/oauth2/default/introspect"
```

Hostnames in this repository are placeholders. The project instances have been
decommissioned.

---

Next: [FHIR R4 and OAuth2 →](02-fhir-oauth2-setup.md)
