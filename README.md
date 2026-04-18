# deployment.io Skills

SKILL.md distributions for deployment.io coding-agent integrations. Install once per agent, use across all projects.

## Available skills

| Skill | Description |
|---|---|
| [`deployment-io`](./skills/deployment-io/SKILL.md) | Deploy web services and static sites, read logs, manage environments. |

## Install

The skill is a directory you drop into your agent's skills folder. Clone once, then copy into the directory your agent reads:

```bash
git clone --depth 1 --branch v0.1.0 https://github.com/deployment-io/skills.git /tmp/dio-skills
```

Then copy to the path your agent uses:

```bash
# Claude Code / Claude Desktop
mkdir -p ~/.claude/skills && cp -r /tmp/dio-skills/skills/deployment-io ~/.claude/skills/

# Codex CLI
mkdir -p ~/.codex/skills && cp -r /tmp/dio-skills/skills/deployment-io ~/.codex/skills/

# Gemini CLI
mkdir -p ~/.gemini/skills && cp -r /tmp/dio-skills/skills/deployment-io ~/.gemini/skills/

# Cursor (Feb 2026+)
mkdir -p ~/.cursor/skills && cp -r /tmp/dio-skills/skills/deployment-io ~/.cursor/skills/
```

Restart your agent after copying so it picks up the new skill.

## Required: connect the deployment.io MCP server

This skill calls tools served by the deployment.io MCP server. The skill itself does not authenticate — you must connect the MCP server separately in your agent. See https://deployment.io/docs/coding-agents/mcp-configuration/ for one-click OAuth flows.

## Update

Re-clone with the new tag and re-copy to your agent's skills directory:

```bash
git clone --depth 1 --branch <new-tag> https://github.com/deployment-io/skills.git /tmp/dio-skills
cp -r /tmp/dio-skills/skills/deployment-io ~/.claude/skills/  # or your agent's path
```

## Versioning

Semver via git tags. Breaking changes to `SKILL.md` guidance bump the minor until 1.0, then major.

## License

MIT.
