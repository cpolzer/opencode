# Telemetry and Analytics Audit

**Date:** 2026-03-20
**Scope:** All telemetry, analytics, tracking, and usage reporting code across the opencode monorepo
**Severity Classification:** CRITICAL / HIGH / MEDIUM / LOW / INFO

---

## Executive Summary

The codebase has **no hidden telemetry or analytics** that phones home silently. There are **no third-party analytics SDKs** (Sentry, Mixpanel, Amplitude, Google Analytics, Segment, etc.) embedded in any runtime packages. The only data collection mechanisms are user-initiated session sharing, optional OpenTelemetry, and a CI-only stats script.

---

## Findings

### 1. PostHog Usage -- CI/CD Script Only (NOT in Application) -- INFO

| Detail              | Value                                                                                                                                    |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| **File**            | `script/stats.ts` (lines 3-29, 204-212)                                                                                                  |
| **What**            | Standalone CI/CD script that fetches public download counts from GitHub Releases API and npm, then sends aggregate statistics to PostHog |
| **Data collected**  | Total download counts (public data already available on GitHub/npm)                                                                      |
| **Where sent**      | `https://us.i.posthog.com/i/v0/e/` using `POSTHOG_KEY` from environment                                                                  |
| **User impact**     | **NONE** -- never executed by end users; runs as part of project maintainer workflows                                                    |
| **Can be disabled** | N/A -- not part of the application                                                                                                       |

---

### 2. OpenTelemetry -- Opt-In, Disabled by Default -- LOW

| Detail                  | Value                                                                               |
| ----------------------- | ----------------------------------------------------------------------------------- |
| **Config**              | `packages/opencode/src/config/config.ts` (lines 1213-1216)                          |
| **LLM calls**           | `packages/opencode/src/session/llm.ts` (lines 243-249)                              |
| **Agent calls**         | `packages/opencode/src/agent/agent.ts` (lines 298-303)                              |
| **What**                | Vercel AI SDK's `experimental_telemetry` emits OpenTelemetry spans for AI SDK calls |
| **Data collected**      | AI call spans with `userId` (config username) and `sessionId`                       |
| **Where sent**          | To whatever OpenTelemetry collector the user has configured (none by default)       |
| **Disabled by default** | Yes -- requires explicit `experimental.openTelemetry: true` in config               |

---

### 3. Session Sharing -- User-Initiated / Configurable -- MEDIUM

| Detail              | Value                                                                                                |
| ------------------- | ---------------------------------------------------------------------------------------------------- |
| **File**            | `packages/opencode/src/share/share-next.ts` (288 lines)                                              |
| **Config**          | `packages/opencode/src/config/config.ts` (lines 1061-1070)                                           |
| **What**            | Allows users to share conversation sessions via shareable links                                      |
| **Data collected**  | Session content (messages, AI responses, file diffs) -- only for sessions the user explicitly shares |
| **Where sent**      | `https://opncd.ai/api/share` (default), enterprise URL, or console account URL                       |
| **Modes**           | `"manual"` (default), `"auto"`, `"disabled"`                                                         |
| **Can be disabled** | `share: "disabled"` in config or `OPENCODE_DISABLE_SHARE=true` env var                               |

**Note:** When `"auto"` mode is enabled, all conversation data is transmitted without per-session user consent.

---

### 4. Auto-Update / Version Checking -- LOW

| Detail              | Value                                                                       |
| ------------------- | --------------------------------------------------------------------------- |
| **File**            | `packages/opencode/src/installation/index.ts` (lines 238-302)               |
| **Upgrade CLI**     | `packages/opencode/src/cli/upgrade.ts`                                      |
| **Desktop**         | `packages/desktop-electron/src/main/index.ts` (lines 292-347)               |
| **What**            | Checks for latest available version via public package registries           |
| **Data collected**  | Standard HTTP requests -- no user-identifying information beyond IP address |
| **Endpoints**       | GitHub API, npm registry, Homebrew formulae API, Chocolatey/Scoop APIs      |
| **Can be disabled** | `autoupdate: false` in config or `OPENCODE_DISABLE_AUTOUPDATE=true` env var |

---

### 5. User-Agent Headers Sent to AI Providers -- LOW

| Detail              | Value                                                                       |
| ------------------- | --------------------------------------------------------------------------- |
| **Provider**        | `packages/opencode/src/provider/provider.ts` (line 531)                     |
| **Codex**           | `packages/opencode/src/plugin/codex.ts` (lines 540, 564, 624)               |
| **Copilot**         | `packages/opencode/src/plugin/copilot.ts` (lines 125, 201, 231)             |
| **What**            | Sends standard User-Agent headers to AI provider APIs                       |
| **Data in headers** | `opencode/{VERSION}`, `(platform release; arch)`, originator/intent headers |
| **Where sent**      | Only to user-configured AI provider API endpoints                           |
| **Can be disabled** | No explicit toggle -- standard API identification headers                   |

---

### 6. System Information Collection -- Local Use Only -- INFO

| Detail              | Value                                                                 |
| ------------------- | --------------------------------------------------------------------- |
| **File**            | `packages/opencode/src/config/config.ts` (line 244)                   |
| **What**            | `os.userInfo().username` used as fallback for config `username` field |
| **Usage**           | Local display and OpenTelemetry metadata (if opt-in enabled)          |
| **Sent externally** | **No** (not by default)                                               |

---

### 7. LSP Telemetry -- Explicitly Disabled -- INFO (Positive)

| Detail         | Value                                                                    |
| -------------- | ------------------------------------------------------------------------ |
| **File**       | `packages/opencode/src/lsp/server.ts` (lines 1582-1584)                  |
| **What**       | Intelephense LSP server telemetry explicitly set to `{ enabled: false }` |
| **Assessment** | Good practice -- proactively disables third-party telemetry              |

---

### 8. UI Metrics Components -- Local Display Only -- INFO

| Detail        | Value                                                                                                                     |
| ------------- | ------------------------------------------------------------------------------------------------------------------------- |
| **Files**     | `packages/app/src/components/session/session-context-metrics.ts`, `packages/app/src/components/session-context-usage.tsx` |
| **What**      | UI components that calculate and display token usage and cost metrics                                                     |
| **Data sent** | **None** -- purely local computation and display                                                                          |

---

### 9. Environment Variables -- Local Configuration Only -- INFO

| Detail              | Value                                                                      |
| ------------------- | -------------------------------------------------------------------------- |
| **File**            | `packages/opencode/src/flag/flag.ts`                                       |
| **What**            | Reads `OPENCODE_*`, `HOME`, `SHELL`, `PATH`, `XDG_*` environment variables |
| **Usage**           | Local configuration and feature flags                                      |
| **Sent externally** | **No**                                                                     |

---

## What Was NOT Found

| Category                     | Status                       |
| ---------------------------- | ---------------------------- |
| Sentry SDK                   | Not present                  |
| Mixpanel                     | Not present                  |
| PostHog (runtime)            | Not present (only CI script) |
| Amplitude                    | Not present                  |
| Segment                      | Not present                  |
| Google Analytics             | Not present                  |
| Datadog                      | Not present                  |
| New Relic                    | Not present                  |
| Crash reporting              | Not present                  |
| Machine/device ID generation | Not present                  |
| IP address collection        | Not present                  |
| `navigator.sendBeacon`       | Not present                  |
| Silent phone-home on startup | Not present                  |
| Anonymous usage statistics   | Not present                  |
| A/B testing frameworks       | Not present                  |

**Note:** A `sentry.svg` file exists at `packages/ui/src/assets/icons/file-types/sentry.svg` -- this is purely a file-type icon for displaying Sentry config files in the UI, not an SDK.

---

## Overall Assessment

**The codebase is clean of hidden telemetry.** All external data transmission is:

1. User-initiated (session sharing) or explicitly opt-in (OpenTelemetry)
2. Configurable with disable flags
3. Transparent in purpose

## Recommendations

1. **Document the `"auto"` share mode prominently** -- users who enable it should understand that all conversation data is transmitted automatically
2. **Consider adding a first-run privacy notice** listing all external communication endpoints
3. **Maintain the current approach** of not bundling third-party analytics -- this is a strong privacy posture
