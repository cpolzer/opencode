# Security Review -- Product Owner Summary

**Date:** 2026-03-20
**Project:** opencode
**Review Type:** Data exfiltration, external sharing, information leakage

---

## What Was Reviewed

A comprehensive security review analyzed how opencode handles user data -- specifically where data leaves the application, what gets stored locally, and whether sensitive information could be exposed unintentionally. Five audit areas were examined: network communication, telemetry, credential handling, data persistence, and command execution.

Detailed technical reports are available in [`./security-reviews/`](./security-reviews/).

---

## Overall Assessment

opencode has a **strong privacy posture** with no hidden telemetry, no third-party analytics, and user-configurable data sharing. However, there are **gaps in the permission model** that could allow an AI model to perform actions without user consent, and **credential storage lacks encryption**.

---

## Key Findings

### What opencode does well

| Area                       | Finding                                                                                       |
| -------------------------- | --------------------------------------------------------------------------------------------- |
| **No hidden analytics**    | Zero third-party analytics SDKs (no Sentry, Mixpanel, Google Analytics, etc.) in runtime code |
| **Sharing is opt-in**      | Session sharing defaults to manual -- users must explicitly choose to share each session      |
| **Disable switches exist** | Both sharing and auto-updates can be disabled via config or environment variables             |
| **Env file protection**    | The AI assistant is blocked from reading `.env` files containing user secrets                 |

### What needs attention

#### 1. AI assistant can run some tools without asking the user -- CRITICAL

**Impact:** When users configure MCP (Model Context Protocol) servers, the AI assistant can call any tool those servers expose -- including tools that modify files, run commands, or access external services -- without asking for permission first.

**User story impact:** Users who install MCP servers expect the same permission prompts they see for bash commands. They currently do not get them for MCP tools.

**Recommendation:** Extend the existing permission prompt system to cover MCP tool calls. This is the highest-priority item.

#### 2. "Auto-share" mode sends everything -- HIGH

**Impact:** When a user enables `share: "auto"` in their config, all conversation content -- including source code, command output, and file diffs -- is automatically transmitted to the sharing server (`opncd.ai`) without per-session consent.

**User story impact:** Users who enable auto-share may not realize the full scope of what gets shared, especially if their conversations contain proprietary code or sensitive output.

**Recommendation:**

- Add a prominent warning when users enable auto-share
- Consider adding content filtering to redact potential secrets before sharing
- Consider a "first share" confirmation even in auto mode

#### 3. Credentials stored without encryption -- MEDIUM

**Impact:** API keys, OAuth tokens, and session secrets are stored as plain text in JSON files and an SQLite database on the user's machine. While auth files have restricted file permissions (only the user can read them), the database file does not.

**User story impact:** On shared machines or in environments where multiple processes run as the same user, credentials could be read by other software.

**Recommendation:** Set restricted file permissions on the database. Long-term, integrate with OS-level credential storage (macOS Keychain, Windows Credential Manager).

#### 4. Third-party plugins have unrestricted access -- HIGH

**Impact:** The plugin system grants full shell access to any installed plugin without sandboxing or permission checks.

**User story impact:** A malicious or compromised plugin could execute arbitrary commands, read files, or exfiltrate data without the user being aware.

**Recommendation:** Implement a capability-based permission model for plugins, similar to the existing bash tool permission system.

---

## Data Flow Overview

Understanding where user data goes:

```
                    User's Code & Conversations
                              |
              +---------------+---------------+
              |               |               |
         Always sent     User-initiated    Stored locally
              |               |               |
      AI Provider APIs   Share (opncd.ai)  SQLite database
      (Anthropic, OpenAI,    |              Log files
       Google, etc.)    Manual or Auto     Tool output
                                           Git snapshots
```

### What is sent externally

| Destination                           | Data                             | User control                           |
| ------------------------------------- | -------------------------------- | -------------------------------------- |
| AI provider (Anthropic, OpenAI, etc.) | Full conversation including code | User chooses provider                  |
| opncd.ai (sharing)                    | Full session content, diffs      | Manual (default), auto, or disabled    |
| Exa (search)                          | AI-generated search queries      | Triggered by AI during web/code search |
| Package registries                    | None (version check only)        | Can be disabled                        |

### What is NOT sent externally

- No usage analytics or telemetry
- No system information or device fingerprinting
- No crash reports
- No background phone-home calls

---

## Privacy Feature Inventory

| Feature            | Default                     | User can disable?   | Config key                                                |
| ------------------ | --------------------------- | ------------------- | --------------------------------------------------------- |
| Session sharing    | Manual (opt-in per session) | Yes                 | `share: "disabled"` or `OPENCODE_DISABLE_SHARE=true`      |
| Auto-update checks | Enabled                     | Yes                 | `autoupdate: false` or `OPENCODE_DISABLE_AUTOUPDATE=true` |
| OpenTelemetry      | Disabled                    | N/A (opt-in)        | `experimental.openTelemetry: true` to enable              |
| LSP telemetry      | Disabled                    | N/A (hardcoded off) | --                                                        |

---

## Risk Matrix

| Risk                                | Likelihood | Impact | Severity     | Mitigation status                                                |
| ----------------------------------- | ---------- | ------ | ------------ | ---------------------------------------------------------------- |
| MCP tool abuse via prompt injection | Medium     | High   | **CRITICAL** | No mitigation -- permission system does not cover MCP            |
| Plugin executes malicious code      | Low        | High   | **HIGH**     | No mitigation -- plugins have full access                        |
| Auto-share leaks proprietary code   | Medium     | Medium | **HIGH**     | Partial -- user must explicitly enable, but no content filtering |
| Credential theft from disk          | Low        | High   | **MEDIUM**   | Partial -- auth files have 0o600, database does not              |
| Token exposure in CLI output        | Low        | Medium | **HIGH**     | None -- 20 characters of token printed                           |
| Tool output leaks secrets           | Low        | Medium | **MEDIUM**   | None -- stored for 7 days without redaction                      |

---

## Recommended Roadmap

### Sprint 1 (Immediate)

1. Extend permission system to MCP tool calls
2. Reduce token logging from 20 characters to 4
3. Set file permissions on SQLite database

### Sprint 2 (Near-term)

4. Add content filtering/redaction for auto-share
5. Add prominent warning when enabling auto-share
6. Plugin sandboxing design and implementation

### Sprint 3 (Medium-term)

7. OS keychain integration for credential storage
8. Tool output redaction and shorter retention
9. First-run privacy notice documenting all external endpoints

---

## Compliance Considerations

| Framework | Relevance                                                                                                                                                                                                          |
| --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **GDPR**  | Session sharing transmits conversation data (potentially containing PII) to external servers. A Data Processing Agreement may be needed with the sharing service provider. Users should be informed of data flows. |
| **SOC 2** | Plaintext credential storage and lack of encryption at rest would be flagged. MCP permission bypass would be flagged as an access control gap.                                                                     |
| **HIPAA** | If used in healthcare contexts, the auto-share feature and AI provider transmission of conversation data would need to be evaluated against PHI requirements.                                                      |

These are observations, not legal advice. Consult with legal/compliance teams for formal assessment.
