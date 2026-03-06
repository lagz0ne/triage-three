---
description: "Run prompt variation eval for triage-three against ground truth fixtures. Scores precision, recall, F1."
argument-hint: "[fixture-path] [combination-id]"
allowed-tools: ["Agent", "Read", "Glob", "Write"]
---
Run the triage-three prompt variation eval. Argument: $ARGUMENTS

## Overview

Run the triage-three pipeline across prompt variant combinations, score each against ground truth, and produce a comparison. Use this to find which prompt wording works best for your use case.

## Step 1: Load Eval Config

Read `eval/prompt-variants.json` for the variant matrix and combination list.
Read the fixture file specified in `$ARGUMENTS` (default: `eval/fixtures/buggy-ts.json`).
Read the source file referenced in the fixture.

If `$ARGUMENTS` specifies a single combination id (e.g., `baseline`), run only that one. Otherwise run all combinations in `combinations_to_test`.

## Step 2: Run Each Combination

For each combination, execute the triage pipeline sequentially using Agent subagents:

### Phase [I]: Pusher

```
You are [I] The Pusher — a bug finder.
{variant.mandate}
Depth level: Identify the most obvious and impactful items. Be concise.
Target material:
{source_code}
Output format:
{variant.output_format}
DO NOT use any tools. Analyze the code above and respond with findings only.
```

### Phase [II]: Challenger — feed [I]'s output

```
You are [II] The Challenger — a bug verifier.
{variant.mandate}
[I]'s findings:
{pusher_output}
Source material for verification:
{source_code}
Output format:
{variant.output_format}
DO NOT use any tools. Analyze and respond only.
```

### Phase [III]: Arbiter — feed both [I] and [II]

```
You are [III] The Arbiter — the truth maker.
{variant.mandate}
[I]'s findings:
{pusher_output}
[II]'s challenges:
{challenger_output}
Source material:
{source_code}
{variant.output_format}
DO NOT use any tools. Analyze and respond only.
```

### Phase [IV]: Scorer — feed [III]'s output + ground truth

```
You are a precision scorer. Compare triage results against ground truth.
Real bugs: {ground_truth.real_bugs as JSON}
Not bugs: {ground_truth.not_bugs as JSON}
Debatable: {ground_truth.debatable as JSON}
Triage output:
{arbiter_output}
For each real bug, check if triage found it (fuzzy match on function name + issue type).
For each not-bug, check if triage incorrectly flagged it.
For each debatable, note included or excluded.
Respond ONLY with JSON:
{"combination_id":"...","real_bugs_found":[],"real_bugs_missed":[],"false_positives":[],"debatable_included":[],"debatable_excluded":[],"extra_findings":[],"metrics":{"precision":0.0,"recall":0.0,"f1":0.0,"false_positive_rate":0.0}}
```

## Step 3: Store Results

Write each result to `eval/results/{combination_id}.json` with structure:
```json
{"combination_id":"...","fixture":"...","timestamp":"ISO-8601","variants_used":{"pusher":"...","challenger":"...","arbiter":"..."},"raw_outputs":{"pusher":"...","challenger":"...","arbiter":"..."},"scores":{}}
```

## Step 4: Summary

After all combinations complete, produce a comparison table ranked by F1 score.
Note which prompt variations correlate with higher recall, precision, and debatable handling.

## Parallelization

Each combination is independent. Launch up to 5 in parallel using background agents.

## Notes

- Each agent must NOT use tools — they analyze inline code only
- Results are append-only — re-running adds new files, doesn't overwrite
