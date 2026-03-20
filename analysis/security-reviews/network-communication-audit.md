# Network Communication Audit

**Date:** 2026-03-20
**Scope:** All external network communication across the opencode monorepo
**Severity Classification:** CRITICAL / HIGH / MEDIUM / LOW / INFO

---

## Executive Summary

A thorough audit identified **65+ network touchpoints** across **22 categories**. The primary data exfiltration vectors are LLM provider API calls (which transmit full conversation history including source code), session sharing to `opncd.ai`, and web/code search tools. No undisclosed or covert network channels were found.

---

## Critical Privacy Touchpoints (User Code/Conversations Sent Externally)

### 1. LLM Provider API Calls -- CRITICAL

| Detail          | Value                                                                              |
| --------------- | ---------------------------------------------------------------------------------- |
| **File**        | `packages/opencode/src/session/llm.ts:173`                                         |
| **What**        | `streamText()` sends full conversation history to configured AI provider           |
| **Data sent**   | Complete message history, system prompts, tool call results, source code fragments |
| **Destination** | User-configured AI provider (Anthropic, OpenAI, Google, etc.)                      |

### 2. Custom Fetch Wrapper -- CRITICAL

| Detail          | Value                                                  |
| --------------- | ------------------------------------------------------ |
| **File**        | `packages/opencode/src/provider/provider.ts:~456`      |
| **What**        | All AI traffic flows through this custom fetch wrapper |
| **Data sent**   | Full request bodies including conversation data        |
| **Destination** | AI provider API endpoints                              |

### 3. ChatGPT Codex Proxy -- CRITICAL

| Detail          | Value                                       |
| --------------- | ------------------------------------------- |
| **File**        | `packages/opencode/src/plugin/codex.ts:494` |
| **What**        | Proxies requests to ChatGPT Codex endpoint  |
| **Data sent**   | Full conversation data                      |
| **Destination** | OpenAI Codex API                            |

### 4. GitHub Copilot Proxy -- CRITICAL

| Detail          | Value                                         |
| --------------- | --------------------------------------------- |
| **File**        | `packages/opencode/src/plugin/copilot.ts:137` |
| **What**        | Proxies requests to GitHub Copilot            |
| **Data sent**   | Full conversation data                        |
| **Destination** | GitHub Copilot API                            |

### 5. Session Sharing -- HIGH

| Detail          | Value                                                            |
| --------------- | ---------------------------------------------------------------- |
| **File**        | `packages/opencode/src/share/share-next.ts:214`                  |
| **What**        | Syncs complete session data externally                           |
| **Data sent**   | Messages, code diffs, tool call results, model metadata          |
| **Destination** | `https://opncd.ai/api/share` (default) or enterprise/console URL |
| **Trigger**     | Manual by default; can be set to `"auto"` via config             |

### 6. Web/Code Search (Exa MCP) -- MEDIUM

| Detail          | Value                                                                                |
| --------------- | ------------------------------------------------------------------------------------ |
| **Files**       | `packages/opencode/src/tool/websearch.ts:103`, `codesearch.ts:85`                    |
| **What**        | Sends LLM-generated search queries to Exa search API                                 |
| **Data sent**   | Search queries constructed by the LLM (may contain code snippets or project details) |
| **Destination** | Exa API                                                                              |

---

## Non-Sensitive Network Calls

### Authentication Flows (No User Content)

| Category        | Files                                            | Destination                  |
| --------------- | ------------------------------------------------ | ---------------------------- |
| Codex OAuth     | `plugin/codex.ts`                                | OpenAI auth endpoints        |
| Copilot OAuth   | `plugin/copilot.ts`                              | GitHub device code endpoints |
| MCP OAuth       | `mcp/oauth-provider.ts`, `mcp/oauth-callback.ts` | MCP server auth endpoints    |
| Console Account | `account/effect.ts`                              | Console device code endpoint |

### Infrastructure Calls (No User Content)

| Category              | Count | Destination                                  | Data Sent               |
| --------------------- | ----- | -------------------------------------------- | ----------------------- |
| Version/update checks | 6     | GitHub API, npm, Homebrew, Chocolatey, Scoop | None (GET requests)     |
| LSP server downloads  | 10    | GitHub releases, npm                         | None (binary downloads) |
| Ripgrep download      | 1     | GitHub releases                              | None                    |
| Config well-known     | 2     | Provider config endpoints                    | None                    |
| Models.dev registry   | 2     | `models.dev` API                             | None                    |
| Remote instructions   | 1     | Configured URL                               | None (downloads .md)    |
| Web fetch tool        | 1     | User-specified URL                           | Minimal                 |

### Local-Only Communication

| Category           | Files           | Notes              |
| ------------------ | --------------- | ------------------ |
| VS Code extension  | localhost       | Extension API      |
| MCP OAuth callback | localhost:19876 | Local HTTP server  |
| PTY WebSocket      | localhost       | Terminal streaming |
| SDK client         | localhost       | Local API          |

### Server-Side / CI Only

| Category          | Files     | Notes                               |
| ----------------- | --------- | ----------------------------------- |
| Cloudflare Worker | `infra/`  | Support messages only, no user code |
| GitHub Actions    | `github/` | Repo metadata, not user code        |
| Scripts           | `script/` | Dev/CI tooling only                 |

---

## OpenTelemetry (Optional, Disabled by Default)

| Detail          | Value                                                 |
| --------------- | ----------------------------------------------------- |
| **Files**       | `session/llm.ts:243`, `agent/agent.ts:298`            |
| **What**        | AI SDK call spans with metadata                       |
| **Data sent**   | `userId` (config username), `sessionId`               |
| **Destination** | User-configured OTEL collector (none by default)      |
| **Opt-in**      | Requires `experimental.openTelemetry: true` in config |

---

## Absent Patterns (Not Found)

- No `axios`, `got`, `node-fetch`, `undici` direct usage
- No `http.request`, `https.request`, `dns.lookup`, `dns.resolve`, `tls.connect`
- No `navigator.sendBeacon` or browser beacon calls
- No covert/undisclosed network channels

---

## Recommendations

1. **Document all external endpoints** in a privacy policy / data flow diagram
2. **Audit auto-share mode** -- when enabled, all conversation data (including source code) is transmitted automatically with only a 1-second debounce
3. **Consider data minimization** for LLM calls -- evaluate whether full conversation history is always necessary or if truncation/summarization could reduce exposure
4. **Add network activity indicators** in the UI so users can see when data is being transmitted externally
5. **Review Exa search queries** -- LLM-generated queries may inadvertently include sensitive code or project details
