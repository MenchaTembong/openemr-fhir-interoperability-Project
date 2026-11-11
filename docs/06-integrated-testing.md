# 6 · Integrated testing and the go-live simulation

Configuration that works when you do it alone, in order, knowing where
everything is, is not the same as configuration that works when eight people
hand a patient between them under a script.

## How the test ran

A QA facilitator read a 47-step script in order, called each named actor, and
recorded Pass, Fail or N/A per step. Every step had a defined shape:

```
Action  →  Expected result  →  Required evidence  →  Pass / Fail
```

One role, one actor, one account. A step passed only when the expected result
was visible **and** the required evidence was captured. Knowing how to do the
task was explicitly not sufficient.

The journey ran front office → nurse → physician → laboratory → physician →
front desk → patient portal → analytics, with each role logging in under its own
account.

## My role

I was the **Laboratory / Interoperability tester**, steps 21 to 23, sitting
between the physician placing the order and the physician reviewing the result.

| Step | Action | Evidence required |
|---|---|---|
| 21 | Verify the laboratory and the agreed terminology: LOINC `4548-4` | Code-field screenshot |
| 22 | Process the order through the configured workflow, labelling external transmission as simulated | Order-status screenshot and simulation note |
| 23 | Enter the approved mock result, 7.8% | Result-entry screenshot |

Step 22 carried a condition worth repeating: the expected result required that
the handoff **not imply a live connection existed** where none was configured.
Claiming a working external laboratory integration would have failed the step,
not passed it.

The lab tester also had to be a different person from the physician. Running
both roles from one seat would have proved nothing about the handoff, which is
the thing the step existed to test.

## Rehearsal

Two dry runs preceded the formal test. Both overran their scheduled hour, which
is itself a finding: a 47-step script with evidence capture at each step does not
fit in sixty minutes, and planning as though it will is how a test turns into a
demonstration.

The dry runs surfaced problems before they could derail the real session:

- Access control gaps between roles, which the security team closed
- Provider availability misconfigured so the scheduling step reported the
  physician unavailable
- Physicians unable to add events to their own calendars
- Portal access needing work

None were test failures. They were readiness findings, and the point of
rehearsing was to convert them into fixes while there was still time.

## Go-live simulation

The simulation ran as three concurrent breakout rooms, each taking a different
patient journey: chronic disease follow-up, annual preventive care, and a
same-day acute visit. I was in Room 1.

The distinction the project lead drew between the two exercises is a real one
and worth keeping:

- **Integrated testing** runs on the test patient with test credentials
- **Go-live** runs on the designated "real" patient from the approved clinical
  storyline

Same system, different data, different posture.

## What testing actually found

The three scenarios completed. Testing also surfaced issues with role
permissions, portal access, navigation and some workflow handoffs, which were
documented rather than smoothed over.

That is the correct outcome. A test where everything passes on the first attempt
usually means the script was too shallow to find anything.

## What I took from it

**Evidence is the deliverable.** A step is not passed because the person
performing it knows it works. It is passed because someone else can look at the
screenshot afterwards and reach the same conclusion. When you configured the
thing yourself, that distinction is easy to lose and expensive to lose.

**Handoffs break more often than features.** The individual pieces had all been
tested. What had not been tested was the order carrying its context from one
account to the next, and that is where the friction was.

**Stating a limitation is a pass condition.** The script rewarded saying
plainly that the external transmission was simulated. That shaped how I now
describe scope: name what is real, name what is stubbed, and do not let an
audience fill the gap with an assumption that flatters the work.

---

Next: [Findings →](07-findings.md)
