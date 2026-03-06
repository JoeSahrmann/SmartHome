# SmartHome Architecture (Baseline)

## Layers
- **Frontend (`frontend/`)**: user-facing page shells per feature.
- **Services (`services/`)**: backend microservices, one folder per domain.
- **Shared Contracts (`shared/contracts/`)**: API and schema contracts consumed by frontend and services.
- **Infrastructure (`infra/`)**: local orchestration and environment setup.

## Feature Mapping
- Daily briefing UI: `frontend/daily-briefing/`
- Chores UI + backend: `frontend/chores/`, `services/chores-service/`
- Menu UI + backend: `frontend/menu/`, `services/menu-service/`
- Grocery list UI + backend: `frontend/grocery-list/`, `services/grocery-service/`
- Calendar UI: `frontend/calendar/`
- Habit tracker UI + backend: `frontend/habit-tracker/`, `services/habits-service/`
- Daily briefing backend: `services/briefing-service/`
