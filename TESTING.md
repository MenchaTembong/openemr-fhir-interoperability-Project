# Run this yourself

Everything in this repository can be reproduced against your own OpenEMR. You
need Docker and Postman, and about fifteen minutes.

Nothing here contains a credential. The Postman collection reads every secret
from variables you set locally.

---

## 1 · Start an OpenEMR

```bash
cd testing
docker compose up -d
```

First run takes several minutes: OpenEMR builds its database on startup. Watch
it with `docker compose logs -f openemr` and wait for the login page to answer
at **<https://localhost:9300>**.

Log in as `admin` / `pass`.

The certificate is self-signed, so your browser will object. That is expected
locally. FHIR endpoints will not answer over plain HTTP at all, which is why the
HTTPS port is the one to use.

## 2 · Turn the FHIR API on

`Administration → Globals → Connectors`, then enable:

- Enable OpenEMR Standard FHIR REST API
- Enable OpenEMR FHIR System Scopes
- Enable OpenEMR Standard REST API

Set **Site Address Override** to `https://localhost:9300` and save.

That field is not optional. OAuth2 builds its URLs from it, and leaving it blank
produces authorization failures that look like credential problems.

To use the password grant that the collection's token request expects, also set
**Enable OAuth2 Password Grant** to *On for Both Roles*. OpenEMR labels this
"not considered secure" and it is correct to do so. It is acceptable on a
laptop; see [the note on grant types](docs/02-fhir-oauth2-setup.md#step-3--get-a-token)
for why production uses the authorization code flow instead.

## 3 · Register a client

`Administration → System → API Clients → Register New App`

Create a confidential client and grant it these scopes:

```
openid  api:fhir
user/Patient.read  user/Observation.read  user/Encounter.read
user/DiagnosticReport.read  user/Condition.read
```

Two things that will cost you an hour if you miss them:

- **Copy the client secret immediately.** It is shown once and cannot be
  retrieved afterwards.
- **Enable the client.** New registrations are disabled by default, and the
  failure gives no hint about why.

## 4 · Load the collection

Import both files into Postman:

```
testing/openemr-fhir.postman_collection.json
testing/openemr-fhir.postman_environment.json
```

Select the environment, then fill in `clientId`, `clientSecret`, `username` and
`password`. Leave `accessToken` and `patientId` empty; the collection sets them
for you.

In Postman, turn **SSL certificate verification** off under Settings → General,
because the local certificate is self-signed.

## 5 · Run it

Run the requests in order. The first two matter most:

| Request | What it proves |
|---|---|
| `0 · Capability statement` | The API is on and reachable. Needs no token. Logs the FHIR version and the resources this build supports. |
| `1 · Get access token` | Authorization works. Stores the token for every request after it. |

If request 0 fails, nothing else will work. Fix that before going further.

From there, requests 2 to 9 retrieve patients, observations, diagnostic reports,
encounters and conditions.

### The requests that check conformance

Three of them assert things rather than just fetching:

**`4 · Laboratory observations`** fails if any code under the LOINC system URI
is not a well-formed LOINC code. Run it against an instance whose terminology
import carried a `LOINC:` prefix and it will fail, which is
[the defect documented in the findings](docs/07-findings.md#loinc-codes-carry-a-redundant-prefix).

**`6 · Search by LOINC code`** does a token search for Hemoglobin A1c. A result
stored under a malformed code will not come back, even though it is visibly an
HbA1c to anyone reading the chart. This is the request that shows why exact
coding matters.

**`7 · Diagnostic reports`** warns when a report carries no test code or lists
the same result twice.

A fresh OpenEMR has no laboratory data in it, so these will return empty
bundles until you create some. Build a procedure and result by following
[the lab order walkthrough](docs/04-lab-order-workflow.md).

---

## Checking against the sample payloads

If you would rather not stand an instance up, the `samples/` directory holds
real responses from the project environment with hostnames replaced and staff
records removed. The conformance problems are visible by inspection:

```bash
# LOINC codes that are not well-formed
grep -o '"code": "LOINC:[^"]*"' samples/observation-bundle.json | sort -u

# local codes declared under the LOINC system URI
grep -B1 '"display": "Hemoglobin Result"' samples/demo-observation-bundle.json

# a report with no test code
grep -A3 'v3-NullFlavor' samples/demo-diagnostic-report-bundle.json
```

## Shutting down

```bash
docker compose down         # keeps the data
docker compose down -v      # deletes it
```
