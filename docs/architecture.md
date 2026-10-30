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

Record your plan here before editing code.

- Presentation tier decision: Keep React components responsible for rendering and interaction, and use the shared API client for relative `/api` requests. Every capability must expose loading, meaningful empty, and accessible error states.
- Logic tier decision: Keep HTTP parsing, validation, status codes, and response shapes in Express routes, with domain queries and calculations in the service layer. Return deliberate 4xx responses for invalid or missing resources; do not add authentication in this exercise.
- Data tier decision: Define required fields, references, and defaults in the Mongoose models. Preserve the seed script's idempotency and unrelated existing records.
- Activity tracking approach: List activities from persisted records. On creation, validate the user and team IDs and confirm both records exist, then store the user/team references, type, minutes, computed points, and server-generated occurrence time.
- Team management approach: List and create teams while preserving name, mascot, and member IDs. Reject a missing or blank name with `400`; default omitted members to an empty array.
- Leaderboard approach: Sum activity points by team, sort by total points descending with team name as a deterministic tie-breaker, and assign 1-based ranks in the returned order.
- Workout suggestions approach: Resolve the user by email, consider their fitness goal and recent activities when ordering available workouts, and include a concise reason for the top suggestion. Return a 4xx response when no user matches.

## Test plan

Run the narrow contract suites after changing their tier, then run both builds before considering the implementation complete. A contract failure indicates behavior or schema drift at that boundary; a build failure indicates a TypeScript or bundling issue.

- Data tier: `npm run test:data --prefix octofit-tracker/backend` checks model validation and seed idempotency. Baseline from the full backend suite: four model-validation tests fail; the seed/idempotency test passes.
- Logic tier: `npm run test:api --prefix octofit-tracker/backend` checks the HTTP contracts for activities, teams, leaderboard, and workout suggestions. Baseline from the full backend suite: all four API tests fail; invalid activity/team submissions currently return `501`, the leaderboard is empty, and workout suggestions omit the required reason.
- Presentation tier: `npm test --prefix octofit-tracker/frontend` checks rendered API data, user-specific workout requests, loading states, empty states, and accessible errors. Baseline: all 15 tests fail because the capability views remain at the implementation-pending state.
- Backend build: `npm run build --prefix octofit-tracker/backend` checks TypeScript compilation; this has not yet been run.
- Frontend build: `npm run build --prefix octofit-tracker/frontend` checks TypeScript compilation and the Vite production bundle; this has not yet been run.
