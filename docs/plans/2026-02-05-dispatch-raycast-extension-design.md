# Dispatch — Raycast Extension for OpenClaw

**Date:** 2026-02-05  
**Status:** Design Complete  
**Repo:** `dabigc/dispatch`  
**License:** MIT

---

## Overview

**Name:** Dispatch  
**Type:** Raycast Extension (OSS)  
**Purpose:** Quick access to your OpenClaw assistant fleet from anywhere on macOS

### Commands (MVP)

| Command | Type | Description |
|---------|------|-------------|
| `Dispatch` | Form | Quick message entry, sends to OpenClaw |
| `Dispatch Sessions` | List | Dashboard showing all active sessions |

### Authentication

1. **Auto-discover** — Read `~/.openclaw/openclaw.json` for gateway URL and token
2. **Manual override** — Preferences allow custom gateway URL + token
3. **Validation** — Show error toast if gateway unreachable

### Tech Stack

- TypeScript + React (standard Raycast)
- `@raycast/api` for UI components
- `node-fetch` or native fetch for gateway API calls
- No external dependencies beyond Raycast essentials

---

## Dispatch Command

### User Flow

1. User invokes `Dispatch` (via Raycast search or hotkey)
2. Form appears with text field + session dropdown
3. Session dropdown defaults to "main", populated from `sessions_list`
4. User types message, hits Enter
5. **Behavior determined by preference:**
   - If `waitForResponse: false` (default) → Toast "Sent to [session]!" → Raycast closes
   - If `waitForResponse: true` → Loading state → Detail view with response
6. **Alt action always available** — Inverts the default (wait if default is fire, fire if default is wait)

### Preferences

| Preference | Type | Default | Description |
|------------|------|---------|-------------|
| Wait for Response | Checkbox | false | When enabled, default action waits and displays response |

### Form Fields

| Field | Type | Default | Notes |
|-------|------|---------|-------|
| Message | TextArea | Empty | Required, placeholder: "How can I help?" |
| Session | Dropdown | "main" | Populated dynamically from gateway |

### Actions

| Action | Shortcut | Behavior |
|--------|----------|----------|
| Send (Primary) | ⌘+Enter | Follows preference (wait or fire) |
| Send (Alternate) | ⌘+Shift+Enter | Opposite of preference |
| Cancel | Escape | Close form |

### API Calls

- `sessions_list` — On form mount, populate session dropdown
- `sessions_send` — On submit, with `sessionKey` and `message`

### Error Handling

- Gateway unreachable → Error toast with "Check OpenClaw gateway"
- Session not found → Error toast, refresh session list
- Timeout (30s for "Wait" mode) → Toast with "Response taking longer, check later"

---

## Dispatch Sessions Command

### User Flow

1. User invokes `Dispatch Sessions`
2. List view loads with all sessions from `sessions_list`
3. Each row shows session info at a glance
4. Select a session → ActionPanel with available actions

### List Item Display

| Element | Source | Example |
|---------|--------|---------|
| Title | `displayName` or `sessionKey` | "main", "donarr" |
| Subtitle | Channel | "discord", "webchat" |
| Accessories | Model + Status | `[Opus]` `Active 2m ago` |

### Accessories (right side)

- **Model tag** — "Opus" or "Sonnet" (color-coded)
- **Status** — "Active" / "Idle 5m" / "Idle 2h" (relative time)
- **Token usage** — "12.4k tokens" (if available from API)

### Actions (per session)

| Action | Shortcut | Description |
|--------|----------|-------------|
| Send Message | ⌘+Enter | Opens Send form pre-filled with this session |
| View History | ⌘+H | Push to Detail view with recent messages |
| Switch to Opus | ⌘+O | Model override (only if currently Sonnet) |
| Switch to Sonnet | ⌘+S | Model override (only if currently Opus) |

### API Calls

- `sessions_list` — On mount, with `messageLimit: 0` for list view
- `sessions_history` — When "View History" selected, fetch last 20 messages
- `session_status` — For model override action, with `model` param

---

## Configuration & Preferences

### Auto-Discovery Logic

On extension load:
1. Check for `~/.openclaw/openclaw.json`
2. Parse for `gateway.url` (default: `http://localhost:3117`)
3. Parse for `gateway.auth.token` or fall back to env `OPENCLAW_GATEWAY_TOKEN`
4. If manual override set in preferences, use those instead

### Extension Preferences

| Preference | Type | Default | Description |
|------------|------|---------|-------------|
| Gateway URL | TextField | Auto-discover | Override gateway endpoint |
| Gateway Token | Password | Auto-discover | Override auth token |
| Wait for Response | Checkbox | false | Default send behavior |
| Default Session | TextField | "main" | Pre-selected session in Send form |

### Caching Strategy

- **Session list** — Cache for 30 seconds (avoid hammering gateway on rapid opens)
- **Session history** — No cache (always fresh when explicitly requested)
- **Gateway config** — Cache until preferences change

### Error States

| Scenario | User Feedback |
|----------|---------------|
| No config found + no manual override | Toast: "Configure gateway in extension preferences" |
| Gateway unreachable | Toast: "Can't reach OpenClaw gateway" + retry action |
| Invalid token | Toast: "Authentication failed — check gateway token" |
| No sessions found | Empty list with "No active sessions" message |

---

## Logging

### Approach

Raycast extensions use `console.log/warn/error` — logs appear in the Raycast developer console (⌘+⌥+J in dev mode).

### Log Levels

| Level | Use Case | Example |
|-------|----------|---------|
| `console.debug` | Verbose debugging (dev only) | Config resolution steps |
| `console.log` | General info | "Loaded 5 sessions" |
| `console.warn` | Recoverable issues | "Auto-discover failed, using manual config" |
| `console.error` | Failures | "Gateway request failed: 401" |

### What to Log

- **Startup** — Config source (auto-discover vs manual), gateway URL (redact token)
- **API calls** — Method, endpoint, response status, timing
- **Errors** — Full error context with stack traces
- **User actions** — Command invoked, session selected, message sent (not message content)

### Implementation

```typescript
// src/lib/logger.ts
import { environment } from "@raycast/api";

const LOG_PREFIX = "[Dispatch]";

export const logger = {
  debug: (...args: unknown[]) => {
    if (environment.isDevelopment) console.debug(LOG_PREFIX, ...args);
  },
  info: (...args: unknown[]) => console.log(LOG_PREFIX, ...args),
  warn: (...args: unknown[]) => console.warn(LOG_PREFIX, ...args),
  error: (...args: unknown[]) => console.error(LOG_PREFIX, ...args),
};
```

### Privacy

- Never log message content or full tokens
- Redact sensitive fields: `token: "abc...xyz"`

---

## Project Structure

```
dispatch/
├── src/
│   ├── dispatch.tsx          # Dispatch command (send message)
│   ├── sessions.tsx          # Dispatch Sessions command
│   ├── components/
│   │   └── SessionDetail.tsx # History detail view
│   ├── lib/
│   │   ├── api.ts            # Gateway API client
│   │   ├── config.ts         # Auto-discovery + preferences
│   │   ├── logger.ts         # Logging utility
│   │   └── types.ts          # TypeScript interfaces
│   └── hooks/
│       └── useSessions.ts    # Session list fetching + caching
├── assets/
│   └── icon.png              # Dispatch icon
├── package.json
├── tsconfig.json
└── README.md
```

---

## MVP Scope

- `Dispatch` command (form + send)
- `Dispatch Sessions` command (list + history + model override)
- Auto-discovery + manual config
- Basic error handling
- Logging

---

## Future Releases

| Feature | Priority | Notes |
|---------|----------|-------|
| Copy Session Key | Low | Simple action addition |
| Open in Browser | Low | Needs webchat URL pattern |
| Spawn Sub-agent | Medium | Requires agent picker UI |
| Command Palette | Medium | Gateway restart, cron triggers |
| Conversation Search | High | Full-text search across history |

---

## Raycast Store Publishing

- Extension ID: `dispatch`
- Author: `dabigc`
- License: MIT
- Categories: Productivity, Developer Tools

---

## Icon

Style: To be determined — should evoke "dispatch", command center, or fleet coordination. Options: radio waves, send arrow, command badge.
