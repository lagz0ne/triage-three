# triage-three

Adversarial triage with three roles. High-confidence results through tension and refinement.

## Principle

Progress needs a pusher — someone who moves fast, digs deep, and doesn't self-censor. But speed breeds recklessness: vagueness, exaggeration, even fabrication. So the pusher needs a challenger — someone who questions through Socratic interrogation, exposing weaknesses not by assertion but by asking the questions that reveal whether a finding holds up.

This creates tension. The pusher proposes. The challenger asks: *"What evidence proves this? What would disprove it? Is this the root cause or a symptom? What's the alternative explanation?"* Both sides stretch. But raw tension is noise. A third role — the arbiter — resolves it, emitting only truth that survived the gauntlet.

**Three roles, multiple angles:**

```
[I] Pusher      →  generates findings from a specific angle
[II] Challenger  →  Socratically questions [I]'s output
[III] Arbiter    →  synthesizes across all pairs, emits verified truth
```

Multiple Pusher–Challenger pairs can explore different angles of the same problem (e.g., correctness + security + performance). The Arbiter synthesizes across all pairs, catching cross-angle patterns and resolving conflicts. Everything works in threes: three roles, up to three angles, up to three runs.

## Install

```bash
claude plugin marketplace add lagz0ne/triage-three
claude plugin install triage-three
```

## Usage

```
/triage-three <what to triage and what you're looking for> [style=...]
```

**Examples:**

```
/triage-three find security bugs in src/auth/
/triage-three review this PR for correctness
/triage-three analyze the error handling in lib/
/triage-three audit the API design style=wide-funnel
/triage-three find performance issues in the hot path
```

The command will:
1. Infer roles from your goal and calibrate effort (angles, depth) from the query's scope and stakes
2. Scope the target files
3. Run Pusher–Challenger pairs (in parallel across angles)
4. Synthesize with the Arbiter across all angles
5. Present only verified findings with cross-angle insights

## Style Presets

All presets were eval-tested against real CVEs (jsonwebtoken v8.5.1) and synthetic bug fixtures. Every preset achieves F1 1.0 on ground truth.

| Style | Pusher | Challenger | Arbiter | Best for |
|---|---|---|---|---|
| `measured` (default) | Accuracy over quantity | Balanced verification | Conservative | General use. Least noise, least tokens. |
| `wide-funnel` | Aggressive | Balanced verification | Inclusive | When you'd rather over-report than miss something. Surfaces borderline findings as "LIKELY". |
| `baseline` | Aggressive | Ruthless | Conservative | Maximum filtering. May exclude borderline-real issues. |
| `paranoid` | Assume everything is broken | Disprove every finding | Strict (zero FP) | Security audits. Highest token cost. |
| `max-tension` | Assume everything is broken | Disprove every finding | Conservative | Deep search with strong filtering. |

## Effort Calibration

Effort is inferred from your query — no parameter tables to fill out.

| Effort | Angles | Depth | Triggered by |
|---|---|---|---|
| **Light** | 1 | surface | Focused question, single file, low stakes |
| **Standard** | 3 | thorough | Most tasks. Module-level, multiple concerns. |
| **Deep** | 3 | exhaustive | Critical systems, security audits, complex problems |

You can always override by passing `style=...` for prompt presets. Everything else is calibrated from context.

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
