# Tech stack & architecture preferences

Author's stated preferences (to drive the implementation plan). Rough capture —
decisions marked *considering* are not final.

## Preferences

- **Frontend: React.** Already the stack — this is an Expo / React Native app
  (`package.json`), so React knowledge transfers directly. ✅ no change.
- **Event sourcing.** Strong fit (see below).
- **A command-pattern library of the author's own creation.** Use it for the
  write/command side.
- **Azure cloud services.** *Considering:* an Azure **event-ingestion** service
  (Event Hubs) + an Azure **NoSQL** store (Cosmos DB) for the large volume of
  timestamped stream data.

## Why event sourcing fits this project well

The whole app is, fundamentally, **append-only timestamped streams** (sensor
readings) plus discrete **commands/events** (workout started, dose logged, shoe
swapped). That is event sourcing's home turf:

- **Events** = immutable facts: each HR sample, GPS fix, RR interval, med dose,
  RPE entry. This *is* the F0 data store, expressed as an event log.
- **Projections** = derived read models built from the event stream: a session
  summary, daily readiness (F9), the F2 model's training inputs, PMC/CTL-ATL-TSB
  (F6). Recompute projections any time the model changes — replay the log.
- The author's **command pattern** library handles the command side (intent →
  validated state change → emitted event).
- Bonus: event sourcing makes the local↔cloud sync below natural — you ship an
  event stream, not a mutable database diff.

## The one tension to resolve in the plan: local-first vs. cloud

F0's ethos is **local-first, own-your-data, works-offline**. That matters
concretely here: you run *outdoors*, often with poor/no connectivity, and the
app must capture every sample mid-run regardless. So:

- **Device is the source of truth during capture.** Append events to a local
  event log on the phone (e.g. SQLite / on-device store) — no network required.
- **Azure is the durable sink + heavy-compute tier.** Forward the local event
  stream to **Event Hubs** when connectivity allows; project into cloud storage
  for durability, long-term history, and the compute-heavy model fitting (F2)
  and training analytics. This *complements* local-first rather than replacing
  it; export-to-open-formats (F0) still holds.

This reconciles "I like Azure" with "own your data, no paywall, runs offline":
local-first capture, cloud as an optional sync/analysis layer.

## Storage choice for the timestamp firehose (decide in plan)

Cosmos DB works, but for high-rate, multi-channel telemetry it's worth weighing
options on cost and fit (not yet decided):

- **Azure Cosmos DB** (NoSQL) — flexible, globally distributed; watch
  RU/throughput cost for high-frequency multi-stream writes.
- **Azure Data Explorer (Kusto)** — purpose-built for time-series / telemetry at
  volume; often a better and cheaper fit for "vast timestamp stream data," with
  strong time-series query operators. Worth a serious look.
- **Blob storage (Parquet) for cold data** — cheap archival of raw event
  streams; pairs well with the F0 open-export goal.

Likely a tiered approach: hot recent data in a queryable store, cold raw events
archived as Parquet in Blob.

## Implications for milestones

- Even **M1** (the minimal HR+GPS logger) should write through the
  **event-sourced local store** from day one, so the architecture is right from
  the start. Cloud sync (Event Hubs → Azure storage) is a *later* milestone, not
  M1 — M1 stays local and offline.
- The command-pattern library and projection layer get introduced early so
  features layer on as new projections rather than schema rewrites.
