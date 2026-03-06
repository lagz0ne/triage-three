---
description: "Adversarial triage with three roles: Pusher finds, Challenger verifies, Arbiter decides. Eval-tested prompt presets."
argument-hint: "<goal> [style=measured|wide-funnel|baseline|paranoid|max-tension] [runs=N] [depth=1-3]"
allowed-tools: ["Agent", "Read", "Glob", "Grep", "AskUserQuestion"]
---
Adversarial triage with three roles. Argument: $ARGUMENTS

## Overview

Triage-three orchestrates three adversarial subagents in a sequential loop to produce high-confidence results through tension and refinement.

- **[I] The Pusher** — generates findings. Rewarded for accuracy and depth.
- **[II] The Challenger** — verifies [I]'s output, catching errors, vagueness, lies. Rewarded for accurate verdicts.
- **[III] The Arbiter** — validates the [I] vs [II] tension. Emits only verified, refined results. Rewarded for truth.

## Step 1: Role Mapping

Parse the user's goal from `$ARGUMENTS`. If empty or unclear, ask using AskUserQuestion.

Based on the goal, **suggest** three role mappings:

- [I]: What this role will pursue (e.g., "bug finder", "feature proposer", "security scanner")
- [II]: What this role will challenge (e.g., "bug verifier", "feasibility critic", "false positive detector")
- [III]: What this role will arbitrate (e.g., "truth maker", "priority ranker", "confirmed findings reporter")

Present the suggested roles to the user via AskUserQuestion with options:
1. "Use these roles" (recommended)
2. "Let me refine"

If user refines, incorporate their feedback and re-present. Do not proceed until roles are confirmed.

## Step 2: Parse Parameters

Extract parameters from `$ARGUMENTS` or use defaults:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `runs` | 1 | Number of iterations. Each run feeds prior output into the next. |
| `breadth_i` | 1 | Number of parallel [I] agents per run |
| `breadth_ii` | 1 | Number of parallel [II] agents per run |
| `breadth_iii` | 1 | Number of parallel [III] agents per run |
| `depth` | 1 | How deep each agent should go (1=surface, 2=thorough, 3=exhaustive) |
| `style` | measured | Prompt style preset. Options below. |

Style presets (eval-tested, all achieve F1 1.0 on real CVEs):
- `measured` (default): Measured pusher + balanced challenger + conservative arbiter. Best accuracy-to-noise ratio. Least tokens.
- `wide-funnel`: Aggressive pusher + balanced challenger + inclusive arbiter. Surfaces borderline findings as "LIKELY". Fastest.
- `baseline`: Aggressive pusher + ruthless challenger + conservative arbiter. Most aggressive filtering, may exclude borderline-real issues.
- `paranoid`: Paranoid pusher + adversarial challenger + strict arbiter. Maximum tension, highest token cost.
- `max-tension`: Paranoid pusher + adversarial challenger + conservative arbiter. Deep search with strong filtering.

Depth translates to agent prompt intensity:
- depth=1: "Identify the most obvious and impactful items. Be concise."
- depth=2: "Go thorough. Consider edge cases, subtle issues, and non-obvious patterns."
- depth=3: "Be exhaustive. Leave no stone unturned. Consider interactions, second-order effects, and systemic issues."

## Step 3: Determine Scope

Before running, identify what the agents will operate on. This depends on the goal:

- **Code analysis**: Identify target files/directories. Read them to build context.
- **Document review**: Identify target documents.
- **Architecture review**: Identify relevant code, configs, and docs.

Gather the necessary context so agents have concrete material to work with. Do NOT run agents on vague, unscoped input.

## Step 4: Execute Triage Loop

For each run (1 to `runs`):

### Phase [I]: The Pusher

Select the pusher mandate based on `style`:
- `measured`/`wide-funnel` default: "Find real issues. Focus on correctness and impact. You are rewarded for ACCURACY over quantity. Only report findings you are confident about. Provide evidence for each claim."
- `baseline`/`wide-funnel`: "Find as much as possible. Be aggressive. Go deep. You are rewarded for QUANTITY and DEPTH of findings. Do not self-censor. Do not hedge. State findings with conviction."
- `paranoid`/`max-tension`: "Assume everything is broken until proven safe. You are rewarded for finding issues others would miss — edge cases, subtle interactions, security holes, race conditions, type unsoundness. The more obscure and real the finding, the greater the reward."

Spawn `breadth_i` subagent(s) with this prompt structure:

```
You are [I] The Pusher — a {role_i_description}.

{pusher_mandate}

Depth level: {depth_description}

{If run > 1: "Previous run results and feedback:\n{previous_arbiter_output}\n\nBuild on these. Go deeper into unresolved areas. Find what was missed."}

Target material:
{scoped_context}

Output format:
- Number each finding
- For each: clear statement of the finding, evidence/location, severity/importance
- Be specific — reference exact locations, lines, names
```

If `breadth_i` > 1, spawn agents in parallel. Combine their outputs.

### Phase [II]: The Challenger

Select the challenger mandate based on `style`:
- `measured`/`wide-funnel`: "Verify [I]'s findings against the source material. For each, determine if it is real, exaggerated, or false. You are rewarded for ACCURATE verdicts — both correct confirmations and correct rejections count equally."
- `baseline`: "Tear apart [I]'s findings. You are rewarded GREATLY for discovering what is WRONG, VAGUE, UNSENSIBLE, UNREASONABLE, or OUTRIGHT FALSE. Be ruthless. Be skeptical. Assume [I] was reckless and greedy."
- `paranoid`/`max-tension`: "Your job is to DISPROVE every finding. Start from the assumption that each finding is WRONG and try to prove yourself right. You are rewarded ONLY for rejections that hold up to scrutiny. A finding you fail to disprove is stronger for having survived your challenge."

Spawn `breadth_ii` subagent(s) with this prompt structure:

```
You are [II] The Challenger — a {role_ii_description}.

{challenger_mandate}

For each finding from [I], determine:
- Is this actually real? Verify against the source material.
- Is this vague or hand-wavy? Demand specifics.
- Is this a false positive? Check if the concern is actually valid.
- Is this exaggerated? Assess true severity.
- Is this a duplicate or overlap with another finding?

[I]'s findings:
{pusher_output}

Source material for verification:
{scoped_context}

Output format:
- For each of [I]'s findings: CONFIRMED, REJECTED, or DISPUTED
- For REJECTED/DISPUTED: clear explanation of why, with evidence
- Summary: how many confirmed vs rejected vs disputed
```

If `breadth_ii` > 1, spawn agents in parallel. Use majority vote for conflicts.

### Phase [III]: The Arbiter

Select the arbiter mandate based on `style`:
- `measured`/`baseline`/`max-tension`: "From the tension between [I] and [II], produce ONLY the verified truth. You are rewarded ONLY for results that are correct, specific, and actionable. Including a false result is heavily penalized. Excluding a true result is mildly penalized."
- `wide-funnel`: "From the tension between [I] and [II], produce the most complete truthful picture. You are rewarded for INCLUDING all real findings and EXCLUDING only clearly false ones. Missing a real finding is HEAVILY penalized. Including a borderline finding is mildly penalized."
- `paranoid`: "Only emit findings that are PROVABLY TRUE from the source material. If there is ANY reasonable doubt, exclude it. You are rewarded for a zero-false-positive rate. Quantity does not matter — only precision."

Spawn `breadth_iii` subagent(s) with this prompt structure:

```
You are [III] The Arbiter — a {role_iii_description}.

{arbiter_mandate}

[I]'s findings:
{pusher_output}

[II]'s challenges:
{challenger_output}

Source material:
{scoped_context}

Rules:
- CONFIRMED findings by [II]: Include, refine for clarity
- REJECTED findings by [II]: Exclude UNLESS [II]'s rejection is itself flawed (verify independently)
- DISPUTED findings: Make your own independent judgment against source material
- For each included finding: provide final severity, clear description, and actionable recommendation

Output format:
- Numbered list of verified findings only
- Each: finding, evidence, severity, recommendation
- Confidence score (high/medium) — do not include low-confidence findings
- Summary statistics: total from [I], confirmed, rejected, disputed, final count
```

If `breadth_iii` > 1, spawn agents in parallel. Include only findings that appear in majority of arbiters' outputs.

### End of Run

Store [III]'s output as the run result. If more runs remain, feed this into the next [I] phase as "previous run results."

## Step 5: LLM-as-Judge Evaluation

After all runs complete, spawn a **Judge** subagent to evaluate the triage process quality.

```
You are the Judge — an impartial evaluator of a triage process.

Score each dimension on a 1-5 scale.

## Triage Output to Evaluate

[I] The Pusher's findings:
{pusher_output}

[II] The Challenger's verdicts:
{challenger_output}

[III] The Arbiter's final results:
{arbiter_output}

Source material:
{scoped_context}

## Scoring Dimensions

1. Coverage (1-5): Did [I] find the real issues?
2. Signal-to-Noise (1-5): What ratio of [I]'s findings were real?
3. Challenge Quality (1-5): Did [II] correctly identify false positives?
4. Arbitration Accuracy (1-5): Did [III] correctly resolve disputes?
5. Actionability (1-5): Are findings specific enough to act on?

## Output — respond with JSON only:

{"scores":{"coverage":{"score":N,"rationale":"..."},"signal_to_noise":{"score":N,"rationale":"..."},"challenge_quality":{"score":N,"rationale":"..."},"arbitration_accuracy":{"score":N,"rationale":"..."},"actionability":{"score":N,"rationale":"..."}},"overall":N,"verdict":"STRONG|ADEQUATE|WEAK","improvement_suggestions":["..."]}

Be honest and critical. A perfect score should be rare.
```

## Step 6: Present Results

Present to the user:

1. **Final verified findings** — the arbiter's output from the last run
2. **Triage statistics** — findings generated -> challenged -> verified (counts per phase)
3. **Judge scorecard** — the 5 scores, overall verdict, improvement suggestions
4. **Refinement trajectory** — if multiple runs, show how findings evolved

If the Judge verdict is WEAK, flag this and suggest re-running with higher depth or more runs.

## Notes

- Each subagent is a fresh Agent tool invocation (general-purpose)
- For breadth > 1, launch agents in parallel using multiple Agent tool calls in a single message
- Read scoped context ONCE and reuse across all agents in a run
- Keep agent prompts focused — only relevant files, not entire codebase
- If a run produces zero findings, stop early
