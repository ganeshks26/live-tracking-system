# Architecture decision: durable location ingestion

Status: proposed implementation architecture. Capacity targets below require benchmarking.

## Responsibilities

The mobile app records location samples into a persistent local SQLite queue.
A React Native UI controls recording; native Android and iOS location facilities
perform background collection. Recording intervals are requests, not timing guarantees.

The authenticated ingestion API validates bounded batches and publishes to Kafka.
History and live processing use independent consumer groups. PostgreSQL stores
journey history; Redis provides a rebuildable latest-position view. The web application
accesses these through authenticated backend APIs.

Start with one backend codebase with separate modules for accounts, trips, ingestion,
history, live processing, and sharing. API and worker execution can later be deployed
independently from that codebase. Separate repositories or microservices are not
required to preserve these boundaries.

## Point contract

Each point contains a client-generated unique point ID, trip ID, recording-session ID,
monotonic session sequence, UTC recording timestamp, latitude, longitude, and
horizontal accuracy when available. Speed and heading are optional.
The server adds its receipt timestamp. Identity comes from authentication.

Validate coordinate ranges, finite numeric values, timestamps, allowed fields, payload
bytes, batch count, ownership, trip state, and rate limits. Exact limits will be
specified in the API contract and verified with boundary tests.

## Delivery guarantees

- Save locally before attempting upload. Delete pending data only after acknowledgement.
- Acknowledge ingestion only after durable stream publication succeeds using appropriate
  replication and producer acknowledgement settings. A receipt means queued durably,
  not yet queryable in history.
- Network timeouts may leave publication outcome unknown. Retry with the same IDs.
  Transport is at least once; consumers must tolerate duplicates.
- Partition by trip ID for processing locality. Offline uploads can arrive out of
  recording order, so arrival order is not journey order.
- Persist history using uniqueness constraints on point identity. Commit consumer
  offsets after the database transaction succeeds.
- A stale point can enter history without changing the current live position.
  Define a deterministic ordering key using validated recording time and session
  sequence; detect clock anomalies rather than assuming device clocks are perfect.
- Redis updates must atomically compare the ordering key before replacing a position.
- Keep API and worker retry limits bounded; isolate poison events for inspection.
  Alert on rejected batches, stream lag, and consumer failures.

## Trip lifecycle and offline sync

Trips have client-generated IDs to support offline recording. Trip creation is
idempotent. A stopped trip may still have pending points. Represent recording completion
and upload completion separately; accept eligible late points captured before its
end, with a documented grace policy.

A phone can be offline or suspended without the user intentionally stopping a trip.
Show last-recorded and last-received times, and distinguish stale, syncing, and completed
states. Never promise continuous GPS recording through force-stop or disabled permissions.

## Live delivery

A viewer loads a snapshot and ordered route points, then subscribes using authenticated
SSE or WebSocket delivery. Implement cursor-based catch-up to close the gap between
snapshot loading and subscription. Events carry point IDs for deduplication.

A multi-instance backend needs shared live-event distribution; an in-memory listener
alone cannot reach viewers connected to another instance. Select the fan-out mechanism
in the live-delivery increment. Redis Pub/Sub may carry transient notifications only
when reconnect and missed-event recovery come from durable history.

Only stream updates to authorised viewers. Slow connections have bounded buffers.
Sharing revocation must invalidate access and close existing subscriptions.

## History and map rendering

Return points in recording order with cursor pagination and limits. Return simplified
route geometry for large overviews, with detail fetched as needed. Mark large recording
gaps rather than drawing a misleading continuous route. Keep accuracy metadata;
filter unreliable points for presentation and distance estimates using documented rules.

Recorded route geometry is distinct from directions returned by a map provider.

## Capacity model

Illustrative workload: 100,000 simultaneously recording users, one point every
5 seconds, one upload every 15 seconds, continuously for 5 hours:

- Approximately 20,000 points per second.
- Approximately 6,667 upload requests per second, excluding retries.
- Approximately 360 million points over that window.

These are planning inputs, not demonstrated capacity. Registered-user count is not
concurrent-user count. Also model concurrent viewers, fan-out, offline reconnect bursts,
payload size, index overhead, replication, and retention.

Use batch database writes and time-based history partitions as volume warrants.
Choose partition keys and unique constraints together; PostgreSQL partitioned-table
uniqueness rules affect idempotency design. Benchmark before fixing partition sizing.
Archive older compressed journeys to object storage under an explicit retention policy.

## Security and operations

Use TLS, verified identities, ownership checks, explicit sharing grants, and per-user
and per-device rate limits. Keep coordinates out of routine logs and tracing.
Define deletion across phone queues, database, caches, streams, and archives;
stream retention and backup expiry must be disclosed in that policy.

Monitor ingestion latency, failed uploads, duplicate rates, processing lag, history
query latency, stale trips, live connections, and storage growth. Test crash recovery
and replay. Document backup and restore procedures.

## Decisions required before a scale claim

Agree on expected active users and viewers, acceptable live delay, battery budget,
history fidelity, offline duration, availability objectives, and retention period.
Then load-test sustained traffic, bursts, consumer outages, and recovery.

## First implementation boundary

Backend startup and health verification come first. Authentication and trip ownership
precede location APIs. Durable ingestion, history consumption, and live delivery are
separate, reviewable feature increments. Real location data must remain private even
though this source repository is public.
