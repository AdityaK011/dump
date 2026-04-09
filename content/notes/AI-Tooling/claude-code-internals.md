---
title: "Claude Code Internals -- Skills, Agents, Hooks, and the Plugin System"
---

Claude Code is not just an LLM chatbot with file access. It is a programmable agent runtime with a proper extension architecture -- skills, agents, hooks, plugins, and marketplaces -- that turns a single LLM session into a multi-agent orchestration system. Understanding these internals matters because the way you configure and extend Claude Code directly determines whether it behaves like a dumb autocomplete or a genuine engineering partner.

This note is based on hands-on exploration of the `~/.claude/` directory, the plugin cache, marketplace registries, and real-world plugin structures (including `asdd-kit` and `superpowers`). The goal is to document how it all fits together, because the official docs are thin on the architecture side.

## The Big Picture

Before diving into components, here is the full mental model:

```
                           +----------------------------+
                           |    Claude Code Runtime     |
                           |   (Main Agent / "Brain")   |
                           +----------------------------+
                                       |
              +------------------------+------------------------+
              |                        |                        |
        +-----v-----+          +------v------+          +------v------+
        |   Skills   |          |   Agents    |          |   Hooks     |
        | (knowledge |          | (sub-agents |          | (lifecycle  |
        |  injection)|          |  with tools)|          |  callbacks) |
        +-----+-----+          +------+------+          +------+------+
              |                        |                        |
              |   Cannot invoke        |   Can invoke           |  Triggered by
              |   agents               |   tools, bash,         |  events:
              |                        |   read, write          |  SessionStart,
              |   Injected into        |                        |  etc.
              |   system prompt        |   Forked context       |
              |   or loaded on         |   (isolated)           |
              |   demand via Skill     |                        |
              |   tool                 |                        |
              +------------------------+------------------------+
                                       |
                              +--------v--------+
                              |    Plugins      |
                              | (bundle skills, |
                              |  agents, hooks, |
                              |  commands)      |
                              +---------+-------+
                                        |
                              +---------v--------+
                              |  Marketplaces    |
                              | (git repos with  |
                              |  marketplace.json|
                              |  catalogs)       |
                              +------------------+
```

## Skills -- The Brain's Knowledge Library

A **skill** is a markdown file that provides domain-specific knowledge and behavioral instructions to the LLM. Skills do not execute code. They do not call tools. They are pure prompt injection -- carefully structured text that tells Claude *how* to approach a class of problems.

### Anatomy of a Skill

Every skill lives in a directory containing at minimum a `SKILL.md` file:

```
~/.claude/skills/
  write/
    SKILL.md          <-- This is the skill definition
  dump/
    SKILL.md
```

The `SKILL.md` has YAML frontmatter followed by the actual skill content:

```yaml
---
name: write
description: Write or update technical notes in the digital garden.
user-invocable: true
disable-model-invocation: false
context: fork
agent: content-writer
allowed-tools: Read, Write, Edit, Bash, Grep, Glob
argument-hint: "<topic or instruction>"
---

Write or update technical notes in the digital garden at `~/Work/dump/`.
...
```

### Key Frontmatter Fields

| Field | Purpose |
|-------|---------|
| `name` | Identifier used in `/skill-name` commands and Skill tool lookups |
| `description` | What the LLM sees when deciding whether to invoke this skill |
| `user-invocable` | Whether the user can trigger it with `/name` |
| `disable-model-invocation` | If `true`, only the user (not the model) can trigger it |
| `context` | `fork` = run in isolated context; `inherit` = share parent context |
| `agent` | Which agent definition to use when this skill runs |
| `allowed-tools` | Whitelist of tools the skill's agent can use |

### The Critical Insight: Skills Cannot Invoke Agents

This is the single most important architectural constraint. A skill can specify an `agent` to handle its execution, but a skill **cannot call the Agent tool**. If you look at the `allowed-tools` list:

```yaml
allowed-tools: Read, Write, Edit, Bash, Grep, Glob
```

Notice what is absent: `Agent`. A skill operates within the context of its designated agent but cannot spawn sub-agents. This is a deliberate design choice -- skills are knowledge, not orchestrators. The brain analogy works well here: a skill is like a book of knowledge. It can inform how you work, but the book itself cannot pick up a phone and delegate tasks.

Compare this with the `content-writer` agent definition:

```yaml
---
name: content-writer
tools: Read, Write, Edit, Bash, Grep, Glob, Agent
---
```

The agent *does* have access to the `Agent` tool. Agents can delegate to sub-agents. Skills cannot. This hierarchy is:

```
User --> Skill (knowledge + behavioral rules)
            |
            +--> Agent (hands that do the work)
                    |
                    +--> Sub-Agent (delegated task)
```

### Skills from Plugins

Plugins can also provide skills. The `superpowers` plugin, for example, ships 14 skills:

```
superpowers/skills/
  brainstorming/
  dispatching-parallel-agents/
  executing-plans/
  finishing-a-development-branch/
  receiving-code-review/
  requesting-code-review/
  subagent-driven-development/
  systematic-debugging/
  test-driven-development/
  using-git-worktrees/
  using-superpowers/
  verification-before-completion/
  writing-plans/
  writing-skills/
```

Each follows the same `SKILL.md` convention. The `using-superpowers` skill is special -- it gets injected at session start via a hook (more on this below) and establishes the rule that the model must check for relevant skills before taking any action.

---

## Agents -- The Hands That Do the Work

An **agent** is a full LLM sub-session with its own system prompt, tool access, and execution context. Think of it as spawning a fresh Claude instance with specific instructions and capabilities.

### Anatomy of an Agent

Agents are markdown files in `~/.claude/agents/`:

```
~/.claude/agents/
  content-writer.md
  daily-dump.md
```

The frontmatter is the control plane:

```yaml
---
name: content-writer
description: "Technical blog post and content writer for the digital garden."
tools: Read, Write, Edit, Bash, Grep, Glob, Agent
model: opus
---
```

The body is the system prompt -- it tells the agent exactly what it is, what it should do, and how it should behave. For the `content-writer` agent, this includes the entire repository structure, formatting rules, writing style guidelines, and a step-by-step workflow.

### Agent vs. Skill -- The Brain vs. Hands Analogy

| Dimension | Skill | Agent |
|-----------|-------|-------|
| What it is | Knowledge / behavioral instructions | Autonomous sub-session |
| Can use tools? | No (only through its agent) | Yes -- full tool access |
| Can spawn sub-agents? | No | Yes (if `Agent` is in tools list) |
| Context | Injected into existing context | Forked (isolated) or inherited |
| File format | `SKILL.md` in a directory | Markdown file with frontmatter |
| Persistence | Loaded on demand, no state | Runs to completion, returns result |

The relationship is: a **skill** is *what to know*, an **agent** is *who does the work*. A skill can specify which agent should execute when the skill is invoked:

```yaml
# In SKILL.md:
agent: content-writer    # <-- delegates to this agent
```

But the agent itself can exist independently. You can invoke an agent directly from Claude Code without going through a skill.

### Plugin Agents

Plugins ship agents too. The `asdd-kit` plugin includes 14 agents:

```
asdd-kit/agents/
  code-explorer.md       # Read-only codebase analysis
  code-reviewer.md       # Code review with specific rubrics
  figma-codegen.md       # Generate code from Figma designs
  jira-crawler.md        # Extract data from Jira
  reconciler.md          # Verify code matches spec
  security-agent.md      # Security audit
  slack-crawler.md       # Extract context from Slack
  spec-drafter.md        # Draft agent specifications
  spec-reviewer.md       # Review specs for completeness
  test-runner.md         # Run tests in isolation
  ...
```

Each agent has a constrained tool list. The `reconciler` agent, for instance, is deliberately limited:

```yaml
tools: Glob, Grep, Read, Bash, BashOutput
```

No `Write`, no `Edit` -- it can read and analyze but never modify code. This is enforced at the runtime level, not by asking nicely.

### Sub-Agent Patterns

The `superpowers` plugin popularizes "subagent-driven development" -- a pattern where the main agent dispatches fresh sub-agents per task:

```
Main Agent (coordinator)
  |
  +-- Agent("implement feature X")   --> commits code
  |
  +-- Agent("review spec compliance") --> reports gaps
  |
  +-- Agent("review code quality")    --> reports issues
  |
  +-- Agent("fix quality issues")     --> commits fixes
```

Each sub-agent gets a fresh context (no accumulated history from previous tasks), preventing context pollution. The coordinator retains oversight without burning context window on implementation details.

---

## Hooks -- Lifecycle Callbacks

A **hook** is a shell command that runs in response to a Claude Code lifecycle event. Hooks are the mechanism for injecting context, validating actions, or side-effecting at specific points in a session.

### Two Sources of Hooks

**1. User hooks** -- defined in `settings.json`:

User hooks are configured directly in the Claude Code settings file. They follow the same event-based pattern but are defined at the user level rather than bundled with a plugin.

**2. Plugin hooks** -- defined in a plugin's `hooks/hooks.json`:

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup|clear|compact",
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd\" session-start",
            "async": false
          }
        ]
      }
    ]
  }
}
```

### Hook Event Types

| Event | When It Fires |
|-------|--------------|
| `SessionStart` | Session begins, or after `/clear` or `/compact` |
| `PreToolUse` | Before a tool is invoked |
| `PostToolUse` | After a tool completes |
| `Notification` | When Claude wants to notify the user |
| `Stop` | When Claude finishes a response |

The `matcher` field controls which specific triggers fire the hook. For `SessionStart`, the matcher `"startup|clear|compact"` means the hook fires on initial startup and also when the user clears or compacts the conversation.

### SessionStart Injection -- How Superpowers Bootstraps Itself

The `superpowers` plugin's SessionStart hook is the most instructive example. Here is what happens:

```
Session Start
     |
     v
hooks.json fires SessionStart hook
     |
     v
run-hook.cmd dispatches to "session-start" script
     |
     v
session-start script:
  1. Reads using-superpowers/SKILL.md
  2. Escapes it for JSON
  3. Wraps it in <EXTREMELY_IMPORTANT> tags
  4. Outputs JSON with hookSpecificOutput.additionalContext
     |
     v
Claude Code runtime injects the additionalContext
into the system prompt for this session
     |
     v
Claude now "knows" about superpowers skills
and will invoke the Skill tool before every action
```

The shell script outputs a specific JSON structure:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "SessionStart",
    "additionalContext": "<EXTREMELY_IMPORTANT>\nYou have superpowers.\n..."
  }
}
```

The runtime picks up `additionalContext` and injects it into the model's context. This is how a plugin can modify Claude's behavior from the very first message -- without the user doing anything.

The script even handles platform differences:

```bash
if [ -n "${CURSOR_PLUGIN_ROOT:-}" ]; then
  # Cursor expects additional_context (snake_case)
  printf '{\n  "additional_context": "%s"\n}\n' "$session_context"
elif [ -n "${CLAUDE_PLUGIN_ROOT:-}" ]; then
  # Claude Code expects hookSpecificOutput.additionalContext
  printf '{\n  "hookSpecificOutput": ...\n}\n' "$session_context"
else
  # Copilot CLI / unknown -- SDK standard format
  printf '{\n  "additionalContext": "%s"\n}\n' "$session_context"
fi
```

### The `async` Flag

Hooks can be synchronous (`async: false`) or asynchronous (`async: true`). The SessionStart hook for superpowers is synchronous -- Claude waits for it to complete before proceeding. This makes sense because the injected context must be available before the model generates its first response.

---

## Plugins and Marketplaces -- The Distribution Layer

### What Is a Plugin?

A plugin is a git repository that bundles skills, agents, hooks, and commands into a distributable package. The plugin declares its contents via `.claude-plugin/plugin.json`:

```json
{
  "name": "asdd-kit",
  "version": "0.4.20260331175938",
  "description": "Agent Spec Driven Development Kit",
  "agents": [
    "./plugins/asdd-kit/agents/figma-codegen.md",
    "./plugins/asdd-kit/agents/reconciler.md",
    ...
  ],
  "commands": "./plugins/asdd-kit/commands/"
}
```

The directory structure of a plugin looks like:

```
asdd-kit/
  .claude-plugin/
    plugin.json           # Plugin manifest
    marketplace.json      # If this plugin IS a marketplace
  .claude/
    settings.json         # Plugin's own settings (can depend on other plugins)
  plugins/asdd-kit/
    agents/               # Agent markdown files
    skills/               # Skill directories with SKILL.md
    commands/             # Slash-command definitions
    resources/            # Static resources
```

### Commands -- The Slash-Command Interface

Commands are markdown files that define slash-commands the user can invoke. They are distinct from skills -- a command is a user-triggered workflow, while a skill is model-triggered knowledge.

```yaml
# commands/kick.md
---
name: asdd-kit:kick
description: "Implements a single task from an Agent Spec document."
---

# Implements a single task from an Agent Spec document
...
```

The user invokes it as `/asdd-kit:kick` in the Claude Code prompt. The command's body becomes the task instruction. Commands can reference agents, skills, and tools within their workflow.

### What Is a Marketplace?

A marketplace is a git repository that contains a `.claude-plugin/marketplace.json` file -- a catalog of available plugins. Think of it as an app store index.

```json
{
  "$schema": "https://anthropic.com/claude-code/marketplace.schema.json",
  "name": "claude-plugins-official",
  "description": "Directory of popular Claude Code extensions",
  "owner": {
    "name": "Anthropic",
    "email": "support@anthropic.com"
  },
  "plugins": [
    {
      "name": "superpowers",
      "description": "Core skills library: TDD, debugging, collaboration",
      "source": {
        "source": "url",
        "url": "https://github.com/obra/superpowers.git",
        "sha": "917e5f53..."
      }
    },
    ...
  ]
}
```

### Plugin Sources

Plugins in a marketplace can come from three source types:

| Source Type | Example | Resolution |
|------------|---------|------------|
| `./path` | `"source": "./plugins/ds4-web"` | Relative to marketplace repo |
| `url` | `"source": {"source": "url", "url": "https://github.com/..."}` | External git repo |
| `github` | `"source": {"source": "github", "repo": "kouzoh/asdd-kit"}` | GitHub shorthand |
| `git-subdir` | `"source": {"source": "git-subdir", "url": "...", "path": "plugins/foo"}` | Subdirectory of a git repo |

### Official vs. Self-Hosted Marketplaces

Claude Code ships with one built-in marketplace:

```
claude-plugins-official  -->  anthropics/claude-plugins-official (GitHub)
```

But you can register additional marketplaces in `settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "coding-agent-plugins": {
      "source": {
        "source": "github",
        "repo": "kouzoh/coding-agent-plugins"
      }
    }
  }
}
```

This is how organizations distribute internal plugins -- you stand up a git repo with a `marketplace.json`, register it, and your team can install plugins from it.

The marketplace metadata is cached locally:

```
~/.claude/plugins/
  known_marketplaces.json         # Registry of all known marketplaces
  installed_plugins.json          # Which plugins are installed, versions, SHAs
  marketplaces/
    claude-plugins-official/      # Cloned marketplace repo
      .claude-plugin/
        marketplace.json          # The plugin catalog
      plugins/                    # Bundled plugins
      external_plugins/           # Plugins with external sources
    coding-agent-plugins/         # Self-hosted marketplace
      .claude-plugin/
        marketplace.json
  cache/
    claude-plugins-official/
      superpowers/5.0.7/          # Installed plugin at specific version
    coding-agent-plugins/
      asdd-kit/0.4.20260331175938/
```

### Installation and Versioning

When you install a plugin, Claude Code:

1. Resolves the source from `marketplace.json`
2. Clones or fetches the plugin repo
3. Caches it at a versioned path under `~/.claude/plugins/cache/`
4. Records the install in `installed_plugins.json` with the git commit SHA:

```json
{
  "asdd-kit@coding-agent-plugins": [
    {
      "scope": "user",
      "installPath": "~/.claude/plugins/cache/coding-agent-plugins/asdd-kit/0.4.20260331175938",
      "version": "0.4.20260331175938",
      "gitCommitSha": "ca2740e49c8df2d2352cd96e515c3d171db6dca3"
    }
  ]
}
```

The `scope` field indicates where the plugin is active. `"user"` means it is globally available. Plugins can also be scoped to specific projects.

### Enabling Plugins

Installing is not the same as enabling. You enable plugins in `settings.json`:

```json
{
  "enabledPlugins": {
    "asdd-kit@coding-agent-plugins": true,
    "superpowers@claude-plugins-official": true
  }
}
```

The key format is `plugin-name@marketplace-name`. This two-step (install then enable) gives you control over which plugins are active without uninstalling them.

---

## The Full Session Lifecycle

Here is what happens from the moment you type `claude` to the moment your task completes:

```
1. Claude Code starts
   |
   v
2. Load settings.json
   - Model selection (opus, sonnet, etc.)
   - Enabled plugins
   - Extra marketplaces
   |
   v
3. Load settings.local.json
   - Permissions (tool allowlists)
   |
   v
4. Load installed plugins from cache
   - Read each plugin's plugin.json
   - Register agents, skills, commands
   - Register hooks from hooks.json
   |
   v
5. Fire SessionStart hooks (synchronous)
   - superpowers: injects using-superpowers skill
   - Other plugins: inject their own context
   |
   v
6. Load CLAUDE.md / AGENTS.md from project root
   - Project-specific instructions
   - Merged into system context
   |
   v
7. Load user skills from ~/.claude/skills/
   - Register SKILL.md files
   |
   v
8. Session ready -- user sends first message
   |
   v
9. Model processes message
   - Sees: system prompt + hook-injected context + CLAUDE.md + skill descriptions
   - Decides: which tools to call, whether to invoke skills
   |
   v
10. Tool calls trigger PreToolUse hooks
    |
    v
11. Tool executes
    |
    v
12. PostToolUse hooks fire
    |
    v
13. If skill invoked:
    - Skill content loaded
    - If skill specifies agent: fork context, spawn agent
    - Agent executes with its own tool list
    - Result returned to main session
    |
    v
14. Model generates response
    |
    v
15. Stop hooks fire
```

### Practical Example: Writing a Note in the Digital Garden

When I type `/write kubernetes admission controllers`, here is the exact flow:

```
/write ...
  |
  v
Matches skill: "write" (user-invocable: true)
  |
  v
SKILL.md loaded, $ARGUMENTS = "kubernetes admission controllers"
  |
  v
agent: content-writer  --> forks context
  |
  v
content-writer agent starts with:
  - tools: Read, Write, Edit, Bash, Grep, Glob, Agent
  - system prompt: full content-writer instructions
  - task: "Write or update... kubernetes admission controllers"
  |
  v
Agent reads existing notes (Glob, Read)
Agent reads indexes (Read)
Agent writes note (Write)
Agent writes summary (Write)
Agent updates indexes (Edit)
  |
  v
Result returned to main session
```

### Practical Example: Terraform Work with asdd-kit

When working on `gke-cluster-kit` terraform configuration, the `asdd-kit` plugin's workflow looks like:

```
/asdd-kit:kick
  |
  v
Command: kick.md loaded
  |
  v
Pre-flight checks:
  - .asdd-kit/config exists?
  - Load spec path, task ID
  |
  v
Dispatch implementer sub-agent
  |
  v
Sub-agent reads spec, implements terraform changes
Sub-agent commits code
  |
  v
Dispatch reconciler agent (read-only)
  - Verifies implementation matches spec
  - Reports discrepancies without modifying code
  |
  v
Dispatch code reviewer agent
  - Reviews quality, patterns, security
  |
  v
Main agent aggregates results
```

The key pattern: the command orchestrates multiple agents, each with constrained capabilities. The implementer can write. The reconciler cannot. The reviewer can read but not modify. Separation of concerns at the agent level.

---

## The Filesystem Layout

For reference, here is the complete `~/.claude/` directory structure as it relates to the extension system:

```
~/.claude/
  settings.json                    # Global settings, model, enabled plugins,
                                   # extra marketplaces
  settings.local.json              # Permissions (tool allowlists)
  agents/                          # User-defined agents
    content-writer.md
    daily-dump.md
  skills/                          # User-defined skills
    write/SKILL.md
    dump/SKILL.md
  plugins/
    config.json                    # Plugin system config
    known_marketplaces.json        # Registry of all marketplaces
    installed_plugins.json         # Installed plugin versions + SHAs
    marketplaces/
      claude-plugins-official/     # Official Anthropic marketplace
        .claude-plugin/
          marketplace.json         # Catalog of all official plugins
        plugins/                   # ~33 bundled plugins
        external_plugins/          # ~17 external plugins
      coding-agent-plugins/        # Self-hosted org marketplace
        .claude-plugin/
          marketplace.json         # Catalog of org plugins
    cache/
      <marketplace>/<plugin>/<version>/
                                   # Installed plugin files
```

---

## Design Patterns and Gotchas

### Pattern: Skill Chains via Agent Delegation

A skill cannot call another skill directly. But a skill can delegate to an agent, and that agent can invoke the Skill tool (if it is in the tools list). This creates an indirect skill chain:

```
Skill A --> Agent X (tools include Skill) --> invokes Skill B --> Agent Y
```

This is intentional -- it keeps the dependency graph explicit and prevents circular skill invocations.

### Pattern: Read-Only Agents for Verification

Both `asdd-kit` and `superpowers` use agents with deliberately limited tool access for verification tasks. The reconciler agent has no Write or Edit tools. The code reviewer agent is read-only. This is enforced at the runtime level, not by prompt instruction.

### Gotcha: Plugin Settings Can Reference Other Plugins

The `asdd-kit` plugin's own `.claude/settings.json` enables another plugin:

```json
{
  "enabledPlugins": {
    "plugin-dev@claude-plugins-official": true
  }
}
```

Plugins can depend on other plugins. This creates a dependency graph that Claude Code resolves at load time.

### Gotcha: Context Fork Isolation

When a skill specifies `context: fork`, the agent runs in isolation. It does not see the conversation history. This means:

- The agent does not know what you discussed earlier in the session
- All necessary context must be in the skill's instructions or passed as arguments
- The result is returned to the main session, but the agent's internal reasoning is discarded

This is usually what you want (prevents context pollution), but it means skills need to be self-contained in their instructions.

### Gotcha: Hook Output Format Is Platform-Dependent

The `superpowers` session-start hook outputs different JSON structures depending on the platform:

- **Claude Code**: `hookSpecificOutput.additionalContext` (nested)
- **Cursor**: `additional_context` (snake_case, top-level)
- **Copilot CLI**: `additionalContext` (top-level, SDK standard)

If you write custom hooks, you need to handle this branching.

---

## Interview Prep

### Q: What is the difference between a skill and an agent in Claude Code?

**A:** A skill is a markdown file that provides knowledge and behavioral instructions -- it tells the LLM *how to think* about a class of problems. An agent is a full sub-session with its own system prompt, tool access, and execution context -- it is *what does the work*.

The critical architectural constraint: skills cannot invoke agents directly. A skill can specify which agent should handle its execution (via the `agent:` frontmatter field), but it cannot call the Agent tool. Skills have a restricted tool list (Read, Write, Edit, Bash, Grep, Glob) while agents can have the Agent tool, enabling sub-agent orchestration.

The analogy is brain vs. hands: the skill is the knowledge in your head, the agent is the person who acts on that knowledge.

### Q: How does a plugin inject context at session start?

**A:** Through the `SessionStart` hook mechanism. A plugin defines hooks in `hooks/hooks.json`, which specifies a shell command to run when the session starts. The shell script executes and outputs a JSON object with an `additionalContext` field (nested under `hookSpecificOutput` for Claude Code). The runtime reads this output and injects the text into the model's system context.

For example, the `superpowers` plugin reads its `using-superpowers/SKILL.md`, wraps it in `<EXTREMELY_IMPORTANT>` tags, escapes it for JSON, and outputs it. This makes Claude aware of all available superpowers skills from the very first message.

The hook is synchronous (`async: false`), meaning Claude Code waits for the script to complete before proceeding. This guarantees the injected context is available before the model generates any response.

### Q: How do Claude Code marketplaces work, and how would you set up an internal one for your organization?

**A:** A marketplace is a git repository containing a `.claude-plugin/marketplace.json` file that catalogs available plugins. Each entry in the catalog specifies a plugin name, description, and source (which can be a relative path within the repo, a git URL, a GitHub shorthand, or a subdirectory of another repo).

To set up an internal marketplace:

1. Create a git repo with a `.claude-plugin/marketplace.json` following the schema at `https://anthropic.com/claude-code/marketplace.schema.json`
2. Add plugin entries pointing to internal repos or bundled plugin directories
3. Register the marketplace in users' `~/.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "my-org-plugins": {
      "source": {
        "source": "github",
        "repo": "my-org/claude-plugins"
      }
    }
  }
}
```

Claude Code clones the marketplace repo locally, reads the catalog, and allows users to install plugins from it. Plugin versions are tracked by git commit SHA, and installations are cached under `~/.claude/plugins/cache/`.

### Q: Why can't skills invoke sub-agents? What problem does this constraint solve?

**A:** The constraint prevents uncontrolled agent spawning and keeps the execution graph manageable. If skills could spawn agents, and those agents could invoke skills that spawn more agents, you would get an unbounded tree of concurrent sub-sessions, each consuming context window and API tokens.

By restricting skills to knowledge injection only, the architecture ensures that:

1. **Agent spawning is explicit** -- only an agent (or the main session) can spawn sub-agents via the Agent tool
2. **The dependency graph is visible** -- you can trace skill -> agent -> sub-agent without hidden intermediate steps
3. **Resource usage is predictable** -- the number of active agents is bounded by explicit delegation, not implicit skill chaining
4. **Skills remain composable** -- since skills are just text, they can be combined, overridden, or extended without side effects

### Q: How would you design a Claude Code plugin for a platform engineering team managing Terraform infrastructure?

**A:** Based on the patterns I have seen in `asdd-kit` and `superpowers`, here is what I would include:

**Skills:**
- `terraform-plan-review` -- knowledge about how to read and analyze terraform plan output, common gotchas (state drift, provider version conflicts, resource recreation vs. in-place update)
- `infrastructure-patterns` -- organization-specific patterns (naming conventions, tagging standards, module structure)

**Agents:**
- `plan-analyzer` (read-only tools) -- takes a terraform plan and identifies risks, breaking changes, cost implications
- `state-inspector` (read-only tools) -- examines terraform state for drift, orphaned resources
- `module-implementer` (full tools) -- writes terraform modules following org patterns

**Hooks:**
- `SessionStart` -- inject the organization's terraform module catalog and naming conventions into context
- `PreToolUse` on Bash -- validate that destructive terraform commands (`terraform destroy`, `terraform apply` without `-auto-approve`) get user confirmation

**Commands:**
- `/infra:plan` -- run terraform plan and pipe through the plan-analyzer agent
- `/infra:apply` -- multi-stage apply with plan review, approval gate, and post-apply verification

The key pattern is separation of concerns: read-only agents for analysis, write-capable agents for implementation, and hooks for guardrails.

---

## Related Notes

- [[notes/K8s/kubelens-mcp-server|KubeLens: Building a Kubernetes MCP Server from Scratch]]
