# Multiple coding agents

Academic Studio was built around Claude Code. This note describes the layer
that lets it drive other coding agents (OpenAI Codex, Gemini, ...) without
copying the Claude-specific code for each one.

**Status:** step 1 of the plan below is done. Claude is the only agent in the
registry, and with it the app behaves exactly as before.

## The agent registry

Every agent-specific id, folder, file name, command id and URL lives in one
place: `academicStudioAgents` in `overlay/product.overrides.json`, which
`apply-overlay.sh` merges into the app's `product.json`. Three consumers read it:

| Consumer | What it takes from the registry |
| --- | --- |
| core, `contrib/academicStudio/browser/agents.ts` + `claudeMenu.contribution.ts` (patch 55) | the agent menu: chat commands, home/project folders, instructions file, skills folder, URLs, feature gates |
| `academic-studio-defaults` | the chat tab opened on startup and how it is recognized; the permission-mode seeding |
| `academic-studio-setup` | where skills install, the `skills` CLI `-a` agent name, the connector config file, the plugin CLI |

The active agent is the user setting `academicStudio.agent` (registered by
`agents.ts`; its allowed values are the registry's keys). When it is unset or
names an unknown agent, `academicStudioDefaultAgent` applies. It is an
application-wide setting (no per-folder override), and the extensions read it
at activation, so a change takes effect after a window reload. An entry missing
any of `displayName`, `extensionId`, `commands.open`, `homeDir`, `projectDir`,
`instructionsFile` or `skillsDir` is ignored by all three consumers.

Keys starting with `_` in the registry are comments. Both extensions carry a
small `FALLBACK_AGENT` equal to the claude entry, used only if `product.json`
cannot be read.

### Fields

| Field | Claude | Meaning |
| --- | --- | --- |
| `displayName` | `Claude` | shown in menus and messages |
| `extensionId` | `anthropic.claude-code` | the bundled extension providing the chat |
| `settingsSection` | `claudeCode` | the extension's settings prefix |
| `commands.open` | `claude-vscode.primaryEditor.open` | opens the chat; may take a conversation id to resume |
| `commands.newChat` | `claude-vscode.newConversation` | optional; falls back to `open` |
| `tabPattern` | `claude` | case-insensitive regex recognizing the chat tab |
| `homeDir` | `.claude` | user config folder under `~` |
| `projectDir` | `.claude` | per-project config folder |
| `instructionsFile` | `CLAUDE.md` | read from the project root and `homeDir` |
| `importSyntax` | `@{0}` | how the instructions file imports another; enables the shared `AGENTS.md` item |
| `userConfigFile` | `.claude.json` | user-level config under `~` (connectors) |
| `skillsDir` | `skills` | skills folder inside `homeDir` / `projectDir` |
| `skillsCliAgent` | `claude-code` | the `skills` CLI `-a` value |
| `cli` | `claude` | the agent's command-line tool (plugin installs) |
| `urls.*` | | skills collection, MCP docs, plugin directories; items without a URL are hidden |
| `features.*` | see below | which implementation serves each capability |

### Features

Some menu items depend on one agent's own file formats, so they are gated by
`features`. A missing feature makes the item say it is not available for that
agent yet; it never reads another agent's files.

| Feature | Claude value | Used by |
| --- | --- | --- |
| `conversations` | `claude-jsonl` | Conversations… (reads `~/.claude/projects/<cwd>/*.jsonl`) |
| `permissions` | `claude-code` | Permissions… and the bypass-mode seeding (`claudeCode.initialPermissionMode`, `allowDangerouslySkipPermissions`) |
| `toggles` | `claude-settings` | Skills… on/off (`skillOverrides`, `enabledPlugins` in `settings.json`) |
| `connectors` | `claude-json` | Connectors… and Run Setup's connector installs (`mcpServers` in `~/.claude.json`) |
| `plugins` | `claude-marketplace` | Plugins…, plugin skills in Skills…, Run Setup's marketplace installs (`claude plugin …`) |

New Chat, New Skill, Get More Skills and the instructions-file item work for any
agent from the plain fields alone.

## Adding an agent

1. Add its extension to `overlay/extensions.json` and the Run Setup extension
   catalog, and its entry to the registry with the fields above.
2. For each feature it should support, add a new implementation value (for
   example `connectors: 'codex-toml'` for `~/.codex/config.toml`), implement it
   next to the Claude one, and add the value to `IAgentFeatures` in `agents.ts`.
3. Labels inside Claude-only features still say "Claude"; give them the
   agent's name as each feature gains a second implementation.

## Plan

1. **Done:** registry and adapter layer, Claude only, no behaviour change.
2. Codex: bundle `openai.chatgpt` (Open VSX), `AGENTS.md`, `~/.agents/skills`,
   `~/.codex/config.toml` connectors, `codex plugin marketplace` installs.
3. Gemini: Gemini Code Assist or the Gemini CLI companion; `GEMINI.md`,
   `~/.gemini/settings.json`.
4. Optionally a provider-agnostic agent (Cline or opencode) for free choice of model.
5. Plugin ports, starting with finance-data and wrds.

Known limits of step 1: the menu title ("Claude", patch 54) and the command
palette category are still fixed strings, and the agent setting is chosen in
Settings rather than in Run Setup.
