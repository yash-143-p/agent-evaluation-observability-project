# Perturbation Log

For each system, make one deliberate change to an input or configuration, predict the outcome, run it, and record what actually happened. See the starters in the Instructions, or design your own (your own experiment earns more credit).

---

### System 1 — validated, routed pipeline

- **Change I made (file + what I changed):**
  Created a copy of `data/policies/POL-2025-001.txt` at `/tmp/policy-premium-perturbation/POL-2025-001.txt` and blanked the `Total Policy Premium` value, changing `$ 1,847.62` to a blank value. The original policy file was left unchanged.

- **Command I ran:**
  `.venv/bin/python -m policy_extractor pipeline /tmp/policy-premium-perturbation --routing-out "/workspace/Project-Evaluation and Observability Project/01-policy-pipeline/premium-perturbed-routing-decisions.json" | tee "/workspace/Project-Evaluation and Observability Project/01-policy-pipeline/premium-perturbed-pipeline-run.txt"`

- **What I predicted:**
  Removing a field that the extractor normally returns should cause the validation/routing layer to reject the extraction and record the missing premium as a source/validation issue rather than inventing a premium value.

- **What actually happened:**
  `validation_failed`

  `decisions_written: 0`

  `human_review: 0`

  `escalations: 1`

  The pattern summary recorded `premium_amount_absent` for `POL-2025-001` with category `missing_source`.

- **How this differs from the unperturbed run:**
  The unperturbed `POL-2025-001` run produced `decision: human_review`, `fields_below_threshold: ["endorsements"]`, and `reviewer_disagreements: ["coverage_limit"]`. The premium confidence was `1.0`.

  With the premium value blanked, the perturbed run instead failed validation, wrote zero routing decisions, and recorded `premium_amount_absent` as a `missing_source` escalation. No replacement premium value was invented.

---

### System 2 — schema-enforced two-pass extraction

- **Change I made (file + what I changed):**
  Created my own copy of the mortgage income mismatch document at `/tmp/mortgage-perturbation/income_sum_mismatch_own_change.txt` and changed the stated monthly earnings from `10,892.17` to `10,492.17`. This was my own modification rather than using the bundled mismatch unchanged.

- **Command I ran:**
  Invoked the existing mortgage validator against the modified extraction values and saved the observed result to `02-mortgage-extraction/own-perturbation-run.txt`.

- **What I predicted:**
  Changing the stated total should change the validation delta while preserving the calculated component total, causing the mathematical consistency check to remain false.

- **What actually happened:**
  `consistent: False`

  `calculated: 9642.17`

  `stated: 10492.17`

  `delta: -850.0`

- **How this differs from the unperturbed run:**
  The original consistent extraction reported `consistent: true` with no discrepancy. The modified stated total produced a different validation delta of `-850.0` and remained inconsistent.

---

### System 3 — multi-source synthesis

- **Change I made (file + what I changed):**
  Simulated a timeout for the logistics source while investigating Meridian.

- **Command I ran:**
  `.venv/bin/supply-chain-investigate meridian --offline --simulate-timeout | tee "/workspace/Project-Evaluation and Observability Project/03-supply-chain/timeout-run.txt"`

- **What I predicted:**
  The investigation should finish even when the logistics source is unavailable, and information dependent on that source should be marked incomplete rather than treated as confirmed.

- **What actually happened (paste the key output line):**
  `Sources unavailable: logistics unavailable (timeout)`

  `late_shipment_count [missing source: timeout reading logistics]`

- **How this differs from the unperturbed run:**
  The normal run included logistics information and reported a contested `on_time_delivery_rate` of 95.0% versus 78.0% across sources. With the simulated timeout, the logistics-dependent `late_shipment_count` was marked incomplete and the investigation still finished.
