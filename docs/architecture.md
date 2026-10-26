# OctoFit Tracker architecture

OctoFit Tracker is a three-tier application for a school fitness program. The starter code includes API contracts and intentionally incomplete implementation points so you can use GitHub Copilot agent mode in a test-driven loop.

![Three-tier OctoFit architecture](../.github/images/octofit-architecture.svg)

## Tier boundaries

- **Presentation tier**: React 19, Vite, React Router, Bootstrap, and a shared API client in `octofit-tracker/frontend/src`.
- **Logic tier**: Express and TypeScript routes/services in `octofit-tracker/backend/src` expose `/api/*` behavior on port `8000`.
- **Data tier**: Mongoose models and seed data in `octofit-tracker/backend/src/models` and `src/scripts/seed.ts` persist to MongoDB on port `27017`.

## Ports and environment rules

- MongoDB: `27017`
- Express API: `8000`
- Vite frontend: `5173`
- The browser calls relative `/api` paths. Vite proxies them to `http://localhost:8000`, so the unauthenticated API port stays private in Codespaces. `VITE_API_BASE_URL` can override the base URL.

## API contract

| Capability | Method and path | Expected behavior |
| --- | --- | --- |
| Activity tracking | `GET /api/activities`, `POST /api/activities` | List and create valid activity records. POST accepts `userId`, `teamId`, `type`, and `minutes`; `userId` and `teamId` must be well-formed IDs for an existing user and team; otherwise respond with a deliberate 4xx (never a 500 from a cast error). A created activity stores the submitted `user`, `team`, `type`, and `minutes` plus computed `points` and `occurredAt`. |
| Teams | `GET /api/teams`, `POST /api/teams` | List and create teams with members. Created and listed teams keep the submitted `name` and `mascot`. Reject a team without a name with `400`. |
| Leaderboard | `GET /api/leaderboard` | Return teams ordered by the sum of their activity points, highest first. Each entry has `teamName`, `totalPoints`, and a 1-based `rank`. |
| Workout suggestions | `GET /api/workouts/suggestions/:userEmail` | Recommend workouts from recent activity patterns for the matching seeded user's email address. The first suggestion's `focus` matches that user's goal or recent activity, with a `reason` explaining why. An email with no matching user returns a deliberate 4xx (for example, `404`). |

## Implementation decisions

- Presentation tier decision: Build the React UI to consume the contract-driven API responses, with route-level fetches for activities, teams, leaderboard, and workout suggestions, and accessible loading/empty/error states so the browser can handle data transitions without custom business logic.
- Logic tier decision: Keep the Express API as the single contract boundary. Validate all incoming payloads in the route/service layer, convert schema and reference problems into deliberate 4xx responses, and compute derived values like activity points and rankings centrally instead of in the UI.
- Data tier decision: Model the domain with Mongoose schemas for users, teams, activities, and workouts. Require the meaningful fields the tests expect, keep arrays and ObjectId references consistent, and seed representative data idempotently without destroying sentinel records.
- Activity tracking approach: Accept `userId`, `teamId`, `type`, and `minutes`, confirm both references exist, create the activity with `user`, `team`, `type`, and `minutes`, and compute `points` plus `occurredAt` server-side. Invalid payloads must fail as client errors rather than throwing Mongo cast exceptions.
- Team management approach: Require a team name and mascot at creation time, allow a `members` array default to an empty list, and preserve submitted member IDs while rejecting incomplete payloads with `400`.
- Leaderboard approach: Aggregate total activity points by team, sort highest first, and assign a 1-based rank in the response payload so the front end can render stable rankings without recalculating them locally.
- Workout suggestions approach: Match a user by email, inspect recent activity patterns or fitness goals, and return the most relevant workouts in order with a clear `reason` string. Missing email matches should return a deliberate 4xx, not a generic success response.

## Test plan

- Data-tier command: `npm run test:data --prefix octofit-tracker/backend`
  - Current result: 4 failed, 1 passed.
  - Failure meaning: the data model validation is incomplete; required fields such as `name`, `email`, `fitnessGoal`, `type`, `user`, `team`, `points`, `occurredAt`, and workout metadata are not yet enforced by the Mongoose schemas.
- API command: `npm run test:api --prefix octofit-tracker/backend`
  - Current result: 4 failed, 0 passed.
  - Failure meaning: the Express layer is still stubbed or placeholder logic; invalid activity/team submissions are returning `501` instead of client errors, leaderboard results are empty, and workout suggestion matching is picking the wrong recommendation for the wrong user.
- Frontend command: `npm test --prefix octofit-tracker/frontend`
  - Current result: 15 failed, 0 passed.
  - Failure meaning: the presentation layer is not yet wired to fetch and render the API responses; route components are still showing placeholder text and failing to handle loading, empty, success, and error states.
- Backend build command: `npm run build --prefix octofit-tracker/backend`
  - Purpose: confirm TypeScript compiles cleanly once the data and API contracts are implemented.
- Frontend build command: `npm run build --prefix octofit-tracker/frontend`
  - Purpose: confirm the React app builds successfully with the final API client and component contracts.
- Execution order to resolve the current failures: data schema validation first, then API logic, then UI rendering, and finally build verification for both tiers.
