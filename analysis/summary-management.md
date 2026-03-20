# Security Review -- Management Summary

**Date:** 2026-03-20
**Project:** opencode
**Review Type:** Data exfiltration, external sharing, information leakage
**Classification:** Internal

---

## Purpose

This review assessed whether the opencode project leaks data externally, shares user content inappropriately, or exposes sensitive information. Five audit domains were examined: network communication, telemetry, credentials, data persistence, and command execution.

---

## Bottom Line

**opencode does not contain hidden telemetry or covert data collection.** The application has a strong privacy posture compared to competitors. However, three gaps need to be addressed to reduce risk of unintended data exposure.

---

## Three Things That Matter

### 1. Permission gap in AI tool execution -- CRITICAL

The application prompts users before the AI runs shell commands. However, this permission gate does **not** cover two categories of tool execution: MCP server tools and third-party plugins. This means the AI can perform actions through these channels -- including modifying files, running commands, or contacting external services -- without the user being asked.

**Business risk:** A prompt injection attack (malicious instruction hidden in code or a web page the AI reads) could exploit this gap to execute unauthorized actions on the user's machine.

**Remediation effort:** Medium (2-3 engineering weeks). The permission framework already exists and needs to be extended.

### 2. Auto-share mode can expose proprietary code -- HIGH

Users can enable an "auto-share" mode that automatically uploads all conversation content to an external server. This includes source code, command output, and file changes. The feature defaults to off (manual sharing), but once enabled, there is no content filtering or per-session consent.

**Business risk:** If an enterprise user enables auto-share while working with proprietary or regulated code, that code is transmitted externally without review.

**Remediation effort:** Low-Medium (1-2 engineering weeks). Add content filtering and a confirmation step.

### 3. Credentials stored without encryption -- MEDIUM

API keys and authentication tokens are stored as plain text on the user's disk. Two credential files use restricted file permissions (only the user can read them), but the main database file does not apply the same restriction.

**Business risk:** On shared systems or compromised machines, stored credentials could be extracted. This would be flagged in any SOC 2 or enterprise security audit.

**Remediation effort:** Low (< 1 week for file permissions; 3-4 weeks for OS keychain integration).

---

## What opencode Gets Right

| Area                        | Assessment                                                                                             |
| --------------------------- | ------------------------------------------------------------------------------------------------------ |
| **No hidden analytics**     | Zero third-party analytics SDKs in the application. No Sentry, no Mixpanel, no Google Analytics.       |
| **User-controlled sharing** | Sharing is off by default. Users must explicitly share each session. Sharing can be fully disabled.    |
| **Transparent data flows**  | All external communication serves a clear, user-visible purpose (AI queries, version checks, sharing). |
| **Disable switches**        | Both sharing and auto-updates can be turned off via configuration.                                     |

---

## Risk Overview

| #   | Finding                                                 | Severity | Likelihood | Business Impact                               | Effort to Fix      |
| --- | ------------------------------------------------------- | -------- | ---------- | --------------------------------------------- | ------------------ |
| 1   | AI tool execution without user consent (MCP/plugins)    | CRITICAL | Medium     | High -- unauthorized actions on user machines | 2-3 weeks          |
| 2   | Auto-share transmits all data without content filtering | HIGH     | Medium     | Medium -- proprietary code exposure           | 1-2 weeks          |
| 3   | Third-party plugins have unrestricted system access     | HIGH     | Low        | High -- malicious plugin risk                 | 3-4 weeks          |
| 4   | Credentials stored in plain text                        | MEDIUM   | Low        | High -- credential theft                      | < 1 week (partial) |
| 5   | Token partially logged in CLI output                    | HIGH     | Low        | Medium -- token exposure                      | < 1 day            |
| 6   | Sensitive command output stored 7 days                  | MEDIUM   | Low        | Medium -- data retention risk                 | < 1 week           |

---

## Comparison to Industry Standards

| Standard                          | Status                                                                          |
| --------------------------------- | ------------------------------------------------------------------------------- |
| **No hidden telemetry**           | Meets standard. Stronger than most commercial developer tools.                  |
| **Credential encryption at rest** | Below standard. Most security-conscious tools use OS keychains.                 |
| **User consent for AI actions**   | Partially meets standard. Good for shell commands; missing for MCP and plugins. |
| **Data sharing transparency**     | Meets standard. Clear controls, disable switches, documented endpoints.         |
| **SOC 2 readiness**               | Gaps in credential storage and access control would need remediation.           |

---

## Recommended Actions

| Priority | Action                                             | Owner                      | Timeline |
| -------- | -------------------------------------------------- | -------------------------- | -------- |
| **P0**   | Extend permission system to MCP tool calls         | Engineering                | Sprint 1 |
| **P0**   | Fix token logging (reduce from 20 to 4 characters) | Engineering                | Sprint 1 |
| **P1**   | Set file permissions on database file              | Engineering                | Sprint 1 |
| **P1**   | Add content filtering for auto-share               | Engineering                | Sprint 2 |
| **P1**   | Design plugin sandboxing model                     | Engineering + Architecture | Sprint 2 |
| **P2**   | OS keychain integration                            | Engineering                | Sprint 3 |
| **P2**   | First-run privacy notice                           | Product + Engineering      | Sprint 3 |

---

## Data Flow Summary (Non-Technical)

```
What the user types and their code
         |
         v
    +----+----+
    |         |
  Sent to   Stored on
  AI cloud  user's machine
    |         |
    v         v
 Anthropic  Database
 OpenAI     Log files
 Google     Snapshots
 etc.
    |
    +---> Optionally shared via link (user-controlled)
```

- **AI cloud:** Required for the application to function. User chooses which AI provider.
- **Local storage:** Conversation history, credentials, and tool output stored on the user's machine.
- **Sharing:** Off by default. Can be disabled entirely.
- **Analytics:** None. No usage data is collected or transmitted.

---

## Appendix: Report Inventory

| Report                         | Audience    | Location                                                                                               |
| ------------------------------ | ----------- | ------------------------------------------------------------------------------------------------------ |
| Network Communication Audit    | Technical   | [`security-reviews/network-communication-audit.md`](./security-reviews/network-communication-audit.md) |
| Telemetry and Analytics Audit  | Technical   | [`security-reviews/telemetry-analytics-audit.md`](./security-reviews/telemetry-analytics-audit.md)     |
| Credential and Secret Handling | Technical   | [`security-reviews/credential-secret-handling.md`](./security-reviews/credential-secret-handling.md)   |
| Data Persistence Audit         | Technical   | [`security-reviews/data-persistence-audit.md`](./security-reviews/data-persistence-audit.md)           |
| Subprocess Execution Audit     | Technical   | [`security-reviews/subprocess-execution-audit.md`](./security-reviews/subprocess-execution-audit.md)   |
| Developer Summary              | Engineering | [`summary-developer.md`](./summary-developer.md)                                                       |
| Product Owner Summary          | Product     | [`summary-product-owner.md`](./summary-product-owner.md)                                               |
| Management Summary             | Leadership  | This document                                                                                          |
