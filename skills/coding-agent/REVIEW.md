# Code Review: `skills/coding-agent/SKILL.md`

> Reviewed by Copilot coding agent · March 2026
>
> **Note:** This review was written in response to a request to post findings as a GitHub issue comment. GitHub API access is unavailable from the sandbox, so the review is recorded here instead.

---

## Overview

`coding-agent` is a skill that teaches an AI assistant how to delegate coding tasks to one of four sub-agents — **Codex CLI**, **Claude Code**, **OpenCode**, and **Pi** — using bash calls with optional background/PTY modes. It covers:

- When and how to use PTY vs. non-PTY execution per agent
- One-shot vs. background task patterns
- Per-agent CLI reference (flags, modes)
- Parallel PR reviews and parallel issue-fixing with git worktrees
- Progress-update etiquette and an auto-notify (wake-event) pattern
- A rules checklist and a dated "Learnings" retrospective

Overall the skill is well-structured, readable, and genuinely useful. The concerns below range from minor formatting issues to logic inconsistencies.

---

## ✅ Strengths

1. **Clear PTY guidance upfront** — the `⚠️ PTY Mode` header immediately explains the key behavioral difference between agents.
2. **Reference tables** — the bash-tool parameter table and process-action table are concise and scannable.
3. **Good use of code blocks** — every pattern is shown with an executable example.
4. **Concrete worktree workflow** — the parallel issue-fixing section is practical and covers the full lifecycle (create → run → PR → cleanup).
5. **Auto-notify pattern** — the `openclaw system event` wake trigger is a genuinely clever UX detail worth keeping.
6. **Rule deduplication** — rules #8 and #9 prevent known failure modes (wandering agent, live-repo mutations).

---

## 🔴 Bugs / Correctness Issues

### 1. Invalid JSON in YAML frontmatter (trailing comma)

```yaml
metadata:
  {
    "openclaw": { "emoji": "🧩", "requires": { "anyBins": ["claude", "codex", "opencode", "pi"] } },
  }
```

The trailing comma after the closing `}` is invalid JSON. Most JSON parsers will reject this. It should be:

```yaml
metadata:
  {
    "openclaw": { "emoji": "🧩", "requires": { "anyBins": ["claude", "codex", "opencode", "pi"] } }
  }
```

### 2. `codex --yolo exec` flag order (auto-notify section)

The auto-notify example shows:

```bash
codex --yolo exec 'Build a REST API for todos. ...'
```

But every other usage in the document separates the subcommand and the flag:

- `codex exec --full-auto '...'` (building section)
- `codex --yolo 'Refactor ...'` (background example)

`codex --yolo exec` mixes both styles. `exec` is a subcommand, not a flag, so it needs to come before `--yolo` or the CLI may not recognise it. It should be either:

```bash
codex exec --yolo 'Build a REST API for todos. ...'
```

---

## 🟡 Inconsistencies

### 3. Rule #6 contradicts the reviewing section

> **Rule #6:** `vanilla for reviewing` — no special flags needed

But the reviewing code blocks use:

```bash
codex review --base origin/main
```

`--base origin/main` is a flag. The rule should either say "no approval/auto flags needed" (to distinguish from `--full-auto` / `--yolo`) or be reworded to avoid the misleading "no special flags" claim.

### 4. Inconsistent `exec` usage in background examples

| Example location | Command form |
|---|---|
| Building section | `codex exec --full-auto 'Build a snake game'` |
| Background refactor | `codex --yolo 'Refactor the auth module'` ← no `exec` |
| Batch PR review | `codex exec 'Review PR #86...'` |
| Parallel issue fix | `codex --yolo 'Fix issue #78...'` ← no `exec` |

Whether `exec` is required for background/non-interactive calls is unclear. The document should pick one canonical form or explicitly note when `exec` can be omitted.

---

## 🟠 Documentation Quality Issues

### 5. `gpt-5.2-codex` model name

> **Model:** `gpt-5.2-codex` is the default (set in `~/.codex/config.toml`)

`gpt-5.2-codex` is not a publicly documented OpenAI model name. This may be a placeholder, a typo, or an internal alias. If it is the actual model name, a brief note explaining it would help readers who try to reproduce the setup. If it is fictional, it should be removed or corrected.

### 6. `description` field is very long

The YAML `description` value is a single 300+ character run-on string. Per the `skill-creator` skill spec, the description is the **primary trigger mechanism** — evaluated frequently. An overly long description increases token cost and may reduce trigger precision. Consider splitting the "NOT for" items or separating positive and negative triggers.

---

## 🔵 Minor / Portability

### 7. "Skippy" is an org-specific persona name

> This triggers an immediate wake event — **Skippy gets pinged in seconds**, not 10 minutes.

"Skippy" is an internal bot/persona name with no explanation. First-time readers won't know who Skippy is. A generic term like "the assistant" or "your OpenClaw agent" would be clearer.

### 8. Rule #9 contains an org-specific path

> **NEVER checkout branches in `~/Projects/openclaw/`** — that's the LIVE OpenClaw instance!

This rule is meaningful only to OpenClaw maintainers running the agent against their personal dev machine. For any other user the path `~/Projects/openclaw/` is irrelevant or misleading. Consider generalising: "Never check out branches in the directory where your live OpenClaw instance runs."

### 9. Dated "Learnings" section may go stale

The `## Learnings (Jan 2026)` section is a snapshot. As agents evolve, items like "PTY is essential" may become outdated without anyone updating the skill. Options:
- Remove the date heading and treat these as permanent guidelines
- Fold each item into the relevant section above
- Add a note to review when upgrading agent tools

### 10. `trash` is macOS-only in cleanup step

The PR review section uses `trash $REVIEW_DIR` as a cleanup comment. On Linux the equivalent is `rm -rf "$REVIEW_DIR"`. This is a minor portability issue.

### 11. Quick Start section has a flow gap

The section jumps from a combined `SCRATCH=$(mktemp -d) && cd $SCRATCH && git init && codex exec ...` to `bash pty:true workdir:~/Projects/myproject ...` without a bridging explanation. A one-line transition would clarify when to use each form.

---

## Summary Table

| # | Severity | Category | Description |
|---|---|---|---|
| 1 | 🔴 Bug | Syntax | Trailing comma in JSON frontmatter metadata |
| 2 | 🔴 Bug | Correctness | `codex --yolo exec` subcommand/flag order |
| 3 | 🟡 Inconsistency | Logic | Rule #6 contradicts review code blocks |
| 4 | 🟡 Inconsistency | Style | Mixed `exec` / no-`exec` command forms |
| 5 | 🟠 Quality | Accuracy | Unrecognised model name `gpt-5.2-codex` |
| 6 | 🟠 Quality | Tokens | Overlong `description` field |
| 7 | 🔵 Minor | Clarity | Persona name "Skippy" unexplained |
| 8 | 🔵 Minor | Portability | Org-specific path in Rule #9 |
| 9 | 🔵 Minor | Maintenance | Dated "Learnings" section may go stale |
| 10 | 🔵 Minor | Portability | `trash` is macOS-only in cleanup step |
| 11 | 🔵 Minor | Clarity | Flow gap in Quick Start section |
