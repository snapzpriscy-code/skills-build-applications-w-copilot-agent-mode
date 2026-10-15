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

- Presentation tier decision: Keep routing and rendering in the React components; load data through the shared API client and show accessible loading, empty, and error states.
- Logic tier decision: Keep request handling in Express routes and validation/domain behavior in `octofitService.ts`; return deliberate 4xx responses for invalid input instead of leaking Mongoose cast errors as 500s.
- Data tier decision: Add required Mongoose validation and references for the documented fields. Keep the seed idempotent and use targeted upserts so it preserves records not owned by the demo fixtures.
- Activity tracking approach: Validate the user and team IDs and confirm both records exist before creating an activity. Store their references, type, minutes, computed points (one point per minute, consistent with the seed data), and an occurrence timestamp.
- Team management approach: Require a team name, accept its mascot, and default members to an empty array. Validate submitted member IDs and persist members as user references; list the saved team details and member IDs.
- Leaderboard approach: Aggregate activity points by team, include each team's name, sort by total points descending, and assign 1-based ranks in that order.
- Workout suggestions approach: Find the user by decoded email; return a 4xx when no user matches. Use the user's fitness goal and recent activity to rank relevant workouts, with the leading suggestion including a reason tied to that goal or activity.

## Test plan

Run the focused contract tests first, then both builds. A failing test should be fixed in its owning tier before moving on.

- Data-tier command: `npm run test:data --prefix octofit-tracker/backend`. Failures indicate incorrect model validation, references/defaults, or seed persistence/idempotence. Last run: 4 failed, 1 passed; the seed test passed.
- API command: `npm run test:api --prefix octofit-tracker/backend`. Failures indicate service or route contract issues, including input validation/status codes, stored activity values, team membership, leaderboard totals/ranks, or workout selection/reasons. Last run: 4 failed.
- Frontend command: `npm test --prefix octofit-tracker/frontend`. Failures indicate a view is not rendering API data or does not expose the required loading, empty, or accessible error state. Last run: 15 failed.
- Backend build command: `npm run build --prefix octofit-tracker/backend`. Failures indicate TypeScript compilation or backend type errors. Not yet run.
- Frontend build command: `npm run build --prefix octofit-tracker/frontend`. Failures indicate frontend TypeScript or Vite production-build errors. Not yet run.
