# SmartHome Monorepo Baseline

This repository now includes a baseline structure with clear boundaries between frontend, backend services, shared contracts, and infrastructure.

## Project Layout
- `frontend/` – feature page shells for:
  - `daily-briefing`
  - `chores`
  - `menu`
  - `grocery-list`
  - `calendar`
  - `habit-tracker`
- `services/` – backend service boundaries:
  - `chores-service`
  - `habits-service`
  - `menu-service`
  - `grocery-service`
  - `briefing-service`
- `shared/contracts/` – shared API/event/schema contract area.
- `infra/` – deployment/orchestration definitions (currently Docker Compose).
- `docs/` – architecture and documentation.

## Run Commands
From repository root:

```bash
# Start local baseline stack (frontend + services + postgres)
docker compose -f infra/docker-compose.yml up -d

# View running containers
docker compose -f infra/docker-compose.yml ps

# Stop the stack
docker compose -f infra/docker-compose.yml down

# Stop and remove database volume (destructive)
docker compose -f infra/docker-compose.yml down -v
```

## Architecture Overview
The baseline follows a layered architecture:
1. **Frontend layer** (`frontend/`) for user-facing screens.
2. **Service layer** (`services/`) where domain logic lives in isolated services.
3. **Contract layer** (`shared/contracts/`) for shared schemas and API definitions.
4. **Infrastructure layer** (`infra/`) for local/dev orchestration and shared dependencies like Postgres.

A persistent PostgreSQL container is provisioned via Docker volume `smarthome-db-data`.

## Feature Implementation Map
- **Daily Briefing**: `frontend/daily-briefing/` + `services/briefing-service/`
- **Chores**: `frontend/chores/` + `services/chores-service/`
- **Menu**: `frontend/menu/` + `services/menu-service/`
- **Grocery List**: `frontend/grocery-list/` + `services/grocery-service/`
- **Calendar**: `frontend/calendar/`
- **Habit Tracker**: `frontend/habit-tracker/` + `services/habits-service/`

For more details, see `docs/architecture.md`.
