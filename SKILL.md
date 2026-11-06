---
name: money-path-probe
description: Audit any billing, metering, or payment integration for the ten ways it silently charges the wrong amount. Use when testing a payments API, a usage meter, an agent spend cap, a webhook handler, or a subscription flow, and when asked to review money-handling code before launch. Produces a findings report that states who loses money on each defect.
license: MIT
---

# Money-path probe

Most billing bugs are not arithmetic. They are a request counted twice, a charge that fires
before fulfilment, or a cap that holds in sequence and fails in parallel. None of them show up
in a unit test, because each one needs a *second* request to expose it.

This skill is a fixed battery of ten probes against anything that moves money or meters usage.
Each probe is one question with a yes or no answer and a one-line reproduction. Run them
deliberately, in order, and record the answers.

## When to use this

- Before launching a metered or paid API.
- When integrating a payment provider, an agent spend cap, or a usage meter.
- When reviewing money-handling code: look for the defect class, then prove it with the probe.
- When invoices and per-call charges disagree and nobody knows which is wrong.

## The rule that makes the report worth reading

For every defect, record **who loses money when it is wrong, the operator or the user.**

A finding without that sentence is a curiosity. With it, it is a priority. "Retries double-bill,
the user loses" is a refund queue. "Failed calls are not metered, you lose" is revenue leakage.
Both matter; they go to different people and get fixed on different timelines.

## How to run it

1. **Get a sandbox and a baseline.** Use the provider's test mode and test keys. Never probe a
   production account or anyone else's account. Record the starting balance, meter reading, or
   invoice total before you touch anything, because half the probes are a delta measurement.
2. **One probe at a time.** Each probe changes state. Running two at once makes the result
   unattributable. Re-read the meter between probes.
3. **Prefer the smallest unit of money the system accepts.** You want many observations cheaply,
   and probe 6 specifically needs an amount below the minimum billable unit.
4. **Log the request id.** Every finding needs one for the maintainer to look it up. A finding
   they cannot locate in their own logs gets closed.
5. **Fill in the report template** at `assets/report-template.md` as you go, not afterwards.

Full reproduction detail for each probe, including what correct behaviour looks like and the
common wrong implementations, is in `references/probes.md`. Read it before running probe 7 or 8,
which are the two that are easy to run incorrectly and conclude the wrong thing.

## The ten probes

| # | Probe | The question | Repro in one line |
|---|---|---|---|
| 1 | Idempotency | Charged once or twice? | Fire the same request twice with the same idempotency key |
| 2 | Charge on failure | Does the meter still tick? | Make the call fail after auth but before fulfilment |
| 3 | Non-billable verbs | Any of them billed? | Send `OPTIONS`, `HEAD`, and a request that returns `304` |
| 4 | Retry amplification | One charge or two? | Kill the connection mid-flight, then retry |
| 5 | Reported vs actual | Do they agree? | Reconcile the amount the response reports against the dashboard total |
| 6 | Sub-unit rounding | Floored, rounded up, or dropped? | Submit an amount below the smallest billing unit |
| 7 | Webhook signatures | Accepted? | Replay a captured webhook with a stale timestamp |
| 8 | Budget caps | Does the cap hold under parallelism? | Set a hard cap, then try to exceed it concurrently |
| 9 | Revoked credentials | Does a cache keep it alive? | Revoke a key, then immediately reuse it |
| 10 | Reconciliation | Does the sum equal the invoice? | Add up every per-call charge and compare to the bill |

### Why these ten

They are the defect classes that survive code review. Probes 1, 4 and 8 are all the same
underlying mistake, a read-modify-write that is not atomic, showing up in three places where
teams fix one and miss the others. Probes 2, 3 and 6 are boundary errors in deciding *whether*
to charge, which is a decision most codebases make in a middleware nobody owns. Probes 5 and 10
are the only ones that catch a system that is internally consistent and still wrong. Probes 7 and
9 are the two places where a stale cache becomes a payment.

Probe 3 is drawn from a real fix: a CORS preflight was billed as a payment
(`x402-foundation/x402#3738`). An `OPTIONS` request carries no payment header by definition, so
the middleware treated the absence as a fresh charge. Nothing in the test suite sent an `OPTIONS`.

### Severity, consistently applied

- **Critical**: money moves that should not, or a stated control does not hold. Probes 1, 4, 7, 8, 9.
- **High**: the amount is wrong in a way that compounds per request. Probes 2, 5, 6.
- **Medium**: the amount is right but unverifiable by the customer. Probe 10 on its own.

Do not inflate. A maintainer who finds one overstated severity in your report stops reading it.

## What to do with the findings

Report privately first, through the provider's security contact or private vulnerability
reporting, for anything in the Critical list. Probes 1, 4, 7, 8 and 9 can be abused by a third
party, so they are vulnerability reports, not bug reports, and they do not go in a public issue.

Probes 2, 3, 5, 6 and 10 are ordinary correctness bugs. A public issue with the request id and
the one-line repro is usually the fastest path to a fix.

## Boundaries

- Test mode and your own account only. These probes charge money and create load.
- Probe 8 is a concurrency probe, not a load test. Twelve parallel requests is enough. Do not
  point it at a production endpoint, and do not raise the count to find a rate limit.
- Probe 7 replays a webhook you captured from your own account. Never replay one you did not
  originate.
- If a probe needs a credential you were not given, stop and ask. Do not work around an auth
  boundary to finish the battery.
