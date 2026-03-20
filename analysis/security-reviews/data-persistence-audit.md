# Data Persistence and Sharing Audit

**Date:** 2026-03-20
**Scope:** All data persistence mechanisms, storage locations, and sharing/export functionality across the opencode monorepo
**Severity Classification:** CRITICAL / HIGH / MEDIUM / LOW / INFO

---

## Executive Summary

The codebase persists data across **14 distinct mechanisms** spanning SQLite databases, JSON files, git repositories, log files, cache directories, and external API calls. Only two files use restricted permissions (`0o600`). The primary external data exposure vector is the session sharing feature, which can be configured to automatically transmit all conversation data (including source code). Tool output persistence can capture arbitrary sensitive data. No file-level encryption is used.

---

## Findings

### 1. SQLite Database (Primary Storage) -- MEDIUM

**Location:** `$XDG_DATA_HOME/opencode/opencode.db`

**Init files:**

- `packages/opencode/src/storage/db.ts` -- path resolution
- `packages/opencode/src/storage/db.bun.ts` -- Bun runtime init
- `packages/opencode/src/storage/db.node.ts` -- Node runtime init

**PRAGMAs:** WAL mode, NORMAL sync, busy_timeout=5000, cache_size=-64000, foreign_keys=ON

**No explicit file permissions set on the database file.**

#### Tables and Sensitivity

| Table                      | Schema File                          | Data Stored                                                                                | Sensitivity  |
| -------------------------- | ------------------------------------ | ------------------------------------------------------------------------------------------ | ------------ |
| `account`                  | `src/account/account.sql.ts`         | email, **access_token**, **refresh_token**, token_expiry                                   | **CRITICAL** |
| `account_state`            | `src/account/account.sql.ts`         | active account/org selection                                                               | Low          |
| `control_account` (legacy) | `src/account/account.sql.ts`         | email, **access_token**, **refresh_token**                                                 | **CRITICAL** |
| `session`                  | `src/session/session.sql.ts`         | metadata, share_url, directory paths, permission rulesets, summary diffs, revert snapshots | Medium       |
| `message`                  | `src/session/session.sql.ts`         | full message data as JSON blob (conversation content)                                      | Medium-High  |
| `part`                     | `src/session/session.sql.ts`         | message parts as JSON blob (tool calls, text, code)                                        | Medium-High  |
| `todo`                     | `src/session/session.sql.ts`         | task items per session                                                                     | Low          |
| `permission`               | `src/session/session.sql.ts`         | permission rulesets per project                                                            | Low          |
| `project`                  | `src/project/project.sql.ts`         | project metadata, worktree/sandbox paths                                                   | Low          |
| `session_share`            | `src/share/share.sql.ts`             | share **id, secret, url**                                                                  | **HIGH**     |
| `workspace`                | `src/control-plane/workspace.sql.ts` | workspace metadata                                                                         | Low          |

---

### 2. Legacy JSON File Storage -- LOW

| Detail          | Value                                                                                     |
| --------------- | ----------------------------------------------------------------------------------------- |
| **Location**    | `$XDG_DATA_HOME/opencode/storage/`                                                        |
| **File**        | `packages/opencode/src/storage/storage.ts`                                                |
| **What**        | Individual `.json` files for sessions, messages, parts, todos, permissions, shares, diffs |
| **Permissions** | **No explicit file permissions set**                                                      |
| **Status**      | Migrated to SQLite via `src/storage/json-migration.ts`                                    |

---

### 3. Credential / Auth Files (Restricted Permissions) -- See credential-secret-handling.md

| File                                    | Data                                 | Permissions    |
| --------------------------------------- | ------------------------------------ | -------------- |
| `$XDG_DATA_HOME/opencode/auth.json`     | OAuth tokens, API keys               | **mode 0o600** |
| `$XDG_DATA_HOME/opencode/mcp-auth.json` | MCP OAuth tokens, client credentials | **mode 0o600** |

These are the **only files with explicit restricted permissions.**

---

### 4. Cache Directories -- LOW

**Base:** `$XDG_CACHE_HOME/opencode/` (versioned; version "21"; auto-cleared on version bump)

| Path                | Code Location                      | Data                                                          |
| ------------------- | ---------------------------------- | ------------------------------------------------------------- |
| `cache/models.json` | `src/provider/models.ts` (line 16) | Cached models.dev API data, refreshed hourly                  |
| `cache/skills/`     | `src/skill/discovery.ts` (line 36) | Downloaded skill files                                        |
| `cache/bin/`        | `src/global/index.ts`              | Downloaded LSP server binaries (gopls, pyright, clangd, etc.) |

---

### 5. Snapshot System (Git-based) -- MEDIUM

| Detail       | Value                                                              |
| ------------ | ------------------------------------------------------------------ |
| **Location** | `$XDG_DATA_HOME/opencode/snapshot/{projectId}/`                    |
| **File**     | `packages/opencode/src/snapshot/index.ts` (line 99)                |
| **What**     | Bare git repository per project tracking full filesystem state     |
| **Data**     | Complete file contents of the working directory via git operations |
| **Cleanup**  | Hourly with 7-day prune                                            |
| **Risk**     | Could contain source code and sensitive files from the project     |

---

### 6. Log Files -- MEDIUM

| Detail          | Value                                                                  |
| --------------- | ---------------------------------------------------------------------- |
| **Location**    | `$XDG_DATA_HOME/opencode/log/`                                         |
| **File**        | `packages/opencode/src/util/log.ts` (lines 60-69)                      |
| **What**        | Log files named `{ISO-timestamp}.log` or `dev.log`                     |
| **Data**        | Service names, timestamps, arbitrary metadata including error messages |
| **Cleanup**     | Keeps last 10 files                                                    |
| **Permissions** | **No explicit file permissions set**                                   |

---

### 7. Tool Output / Truncation -- MEDIUM

| Detail        | Value                                                                                                                 |
| ------------- | --------------------------------------------------------------------------------------------------------------------- |
| **Location**  | `$XDG_DATA_HOME/opencode/tool-output/`                                                                                |
| **Files**     | `src/tool/truncation-dir.ts`, `src/tool/truncate-effect.ts`                                                           |
| **What**      | Full output of truncated tool calls (bash, grep results)                                                              |
| **Retention** | 7 days                                                                                                                |
| **Risk**      | **Could contain any command output including environment variables, file contents, or credentials printed to stdout** |

---

### 8. Share Functionality (External Data Transmission) -- HIGH

| Detail            | Value                                                                          |
| ----------------- | ------------------------------------------------------------------------------ |
| **File**          | `packages/opencode/src/share/share-next.ts`                                    |
| **Endpoint**      | `https://opncd.ai/api/share` (or console API for org accounts)                 |
| **Data sent**     | Full conversation content, tool call results, file diffs, model configurations |
| **Modes**         | `"manual"` (default), `"auto"`, `"disabled"`                                   |
| **Auto trigger**  | `OPENCODE_AUTO_SHARE` env var or `share: "auto"` config                        |
| **Disable**       | `OPENCODE_DISABLE_SHARE=true` env var or `share: "disabled"` config            |
| **Sync behavior** | `fullSync` sends ALL messages and parts for a session; debounced at 1 second   |

**This is the primary external data exposure vector.** When auto-share is enabled, all conversation data (which may include source code, file contents, command output) is transmitted to an external server.

---

### 9. Clipboard Access -- LOW

| Detail        | Value                                                                     |
| ------------- | ------------------------------------------------------------------------- |
| **File**      | `packages/opencode/src/cli/cmd/tui/util/clipboard.ts`                     |
| **Read**      | Text and images from clipboard (osascript/powershell/xclip/wl-paste)      |
| **Write**     | OSC 52 escape sequence + native tools                                     |
| **Temp file** | `$TMPDIR/opencode-clipboard.png` for macOS image reads (cleaned up after) |

---

### 10. State Persistence -- LOW

| Path                                      | Code Location                          | Data                           |
| ----------------------------------------- | -------------------------------------- | ------------------------------ |
| `$XDG_STATE_HOME/opencode/model.json`     | `src/provider/provider.ts` (line 1361) | Recently used model selections |
| `$XDG_CONFIG_HOME/opencode/opencode.json` | `src/config/config.ts`                 | Global configuration           |
| `$XDG_CONFIG_HOME/opencode/tui.json`      | TUI settings                           | TUI-specific preferences       |

---

### 11. Worktree Data -- LOW

| Detail       | Value                                                                           |
| ------------ | ------------------------------------------------------------------------------- |
| **Location** | `$XDG_DATA_HOME/opencode/worktree/{projectId}/`                                 |
| **File**     | `packages/opencode/src/worktree/index.ts` (line 343)                            |
| **What**     | Git worktree directories -- contains full working copies of project source code |

---

### 12. Plan Files -- LOW

| Detail       | Value                                                                                                           |
| ------------ | --------------------------------------------------------------------------------------------------------------- |
| **File**     | `packages/opencode/src/session/index.ts` (lines 340-344)                                                        |
| **Location** | `{worktree}/.opencode/plans/{timestamp}-{slug}.md` (git projects) or `$XDG_DATA_HOME/opencode/plans/` (non-git) |
| **What**     | AI-generated planning documents                                                                                 |

---

### 13. Directory Structure -- INFO

**File:** `packages/opencode/src/global/index.ts`

All directories created at startup with `{ recursive: true }`, **no explicit permissions:**

- `$XDG_DATA_HOME/opencode/`
- `$XDG_CONFIG_HOME/opencode/`
- `$XDG_STATE_HOME/opencode/`
- `$XDG_DATA_HOME/opencode/log/`
- `$XDG_CACHE_HOME/opencode/bin/`

Cleanup on uninstall: `packages/opencode/src/cli/cmd/uninstall.ts`

---

## Data Flow Summary

```
User Input / Source Code
    |
    v
[LLM Conversation] --> [SQLite: message/part tables]
    |                        |
    |                        v
    |                   [Snapshots: git repos with full file contents]
    |                        |
    |                        v
    |                   [Tool Output: bash/grep results stored 7 days]
    |
    +---> [AI Provider APIs] (always -- required for operation)
    |
    +---> [opncd.ai Share API] (optional -- manual or auto)
    |
    +---> [Log Files] (metadata, errors)
    |
    +---> [Clipboard] (user-initiated read/write)
```

---

## Key Security Observations

1. **Only two files use restricted permissions (0o600):** `auth.json` and `mcp-auth.json`. The SQLite database containing access/refresh tokens, log files, tool output, and JSON storage have **no explicit permissions** -- they inherit the process umask.

2. **Auto-share can silently transmit all conversation data** (including source code and command output) to `opncd.ai` when enabled.

3. **Tool output persistence can capture arbitrary sensitive data** from bash commands and store it for 7 days.

4. **Snapshots contain full file contents** of project working directories, potentially including `.env` files or other secrets.

5. **No file-level encryption** is used for any persisted data.

---

## Recommendations

| Priority   | Recommendation                                                                                                  |
| ---------- | --------------------------------------------------------------------------------------------------------------- |
| **HIGH**   | Set explicit permissions (0o600 or 0o700) on the SQLite database file, log directory, and tool-output directory |
| **HIGH**   | Consider excluding `.env` and other sensitive files from the snapshot system                                    |
| **MEDIUM** | Add a content filter to tool output persistence to redact potential secrets before storage                      |
| **MEDIUM** | Implement at-rest encryption for the SQLite database or at minimum for credential columns                       |
| **LOW**    | Document all data storage locations in user-facing documentation                                                |
| **LOW**    | Consider shorter retention for tool output (7 days may be excessive for sensitive data)                         |
