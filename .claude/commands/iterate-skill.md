Iterative skill development loop. Argument: $ARGUMENTS

## Step 0: Load prerequisites

BLOCKING REQUIREMENT: Before doing ANYTHING else, invoke the `skill-development` skill from `plugin-dev`. This loads the skill-creator knowledge needed for every iteration, regardless of what the user is building.

Then check if `.claude/commands/iterate-skill.lessons.md` exists. If it does, read it and internalize all lessons before proceeding. These are hard-won rules from previous runs — treat them as constraints, not suggestions.

## Step 1: Define evaluation criteria FIRST

"Cheap to talk, tough to prove."

Before writing a single line of skill content, establish how we'll know it works. Ask the user:

1. **What does success look like?** — concrete, observable outcomes (not vague "it should work well")
2. **How do we test it?** — what command, scenario, or input would prove the skill does its job?
3. **What's the failure mode?** — what does a bad result look like? What should the skill NOT do?

Propose evaluation criteria based on the user's answers. The criteria must be:
- **Executable** — can be tested in this session, not "we'll see over time"
- **Binary** — pass/fail, not subjective
- **Minimal** — fewest checks that cover the most ground

Do NOT proceed until the user confirms the eval criteria.

## Step 2: Socratic alignment loop

Now refine the skill requirements through questioning. The goal: make the request and the proof method match perfectly.

For each round, ask ONE focused question that resolves the biggest ambiguity. Examples:
- "You said it should handle X — does that mean Y or Z?"
- "The eval checks for A, but the description implies B — which is it?"
- "What happens when [edge case]? Should the skill handle it or explicitly reject it?"

Exit conditions for this loop (ALL must be true):
- [ ] The skill's purpose is a single clear sentence
- [ ] The eval criteria directly test that purpose
- [ ] No ambiguity remains between what the user described and what the eval will prove
- [ ] The user explicitly says "ready" or equivalent

If stuck after 3 rounds, summarize what's clear and what's not, and ask the user to resolve the gaps directly.

## Step 3: Build and iterate

Now implement the skill using the skill-development patterns loaded in Step 0.

After each implementation attempt:
1. Run the eval criteria from Step 1
2. If any criterion fails: diagnose why, fix, re-run (do not ask the user unless blocked)
3. If all pass: show the user the results and ask for feedback

Repeat until the user confirms the skill is done.

## Step 4: Extract lessons

When the user confirms completion, spawn an isolated subagent to:

1. Review the full conversation history of this session
2. Extract patterns that would help future runs of this command — things like:
   - Common eval criteria patterns that worked well
   - Questioning patterns that resolved ambiguity quickly
   - Mistakes made during iteration and how they were fixed
   - Structural patterns in skills that passed eval on first try
3. Write findings to `.claude/commands/iterate-skill.lessons.md` (append, don't overwrite)
   - Each lesson: one line, actionable, prefixed with the date
   - Remove lessons that contradict new findings
   - Keep the file under 50 lines — prune low-value lessons

This is how the command evolves itself.
