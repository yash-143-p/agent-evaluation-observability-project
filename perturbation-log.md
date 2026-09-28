# Perturbation Log

For each system, make one deliberate change to an input or configuration, predict the outcome, run it, and record what actually happened. See the starters in the Instructions, or design your own (your own experiment earns more credit).

---

### System 1 — validated, routed pipeline

- **Change I made (file + what I changed):**
  Created a copy of `data/policies/POL-2025-001.txt` at `/tmp/POL-2025-001-perturbed.txt` and blanked the required `Named Insured` field. The original policy file was left unchanged.

- **Command I ran:**
  `.venv/bin/policy-extractor pipeline /tmp/policy-perturbed --routing-out "/workspace/Project-Evaluation and Observability Project/01-policy-pipeline/perturbed-routing-decisions.json" --spot-check-pct 0 --seed 42 | tee "/workspace/Project-Evaluation and Observability Project/01-policy-pipeline/perturbation-run.txt"`

- **What I predicted:**
  The missing required `Named Insured` field should be detected by the validated extraction/routing pipeline and should result in escalation rather than an invented insured name.

- **What actually happened (paste the key output line):**
  `TypeError: "Could not resolve authentication method. Expected one of api_key, auth_token, or credentials to be set"`

- **How this differs from the unperturbed run:**
  The perturbed run could not reach extraction or routing because Anthropic API authentication was unavailable. Therefore, no routing decision was produced and the predicted escalation could not be verified.

---

### System 2 — schema-enforced two-pass extraction

- **Change I made (file + what I changed):**
  Used `fixtures/documents/income_sum_mismatch.txt`, where the stated total monthly income does not match the sum of the individual income components.

- **Command I ran:**
  `.venv/bin/mortgage-extract fixtures/documents/income_sum_mismatch.txt --mode replay | tee "/workspace/Project-Evaluation and Observability Project/02-mortgage-extraction/discrepancy-run.txt"`

- **What I predicted:**
  The validation stage should detect the mathematical inconsistency instead of silently accepting the stated total.

- **What actually happened (paste the key output line):**
  `consistent: false`
  
  `calculated: 9642.17`
  
  `stated: 10892.17`
  
  `delta: -1250.0`

- **How this differs from the unperturbed run:**
  The unperturbed extraction was consistent and reported `consistent: true` with no discrepancies. The perturbed input produced a validation discrepancy because the stated total exceeded the calculated total by $1,250.00.

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

### System 1 — final observed result

The final live rerun succeeded after aligning the reference implementation's model names with the models available through the Vocareum gateway. The perturbed policy produced one routing decision.

Observed result:
- `policy_id`: `POL-2025-001`
- `decision`: `human_review`
- `reason`: `reviewer_disagreement=['coverage_limit']`
- `reviewer_disagreements`: `["coverage_limit"]`
- `fields_below_threshold`: `[]`
- `decisions_written`: `1`
- `human_review`: `1`
- `auto_approve`: `0`
- `spot_check`: `0`
- `escalations`: `0`

This contrasts with the intended unperturbed behavior by showing that the perturbed input reached human review rather than being silently accepted. The independent reviewer disagreement on `coverage_limit` was preserved as the explicit routing signal. The output does not show `Named Insured` as a routing trigger, so I do not attribute the human-review decision directly to that missing field.
