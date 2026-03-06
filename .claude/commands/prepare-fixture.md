Prepare a ground truth fixture for triage-eval. Argument: $ARGUMENTS

## Overview

Build a ground truth fixture JSON from a real project with known bugs. The fixture defines what is a real bug, what is not, and what is debatable — used by `/triage-eval` to score prompt variants.

## Step 1: Identify Source

Parse `$ARGUMENTS` for the target. Accepts:
- A git repo URL + version tag (e.g., `https://github.com/auth0/node-jsonwebtoken v8.5.1`)
- A local directory path
- A specific file path

If a repo URL is provided:
1. Clone to `eval/repos/{repo-name}` (shallow clone)
2. Checkout the specified version tag
3. Identify the key source files (not tests, not docs)

If unclear, ask using AskUserQuestion what files to focus on.

## Step 2: Gather Known Bugs

Use parallel agents to discover documented bugs from multiple sources:

### Agent A: CVE/Advisory Search
```
Search for known CVEs, security advisories, and vulnerability reports for {project} version {version}.
For each, extract: CVE ID, affected file, description, severity, fix description.
Use WebSearch to find this information.
Return structured findings only.
```

### Agent B: Closed Issues/PRs
```
Search the project's GitHub issues and PRs for bug fixes between {version} and the next major version.
Focus on issues labeled "bug", "security", "fix".
For each, extract: issue/PR number, affected file, description, fix description.
Use Bash with `gh` CLI to query: gh issue list --repo {repo} --state closed --label bug
Return structured findings only.
```

### Agent C: Diff Analysis
```
Analyze the git diff between {version} and the next release to identify security-relevant changes.
Focus on: new validation, removed unsafe patterns, added error handling, new security checks.
For each change, extract: file, lines changed, what was fixed, why it was a bug.
Use Bash with git diff to analyze.
Return structured findings only.
```

Combine outputs from all three agents. Deduplicate by matching on file + description.

## Step 3: Classify Findings

Present the combined findings to the user via AskUserQuestion, grouped by confidence:

**High confidence (likely real bugs):**
- Items with CVE IDs
- Items confirmed by multiple sources (CVE + diff + issue)

**Medium confidence (needs judgment):**
- Items found only in diff analysis
- Items from issues without clear CVE

**Low confidence (likely not bugs):**
- Style changes (var -> const, etc.)
- Refactoring without security impact
- Dependency updates

Ask the user to classify each group:
1. "Real bug" — confirmed issue
2. "Not a bug" — style, refactoring, or non-issue
3. "Debatable" — with lean direction (real or not-bug)

## Step 4: Read Source Files

Read the source files at the specified version. These will be the "target material" that triage agents analyze during eval.

## Step 5: Generate Fixture JSON

Write the fixture to `eval/fixtures/{project}-{version}.json`:

```json
{
  "id": "{project}-{version}",
  "name": "{project} {version} — {brief description}",
  "source_files": ["relative/path/to/file1.js", "relative/path/to/file2.js"],
  "context": "Brief description of what this code does and what kind of bugs to look for.",
  "ground_truth": {
    "real_bugs": [
      {
        "id": "short-id",
        "cve": "CVE-YYYY-NNNNN (if applicable)",
        "file": "filename.js",
        "lines": "N-M (if known)",
        "description": "Clear description of the bug",
        "severity": "critical|high|medium|low",
        "type": "security-*|runtime-crash|silent-failure|logic-error",
        "fix_in_next": "Brief description of how it was fixed"
      }
    ],
    "not_bugs": [
      {
        "id": "short-id",
        "file": "filename.js (optional)",
        "description": "Why this is not a bug"
      }
    ],
    "debatable": [
      {
        "id": "short-id",
        "file": "filename.js (optional)",
        "description": "Why this is debatable",
        "lean": "real-bug|not-a-bug"
      }
    ]
  }
}
```

## Step 6: Verify

After writing the fixture, present a summary:
- Total real bugs, not-bugs, debatable items
- Source files included
- Any gaps (e.g., CVEs found but no matching code location)

Ask user to confirm or adjust before finalizing.

## Notes

- Fixture files are gitignored — they stay local for eval development
- Each fixture should have at least 3 real bugs and 1 not-bug for meaningful eval
- Debatable items are what differentiate prompt variants — include them when found
- Always include the version tag so the fixture is reproducible
