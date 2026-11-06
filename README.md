# money-path-probe

A fixed battery of ten probes for billing, metering, and payment integrations. Each probe is one
question with a yes or no answer and a one-line reproduction, aimed at the defect classes that
survive code review because they need a *second* request to expose them: a retry that double
charges, a charge that fires before fulfilment, a spend cap that holds in sequence and fails in
parallel.

Packaged as an agent skill. `SKILL.md` is the workflow, `references/probes.md` is the per-probe
detail, `assets/report-template.md` is the output format.

Every finding records **who loses money when it is wrong, the operator or the user.** That
sentence is what separates a curiosity from a priority.

Probe 3 comes from a real fix upstream: a CORS preflight billed as a payment
(`x402-foundation/x402#3738`). An `OPTIONS` request cannot carry a payment header, so the
middleware read its absence as a fresh charge.

Test mode and your own accounts only. See the Boundaries section in `SKILL.md`.

MIT.
