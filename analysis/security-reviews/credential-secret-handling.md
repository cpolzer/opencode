# Credential and Secret Handling Audit

**Date:** 2026-03-20
**Scope:** All credential storage, retrieval, transmission, and potential leakage across the opencode monorepo
**Severity Classification:** CRITICAL / HIGH / MEDIUM / LOW / INFO

---

## Executive Summary

All credentials are stored **in plaintext** -- on disk as JSON files, in SQLite databases, and in environment variables. There is **no keychain/keyring integration** and **no encryption at rest**. File permissions (`0o600`) are the only protection for on-disk credential files. One instance of partial secret logging was identified. Multiple OAuth flows exist, all storing tokens in plaintext. API keys are transmitted to LLM providers via standard SDK mechanisms (HTTP Authorization headers).

---

## Findings

### Finding 1: Primary Auth Store -- Plaintext JSON on Disk -- MEDIUM

| Detail          | Value                                                             |
| --------------- | ----------------------------------------------------------------- |
| **File**        | `packages/opencode/src/auth/effect.ts` (lines 36, 69-88)          |
| **Credentials** | Provider API keys, OAuth access/refresh tokens, expiry timestamps |
| **Storage**     | Plaintext JSON at `${XDG_DATA_HOME}/opencode/auth.json`           |
| **Protection**  | File mode `0o600` (owner read/write only)                         |
| **Leak risk**   | Any process running as the same user can read the file            |

Three credential types are stored:

- `OAuth`: access token, refresh token, expiry
- `Api`: raw API key string
- `WellKnown`: key + token pair

---

### Finding 2: MCP Auth Store -- Plaintext JSON on Disk -- MEDIUM

| Detail          | Value                                                                                            |
| --------------- | ------------------------------------------------------------------------------------------------ |
| **File**        | `packages/opencode/src/mcp/auth.ts` (lines 32, 66, 72)                                           |
| **Credentials** | OAuth access/refresh tokens, client IDs, client secrets, PKCE code verifiers, OAuth state values |
| **Storage**     | Plaintext JSON at `${XDG_DATA_HOME}/opencode/mcp-auth.json`                                      |
| **Protection**  | File mode `0o600`                                                                                |
| **Leak risk**   | Client secrets and code verifiers stored alongside tokens; same-user process exposure            |

---

### Finding 3: Account Tokens -- Plaintext in SQLite -- MEDIUM

| Detail          | Value                                                                                                                               |
| --------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| **Files**       | `packages/opencode/src/account/account.sql.ts` (lines 6-14), `account/repo.ts` (lines 111-147), `account/effect.ts` (lines 157-188) |
| **Credentials** | `access_token`, `refresh_token` for opencode console accounts                                                                       |
| **Storage**     | Plaintext `text()` columns in SQLite database                                                                                       |
| **Protection**  | None beyond filesystem permissions on the SQLite file                                                                               |
| **Leak risk**   | Readable by any same-user process; token refresh stores new tokens back to DB                                                       |

---

### Finding 4: Share Secrets -- Plaintext in SQLite, Transmitted Externally -- MEDIUM

| Detail           | Value                                                                                                 |
| ---------------- | ----------------------------------------------------------------------------------------------------- |
| **Files**        | `packages/opencode/src/share/share.sql.ts` (line 10), `share/share-next.ts` (lines 129-137, 218, 241) |
| **Credentials**  | Share secrets (authenticate share sync/delete operations)                                             |
| **Storage**      | Plaintext `text()` column in SQLite                                                                   |
| **Transmission** | Sent in POST/DELETE request bodies to external share APIs                                             |
| **Leak risk**    | Transmitted over network; depends on HTTPS enforcement at the API endpoint                            |

---

### Finding 5: No Keychain / Keyring Integration -- MEDIUM

| Detail     | Value                                                                                                  |
| ---------- | ------------------------------------------------------------------------------------------------------ |
| **Search** | Grep for `keychain`, `keyring`, `credential.store`, `SecretStorage`                                    |
| **Result** | Only hits are macOS code signing in GitHub Actions workflows                                           |
| **Impact** | **Zero user-facing credential store integration** -- all secrets rely solely on filesystem permissions |

---

### Finding 6: Environment Variable API Key Loading -- LOW

| Detail          | Value                                                                                                   |
| --------------- | ------------------------------------------------------------------------------------------------------- |
| **File**        | `packages/opencode/src/provider/provider.ts` (lines 958-968, 1128, 268-276, 487-491)                    |
| **Credentials** | `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GOOGLE_API_KEY`, `AWS_BEARER_TOKEN_*`, SAP AI Core service keys |
| **How**         | Iterates provider `env` arrays, reads from `process.env`, passes to AI SDK as `apiKey` option           |
| **Concern**     | AWS and SAP keys are written directly into `process.env`, making them visible to child processes        |

---

### Finding 7: Partial Access Token Logged to Console -- HIGH

| Detail             | Value                                                                                                                                              |
| ------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| **File**           | `packages/opencode/src/cli/cmd/mcp.ts` (line 630)                                                                                                  |
| **What**           | First 20 characters of MCP OAuth access token printed: `entry.tokens.accessToken.substring(0, 20)...`                                              |
| **Leak risk**      | **HIGH** -- partial token exposure in CLI output; 20 chars may be sufficient to reconstruct or brute-force the remainder depending on token format |
| **Recommendation** | Mask to at most 4-6 characters, or show only a hash/fingerprint                                                                                    |

---

### Finding 8: Config Substitution Injects Secrets into process.env -- MEDIUM

| Detail   | Value                                                                                            |
| -------- | ------------------------------------------------------------------------------------------------ |
| **File** | `packages/opencode/src/config/config.ts` (lines 93, 188-189)                                     |
| **What** | `WellKnown` auth tokens and console tokens set directly into `process.env`                       |
| **Risk** | Any spawned subprocess (tools, shell commands, MCP servers) inherits these environment variables |

**Also:** `packages/opencode/src/config/paths.ts` (lines 85-141) -- `{env:VAR}` and `{file:path}` substitution reads secrets from env vars and files into config values as plaintext.

---

### Finding 9: HTTP Basic Auth for Server -- MEDIUM

| Detail          | Value                                                                                                                         |
| --------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| **Files**       | `packages/opencode/src/server/server.ts` (lines 82-85), `cli/cmd/tui/worker.ts` (lines 151-155), `flag/flag.ts` (lines 39-40) |
| **Credentials** | `OPENCODE_SERVER_PASSWORD`, `OPENCODE_SERVER_USERNAME`                                                                        |
| **How**         | HTTP Basic Auth with `btoa` encoding (base64, NOT encryption)                                                                 |
| **Risk**        | If server is accessed over non-TLS, credentials are trivially interceptable                                                   |

---

## OAuth Flows (4 Separate Implementations)

### GitHub Copilot -- Device Code Flow -- LOW

| Detail   | Value                                                                   |
| -------- | ----------------------------------------------------------------------- |
| **File** | `packages/opencode/src/plugin/copilot.ts` (lines 257-260)               |
| **Flow** | Device code grant -> access token stored as refresh token in auth store |

### OpenAI Codex -- PKCE OAuth Flow -- LOW

| Detail   | Value                                                              |
| -------- | ------------------------------------------------------------------ |
| **File** | `packages/opencode/src/plugin/codex.ts` (line 477)                 |
| **Flow** | PKCE authorization code grant -> access token set as Bearer header |

### MCP Server -- OAuth with Local Callback -- MEDIUM

| Detail      | Value                                                                                           |
| ----------- | ----------------------------------------------------------------------------------------------- |
| **Files**   | `packages/opencode/src/mcp/oauth-provider.ts`, `mcp/oauth-callback.ts` (lines 88, 118)          |
| **Flow**    | Standard OAuth with local HTTP server on port 19876, CSRF via state parameter                   |
| **Concern** | Pending auth states logged at line 118; URL logged on error at line 88 may contain query params |

### Console Account -- Device Code Flow -- LOW

| Detail   | Value                                                     |
| -------- | --------------------------------------------------------- |
| **File** | `packages/opencode/src/account/effect.ts` (lines 157-188) |
| **Flow** | Device code grant -> tokens stored in SQLite              |

---

## Positive Findings

### .env File Read Protection -- INFO (Positive)

| Detail         | Value                                                                                                      |
| -------------- | ---------------------------------------------------------------------------------------------------------- |
| **File**       | `packages/opencode/test/tool/read.test.ts` (lines 158-163)                                                 |
| **What**       | The read tool blocks LLM access to `.env`, `.env.local`, `.env.production`, `.env.development.local` files |
| **Assessment** | Good -- prevents LLM tools from reading user `.env` files                                                  |

### Test Isolation Cleans API Keys -- INFO (Positive)

| Detail         | Value                                                                    |
| -------------- | ------------------------------------------------------------------------ |
| **File**       | `packages/opencode/test/preload.ts` (lines 55-62)                        |
| **What**       | Test preload script deletes API key env vars to prevent accidental usage |
| **Assessment** | Good -- proper test hygiene                                              |

---

## Recommendations

| Priority   | Recommendation                                                                                                 |
| ---------- | -------------------------------------------------------------------------------------------------------------- |
| **HIGH**   | Reduce partial access token logging in `cli/cmd/mcp.ts:630` from 20 chars to 4-6 max or use a hash fingerprint |
| **MEDIUM** | Integrate with OS keychains (macOS Keychain, Linux Secret Service) for `auth.json` and `mcp-auth.json`         |
| **MEDIUM** | Audit child process environment inheritance -- tokens in `process.env` are visible to all spawned subprocesses |
| **MEDIUM** | Replace PAT tokens in git URLs (`sync-zed.ts`) with `GIT_ASKPASS` or credential helpers                        |
| **LOW**    | Document that HTTP Basic Auth server is localhost-only; require TLS if network-exposed                         |
| **LOW**    | Consider encrypting SQLite columns containing `access_token`, `refresh_token`, and `secret`                    |
