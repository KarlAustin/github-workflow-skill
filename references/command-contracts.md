# Command Contracts

Detailed input/output specifications for each workflow stage command.

## Stage Commands

### `kdaws:workflow-install`

- **Input:** None (operates on current project directory)
- **Output:** Project configured with templates, labels, CLAUDE.md section
- **Side effects:**
  - Creates `.github/ISSUE_TEMPLATE/` with feature.md, bug.md, refactor.md, config.yml
  - Creates GitHub labels: p1, p2, p3, refactor
  - Enables auto-delete head branches in repo settings
  - Appends Workflow section to CLAUDE.md
- **Idempotent:** Skips items that already exist (labels, templates) with user confirmation

### `/workflows:brainstorm #N`

- **Input:** Reads issue `#N` body for problem/goal description
- **Output:** Brainstorm document exploring approaches, decisions, trade-offs
- **Side effects:** Posts collapsible comment on issue `#N` (stage: `brainstorm`)
- **Error:** If issue not found, asks user for issue number

### `/workflows:plan #N`

- **Input:** Reads issue `#N` body + brainstorm comment (if exists)
- **Output:** Step-by-step implementation plan with file list, acceptance criteria
- **Side effects:** Posts collapsible comment on issue `#N` (stage: `plan`), writes plan file to `docs/plans/`
- **Error:** If no brainstorm found, proceeds from issue body alone (fine for simple workflow)

### `/deepen-plan`

- **Input:** Most recent plan file (local, from `docs/plans/`)
- **Output:** Enhanced plan with parallel research (best practices, edge cases, implementation details)
- **Side effects:**
  - Updates plan file in place locally
  - Posts the deepened plan as a new collapsible comment on the issue (stage: `plan`, summary: `Implementation Plan (Deepened) — YYYY-MM-DD`)
  - This ensures `/workflows:work` reads the deepened version from GitHub, not the original
- **Error:** If no plan file found, asks user for path

### `/plan_review #N`

- **Input:** Reads the most recent plan comment on issue `#N`
- **Output:** Critique with suggested revisions
- **Side effects:** Posts collapsible comment on issue `#N` (stage: `plan-review`)
- **Error:** If no plan found, prompts user to run `/workflows:plan` first

### `/workflows:work #N`

- **Input:** Reads the most recent plan comment on issue `#N`
- **Output:** Code changes on a new branch, committed and pushed as a **draft** PR
- **Side effects:**
  - Creates branch `{type}/{N}-{slug}`
  - Opens **draft** PR (`gh pr create --draft`) with `Closes #{N}` in body
  - PR title matches issue title
  - Draft status signals "work in progress, not ready for review"
- **Branch naming:** `feat/42-osm-event-sync`, `fix/87-broken-login`, `refactor/100-extract-service`
- **Error:** If branch already exists, errors with message to delete first

### `/workflows:review`

- **Input:** Reads the current PR (from current branch)
- **Output:** Multi-agent code review findings as PR comments
- **Side effects:**
  - Converts draft PR to ready-for-review via `gh pr ready` (if still in draft)
  - Posts review comments on the PR
- **Agents used:** Architecture, security, performance, patterns, simplicity reviewers (as applicable, based on PR content)

### `/resolve_pr_parallel`

- **Input:** Current PR review comments
- **Output:** Auto-resolves agreed-upon simple fixes
- **Side effects:** Commits fixes to PR branch

### `/triage`

- **Input:** Current PR review comments
- **Output:** Walks through each finding interactively with user
- **Side effects:** User decides per-finding (fix, skip, wontfix)

### `/workflows:compound #N`

- **Input:** Reads issue `#N` thread + PR diff + review comments
- **Output:** Documented lesson in `docs/solutions/{category}/`
- **Side effects:** Posts summary as collapsible comment on issue `#N` (stage: `compound`), writes solution file
- **Categorisation:** Selects the most appropriate `docs/solutions/` subdirectory based on the lesson topic

## Rework and Rejection Paths

| Situation | Action |
|-----------|--------|
| Brainstorm reveals issue should be split | Close original issue. Create child issues with "Split from #N" in body. |
| Plan review identifies wrong approach | Loop back to plan (re-run `/workflows:plan`). Post revised plan as new collapsible comment. |
| Code review rejects PR entirely | Close the PR. Post a comment explaining why. Loop back to plan stage. |
| Review finds simple fixes | Run `/resolve_pr_parallel` then re-run `/workflows:review`. Max 2 review cycles. |
| Review finds complex fixes | Run `/triage` to walk through each with user. Then one final review pass. |

## Reading Prior Stage Outputs

To find a specific stage's output on an issue:

```bash
# Get the repo owner/name
REPO=$(gh repo view --json nameWithOwner --jq '.nameWithOwner')

# Find the most recent comment for a given stage
gh api "repos/${REPO}/issues/{N}/comments" \
  --jq '[.[] | select(.body | contains("<!-- workflow-stage: {stage}"))] | last | .body'
```

Replace `{N}` with the issue number and `{stage}` with one of: `brainstorm`, `plan`, `plan-review`, `compound`.

If no matching comment is found (empty output), the stage hasn't been run yet.

## Collapsible Comment Posting

```bash
REPO=$(gh repo view --json nameWithOwner --jq '.nameWithOwner')
TODAY=$(date +%Y-%m-%d)

gh issue comment {N} --body "$(cat <<EOF
<!-- workflow-stage: {stage}, date: ${TODAY}, command: {command} -->
<details>
<summary>{emoji} {Stage Name} — ${TODAY}</summary>

{content}

---
*Generated by \`{command}\` • Based on {input source}*
</details>
EOF
)"
```
