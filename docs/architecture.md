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

- Presentation tier decision: Keep the React app focused on UI rendering and user interaction. Use the shared API client in `octofit-tracker/frontend/src/api/client.ts` for all `/api` fetches, and make each page responsible only for presentation, loading, empty, success, and accessible error states. The app should not contain business rules or data validation; it should only render contract-driven results from the backend.
- Logic tier decision: Implement the backend contracts in Express/TypeScript services under `octofit-tracker/backend/src/services`, with the route layer acting as a thin adapter for request validation and HTTP status handling. Validation, ID checks, business logic, and computed fields belong here; routes should not hide errors or let Mongoose cast failures leak as unexpected 500 responses.
- Data tier decision: Preserve MongoDB/Mongoose model and seed-data contracts exactly as defined by the tests. Keep schema validation, references, defaults, and idempotent `seed()` behavior aligned with the data layer. This ensures user/team/activity/workout documents remain consistent across repeated runs and test resets.
- Activity tracking approach: Validate `userId`, `teamId`, `type`, and `minutes` in the service layer before saving. Require valid existing user and team IDs, reject invalid or missing references with deliberate 4xx responses, and compute `points` and `occurredAt` when creating an activity. List endpoints should return the persisted records in the expected shape without leaking internals.
- Team management approach: Validate team creation in the Express service layer by ensuring a team has a `name` and `mascot`, keeps `members` as an array, and stores valid member references. Listing teams should return the persisted team objects with member IDs intact, while rejecting invalid payloads with a 400-level response instead of a database cast error.
- Leaderboard approach: Aggregate activity points by team, order by descending `totalPoints`, and assign a 1-based rank in order. The `/api/leaderboard` response should expose `teamName`, `totalPoints`, and `rank` values exactly as the contract describes, using the user/team relationship and recent activity data already stored in MongoDB.
- Workout suggestions approach: Resolve the target user by email, then select workouts based on that user’s goal or recent activity pattern. The first suggestion should match the user’s focus (for example endurance or strength), include a `reason` explaining the match, and return a deliberate 4xx for an unknown email instead of a generic server error.

## Test plan

Record the commands you run and what each failure means.

- Data-tier command: `npm run test:data --prefix octofit-tracker/backend`  
  This verifies the Mongoose schema and seed behavior. Failures here mean the data contracts are not enforced: required fields are missing, default values are wrong, team/member relationships are inconsistent, or the idempotent seed script is not creating the expected demo data for users, teams, activities, and workouts.
- API command: `npm run test:api --prefix octofit-tracker/backend`  
  This verifies the Express contract for `/api/activities`, `/api/teams`, `/api/leaderboard`, and `/api/workouts/suggestions/:userEmail`. Failures here typically indicate missing service logic, incorrect validation, cast-error leakage, wrong HTTP status codes, leaderboard ordering issues, or workout recommendation logic that does not match the user’s goal/activity pattern.
- Frontend command: `npm test --prefix octofit-tracker/frontend`  
  This covers the presentation tier and currently shows the baseline of 15 failed tests. Failures here mean the UI is not correctly fetching and rendering activities, teams, leaderboard data, or workout suggestions, and/or it does not include the required loading, empty, and accessible error states required by the contract.
- Backend build command: `npm run build --prefix octofit-tracker/backend`  
  This checks TypeScript compilation for the logic/data tiers. A backend build failure means a type contract or code structure issue exists before runtime tests, such as invalid imports, schema typing problems, or service code that does not match the declared Express/Mongoose interfaces.
- Frontend build command: `npm run build --prefix octofit-tracker/frontend`  
  This checks the Vite/React presentation build, including TypeScript and bundling. A frontend build failure means the UI components or API client are structurally invalid or incompatible with the current React/Vite configuration, even if the test suite is not yet green.

Implementation order to follow while editing:
1. Data tier
2. API/logic tier
3. Presentation tier
4. Backend and frontend builds and regression tests

This order is safe because the schema and seed data define the contract the services must satisfy, the API depends on those contracts, and the UI can only be trusted once the backend behavior is correct and stable.
