# SmartHome API Contracts (v1)

This document defines **implementation-ready API contracts** for SmartHome features.
The contracts are versioned under `/api/v1/...` and machine-readable schemas are in `docs/api/schemas/v1/`.

## Global conventions (apply to all features)

### Data formats and date/time rules
- **Timezone source**: the client must send `X-Timezone` as an IANA timezone ID (example: `America/New_York`) for user-scoped calculations. If omitted, backend defaults to `UTC`.
- **Timestamp format**: RFC 3339 / ISO-8601 UTC string (`YYYY-MM-DDTHH:mm:ss.sssZ`) for all `...At` fields.
- **Date format**: calendar dates use `YYYY-MM-DD`.
- **Week boundary rule (HM)**: weeks are computed from local timezone using **Monday 00:00:00 through Sunday 23:59:59.999**.
- **Server clock authority**: write acknowledgements include server-generated timestamps to avoid client clock skew.

### Idempotency
- `POST` actions triggered by UI buttons that mutate state MUST support `Idempotency-Key`.
- Header format: opaque string up to 128 chars (`^[A-Za-z0-9_-]{8,128}$`).
- Key scope: unique per `(feature, householdId, actorId, operation)` for 24 hours.
- First successful request stores the result; retries with same key return the same semantic result and do not re-apply side effects.
- Mandatory for **CCO** and **HBF**.

### Standard error shape
All non-2xx responses return:
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Human-readable summary",
    "details": [
      { "field": "date", "issue": "must match YYYY-MM-DD" }
    ],
    "requestId": "req_01H..."
  }
}
```

---

## BGL — Build Grocery List

Builds or refreshes a grocery list for a target week from menu plan entries.

- **Endpoint**: `POST /api/v1/grocery-list/build`
- **Required inputs**:
  - Body: `householdId` (string), `weekStartDate` (`YYYY-MM-DD`), optional `overwrite` (boolean, default `false`).
  - Header: `X-Timezone` (recommended).
- **Validation rules**:
  - `weekStartDate` must be a Monday in request timezone.
  - `householdId` min length 3.
  - If `overwrite=false`, existing unlocked list for that week cannot be replaced.
- **Response shape** (`200`):
  - `listId`, `householdId`, `weekStartDate`, `generatedAt`, `items[]` with `name`, `quantity`, `unit`, `sourceRecipeId`.
- **Error codes**:
  - `400 VALIDATION_ERROR`
  - `404 MENU_PLAN_NOT_FOUND`
  - `409 GROCERY_LIST_LOCKED`

## SGL — Sync Grocery List

Synchronizes client-side grocery edits with server state.

- **Endpoint**: `POST /api/v1/grocery-list/sync`
- **Required inputs**:
  - Body: `listId`, `householdId`, `baseVersion` (int >= 0), `changes[]`.
  - Each change: `op` (`add|update|remove|toggleChecked`), `itemId` (required except add), `payload` (required for add/update).
- **Validation rules**:
  - `changes` must contain at least 1 operation and at most 200.
  - Unknown `op` rejected.
  - `baseVersion` must match current list version for optimistic concurrency.
- **Response shape** (`200`):
  - `listId`, `newVersion`, `appliedChanges`, `conflicts[]`, `updatedAt`.
- **Error codes**:
  - `400 VALIDATION_ERROR`
  - `404 GROCERY_LIST_NOT_FOUND`
  - `409 VERSION_CONFLICT`

## CCO — Complete Chore Once

Marks one chore occurrence complete for a date.

- **Endpoint**: `POST /api/v1/chores/complete-once`
- **Required inputs**:
  - Header: `Idempotency-Key` (**required**), optional `X-Timezone`.
  - Body: `householdId`, `choreId`, `completedBy`, `date` (`YYYY-MM-DD`), optional `note` (max 500 chars).
- **Validation rules**:
  - `date` cannot be more than 7 days in the future (local timezone).
  - `completedBy` and `choreId` must be non-empty IDs.
  - Duplicate completion for same `choreId` + `date` is a no-op (idempotent semantic success).
- **Response shape** (`200`):
  - `completionId`, `choreId`, `date`, `completedAt`, `completedBy`, `status` (`created|already_completed`).
- **Error codes**:
  - `400 VALIDATION_ERROR`
  - `404 CHORE_NOT_FOUND`
  - `409 CHORE_ARCHIVED`

## CM — Calendar Month View

Returns normalized month view data for planning screens.

- **Endpoint**: `GET /api/v1/calendar/month`
- **Required inputs**:
  - Query: `householdId`, `month` (`YYYY-MM`), optional `include` CSV (`chores,menu,habits,briefings`).
  - Header: `X-Timezone` strongly recommended.
- **Validation rules**:
  - `month` must be valid Gregorian year-month.
  - `include` values must be in allow-list.
- **Response shape** (`200`):
  - `month`, `timezone`, `weeks[]`, where each week contains `days[]` with `date`, `inMonth`, and `summaries` object.
- **Error codes**:
  - `400 VALIDATION_ERROR`
  - `404 HOUSEHOLD_NOT_FOUND`

## CH — Chore History

Retrieves chore completion history in a date range.

- **Endpoint**: `GET /api/v1/chores/history`
- **Required inputs**:
  - Query: `householdId`, `from` (`YYYY-MM-DD`), `to` (`YYYY-MM-DD`), optional `choreId`, optional `assigneeId`.
  - Header: `X-Timezone` recommended.
- **Validation rules**:
  - `from <= to`.
  - date range must be <= 93 days.
- **Response shape** (`200`):
  - `householdId`, `from`, `to`, `entries[]` with `completionId`, `choreId`, `choreName`, `date`, `completedAt`, `completedBy`.
- **Error codes**:
  - `400 VALIDATION_ERROR`
  - `404 HOUSEHOLD_NOT_FOUND`

## HM — Habit Metrics

Computes weekly habit progress and streak statistics.

- **Endpoint**: `GET /api/v1/habits/metrics`
- **Required inputs**:
  - Query: `householdId`, `userId`, `weekStart` (`YYYY-MM-DD`, must be Monday local), optional `includeDaily=true|false`.
  - Header: `X-Timezone` required for week calculations.
- **Validation rules**:
  - Week starts Monday in `X-Timezone`.
  - If timezone missing, request fails (HM requires explicit timezone).
- **Response shape** (`200`):
  - `householdId`, `userId`, `weekStart`, `weekEnd`, `timezone`, `completionRate`, `streaks[]`, optional `daily[]`.
- **Error codes**:
  - `400 VALIDATION_ERROR`
  - `404 USER_NOT_FOUND`

## HBF — Habit Button Fire

Records a single habit button press from the tracker UI.

- **Endpoint**: `POST /api/v1/habits/button-fire`
- **Required inputs**:
  - Header: `Idempotency-Key` (**required**), `X-Timezone` required.
  - Body: `householdId`, `userId`, `habitId`, `occurredAt` (RFC 3339 timestamp), optional `clientEventId`.
- **Validation rules**:
  - Duplicate presses with same idempotency key must not create additional completion events.
  - `occurredAt` cannot be older than 30 days or more than 10 minutes in future (timezone-aware validation).
- **Response shape** (`200`):
  - `eventId`, `habitId`, `userId`, `countedDate`, `recordedAt`, `status` (`recorded|duplicate`).
- **Error codes**:
  - `400 VALIDATION_ERROR`
  - `404 HABIT_NOT_FOUND`
  - `409 HABIT_INACTIVE`
