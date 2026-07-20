# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file, static travel itinerary web app (Korean-language) for a Hokkaido trekking trip (Aug 23–29, 2026). There is no build system, no package manager, and no test suite — everything lives in `index.html` (HTML + CSS + vanilla JS in one file) plus `manifest.json` (PWA manifest with inline SVG data-URI icons).

## 규칙

- 단일 `index.html` 구조를 유지한다 — 별도 JS/CSS 파일이나 빌드 단계로 분리하지 않는다.
- `index.html` 수정 시 서비스 워커 캐시 버전을 반드시 증가시킨다.
- 커밋 메시지는 한국어로 작성한다.
- 대화는 한국어로 한다.

## Running / testing changes

There is no dev server or build step. Open `index.html` directly in a browser, or serve the directory with any static file server (e.g. `python -m http.server`) to test PWA/manifest behavior. Verify changes by loading the page and clicking through day tabs, opening modals, and toggling edit mode (see Architecture below) — there are no automated tests.

## Architecture

Everything is in `index.html`, in three parts: `<style>` block, static HTML shell (hero, tabs, panels, modals), and a single `<script>` block containing all app state and logic.

### Data model
- `DAYS`: array of 7 day objects (`n`, `date`, `dow`, `wx`, `head`, `sub`, `route`, `items[]`). Each `items[]` entry is a schedule card with `t` (time), `k` (kind: `trek`/`move`/`stay`/`air` — drives color coding via CSS classes), `kind` (label), `ttl` (title), `desc`, optional `dur`, `pin` (Google Maps search query or `"lat,lng"`), `gmap` (trek route override), `box` (key-value info box, e.g. reservation details), `chips`, `memo`, and `alts[]` (alternative place suggestions, each with optional `rating`/`rcount`).
- `BASE_DAYS`: a deep-cloned snapshot of the original hardcoded `DAYS`, captured once at load, used only to seed Firebase on first run and to power `restoreLostItem()` (a one-time self-healing patch for a specific previously-deleted item — see below).
- `opts`, `personal`, `costs`: user-editable state for the options/notes section, 3 fixed per-person packing lists, and shared expense tracking.

### Persistence — three layers, in priority order
1. **Firebase Realtime Database** (`firebaseConfig` near the top of the script) is the source of truth when reachable. Paths: `hokkaido2026/{days,opts,personal,cost}`. Each ref has a live `.on("value", ...)` subscriber, so any client's edit propagates to all open tabs/devices in real time.
2. **localStorage** (`lsSave`/`lsLoad`, key prefix `hokkaido2026_`) mirrors every write as an offline/no-Firebase fallback and is restored on load before Firebase responds.
3. **URL hash** (`#view=<base64>` / `#edit=<base64>`) is a last-resort carrier for `opts` only, used solely when Firebase isn't configured (`fbReady === false`).

When editing persistence logic, preserve this fallback order — the app must degrade gracefully with Firebase fully absent.

### Edit mode / sharing
`editMode` is derived from the URL hash: any hash not starting with `view` is editable (including no hash at all — bookmarked/installed PWA defaults to editable). The share sheet (`openShare`) generates two links: `#view` (read-only) and the bare URL (editable). In read-only mode, `applyMode()` hides the "add" buttons and intercepts clicks on `.mini` (edit/delete) buttons to show a toast instead.

### Rendering
There's no virtual DOM or diffing — `buildTabs()`, `buildPanels()`, `itemHtml()`, `renderOpts()`, `renderCost()`, `personalHtml()` all re-render via full `innerHTML` replacement from current state, called after every mutation. `showDay(i)` just toggles `.active` classes; it doesn't rebuild. Any new piece of persisted state needs: a render function, a save path through `persist()`/`persistDays()`/`lsSave()`, and (if it should sync live) a Firebase ref + `.on("value", ...)` subscriber wired up in `initFirebase()`.

### Notable gotchas
- Firebase rejects `undefined` — always route objects through `cleanForFirebase()` (JSON round-trip that maps `undefined` → `null`) before writing.
- `normalizeDays()` exists because Firebase returns arrays-with-gaps as objects; any code reading `days` back from Firebase must go through it.
- `restoreLostItem()` in `initFirebase()`'s `daysRef` subscriber is a one-off migration that re-inserts a specific item (`pin === "イオン千歳店"`) if a prior bug deleted it from someone's live database. Don't extend this pattern for new fixes — it's a dead-end workaround, not a general mechanism.
- Day/date logic (`todayDayIndex()`, `tripProgress()`) hardcodes the trip window (2026-08-23 to 2026-08-29) directly in the functions — update both if the trip dates ever change.
