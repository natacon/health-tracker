# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Personal health/diet PWA for daily logging (weight, meals, exercise, mood, weather, checks). Single-user, no backend, no accounts. UI text is Japanese; code identifiers and comments are English.

## Who this is for

The single user is the developer's wife. Treat the following as the design context for any UX decision:

- **Heavy SNS user.** She lives in Instagram/Threads/X. Visual outputs (the Preview tab, screenshots, share cards) and chip-style one-tap interactions match her habits; long forms and multi-step flows do not.
- **Was logging diet via ChatGPT.** This app is replacing that workflow. The bar is conversational speed: if logging takes more taps than typing a sentence to a chatbot, she'll go back. Favor: favorites/`★いつもの`, paste-friendly fields, autofill (weather), Web Share Target.
- **YouTube exercise videos.** She watches workout videos and screenshots her YouTube watch history; exercise entries are populated by OCR on the attached screenshot (Tesseract.js) — there is no URL field anymore.
- **Mobile-first, Tokyo, JP language.** Layout assumes a single-column phone viewport. Auto-features (weather, time, dates) hardcode Tokyo / `Asia/Tokyo` / Japanese day names — generalizing them is a non-goal.
- **Solo, on her own device.** No multi-user, no auth, no sync server. Cross-device backup is the export/import folder, not a backend.

When proposing features, prefer the SNS-pattern equivalent (streaks, share cards, badges, one-tap) over the productivity-app equivalent (forms, settings, configuration).

## Stack and conventions

- Vanilla HTML/CSS/JS in a **single `index.html`** — all CSS in one `<style>`, all JS in one `<script>`. There is no build step, no bundler, no package manager, no test framework.
- PWA: `manifest.json` + `sw.js` (cache-first service worker). Registered as a Web Share Target — shared text/title lands in the exercise form's first row.
- Deployed to GitHub Pages via `.github/workflows/pages.yml` on push to `main`. The repo's Pages source must be set to "GitHub Actions" in repo settings (one-time, manual).

## Running locally

```sh
python3 -m http.server 8000
```

Then open `http://localhost:8000`. Service worker only registers over HTTPS or localhost, so file:// won't fully work.

## Architecture

### Data model: timeline of events

Each day is `{ events: [...] }` where every event has `{ id, time, type, ...payload }`. Types are `weight | meal | exercise | mood | weather | check`. Older days may have been stored in a denormalized shape (single weight/meals string/etc.); `migrateEntry()` converts those to the events array on read. Always write the events shape — never reintroduce the old fields.

The `aggregate()` function reduces an events array to per-day summaries (latest weight/mood/weather wins, checks are presence-based, meal/exercise totals sum kcal). This is what powers both the Preview tab and history rows.

### Storage: OPFS first, localStorage fallback

`storage` (in `index.html`) auto-selects mode at init:
- **OPFS** (`navigator.storage.getDirectory()`) — primary. One file per day: `YYYY-MM-DD.json`. No permission dialog; uses `navigator.storage.persist()`.
- **localStorage** — fallback under `health-tracker-v1` as `{ [date]: entry }`. Settings always live in `health-tracker-settings-v1`.

On first OPFS init, legacy localStorage entries are one-time migrated into OPFS files. Never delete the localStorage migration without ensuring all users have already migrated.

Settings (`goalWeight`, `favorites`) and the "auto weather fetched today" flag (`health-tracker-auto-weather-date`) always use localStorage regardless of mode.

Export/Import via `showDirectoryPicker()` writes/reads the same per-day JSON shape — useful for cross-device backup.

### Add form / favorites

`buildAddForm(type, prefill)` returns the form HTML for one event type. `openAddForm()` mounts it and binds events. `editingId` distinguishes add-new (null) vs edit-existing — when editing, the favorites row and "★ いつもの" button are hidden.

Favorites for `meal` and `exercise` live in `settings.favorites[type]`. Tapping a favorite chip in the form bypasses the form entirely and directly pushes an event using `nowTimeForDate()`.

### Auto features that hit external APIs

- **Tokyo weather**: Open-Meteo (`api.open-meteo.com`), no key. Hourly endpoint, picking the 12:00 (noon) slot of the displayed date. Auto-fetched once per day (gated by `AUTO_WEATHER_FLAG_KEY`) when viewing today and no weather event exists. Stored with `time: '12:00'`.
- **Kcal lookup**: `🔍 推定` button in meal/exercise add forms tries to auto-fill kcal:
  - Meal: Open Food Facts search → ja.wikipedia search/extract — first plausible `<n>kcal` wins.
  - Exercise: hardcoded MET dictionary keyed on workout name + `<n>分` duration in the value field, multiplied by latest known body weight (default 50kg).
  - On miss/error, prompts the user and opens Google search in a new tab as fallback. Direct Google scraping isn't possible from JS (CORS).
- **Meal text autocomplete**: 内容 field is a `<datalist>`-backed combobox showing every `★いつもの` meal text. Picking a known entry also auto-fills 区分 + kcal.
- **Exercise OCR**: Tesseract.js (v5, lazy-loaded from jsDelivr) runs Japanese OCR on the first picked image. `parseYouTubeOcr` splits the text by `MM:SS` / `H:MM:SS` duration markers, then `formExerciseRows` replaces empty rows or appends new ones, keeping the form in its always-on multi-row mode.
- **Shared image refcount**: Because batch registration attaches one image to multiple events, `imageReferenceCount` scans `entry.events` + `cache` before `deleteEvent` / `clearDayBtn` purge files from OPFS — only truly orphaned files get removed.

## Things that bite

- **Bump `CACHE` in `sw.js`** on every user-facing change, otherwise the old cached `index.html` keeps serving. Currently `health-tracker-v20`.
- **OPFS is per-origin and per-browser**: data does not sync across devices. Use the export/import buttons.
- The SW's fetch handler falls back to `./index.html` on navigation when offline — keep that intact for the PWA experience.
- The `start_url` and `scope` in `manifest.json` are `.` so the app works at any subpath (e.g. `/health-tracker/` on Pages).

## Workflow

- **Push via PR, then squash-merge immediately.** Direct push to `main` is blocked by the local proxy (`HTTP 403`). The working flow is: commit locally → `git push --force-with-lease origin HEAD:claude/diet-app-ux-improvements-z8TGu` → `mcp__github__create_pull_request` (base `main`, head feature branch, `draft: false`) → `mcp__github__merge_pull_request` (`merge_method: squash`) → `git fetch && git reset --hard origin/main` locally. Don't leave PRs open for review — squash and merge in the same turn.
