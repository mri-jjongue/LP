# AGENTS.md — LP Designer Pro

This file provides guidance for AI coding agents working in this repository.

---

## Repository Overview

**LP Designer Pro** is a self-contained, single-page web application for creating and managing trainer-led onboarding learning plans at MRI Software (Education Services). There is no build step, no package manager, and no server-side code — the entire application ships as static HTML files with inline CSS and JavaScript.

---

## Repository Structure

```
LP/
├── index.html              # Landing/splash page — entry point for users
├── v1.12.html              # Main application (~7,700 lines of HTML/CSS/JS)
├── assets/                 # Static image assets (PNG logos, watermark)
│   ├── mri_lp_logo.png
│   ├── trend_watermark.png
│   └── ...
└── template/
    └── apac/               # Pre-built JSON workspace templates for the APAC region
        ├── PMX_Support.json
        ├── PMX_Consultant.json
        ├── PT_Support.json
        └── ... (9 role-based templates)
```

### Key Files

| File | Purpose |
|---|---|
| `index.html` | Splash/marketing page. Animated logo, feature overview cards, "Launch Workspace" button that links to `v1.12.html`. |
| `v1.12.html` | The complete application. All UI, logic, and styling is inline in this single file. |
| `template/apac/*.json` | Saved workspace state files for APAC region job roles. Users can load these to pre-populate a session. |

---

## Technology Stack

No framework, no bundler, no dependencies to install. All third-party libraries are loaded from CDN at runtime.

| Library | Version | CDN | Purpose |
|---|---|---|---|
| xlsx-js-style | 1.2.0 | jsdelivr | Export styled `.xlsx` Excel files |
| Chart.js | latest | jsdelivr | Analytics/summary charts |
| Driver.js | 1.3.1 | jsdelivr | In-app guided product tour |
| Google Fonts | — | fonts.googleapis.com | Inter, Montserrat, Sora typefaces |

---

## Architecture of `v1.12.html`

The application is structured as a single HTML file. All CSS is in a `<style>` block in `<head>`. All JavaScript is in a `<script>` block at the end of `<body>`. There is no module system.

### Core Data Model

```js
let sessions = [];          // Array of session objects (up to 10 tabs)
let activeSessionId = null; // UUID of the currently active session
let workspaceDataset = [];  // The rows of the active session's learning plan table
```

Each **session object** holds:
- `sessionId` — UUID
- `templateNum` — display number for the tab
- `config` — metadata fields (trainee name, role, start date, region, etc.)
- `dataset` — array of row objects (the learning plan milestones)
- `selectedIds` — `Set` of selected row IDs
- `undoStack` / `redoStack` — history for undo/redo

### Major Function Groups

| Group | Key Functions |
|---|---|
| **State / Persistence** | `saveStateToLocalStorage`, `saveCurrentUIIntoSession`, `loadSessionIntoUI` |
| **Undo/Redo** | `recordStateSnapshot`, `executeUndo`, `executeRedo`, `restoreSnapshot` |
| **Tab management** | `addNewTab`, `switchTab`, `deleteTab`, `renderTabs`, `setupTabDragAndDrop` |
| **Date scheduling engine** | `parsePastedDate`, `handleDatePaste`, `updateDueDateFromCalc`, `runGlobalRecalculationEngine` |
| **Calendar widget** | `initCalendar`, `renderCalendar`, `createMonthDOM`, `dockCalendar`, `undockCalendar` |
| **Table rendering** | `renderWorkspaceView`, `setupTableDelegation`, `setupSortingHeaders`, `sortTableByFieldColumn` |
| **EMS importer** | `parseEMSDate` (parses Excel/CSV from MRI's EMS system) |
| **Excel export** | `generateAndDownloadBinaryXLSX`, `constructExcelSheetDataMatrix` |
| **CRT form** | `launchPrepopulatedCRTForm` (builds a Smartsheet URL with query params) |
| **Email modal** | Generates pre-populated email templates for manager/new-hire communication |
| **Summary/Analytics** | `updateSummaryBlock`, `renderChart`, `calculateBusinessDays` |
| **Bulk actions** | `updateBulkActionBar`, `executeBulkAction` |
| **Theme** | `initThemeToggle` (light/dark via CSS custom properties + `localStorage`) |
| **Utility** | `escapeHtml`, `sanitizeForTSV`, `generateId`, `createEmptyRowModel` |

### Configuration Fields (Control Panel)

The left-side control panel captures metadata that drives scheduling:

| DOM ID | Field |
|---|---|
| `metaPrideName` | Trainee (Pride Member) name |
| `metaRoleTitle` | Role title |
| `metaStartDate` | Training start date |
| `paramB4` | Orientation type |
| `paramB7` | Region (drives date locale: DD/MM vs MM/DD) |
| `metaProduct` | Product being trained on |
| `metaLPOwner` | Learning Plan owner/trainer name |
| `metaLPEmail` | LP owner email |

### Date Parsing Engine

The `parsePastedDate` function supports three input formats:
1. **ISO** — `YYYY-MM-DD` (highest priority, locale-agnostic)
2. **Verbal** — `"15 Jul 2026"` or `"July 15, 2026"`
3. **Numeric regional** — `DD/MM/YYYY` (APAC regions) or `MM/DD/YYYY` (others), determined by `paramB7`

### Template JSON Format

Templates in `template/apac/` are serialized session objects (the same shape as what `localStorage` stores). The top-level structure is an array with one session object:

```json
[
  {
    "sessionId": "<uuid>",
    "templateNum": 7,
    "config": { "prideName": "", "roleTitle": "...", ... },
    "dataset": [ { "id": "...", "courseName": "...", "dueDate": "...", ... } ],
    "selectedIdsArray": []
  }
]
```

---

## Development Guidelines

### Making Changes

- **No build step required.** Edit HTML files directly and open in a browser to test.
- All CSS is inline in `<style>` tags; all JS is inline in `<script>` tags. Do not extract them to separate files unless the task specifically requires it.
- The versioned filename convention is `v{major}.{minor}.html` (e.g., `v1.12.html`). When making significant changes, the version number in the `<title>` tag and any internal references should be updated accordingly.
- `index.html` contains a hard-coded `href="v1.12.html"` link to the app. Update this if the app filename changes.

### Dark Mode

CSS custom properties are defined in `:root` for light mode and overridden in `[data-theme="dark"]`. Always define both when adding new color variables.

### Avoiding Regressions

- The undo/redo engine (`recordStateSnapshot`) is called on every user data mutation. Any new data-mutating function must call `recordStateSnapshot()` before or after the mutation, consistent with the surrounding pattern.
- `runGlobalRecalculationEngine()` must be called after any change that modifies `workspaceDataset` to keep the table and summary in sync.
- `escapeHtml()` must be used whenever user-provided strings are interpolated into HTML templates.
- Do not break the `saveStateToLocalStorage` / `loadSessionIntoUI` round-trip; the schema of session objects must stay backward-compatible (new optional fields only, with safe defaults).

### Template Files

- Templates are plain JSON. Validate any edited template is valid JSON before committing.
- The `dataset` array items must include at minimum: `id`, `courseName`, `courseType`, `dueDate`.
- Do not hardcode trainee-specific data (names, emails) in template files — `prideName`, `metaLPOwner`, and `metaLPEmail` should be empty strings in templates.

---

## No Linting, Building, or Testing

There is no test suite, linter, or build system in this repository. Validation is done by opening the HTML files in a browser. When verifying changes:

1. Open `index.html` in a browser and confirm the splash page renders correctly.
2. Click "Launch Workspace" and confirm the transition to `v1.12.html` works.
3. Exercise the specific feature changed (date entry, import, export, theme toggle, etc.).
4. Check browser DevTools console for JavaScript errors.
