# SmartHome Normalized Data Model and Migration Plan

This document defines a normalized logical schema that supports chores, habits, menu planning, grocery list management, recipes, and calendar workflows.

> **Scope note:** This is technology-agnostic SQL modeling. Concrete SQL migration files should be created under `services/<service-name>/migrations/` **after** selecting the database engine (e.g., Postgres, MySQL, SQLite) so engine-specific types and indexing syntax are correct.

## Modeling Principles

- Use a stable UUID (or equivalent opaque ID) primary key for all entities.
- Use `created_at` and `updated_at` timestamps on all mutable tables.
- Keep recurring definitions separate from completion/event facts.
- Derive statuses from due dates + completion facts at query time (or via read-model projection), rather than persisting denormalized status fields.
- Store timezone-aware schedule data to avoid date-boundary errors.

## Core Entities

## 1) `User`
Represents an application user.

**Fields**
- `id` (PK)
- `email` (required, unique)
- `display_name` (required)
- `locale` (required; BCP-47 like `en-US`)
- `timezone` (required; IANA like `America/Los_Angeles`)
- `created_at` (required)
- `updated_at` (required)

**Indexes / constraints**
- Unique: `email`
- Index: `timezone`

---

## 2) `Chore`
A recurring or one-off household task template assigned to a user.

**Fields**
- `id` (PK)
- `user_id` (FK -> `User.id`, required)
- `title` (required)
- `description` (nullable)
- `active` (required, default true)
- `start_date` (required; first local date eligible)
- `end_date` (nullable; last local date eligible)
- `cadence_type` (required enum: `daily`, `weekly`, `interval_days`, `monthly_by_day`, `one_time`)
- `cadence_interval` (nullable integer; required for `interval_days`, e.g., every 3 days)
- `cadence_weekdays` (nullable set/array of ISO weekdays 1-7; required for `weekly`)
- `cadence_monthday` (nullable integer 1-31; required for `monthly_by_day`)
- `due_time_local` (nullable local time; optional same-day priority ordering)
- `grace_period_hours` (required integer, default 0)
- `timezone` (required; defaults from user but copied for historical stability)
- `created_at` (required)
- `updated_at` (required)

**Indexes / constraints**
- Index: (`user_id`, `active`)
- Index: (`user_id`, `cadence_type`)
- Check constraints to enforce cadence field consistency.

---

## 3) `ChoreCompletion`
Fact table recording each completed occurrence of a chore.

**Fields**
- `id` (PK)
- `chore_id` (FK -> `Chore.id`, required)
- `user_id` (FK -> `User.id`, required)
- `occurrence_date_local` (required local date this completion satisfies)
- `completed_at` (required timestamp)
- `completion_note` (nullable)
- `created_at` (required)

**Indexes / constraints**
- Unique: (`chore_id`, `occurrence_date_local`) to prevent duplicate completions for one scheduled occurrence
- Index: (`user_id`, `occurrence_date_local`)
- Check: `user_id` must match owning `Chore.user_id` (via FK strategy/trigger depending on DB)

---

## 4) `Habit`
A recurring behavior definition for streaks/weekly progress.

**Fields**
- `id` (PK)
- `user_id` (FK -> `User.id`, required)
- `name` (required)
- `description` (nullable)
- `active` (required, default true)
- `target_type` (required enum: `times_per_week`, `days_per_week`)
- `target_count` (required integer > 0)
- `week_start_day` (required ISO weekday 1-7; default derived from user locale)
- `timezone` (required)
- `start_date` (required)
- `end_date` (nullable)
- `created_at` (required)
- `updated_at` (required)

**Indexes / constraints**
- Index: (`user_id`, `active`)
- Index: (`user_id`, `week_start_day`)

---

## 5) `HabitCompletion`
Fact table for habit check-ins.

**Fields**
- `id` (PK)
- `habit_id` (FK -> `Habit.id`, required)
- `user_id` (FK -> `User.id`, required)
- `completed_at` (required timestamp)
- `completion_date_local` (required local date in habit timezone)
- `value` (nullable numeric; for future quantity-based habits)
- `note` (nullable)
- `created_at` (required)

**Indexes / constraints**
- Index: (`habit_id`, `completion_date_local`)
- Index: (`user_id`, `completion_date_local`)

---

## 6) `MenuSelection`
Represents a meal plan decision per date/meal slot.

**Fields**
- `id` (PK)
- `user_id` (FK -> `User.id`, required)
- `selection_date_local` (required)
- `meal_slot` (required enum: `breakfast`, `lunch`, `dinner`, `snack`)
- `recipe_id` (nullable FK -> `Recipe.id`)
- `custom_label` (nullable; used when no recipe is linked)
- `servings` (nullable integer > 0)
- `created_at` (required)
- `updated_at` (required)

**Indexes / constraints**
- Unique: (`user_id`, `selection_date_local`, `meal_slot`)
- At least one of `recipe_id` or `custom_label` required.

---

## 7) `Recipe`
Reusable recipe metadata.

**Fields**
- `id` (PK)
- `user_id` (FK -> `User.id`, required; owner)
- `name` (required)
- `description` (nullable)
- `default_servings` (nullable integer > 0)
- `instructions` (nullable text/json)
- `created_at` (required)
- `updated_at` (required)

**Indexes / constraints**
- Index: (`user_id`, `name`)

---

## 8) `RecipeIngredient`
Normalized ingredient lines for a recipe.

**Fields**
- `id` (PK)
- `recipe_id` (FK -> `Recipe.id`, required)
- `name` (required)
- `quantity` (nullable decimal)
- `unit` (nullable; e.g., `g`, `cup`, `pcs`)
- `preparation` (nullable; e.g., `chopped`)
- `optional` (required boolean, default false)
- `sort_order` (required integer, default 0)
- `created_at` (required)
- `updated_at` (required)

**Indexes / constraints**
- Index: (`recipe_id`, `sort_order`)

---

## 9) `GroceryItem`
A grocery list entry, either manually entered or generated from planning context.

**Fields**
- `id` (PK)
- `user_id` (FK -> `User.id`, required)
- `name` (required)
- `quantity` (nullable decimal)
- `unit` (nullable)
- `needed_by_date_local` (nullable)
- `status` (required enum: `open`, `purchased`, `archived`)
- `provenance_type` (required enum: `manual`, `bgl_generated`)
- `provenance_ref` (nullable string/json; source identifier such as BGL run ID)
- `source_menu_selection_id` (nullable FK -> `MenuSelection.id`)
- `source_recipe_id` (nullable FK -> `Recipe.id`)
- `source_recipe_ingredient_id` (nullable FK -> `RecipeIngredient.id`)
- `created_by_user_id` (nullable FK -> `User.id`; typically same as `user_id` for manual)
- `created_at` (required)
- `updated_at` (required)

**Indexes / constraints**
- Index: (`user_id`, `status`)
- Index: (`user_id`, `provenance_type`)
- Index: (`user_id`, `needed_by_date_local`)
- Constraint: if `provenance_type = manual`, all `source_*` fields must be null.
- Constraint: if `provenance_type = bgl_generated`, at least one `source_*` or `provenance_ref` must be present.

**Provenance workflow notes**
- Manual and generated items remain in one table for unified shopping UX.
- Provenance fields enable filters (`Manual`, `Generated`) and safer bulk actions (e.g., "regenerate generated-only items").
- Optionally maintain soft-linking (no cascade delete) for source references so historical groceries remain auditable.

---

## 10) `CalendarEvent`
User-owned calendar events and reminders.

**Fields**
- `id` (PK)
- `user_id` (FK -> `User.id`, required)
- `title` (required)
- `description` (nullable)
- `start_at` (required timestamp)
- `end_at` (nullable timestamp)
- `all_day` (required boolean, default false)
- `timezone` (required)
- `external_source` (nullable enum/string; e.g., `google`, `outlook`, `internal`)
- `external_event_id` (nullable string)
- `created_at` (required)
- `updated_at` (required)

**Indexes / constraints**
- Index: (`user_id`, `start_at`)
- Unique nullable pair for external sync: (`external_source`, `external_event_id`)

---

## Chore Recurrence and Status Derivation

## Required recurrence/cadence fields
For each `Chore`, these fields are required for schedule derivation:
- `cadence_type`
- `start_date`
- `timezone`
- Cadence-specific required field(s):
  - `interval_days` -> `cadence_interval`
  - `weekly` -> `cadence_weekdays`
  - `monthly_by_day` -> `cadence_monthday`

`due_time_local` and `grace_period_hours` are optional/auxiliary but recommended for clearer due boundaries.

## Derived status logic (`good`, `due today`, `past due`)
For a given `Chore` and "now" in chore timezone:
1. Generate scheduled occurrence dates up to current local date.
2. Exclude occurrences before `start_date` or after `end_date` (if set).
3. Mark an occurrence complete if a `ChoreCompletion` exists for that `occurrence_date_local`.
4. Let `latest_uncompleted_occurrence_date` be the earliest uncompleted scheduled date that is not in the future.

Then derive:
- `good`: no uncompleted occurrence exists up to now.
- `due today`: earliest uncompleted occurrence date == current local date.
- `past due`: earliest uncompleted occurrence date < current local date.

If using `due_time_local` + `grace_period_hours`, transition from `due today` to `past due` only after `occurrence_date_local + due_time_local + grace_period` passes in chore timezone.

## Habit Weekly Aggregation Logic (Locale + Timezone Explicit)

Weekly aggregation must be deterministic and user-comprehensible:

1. Determine timezone from `Habit.timezone` (fallback `User.timezone` only for legacy data).
2. Determine week start day from `Habit.week_start_day`; if not explicitly set in legacy records, derive from `User.locale` mapping.
3. Convert each `HabitCompletion.completed_at` to `completion_date_local` in the habit timezone.
4. Compute `week_start_date_local` for each completion using week start day.
5. Aggregate per (`habit_id`, `week_start_date_local`):
   - `completion_count = COUNT(*)`
   - For `days_per_week`, use `COUNT(DISTINCT completion_date_local)` as the progress measure.
6. Weekly status:
   - `met` when progress >= `target_count`
   - `in_progress` when progress > 0 and < `target_count`
   - `not_started` when progress = 0

**DST / locale handling requirements**
- Convert timestamps to local date *before* bucketing.
- Never compute week buckets in UTC.
- Treat `week_start_day` as persisted data to avoid locale-rule drift if OS/library defaults change.

## Normalization Notes

- `Chore`/`Habit` are definitions; `ChoreCompletion`/`HabitCompletion` are immutable-ish facts.
- `Recipe` and `RecipeIngredient` are split to avoid repeated ingredient text blobs and support grocery generation traceability.
- `MenuSelection` references recipes but allows free-form entries via `custom_label`.
- `GroceryItem` stores provenance pointers instead of duplicating recipe/menu metadata.

## Migration Plan (Technology-Agnostic)

After database selection, add SQL migrations under each owning service:

- `services/chores-service/migrations/`
  - `001_create_chores.sql`
  - `002_create_chore_completions.sql`
- `services/habits-service/migrations/`
  - `001_create_habits.sql`
  - `002_create_habit_completions.sql`
- `services/menu-service/migrations/`
  - `001_create_recipes.sql`
  - `002_create_recipe_ingredients.sql`
  - `003_create_menu_selections.sql`
- `services/grocery-service/migrations/`
  - `001_create_grocery_items.sql`
  - `002_add_grocery_provenance_constraints.sql`
- `services/calendar-service/migrations/`
  - `001_create_calendar_events.sql`
- Shared identity ownership decision:
  - Either central auth/user DB migration (`services/user-service/migrations/001_create_users.sql`) or externally managed user table with foreign-key strategy adjusted accordingly.

## Migration rollout phases

1. **Foundation phase**: create `User`, `Recipe`, `RecipeIngredient`, `Chore`, `Habit`, `CalendarEvent`.
2. **Fact phase**: create `ChoreCompletion`, `HabitCompletion`, `MenuSelection`, `GroceryItem`.
3. **Constraint phase**: add check constraints, unique constraints, and foreign keys where cross-service boundaries allow.
4. **Performance phase**: add secondary indexes validated against real query plans.
5. **Backfill phase (if migrating existing data)**:
   - Populate local date helper columns (`completion_date_local`, `occurrence_date_local`).
   - Backfill `provenance_type` as `manual` for legacy grocery items with unknown origin.
   - Set explicit `week_start_day` and `timezone` for habits.

## Open decisions before writing concrete SQL

- Database engine + migration tool (Flyway, Liquibase, Prisma, Alembic, etc.).
- Enum representation strategy (native enums vs check constraints).
- Cross-service FK enforcement (strict DB FKs vs application-level references in microservice boundaries).
- Whether to materialize read models for chore status and habit weekly summaries.
