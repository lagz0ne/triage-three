---
description: "This skill should be used when the user asks for adversarial code review, security audit, bug triage, architecture review, or any deep analysis that benefits from multiple perspectives challenging each other. Trigger phrases include: 'triage this', 'find bugs in', 'security audit', 'review this code', 'adversarial review', 'find vulnerabilities', 'audit this for', 'review this PR for correctness'. Uses three adversarial roles (Pusher finds, Challenger questions, Arbiter decides) with style presets (measured, wide-funnel, paranoid, baseline, max-tension)."
argument-hint: "<what to triage and what you're looking for>"
allowed-tools: ["Agent", "Read", "Glob", "Grep", "AskUserQuestion"]
---
Adversarial triage with three roles. Argument: $ARGUMENTS

## Overview

Triage-three orchestrates adversarial subagents — Pushers propose, Challengers question, Arbiters decide — to produce high-confidence results through tension.

The core unit is the **Pusher–Challenger pair**: a thesis and its Socratic interrogation. Multiple pairs explore different angles of the same problem. The Arbiter synthesizes across all pairs into verified truth.

For multi-run triage, the **Investor** sits between runs — a fund manager operating under the user's goal as its charter. It analyzes ROI, reallocates effort, and issues a directed investment thesis. Every allocation must trace back to the charter. Runs are controlled capital deployment, not open-ended repetition.

Everything works in threes: three roles, up to three angles, up to three runs.

## Scoring System

Each finding follows a lifecycle: **Proposed → Questioned → Decided**. Every agent has a point ledger that makes trade-offs explicit. State the relevant scoring table in each agent's prompt.

### Pusher's Ledger

The Pusher should NOT self-censor — that's the Challenger's job. Points reward finding broadly while still valuing quality.

| Finding outcome | Points |
|-----------------|--------|
| CONFIRMED by Challenger | **+3** |
| DISPUTED by Challenger | 0 |
| REJECTED by Challenger | -1 |

*"Your score is the sum across all findings. Maximize your score. A single confirmed finding (+3) outweighs three rejections (-3). DISPUTED earns nothing — take a position. Find broadly, but find real things."*

### Challenger's Ledger

The Challenger earns points for honest investigation — both confirming and rejecting count, but wrong rejections are costly.

| Verdict outcome | Points |
|-----------------|--------|
| CONFIRMED — and Arbiter includes it | **+2** |
| REJECTED — and Arbiter agrees (correct rejection) | **+3** |
| REJECTED — but Arbiter overrules (wrong rejection) | **-3** |
| DISPUTED — with substantive reasoning | 0 |

*"You are rewarded for accurate verdicts. A correct confirmation is worth almost as much as a correct rejection. But rejecting a real finding costs you dearly. DISPUTED earns nothing — it's a non-verdict. Take a position. Question honestly, don't reject reflexively."*

### Arbiter's Ledger (style-dependent)

The Arbiter's penalties shift by style to encode different risk tolerances.

| Decision | measured | wide-funnel | paranoid |
|----------|----------|-------------|----------|
| Include true finding | +3 | +3 | +3 |
| Exclude false finding | +1 | +1 | +3 |
| **Include false finding** | **-5** | -2 | **-10** |
| **Exclude true finding** | -2 | **-5** | -1 |

- `measured` / `baseline` / `max-tension`: Use `measured` column. Precision matters more than recall.
- `wide-funnel`: Fears missing things more than including noise. Missing a real finding is the worst outcome.
- `paranoid`: A single false positive is catastrophic. Excluding a real finding is acceptable.

*"Your score is the sum across all decisions. The penalties tell you what this triage values most — optimize accordingly."*

### Escalation Across Runs

Points multiply per run. Early runs are cheap to explore; later runs are expensive to be wrong.

| Run | Multiplier | Character |
|-----|-----------|-----------|
| 1 | **1×** | Exploratory — cast wide, find the landscape |
| 2 | **2×** | Refinement — mistakes cost double, focus sharpens |
| 3 | **3×** | Convergence — only high-confidence work survives |

All point values (positive and negative) are multiplied. A REJECTED finding costs -1 in run 1, -2 in run 2, -3 in run 3. A CONFIRMED finding earns +3, +6, +9.

### The Investor (between runs)

Between runs, spawn an **Investor** subagent. The Investor is not a triage role — it's a fund manager operating under a charter. It analyzes ROI from the completed run and issues a **directed investment thesis** that shapes the next run. This replaces generic "go deeper" with controlled, evidence-based resource allocation.

#### Charter = User's Goal

The user's original goal is the Investor's fund charter. The Investor allocates within this charter but cannot drift outside it. Every angle must trace back to the charter — a high-ROI tangent that doesn't serve the user's goal is worth zero.

The `style` preset encodes the Investor's risk profile:
- `measured`: Balanced fund. Invest where returns are proven. Conservative on tangents.
- `wide-funnel`: Growth fund. Aggressive allocation — fund anything that might produce within charter.
- `paranoid`: Capital preservation. Only fund near-certain returns. Cut losses early.

#### Investor's Ledger

The Investor has skin in the game. Its score is computed after the run it directed. Penalties vary by style — matching the Investor's risk profile:

| Outcome | measured | wide-funnel | paranoid |
|---------|----------|-------------|----------|
| New confirmed finding | **+3** | +2 | **+5** |
| Barren angle (nothing new) | -2 | -1 | **-4** |
| Continue, next run found nothing new | -2 | -1 | **-4** |

- `measured`: Balanced. Moderate reward, moderate penalty.
- `wide-funnel`: Growth fund. Lower penalties for misses — exploration is expected to have some barren runs.
- `paranoid`: Capital preservation. High reward for hits, heavy penalty for wasted effort.
- `baseline`: Use `measured` column.
- `max-tension`: Use `paranoid` column.

The escalating multiplier applies to the Investor too. At 3×, a bad allocation costs triple.

For run > 1, the Investor sees its own previous score. It learns from its own track record.

#### Charter Drift Detection

After the Investor produces its directive, the orchestrator checks: does every `invest` angle clearly serve the original charter? If an angle can't trace back to the user's goal, **do not proceed** — flag it to the user via AskUserQuestion:

> "The Investor suggests exploring [angle] — but this may be outside your original goal of [charter]. Should I include this direction or stay focused?"

This prevents score-maximizing drift.

#### Investor Prompt

Spawn with this prompt:
```
You are the Investor — a fund manager allocating effort across triage runs.

## Your Charter (this is your mandate — do not drift outside it)
"{original_user_goal}"

Style/risk profile: {style}

## Your Ledger (run {N}, {multiplier}× multiplier)
{If run > 1: "Your previous investment scored {prev_investor_score}. {breakdown of what produced and what didn't}."}

Scoring (style-dependent — your risk profile determines penalties):
- New confirmed finding from invested angle: +{confirm_reward × multiplier}
- Barren angle (nothing new): -{barren_penalty × multiplier}
- Continue, next run found nothing new: -{wasted_penalty × multiplier}

## Run {N} Results

Scorecard:
- Pusher: {pusher_score} ({X} confirmed, {Y} disputed, {Z} rejected out of {N_total} proposed)
- Challenger: {challenger_score} ({A} correct confirms, {B} correct rejects, {C} wrong rejects, {D} disputed)
- Arbiter: {arbiter_score} ({E} included, {F} excluded)
- Confirmation rate: {confirmed / total_proposed}%

Results by angle:
{For each angle: name, findings proposed, confirmed count, rejected count, areas/files explored, points earned}

Verified findings so far:
{arbiter_output}

## Your Job

Given your charter, analyze ROI and allocate the next run:
- Which angles had the best return WITHIN the charter?
- Which angles burned effort for nothing?
- Where are the unexplored adjacencies — areas NEAR confirmed findings that the charter cares about?
- Is another run worth the cost, or has the charter been satisfied?

## Output — respond with JSON only:

{
  "continue": true/false,
  "rationale": "Why continue or stop. Cite ROI numbers AND how this serves the charter.",
  "invest": [
    {
      "angle": "name or new angle",
      "depth": "surface | thorough | exhaustive",
      "focus": "Specific area, file, or topic. Be concrete.",
      "charter_link": "How this angle serves the original goal.",
      "why": "What ROI signal justifies this investment."
    }
  ],
  "divest": [
    { "angle": "name", "why": "ROI signal + charter relevance." }
  ],
  "thesis": "1-2 sentence investment thesis. What are we hunting for, why do we believe it's there, and how does it serve the charter?"
}

Rules:
- Every investment must include `charter_link` — no unmoored allocations.
- If confirmation rate falls below the style threshold, recommend stopping. Thresholds: measured/baseline = 20%, wide-funnel = 10%, paranoid/max-tension = 30%.
- If all angles had 0 new confirmed findings beyond previous run → recommend stopping. Diminishing returns.
- Never recommend more than 3 angles (factor of three).
- New angles are welcome IF they serve the charter. Explain the connection.
- Kill angles that aren't producing, even if they were "primary" last run.
- Your risk profile is {style}: {risk_profile_description}.
```

The Investor's directive replaces all generic feedback. The next run's agents receive the thesis, specific focus areas, and charter context instead of "go deeper."

## Step 1: Understand & Calibrate

Parse the goal from `$ARGUMENTS`. If empty or unclear, ask using AskUserQuestion.

### Infer Roles Silently

From the goal, determine what the three roles do. Do NOT ask the user to name or confirm them — just proceed.

- [I] Pusher: what to pursue (e.g., "find bugs", "spot security issues", "identify refactoring opportunities")
- [II] Challenger: what to question (e.g., "verify these are real bugs", "challenge severity claims")
- [III] Arbiter: what to decide (e.g., "emit only confirmed issues", "rank by actual impact")

### Calibrate Effort from the Query

Read the goal and reason about the right effort level. Consider:

- **Scope**: A single file → light. A module → standard. A whole system → deep.
- **Stakes**: Quick check → light. Production audit → deep. Security review → deep.
- **Specificity**: "Are there bugs in this function?" → 1 angle. "Review this codebase" → 3 angles.

Derive effort:

| Effort | Angles | Depth | When |
|--------|--------|-------|------|
| **Light** | 1 | surface | Focused question, small scope, low stakes |
| **Standard** | 3 | thorough | Most tasks. Broad enough to catch what matters. |
| **Deep** | 3 | exhaustive | Critical systems, security audits, complex problems |

**Briefly tell the user** your reasoning and chosen effort level before proceeding:

> "This looks like a [light/standard/deep] triage. I'll explore [N] angle(s): [list]. Here's why: [1-sentence rationale]."

If the user pushes back, adjust. Otherwise, proceed. Do NOT present parameter tables or ask the user to pick from systematic options.

### Plan Angles (for Standard/Deep)

When using multiple pairs, identify distinct angles — genuinely different perspectives, not overlapping.

- **Prioritize**: primary angle gets full depth, others get depth−1 (minimum surface).
- Examples: correctness + security + performance, feasibility + impact + risk, logic + edge-cases + API contract.

## Step 2: Determine Scope

Identify what agents will operate on:

- **Code**: Target files/directories. Read them to build context.
- **Documents**: Target docs, specs, PRs.
- **Architecture**: Code + configs + docs together.

Gather context so agents work on concrete material. Do NOT run agents on vague input.

## Step 3: Execute Triage

### Phase [I]–[II]: Pusher–Challenger Pairs

Execution order:
1. Spawn all Pusher agents in parallel (one per angle). Wait for all to complete.
2. Spawn all Challenger agents in parallel, each receiving its paired Pusher's output. Wait for all to complete.
3. Spawn the Arbiter with all Pusher + Challenger outputs.

#### [I] The Pusher

Select mandate from `style` (default: `measured`):

| Style | Mandate |
|-------|---------|
| `measured` | "Find real issues. Accuracy over quantity. Evidence for each claim." |
| `wide-funnel` / `baseline` | "Find as much as possible. Quantity and depth. Don't self-censor." |
| `paranoid` / `max-tension` | "Assume everything is broken. Find what others would miss." |

Prompt structure:
```
You are [I] The Pusher — a {role_i_description}.
Angle: {angle_description}

{mandate}

Scoring (run {run_number}, {multiplier}× multiplier):
- CONFIRMED: +{3 × multiplier}
- DISPUTED: 0 (no reward — take a position)
- REJECTED: -{1 × multiplier}
A single confirmed finding outweighs three rejections. Find broadly, but find real things. Do NOT self-censor — the Challenger exists to test your work.

Depth: {depth_description}
- surface: "Most obvious and impactful items. Be concise."
- thorough: "Consider edge cases, subtle issues, non-obvious patterns."
- exhaustive: "Leave no stone unturned. Second-order effects, systemic issues."

{If run > 1:
"## Investor's Directive (from Run {N-1})
Thesis: {investor_thesis}
Your angle's focus: {investor_focus_for_this_angle}
Why this investment: {investor_why}

Previous verified findings (do NOT re-propose these):
{previous_arbiter_output}

Previous scorecard: Pusher {prev_pusher_score}, Challenger {prev_challenger_score}, Arbiter {prev_arbiter_score}

Follow the Investor's directive. Hunt where the ROI signal points."}

Material:
{scoped_context}

Output: numbered findings, each with statement + evidence/location + severity.
```

#### [II] The Challenger (Socratic)

The Challenger uses **Socratic questioning** — not assertion — to stress-test findings. This is the source of adversarial tension. The Challenger genuinely investigates each question against the source material before reaching a verdict.

Select intensity from `style`:

| Style | Intensity |
|-------|-----------|
| `measured` / `wide-funnel` | Genuine inquiry. Verify together. Benefit of the doubt when evidence is ambiguous. |
| `baseline` | Skeptical cross-examination. Prove it or it's out. |
| `paranoid` / `max-tension` | Hostile interrogation. Wrong until proven right. |

Prompt structure:
```
You are [II] The Challenger — a {role_ii_description}.
Angle: {angle_description}

You stress-test findings through SOCRATIC QUESTIONING. You do not simply accept or reject — you interrogate each finding through probing questions, then investigate the answers yourself against the source material.

Scoring (run {run_number}, {multiplier}× multiplier):
- Correct CONFIRMED: +{2 × multiplier}
- Correct REJECTED: +{3 × multiplier}
- Wrong REJECTED (Arbiter overrules): -{3 × multiplier}
- DISPUTED: 0 (non-verdict — take a position)
Question honestly — don't reject reflexively. A wrong rejection at {multiplier}× hurts. DISPUTED earns nothing.

{If run > 1:
"Investor's thesis: {investor_thesis}
Previous Challenger scored {prev_challenger_score}. The Investor directed this angle toward: {investor_focus_for_this_angle}. Question findings against that context."}

For EACH finding from [I], work through these questions:

1. "What specific evidence proves this?" — Point to the exact code, line, or behavior. Vague evidence = vague finding.

2. "What would need to be true for this NOT to be an issue?" — Identify the disproof condition, then check if it exists in the source material.

3. "Is this the root cause or a symptom?" — Could something deeper explain this? Is [I] treating a surface effect as the disease?

4. "What's the alternative explanation?" — Is there a reasonable reading where this is intentional, acceptable, or a known trade-off?

5. "Under what real conditions does this matter?" — Is this a practical concern or a theoretical edge case that never triggers?

{intensity_mandate}

After questioning, reach a verdict:
- CONFIRMED: Survived all questions. It's real. State what convinced you.
- REJECTED: A question broke it. State which question and what you found.
- DISPUTED: Nuance revealed — partially true, context-dependent, or needs more info.

[I]'s findings:
{pusher_output}

Source material:
{scoped_context}

Output: for each finding, the questions asked, what investigation revealed, and verdict. End with confirmed / rejected / disputed counts.
```

### Phase [III]: The Arbiter

After ALL pairs complete, the Arbiter synthesizes across every angle.

Select mandate from `style`:

| Style | Mandate |
|-------|---------|
| `measured` / `baseline` / `max-tension` | "Only verified truth. False positive = heavy penalty. Missing true finding = mild penalty." |
| `wide-funnel` | "Most complete truth. Missing real finding = HEAVY penalty. Borderline inclusion = mild penalty." |
| `paranoid` | "Only PROVABLY TRUE. Any doubt → exclude." |

Prompt structure:
```
You are [III] The Arbiter — a {role_iii_description}.

{mandate}

Scoring (run {run_number}, {multiplier}× multiplier):
- Include true finding: +{3 × multiplier}
- Exclude false finding: +{1 × multiplier}
- Include false finding: -{include_false_penalty × multiplier}
- Exclude true finding: -{exclude_true_penalty × multiplier}
The penalties tell you what this triage values most. At {multiplier}× multiplier, a single false positive costs {include_false_penalty × multiplier} points — optimize accordingly.

{If run > 1:
"Investor's thesis: {investor_thesis}
Previous Arbiter scored {prev_arbiter_score}. The Investor {invested_or_divested_summary}. Weigh findings accordingly — the Investor's thesis reflects where ROI was found."}

{N} Pusher–Challenger pair(s) to synthesize:

{For each angle:}
### Angle: {angle}
[I]'s findings: {pusher_output}
[II]'s Socratic challenges: {challenger_output}

Source material:
{scoped_context}

Rules:
- CONFIRMED by [II]: Include, refine for clarity
- REJECTED by [II]: Exclude UNLESS [II]'s reasoning is itself flawed (verify independently)
- DISPUTED by [II]: Independent judgment against source material
- Cross-angle convergence: If multiple angles flag the same area → higher priority
- Cross-angle conflict: If angles disagree, investigate the discrepancy

Output:
- Verified findings grouped by priority
- Each: finding, evidence, severity, recommendation
- Confidence: high/medium only
- Stats: total from all [I]s → confirmed → rejected → disputed → final
- Cross-angle insights if multiple angles were used
```

### Iteration (Investor-directed loop)

If effort is Deep or the user requested multiple runs:

1. **Compute scorecard** — calculate Pusher, Challenger, and Arbiter point totals for this run (using the current multiplier). Calculate confirmation rate per angle.

2. **Spawn the Investor** — give it the full scorecard, per-angle ROI breakdown, and verified findings. The Investor returns a structured directive: continue/stop, angles to invest in, angles to divest from, and an investment thesis.

3. **If Investor says stop** → the triage has converged or ROI is negative. Present final results.

4. **If Investor says continue** → the directive reshapes the next run:
   - **New angles**: The Investor can introduce angles not in the previous run (following a signal from confirmed findings).
   - **Killed angles**: The Investor can drop angles that produced no return, even if they were "primary."
   - **Refocused angles**: Same angle, but the Investor narrows the focus area based on where returns were found.
   - **Depth adjustments**: The Investor sets depth per angle based on ROI.

5. **Next run** uses multiplier N+1 (run 2 = 2×, run 3 = 3×). All point values scale. Higher stakes make the Investor's allocation decisions matter more — wasting effort at 3× is expensive.

6. **Feed forward**: Each agent in the next run receives the Investor's thesis and their angle's specific focus. They don't get generic "go deeper" — they get directed orders backed by ROI evidence.

## Step 4: Present Results

1. **Verified findings** — the Arbiter's final output
2. **Triage flow** — findings generated → questioned → verified (per angle + total)
3. **Scorecard** — compute each agent's point total from the scoring system. Show the ledger:
   - Pusher: N findings proposed, X confirmed (+3 each), Y disputed (0), Z rejected (-1) = **total**
   - Challenger: X confirmed (+2), W correctly rejected (+3), V wrongly rejected (-3), U disputed (0) = **total**
   - Arbiter: decisions × weights from style column = **total**
   - ROI per angle: confirmation rate and points earned
4. **Investment trail** (multi-run only) — show the Investor's directive after each run: what was invested in, what was divested, and why. This makes the reasoning chain visible.
5. **Cross-angle patterns** — what multiple lenses revealed together (if applicable)
6. **Evolution** — if multiple runs, how findings and angles shifted across runs

## Notes

- Each subagent is a fresh Agent tool invocation (general-purpose)
- Launch [I]–[II] pairs in parallel across angles
- Read scoped context ONCE and reuse across all agents
- If a pair produces zero findings, skip it in arbitration
- If a run produces zero findings total, stop early
- Style: pass as `style=X` in arguments; defaults to `measured`
- Available styles: `measured`, `wide-funnel`, `baseline`, `paranoid`, `max-tension`
