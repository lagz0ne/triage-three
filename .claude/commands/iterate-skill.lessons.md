# iterate-skill lessons

## 2026-03-06: triage-three skill build

- Synthetic test fixtures (obvious bugs) won't differentiate prompt variants — all combos score perfectly. Use real-world code with CVEs for meaningful eval.
- "measured" pusher (accuracy over quantity) produces less noise and uses fewer tokens than "aggressive" without sacrificing recall on real bugs.
- The key differentiator between prompt variants is handling of DEBATABLE items, not clear-cut bugs. Design ground truth with a "debatable" category and lean direction.
- Ruthless challenger + conservative arbiter combo over-filters: excludes borderline-real findings. Balanced challenger preserves them.
- Paranoid/adversarial variants cost ~2x tokens and time with no recall benefit on tested fixtures.
- LLM-as-judge built into the pipeline provides useful self-assessment but adds latency. Consider making it optional.
- Eval JSON schema varies across agents — enforce a strict schema in the scorer prompt for consistent programmatic comparison.
- Ground truth from closed CVEs/issues is the gold standard for eval fixtures. Include exact file+line references.
- Background agents work well for parallel eval runs — 5 combos in parallel saves significant wall time.
- AskUserQuestion tool is better than inline questions for structured decision points during skill building.
