---
name: deployment-io
description: Deploy web services and static sites to deployment.io, read deployment logs, and manage environments via the deployment.io MCP server. Use when the user says "deploy this", "ship to staging/production", "push to deployment.io", "check deployment logs", "redeploy", or asks about deployment.io environments and runners.
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

## References

- Full tool reference: https://deployment.io/docs/coding-agents/available-mcp-tools/
- MCP server: https://api.deployment.io/v1/mcp (protocol version 2025-11-25)
- This repo: https://github.com/deployment-io/skills
