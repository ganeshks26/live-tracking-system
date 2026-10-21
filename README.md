# Live Tracking System

Consent-based live journey tracking and saved trip history for Android, iOS, and the web.

## Stack

- Mobile: React Native and TypeScript, with platform-specific background location integration.
- Web: React and TypeScript.
- Backend: Java and Spring Boot.
- Durable location pipeline: Kafka.
- Journey storage: PostgreSQL; evaluate PostGIS when spatial queries are required.
- Latest-position cache: Redis.
- Long-term archives: object storage.

These are architectural choices, not a claim that the system has been implemented or load-tested.

## How we build

Deliver one small, working increment per branch and pull request. Keep main reviewable.
Each feature includes its API contract, relevant verification, and run instructions.
Do not commit credentials, real journeys, or personal coordinates.

| Increment | Deliverable | Completion check |
|---|---|---|
| 0 | Architecture and roadmap | Responsibilities and failure rules documented |
| 1 | Backend foundation | App starts; health endpoint and CI checks pass |
| 2 | Authentication and ownership | Users cannot access another user's trips |
| 3 | Trip lifecycle | Start, stop, and list owned trips with database migrations |
| 4 | Durable point ingestion | Bounded batches reach Kafka; acknowledgements and duplicate retries tested |
| 5 | Journey persistence | Consumer saves points idempotently; history endpoint returns ordered points |
| 6 | Live processing | Newer positions update Redis; old uploads cannot regress live position |
| 7 | Live delivery | Authenticated viewers receive updates and recover after reconnect |
| 8 | Mobile recording | SQLite queue survives network loss; locked-screen journeys tested on devices |
| 9 | Web dashboard | Live map and selectable trip history use real APIs |
| 10 | Sharing | Explicit grants, revocation, and live disconnection enforced |
| 11 | Scale validation | Load, backlog, storage, retention, and recovery measured against agreed targets |

Mobile and website work can use agreed API fixtures while backend work progresses.
Android and iOS background recording must each be tested; shared application code does not remove native platform requirements.

## First code increment

Create a Java 21 / Spring Boot backend under `backend/`, with a build wrapper,
a health endpoint, configuration through environment variables, and CI verification.
Select and pin supported dependency versions in that change.

Do not expose location ingestion or journey APIs before authentication and ownership checks exist.

## Planned layout

- `backend/`: application code and tests.
- `mobile/`: React Native application.
- `web/`: history and live tracking website.
- `infra/`: local infrastructure and deployment configuration.
- `docs/`: architecture decisions and API contracts.

Directories will be added with their first actual implementation.

## Design

See [the architecture decision](docs/architecture.md).

## Current status

Repository planning only. No tracking application, deployment, or capacity guarantee yet.
