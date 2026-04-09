---
title: "Summary: Claude Code Internals -- Skills, Agents, Hooks, and the Plugin System"
---

> **Full notes:** [[notes/AI-Tooling/claude-code-internals|Claude Code Internals -- Skills, Agents, Hooks, and the Plugin System -->]]

## Key Concepts

### Skills vs Agents
- **Skill** = markdown knowledge injection (SKILL.md) -- tells the LLM *how to think*
- **Agent** = autonomous sub-session with tools -- *does the work*
- **Critical constraint: skills CANNOT invoke agents** (no Agent tool in allowed-tools)
- Skills specify `agent:` in frontmatter to delegate execution
- Skills have `context: fork` (isolated) or `context: inherit` (shared)
- Analogy: skill = brain (knowledge), agent = hands (action)

### Hooks
- **Shell commands** triggered by lifecycle events
- Two sources: **user hooks** (settings.json) and **plugin hooks** (hooks/hooks.json)
- Events: `SessionStart`, `PreToolUse`, `PostToolUse`, `Notification`, `Stop`
- `SessionStart` hooks inject `additionalContext` into system prompt
- `async: false` = blocking (Claude waits), `async: true` = fire-and-forget
- Output format differs by platform (Claude Code vs Cursor vs Copilot CLI)

### Plugins
- Git repo bundling skills, agents, hooks, and commands
- Manifest: `.claude-plugin/plugin.json` (name, version, agent paths, commands path)
- Plugins can depend on other plugins via `.claude/settings.json`
- Install != enable -- `enabledPlugins` in settings.json controls activation
- Key format: `plugin-name@marketplace-name`
- Cached at `~/.claude/plugins/cache/<marketplace>/<plugin>/<version>/`

### Marketplaces
- Git repo with `.claude-plugin/marketplace.json` catalog
- **Official**: `anthropics/claude-plugins-official` (~50 plugins)
- **Self-hosted**: register via `extraKnownMarketplaces` in settings.json
- Plugin sources: relative path, git URL, GitHub shorthand, git-subdir
- Version tracking by git commit SHA

### Session Lifecycle
- Load settings -> load plugins -> fire SessionStart hooks -> load CLAUDE.md -> register skills -> ready
- Skill invocation: load SKILL.md -> fork context -> spawn designated agent -> return result
- Agent spawning: agent can use Agent tool to dispatch sub-agents with constrained tools

## Quick Reference

```
~/.claude/
  settings.json              # model, enabledPlugins, extraKnownMarketplaces
  settings.local.json        # permissions / tool allowlists
  agents/*.md                # user-defined agents
  skills/*/SKILL.md          # user-defined skills
  plugins/
    installed_plugins.json   # version + SHA tracking
    known_marketplaces.json  # marketplace registry
    marketplaces/            # cloned marketplace repos
    cache/                   # installed plugin files

Plugin structure:
  .claude-plugin/plugin.json   # manifest
  agents/*.md                  # agent definitions
  skills/*/SKILL.md            # skill definitions
  commands/*.md                # slash commands
  hooks/hooks.json             # lifecycle hooks
```

```
Skill -----> Agent -----> Sub-Agent
(knowledge)  (execution)  (delegation)
 cannot        can          can
 spawn         spawn        spawn
 agents        agents       agents
```

| Component | File | Can Use Tools | Can Spawn Agents |
|-----------|------|---------------|------------------|
| Skill | SKILL.md | No (through agent only) | No |
| Agent | *.md | Yes | Yes (if Agent in tools) |
| Hook | hooks.json + script | Shell only | No |
| Command | *.md | Through workflow | Through workflow |
