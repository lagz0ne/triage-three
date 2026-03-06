Run the triage-three prompt variation eval. Argument: $ARGUMENTS

## Overview

This command runs the triage-three pipeline across prompt variant combinations, scores each against ground truth, and stores results as JSON. Use this to find which prompt wording produces the best results.

## Step 1: Load Eval Config

Read `eval/prompt-variants.json` for the variant matrix and combination list.
Read the fixture file specified in `$ARGUMENTS` (default: `eval/fixtures/buggy-ts.json`).
Read the source file referenced in the fixture.

If `$ARGUMENTS` specifies a single combination id (e.g., `baseline`), run only that one. Otherwise run all combinations in `combinations_to_test`.

## Step 2: Run Each Combination

For each combination, execute the triage pipeline sequentially:

### Phase [I]: Pusher

Use the variant's `mandate` and `output_format` in this template:

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

### Phase [II]: Challenger

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

### Phase [III]: Arbiter

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

### Phase [IV]: Scorer

After each combination, spawn a Scorer agent:

```
You are a precision scorer. Compare triage results against ground truth.

## Ground Truth
Real bugs: {ground_truth.real_bugs as JSON}
Not bugs: {ground_truth.not_bugs as JSON}
Debatable: {ground_truth.debatable as JSON}

## Triage Final Output (from Arbiter)
{arbiter_output}

## Scoring Rules

For each ground truth real bug, check if the triage output identified it (fuzzy match on function name + issue type is sufficient). For each not-bug, check if the triage incorrectly flagged it. For each debatable item, note if it was included or excluded.

Respond ONLY with this JSON:

{
  "combination_id": "{combination_id}",
  "real_bugs_found": ["list of ground truth bug ids that were found"],
  "real_bugs_missed": ["list of ground truth bug ids that were missed"],
  "false_positives": ["list of not-bug ids that were incorrectly flagged"],
  "debatable_included": ["list of debatable ids that were included"],
  "debatable_excluded": ["list of debatable ids that were excluded"],
  "extra_findings": ["findings not in ground truth at all"],
  "metrics": {
    "precision": 0.0,
    "recall": 0.0,
    "f1": 0.0,
    "false_positive_rate": 0.0
  }
}

Precision = real_bugs_found / (real_bugs_found + false_positives + extra_findings)
Recall = real_bugs_found / total_real_bugs
F1 = 2 * (precision * recall) / (precision + recall)
False positive rate = false_positives / total_not_bugs

Use counts, not lists, for the metric calculations.
```

## Step 3: Store Results

For each combination, write the full result to `eval/results/{combination_id}.json`:

```json
{
  "combination_id": "baseline",
  "fixture": "buggy-ts",
  "timestamp": "ISO-8601",
  "variants_used": {
    "pusher": "v1_aggressive",
    "challenger": "v1_ruthless",
    "arbiter": "v1_conservative"
  },
  "raw_outputs": {
    "pusher": "...",
    "challenger": "...",
    "arbiter": "..."
  },
  "scores": { ... from scorer ... }
}
```

## Step 4: Summary

After all combinations complete, produce a comparison table:

| Combination | Precision | Recall | F1 | FP Rate | Debatable Included |
|---|---|---|---|---|---|

Rank by F1 score. Highlight the best and worst performing combinations.

Note which prompt variations correlate with:
- Higher recall (finding more real bugs)
- Higher precision (fewer false positives)
- Better handling of debatable items

## Parallelization

Each combination is independent. Launch up to 3 combinations in parallel using background agents. Wait for all to complete before producing the summary.

## Notes

- Each agent must NOT use tools — they analyze inline code only
- The scorer must output valid JSON — retry once if it doesn't
- If a combination fails, log the error and continue with remaining combinations
- Results are append-only — running eval again adds new result files, doesn't overwrite
