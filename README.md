# GitHub PR Review Skill for Claude Code

> **Fork notice:** This is a fork of [aidankinzett/claude-git-pr-skill](https://github.com/aidankinzett/claude-git-pr-skill). See [what changed](#whats-changed-in-this-fork) below.

A Claude Code skill for full-lifecycle GitHub pull request reviews — from automated multi-agent analysis to posting batched review comments via the `gh` CLI.

## What's Changed in This Fork

The original skill focused on the **posting workflow** (pending reviews, code suggestions, approval flow). This fork adds a **multi-faceted analysis phase** that runs before posting:

- **Three parallel review agents** — Architecture review, anti-pattern detection (Python), and change summary run concurrently as Explore sub-agents
- **Architecture review** — Evaluates separation of concerns, encapsulation, design intent, data flow, and consistency with existing codebase patterns
- **Anti-pattern detection** — Flags bare dicts, error suppression, missing types, unjustified type-ignore/noqa, and f-string logging in Python code
- **Change summary** — Factual, non-evaluative description of existing state, what changed, and implementation approach
- **Consolidated report** — Aggregates all findings into a structured table with a suggested verdict (APPROVE/COMMENT/REQUEST_CHANGES)
- **Interactive post-analysis flow** — Choose to post all findings, select specific ones, or skip posting entirely
- **Large PR handling** — Agents prioritize architecturally significant files for PRs with 500+ changed lines
- **Simplified error handling** — Removed upfront gh CLI prerequisite check in favor of escalating to the user on failure

See [CHANGELOG.md](CHANGELOG.md) for the full history.

## What This Skill Does

This skill teaches Claude to:
- **Analyze PRs with parallel sub-agents** for architecture, anti-patterns, and change summarization
- **Always use pending reviews** to batch comments (even under time pressure)
- **Create code suggestions** using the ```suggestion syntax
- **Choose the right event type** (COMMENT, APPROVE, or REQUEST_CHANGES)
- **Use correct `gh api` syntax** with proper quoting and flags

## Installation

### Option 1: Plugin Marketplace (Recommended)

Install directly from the marketplace using Claude Code:

```bash
# Add this marketplace
/plugin marketplace add joshelser/claude-git-pr-skill

# Install the plugin
/plugin install github-pr-review

# Verify installation
/plugin list
```

**For team-wide installation**, add to `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": [
    {
      "name": "github-pr-skills",
      "source": {
        "source": "github",
        "repo": "joshelser/claude-git-pr-skill"
      }
    }
  ],
  "plugins": {
    "github-pr-review": {
      "enabled": true
    }
  }
}
```

### Option 2: Manual Copy

Copy the skill directly to your skills directory:

```bash
# Personal (all projects)
cp -r github-pr-review/skills/github-pr-review ~/.claude/skills/

# Project-specific
cp -r github-pr-review/skills/github-pr-review .claude/skills/
```

## Usage

Once installed, Claude will automatically use this skill when you ask it to review pull requests. For example:

```
You: "Review PR #123 and suggest improvements"
Claude: *Uses github-pr-review skill to create pending review with batched comments*
```

## What Makes This Skill Different

**Without this skill**, Claude might:
- Post comments immediately without showing you first
- Skip pending reviews under time pressure
- Use incorrect `gh api` syntax
- Choose the wrong event type
- Post comments you didn't approve

**With this skill**, Claude will:
- ✅ **Show you exactly what will be posted** before posting
- ✅ **Ask for explicit approval** using yes/no questions
- ✅ Always create pending reviews first
- ✅ Batch all comments together
- ✅ Use code suggestions with ```suggestion blocks
- ✅ Choose appropriate event types (APPROVE for minor suggestions, REQUEST_CHANGES for blocking issues)
- ✅ Use correct syntax (single quotes around `comments[][]` parameters)

## Workflow Pattern

The skill enforces this workflow:

**1. Draft → 2. Show & Approve → 3. Post**

### Step 1: Draft Review
Claude analyzes the PR and prepares comments with code suggestions.

### Step 2: Show & Get Approval
Claude shows you EXACTLY what will be posted:
- Each comment with file and line number
- Code suggestions formatted
- Event type (APPROVE/REQUEST_CHANGES/COMMENT)
- Overall review message

You review and approve (or request changes).

### Step 3: Post Review (Technical)

After your approval, Claude posts using this pattern:

```bash
# Step 1: Create PENDING review with all comments
gh api repos/:owner/:repo/pulls/123/reviews \
  -X POST \
  -f commit_id="<SHA>" \
  -f 'comments[][path]=file.ts' \
  -F 'comments[][line]=42' \
  -f 'comments[][side]=RIGHT' \
  -f 'comments[][body]=Comment with ```suggestion block...' \
  --jq '{id, state}'

# Step 2: Submit when ready
gh api repos/:owner/:repo/pulls/123/reviews/<REVIEW_ID>/events \
  -X POST \
  -f event="APPROVE" \
  -f body="Overall review message"
```

## Benefits

- **Consistent workflow** across all PR reviews
- **Professional batched comments** instead of scattered notifications
- **Correct syntax** every time
- **Better decision making** on event types
- **Time-tested pattern** that works under pressure

## Prerequisites

- Claude Code
- GitHub CLI (`gh`) installed and authenticated

## License

MIT (or specify your license)

## Repository Structure

```
.claude-plugin/
  marketplace.json          # Plugin marketplace definition
github-pr-review/           # Plugin root
  skills/                   # Skills directory
    github-pr-review/       # The skill
      SKILL.md              # Skill definition
CHANGELOG.md                # Version history and changes
```

## Versioning

This skill follows [Semantic Versioning](https://semver.org/):
- **Current version:** 1.0.0
- **Version location:** `.claude-plugin/marketplace.json`
- **Change history:** See [CHANGELOG.md](CHANGELOG.md)

### Updating the Version

When making changes:

1. **Update CHANGELOG.md** - Add entries under `[Unreleased]`
2. **When releasing:**
   - Move unreleased items to new version section in CHANGELOG.md
   - Update `version` in `.claude-plugin/marketplace.json`
   - Create git tag: `git tag -a v1.0.1 -m "Release v1.0.1"`
   - Push with tags: `git push --tags`

**Version numbers:**
- **PATCH** (1.0.1) - Bug fixes, typos, small improvements
- **MINOR** (1.1.0) - New features, backwards compatible
- **MAJOR** (2.0.0) - Breaking changes to skill behavior

## Contributing

This skill was created using Test-Driven Development for skills:
1. Baseline tests identified violations (posting immediately under time pressure, no user approval)
2. Skill was written to address those violations
3. Tests verified the skill works correctly
4. Edge cases were tested and handled
5. Approval workflow added after user feedback

If you find edge cases where the skill doesn't work correctly, please open an issue!

## Development

To test changes locally:

```bash
# Symlink for testing
ln -s $(pwd)/github-pr-review/skills/github-pr-review ~/.claude/skills/github-pr-review

# Or use the plugin marketplace locally
/plugin marketplace add file://$(pwd)
/plugin install github-pr-review
```
