---
name: github-pr-review
description: Use when reviewing GitHub pull requests with gh CLI - creates pending reviews with code suggestions, batches comments, and chooses appropriate event types (COMMENT/APPROVE/REQUEST_CHANGES)
allowed-tools: Bash(gh:*)
---

# GitHub PR Review

## Overview

Full-lifecycle PR review skill: **analyze** a PR with parallel sub-agents, present findings, then optionally **post** a review via `gh api`. The analysis phase launches three read-only agents in parallel (change summary + architecture + anti-patterns) and aggregates their results into a single report. The posting phase uses pending reviews to batch comments.

**CRITICAL: Always get explicit user approval before posting any review comments.** Show exactly what will be posted and ask for yes/no confirmation using AskUserQuestion.

---

## Automated PR Analysis

Use this workflow when you need to review a PR. It fetches the diff, launches three parallel analysis agents, and presents a consolidated report.

### Step 1: Fetch PR Data

```bash
# Get PR metadata
gh pr view <PR_NUMBER> --json title,author,additions,deletions,changedFiles,commits,baseRefName,headRefName

# Get the full diff
gh pr diff <PR_NUMBER>

# Get the latest commit SHA (needed if posting later)
gh pr view <PR_NUMBER> --json commits --jq '.commits[-1].oid'
```

Store the diff text, PR title, author, additions/deletions counts, and changed file count for use in the agent prompts.

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

Apply these standards:
- Strategy over optionality (explicit paths, not feature flags for everything)
- Composition over inheritance
- Rich domain models over primitive obsession
- KISS/YAGNI — no unnecessary abstractions or premature generalization
- Single Responsibility Principle
- Accept resolved domain objects, not raw IDs — when multiple methods in a class each fetch the same entity by ID internally, the API boundary should require callers to provide the resolved object instead (reduces hidden I/O, simplifies testing, makes data flow explicit)

Output your findings as a JSON array. Each finding:
{
  "file": "path/to/file.ext",
  "line": 42,
  "category": "separation-of-concerns" | "encapsulation" | "naming" | "design-intent" | "consistency" | "complexity",
  "severity": "concern" | "suggestion" | "question",
  "description": "Clear explanation of the finding"
}

After the JSON array, answer the four questions we started with. Remember that our focus is on the application as a whole.
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

2. **What Is Changing** — Concretely list: new files/classes/functions introduced, existing code modified, code deleted, and any config or dependency changes. Use specific names (class names, function names, file paths). Keep it factual and concise.

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
  - Post all findings as PR review: Creates a pending review with all findings as comments
  - Select which findings to post: Let me choose which findings to include
  - Done, don't post: End the review without posting
```

If the user selects "Post all findings" or "Select which findings", transition to the **Review Posting Workflow** below. Map each finding to a review comment using the file path and line number from the findings table.

If the user selects "Done", end cleanly.

---

## Review Posting Workflow

Use this workflow to post review comments to GitHub. This is used either as the second phase after automated analysis, or standalone when the user wants to post a review directly.

### When to Use

- After automated analysis when user wants to post findings
- Reviewing pull requests manually
- Adding code suggestions to PRs
- Posting review comments with the gh CLI

### Core Workflow

**REQUIRED STEPS (do not skip):**

1. **Draft the review** - Analyze PR and prepare all comments
2. **Show user exactly what will be posted** - Use AskUserQuestion with yes/no
3. **Get explicit approval** - Wait for user confirmation
4. **Post the review** - Only after approval

### Approval Pattern

Before posting ANY review, use AskUserQuestion to show:
- File and line number for each comment
- Exact comment text (including code suggestions)
- Event type (APPROVE/REQUEST_CHANGES/COMMENT)
- Overall review message

**Example:**
```
Question: "Ready to post this review?"
Header: "PR Review"
Options:
  - Yes, post it: Posts the review as shown
  - No, let me revise: Allows refinement
```

### Technical Workflow

**ALWAYS use the pending review pattern, even for single comments:**

```bash
# Step 1: Create PENDING review (no event field)
gh api repos/:owner/:repo/pulls/<PR_NUMBER>/reviews \
  -X POST \
  -f commit_id="<COMMIT_SHA>" \
  -f 'comments[][path]=path/to/file.ts' \
  -F 'comments[][line]=<LINE_NUMBER>' \
  -f 'comments[][side]=RIGHT' \
  -f 'comments[][body]=Comment text

```suggestion
// suggested code here
```

Additional explanation...' \
  --jq '{id, state}'

# Returns: {"id": <REVIEW_ID>, "state": "PENDING"}

# Step 2: Submit the pending review
gh api repos/:owner/:repo/pulls/<PR_NUMBER>/reviews/<REVIEW_ID>/events \
  -X POST \
  -f event="COMMENT" \
  -f body="Optional overall review message"
```

## Event Types

Choose the appropriate event type when submitting:

| Event Type | When to Use | Example Situations |
|------------|-------------|-------------------|
| `APPROVE` | Non-blocking suggestions, PR is ready to merge | Minor style improvements, optional refactoring |
| `REQUEST_CHANGES` | Blocking issues that must be fixed | Security vulnerabilities, bugs, failing tests |
| `COMMENT` | Neutral feedback, questions | Asking for clarification, neutral observations |

## Quick Reference

### Getting Prerequisites

```bash
# Get commit SHA
gh pr view <PR_NUMBER> --json commits --jq '.commits[-1].oid'

# Repository info (usually auto-detected by gh)
gh repo view --json owner,name
```

### Required Parameters

- `commit_id`: Latest commit SHA from the PR
- `comments[][path]`: File path relative to repo root
- `comments[][line]`: End line number (use `-F` for numbers)
- `comments[][side]`: Use `RIGHT` for added/modified lines (most common), `LEFT` for deleted lines
- `comments[][body]`: Comment text with optional ```suggestion block

### Optional Parameters

- `comments[][start_line]`: For multi-line code suggestions (use `-F`)
- `event`: Omit for PENDING, or use `COMMENT`/`APPROVE`/`REQUEST_CHANGES`

### Syntax Rules

- Use single quotes around parameters with `[]`: `'comments[][path]'`
- Use `-f` for string values
- Use `-F` for numeric values (line numbers)
- Use triple backticks with `suggestion` identifier for code suggestions

**DON'T:**
- Use double quotes around `comments[][]` parameters
- Mix up `-f` and `-F` flags
- Forget to get commit SHA first

## Code Suggestions Format

```bash
-f 'comments[][body]=Your comment explaining the issue

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
| Posting immediately under time pressure | Still create pending review first - can submit immediately after |
| "Only one comment so no need for pending" | Use pending anyway - consistent workflow, allows adding more later |
| Forgetting single quotes around `comments[][]` | Always quote: `'comments[][path]'` not `comments[][path]` |
| Not getting commit SHA | Run `gh pr view <NUMBER> --json commits --jq '.commits[-1].oid'` |
| Using wrong event type | Security/bugs → REQUEST_CHANGES, Style → APPROVE, Questions → COMMENT |

## Red Flags - You're About to Violate the Pattern

Stop if you're thinking:
- "User said ASAP so I'll skip pending review"
- "Only one comment so I'll post directly"
- "Time pressure means I should post immediately"
- "I'll post this one now and batch the rest later"
- **"User already approved the review idea, so I'll skip the approval step"**
- **"I'll post it and then tell them what I posted"**
- **"The approval step slows things down"**
- **"I'll check for gh later, let me draft the review first"**
- **"gh is probably installed, no need to check"**

**All of these mean: STOP. Check gh first, get explicit approval, then use pending review.**

**Why pending reviews?** Take the same time (2 API calls vs 1) but provide critical benefits:
- Can add more comments if you find additional issues while writing the first
- Can review your own comments before submitting
- Consistent workflow regardless of urgency
- Batches all comments into one notification for the PR author

**Why approval step?** Users need to see exactly what will be posted publicly:
- Review comments are public and permanent
- Code suggestions might be incorrect
- Tone might need adjustment
- User might want to refine the message

## Complete Example with Approval

**Step 1: Draft and show for approval**

First, analyze the PR and draft your comments. Then use AskUserQuestion:

```
I've reviewed PR #123 and found 3 issues. Here's what I'll post:

**Comment 1:** src/auth.ts line 20
Token expiry validation is missing...
[code suggestion shown]

**Comment 2:** src/auth.ts line 35
Missing error handling...
[code suggestion shown]

**Comment 3:** tests/auth.test.ts line 12
Missing error case test...
[code suggestion shown]

**Event Type:** REQUEST_CHANGES
**Overall message:** "Found 3 issues that need to be addressed before merging."

Ready to post this review?
```

**Step 2: After approval, post the review**

```bash
# Create pending review with multiple comments
gh api repos/:owner/:repo/pulls/123/reviews \
  -X POST \
  -f commit_id="abc123" \
  -f 'comments[][path]=src/auth.ts' \
  -F 'comments[][line]=20' \
  -f 'comments[][side]=RIGHT' \
  -f 'comments[][body]=First issue...' \
  -f 'comments[][path]=src/auth.ts' \
  -F 'comments[][line]=35' \
  -f 'comments[][side]=RIGHT' \
  -f 'comments[][body]=Second issue...' \
  -f 'comments[][path]=tests/auth.test.ts' \
  -F 'comments[][line]=12' \
  -f 'comments[][side]=RIGHT' \
  -f 'comments[][body]=Third issue...' \
  --jq '{id, state}'

# Submit with appropriate event type
gh api repos/:owner/:repo/pulls/123/reviews/<REVIEW_ID>/events \
  -X POST \
  -f event="REQUEST_CHANGES" \
  -f body="Found 3 issues that need to be addressed before merging."
```

## Real-World Impact

**Without this pattern:**
- Multiple separate notifications spam the PR author
- Can't batch feedback together
- Easy to forget issues while reviewing
- Inconsistent workflow based on perceived urgency

**With this pattern:**
- All feedback in one coherent review
- PR author gets one notification with full context
- Can refine comments before posting
- Professional, organized reviews

## Error Handling

If the gh cli tool is not installed or is not authenticated, escalate the issue to the user.
