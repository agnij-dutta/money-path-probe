# Probe reference

One section per probe: the setup, the exact reproduction, what correct behaviour looks like, the
common wrong implementations, and who loses money.

---

## 1. Idempotency

**Setup.** A write endpoint that accepts an idempotency key. If the API has no such concept, that
is itself the finding: record it and move to probe 4.

**Repro.** Send the identical request body twice with the identical key, a few seconds apart.
Then send it a third time with the same key and a *different* body.

**Correct.** The second call returns the first call's stored response and creates no new charge.
The third call is rejected, because reusing a key with a different payload is a client bug the
server should surface rather than absorb.

**Common wrong implementations.**
- The key is checked but the record is written after the charge, so two concurrent calls both pass.
- The key is scoped globally rather than per account, which leaks one tenant's response to another.
- The key expires in minutes, so a retry after a long outage double-charges.
- The third call succeeds and silently returns the *first* response, so the caller believes a
  different request was processed.

**Who loses.** The user, twice over: charged twice and holding a receipt for one.

---

## 2. Charge on failure

**Setup.** Any endpoint where authorisation and fulfilment are separate steps.

**Repro.** Cause a failure in the window after authorisation and before the work is delivered.
Easiest levers: request a resource that does not exist, send a payload that fails downstream
validation, or ask for an output the backend cannot produce.

**Correct.** No charge, or an authorisation that is explicitly voided. If the system charges for
attempted work by design, the documentation must say so and the response must state it.

**Common wrong implementations.**
- The meter increments in a request middleware rather than on a successful response.
- The charge is captured on authorisation and the void path exists but is never called on the
  error branch.
- A `5xx` from the provider's own infrastructure is billed.

**Who loses.** The user pays for nothing. This is the single most common complaint about metered
APIs and the easiest to confirm.

---

## 3. Non-billable verbs

**Setup.** A metered HTTP endpoint behind payment or usage middleware.

**Repro.** Three requests: an `OPTIONS` preflight, a `HEAD`, and a conditional `GET` with an
`If-None-Match` that will return `304 Not Modified`. Read the meter after each.

**Correct.** None of the three is billed. `OPTIONS` is a browser preflight the user agent sends
without being asked. `HEAD` returns no body. `304` returns no body and is the mechanism by which
a well-behaved client *avoids* consuming the resource.

**Common wrong implementations.** This is `x402-foundation/x402#3738`. The middleware looks for a
payment header, finds none on the preflight because a preflight cannot carry one, and treats the
absence as a new payment required. The browser then pays for the privilege of asking whether it is
allowed to ask.

**Who loses.** The user, and silently, because preflights are invisible in application code. One
browser call can meter as two.

---

## 4. Retry amplification

**Setup.** Any charged endpoint. A client you can interrupt mid-flight.

**Repro.** Start the request, terminate the connection before the response arrives, then retry the
same request. Compare the charge count to one.

**Correct.** One charge. The server completed the work once; the client simply did not hear about it.

**Common wrong implementations.**
- No idempotency on a path the client is expected to retry.
- A client SDK that retries automatically on timeout without attaching a key, so the defect is in
  the SDK and the server is blameless.
- A gateway timeout shorter than the handler, so the gateway retries into a still-running handler.

**Who loses.** The user, and at a multiple: an automatic retry policy turns one interrupted call
into three charges.

---

## 5. Reported vs actual

**Setup.** An API whose response states the amount charged. Skip if it does not.

**Repro.** Run a known sequence of calls, sum the amounts the responses reported, and compare to
the dashboard or balance delta over the same window.

**Correct.** They agree exactly, or any difference is itemised somewhere the customer can see.

**Common wrong implementations.**
- The response reports a list price while the account is on a discount, so the number is right in
  neither direction.
- The response reports the amount *reserved*, the invoice shows the amount *settled*, and nothing
  shows the difference.
- The response rounds for display at a different precision than the ledger stores.

**Who loses.** Nobody immediately, which is why it survives. It becomes a dispute later, and at
that point neither number can be trusted and the operator loses the argument.

---

## 6. Sub-unit rounding

**Setup.** Any priced unit with a minimum, which is most per-token, per-second, and per-byte pricing.

**Repro.** Submit the smallest possible unit of work, one well below the minimum billable amount.
Then submit a thousand of them.

**Correct.** Documented and consistent. Flooring to zero and rounding up to the minimum are both
defensible. Doing one at the call level and the other at the invoice level is not.

**Common wrong implementations.**
- Each call rounds up to the minimum, so a thousand tiny calls cost a thousand minimums while the
  usage total shows almost nothing.
- Each call floors to zero, so a thousand tiny calls are free and the operator bleeds.
- The call floors and the invoice rounds up, so they disagree by construction, which also fails
  probe 10.

**Who loses.** Either side, depending on the direction. Say which.

---

## 7. Webhook signatures

**Setup.** A webhook you received on your own account, captured with its full headers.

**Repro.** Replay the captured payload to your own endpoint, unmodified, well outside whatever
tolerance window the provider documents. Then replay it again with the body altered by one
character and the original signature.

**Correct.** The stale replay is rejected on the timestamp. The altered body is rejected on the
signature. A signature scheme that signs the body but not the timestamp is only half a scheme,
because a valid signature stays valid forever.

**Common wrong implementations.**
- The timestamp is parsed and never compared.
- The tolerance is measured but the comparison is the wrong way round, so only *future*
  timestamps are rejected.
- Signature comparison uses `==` rather than a constant-time compare.
- The handler is idempotent on its own id, which masks the problem until the id is also replayed
  from a different event.

**Who loses.** The operator. A replayable "payment succeeded" webhook is free goods.

**Run this one carefully.** Replay only a webhook addressed to you. Confirm you are hitting your
own endpoint before you send anything.

---

## 8. Budget caps under parallelism

**Setup.** A hard spend cap, budget, or quota you can set low. Set it to ten units.

**Repro.** Fire twelve requests of one unit each, concurrently, not in sequence. Count how many
were accepted and what the final committed total is.

**Correct.** Ten accepted, two refused, total exactly ten.

**Common wrong implementations.** Read the current spend, compare to the cap, write the new spend.
Under concurrency every request reads the same pre-update value and every one of them passes.
This is a read-modify-write that needs a lock or a compare-and-swap, and in a serverless runtime
the instances do not share memory, so an in-process mutex looks correct in local testing and does
nothing in production.

**What makes this probe wrong.** Sequential requests will pass. If you send them in a loop with
`await`, you have tested nothing. The requests must be in flight at the same time.

**Who loses.** The operator, and the customer who set a cap specifically so this could not happen.
A cap that does not hold is worse than no cap, because it was relied upon.

---

## 9. Revoked credentials

**Setup.** A key, token, or mandate you can revoke.

**Repro.** Make a successful call. Revoke the credential. Immediately make the same call again,
then again after thirty seconds, then after five minutes.

**Correct.** The first call after revocation is refused. If the system documents an eventual
consistency window, the window must be stated and short.

**Common wrong implementations.**
- The auth layer caches a validated key for its TTL, so a revoked key keeps working for minutes.
- Revocation writes to the primary and the auth path reads a replica.
- An established session or long-lived connection is never rechecked.
- Revocation is recorded locally and never propagated to the edge.

**Who loses.** The operator and the customer. Revocation is the control you reach for *during* an
incident, so a delay here is a delay in stopping an active loss.

---

## 10. Reconciliation

**Setup.** A full billing period with known activity, or a window you can bound precisely.

**Repro.** Export every per-call charge for the window. Sum them. Compare to the invoice total for
the same window, to the account balance delta, and to the usage dashboard.

**Correct.** All four agree, or each difference has a named line item: tax, a credit, a minimum
commitment, a rounding policy.

**Common wrong implementations.**
- Per-call records and the invoice are computed from different sources with different rounding.
- The export is paginated and the last page is quietly missing.
- Reserved-but-never-committed amounts appear in one view and not the other.
- Timezone boundaries differ between the export and the billing period.

**Who loses.** Whoever cannot prove their number. Usually the customer, because the operator owns
the ledger.

**This probe is the one worth running last** and the one most likely to find something, because
it is the only probe that checks the system against itself across its whole surface rather than
one endpoint at a time.
