# Scoring System Reference

Complete scoring tables for all triage-three roles. Include the relevant table in each agent's prompt.

## Pusher's Ledger

The Pusher should NOT self-censor — that's the Challenger's job. Points reward finding broadly while still valuing quality.

| Finding outcome | Points |
|-----------------|--------|
| CONFIRMED by Challenger | **+3** |
| DISPUTED by Challenger | 0 |
| REJECTED by Challenger | -1 |

*"Your score is the sum across all findings. Maximize your score. A single confirmed finding (+3) outweighs three rejections (-3). DISPUTED earns nothing — take a position. Find broadly, but find real things."*

## Challenger's Ledger

The Challenger earns points for honest investigation — both confirming and rejecting count, but wrong rejections are costly.

| Verdict outcome | Points |
|-----------------|--------|
| CONFIRMED — and Arbiter includes it | **+2** |
| REJECTED — and Arbiter agrees (correct rejection) | **+3** |
| REJECTED — but Arbiter overrules (wrong rejection) | **-3** |
| DISPUTED — with substantive reasoning | 0 |

*"You are rewarded for accurate verdicts. A correct confirmation is worth almost as much as a correct rejection. But rejecting a real finding costs you dearly. DISPUTED earns nothing — it's a non-verdict. Take a position. Question honestly, don't reject reflexively."*

## Arbiter's Ledger (style-dependent)

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

## Escalation Across Runs

Points multiply per run. Early runs are cheap to explore; later runs are expensive to be wrong.

| Run | Multiplier | Character |
|-----|-----------|-----------|
| 1 | **1×** | Exploratory — cast wide, find the landscape |
| 2 | **2×** | Refinement — mistakes cost double, focus sharpens |
| 3 | **3×** | Convergence — only high-confidence work survives |

All point values (positive and negative) are multiplied. A REJECTED finding costs -1 in run 1, -2 in run 2, -3 in run 3. A CONFIRMED finding earns +3, +6, +9.

## Investor's Ledger

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

## Charter Drift Detection

After the Investor produces its directive, the orchestrator checks: does every `invest` angle clearly serve the original charter? If an angle can't trace back to the user's goal, **do not proceed** — flag it to the user via AskUserQuestion:

> "The Investor suggests exploring [angle] — but this may be outside your original goal of [charter]. Should I include this direction or stay focused?"

This prevents score-maximizing drift.

## Scorecard Output Format

Present the scorecard as:
- Pusher: N findings proposed, X confirmed (+3 each), Y disputed (0), Z rejected (-1) = **total**
- Challenger: X confirmed (+2), W correctly rejected (+3), V wrongly rejected (-3), U disputed (0) = **total**
- Arbiter: decisions × weights from style column = **total**
- ROI per angle: confirmation rate and points earned
