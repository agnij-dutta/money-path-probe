# Money-path probe: <product>

**Probed** <date> · **Environment** test mode / sandbox · **Account** <your own test account>
**Versions** API version, SDK version, server version if exposed

## Summary

<One paragraph. How many probes ran, how many found something, and the single most important
finding with who loses money on it. Written so the maintainer can decide priority without
reading further.>

## Findings

### F-1 · <one-line title> · <Critical | High | Medium>

**Probe** <number and name>
**Request ids** <ids the maintainer can look up in their own logs>

**Repro**
1. <step>
2. <step>
3. <step>

**Expected** <what should have happened>
**Actual** <what happened, with the numbers>
**Who loses money** <the operator or the user, and roughly how much per occurrence>

**Fix direction** <one or two sentences, offered not insisted on. Skip this if you are not sure.>

---

## Probe results

Every probe, including the ones that passed. A battery with gaps is not evidence of anything.

| # | Probe | Result | Note |
|---|---|---|---|
| 1 | Idempotency | pass / **fail** / n/a | |
| 2 | Charge on failure | | |
| 3 | Non-billable verbs | | |
| 4 | Retry amplification | | |
| 5 | Reported vs actual | | |
| 6 | Sub-unit rounding | | |
| 7 | Webhook signatures | | |
| 8 | Budget caps under parallelism | | |
| 9 | Revoked credentials | | |
| 10 | Reconciliation | | |

Use `n/a` where the feature does not exist, and say so in the note. "No idempotency keys offered"
is a finding, not a skipped probe.

## Not tested

<Anything the battery could not reach, and why. A credential you did not have, an endpoint that
needs a sales call, a webhook you could not trigger. This section is what makes the rest
credible.>
