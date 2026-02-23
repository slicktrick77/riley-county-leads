# LeadMap v4 — Architecture Reference

_For use by Claude at the start of each session. Read this + BUGS.md before touching any code._

---

## File Overview

```
riley-county-leads/
├── index.html          ← Entire Phase 1 app (single file, ~2400 lines)
├── schema.json         ← Data model + field docs + migration notes
├── generate_leads.py   ← ETL script: merges Phase 1+2 CSVs → leads.json
├── BUGS.md             ← Running bug log (read every session)
├── ARCHITECTURE.md     ← This file
└── data/               ← (gitignored) leads.json, geocode_cache.json
```

---

## index.html Structure

The app is a single HTML file. Sections in order:

```
1.  <head>              Lines ~1–40     Leaflet CSS, fonts, MarkerCluster
2.  <style>             Lines ~41–380   All CSS (design system, layout, components)
3.  #topbar             Lines ~381–420  Search, filter pills, sort, stats, action buttons
4.  #shell              Lines ~421–500  Three-panel layout wrapper
5.    #left-panel       Lines ~421–450  Lead list + bulk action bar
6.    #map-wrap         Lines ~451–465  Leaflet map + Today overlay
7.    #right-panel      Lines ~466–510  Property panel (empty state + prop-panel)
8.  Modals              Lines ~511–700  Outreach, Reminder, Snooze, Notes, Tag, Bulk, Script, Import
9.  <script>            Lines ~701–end  All JavaScript
```

---

## JavaScript Architecture

### Constants (top of script)
```js
OFFER_PCT  = 0.78    // offer = 78% of appraised value
LOAN_LTV   = 0.80    // DSCR loan LTV
RENT_RATIO = 0.006   // monthly rent estimate
DB_NAME    = 'leadmap4'
CLUSTER_THRESHOLD = 300   // auto-cluster above this many properties
```

### State Variables
```js
db              // IndexedDB instance (null if unavailable)
properties[]    // master array — all property objects
selectedId      // currently open property id (string)
activeFilter    // 'all' | 'hot' | 'warm' | 'new' | 'contacted' | 'under_contract' | 'closed' | 'dead' | 'today'
sortMode        // 'score' | 'value' | 'last_outreach' | 'next_reminder' | 'status'
selectedIds     // Set of ids for bulk operations
activeTab       // 'overview' | 'notes' | 'outreach' | 'reminders'
undoStack[]     // array of {label, snapshot, ids} — up to 20 entries
```

### Key Functions by Feature

**Persistence**
| Function | What it does |
|---|---|
| `openDB()` | Opens IndexedDB, creates stores on upgrade |
| `idbGetAll()` | Reads all properties from IDB |
| `idbPutAll(arr)` | Writes array of properties to IDB |
| `persistAll()` | Saves entire properties[] to IDB + localStorage fallback |
| `persistOne(p)` | Saves single property (faster for single-field updates) |

**Rendering**
| Function | What it does |
|---|---|
| `refreshAll()` | Calls renderMarkers + renderList + renderPanel + stats |
| `renderMarkers()` | Rebuilds all Leaflet markers (or cluster group if >300) |
| `renderList()` | Rebuilds left panel lead rows |
| `renderPanel()` | Renders right panel for selectedId |
| `renderTabContent(p)` | Renders active tab inside panel |
| `tabOverview(p)` | Returns HTML string for Overview tab |
| `tabNotes(p)` | Returns HTML string for Notes tab |
| `tabOutreach(p)` | Returns HTML string for Outreach tab |
| `tabReminders(p)` | Returns HTML string for Reminders tab |

**Data Mutations**
| Function | What it does |
|---|---|
| `setProp(id, key, val)` | Sets a single field on a property, persists, re-renders |
| `makeFreshProp(raw)` | Normalizes any raw object into a valid v4 property |
| `migrateActivity(arr)` | Converts v3 activity[] to v4 outreach[] format |

**Filtering / Sorting**
| Function | What it does |
|---|---|
| `getFiltered()` | Returns properties matching activeFilter + search query |
| `getSorted(list)` | Sorts a filtered list by sortMode |

**Bulk Actions**
| Function | What it does |
|---|---|
| `pushUndo(label)` | Snapshots selected properties onto undoStack |
| `applyBulkAndUndo(label, fn)` | Runs fn on each selected prop, calls pushUndo, shows undo toast |
| `performUndo()` | Restores last snapshot from undoStack |

**Reminders**
| Function | What it does |
|---|---|
| `activeReminders(p)` | Returns non-done reminders for a property |
| `nextReminderTs(p)` | Returns earliest upcoming reminder timestamp |
| `isDueToday(r)` | True if reminder effective date is today |
| `isOverdue(r)` | True if reminder effective date is before today |
| `reminderStatusForProp(p)` | Returns 'overdue' | 'today' | 'upcoming' | null |
| `countTodayReminders()` | Total overdue+today reminders across all properties |

**Import / Export**
| Function | What it does |
|---|---|
| `importProperties(raw[])` | Dedupes by id then address, merges CRM fields, persists |
| `exportJSON()` | Downloads full properties[] as .json |
| `exportCSV()` | Downloads snapshot as .csv |
| `exportICS()` | Downloads active reminders as .ics calendar file |

---

## Data Model (Property Object)

```js
{
  id:              string,      // unique — string preferred
  address:         string,      // "1412 Fairchild Ave"
  city:            string,      // "Manhattan, KS"
  owner:           string,      // "HENDERSON, ROBERT L"
  mailing:         string,      // out-of-state address
  value:           number,      // appraised value USD
  lat:             number,
  lng:             number,
  status:          string,      // hot|warm|new|contacted|under_contract|closed|dead
  tags:            string[],
  notes:           string,      // quick notes (autosaved)
  structuredNotes: [{id, ts, type, body}],
  outreach:        [{id, ts, channel, outcome, summary, template}],
  reminders:       [{id, dueTs, label, note, done, snoozedTo, channel}],
  contacted:       boolean,
  offerMade:       boolean,
  offerAmount:     string,
  activeRental:    boolean,
  createdTs:       number,      // unix ms
  updatedTs:       number,      // unix ms
}
```

---

## Scoring System

Score = sum of active signals (max 100):
- Military APO/FPO address: **+40**
- Virginia address (non-military): **+25**
- Any other out-of-state: **+15**
- Value $180k–$320k: **+15**
- Has been contacted: **+5**
- Active rental listing: **+5**

Tiers: 55+ = Highest Priority · 40+ = High · 25+ = Moderate · <25 = Low

---

## Status Colors

| Status | Color |
|---|---|
| hot | #e05252 (red) |
| warm | #e8813a (orange) |
| new | #8594b0 (gray-blue) |
| contacted | #4a9eff (blue) |
| under_contract | #8b5cf6 (purple) |
| closed | #3db87a (green) |
| dead | #4a5870 (dark gray) |

---

## XSS Safety Rule

**All** user-provided or imported text must pass through `esc()` before being
placed in `innerHTML`. The `esc()` function encodes `& < > " '`.
For multi-line content use `escNL()` which also replaces `\n` with `<br>`.
Use `textContent` for plain strings that don't need HTML.

---

## How to Start a Bug-Fix Session

1. Read BUGS.md — pick a bug (e.g. B004)
2. Look up the function in the table above
3. Ask Jefferson to paste just the relevant function from index.html
4. Fix it, return the corrected function only
5. Jefferson pastes it back into index.html using Find & Replace
6. Mark bug resolved in BUGS.md, commit

---

## Session Log

| Date | What changed |
|---|---|
| 2026-02-22 | Phase 1 complete — initial build |

