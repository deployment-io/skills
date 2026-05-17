---
name: deployment-io
description: Deploy web services and static sites to deployment.io, read deployment logs, manage environments, AND create AI coding-agent Tasks that open PRs in your repos via the deployment.io MCP server. Use when the user says "deploy this", "ship to staging/production", "push to deployment.io", "check deployment logs", "redeploy", "have an agent fix X", "open a PR for Y", "create a task to migrate Z", or asks about deployment.io environments, runners, or Tasks.
license: MIT
---

# deployment.io

Deploy and operate apps on deployment.io from any SKILL.md-compliant coding agent (Claude Code, Codex, Cursor, Gemini CLI, Copilot, Antigravity, JetBrains Junie, and more).

## Prerequisites

This skill assumes the deployment.io MCP server is connected to your agent. If `deploy_static_site`, `deploy_web_service`, `list_environments`, etc. are not available as tools:

- Sign in at https://deployment.io and follow the agent-connection guide: https://deployment.io/docs/coding-agents/mcp-configuration/
- The MCP server uses OAuth 2.1 + PKCE; no tokens to copy-paste.

## When to use this skill

- "Deploy the current branch to staging."
- "Ship this to production."
- "Why is the API returning 500? Check logs."
- "Create a new environment called preview-42."
- "Redeploy service X."

## Parameter detection: static sites

For `deploy_static_site`:

- **repository_url**: `git remote get-url origin`
- **branch**: `git branch --show-current` (or the branch the user names)
- **build_command**: `package.json` `scripts.build` (check the branch with `git show <branch>:package.json` if not checked out)
- **publish_directory**: pick from the framework table below, or check framework config for overrides

| Framework | publish_directory |
|---|---|
| Vite, Svelte, Astro | `dist` |
| Create React App | `build` |
| Next.js static export | `out` |
| Hugo | `public` |
| Jekyll | `_site` |

- **is_spa**: `true` for apps that import `react-router-dom` or `vue-router`; `false` for static site generators (Hugo, Jekyll, Astro, Next.js export).

## Parameter detection: web services

For `deploy_web_service`:

- **port**: the port the service listens on *inside* the container. Find it in one of:
  - Dockerfile `EXPOSE` directive or `ENV PORT=...`
  - App listen call: `app.listen(3000)`, `http.ListenAndServe(":8080")`, `uvicorn.run(port=8000)`
  - `process.env.PORT`, `os.Getenv("PORT")` usage
- **dockerfile_path**: omit unless the Dockerfile is not at the repo root (e.g. `docker/Dockerfile`).
- **health_check_path**: search the codebase for common routes — `/health`, `/healthz`, `/ping`, `/ready`, `/status`. Check Dockerfile for a `HEALTHCHECK` directive. If none found, omit (defaults to `/`).
- **Dockerfile must exist on the deploy branch.** For a non-checked-out branch: `git show <branch>:Dockerfile`. If missing and you're creating one, commit and push before calling the tool.

## Git pre-flight (before any `deploy_*` call)

The runner clones from the remote, so only *pushed* commits deploy.

1. **Fetch the remote branch.**
   ```
   git fetch origin <branch>
   ```
2. **Check for unpushed commits:**
   ```
   git log origin/<branch>..HEAD
   ```
   Push anything listed. Do not proceed with unpushed work.
3. **Capture the exact SHA** to pass as `commit_hash`:
   ```
   git rev-parse origin/<branch>
   ```

## Write a `changes_summary`

Every `deploy_*` takes a `changes_summary`. Human reviewers see it on approval requests; keep it one-line and plain-English (don't paste raw `git log`). Example: "Hotfix for null-pointer in /api/users endpoint when email is missing."

## Resolve `environment_id`

Always call `list_environments` first. If none match the target, offer to create one via `create_environment` (which itself calls `list_runners` first).

## After an async call

`deploy_*` and `get_deployment_logs` are async — they return a structured `accepted` result containing `job_id`, `poll_tool`, `poll_args`, and `poll_interval`. Follow those fields: poll `get_job_status` at `poll_interval` until `is_done: true`. Log output lands in `output`.

## Approval flow

Write tools on protected environments return a message starting `"⏳ Approval required"` with a `request_id`. Surface the approval URL to the user. Then poll `get_approval_status` with the `request_id`. **Do not retry the original deploy call** — the server runs it automatically on approval.

## Diagnosing a bad deploy

1. `list_deployments <environment_id>` → find the `deployment_id`.
2. `get_deployment_logs` with a 5-minute window around the failure → returns a `job_id`.
3. Poll `get_job_status` until `is_done`; read `output`.
4. For **build** (not runtime) failures, the original `deploy_*` job's `get_job_status` already contains build logs; dashboard URL in the message body.

## Creating Tasks (AI coding-agent PRs)

A **Task** is a unit of engineering work that runs on the user's AWS runner: deployment.io spawns Claude Code inside an agentbox container, the agent edits one or more repos, and the runner opens a PR per repo. Use Tasks when the user wants the agent to make the change for them rather than deploy code they've already written.

### Prerequisites (the tool will error clearly if missing)

- A **non-SaaS runner** in the user's AWS account. The SaaS runner is not supported for Tasks (cost + credential model). If missing, the tool surfaces a link to `/org-settings/runners`.
- **Anthropic credentials** saved at the org level (API key or Bedrock IAM). If missing, the tool surfaces a link to `/org-settings/tasks`.
- **Repos connected with write scope** (GitHub App `contents:write` + `pull_requests:write`, or GitLab/Bitbucket equivalents). Read-only installations cannot open PRs.

### Cost awareness

Tasks burn the user's Anthropic/Bedrock budget — typical runs use 50k–500k tokens, larger refactors more. Do not call `create_task` speculatively. Confirm with the user before calling, especially when wrapping the call inside a longer agent workflow ("I'll have deployment.io fix this for you, is that OK?").

### Parameter detection: `create_task`

- **title**: short noun phrase (≤ 200 chars), e.g. "Migrate users service to TypeScript".
- **description**: the actual prompt the agent will receive. Be specific — list constraints, files, acceptance criteria. Quality of description drives quality of output. Do not paste raw chat transcripts.
- **repositories**: one entry per repo the agent should touch. For each:
  - `repository_url`: `git remote get-url origin` (or the URL the user names).
  - `branch`: the base branch the Task should branch off. Use `git symbolic-ref refs/remotes/origin/HEAD | sed 's@^refs/remotes/origin/@@'` to detect the default, or the branch the user names. Push any local commits to this branch first — the runner clones from the remote.
- **model** (optional): omit unless the user specifies. Server falls back to the org's configured default (typically `"sonnet"`). Explicit values: `"haiku"` (cheaper, simpler tasks), `"sonnet"` (balanced), `"opus"` (harder refactors).
- **max_turns** (optional): omit unless the user has a reason. Server default is 30, which fits most tasks. Raise to 50–80 explicitly for large refactors; lower to 10–15 for tightly-scoped edits.
- **token_budget** (optional): don't set this. The dashboard never sends it either — it's a future-proofing field, and 0 means uncapped (server default). Only pass a value if the user explicitly asks for a cost ceiling.

### Polling

`create_task` returns an `accepted` envelope with `task_id`, `poll_tool: "get_task_status"`, `poll_args`, `poll_interval: "15s"`. Poll until `is_done: true`. Tasks take minutes to hours; don't poll faster than 15s.

### Reading status

`get_task_status` returns the Task's rolled-up state, per-repo PR URLs (populated once the runner opens them), and the last 10 log lines from the most recent agent run. Full agent transcripts live at the `url` field — share that with the user for deep inspection.

Terminal states:
- `Succeeded` — PRs opened (check `repositories[].pr_url`). Surface each PR link to the user for review.
- `NoChanges` — agent ran but produced no diffs. Usually means the description was satisfied by existing code, or the agent couldn't find a path. Read `latest_run_logs` and surface to the user.
- `Failed` — read `latest_run_logs` and the dashboard `url` to diagnose. Common causes: agent ran out of turns, build/test failures during agent's self-checks, agentbox container error.
- `Cancelled` — user (or admin) stopped the Task.

## References

- Full tool reference: https://deployment.io/docs/coding-agents/available-mcp-tools/
- MCP server: https://api.deployment.io/v1/mcp (protocol version 2025-11-25)
- This repo: https://github.com/deployment-io/skills
