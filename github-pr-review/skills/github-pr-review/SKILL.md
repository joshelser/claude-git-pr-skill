---
name: github-pr-review
description: Use when reviewing GitHub pull requests with gh CLI - creates pending reviews with code suggestions, batches comments, and chooses appropriate event types (COMMENT/APPROVE/REQUEST_CHANGES)
allowed-tools: Bash(gh:*), Bash(python3:*), Bash(mkdir:*), Bash(rm:*)
---

# GitHub PR Review

## Overview

Full-lifecycle PR review skill: **analyze** a PR with parallel sub-agents, present findings, then optionally **create a draft (pending) review** via `gh api`. The analysis phase launches three read-only agents in parallel (change summary + architecture + anti-patterns) and aggregates their results into a single report. The drafting phase creates a pending review with comments — it is NEVER submitted/published by this skill.

**CRITICAL: This skill NEVER publishes reviews to GitHub.** It only creates draft (pending) reviews. The user publishes them manually on GitHub. Always get explicit user approval before even creating a draft, and show exactly what will be drafted using AskUserQuestion.

---

## Automated PR Analysis

Use this workflow when you need to review a PR. It fetches the diff, launches three parallel analysis agents, and presents a consolidated report.

### Step 1: Fetch & Cache All PR Data

**Fetch everything upfront with exactly 3 `gh` calls**, then work from local files for the rest of the review. This avoids redundant network round-trips.

The temp directory is namespaced by repo and PR number to avoid collisions: `/tmp/pr-review-<OWNER>-<REPO>-<PR_NUMBER>`.

```bash
# Call 1: Get repo owner/name, create workspace, and save repo info
# This single command fetches repo info, creates the temp dir, and saves the JSON
python3 -c "
import subprocess, json, os
repo = json.loads(subprocess.run(['gh', 'repo', 'view', '--json', 'owner,name'], capture_output=True, text=True).stdout)
owner = repo['owner']['login']
name = repo['name']
d = f'/tmp/pr-review-{owner}-{name}-<PR_NUMBER>'
os.makedirs(d, exist_ok=True)
with open(f'{d}/repo.json', 'w') as f:
    json.dump(repo, f)
print(d)
"
# Prints the workspace path, e.g.: /tmp/pr-review-acme-webapp-42
# Use this path for all subsequent commands.

# Call 2: Get ALL PR metadata in one shot (title, author, stats, commit SHA)
gh pr view <PR_NUMBER> --json title,author,additions,deletions,changedFiles,commits,baseRefName,headRefName,url > /tmp/pr-review-<OWNER>-<REPO>-<PR_NUMBER>/metadata.json

# Call 3: Get the full diff and save locally
gh pr diff <PR_NUMBER> > /tmp/pr-review-<OWNER>-<REPO>-<PR_NUMBER>/diff.patch
```

Then extract the fields you need from the cached files:

```bash
# Extract commit SHA from cached metadata (no gh calls)
python3 -c "
import json
meta = json.load(open('/tmp/pr-review-<OWNER>-<REPO>-<PR_NUMBER>/metadata.json'))
print(meta['commits'][-1]['oid'])
" > /tmp/pr-review-<OWNER>-<REPO>-<PR_NUMBER>/commit_sha.txt
```

Use the Read tool to load `/tmp/pr-review-<OWNER>-<REPO>-<PR_NUMBER>/metadata.json`, `/tmp/pr-review-<OWNER>-<REPO>-<PR_NUMBER>/repo.json`, and `/tmp/pr-review-<OWNER>-<REPO>-<PR_NUMBER>/diff.patch` into your context. Extract the PR title, author, additions/deletions counts, changed file count, commit SHA, and repo owner/name for use in agent prompts and later review creation.

**From this point forward, NEVER call `gh pr view`, `gh pr diff`, or `gh repo view` again. All data is local.**

### Step 2: Launch Three Explore Sub-Agents in Parallel

Launch **all three agents in a single message** so they run concurrently. All use `subagent_type: "Explore"` (read-only, fast, has access to Glob/Grep/Read).

IMPORTANT: When you need to view specific line ranges from a file, use the Read tool with offset and limit parameters (e.g., offset=40, limit=20 to read lines 40-59). Do NOT use Bash with sed, head, or tail — those require permission prompts and will interrupt the review.

For large PRs (500+ changed lines), include this instruction in all three prompts: "This is a large PR. Prioritize the most architecturally significant files. If you cannot cover everything, note which files were skipped and why."

#### Agent A: Architecture Review

Prompt template (fill in `{DIFF}`, `{PR_TITLE}`, `{PR_AUTHOR}`):

```
You are reviewing PR "{PR_TITLE}" by {PR_AUTHOR}. Your job is to assess the architectural quality of the changes as a function of the entire application.

Here is the full diff:

<diff>
{DIFF}
</diff>

IMPORTANT: Use Glob, Grep, and Read to explore the surrounding codebase. Check whether new code is consistent with existing patterns, understand how it fits into the broader architecture, and verify that similar problems are solved similarly elsewhere.

IMPORTANT: When you need to view specific line ranges from a file, use the Read tool with offset and limit parameters (e.g., offset=40, limit=20 to read lines 40-59). Do NOT use Bash with sed, head, or tail — those require permission prompts and will interrupt the review.

Answer these questions:

1. **Problem & solution**: What problem is the author solving? How is the solution organized?
2. **Separation of concerns**: Is there a clear division of responsibilities? Are layers/modules properly separated?
3. **Encapsulation**: If a bug shows up in this code, is ownership obvious? Are implementation details properly hidden?
4. **Intent test**: Does this look like deliberate design, or like generated code pasted without thought?
5. **Data flow**: Trace how domain objects move through the new code. Are the same entities fetched repeatedly from storage? Could methods accept pre-resolved objects to reduce hidden I/O and simplify testing?
6. **Architectural commitment**: Does this PR make decisions — coupling to a specific vendor/implementation, establishing new patterns, choosing data formats or storage strategies — that will be expensive to reverse? Is the level of commitment proportionate to the problem? Look for vendor-specific types leaking into core interfaces, patterns that future code will be forced to follow, and tight coupling where the codebase already shows the need for flexibility.

Apply these standards:
- Strategy over optionality (explicit paths, not feature flags for everything)
- Composition over inheritance
- Rich domain models over primitive obsession
- KISS/YAGNI — no unnecessary abstractions or premature generalization
- Single Responsibility Principle
- Accept resolved domain objects, not raw IDs — when multiple methods in a class each fetch the same entity by ID internally, the API boundary should require callers to provide the resolved object instead (reduces hidden I/O, simplifies testing, makes data flow explicit)
- Proportion commitment to certainty — the harder a decision is to reverse, the more it deserves an abstraction boundary. Internal helpers can be inlined (YAGNI); external service integrations, data formats baked into storage, and patterns that every future feature must follow deserve interfaces or extension points. When the codebase already abstracts similar concerns (e.g., multiple auth backends), a new concrete dependency that bypasses that abstraction is a red flag.

Output your findings as a JSON array. Each finding:
{
  "file": "path/to/file.ext",
  "line": 42,
  "category": "separation-of-concerns" | "encapsulation" | "naming" | "design-intent" | "consistency" | "complexity" | "architectural-commitment",
  "severity": "concern" | "suggestion" | "question",
  "description": "Clear explanation of the finding"
}

After the JSON array, answer the six questions we started with. Remember that our focus is on the application as a whole.
```

#### Agent B: Anti-Pattern Review (Python only)

Prompt template (fill in `{DIFF}`, `{PR_TITLE}`):

```
You are reviewing PR "{PR_TITLE}" for common Python anti-patterns. Analyze ONLY the added lines (lines starting with `+`) in `.py` files from the diff below.

<diff>
{DIFF}
</diff>

IMPORTANT: When you need to view specific line ranges from a file, use the Read tool with offset and limit parameters (e.g., offset=40, limit=20 to read lines 40-59). Do NOT use Bash with sed, head, or tail — those require permission prompts and will interrupt the review.

Flag ONLY these specific anti-patterns:

1. **bare-dicts**: Using `dict`, `Dict`, untyped dictionaries, or `TypedDict` where a Pydantic `BaseModel` or `@dataclass` should be used for domain data. Plain dicts are acceptable for transient mappings (e.g., kwargs, headers), but NOT for structured domain objects.

2. **error-suppression**: Catching exceptions and returning None/default/empty to hide failures. Code should fail fast and let errors propagate. Acceptable ONLY when the docstring explicitly documents why suppression is intentional.

3. **missing-types**: Bare `list`, bare `dict`, `Any`, or missing argument/return type annotations on functions/methods. Every function signature should have complete type annotations.

4. **type-ignore**: `# type: ignore` comments. These should never appear without a specific error code (e.g., `# type: ignore[assignment]`) and a justifying comment.

5. **noqa**: `# noqa` comments. Should be rare and always include the specific rule code and a justifying comment.

6. **f-string-logging**: Using f-strings or `.format()` in `logger.*()` / `log.*()` calls instead of structlog keyword arguments. Bad: `logger.info(f"User {user_id}")`. Good: `logger.info("user action", user_id=user_id)`.

Rules:
- ONLY flag patterns in added lines (starting with `+`).
- Use the NEW file line numbers (right side of the diff) for line references.
- If a file is not a `.py` file, skip it entirely.
- Do NOT flag imports, test files, or configuration files for missing-types.

Output your findings as a JSON array. Each finding:
{
  "file": "path/to/file.ext",
  "line": 42,
  "pattern": "bare-dicts" | "error-suppression" | "missing-types" | "type-ignore" | "noqa" | "f-string-logging",
  "severity": "concern" | "suggestion",
  "snippet": "the offending line of code",
  "fix": "what the code should look like instead, or why it matters"
}

After the JSON array, write a 2-3 sentence summary of the anti-pattern assessment.

If there are no findings, return an empty array and a summary saying no anti-patterns were detected.
```

#### Agent C: Change Summary

Prompt template (fill in `{DIFF}`, `{PR_TITLE}`, `{PR_AUTHOR}`, `{CHANGED_FILES_COUNT}`, `{ADDITIONS}`, `{DELETIONS}`):

```
You are summarizing PR "{PR_TITLE}" by {PR_AUTHOR} ({ADDITIONS} additions, {DELETIONS} deletions across {CHANGED_FILES_COUNT} files). Your job is to help reviewers understand the change — NOT to evaluate or critique it.

Here is the full diff:

<diff>
{DIFF}
</diff>

IMPORTANT: Use Glob, Grep, and Read to explore the surrounding codebase so you can describe the existing state accurately. When you need to view specific line ranges from a file, use the Read tool with offset and limit parameters (e.g., offset=40, limit=20 to read lines 40-59). Do NOT use Bash with sed, head, or tail — those require permission prompts and will interrupt the review.

Produce exactly three sections:

1. **Existing State** — What currently exists in the codebase that this PR touches. Describe the purpose of the affected modules/files, key interfaces or classes, and how they fit into the broader system. For entirely new files, describe the area of the codebase where they are being added and what exists there today.

2. **What Is Changing** — Concretely list: new files/classes/functions introduced, existing code modified, code deleted, and any config or dependency changes. Call out any new external service integrations or vendor SDK dependencies introduced, and any new architectural patterns established. Use specific names (class names, function names, file paths). Keep it factual and concise.

3. **Implementation Approach** — Describe HOW the author implemented the change: design patterns used, integration strategy with existing code, key abstractions introduced, and notable technical choices.

Rules:
- Stay under 500 words total.
- Be factual and descriptive, NOT evaluative. Do not say "good", "bad", "should", "concern", or make suggestions.
- Do not produce findings, suggestions, or a JSON array. Plain prose only.
- Use specific names from the code (class names, function names, file paths).
```

### Step 3: Aggregate and Present Report

Once all three agents complete, combine their results into this format:

```
## PR #<NUMBER> Analysis: <TITLE>
Author: <author> | Changes: +<additions> -<deletions> across <N> files

### Change Summary
<Full output from Agent C, preserving its three sub-sections:>

#### Existing State
<Agent C existing state section>

#### What Is Changing
<Agent C what is changing section>

#### Implementation Approach
<Agent C implementation approach section>

### Architecture Assessment
<2-3 sentence summary from Agent A>

### Anti-Pattern Assessment
<2-3 sentence summary from Agent B>

### Findings
| # | File | Line(s) | Category | Severity | Description |
|---|------|---------|----------|----------|-------------|
| 1 | path/to/file.py | 42 | bare-dicts | concern | Using plain dict for user profile data — should be a Pydantic model |
| 2 | path/to/other.py | 15-20 | separation-of-concerns | suggestion | Database query mixed with business logic |
| ... | | | | | |

### Suggested Verdict
<APPROVE | COMMENT | REQUEST_CHANGES> — <one sentence reasoning>
(This is a suggestion only — the user will choose the actual verdict when they publish the review on GitHub.)
```

Rules for the suggested verdict:
- **REQUEST_CHANGES**: Any finding with severity "concern" that indicates a bug, security issue, or design flaw
- **COMMENT**: Findings exist but are all suggestions/questions
- **APPROVE**: No findings or only minor suggestions

### Step 4: Ask User What to Do Next

Use AskUserQuestion with these options:

```
Question: "How would you like to proceed with these findings?"
Header: "PR Review"
Options:
  - Draft all findings as review comments: Creates a draft (pending) review — NOT published, only visible to you
  - Select which findings to draft: Let me choose which findings to include in the draft
  - Done, don't draft anything: End the review without creating a draft
```

If the user selects "Draft all findings" or "Select which findings", transition to the **Draft Review Workflow** below. Map each finding to a review comment using the file path and line number from the findings table.

**CRITICAL: This only creates a DRAFT (pending) review on GitHub. It is NEVER submitted/published. The user must manually submit the review on GitHub when they are ready.**

If the user selects "Done", end cleanly.

---

## Draft Review Workflow

Use this workflow to create a **draft (pending) review** on GitHub. The review is NEVER submitted/published — only the user can do that manually on GitHub.

**CRITICAL: This skill MUST NEVER call the submit/events endpoint. We only create pending reviews. The user publishes them manually on GitHub.**

### When to Use

- After automated analysis when user wants to draft findings as review comments
- Preparing review comments for a PR
- Adding code suggestions to PRs as a draft

### Core Workflow

**REQUIRED STEPS (do not skip):**

1. **Prepare the review** - Analyze PR and prepare all comments
2. **Show user exactly what will be drafted** - Use AskUserQuestion with yes/no
3. **Get explicit approval** - Wait for user confirmation
4. **Create the draft review** - Only after approval. This creates a PENDING review (visible only to the reviewer, NOT published)

**NEVER proceed to submit/publish the review. The user will do that on GitHub.**

### Approval Pattern

Before creating ANY draft review, use AskUserQuestion to show:
- File and line number for each comment
- Exact comment text (including code suggestions)
- A reminder that this creates a draft only — they will publish it themselves on GitHub

**Example:**
```
Question: "Ready to create this draft review? (You'll publish it yourself on GitHub)"
Header: "PR Review — Draft Only"
Options:
  - Yes, create draft: Creates a pending review visible only to you — NOT published
  - No, let me revise: Allows refinement
```

### Technical Workflow

**ALWAYS use the pending review pattern, even for single comments. NEVER submit/publish the review.**

**IMPORTANT: Use JSON input (`--input -`), not `-f` array syntax.** The `-f 'comments[][...]'` array syntax is unreliable with `gh api` and produces GraphQL errors. Always pipe a JSON body via `--input -` instead.

**IMPORTANT: Use `position` (diff hunk position), not `line`/`side`.** The review comments API requires `position` — the 1-based line index within the file's unified diff. The `line` and `side` fields are NOT valid for `DraftPullRequestReviewComment` and will cause 422 errors.

#### Calculating diff positions

The `position` for each comment must be computed from the cached PR diff. **Read from the local file — do NOT call `gh pr diff` again.**

```bash
# Calculate positions from the CACHED diff (no network call)
python3 -c "
diff = open('/tmp/pr-review-<OWNER>-<REPO>-<PR_NUMBER>/diff.patch').read()

file = ''
pos = 0

for line in diff.split('\n'):
    if line.startswith('diff --git'):
        file = ''
        pos = 0
    elif line.startswith('+++ b/'):
        file = line[6:]
        pos = 0
    elif line.startswith('@@'):
        pos = 0

    if file:
        pos += 1
        # Add search terms for lines you want to comment on
        if 'your_search_term' in line:
            print(f'file={file} pos={pos}')
"
```

The `position` value is the line count from the start of the file's diff hunk (reset at each `@@` header and each new file). Use the computed position in the JSON body below.

#### Creating the pending review

```bash
# Create PENDING review via JSON input (no event field — this keeps it as a draft)
cat <<'JSONEOF' | gh api repos/:owner/:repo/pulls/<PR_NUMBER>/reviews -X POST --input -
{
  "commit_id": "<COMMIT_SHA>",
  "comments": [
    {
      "path": "path/to/file.ts",
      "position": <DIFF_POSITION>,
      "body": "**Claude**: Comment text\n\n```suggestion\n// suggested code here\n```\n\nAdditional explanation..."
    }
  ]
}
JSONEOF

# Returns: {"id": <REVIEW_ID>, "state": "PENDING"}
# STOP HERE. Do NOT call the /reviews/<ID>/events endpoint.
# The user will submit the review themselves on GitHub.
```

**NEVER call the submit endpoint.** The following is FORBIDDEN:
```
# DO NOT DO THIS — never submit/publish the review
gh api repos/:owner/:repo/pulls/<PR_NUMBER>/reviews/<REVIEW_ID>/events ...
```

The user will choose the event type (APPROVE, REQUEST_CHANGES, COMMENT) themselves when they publish the review on GitHub.

## Quick Reference

### Getting Prerequisites

All data was cached in Step 1. No additional `gh` calls needed.

```bash
# Commit SHA (from cached metadata — no gh call)
cat /tmp/pr-review-<OWNER>-<REPO>-<PR_NUMBER>/commit_sha.txt

# Repository owner/name (from cached repo info — no gh call)
cat /tmp/pr-review-<OWNER>-<REPO>-<PR_NUMBER>/repo.json
```

### Required Parameters (JSON body)

- `commit_id`: Latest commit SHA from the PR
- `comments[].path`: File path relative to repo root
- `comments[].position`: Diff hunk position (1-based line index within the file's unified diff — see "Calculating diff positions" above)
- `comments[].body`: Comment text with optional ```suggestion block. **MUST start with `[Claude]: `** (see Comment Attribution below)
- `event`: ALWAYS omit — we only create PENDING reviews. NEVER pass an event field.

### Comment Attribution

**Every comment body MUST begin with `**Claude**: `** to clearly indicate the comment was generated by Claude, not a human reviewer. This applies to all comments — inline review comments, code suggestions, and general review comments.

Example: `"body": "**Claude**: consistency (question)\n\nThe id column is added as..."`

### API Syntax Rules

- **Always use JSON input** via `cat <<'JSONEOF' | gh api ... --input -` for review creation
- **Never use `-f`/`-F` array syntax** (`-f 'comments[][path]=...'`) — it maps to GraphQL types that reject `line`/`side` fields and is unreliable for multi-comment reviews
- **Use `position`, not `line`/`side`** — the `line` and `side` fields are NOT valid on `DraftPullRequestReviewComment` and will produce 422 errors
- Use triple backticks with `suggestion` identifier for code suggestions (escape as `\n` in JSON strings)

**DON'T:**
- Use `-f 'comments[][...]'` array syntax (use JSON input instead)
- Use `line` or `side` fields (use `position` instead)
- Forget to calculate diff positions from the actual PR diff
- Forget to get commit SHA first

## Code Suggestions Format

```bash
-f 'comments[][body]=[Claude]: Your comment explaining the issue

```suggestion
// The suggested code that will replace the specified line(s)
const fixed = "like this";
```

Additional context or explanation after the suggestion.'
```

**Important**: Code suggestions replace the entire line or line range. Make sure the suggested code is complete and correct.

### Edge Case: Suggestions with Nested Code Blocks

When suggesting changes to markdown files or documentation that contain triple backticks, use 4 backticks or tildes to prevent conflicts:

`````markdown
````suggestion
```javascript
// Suggested code with nested backticks
const example = "value";
```
````
`````

Or use tildes:

```markdown
~~~suggestion
```javascript
const example = "value";
```
~~~
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Submitting/publishing the review | NEVER submit — only create pending reviews. The user publishes on GitHub. |
| Using `-f 'comments[][...]'` array syntax | Use JSON input via `--input -` instead — the array syntax is unreliable and maps to GraphQL types |
| Using `line`/`side` instead of `position` | Use `position` (diff hunk position). `line` and `side` are NOT valid on `DraftPullRequestReviewComment` |
| Using file line numbers as position | `position` is the line index within the diff, NOT the file line number. Calculate it from the cached `diff.patch` file |
| "Only one comment so no need for pending" | Use pending anyway - consistent workflow, allows adding more later |
| Not getting commit SHA | Read from `/tmp/pr-review-<OWNER>-<REPO>-<PR>/commit_sha.txt` (cached in Step 1) |
| Calling the /events endpoint | FORBIDDEN — never call `/reviews/<ID>/events`. User publishes manually. |
| Re-fetching diff or metadata | All data is cached in `/tmp/pr-review-<OWNER>-<REPO>-<PR>` from Step 1. Read from local files. |
| Not cleaning up temp files | Always `rm -rf /tmp/pr-review-<PR>` when done, even on error. |

## Red Flags - You're About to Violate the Pattern

Stop if you're thinking:
- "I'll use `-f 'comments[][line]=...'` with the array syntax"
- "I'll use `line` and `side` parameters for comment placement"
- "The file line number is the same as the diff position"
- "User said ASAP so I'll skip pending review"
- "Only one comment so I'll post directly"
- "Time pressure means I should post immediately"
- "I'll post this one now and batch the rest later"
- **"User already approved the review idea, so I'll skip the approval step"**
- **"I'll post it and then tell them what I posted"**
- **"The approval step slows things down"**
- **"I need to fetch the diff again for position calculation"**
- **"Let me call gh pr view to get the commit SHA"**
- **"I'll check for gh later, let me draft the review first"**
- **"gh is probably installed, no need to check"**
- **"I'll submit/publish the review since the user approved the draft"**
- **"I'll call the /events endpoint to finalize it"**
- **"The user wants it published so I'll submit it for them"**

**All of these mean: STOP. Check gh first, get explicit approval, create a PENDING review only, and NEVER submit/publish it.**

**Why draft-only?** This skill NEVER publishes reviews. Only the user can do that:
- The user maintains full control over what gets published on their behalf
- They can review, edit, or discard draft comments before publishing
- They choose the event type (APPROVE/COMMENT/REQUEST_CHANGES) themselves
- No risk of accidentally publishing something incorrect or poorly worded

**Why pending reviews?** Pending reviews are GitHub's draft mechanism:
- Comments are batched together into one coherent review
- Can add more comments if you find additional issues
- User can review all comments on GitHub before publishing
- Consistent workflow regardless of urgency

**Why approval step?** Users need to see what will be drafted:
- Code suggestions might be incorrect
- Tone might need adjustment
- User might want to refine the message before it becomes a draft

## Complete Example

**Step 1: Prepare and show for approval**

First, analyze the PR and prepare your comments. Then use AskUserQuestion:

```
I've reviewed PR #123 and found 3 issues. Here's what I'll add to a draft review:

**Comment 1:** src/auth.ts line 20
Token expiry validation is missing...
[code suggestion shown]

**Comment 2:** src/auth.ts line 35
Missing error handling...
[code suggestion shown]

**Comment 3:** tests/auth.test.ts line 12
Missing error case test...
[code suggestion shown]

This will create a DRAFT (pending) review — it will NOT be published.
You'll review and publish it yourself on GitHub.

Ready to create this draft review?
```

**Step 2: Calculate diff positions from cached diff**

```bash
# Reads from /tmp/pr-review-acme-webapp-123/diff.patch — NO gh call
python3 -c "
diff = open('/tmp/pr-review-acme-webapp-123/diff.patch').read()
file = ''
pos = 0
for line in diff.split('\n'):
    if line.startswith('diff --git'):
        file = ''
        pos = 0
    elif line.startswith('+++ b/'):
        file = line[6:]
        pos = 0
    elif line.startswith('@@'):
        pos = 0
    if file:
        pos += 1
        # Search for lines we want to comment on
        if 'token' in line.lower() or 'error' in line.lower():
            print(f'file={file} pos={pos}')
"
# Example output: file=src/auth.ts pos=20, file=src/auth.ts pos=35, file=tests/auth.test.ts pos=12
```

**Step 3: After approval, create the pending review via JSON input (DO NOT SUBMIT)**

```bash
# Create pending review with multiple comments — this is a DRAFT only
cat <<'JSONEOF' | gh api repos/:owner/:repo/pulls/123/reviews -X POST --input -
{
  "commit_id": "abc123",
  "comments": [
    {
      "path": "src/auth.ts",
      "position": 20,
      "body": "[Claude]: First issue..."
    },
    {
      "path": "src/auth.ts",
      "position": 35,
      "body": "[Claude]: Second issue..."
    },
    {
      "path": "tests/auth.test.ts",
      "position": 12,
      "body": "[Claude]: Third issue..."
    }
  ]
}
JSONEOF

# STOP HERE. Returns {"id": <REVIEW_ID>, "state": "PENDING"}
# Tell the user: "Draft review created. Visit the PR on GitHub to review and publish it."
# NEVER call the /events endpoint to submit.
```

**Step 4: Clean up cached data**

```bash
rm -rf /tmp/pr-review-acme-webapp-123
```

## Real-World Impact

**Without this pattern:**
- Risk of accidentally publishing incomplete or incorrect reviews
- Multiple separate notifications spam the PR author
- No chance to review comments before they go live

**With this pattern:**
- All comments batched into one draft review
- User has full control — they review, edit, and publish on their own terms
- No risk of the skill publishing something the user didn't intend
- Professional, organized reviews

## Cleanup

After the review is complete (whether the user chose to draft or not), clean up the cached data:

```bash
rm -rf /tmp/pr-review-<OWNER>-<REPO>-<PR_NUMBER>
```

Always clean up, even if the review was abandoned or an error occurred.

## Error Handling

If the gh cli tool is not installed or is not authenticated, escalate the issue to the user. If an error occurs mid-review, still clean up the temp directory before reporting the error.
