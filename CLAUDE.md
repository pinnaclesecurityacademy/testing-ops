# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a mobile-first Progressive Web App (PWA) for security operations at **Tower 3 ITS**. It has no build system, no framework, and no package manager — everything is plain HTML, CSS, and vanilla JavaScript in two self-contained files.

## Development

Open either file directly in a browser or serve locally:

```
# Python
python -m http.server 8080

# Node
npx serve .
```

There are no build, lint, or test commands.

## File Structure

- **`index.html`** — The field officer app (mobile PWA). All CSS, HTML templates, and JavaScript are in one file.
- **`dashboard.html`** — The controller dashboard (desktop). Also self-contained. Read-only view polling Google Sheets every 30 seconds.
- **`manifest.json`** — PWA manifest for home screen install.
- **`sw.js`** — Service worker providing offline caching for `index.html`.

## Architecture

### Single-page navigation (index.html)

Pages are `<div id="page-*">` elements that are shown/hidden by toggling the `.active` class. Navigation is handled by `navigateTo(page)`. The bottom nav highlights via `BNAV_MAP`, which maps sub-pages back to their parent nav tab.

### Data model

All task/audit definitions are static JavaScript constants at the top of `index.html`:

- `DAY_SHIFT_TASKS` — ordered list of 11 day shift tasks. Each has a `type`: `'flexible'`, `'plant'`, or `'lobby'`.
- `PLANT_LEVELS` — 5 plant room levels (L15, L27, L39, L40, L41), each with rooms requiring mandatory photos.
- `AUDIT_TYPES` — audit definitions with per-asset question sets (lifts, bathrooms, etc.).
- `SITE` — site name and audit target count.

### State and persistence

`STATE` is a single global object holding all session state. It is persisted to `localStorage` under keys prefixed with `t_`. On load, `loadState()` checks if `t_date` matches today and clears all keys if not (daily reset). Photos are stored in `STATE` as base64 data URLs but are **not** persisted to localStorage.

### Google Sheets integration

`SHEETS_URL` (a Google Apps Script web app URL) is the backend. Communication is via **JSONP** (dynamically injected `<script>` tags) to avoid CORS. Two operations:
- `syncToSheets(payload)` — posts patrol/audit/escalation/activity records using a hidden `<form>` POST.
- `syncActivityToSheets(text, type)` — posts a lightweight activity row.
- Dashboard (`dashboard.html`) reads via `?action=getDashboard&callback=handleDashboardData`.
- PDF download via `?action=getPDF&filename=...&callback=handlePDFData`.

### PDF generation

`jsPDF` (loaded from CDN) is used client-side. Each task type has its own PDF generator function (`generateTaskPDF`, `generateAuditPDF`, `generateLobbyPDF`, `generateEODPDF`). Photos captured during patrol are embedded as base64 images in the PDF. After generation, the PDF is also uploaded to Google Sheets via a base64-encoded POST for controller download.

### Task flow by type

- **`flexible`** — room-by-room checklist: `startRoverTask` → `renderFlexibleRooms` → `startRoom` → `confirmRoom` → `completeFlexibleTask`.
- **`plant`** — two-level drill-down (level → room): `startPlantPatrol` → `renderPlantLevels` → `startPlantLevel` → `startPlantRoom` → `confirmPlantRoom` → `completePlantPatrol`.
- **`lobby`** — timed static observation: `startLobby` → `lobbyStart` → `lobbyFinish`. Lobby in-progress state is separately persisted via `saveLobbyState()`.
- **`audit`** — asset selection then sequential yes/no questions: `startAudit` → asset picker → `advanceAuditQuestion` → `completeAudit`.

### Site configuration

To adapt this app for a different site, change the `SITE` constant and update `DAY_SHIFT_TASKS`, `PLANT_LEVELS`, and `AUDIT_TYPES`. The `#nav-site-name` element and `dashboard.html` `#site-name` also need updating.
