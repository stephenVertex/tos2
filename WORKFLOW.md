---
tracker:
  kind: linear
  project_slug: "test-opensymphony-project-68a2515d8dd7"
  active_states:
    - Todo
    - In Progress
    - Human Review
    - Merging
    - Rework
  terminal_states:
    - Done
    - Closed
    - Cancelled
    - Canceled
    - Duplicate

polling:
  interval_ms: 5000

workspace:
  root: ~/.opensymphony/workspaces

hooks:
  after_create: |
    set -e
    if [ -d .git ]; then
      if ! git log --oneline -1 >/dev/null 2>&1; then
        rm -rf .git
      fi
    fi
    if [ ! -d .git ]; then
      git clone --depth 1 'https://github.com/stephenVertex/tos2.git' .
    fi
    echo 'export BASH_SILENCE_DEPRECATION_WARNING=1' > .envrc
    echo 'export PAGER=cat' >> .envrc
    echo 'export GIT_PAGER=cat' >> .envrc
    echo 'export LESS=FRX' >> .envrc
    echo 'export GH_PAGER=cat' >> .envrc
    git config core.pager cat || true
    git config pager.branch false || true
    git config pager.log false || true
    git config pager.status false || true
    git config pager.diff false || true
  before_run: |
    git status --short
  after_run: |
    git status --short
  before_remove: |
    git status --short
  timeout_ms: 60000

agent:
  max_concurrent_agents: 4
  max_turns: 20
  max_retry_backoff_ms: 300000
  stall_timeout_ms: 300000

openhands:
  transport:
    base_url: "http://127.0.0.1:8000"
  local_server:
    enabled: true
  conversation:
    persistence_dir_relative: ".opensymphony/openhands"
    max_iterations: 500
    stuck_detection: true
    confirmation_policy:
      kind: NeverConfirm
    agent:
      kind: Agent
      llm:
        model: ${LLM_MODEL}
---

You are working on a Linear ticket `{{ issue.identifier }}`

Issue context:
Identifier: {{ issue.identifier }}
Title: {{ issue.title }}
Current status: {{ issue.state }}

Description:
{% if issue.description %}
{{ issue.description }}
{% else %}
No description provided.
{% endif %}

Instructions:

1. This is an unattended orchestration session. Never ask a human to perform follow-up actions.
2. Only stop early for a true blocker (missing required auth/permissions/secrets).
3. Final message must report completed actions and blockers only.

## Related skills

- `linear`: interact with Linear.
- `commit`: produce clean, logical commits during implementation.
- `push`: keep remote branch current and publish updates.
- `pull`: keep branch updated with latest `origin/main` before handoff.

## Status map

- `Backlog` -> out of scope for this workflow; do not modify.
- `Todo` -> queued; immediately transition to `In Progress` before active work.
- `In Progress` -> implementation actively underway.
- `Human Review` -> PR is attached and validated; waiting on human approval.
- `Merging` -> approved by human; execute the `land` skill flow.
- `Rework` -> reviewer requested changes; planning + implementation required.
- `Done` -> terminal state; no further action required.

## Step 0: Determine current ticket state and route

1. Fetch the issue by explicit ticket ID.
2. Read the current state.
3. Route to the matching flow:
   - `Backlog` -> do not modify issue content/state; stop and wait for human to move it to `Todo`.
   - `Todo` -> immediately move to `In Progress`, then ensure bootstrap workpad comment exists (create if missing), then start execution flow.
   - `In Progress` -> continue execution flow from current scratchpad comment.
   - `Human Review` -> wait and poll for decision/review updates.
   - `Merging` -> on entry, open and follow `.agents/skills/land/SKILL.md`.
   - `Rework` -> run rework flow.
   - `Done` -> do nothing and shut down.
4. For `Todo` tickets, do startup sequencing in this exact order:
   - `update_issue(..., state: "In Progress")`
   - find/create `## Agent Harness Workpad` bootstrap comment
   - only then begin analysis/planning/implementation work.

## Step 1: Start/continue execution

1.  Find or create a single persistent scratchpad comment for the issue.
2.  Reconcile the workpad before new edits.
3.  Run the `pull` skill to sync with latest `origin/main`.
4.  Implement against the plan.
5.  Run validation/tests.
6.  Attach the PR URL to the Linear issue.
7.  Update the workpad comment with final checklist status.
8.  Only then move issue to `Human Review`.

## Step 2: Human Review handling

1. When the issue is in `Human Review`, do not code or change ticket content.
2. Poll silently by default.
3. If any actionable feedback arrives, move the issue to `Rework`.
4. If approved by human, move to `Merging`.

## Step 3: Rework handling

1. Keep the existing PR and branch open.
2. Address each piece of feedback.
3. Push new commits to the same branch.
4. Move the issue back to `Human Review` once all feedback is addressed.

## Step 4: Merging handling

1. Inspect the attached PR state.
2. If the PR is already `MERGED`, move the issue directly to `Done`.
3. If the PR is still open, follow `.agents/skills/land/SKILL.md`.

## Guardrails

- Do not close an open PR for minor feedback.
- Use exactly one persistent workpad comment per issue.
- If state is terminal (`Done`), do nothing and shut down.
- Keep issue text concise, specific, and reviewer-oriented.
