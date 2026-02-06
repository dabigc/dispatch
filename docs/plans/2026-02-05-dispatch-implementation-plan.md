# Dispatch — Raycast Extension Implementation Plan

**Date:** 2026-02-05  
**Status:** Ready for Implementation  
**Design:** [2026-02-05-dispatch-raycast-extension-design.md](./2026-02-05-dispatch-raycast-extension-design.md)  
**Related to:** #1

---

## Overview

This plan breaks down the Dispatch Raycast extension implementation into 2-5 minute tasks, each assigned to a specialist agent or generic subagent for mixed TypeScript/React work.

### Task Categories

1. **Project Setup** — Raycast extension scaffolding
2. **Core Infrastructure** — Config, logging, types
3. **Gateway Integration** — API client and data fetching
4. **UI Components** — Commands and views
5. **Testing** — Unit and integration tests
6. **Documentation** — README and contribution guides

---

## Phase 1: Project Setup & Infrastructure

### Task 1.1: Initialize Raycast Extension Project
**Agent:** generic-subagent  
**Time:** 3 minutes  
**Steps:**
1. Run `npm create @raycast/extension@latest` in dispatch repo root
2. Configure package.json with extension metadata:
   - Name: "Dispatch"
   - Author: "dabigc"
   - License: "MIT"
   - Commands: "dispatch" and "sessions"
3. Create directory structure:
   ```
   src/
   ├── dispatch.tsx
   ├── sessions.tsx
   ├── components/
   ├── lib/
   └── hooks/
   ```
4. Verify TypeScript config includes strict mode
5. Test build with `npm run build`

**Deliverable:** Clean Raycast extension project structure

---

### Task 1.2: Create TypeScript Type Definitions
**Agent:** generic-subagent  
**Time:** 4 minutes  
**Steps:**
1. Create `src/lib/types.ts`
2. Define interfaces matching Gateway API responses:
   ```typescript
   interface Session {
     sessionKey: string;
     displayName?: string;
     channel: string;
     model?: string;
     status: string;
     lastActivity?: number;
     tokenUsage?: number;
   }
   
   interface Message {
     role: "user" | "assistant" | "system";
     content: string;
     timestamp: number;
   }
   
   interface GatewayConfig {
     url: string;
     token: string;
     source: "auto-discover" | "manual";
   }
   
   interface SessionsListResponse {
     sessions: Session[];
   }
   
   interface SessionHistoryResponse {
     sessionKey: string;
     messages: Message[];
   }
   
   interface SendMessageResponse {
     success: boolean;
     response?: string;
     error?: string;
   }
   ```
3. Export all interfaces
4. Add JSDoc comments for clarity

**Deliverable:** `src/lib/types.ts` with complete type definitions

---

### Task 1.3: Implement Logger Utility
**Agent:** generic-subagent  
**Time:** 2 minutes  
**Steps:**
1. Create `src/lib/logger.ts`
2. Implement logger following design spec:
   ```typescript
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
3. Add token redaction helper:
   ```typescript
   export const redactToken = (token: string): string => {
     if (token.length <= 8) return "***";
     return `${token.slice(0, 4)}...${token.slice(-4)}`;
   };
   ```
4. Export logger and utilities

**Deliverable:** `src/lib/logger.ts` with privacy-safe logging

---

### Task 1.4: Implement Config Auto-Discovery
**Agent:** generic-subagent  
**Time:** 5 minutes  
**Steps:**
1. Create `src/lib/config.ts`
2. Implement `readOpenClawConfig()` function:
   - Read `~/.openclaw/openclaw.json`
   - Parse `gateway.url` (default: `http://localhost:3117`)
   - Parse `gateway.auth.token` or fallback to `process.env.OPENCLAW_GATEWAY_TOKEN`
   - Handle file not found gracefully
3. Implement `getGatewayConfig()` function:
   - Check Raycast preferences first (manual override)
   - Fall back to auto-discover
   - Log config source (redact token)
4. Add config validation:
   - Ensure URL is valid
   - Ensure token exists
   - Return error message if invalid
5. Implement simple 60-second cache to avoid repeated file reads

**Deliverable:** `src/lib/config.ts` with auto-discovery and preference override

---

## Phase 2: Gateway API Client

### Task 2.1: Implement Base API Client
**Agent:** fastapi-pro  
**Time:** 4 minutes  
**Steps:**
1. Create `src/lib/api.ts`
2. Implement `createApiClient(config: GatewayConfig)` factory function
3. Add base request handler with:
   - Authorization header: `Bearer ${token}`
   - Error handling for network failures
   - Timeout handling (30s default)
   - Response parsing and validation
4. Log all requests with timing (redact tokens)
5. Export typed client interface

**Deliverable:** Base API client with auth and error handling

---

### Task 2.2: Implement sessions_list Endpoint
**Agent:** fastapi-pro  
**Time:** 3 minutes  
**Steps:**
1. Add `sessionsList(messageLimit?: number)` method to API client
2. Make GET request to `/sessions/list`
3. Include `messageLimit` query param if provided
4. Parse response into `SessionsListResponse` type
5. Handle errors:
   - 401: Authentication failure
   - 503: Gateway unavailable
   - Network timeout
6. Return typed response or throw descriptive error

**Deliverable:** Working `sessionsList()` API method

---

### Task 2.3: Implement sessions_send Endpoint
**Agent:** fastapi-pro  
**Time:** 3 minutes  
**Steps:**
1. Add `sessionsSend(sessionKey: string, message: string, waitForResponse: boolean)` method
2. Make POST request to `/sessions/send`
3. Include body: `{ sessionKey, message }`
4. Set timeout: 5s for fire-and-forget, 30s for wait-response
5. Parse response into `SendMessageResponse` type
6. Handle errors with user-friendly messages
7. Log send action (not message content)

**Deliverable:** Working `sessionsSend()` API method

---

### Task 2.4: Implement sessions_history Endpoint
**Agent:** fastapi-pro  
**Time:** 2 minutes  
**Steps:**
1. Add `sessionsHistory(sessionKey: string, limit: number = 20)` method
2. Make GET request to `/sessions/history/${sessionKey}`
3. Include `limit` query param
4. Parse response into `SessionHistoryResponse` type
5. Handle 404 (session not found) gracefully
6. Return typed message array

**Deliverable:** Working `sessionsHistory()` API method

---

### Task 2.5: Implement session_status Endpoint (Model Override)
**Agent:** fastapi-pro  
**Time:** 2 minutes  
**Steps:**
1. Add `sessionStatus(sessionKey: string, model?: string)` method
2. Make POST request to `/session/status`
3. Include body: `{ sessionKey, model }` (if model provided)
4. Handle response and errors
5. Return success boolean

**Deliverable:** Working `sessionStatus()` API method for model override

---

## Phase 3: Data Fetching Hooks

### Task 3.1: Implement useSessions Hook
**Agent:** generic-subagent  
**Time:** 5 minutes  
**Steps:**
1. Create `src/hooks/useSessions.ts`
2. Implement custom hook using Raycast's `useCachedState` and `useEffect`
3. Features:
   - Fetch sessions on mount
   - Cache results for 30 seconds
   - Handle loading/error states
   - Auto-refresh capability
4. Return: `{ sessions, loading, error, refresh }`
5. Use API client from config
6. Log fetch operations

**Deliverable:** `useSessions.ts` hook with caching and error handling

---

## Phase 4: Dispatch Command (Send Message)

### Task 4.1: Create Dispatch Form Component
**Agent:** generic-subagent  
**Time:** 5 minutes  
**Steps:**
1. Create `src/dispatch.tsx`
2. Implement Form component with:
   - TextArea field for message (placeholder: "How can I help?")
   - Dropdown field for session selection
   - Load sessions with `useSessions()` hook
   - Default session to "main" or preference value
3. Add form validation (message required)
4. Handle loading state while sessions load
5. Show error toast if sessions fetch fails

**Deliverable:** Basic Dispatch form UI

---

### Task 4.2: Implement Send Message Action (Fire-and-Forget)
**Agent:** generic-subagent  
**Time:** 4 minutes  
**Steps:**
1. In `src/dispatch.tsx`, add form submission handler
2. Read `waitForResponse` preference
3. Implement fire-and-forget flow (default):
   - Call `sessionsSend(session, message, false)`
   - Show success toast: "Sent to [session]!"
   - Close form with `popToRoot()`
   - Handle errors with error toast
4. Log send action (not message content)

**Deliverable:** Working fire-and-forget send

---

### Task 4.3: Implement Send Message Action (Wait for Response)
**Agent:** generic-subagent  
**Time:** 4 minutes  
**Steps:**
1. In `src/dispatch.tsx`, add wait-response flow
2. When `waitForResponse: true`:
   - Show loading toast: "Waiting for response..."
   - Call `sessionsSend(session, message, true)`
   - Push to Detail view with response
   - Handle timeout (30s) with toast
3. Implement alternate action (⌘+Shift+Enter) to invert preference
4. Add action panel with both options

**Deliverable:** Working wait-for-response flow with alternate action

---

## Phase 5: Sessions Command (Dashboard)

### Task 5.1: Create Sessions List View
**Agent:** generic-subagent  
**Time:** 5 minutes  
**Steps:**
1. Create `src/sessions.tsx`
2. Implement List component with:
   - `useSessions()` hook for data
   - Loading state
   - Empty state: "No active sessions"
3. Map sessions to List.Item components:
   - Title: `displayName || sessionKey`
   - Subtitle: `channel`
   - Icon: Use Raycast icons based on channel type
4. Handle errors with error toast
5. Add manual refresh action

**Deliverable:** Basic sessions list UI

---

### Task 5.2: Add Session List Accessories
**Agent:** generic-subagent  
**Time:** 4 minutes  
**Steps:**
1. In `src/sessions.tsx`, enhance List.Item with accessories
2. Add model tag:
   - Show "Opus" or "Sonnet" based on session model
   - Use color-coded badges (green for Opus, blue for Sonnet)
3. Add status indicator:
   - "Active" if `lastActivity` within 2 minutes
   - "Idle 5m" for relative time format
4. Add token usage if available: "12.4k tokens"
5. Format with List.Item.Accessory components

**Deliverable:** Enhanced list items with model, status, and token info

---

### Task 5.3: Implement Send Message Action from Sessions
**Agent:** generic-subagent  
**Time:** 3 minutes  
**Steps:**
1. In `src/sessions.tsx`, add Action.Push to ActionPanel
2. Push to Dispatch form with pre-selected session:
   - Pass session key as prop
   - Pre-populate session dropdown
3. Bind to ⌘+Enter shortcut
4. Add icon and title: "Send Message"

**Deliverable:** Quick send action from sessions list

---

### Task 5.4: Implement View History Action
**Agent:** generic-subagent  
**Time:** 4 minutes  
**Steps:**
1. Create `src/components/SessionDetail.tsx`
2. Implement Detail component that:
   - Accepts sessionKey prop
   - Fetches history with `sessionsHistory()` on mount
   - Shows loading state
   - Renders messages in markdown format
3. Format messages:
   - **User:** message
   - **Assistant:** message
   - Include timestamps
4. In `src/sessions.tsx`, add Action.Push to SessionDetail
5. Bind to ⌘+H shortcut

**Deliverable:** Working history view for sessions

---

### Task 5.5: Implement Model Override Actions
**Agent:** generic-subagent  
**Time:** 3 minutes  
**Steps:**
1. In `src/sessions.tsx`, add conditional actions to ActionPanel:
   - "Switch to Opus" (⌘+O) — only if current model is Sonnet
   - "Switch to Sonnet" (⌘+S) — only if current model is Opus
2. Implement handler:
   - Call `sessionStatus(sessionKey, newModel)`
   - Show success toast: "Switched [session] to [model]"
   - Refresh sessions list
3. Handle errors with error toast

**Deliverable:** Working model override actions

---

## Phase 6: Extension Preferences

### Task 6.1: Define Extension Preferences
**Agent:** generic-subagent  
**Time:** 2 minutes  
**Steps:**
1. Edit `package.json` to add preferences schema:
   ```json
   "preferences": [
     {
       "name": "gatewayUrl",
       "type": "textfield",
       "required": false,
       "title": "Gateway URL",
       "description": "Override auto-discovered gateway URL",
       "placeholder": "http://localhost:3117"
     },
     {
       "name": "gatewayToken",
       "type": "password",
       "required": false,
       "title": "Gateway Token",
       "description": "Override auto-discovered auth token"
     },
     {
       "name": "waitForResponse",
       "type": "checkbox",
       "required": false,
       "title": "Wait for Response",
       "description": "Default send behavior waits and displays response",
       "default": false
     },
     {
       "name": "defaultSession",
       "type": "textfield",
       "required": false,
       "title": "Default Session",
       "description": "Pre-selected session in Send form",
       "default": "main"
     }
   ]
   ```
2. Test preference UI in Raycast

**Deliverable:** Working extension preferences

---

### Task 6.2: Integrate Preferences with Config
**Agent:** generic-subagent  
**Time:** 2 minutes  
**Steps:**
1. Update `src/lib/config.ts` to read preferences using `getPreferenceValues()`
2. Prioritize manual preferences over auto-discovery
3. Update `getGatewayConfig()` to respect preference overrides
4. Log config source: "Using manual override" vs "Auto-discovered from ~/.openclaw/openclaw.json"
5. Test with and without manual preferences

**Deliverable:** Preferences integrated into config system

---

## Phase 7: Testing

### Task 7.1: Write Unit Tests for Config Module
**Agent:** test-automator  
**Time:** 5 minutes  
**Steps:**
1. Create `src/lib/__tests__/config.test.ts`
2. Test scenarios:
   - Auto-discovery succeeds
   - Auto-discovery fails, falls back to env var
   - Manual preference overrides auto-discovery
   - Invalid config returns error
   - Token redaction works correctly
3. Mock file system and environment variables
4. Use Jest or Vitest

**Deliverable:** Unit tests for config.ts

---

### Task 7.2: Write Unit Tests for API Client
**Agent:** test-automator  
**Time:** 5 minutes  
**Steps:**
1. Create `src/lib/__tests__/api.test.ts`
2. Test scenarios:
   - Successful API calls return parsed data
   - 401 errors throw authentication error
   - Network failures throw descriptive errors
   - Timeout handling works (mock slow responses)
   - Request logging includes timing
3. Mock fetch with MSW or similar
4. Test all API methods: sessionsList, sessionsSend, sessionsHistory, sessionStatus

**Deliverable:** Unit tests for api.ts

---

### Task 7.3: Write Integration Tests for Commands
**Agent:** test-automator  
**Time:** 5 minutes  
**Steps:**
1. Create `src/__tests__/dispatch.test.tsx` and `src/__tests__/sessions.test.tsx`
2. Use Raycast's testing utilities for component rendering
3. Test scenarios:
   - Dispatch form renders with session dropdown
   - Send message action calls API and shows toast
   - Sessions list renders sessions correctly
   - Actions trigger correct API calls
   - Error states display properly
4. Mock API client responses

**Deliverable:** Integration tests for commands

---

## Phase 8: Documentation & Polish

### Task 8.1: Write README.md
**Agent:** generic-subagent  
**Time:** 4 minutes  
**Steps:**
1. Create comprehensive README.md:
   - Project description
   - Features list
   - Installation instructions
   - Configuration (auto-discovery + manual)
   - Usage examples with screenshots
   - Development setup
   - Contributing guidelines
2. Include MIT license badge
3. Add link to design doc
4. Include OpenClaw gateway setup instructions

**Deliverable:** Complete README.md

---

### Task 8.2: Add Icon and Assets
**Agent:** generic-subagent  
**Time:** 2 minutes  
**Steps:**
1. Copy finalized icon from design doc to `assets/icon.png`
2. Ensure icon meets Raycast requirements:
   - PNG format
   - 512x512 pixels recommended
   - Transparent or solid background
3. Add any additional assets (light/dark variants if needed)
4. Update package.json to reference icon

**Deliverable:** Extension icon and assets

---

### Task 8.3: Write CONTRIBUTING.md
**Agent:** generic-subagent  
**Time:** 3 minutes  
**Steps:**
1. Create CONTRIBUTING.md:
   - Development setup instructions
   - Code style guidelines (Prettier, ESLint)
   - Testing requirements
   - PR process
   - Issue reporting guidelines
2. Link to design doc for context
3. Include agent-friendly task breakdown for future features

**Deliverable:** CONTRIBUTING.md

---

### Task 8.4: Final QA and Polish
**Agent:** generic-subagent  
**Time:** 5 minutes  
**Steps:**
1. Run full test suite, ensure all tests pass
2. Test extension manually in Raycast:
   - Auto-discovery with and without config file
   - Manual preference override
   - Send message (fire-and-forget + wait)
   - Sessions list with all actions
   - Model override
   - Error handling for unreachable gateway
3. Check logs for proper formatting and privacy (no tokens logged)
4. Lint and format all code
5. Verify TypeScript compilation with no errors

**Deliverable:** Fully tested and polished extension

---

## Phase 9: Release Preparation

### Task 9.1: Create Raycast Store Submission
**Agent:** generic-subagent  
**Time:** 3 minutes  
**Steps:**
1. Follow Raycast store submission guidelines
2. Prepare submission metadata:
   - Extension ID: `dispatch`
   - Author: `dabigc`
   - Categories: Productivity, Developer Tools
   - Description and screenshots
3. Ensure all store requirements met:
   - Icon quality
   - README completeness
   - License (MIT)
4. Submit for review

**Deliverable:** Raycast store submission

---

## Task Summary by Agent

### fastapi-pro (API Client)
- Task 2.1: Base API client
- Task 2.2: sessions_list endpoint
- Task 2.3: sessions_send endpoint
- Task 2.4: sessions_history endpoint
- Task 2.5: session_status endpoint

**Total:** 5 tasks, ~14 minutes

### test-automator (Testing)
- Task 7.1: Config module tests
- Task 7.2: API client tests
- Task 7.3: Command integration tests

**Total:** 3 tasks, ~15 minutes

### generic-subagent (TypeScript/React)
- Task 1.1: Project setup
- Task 1.2: Type definitions
- Task 1.3: Logger utility
- Task 1.4: Config auto-discovery
- Task 3.1: useSessions hook
- Task 4.1: Dispatch form
- Task 4.2: Fire-and-forget send
- Task 4.3: Wait-for-response send
- Task 5.1: Sessions list view
- Task 5.2: Session accessories
- Task 5.3: Send from sessions
- Task 5.4: View history action
- Task 5.5: Model override actions
- Task 6.1: Define preferences
- Task 6.2: Integrate preferences
- Task 8.1: README
- Task 8.2: Icon and assets
- Task 8.3: CONTRIBUTING
- Task 8.4: Final QA

**Total:** 19 tasks, ~61 minutes

---

## Estimated Total Time

- **Setup & Infrastructure:** 14 minutes
- **API Client:** 14 minutes
- **Hooks:** 5 minutes
- **Dispatch Command:** 13 minutes
- **Sessions Command:** 19 minutes
- **Preferences:** 4 minutes
- **Testing:** 15 minutes
- **Documentation:** 12 minutes
- **Release:** 3 minutes

**Grand Total:** ~99 minutes (~1.5 hours)

---

## Implementation Order

1. **Phase 1:** Setup and infrastructure (Tasks 1.1-1.4)
2. **Phase 2:** API client (Tasks 2.1-2.5)
3. **Phase 3:** Data hooks (Task 3.1)
4. **Phase 4 & 5:** UI components in parallel (Tasks 4.1-5.5)
5. **Phase 6:** Preferences (Tasks 6.1-6.2)
6. **Phase 7:** Testing (Tasks 7.1-7.3)
7. **Phase 8:** Documentation (Tasks 8.1-8.4)
8. **Phase 9:** Release (Task 9.1)

---

## Success Criteria

- ✅ All API endpoints working with proper error handling
- ✅ Auto-discovery and manual config both functional
- ✅ Dispatch command sends messages (fire + wait modes)
- ✅ Sessions command displays all active sessions
- ✅ Model override actions work
- ✅ History view shows message threads
- ✅ Unit and integration tests pass
- ✅ Extension passes Raycast store review
- ✅ README and CONTRIBUTING docs complete

---

## Notes for Implementation

- **API client first:** Foundation for all features
- **Test as you go:** Don't defer testing to the end
- **Log everything:** Debugging Raycast extensions requires good logs
- **Privacy matters:** Never log message content or full tokens
- **Raycast best practices:** Follow official guidelines for UX patterns

---

**Ready for implementation!** 🦀⚡
