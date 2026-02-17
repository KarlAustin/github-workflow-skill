# Orchestrator Commands

This file contains the step-by-step procedures for `wflow:full` and `wflow:simple`. Load this file before executing either command.

## Pre-flight Checks (step 0, run before any work begins)

1. **Verify compound-engineering plugin:**
   ```bash
   ls ~/.claude/plugins/cache/*/compound-engineering/skills/ >/dev/null 2>&1
   ```
   If missing: "compound-engineering plugin is required. Install from the every-marketplace."

2. **Verify quality plugin:**
   ```bash
   ls ~/.claude/plugins/cache/*/quality/skills/security-review/SKILL.md >/dev/null 2>&1
   ```
   If missing: "quality plugin not installed. Security review will be skipped. Install with: `/plugin install quality@kdaws`. Continue without security review? [Y/n]"
   If user says yes, skip the security review step when reached.

3. **Resolve repository context:**
   ```bash
   REPO=$(gh repo view --json nameWithOwner --jq '.nameWithOwner')
   ```

## Orchestrator: `wflow:full #N`

Run the complete pipeline for issue `#N`.

### Steps

1. **Read issue** — `gh issue view N --json title,body,labels`
2. **Brainstorm** — Invoke `/workflows:brainstorm #N`. Post output as collapsible comment on issue.
3. **Plan** — Invoke `/workflows:plan #N`. Post output as collapsible comment on issue.
4. **Deepen plan (optional)** — Ask: "Deepen this plan with parallel research? (recommended for complex features)". If yes, invoke `/deepen-plan` on the plan file, then post the deepened plan as a new collapsible comment on the issue (stage: `plan`, replacing the previous plan as the "most recent").
5. **Technical review** — Invoke `/technical_review #N`. Post output as collapsible comment on issue.
6. **Confirm before implementation** — Ask: "Plan reviewed. Ready to start coding? (review the plan on the issue thread first)". Wait for user confirmation.
7. **Work** — Invoke `/workflows:work #N`. Creates branch and **draft** PR.
8. **Review** — Convert to ready-for-review via `gh pr ready`, then invoke `/workflows:review`. Posts review comments on PR.
9. **Resolve fixes** — If review has findings, see `reference.md` for [Rework and Rejection Paths].
10. **Security Review** — If quality plugin was confirmed missing in pre-flight, skip this step and warn: "Security review skipped — quality plugin not installed." Otherwise, invoke `quality:security-review`. If blocking findings:
    - Fix findings directly on the current branch, then re-run `quality:security-review`.
    - Max 3 security review cycles. After 3rd failure, escalate to manual review.
    - If clean → proceed to compound.
11. **Compound** — After merge, invoke `/workflows:compound #N`. Documents lessons learned.

### Between Stages

- Post each stage's output to the GitHub issue as a collapsible comment (see SKILL.md for format). Exception: `quality:security-review` posts to the PR.
- The issue thread is the shared context — each stage reads prior outputs from GitHub
- User can stop the pipeline at any point by saying "stop" or "pause"
- If any stage fails, stop and report the error

## Orchestrator: `wflow:simple #N`

Run the quick pipeline for issue `#N`.

### Steps

1. **Read issue** — `gh issue view N --json title,body,labels`
2. **Plan** — Invoke `/workflows:plan #N`. Post output as collapsible comment on issue.
3. **Confirm before implementation** — Ask: "Plan ready. Start coding?". Wait for user confirmation.
4. **Work** — Invoke `/workflows:work #N`. Creates branch and **draft** PR.
5. **Security Review** — If quality plugin was confirmed missing in pre-flight, skip this step and warn: "Security review skipped — quality plugin not installed." Otherwise, invoke `quality:security-review`. Fix findings on current branch and re-run. Max 3 cycles, then escalate.

No brainstorm, technical review, code review, or compound learning.

## Non-Interactive Mode

When invoked programmatically via the Skill tool from another orchestrator (not directly by a user), use these defaults for confirmation gates:

| Gate | Default |
|------|---------|
| "Deepen this plan?" (wflow:full step 4) | Skip |
| "Ready to start coding?" (wflow:full step 6) | Proceed |
| "Plan ready. Start coding?" (wflow:simple step 3) | Proceed |
| "Continue without security review?" (pre-flight) | Yes, continue |

## Review Cycle Limits

Maximum 2 review cycles per PR. After the second `/workflows:review`, prompt:

```
This PR has been reviewed twice. Options:
  1. Merge as-is
  2. Close PR and rework from plan stage
  3. Run one more review (not recommended)
```

This is a guideline enforced conversationally, not mechanically.

## Security Review Cycle Limits

Maximum 3 security review cycles per PR. After the third `quality:security-review`:

```
Security review has failed 3 times. Remaining findings need manual review:
  1. [finding]
  2. [finding]
Fix these manually or confirm as false positives before merging.
```

This is escalation to manual review, not approval to merge with known issues.
