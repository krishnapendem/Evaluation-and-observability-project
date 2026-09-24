# Perturbation Log

For each system, make one deliberate change to an input or configuration, predict the outcome, run
it, and record what actually happened. See the starters in the Instructions, or design your own (your
own experiment earns more credit).

---

### System 1 — validated, routed pipeline

- **Change I made (file + what I changed):** Blanked a required field in a copied policy fixture (`coverage_limit` set to empty/absent) before running the extraction path.
- **Command I ran:** `.venv/bin/python -c "from policy_extractor.retry import extract_with_retry; ..."` (synthetic missing-field validation) / local policy fixture check.
- **What I predicted:** The extractor would flag the record as a futile retry case and escalate instead of fabricating a value.
- **What actually happened (paste the key output line):** `{"kind":"escalation","policy_id":"POL-EMPTY","field":"coverage_limit","reason":"missing required field","retries_used":1,"status":"retry_futile_escalation"}`
- **How this differs from the unperturbed run:** The normal run tries to keep extracting; the perturbed run does not fabricate a value and instead escalates immediately.

---

### System 2 — schema-enforced two-pass extraction

- **Change I made (file + what I changed):** Edited the cash-flow fixture to make `stated_monthly_total` disagree with the component sum in `fixtures/documents/income_sum_mismatch.txt`.
- **Command I ran:** `mortgage-extract fixtures/documents/income_sum_mismatch.txt`
- **What I predicted:** The validator would reject the extraction as inconsistent because the stated total would no longer match the calculated monthly sum.
- **What actually happened (paste the key output line):** `"field": "total_monthly_income", "calculated": 9642.17, "stated": 10892.17, "delta": -1250.0` and `"consistent": false`.
- **How this differs from the unperturbed run:** In the clean run, `validation.consistent` is `true` and `discrepancies` is empty; in the perturbed run, the validator catches the mismatch and blocks a bad extraction.

---

### System 3 — multi-source synthesis

- **Change I made (file + what I changed):** Ran the offline investigation with the logistics reader forced into a timeout path via `--simulate-timeout`.
- **Command I ran:** `supply-chain-investigate meridian --offline --simulate-timeout`
- **What I predicted:** The logistics source would be marked as unavailable and the briefing would still complete with an `Incomplete` section instead of crashing.
- **What actually happened (paste the key output line):** `Sources unavailable: logistics unavailable (timeout)` and `late_shipment_count _[missing source: timeout reading logistics]`.
- **How this differs from the unperturbed run:** The normal run has a `Contested` section for `on_time_delivery_rate`; the timeout run removes logistics from the active evidence set and replaces it with an explicit missing-source annotation while continuing the briefing.
