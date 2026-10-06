# Domain-Independent Governance Vocabulary for Reasoning-Capable RPA

Reproducibility package for the paper *"A Domain-Independent Governance
Vocabulary for Reasoning-Capable Robotic Process Automation in
Manufacturing and Supply Chain Decision-Making."*

This repository contains the complete Answer Set Programming (ASP)
implementation of a two-layer RPA governance architecture, in which a
single, domain-independent declarative governance core is reused,
rather than reconstructed, across structurally unrelated decision
contexts through lightweight, interchangeable adapters, plus a
closed-loop computational demonstration composing all of it together.

## The three original domains

- **Equipment health monitoring and maintenance triage** — criticality
  graded via SAE JA1011 failure-consequence categories.
- **Logistics routing** — criticality graded via SLA/contract tier.
- **Supplier selection** — criticality graded via the Kraljic
  portfolio matrix.

## Two robustness-test domains, deliberately not designed alongside the vocabulary

- **Invoice / accounts payable exception handling** — stresses
  **criticality**: continuous (dollar amount) rather than categorical,
  with two independent derivation paths (amount-based and
  discrepancy-based) reconciled via an ASP `#max` aggregate.
- **Fraud / AML transaction monitoring** — stresses **confidence**:
  multiple, independent, disagreeing signal sources (rule engine, ML
  risk score, watchlist match) resolved via an ASP `#min` aggregate,
  the opposite aggregation logic from invoice's `#max`.

Invoice's adapter is markedly larger than the three original domains
(15 vs. 10 statements); fraud's is not (10 statements, identical).
This refines the theory of when adapter complexity grows: it scales
with the number of **independent derivation paths** an attribute
requires, not with the presence of aggregation or any single stressed
dimension in isolation.

## One adversarial test, which found and fixed a real defect

- **Security Operations Centre (SOC) alert triage** — targets the
  governance **core's rule structure** itself, not an adapter. The
  originally specified confidence gate escalated indiscriminately on
  low confidence regardless of priority, causing alert fatigue on
  trivially low-stakes signals. A minimal, principled fix (one new
  fact, one rule modified in place: `asp/governance_core.lp`,
  `asp/priority_matrix.lp`) was confirmed to generalise correctly
  across all five other domains. `asp/governance_core_original.lp`
  preserves the pre-fix core for direct comparison.

## Two closed-loop demonstrations, composing the whole architecture together

Every result above evaluates the governance core against independently
constructed events. `closed_loop_simulation.py` and
`closed_loop_simulation_v2.py` remove that isolation: a simulated
equipment asset's condition genuinely evolves over successive cycles,
using the **actual, unmodified** governance core and adapter, with a
governed decision at one cycle changing the asset's state entering the
next.

*Note: the paper presents this twelve-cycle trace both as a table and
as a plotted RUL-trajectory figure — both derived from the identical
`closed_loop_simulation.py` output, just presented in two formats.*

- **`closed_loop_simulation.py`** — a single asset (criticality 3),
  twelve cycles, both escalation paths (confidence-gate and
  priority-threshold) exercised via a scripted sensor fault, each
  routed to a mechanistically distinct downstream action (manual
  inspection vs. maintenance work order).
- **`closed_loop_simulation_v2.py`** — found, by independent
  verification, that the first asset's fixed criticality (3) has no
  Routine-tier cell in the priority matrix, so it can *never*
  distinguish the original core from the revised one, despite the
  adjacent prose inviting that connection. This version adds a
  **second asset** (criticality 2) whose matrix row does include
  Routine, and runs it against both `governance_core_original.lp` and
  `governance_core.lp`: the two runs diverge at exactly the cycle
  where the asset reaches Routine tier under low confidence, the
  defect and the fix, live, inside a running loop. It also reads
  escalation reasons directly from a new ASP predicate
  (`asp/governance_core_with_reasons.lp`, `escalate_reason/2`) rather
  than recomputing them in Python, which the original version did —
  a design that would have gone stale if the core's thresholds ever
  changed without the Python logic changing too.

**A genuine finding surfaced by v2, not designed in advance**: at the
diverging cycle, the *original* core's escalation has no recorded
reason at all — not a bug, but because its single, undifferentiated
confidence check has no analogue in the revised core's tier-conditioned
reason structure. The fix did not only correct an outcome; it
introduced a distinction the original core could not have expressed
even in principle.

**Open judgement call, not resolved in this repository**: whether the
two `escalate_reason/2` rules should count toward the paper's
adapter-to-core statement ratio (23 → 25 if counted). They are
additive, mirror `escalate/1`'s existing rule bodies exactly, and add
an explanation surface rather than new decision logic — but this is a
decision for the paper's authors, stated here rather than assumed.

## Architecture

- **Governance core** (`asp/governance_core.lp`, `asp/priority_matrix.lp`):
  a domain-independent, two-stage governance function — a
  criticality×urgency priority matrix (structurally analogous to an
  ITIL Impact–Urgency matrix) and an independent confidence gate
  (structurally analogous to FMEA's Detection dimension) — expressed
  over four abstract attributes (criticality, urgency, confidence,
  source), never fused into a single score. 23 statements in its
  final, post-adversarial-fix form.
- **Adapters** (`asp/adapter_*.lp`): lightweight, domain-specific
  mappings from each domain's raw signals into the shared governance
  vocabulary. No adapter modifies the governance core.

## Reproducing the paper's results

```bash
pip install clingo --break-system-packages   # or your preferred install method
python3 python/run_all.py                    # runs all nine scripts in sequence
```

Or run each individually:

```bash
python3 python/run_correctness.py            # Governance rule-conformance (3 domains)
python3 python/run_audit_completeness.py     # Audit-trail completeness (3 domains)
python3 python/run_size_ratio.py             # Adapter-to-core-model size ratio (3 domains)
python3 python/run_maintainability.py        # Rule-level maintainability under policy change
python3 python/run_robustness_test.py        # Robustness test 1: invoice domain (criticality)
python3 python/run_fraud_stress_test.py      # Robustness test 2: fraud domain (confidence)
python3 python/run_soc_adversarial_test.py   # Adversarial test: SOC domain (core defect + fix)
python3 python/closed_loop_simulation.py     # Closed-loop demo 1: single asset, both escalation paths
python3 python/closed_loop_simulation_v2.py  # Closed-loop demo 2: two assets, live defect/fix divergence
```

All scripts must be run from the repository root, since ASP file
paths are relative to it.

## What each script verifies

| Script | Reproduces | Expected result |
|---|---|---|
| `run_correctness.py` | Every governance decision, across the three original domains, matches an independently implemented (non-ASP) verification method | 7/7 events match per domain, zero discrepancies |
| `run_audit_completeness.py` | Every decision retains a complete justification (attributes, tier, outcome) | 7/7 complete audit records per domain |
| `run_size_ratio.py` | Each adapter's statement count relative to the shared core | 0.435, identical across all three domains |
| `run_maintainability.py` | A policy change made once, to the shared core only, takes effect identically across all domains with zero adapter modification | Confirmed; adapter files byte-for-byte unchanged |
| `run_robustness_test.py` | The invoice domain (criticality-stressed), including two events designed to stress the `#max` aggregate-resolution logic | 6/6 correct and audit-complete; adapter ratio **0.652** — elevated |
| `run_fraud_stress_test.py` | The fraud domain (confidence-stressed), including events with 2 and 3 simultaneously disagreeing signals resolved via `#min` | 7/7 correct and audit-complete; adapter ratio **0.435** — *not* elevated |
| `run_soc_adversarial_test.py` | The SOC domain, demonstrating the original core's defect, the fix, and zero regression across all five other domains | Defect confirmed; fix confirmed correct and non-regressive |
| `closed_loop_simulation.py` | Twelve-cycle single-asset loop exercising both escalation paths via a scripted sensor fault | 4 maintenance actions, 1 manual inspection, 7 autonomous cycles; fully deterministic |
| `closed_loop_simulation_v2.py` | Two-asset loop with ASP-native escalation reasons; second asset run against both core versions | Asset 1: identical under both cores (expected — never reaches Routine); Asset 2: diverges at exactly 1 cycle (defect vs. fix, live) |

## Independent verification methodology

Every result in this repository is cross-checked against a second,
separately implemented computation (`python/independent_verify.py`,
`python/adapter_mappings.py`, and inline reimplementations within the
robustness/adversarial test scripts), which shares no code with the
ASP encoding. The closed-loop scripts were themselves subject to
independent verification after initial development, which is how the
`closed_loop_simulation.py` → `closed_loop_simulation_v2.py` revision
came about: a structural limitation (asset 1 cannot reach Routine
tier) and a design inconsistency (Python-side reason recomputation
contradicting the paper's own explanation claim) were found by
re-deriving the priority matrix independently and running the
original script against both core versions, not by inspection alone.
No reported result in this paper is accepted on the strength of a
single implementation, or a single round of verification, alone.

## Constructed-baseline benchmark

`run_constructed_baseline.py` compares this paper's actual shared-core
architecture against a domain-specific rule architecture built in the
general style of published rule-based systems in the RPA-adoption
literature (Leshob et al., 2024) and computer-integrated manufacturing
(Järvenpää et al., 2023; Sadigh et al., 2016 / OMAVE). **This does not
reproduce any of these papers' actual rule bases or results** — neither
is available at the level of detail needed to reproduce faithfully.
The domain-specific baseline is constructed by inlining this repository's
own adapter and governance-core logic into three self-contained,
per-domain files, verified to produce identical decisions to the real
architecture before any comparison is drawn.

Four scenarios are tested, each as a real file edit followed by
re-verification, not an estimate: tightening the escalation threshold,
changing a priority-matrix cell, adding a new escalation condition, and
adding a fourth domain. In every scenario the domain-specific baseline
requires editing as many files as there are domains (3); the shared-core
architecture requires editing exactly 1. Two results are reported
without qualification because they do **not** favour this paper's
architecture: decision correctness and audit-trail completeness are
identical between both approaches throughout, since the baseline was
constructed to inline the same logic rather than a weaker approximation
of it.

*Note: the paper also plots scenarios A–C's file-modification counts as
a grouped bar chart, in addition to the full table — Scenario D is
deliberately excluded from that chart (new files created, not existing
files modified, a structurally different comparison), though it remains
in the table and in this script's output.*

## Repository structure

```
asp/
  governance_core.lp             — domain-independent governance rules (post-adversarial-fix, final)
  governance_core_original.lp    — pre-adversarial-fix core, preserved for comparison
  governance_core_with_reasons.lp — additive escalate_reason/2 rules (not in paper's Listing 1)
  priority_matrix.lp             — the 4×4 priority matrix and policy thresholds
  adapter_equipment.lp           — equipment health / maintenance triage adapter
  adapter_logistics.lp           — logistics routing adapter
  adapter_supplier.lp            — supplier selection adapter
  adapter_invoice.lp             — invoice/AP exception adapter (robustness test 1)
  adapter_fraud.lp               — fraud/AML transaction adapter (robustness test 2)
  adapter_soc.lp                 — SOC alert triage adapter (adversarial test)
  naive_domain_specific_equipment.lp — Approach A: equipment (constructed baseline)
  naive_domain_specific_logistics.lp — Approach A: logistics (constructed baseline)
  naive_domain_specific_supplier.lp  — Approach A: supplier (constructed baseline)
data/
  equipment_events.lp      — test events for the equipment-health domain
  logistics_events.lp      — test events for the logistics domain
  supplier_events.lp       — test events for the supplier-selection domain
  invoice_events.lp        — test events for the invoice robustness test
  fraud_events.lp          — test events for the fraud robustness test
  soc_events.lp            — test events for the SOC adversarial test
python/
  independent_verify.py         — second, independent governance-logic implementation
  adapter_mappings.py            — second, independent adapter-mapping implementation
  asp_runner.py                  — clingo wrapper shared by all evaluation scripts
  run_correctness.py              — Metric 1 (3 original domains)
  run_audit_completeness.py       — Metric 2 (3 original domains)
  run_size_ratio.py                — Metric 3 (3 original domains)
  run_maintainability.py           — Metric 4
  run_robustness_test.py           — Robustness test 1: invoice (criticality)
  run_fraud_stress_test.py         — Robustness test 2: fraud (confidence)
  run_soc_adversarial_test.py      — Adversarial test: SOC (core defect + fix + regression check)
  closed_loop_simulation.py        — Closed-loop demo 1: single asset
  closed_loop_simulation_v2.py     — Closed-loop demo 2: two assets, live defect/fix divergence
  run_constructed_baseline.py      — Constructed-baseline benchmark: 4 scenarios, domain-specific vs shared core
  run_all.py                        — runs all ten in sequence
```

## Requirements

- Python 3.10+
- `clingo` 5.8+ (Python bindings; no separate CLI installation required)

## Citation

See `CITATION.cff`.

## License

MIT — see `LICENSE`.
