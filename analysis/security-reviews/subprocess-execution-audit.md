# Subprocess Execution Audit

**Date:** 2026-03-20
**Scope:** All subprocess execution and command invocation code across the opencode monorepo
**Severity Classification:** CRITICAL / HIGH / MEDIUM / LOW / INFO

---

## Executive Summary

The monorepo contains **19 distinct categories** of subprocess execution across ~25 files. The most security-critical finding is the **Bash tool** which allows LLMs to execute arbitrary shell commands, mitigated by a user-consent permission system. MCP tool calls from LLMs **bypass** this permission system entirely. The plugin system grants full `Bun.$` shell access with no sandboxing.

---

## Findings

### 1. Bash Tool -- LLM-Directed Arbitrary Shell Execution -- CRITICAL

| Detail             | Value                                                                                                     |
| ------------------ | --------------------------------------------------------------------------------------------------------- |
| **File**           | `packages/opencode/src/tool/bash.ts` (lines 55-270)                                                       |
| **What**           | Any shell command the LLM chooses                                                                         |
| **Input source**   | `command` parameter from LLM tool call (line 64, 167)                                                     |
| **Execution**      | `spawn(params.command, { shell })` -- full shell execution                                                |
| **Injection risk** | **HIGH** -- LLM has full control of the command string; prompt injection could trigger arbitrary commands |

**Mitigations applied:**

- `ctx.ask()` with `permission: "bash"` prompts user before execution (line 154)
- `permission: "external_directory"` check for paths outside project (line 146)
- Tree-sitter bash parser extracts command patterns for human-readable prompts (lines 84-137)
- `BashArity` prefix matching groups commands for permission rules
- Shell selection blacklists `fish` and `nu` (only sh/bash/zsh)
- Users can configure allow/deny rules per command pattern

**Output handling:** stdout + stderr merged, returned to LLM as tool result, stored as message part metadata (truncated to 30KB via `Truncate.output()`).

---

### 2. MCP Tool Execution -- LLM-Directed, No Permission Checks -- CRITICAL

| Detail              | Value                                                                                             |
| ------------------- | ------------------------------------------------------------------------------------------------- |
| **File**            | `packages/opencode/src/mcp/index.ts` (lines 121-149, 448-490)                                     |
| **Server spawning** | Local MCP servers spawned via `StdioClientTransport` with command from user config                |
| **Tool calls**      | LLM provides arguments to any registered MCP tool; passed directly to `client.callTool()`         |
| **Injection risk**  | **HIGH for tool calls** -- LLM arguments flow directly into MCP tool execution with no validation |

**Mitigations applied:** **NONE on MCP tool calls.** MCP tools bypass the bash permission system entirely. Server spawning uses config-driven commands. Environment includes full `process.env` plus config-specified `mcp.environment` overrides. Descendant process tree cleaned up on shutdown (lines 164-180).

**Gap:** If an MCP server exposes dangerous tools (file write, shell exec), the LLM can invoke them freely without user consent.

---

### 3. Plugin System -- Full Shell Access, No Sandboxing -- HIGH

| Detail             | Value                                                                                  |
| ------------------ | -------------------------------------------------------------------------------------- |
| **File**           | `packages/opencode/src/plugin/index.ts`                                                |
| **What**           | Plugins loaded via `import(plugin)` (dynamic import of npm packages or `file://` URLs) |
| **Access**         | Plugin input includes `$: Bun.$`, granting full shell execution                        |
| **Injection risk** | **HIGH** -- any installed plugin can execute arbitrary commands                        |

**Mitigations applied:** **NONE.** Plugins run with full process privileges. Plugin hooks include `shell.env` which can inject environment variables into bash/prompt execution contexts.

---

### 4. Session Prompt -- User Shell Command Execution -- MEDIUM

| Detail             | Value                                                                                  |
| ------------------ | -------------------------------------------------------------------------------------- |
| **File**           | `packages/opencode/src/session/prompt.ts` (lines 1600-1730)                            |
| **What**           | Commands entered by user in prompt UI                                                  |
| **Execution**      | `eval ${JSON.stringify(input.command)}` for zsh/bash; sources user shell config        |
| **Injection risk** | **LOW** -- commands from user directly (not LLM); `JSON.stringify()` provides escaping |

**Mitigations:** User-initiated action. Plugin `shell.env` hook can inject environment variables.

---

### 5. PTY Spawning -- Interactive Terminals -- MEDIUM

| Detail             | Value                                                                  |
| ------------------ | ---------------------------------------------------------------------- |
| **File**           | `packages/opencode/src/pty/index.ts` (lines 120-210)                   |
| **What**           | Interactive terminal sessions via `bun-pty`                            |
| **Command**        | Defaults to user's preferred shell; can be specified via API parameter |
| **Injection risk** | **LOW** -- command from API input (user-controlled), not LLM           |

**Mitigations:** **NONE** -- designed for user-interactive terminals. PTY is API-accessible via server routes.

---

### 6. Grep Tool -- LLM-Directed ripgrep -- LOW

| Detail             | Value                                                                                             |
| ------------------ | ------------------------------------------------------------------------------------------------- |
| **File**           | `packages/opencode/src/tool/grep.ts`                                                              |
| **What**           | `Process.spawn([rgPath, ...args])` -- ripgrep with LLM-provided pattern, path, include parameters |
| **Injection risk** | **LOW** -- array-based arguments (not shell-interpolated); cannot execute arbitrary code          |

**Mitigations:**

- `permission: "grep"` check via `ctx.ask()`
- `assertExternalDirectory()` for paths outside project
- Array-based invocation prevents shell injection

---

### 7. LSP Server Spawning -- LOW

| Detail             | Value                                                                        |
| ------------------ | ---------------------------------------------------------------------------- |
| **Files**          | `packages/opencode/src/lsp/server.ts`, `packages/opencode/src/lsp/launch.ts` |
| **What**           | ~20+ language servers (TypeScript, Go, Python, Rust, etc.)                   |
| **Commands**       | Hardcoded or resolved via `which()`; some auto-install commands              |
| **Injection risk** | **NEGLIGIBLE** -- commands not from user/LLM input                           |

---

### 8. Snapshot/Git Operations -- LOW

| Detail             | Value                                                 |
| ------------------ | ----------------------------------------------------- |
| **File**           | `packages/opencode/src/snapshot/index.ts`             |
| **What**           | `git` commands via Effect-based `ChildProcessSpawner` |
| **Injection risk** | **NEGLIGIBLE** -- no user/LLM input in git commands   |

---

### 9. GitHub Action / CLI -- LOW

| Detail             | Value                                                                                            |
| ------------------ | ------------------------------------------------------------------------------------------------ |
| **Files**          | `packages/opencode/src/cli/cmd/github.ts`, `github/index.ts`                                     |
| **What**           | `git` commands, browser URL opening, `opencode serve` spawning                                   |
| **Injection risk** | **LOW** -- Bun `$` template literal escaping; hardcoded URL patterns                             |
| **Note**           | GitHub action hardcodes `permission: "question"` -> `action: "deny"` (auto-denies LLM questions) |

---

### 10. SDK Server -- LOW

| Detail             | Value                                                               |
| ------------------ | ------------------------------------------------------------------- |
| **Files**          | `packages/sdk/js/src/server.ts`, `packages/sdk/js/src/v2/server.ts` |
| **What**           | Spawns `opencode serve` and `opencode` TUI processes                |
| **Injection risk** | **LOW** -- arguments from SDK configuration, not external input     |

---

### 11. Desktop Electron -- LOW

| Detail             | Value                                                                                  |
| ------------------ | -------------------------------------------------------------------------------------- |
| **Files**          | `packages/desktop-electron/src/main/cli.ts`, `ipc.ts`, `apps.ts`                       |
| **What**           | Sidecar binary, install scripts, `execFileSync` for `which`/`where`/`wsl`/`wslpath`    |
| **Injection risk** | **LOW** -- uses `execFileSync`/`execFile` (no shell); `shellEscape()` for WSL env vars |

---

### 12. Core Utilities -- INFO

| Detail   | Value                                                                     |
| -------- | ------------------------------------------------------------------------- |
| **File** | `packages/opencode/src/util/process.ts`                                   |
| **What** | Central `Process.spawn()` and `Process.run()` wrapper using `cross-spawn` |
| **Risk** | Infrastructure -- risk depends on caller                                  |

---

### 13. Shell Utility -- INFO

| Detail   | Value                                                                 |
| -------- | --------------------------------------------------------------------- |
| **File** | `packages/opencode/src/shell/shell.ts`                                |
| **What** | `killTree()` uses `taskkill` on Windows, `process.kill(-pid)` on Unix |
| **Risk** | **NEGLIGIBLE** -- PIDs are numeric                                    |

---

### 14. Binary Launcher, Build Scripts, Tests -- INFO

| Detail    | Value                                                                         |
| --------- | ----------------------------------------------------------------------------- |
| **Files** | `packages/opencode/bin/opencode`, `script/stats.ts`, various test/build files |
| **Risk**  | **NEGLIGIBLE** -- build-time or test-only code with hardcoded commands        |

---

## Permission System Overview

**Files:** `packages/opencode/src/permission/index.ts`, `arity.ts`, `schema.ts`

The permission system provides user-consent-based gating for bash commands and grep:

| Feature          | Description                                                                     |
| ---------------- | ------------------------------------------------------------------------------- |
| Permission check | `PermissionNext.Service` offers `ask()` and `reply()` Effect-based checks       |
| Rule types       | `allow`, `deny`, `ask` actions per command pattern                              |
| Command grouping | `BashArity` maps command prefixes to arity levels (e.g., "git" vs "git commit") |
| UI prompts       | "Allow always", "Allow once", "Deny"                                            |
| Scope            | Session-level and global rules stored in config                                 |

### Permission System Gap

**MCP tool calls and plugin execution bypass this system entirely.** The permission framework only gates the bash and grep tools. Any MCP server tool or plugin can execute commands without user consent.

---

## Data Flow: LLM to Command Execution

```
LLM Response
    |
    +---> [Bash Tool] --permission check--> spawn(command, {shell})
    |         |
    |         v
    |     stdout/stderr --> [Tool Output Storage] --> [LLM Context]
    |
    +---> [MCP Tool Call] --NO permission check--> client.callTool(args)
    |         |
    |         v
    |     MCP result --> [LLM Context]
    |
    +---> [Grep Tool] --permission check--> spawn([rg, ...args])
    |         |
    |         v
    |     grep output --> [LLM Context]
    |
    +---> [Plugin Hook] --NO permission check--> Bun.$(...)
```

---

## Key Security Gaps

| Gap                                | Severity     | Description                                                                           |
| ---------------------------------- | ------------ | ------------------------------------------------------------------------------------- |
| MCP tools bypass permissions       | **CRITICAL** | LLM can call any MCP tool with arbitrary arguments without user consent               |
| Plugins have full shell access     | **HIGH**     | `Bun.$` is passed directly to plugins with no sandboxing                              |
| Bash tool relies on user vigilance | **MEDIUM**   | Permission prompts can cause fatigue; users may "Allow always" broadly                |
| No output sanitization             | **LOW**      | Command output returned to LLM could contain secrets included in conversation context |

---

## Recommendations

| Priority     | Recommendation                                                                                                                                           |
| ------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **CRITICAL** | Extend the permission system to cover MCP tool calls -- require user consent before executing MCP tools, especially those with side effects              |
| **HIGH**     | Implement a plugin sandboxing model -- restrict or audit `Bun.$` access; consider capability-based permissions                                           |
| **HIGH**     | Add a "dangerous command" blocklist for the bash tool (e.g., `rm -rf /`, `curl \| bash`, `chmod 777`) that requires explicit override                    |
| **MEDIUM**   | Implement output sanitization to detect and redact potential secrets before returning command output to the LLM                                          |
| **MEDIUM**   | Add permission fatigue mitigations -- batch similar permission requests, provide session-scoped "trust profiles"                                         |
| **LOW**      | Consider process-level sandboxing (e.g., `pledge`/`unveil` on OpenBSD, `seccomp` on Linux, sandbox profiles on macOS) for high-risk subprocess execution |
