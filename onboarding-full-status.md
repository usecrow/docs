# Crow Onboarding — Full Plan, Progress & Current Issues

**Branch:** `crow-onboarding-revamp` (all three repos)
**Worktrees:** `crow-backend-worktree-1`, `crow-sdk-worktree-1`, `crow-frontend-worktree-1`
**Date:** 2026-02-21

---

## Table of Contents

1. [High-Level Goal](#high-level-goal)
2. [Architecture](#architecture)
3. [Phase 1: SDK Fullscreen Mode — COMPLETE](#phase-1-sdk-fullscreen-mode--complete)
4. [Phase 2A: Backend — COMPLETE](#phase-2a-backend--complete)
5. [Phase 2B: SDK Onboarding Hook — COMPLETE](#phase-2b-sdk-onboarding-hook--complete)
6. [Phase 3: Dashboard UI — COMPLETE](#phase-3-dashboard-ui--complete)
7. [Current Task: Typewriter Intro Inside SDK — BROKEN](#current-task-typewriter-intro-inside-sdk--broken)
8. [What Has Been Changed (Latest Commits)](#what-has-been-changed-latest-commits)
9. [What's Broken & Needs Debugging](#whats-broken--needs-debugging)
10. [Files to Know About](#files-to-know-about)
11. [How to Test](#how-to-test)
12. [Known Issues / TODOs](#known-issues--todos)

---

## High-Level Goal

Make Crow **chat-first**: new users land in a fullscreen onboarding chat that guides them through setup, then shrinks to a sidebar copilot. The onboarding experience is built as a general Crow product feature — any customer can enable it for their own users.

---

## Architecture

```
Customer embeds <script> tag (no code changes needed)
         |
SDK fetches widget-config → sees onboardingEnabled: true
         |
SDK gets identity token → calls POST /resolve-agent
         |
Backend checks product_users.onboarding_completed_at
         |
  ├─ NULL → "onboarding" → SDK renders fullscreen
  └─ NOT NULL → "default" → SDK renders sidebar
         |
User chats with onboarding agent (agent_mode: "onboarding")
         |
Backend swaps system_prompt, injects complete_onboarding tool
         |
Agent calls complete_onboarding → backend marks user → SSE event
         |
SDK receives onboarding_complete → animates fullscreen → sidebar
```

For Crow's own dashboard (`crow-frontend`), a `forceOnboarding: true` flag skips the resolve-agent call and forces onboarding mode, since new dashboard users always need onboarding.

---

## Phase 1: SDK Fullscreen Mode — COMPLETE

**What:** A `mode` prop on `CrowCopilot`: `"fullscreen" | "sidebar"`.

**Key files:**
- `crow-sdk/packages/ui/src/components/copilot/FullscreenContainer.tsx` — fixed overlay wrapper
- `crow-sdk/packages/ui/src/components/copilot/FullscreenChatView.tsx` — ChatGPT-style layout (empty state + active chat)
- `crow-sdk/packages/ui/src/CrowCopilot.tsx` — mode state, expand/shrink buttons, smooth shrink animation

**Verified working:** fullscreen ↔ sidebar transitions, conversation persistence across mode changes, script tag + React SDK + JS API support.

---

## Phase 2A: Backend — COMPLETE

**What:** Onboarding agent support on the backend.

**Key changes:**
1. **Migration** (`migrations/add_onboarding_columns.sql`, ALREADY RUN):
   - `products`: `onboarding_enabled`, `onboarding_agent_name`, `onboarding_system_prompt`, `onboarding_welcome_message`
   - `product_users`: `onboarding_completed_at`

2. **Widget Config** (`app/api/routes/products.py`):
   - `/widget-config` returns `onboardingEnabled`, `onboardingAgentName`, `onboardingWelcomeMessage`

3. **Resolve Agent** (`POST /api/products/{id}/resolve-agent`):
   - Takes `identity_token`, checks `onboarding_completed_at`
   - Returns `{ agent: "onboarding"|"default", agentName?, welcomeMessage? }`

4. **Chat Endpoint** (`app/api/routes/widget.py`):
   - `agent_mode` field on requests; when `"onboarding"` + `onboarding_enabled`: swaps system prompt

5. **complete_onboarding Tool** (`app/services/langgraph/tools.py`):
   - Agent calls this when done → marks `onboarding_completed_at`, emits SSE event

6. **Dashboard CRUD** (`GET/PUT /api/products/{id}/onboarding`)

---

## Phase 2B: SDK Onboarding Hook — COMPLETE

**What:** SDK-side onboarding flow orchestration.

**Key changes:**
1. **useOnboarding hook** (`packages/ui/src/hooks/useOnboarding.ts`):
   - Resolves agent mode via localStorage cache → `/resolve-agent` API
   - `forceOnboarding` skips resolution
   - Returns `{ agent, agentName, welcomeMessage, markComplete, isResolved }`
   - Supports "returning" state (saved onboarding conversation exists)

2. **useChat** (`packages/ui/src/hooks/useChat.ts`):
   - `agentMode` option → sends `agent_mode` in POST body
   - `onOnboardingComplete` callback on SSE event

3. **CrowCopilot** (`packages/ui/src/CrowCopilot.tsx`):
   - `forceOnboarding` prop
   - When onboarding: disables conversation persistence, overrides agent name/welcome, forces fullscreen
   - `OnboardingResumePrompt` for returning users (Continue / Start Over)

4. **Script Tag Loader** (`crow-backend/widget/src/loaders/core-loader.tsx`):
   - Reads `data-force-onboarding` + `window.__CROW_CONFIG__.forceOnboarding`

---

## Phase 3: Dashboard UI — COMPLETE

**What:** Settings page to configure the onboarding agent.

**Key files:**
- `crow-frontend/src/pages/settings/OnboardingPage.tsx` — toggle, agent name, welcome, system prompt
- `crow-frontend/src/api/products.ts` — `getOnboarding()` / `updateOnboarding()`
- Route at `/agent/onboarding` with nav item in sidebar

---

## Current Task: Typewriter Intro Inside SDK — BROKEN

### Goal

Instead of the frontend rendering a separate `TypewriterIntro` overlay component, the typewriter animation should live **inside the SDK's `FullscreenChatView`**. When the widget enters fullscreen onboarding mode, the empty state (centered welcome text) gets replaced with a typewriter sequence, then reveals the chat.

### The Desired Flow

**First visit** (intro not seen):
1. Fullscreen view shows — top bar visible, content area is dark (`#030712`)
2. Typewriter plays: "Hi" → pulsing dots → "Welcome to Crow" → pause
3. Dark background fades to light (`#f9fafb`)
4. Welcome message slides in as a bot bubble from top
5. Input bar slides up from the bottom
6. User can now chat — localStorage flag set so intro won't replay

**Subsequent visits** (intro already seen):
1. Skip straight to the current empty state (centered welcome text + input bar)
2. No changes to existing behavior

### What Was Changed

Two SDK files were modified + one frontend file:

#### 1. `crow-sdk-worktree-1/packages/ui/src/components/copilot/FullscreenChatView.tsx`

Added a `useTypewriterIntro` hook with the typewriter state machine:
- Phases: `typing-hi` → `dots` → `clearing` → `typing-welcome` → `pause`
- Same timings as the old frontend `TypewriterIntro` (90ms per char, 1.6s dots, etc.)

Added `introPhase` state: `"typewriter" | "reveal" | "done"`
- `typewriter`: dark background, animated text, pulsing dots, cursor
- `reveal`: bg transitions dark→light, welcome bubble slides in, input slides up
- `done`: normal active chat layout

New props: `showIntroAnimation?: boolean`, `onIntroComplete?: () => void`

The `AnimatePresence` branching was changed from 2 states to 3:
```
introPhase === "typewriter" → dark typewriter UI
!hasUserMessages && !introCompleted → old centered empty state
else → active chat layout (also used for post-intro reveal)
```

#### 2. `crow-sdk-worktree-1/packages/ui/src/CrowCopilot.tsx`

Added localStorage check and prop wiring:
```ts
const introStorageKey = `crow_onboarding_intro_${productId}`;
const [introSeen, setIntroSeen] = useState(() => {
  return localStorage.getItem(introStorageKey) === "true";
});
const showIntroAnimation = isOnboarding && !introSeen;
```

`handleIntroComplete` sets the localStorage flag.

Both `showIntroAnimation` and `onIntroComplete` are passed to `FullscreenChatView` in the onboarding render block (~line 1449).

#### 3. `crow-frontend-worktree-1/src/pages/ChatOnboarding.tsx`

Removed the old `TypewriterIntro` overlay. The page now only renders the "Skip to dashboard" button. The SDK widget handles the intro animation internally.

### What's NOT Working

After clearing all `crow_onboarding_intro_*` localStorage keys and hard-refreshing:
- The typewriter animation does NOT appear
- The user sees the normal empty state (light background, centered welcome text, input bar)
- This means either `showIntroAnimation` is evaluating to `false`, or the `AnimatePresence` branching isn't routing to the typewriter state

The built widget (`crow-backend-worktree-1/app/static/crow-widget.js`) DOES contain the new code (verified by grepping for `crow-blink`), and the backend IS serving it (verified by `curl`). So it's not a build or serving issue.

### Likely Root Causes to Investigate

1. **localStorage key collision**: The SDK key is `crow_onboarding_intro_${productId}` — verify that the productId used at runtime matches what's expected, and that there isn't an existing key with a different productId.

2. **`isOnboarding` might be `false` by the time the component renders**: The `useOnboarding` hook has async resolution. If `introSeen` state initializes before `isOnboarding` flips to `true`, the `showIntroAnimation` computed value might not update correctly (though it should since both are reactive).

3. **`AnimatePresence` key conflict**: The `key="typewriter"` state might be exiting immediately due to the `mode="wait"` + the condition changing between renders. Add `console.log` statements in the component to trace `introPhase`, `showIntroAnimation`, and `isOnboarding` values.

4. **Browser caching the old widget JS**: Even with hard refresh, service workers or aggressive caching could serve stale JS. Try opening in incognito or disable cache in DevTools Network tab.

### How to Debug

1. Add `console.log("[FullscreenChatView]", { introPhase, showIntroAnimation, hasUserMessages })` at the top of the render
2. Add `console.log("[CrowCopilot] intro", { isOnboarding, introSeen, showIntroAnimation, introStorageKey })` after the intro state logic
3. Rebuild: `./scripts/build-worktree.sh 1`
4. Hard refresh with DevTools cache disabled
5. Check console output to see which values are wrong

---

## What Has Been Changed (Latest Commits)

### SDK (`crow-sdk-worktree-1`, branch `crow-onboarding-revamp`)
```
3cdea37 feat: add typewriter intro animation to fullscreen onboarding view
f2f99f8 Fix Start Over: prevent stale conversation from auto-loading
497de19 Add solid background to OnboardingResumePrompt
3f87687 Fix forceOnboarding: allow returning state when saved conv exists
b4d5120 Fix remaining secondaryText reference in OnboardingResumePrompt
28fee37 Fix type error: remove non-existent secondaryText color reference
841094c Wire onboarding resume flow into CrowCopilot
edd06cc Add OnboardingResumePrompt component and export it
7085c8b Add returning state and conversation persistence to useOnboarding
b900843 Fix onboarding: fresh conversation, correct name/welcome
45c735f Add forceOnboarding prop to skip resolve-agent for testing
...
```

### Frontend (`crow-frontend-worktree-1`, branch `crow-onboarding-revamp`)
```
2fd41e9 refactor: remove TypewriterIntro from ChatOnboarding page
694bfc6 Simplify onboarding: typewriter overlay only, script-tag widget handles chat
1434025 Rewrite onboarding: sequential slide-up reveal, skip widget on /onboarding
...
```

### Backend (`crow-backend-worktree-1`, branch `crow-onboarding-revamp`)
```
b7a5c19 Remove is_verified_user check from onboarding mode determination
f5a8503 Fix forceOnboarding: also check __CROW_CONFIG__
e843691 Add onboarding agent name and welcome message to widget-config
d6f8125 Add data-force-onboarding support to script tag loader
...
```

---

## Files to Know About

### SDK (where the typewriter work is)
| File | Purpose |
|------|---------|
| `packages/ui/src/components/copilot/FullscreenChatView.tsx` | **THE main file** — fullscreen layout with typewriter intro + reveal + empty + active states |
| `packages/ui/src/CrowCopilot.tsx` | Orchestrates everything — onboarding detection, intro animation flag, fullscreen rendering |
| `packages/ui/src/hooks/useOnboarding.ts` | Resolves which agent to show (onboarding vs default vs returning) |
| `packages/ui/src/hooks/useChat.ts` | Chat state — sends `agent_mode`, handles `onboarding_complete` SSE |
| `packages/ui/src/components/copilot/FullscreenContainer.tsx` | Fixed overlay wrapper for fullscreen mode |
| `packages/ui/src/components/copilot/OnboardingResumePrompt.tsx` | Continue / Start Over prompt for returning onboarding users |

### Frontend
| File | Purpose |
|------|---------|
| `src/pages/ChatOnboarding.tsx` | The `/onboarding` page — now just a "Skip to dashboard" button (TypewriterIntro removed) |
| `src/components/onboarding/TypewriterIntro.tsx` | OLD standalone typewriter component — no longer imported anywhere but file still exists |
| `index.html` | Loads widget script tag, has `forceOnboarding: true` for dev |

### Backend
| File | Purpose |
|------|---------|
| `widget/src/loaders/core-loader.tsx` | Script tag loader — reads `forceOnboarding` from config |
| `app/api/routes/widget.py` | Chat endpoint with `agent_mode` support |
| `app/services/langgraph_service.py` | LangGraph integration with `complete_onboarding` tool |

---

## How to Test

### Setup
- Backend: `crow-backend/venv/bin/uvicorn app.main:app --reload --port 8001` (from `crow-backend-worktree-1/`)
- Frontend: `npx vite --port 5174` (from `crow-frontend-worktree-1/`)
- Widget build: `./scripts/build-worktree.sh 1` (from repo root)
- Logs: `/tmp/backend.log`, `/tmp/frontend.log`

### Test the typewriter intro
1. Clear localStorage: `Object.keys(localStorage).filter(k => k.startsWith('crow_onboarding_intro')).forEach(k => localStorage.removeItem(k))`
2. Also clear: `localStorage.removeItem('crow_onboarding_intro_seen')` (old frontend key)
3. Hard refresh `http://localhost:5174/onboarding` with DevTools cache disabled
4. Should see: dark background → typewriter → fade to light → chat revealed
5. Refresh again: should skip typewriter, show normal empty state

### Force onboarding mode
In `crow-frontend-worktree-1/index.html`, this is already set:
```js
window.__CROW_CONFIG__ = { forceOnboarding: true };
```

---

## Known Issues / TODOs

1. **BLOCKING: Typewriter intro not appearing** — See "What's Broken" section above. Code is in the widget but the animation doesn't show. Needs console debugging.

2. **Security: `is_verified_user` check removed** — Backend no longer requires identity verification for onboarding mode. Re-add or gate behind env var.

3. **`forceOnboarding` is a local-only testing aid** — Remove `window.__CROW_CONFIG__ = { forceOnboarding: true }` from `index.html` before pushing frontend.

4. **`index.html` local-only changes** — `apiUrl` pointing to `localhost:8001` and product ID set for dev. Revert before pushing.

5. **Old `TypewriterIntro.tsx` file** — `crow-frontend-worktree-1/src/components/onboarding/TypewriterIntro.tsx` still exists on disk but is no longer imported. Can be deleted.

6. **Post-intro layout** — After typewriter completes, the "done" phase should render the welcome message as a bot bubble in the active chat layout (MessageList + pinned input), NOT the centered empty state. This way sending the first message is seamless. This behavior is coded but untested since the typewriter never runs.
