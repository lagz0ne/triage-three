# triage-three

Adversarial triage with three roles. High-confidence results through tension and refinement.

## Principle

Progress needs a pusher — someone who moves fast, digs deep, and doesn't self-censor. But speed breeds recklessness: vagueness, exaggeration, even fabrication. So the pusher needs a challenger — someone rewarded to find what's wrong, unsound, or outright false.

This creates tension. Both sides stretch. The pusher finds more. The challenger catches more. But raw tension is noise. A third role — the arbiter — resolves it. The arbiter only gets rewarded for truth that survived the gauntlet. What comes out is refined, verified, and close to the final answer.

**Three roles, one loop:**

```
[I] Pusher    →  generates findings aggressively
[II] Challenger →  tears apart [I]'s output, catches errors
[III] Arbiter   →  emits only what survived both sides
```

Each role has opposing incentives. The pusher is rewarded for depth. The challenger is rewarded for catching flaws. The arbiter is rewarded only for verified truth — including a false result is heavily penalized.

The more runs you do, the deeper it goes. Each iteration feeds the previous arbiter output back into the pusher, who goes deeper into unresolved areas.

## Install

```bash
claude plugin add lagz0ne/triage-three
```

## Usage

```
/triage-three <goal> [style=...] [runs=N] [depth=1-3]
```

**Examples:**

```
/triage-three find security bugs in src/auth/
/triage-three review this PR for correctness
/triage-three analyze the error handling in lib/
/triage-three audit the API design style=wide-funnel depth=2
/triage-three find performance issues runs=2
```

The command will:
1. Suggest three roles based on your goal — you confirm or refine
2. Scope the target files
3. Run the adversarial loop
4. Score itself with an LLM-as-Judge
5. Present only verified findings with a quality scorecard

## Style Presets

All presets were eval-tested against real CVEs (jsonwebtoken v8.5.1) and synthetic bug fixtures. Every preset achieves F1 1.0 on ground truth.

| Style | Pusher | Challenger | Arbiter | Best for |
|---|---|---|---|---|
| `measured` (default) | Accuracy over quantity | Balanced verification | Conservative | General use. Least noise, least tokens. |
| `wide-funnel` | Aggressive | Balanced verification | Inclusive | When you'd rather over-report than miss something. Surfaces borderline findings as "LIKELY". |
| `baseline` | Aggressive | Ruthless | Conservative | Maximum filtering. May exclude borderline-real issues. |
| `paranoid` | Assume everything is broken | Disprove every finding | Strict (zero FP) | Security audits. Highest token cost. |
| `max-tension` | Assume everything is broken | Disprove every finding | Conservative | Deep search with strong filtering. |

## Parameters

| Parameter | Default | Description |
|---|---|---|
| `style` | `measured` | Prompt style preset (see above) |
| `runs` | `1` | Iterations. Each feeds prior output back in. |
| `depth` | `1` | 1=surface, 2=thorough, 3=exhaustive |
| `breadth_i` | `1` | Parallel pusher agents |
| `breadth_ii` | `1` | Parallel challenger agents |
| `breadth_iii` | `1` | Parallel arbiter agents |

## Use Cases

**Security audit** — Find vulnerabilities, verify they're real, exclude false positives.
```
/triage-three security audit on src/ style=paranoid depth=2
```

**Code review** — Catch bugs, logic errors, and edge cases before merge.
```
/triage-three review the changes in this branch
```

**Architecture review** — Evaluate design decisions, find coupling issues, surface hidden assumptions.
```
/triage-three review architecture of the auth system style=wide-funnel
```

**Bug triage** — Analyze a bug report, verify the root cause, rule out red herrings.
```
/triage-three investigate why payments fail intermittently
```

**Documentation audit** — Find inaccuracies, missing sections, and outdated information.
```
/triage-three audit the API docs for accuracy against the implementation
```

**Dependency risk** — Assess a dependency for security, maintenance, and compatibility risks.
```
/triage-three evaluate the risk of upgrading to next major version of express
```

## How It Was Built

The prompt presets were developed using an eval system with:
- A **prompt variation matrix** (3 variants per role, 5 combinations)
- **Ground truth fixtures** from real CVEs (jsonwebtoken v8.5.1) and synthetic bugs
- **Precision/recall/F1 scoring** against known bugs, non-bugs, and debatable items
- 10 eval runs across 2 fixtures to find which wording works best

The key insight: the differentiator between presets isn't whether they find real bugs (all do), but how they handle **debatable items** — findings that are real but borderline. The `measured` preset excludes debatables cleanly. The `wide-funnel` preset surfaces them as "LIKELY". Choose based on your tolerance for noise vs. your fear of missing something.
