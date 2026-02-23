# LeadMap v4 — Bug Tracker

_Last updated: 2026-02-22 · Phase 1_

---

## 🔴 Known Bugs (Not Yet Fixed)

### B001 — Bulk select-all checkbox doesn't visually reset
**Where:** Left panel, top checkbox labeled "All"
**What happens:** After clearing a bulk selection via "✕ Clear", the checkbox stays checked visually even though the selection is cleared.
**Expected:** Checkbox should uncheck when selection is cleared.

### B002 — Today filter button doesn't deactivate map filter
**Where:** Topbar filter pills
**What happens:** Clicking "📅 Today" opens the Today overlay correctly, but also applies a filter to the map/list underneath. Clicking another filter pill (e.g. "All") doesn't cleanly reset the Today state.
**Expected:** Today overlay and list/map filters should be independent.

### B003 — Tag modal can be opened on wrong property
**Where:** Property panel header → "+ tag" button
**What happens:** If you click "+ tag", close without saving, then select a different property and open it again — the tag sometimes saves to the previously targeted property.
**Root cause:** `tagTargetId` state can be stale if panel re-renders between modal open and save.

### B004 — Offer amount input triggers full panel re-render on every keystroke
**Where:** Overview tab → Offer Tracking section
**What happens:** `setProp()` is called `oninput`, which calls `renderPanel()`, which destroys and rebuilds the DOM — causing the input to lose focus after every character typed.
**Expected:** Offer amount should debounce or only save on blur.

### B005 — ICS export uses UTC timestamps, ignores local timezone
**Where:** Export ICS button
**What happens:** Reminder due dates export in UTC, so events appear at wrong times in calendar apps for non-UTC users.
**Expected:** Should use `DTSTART;TZID=America/Chicago:` format for Manhattan KS (Central Time).

### B006 — Import modal doesn't close automatically after success
**Where:** After importing a JSON or CSV file
**What happens:** The import results modal stays open indefinitely — user must manually click Done.
**Expected:** Auto-close after 3 seconds, or at minimum show a clear success state.

### B007 — Structured note delete button has no confirmation
**Where:** Notes tab → structured notes → × button
**What happens:** Clicking × immediately deletes the note with no undo.
**Expected:** Either a confirm dialog or an undo toast (like bulk actions have).

### B008 — Map markers don't update pin style when status changes
**Where:** Overview tab → Pipeline Status buttons
**What happens:** Clicking a status button updates the list row color and badge correctly, but the map marker pin color doesn't change until the next full `renderMarkers()` call (e.g. filter change or page reload).
**Root cause:** `setProp()` calls `renderMarkers()` only when `key === 'status'` — this appears correct but the cluster group may not be getting rebuilt.

### B009 — Snooze modal left open if property is deselected
**Where:** Reminders tab → 😴 button → then clicking "← Back"
**What happens:** If user opens snooze modal then navigates away, the modal stays open and `snoozeTargetId` points to a deselected property.
**Expected:** Modals should close when panel is deselected.

### B010 — No validation on reminder due date in the past
**Where:** Create Reminder modal
**What happens:** User can create a reminder with a due date in the past, which immediately appears as "Overdue" with no warning.
**Expected:** Warn the user if selected date is in the past (soft warning, not a block).

---

## 🟡 UX Issues / Polish

### U001 — No empty state when filter returns zero results
The list panel just goes blank with a count of "0 of N props" — should show a friendly message.

### U002 — Search doesn't highlight matching text in list rows

### U003 — No way to delete a property (only Reset All exists)

### U004 — Tab keyboard navigation inside modals not trapped (focus escapes to background)

### U005 — Reminder count badge on "Today" button doesn't update after completing a reminder from the Today overlay without closing it

### U006 — On narrow screens (<1100px) the three-panel layout overflows instead of collapsing

---

## ✅ Confirmed Working

- IndexedDB persistence + localStorage fallback
- Import JSON (dedupe by id and address, CRM merge)
- Import CSV
- Export JSON, CSV, ICS
- Outreach logging with auto follow-up reminder
- Bulk status / tag / reminder with Undo
- Score calculation and signal display
- Call script generation (Military / Virginia / Other)
- Map clustering above 300 properties
- Today view with overdue / due today / upcoming sections
- Snooze (1d / 3d / 7d / 14d / 30d)
- Reminder done / reopen toggle
- Structured notes (typed, timestamped)
- Quick notes autosave
- Tag add / remove per property
- Status pipeline buttons
- XSS escaping on all user/imported content
- Sample data loads on first run (10 properties, 2 pre-seeded reminders)

---

## 📋 Phase 2 Feature Backlog

- [ ] Property delete (single)
- [ ] Direct mail export (owner name + mailing address, formatted for printing)
- [ ] Adjustable offer % slider per property
- [ ] Parcel count per owner (from merged Phase 1 + Phase 2 dataset)
- [ ] Bulk export filtered view only
- [ ] Email template builder (merge tags: owner name, address, offer)
- [ ] Map heatmap layer by score
- [ ] Import progress bar for large files
- [ ] Mobile/responsive layout
- [ ] Keyboard shortcut reference (? key)
