# Daily Briefing Integration Behavior

This document defines source-of-truth integrations and rendering behavior for the Daily Briefing so implementation decisions remain consistent across frontend and backend work.

## 1) Data Blocks, Source Systems, and Update Intervals

| Data block | Primary source system | Update interval | Notes |
| --- | --- | --- | --- |
| Weather | `services/briefing-service` weather provider adapter (external weather API behind adapter) | Refresh every **15 minutes**; allow manual refresh | Store normalized forecast snapshot in briefing cache for rendering consistency. |
| News | `services/briefing-service` news adapter (configured news feed/API) | Refresh every **30 minutes** | Normalize to headline + source + timestamp + URL fields before UI use. |
| Chores | `services/chores-service` | Refresh every **5 minutes** via polling or event-driven cache invalidation | Chore completion updates should invalidate briefing cache immediately when possible. |
| Calendar | Local SmartHome calendar store by default (`frontend/calendar` + backing calendar domain in platform) | Refresh every **5 minutes**; immediate refresh after local calendar edits | External calendar sync is optional and disabled unless explicitly configured (see scope section). |
| Date/Time | Device-local clock in client runtime | Recompute every **minute** (time) and at local midnight (date) | Date/time should not block briefing rendering; always available from local device. |

## 2) Failure Behavior by Source

### Shared behavior
- Each block renders independently.
- A failed block must not prevent other blocks from rendering.
- Every block reports one of: `fresh`, `stale`, `error`, `empty`.

### Weather
- If refresh fails, show last successful snapshot up to **2 hours** old (`stale` state).
- If no cached snapshot exists, show an inline error tile with retry action (`error` state).
- Show a non-blocking warning banner only when stale data exceeds 60 minutes.

### News
- If refresh fails, keep last successful headlines up to **6 hours** old (`stale`).
- Drop malformed items and render remaining valid items (`partial rendering`).
- If all items fail validation and no cache exists, render empty/error state with message: "News temporarily unavailable.".

### Chores
- If chores service is unavailable, show last cached chores up to **24 hours** old and mark as stale.
- Completion toggles attempted while stale/unavailable should queue locally and retry when connectivity returns.
- If queueing is unavailable, disable toggle actions and show inline action-level error.

### Calendar
- If local calendar query fails, show stale cache up to **24 hours**.
- For optional external sync failures, keep local events visible and show sync-specific warning in calendar block only.
- Rendering should continue with any successfully fetched event subset (`partial rendering`).

### Date/Time
- Always render using local device time.
- If timezone metadata cannot be resolved, default to device locale timezone without error banner.

## 3) Calendar Scope and Supported Views

### Scope
- **Default behavior**: calendar block uses **local-only SmartHome events**.
- **External sync (optional)**: external providers (e.g., Google/Outlook) are out-of-scope by default and only enabled behind explicit configuration/feature flag.
- If external sync is enabled, merged output must preserve source attribution per event for debugging and conflict handling.

### Views (`day/week/month`)
- **Day view**: primary briefing view; shows today's events in chronological order with current-time marker.
- **Week view**: seven-day rolling window beginning at local week start; summarizes counts per day and includes today's expanded events.
- **Month view**: high-level density view (event indicators/counts per day), with drill-in to day details; no heavy detail rows by default.
- All views must apply local timezone and support consistent all-day event handling.

## 4) Top Priority Items Ordering Rules

Top priority items are selected from chores, calendar events, and system alerts (if present).

1. **Urgency first**
   - Overdue chores/events and time-critical alerts rank highest.
2. **Time proximity second**
   - Items with the nearest due/start time rank above later items.
3. **Severity/importance third**
   - Explicit high-priority flags outrank normal items when urgency/proximity are equivalent.
4. **User-defined pinning fourth**
   - Pinned items appear above non-pinned items within the same urgency band.
5. **Deterministic tie-breakers**
   - Sort by `(source priority, due time, created time, stable id)` for stable rendering.

Additional constraints:
- Cap top priority list at **5 items** on mobile and **8 items** on always-on displays.
- Ensure at least one item from each available critical source (calendar/chores/alerts) when possible before filling remaining slots.

## 5) Acceptance Criteria: Mobile vs Always-On Display

### Mobile layout acceptance criteria
- Above the fold must include:
  - current date/time,
  - current weather summary,
  - top 3 priority items,
  - next calendar event (or explicit "No upcoming events" state).
- News may appear below the fold.
- Layout must remain usable at 360px width without horizontal scrolling.
- Error or stale states must be visible inline within each block and not hidden behind interactions.

### Always-on display acceptance criteria
- Above the fold must include:
  - current date/time,
  - weather summary,
  - top priority list (up to 8 items),
  - day calendar strip (next events),
  - at least 3 news headlines.
- Must support glanceable viewing at distance: key metrics visible without scrolling.
- Auto-refresh indicators (e.g., "updated X min ago") must be visible for weather/news/calendar.
- Partial failures must keep layout stable (no collapsing to blank screen).

## 6) Definition of Done for Integration Consistency

- Source adapters expose normalized contracts consumed by briefing renderer.
- Cache policy and stale thresholds are implemented exactly as defined above.
- Per-block independent failure handling is covered by automated tests.
- Priority ordering is deterministic and documented in code comments/tests.
- Layout acceptance criteria are validated on mobile and always-on target display sizes.
