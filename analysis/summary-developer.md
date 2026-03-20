# Security Review -- Developer Summary

**Date:** 2026-03-20
**Project:** opencode
**Review Type:** Data exfiltration, external sharing, information leakage
**Detailed Reports:** [`./security-reviews/`](./security-reviews/)

---

## Scope

Five audit domains were analyzed across the entire monorepo:

| Domain                         | Report                                                                                                 |
| ------------------------------ | ------------------------------------------------------------------------------------------------------ |
| Network communication          | [`security-reviews/network-communication-audit.md`](./security-reviews/network-communication-audit.md) |
| Telemetry and analytics        | [`security-reviews/telemetry-analytics-audit.md`](./security-reviews/telemetry-analytics-audit.md)     |
| Credential and secret handling | [`security-reviews/credential-secret-handling.md`](./security-reviews/credential-secret-handling.md)   |
| Data persistence and sharing   | [`security-reviews/data-persistence-audit.md`](./security-reviews/data-persistence-audit.md)           |
| Subprocess execution           | [`security-reviews/subprocess-execution-audit.md`](./security-reviews/subprocess-execution-audit.md)   |

---

## Finding Summary by Severity

| Severity        | Count | Description                                                                                                                                                                 |
| --------------- | ----- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| CRITICAL        | 3     | MCP tools bypass permission system; bash tool accepts arbitrary LLM commands; full conversation data sent to AI providers                                                   |
| HIGH            | 3     | Partial access token logged (20 chars); plugins get unsandboxed `Bun.$`; auto-share sends all data to opncd.ai                                                              |
| MEDIUM          | 9     | Plaintext credential storage; secrets in `process.env` inherited by children; no DB file permissions; tool output stores sensitive data 7 days; HTTP Basic Auth over base64 |
| LOW             | 8     | Version checks; user-agent headers; cache directories; clipboard access; log files without explicit permissions                                                             |
| INFO (positive) | 4     | No hidden telemetry; .env files blocked from LLM read tool; test preload cleans API keys; LSP telemetry explicitly disabled                                                 |

---

## What You Need to Fix (Actionable Items)

### CRITICAL -- Fix Immediately

#### 1. MCP tool calls bypass permission system

**Where:** `packages/opencode/src/mcp/index.ts:121-149`

The `client.callTool()` invocation passes LLM-provided arguments directly to MCP servers with zero validation or user consent. The permission system (`packages/opencode/src/permission/`) only gates `bash` and `grep` tools.

**Fix:** Extend `PermissionNext.Service.ask()` to cover MCP tool calls. At minimum, require user consent for MCP tools that have side effects (write, execute). Consider a capability manifest for MCP servers.

```
LLM --> callTool(args) --> MCP Server   // current: no gate
LLM --> callTool(args) --> permission.ask() --> MCP Server  // target
```

#### 2. Plugin system has full shell access

**Where:** `packages/opencode/src/plugin/index.ts`

Plugins receive `$: Bun.$` in their input context, granting unrestricted shell execution. The `shell.env` hook lets plugins inject arbitrary environment variables into all bash/prompt contexts.

**Fix:** Remove `Bun.$` from plugin input or implement a capability-based permission model. Audit the `shell.env` hook to prevent injection of `LD_PRELOAD`, `PATH`, or similar sensitive variables.

---

### HIGH -- Fix Before Next Release

#### 3. Partial access token logged to CLI output

**Where:** `packages/opencode/src/cli/cmd/mcp.ts:630`

```ts
entry.tokens.accessToken.substring(0, 20) // too much exposure
```

**Fix:** Reduce to 4-6 characters or use a hash fingerprint:

```ts
entry.tokens.accessToken.substring(0, 4) + "..."
// or
crypto.createHash("sha256").update(entry.tokens.accessToken).digest("hex").substring(0, 8)
```

#### 4. Auto-share transmits all conversation data without per-session consent

**Where:** `packages/opencode/src/share/share-next.ts:214`

When `share: "auto"` is configured, `fullSync` sends ALL messages, parts, and file diffs to `opncd.ai` with only a 1-second debounce. This includes source code, command output, and any secrets that appeared in the conversation.

**Fix:** Add content filtering before transmission (redact potential secrets). Consider a confirmation prompt even in auto mode for sessions containing sensitive patterns.

---

### MEDIUM -- Plan for Next Sprint

#### 5. Set file permissions on SQLite database

**Where:** `packages/opencode/src/storage/db.ts`, `db.bun.ts`, `db.node.ts`

The database contains access tokens, refresh tokens, and full conversation history but has no explicit permissions set. It inherits the process umask (typically `0o644` -- world-readable).

**Fix:** Set `0o600` on the database file after creation, matching the pattern already used for `auth.json` and `mcp-auth.json`.

#### 6. Secrets injected into process.env are inherited by child processes

**Where:** `packages/opencode/src/config/config.ts:93,188-189` and `provider/provider.ts:268-276,487-491`

`WellKnown` auth tokens, console tokens, and AWS/SAP keys are set directly on `process.env`. Every spawned subprocess (bash tool, MCP servers, LSP servers, plugins) inherits these.

**Fix:** Use per-process environment overrides in `spawn()` calls instead of polluting `process.env`. For `Bun.spawn`, pass `env` in the options object.

#### 7. Tool output directory stores unredacted command output for 7 days

**Where:** `packages/opencode/src/tool/truncation-dir.ts`, `truncate-effect.ts`

Full bash/grep output (which may contain env vars, credentials, or sensitive file contents) is stored at `$XDG_DATA_HOME/opencode/tool-output/` with no permissions and no content filtering.

**Fix:** Set `0o600` on output files. Consider a shorter retention period (24h). Optionally, apply a regex-based redaction pass before writing (common patterns: `API_KEY=`, `token=`, `password=`, `secret=`).

---

## Architecture-Level Observations

### Permission System Coverage

```
                        Permission System
                              |
          +-------------------+-------------------+
          |                   |                   |
      Bash Tool           Grep Tool         (nothing else)
     (gated)              (gated)
                                              |
                                    +---------+---------+
                                    |                   |
                                MCP Tools          Plugins
                               (ungated)          (ungated)
```

The permission system is well-designed for its current scope but needs to be extended to MCP and plugins to close the gap.

### Credential Storage Architecture

```
auth.json (0o600) ---- API keys, OAuth tokens
mcp-auth.json (0o600) - MCP OAuth tokens, client secrets
opencode.db (umask) --- access_token, refresh_token columns (plaintext)
process.env ----------- injected tokens visible to all children
```

The inconsistency between file-level protection (auth.json has 0o600) and database-level protection (none) is a gap. Credentials should have uniform protection.

### Data Exfiltration Vectors (Ranked)

1. **AI Provider APIs** -- unavoidable; full conversation including code sent on every LLM call
2. **Session Sharing** -- configurable; can send everything automatically
3. **MCP Tool Calls** -- LLM-directed; no user consent gate
4. **Exa Search** -- LLM-generated queries may leak project details
5. **Tool Output Storage** -- local but unprotected; 7-day window

---

## What Is Done Well

- No hidden telemetry or analytics SDKs in runtime code
- `.env` files are blocked from LLM read tool access
- LSP server telemetry explicitly disabled
- Test suite cleans API keys from environment
- Permission system for bash/grep is well-structured with arity-based command grouping
- Process tree cleanup on shutdown for MCP servers
- Share feature defaults to manual mode with clear disable mechanisms
- Update checks can be disabled via config or environment variable

---

## Testing Recommendations

Add the following test cases to the security test suite:

1. **Permission bypass test:** Verify MCP tool calls trigger permission checks (once implemented)
2. **Environment leakage test:** Verify spawned subprocesses do not inherit credential env vars
3. **File permission test:** Verify SQLite DB, log files, and tool output files have `0o600` permissions
4. **Token logging test:** Verify no more than 4-6 characters of any token appear in log/CLI output
5. **Auto-share content test:** Verify auto-share redacts patterns matching common secret formats
