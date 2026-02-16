---
description: >
  GitHub Issue-driven development workflow. Orchestrates full and simple pipelines.
  Detects code-creation intent and suggests the appropriate workflow.
  Commands: /workflows:full, /workflows:simple, kdaws:workflow-install.
  Trigger: "work on issue", "start feature", "fix bug", describing new functionality or bugs.
argument-hint: "#issue-number"
allowed-tools: Bash(gh *), Bash(git *), Read, Write, Edit, Task, Skill
---

# GitHub Issue-Driven Development Workflow

Every piece of work starts as a GitHub Issue and follows a defined pipeline. This skill provides orchestrator commands that chain stages automatically, and detection logic that suggests the workflow when you describe work.

## Setup

Run `kdaws:workflow-install` to set up a new project. See the [Install](#install-kdawsworkflow-install) section below.

## Quick Reference

| Command | What it does |
|---------|-------------|
| `kdaws:workflow-install` | Set up a new project with templates, labels, and CLAUDE.md section |
| `/workflows:full #N` | Run full pipeline: brainstorm → plan → (deepen?) → plan review → work → review → compound |
| `/workflows:simple #N` | Run simple pipeline: plan → work |
| Individual stages | `/workflows:brainstorm #N`, `/workflows:plan #N`, `/deepen-plan`, `/plan_review #N`, `/workflows:work #N`, `/workflows:review`, `/workflows:compound #N` |

Issue number (`#N`) is **required** for all commands except `/workflows:review` and `kdaws:workflow-install`.

## Install: `kdaws:workflow-install`

Set up a new project for the GitHub Issue-driven workflow.

### What it does

1. **Create issue templates** — Copies `feature.md`, `bug.md`, `refactor.md`, and `config.yml` to `.github/ISSUE_TEMPLATE/`
2. **Create labels** — Runs `gh label create` for `p1`, `p2`, `p3`, and `refactor`
3. **Enable auto-delete head branches** — Runs `gh api` to enable branch cleanup after merge
4. **Add Workflow section to CLAUDE.md** — Appends a ~60-line Workflow section with pipeline overview, conventions, and command reference

### Steps

1. Check if `.github/ISSUE_TEMPLATE/` already exists. If so, ask: "Issue templates already exist. Overwrite?"
2. Copy templates from the skill's `templates/` directory:
   ```bash
   SKILL_DIR=$(dirname "$0")  # or use the skill's root path
   mkdir -p .github/ISSUE_TEMPLATE
   cp "${SKILL_DIR}/templates/feature.md" .github/ISSUE_TEMPLATE/
   cp "${SKILL_DIR}/templates/bug.md" .github/ISSUE_TEMPLATE/
   cp "${SKILL_DIR}/templates/refactor.md" .github/ISSUE_TEMPLATE/
   cp "${SKILL_DIR}/templates/config.yml" .github/ISSUE_TEMPLATE/
   ```
3. Create labels (skip if they already exist):
   ```bash
   gh label create "p1" --color "d73a4a" --description "Critical priority" 2>/dev/null || true
   gh label create "p2" --color "f9a825" --description "Important priority" 2>/dev/null || true
   gh label create "p3" --color "0075ca" --description "Nice-to-have priority" 2>/dev/null || true
   gh label create "refactor" --color "cfd3d7" --description "Code restructuring" 2>/dev/null || true
   ```
4. Enable auto-delete head branches:
   ```bash
   gh api repos/{owner}/{repo} -X PATCH -f delete_branch_on_merge=true
   ```
5. Check if CLAUDE.md has a `## Workflow` section. If not, append the workflow section. If it already exists, ask: "Workflow section already exists in CLAUDE.md. Skip or overwrite?"

### CLAUDE.md Workflow Section

Append this to the project's CLAUDE.md (before `## Architecture` or at the end if no Architecture section exists):

```markdown
## Workflow

All development work is tracked via GitHub Issues. Every feature, bug, and refactor follows a defined pipeline.

### Pipeline

| Path | Stages | When |
|------|--------|------|
| **Full** | brainstorm → plan → (deepen-plan?) → plan review → work → review → compound | Multiple files, architectural decisions, new features |
| **Simple** | plan → work | Single-file changes, quick fixes (~30 min) |
| **Hotfix** | work → review → merge | P1 emergencies |

**Orchestrator commands:**
- `/workflows:full #N` — Runs full pipeline, pauses before implementation for confirmation
- `/workflows:simple #N` — Runs plan then implementation

**Individual stage commands:**
- `/workflows:brainstorm #N`, `/workflows:plan #N`, `/deepen-plan`, `/plan_review #N`
- `/workflows:work #N`, `/workflows:review`, `/workflows:compound #N`
- `/resolve_pr_parallel`, `/triage`

Issue number (`#N`) is a **required argument** for all commands except `/workflows:review`.

### Issue Conventions

- **Title prefixes:** `feat:`, `fix:`, `refactor:` (conventional commits style)
- **Labels:** Type (`feature request`, `bug`, `refactor`) + Priority (`p1`, `p2`, `p3`)
- **Templates:** `.github/ISSUE_TEMPLATE/` — feature.md, bug.md, refactor.md
- **Note:** `feat:` prefix is intentionally short (follows Conventional Commits); the label is `feature request` (long form)

### Branch & PR Conventions

- **Branch naming:** `{type}/{issue-number}-{slug}` (e.g., `feat/42-osm-event-sync`)
- **Standardise on `feat/`** (not `feature/`)
- **PR title:** Matches issue title (e.g., `feat: OSM event sync`)
- **PR body:** Summary + `Closes #N` (auto-closes issue on merge)
- **Commits:** Conventional style with issue ref: `feat: add OAuth connection (#42)`
- **Draft PRs:** `/workflows:work` creates draft PRs. `/workflows:review` converts to ready-for-review.

### Stage Outputs

Each workflow stage posts a collapsible comment on the issue thread:

```markdown
<!-- workflow-stage: plan, date: 2026-02-15, command: /workflows:plan -->
<details>
<summary>📋 Implementation Plan — 2026-02-15</summary>
[content]
</details>
```

Emoji prefixes: 🔍 Brainstorm, 📋 Plan, 🔎 Plan Review, 📚 Lessons Learned.

### Directory Status

| Directory | Status |
|-----------|--------|
| `todos/` | **Frozen** — no new files. Existing pending todos stay as-is. |
| `docs/brainstorms/` | For standalone exploration only (not tied to an issue) |
| `docs/plans/` | Plan files generated by `/workflows:plan` |
| `docs/solutions/` | Active — `/workflows:compound` writes here |
```

### After Init

Report what was set up:
```
✅ Issue templates created in .github/ISSUE_TEMPLATE/
✅ Labels created: p1, p2, p3, refactor
✅ Auto-delete head branches enabled
✅ Workflow section added to CLAUDE.md

You're ready to go! Try:
  /workflows:full #N   — Run full pipeline for an issue
  /workflows:simple #N — Quick plan → implement for a small fix
```

## Detection and Prompting

When you detect that the user is describing work that implies building or changing something, proactively suggest the workflow.

### Detection Signals

**Trigger on:**
- Explicit requests: "add a feature", "fix this bug", "refactor the...", "build a...", "implement..."
- Implicit intent: describing desired behaviour changes, pointing out broken functionality, proposing improvements

**Do NOT trigger on:**
- Research questions: "how does X work?", "what does this code do?"
- File exploration: "show me the routes", "find the controller for..."
- Configuration queries: "what version of PHP are we using?"
- Already mid-workflow on the same issue (on a `{type}/{N}-slug` branch or branch has an open PR)

### Detection Prompt

When triggered, present this prompt:

```
This looks like a [feature/bug/refactor]:

  Title: "[type]: [description inferred from user's message]"
  Labels: [appropriate type label], p2
  Workflow: [full/simple] ([reasoning])

Options:
  1. Run full workflow (/workflows:full — brainstorm → plan → review → implement → review → learn)
  2. Run simple workflow (/workflows:simple — plan → implement)
  3. Just create the issue for later
  4. Skip workflow, work directly
```

### Heuristics for Simple vs Full

**Suggest simple when:**
- Single-file change likely
- User says "quick fix", "small bug", "minor tweak"
- The fix is obvious from the description (no design decisions needed)

**Suggest full when:**
- Multiple files or new models/components
- Unclear requirements needing exploration
- Architectural decisions involved
- User describes a new feature

### After User Chooses

- **Option 1 (full):** Create the issue via `gh issue create`, then run `/workflows:full #N`
- **Option 2 (simple):** Create the issue via `gh issue create`, then run `/workflows:simple #N`
- **Option 3 (create only):** Create the issue, report the URL, stop
- **Option 4 (skip):** Proceed without workflow. Do not suggest again for the same topic.

**Priority:** Default to `p2`. If user specifies urgency, adjust accordingly.

**Title prefix:** If user's description lacks a type prefix, suggest one: "I'll create this as `feat: Add PDF export`. Correct?"

## Orchestrator: `/workflows:full #N`

Run the complete pipeline for issue `#N`.

### Steps

1. **Read issue** — `gh issue view N --json title,body,labels`
2. **Brainstorm** — Invoke `/workflows:brainstorm #N`. Post output as collapsible comment on issue.
3. **Plan** — Invoke `/workflows:plan #N`. Post output as collapsible comment on issue.
4. **Deepen plan (optional)** — Ask: "Deepen this plan with parallel research? (recommended for complex features)". If yes, invoke `/deepen-plan` on the plan file, then post the deepened plan as a new collapsible comment on the issue (stage: `plan`, replacing the previous plan as the "most recent").
5. **Plan review** — Invoke `/plan_review #N`. Post output as collapsible comment on issue.
6. **Confirm before implementation** — Ask: "Plan reviewed. Ready to start coding? (review the plan on the issue thread first)". Wait for user confirmation.
7. **Work** — Invoke `/workflows:work #N`. Creates branch and **draft** PR.
8. **Review** — Convert to ready-for-review via `gh pr ready`, then invoke `/workflows:review`. Posts review comments on PR.
9. **Resolve fixes** — If review has findings:
   - Simple fixes: ask if user wants to run `/resolve_pr_parallel`
   - Complex fixes: ask if user wants to run `/triage`
   - Then optionally re-run `/workflows:review` (max 2 review cycles)
10. **Compound** — After merge, invoke `/workflows:compound #N`. Documents lessons learned.

### Between Stages

- Post each stage's output to the GitHub issue as a collapsible comment (see format below)
- The issue thread is the shared context — each stage reads prior outputs from GitHub
- User can stop the pipeline at any point by saying "stop" or "pause"
- If any stage fails, stop and report the error

## Orchestrator: `/workflows:simple #N`

Run the quick pipeline for issue `#N`.

### Steps

1. **Read issue** — `gh issue view N --json title,body,labels`
2. **Plan** — Invoke `/workflows:plan #N`. Post output as collapsible comment on issue.
3. **Confirm before implementation** — Ask: "Plan ready. Start coding?". Wait for user confirmation.
4. **Work** — Invoke `/workflows:work #N`. Creates branch and **draft** PR.

No brainstorm, plan review, code review, or compound learning.

## Draft PRs

`/workflows:work` creates PRs as **drafts** (`gh pr create --draft`). This signals "work in progress, not ready for review" and prevents accidental early reviews.

- `/workflows:review` converts the draft to ready-for-review via `gh pr ready` before posting review comments
- Manual review is also possible — the draft status is a signal, not a gate

## Collapsible Comment Format

All stage outputs posted to issues use this format:

```markdown
<!-- workflow-stage: {stage}, date: {YYYY-MM-DD}, command: {command} -->
<details>
<summary>{emoji} {Stage Name} — {YYYY-MM-DD}</summary>

{stage output}

---
*Generated by `{command}` • Based on {input source}*
</details>
```

### Stage Reference

| Stage | `{stage}` value | Emoji | Summary Text |
|-------|----------------|-------|-------------|
| Brainstorm | `brainstorm` | 🔍 | `Brainstorm — YYYY-MM-DD` |
| Plan | `plan` | 📋 | `Implementation Plan — YYYY-MM-DD` |
| Deepened Plan | `plan` | 📋 | `Implementation Plan (Deepened) — YYYY-MM-DD` |
| Plan Review | `plan-review` | 🔎 | `Plan Review — YYYY-MM-DD` |
| Compound | `compound` | 📚 | `Lessons Learned — YYYY-MM-DD` |

### Posting Comments

```bash
gh issue comment N --body "$(cat <<'EOF'
<!-- workflow-stage: plan, date: 2026-02-15, command: /workflows:plan -->
<details>
<summary>📋 Implementation Plan — 2026-02-15</summary>

[plan content here]

---
*Generated by `/workflows:plan` • Based on issue body + brainstorm*
</details>
EOF
)"
```

### Reading Prior Stage Outputs

```bash
# Find most recent plan comment on issue #42
gh api repos/{owner}/{repo}/issues/42/comments \
  --jq '[.[] | select(.body | contains("<!-- workflow-stage: plan"))] | last | .body'
```

Use `<!-- workflow-stage: {stage} -->` as the machine-readable marker. The emoji is for human readability; parsing uses the HTML comment.

## Issue Creation

When creating issues (from detection prompt or manually):

```bash
# Feature
gh issue create --title "feat: Description" --label "feature request" --label "p2" \
  --body "## Problem\n\n[description]\n\n## Goal\n\n[goal]\n\n## Acceptance Criteria\n\n- [ ] ...\n\n**Priority:** P2"

# Bug
gh issue create --title "fix: Description" --label "bug" --label "p1" \
  --body "## Bug Description\n\n[description]\n\n## Steps to Reproduce\n\n1. ...\n\n**Priority:** P1"

# Refactor
gh issue create --title "refactor: Description" --label "refactor" --label "p3" \
  --body "## Current State\n\n[description]\n\n## Goal\n\n[goal]\n\n**Priority:** P3"
```

## Branch Naming

When `/workflows:work` creates a branch:

```
{type}/{issue-number}-{slug}
```

- `type`: `feat`, `fix`, or `refactor` (extracted from issue title prefix)
- `issue-number`: The issue number (e.g., `42`)
- `slug`: Kebab-case summary (e.g., `osm-event-sync`)

Examples:
- `feat/42-osm-event-sync`
- `fix/87-broken-login`
- `refactor/100-extract-service`

## Error Handling

- **GitHub API failures:** Fail immediately with the error message. User retries manually.
- **Auth failures:** If `gh` returns auth errors, prompt: "Run `gh auth login` to authenticate."
- **Missing issue:** If `gh issue view N` fails, error: "Issue #N not found. Check the issue number."
- **Branch conflicts:** If branch already exists, error: "Branch `feat/42-title` already exists. Delete it first or choose a different name."

## Review Cycle Limits

Maximum 2 review cycles per PR. After the second `/workflows:review`, prompt:

```
This PR has been reviewed twice. Options:
  1. Merge as-is
  2. Close PR and rework from plan stage
  3. Run one more review (not recommended)
```

This is a guideline enforced conversationally, not mechanically.

## Detailed Command Contracts

See [references/command-contracts.md](references/command-contracts.md) for detailed input/output specs for each stage command.
