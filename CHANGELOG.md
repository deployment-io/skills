# Changelog

All notable changes to this repo are documented here. Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/); versions follow [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.2.0] — 2026-05-19

### Added

- Tasks support: `create_task` and `get_task_status` trigger phrases in the SKILL.md frontmatter description ("have an agent fix X", "open a PR for Y", "create a task to migrate Z").
- New "Creating Tasks (AI coding-agent PRs)" section in `SKILL.md` — cost-awareness guidance, parameter detection for `create_task`, polling cadence (15s), and terminal-state interpretation (Succeeded / NoChanges / Failed / Cancelled).

## [0.1.0] — 2026-04-17

### Added

- Initial release of `skills/deployment-io/SKILL.md` — the SKILL.md-compliant runbook for the deployment.io MCP server.
- Parameter-detection tables: framework → `publish_directory` for static sites; port / Dockerfile / health-check heuristics for web services.
- Git pre-flight runbook (fetch → log unpushed → rev-parse for `commit_hash`).
- Diagnose-bad-deploy workflow using `get_deployment_logs` + `get_job_status`.
- Approval-flow explanation covering the `⏳ Approval required` response and `get_approval_status` polling.
- Per-agent install paths documented in README: `.claude/skills/` (Claude Code + Desktop), `.codex/skills/`, `.gemini/skills/`, `.cursor/skills/`.
