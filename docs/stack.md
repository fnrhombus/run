# Tech stack & architecture preferences

Author's stated preferences and the open stack decision. The **stack choice
itself (React vs. F#) is evaluated in `implementation-plan.md`** — this file just
captures the preferences and the patterns that hold regardless of language.

## Stated preferences

- **The author's own command-pattern library** (CQRS-style) for the write side —
  commands → events, kept separate from the read side (queries / projections).
  (The author has this library already; "think CQRS" was just the hint for its
  shape.)
- **Event sourcing** — the data model is an append-only log of events.
- **Azure cloud** — *considering* an Azure **event-ingestion** service
  (Event Hubs) + an Azure **NoSQL / telemetry** store for the large volume of
  timestamped stream data.
- **Frontend / language is an open decision** — evaluated as **React** vs. an
  **F# solution** (which may or may not use React, e.g. via Fable) vs. any
  genuinely-better option. See `implementation-plan.md`.

> Correction note: an earlier draft of this file asserted a "local-first,
> own-your-data, no-cloud-lock-in" ethos. The author did **not** state that; it
> was invented and has been removed. The real stated motivations are narrower:
> dislike of paywalled features and of poor pace math in existing apps. Azure is
> a *preferred* cloud, not something to route around.

## Why CQRS + event sourcing fit this project well

The app is fundamentally **append-only timestamped streams** (sensor readings)
plus discrete **commands** (workout started, dose logged, shoe swapped). That is
event sourcing's home turf:

- **Events** = immutable facts: each HR sample, GPS fix, RR interval, med dose,
  RPE entry. This *is* the F0 data store, expressed as an event log.
- **Projections (the query side)** = read models derived from the stream: a
  session summary, daily readiness (F9), the F2 model's training inputs,
  PMC/CTL-ATL-TSB (F6). When the model changes, **replay the log** to rebuild
  them.
- **CQRS** keeps the high-rate ingest path (append events) cleanly separate from
  the query/projection path.

## Storage choice for the timestamp firehose (decide in plan)

Weigh on cost and fit (not yet decided):

- **Azure Cosmos DB** (NoSQL) — flexible, globally distributed; watch
  RU/throughput cost for high-frequency multi-stream writes.
- **Azure Data Explorer (Kusto)** — purpose-built for time-series / telemetry at
  volume; often a better and cheaper fit for "vast timestamp stream data," with
  strong time-series query operators. Worth a serious look.
- **Blob storage (Parquet) for cold data** — cheap archival of raw event
  streams; pairs with open-format export.

Likely tiered: hot recent data in a queryable store, cold raw events archived as
Parquet in Blob.

## Engineering consideration (NOT a stated requirement — confirm)

Runs happen outdoors, often with poor or no connectivity, yet the app must
capture every sample mid-run. That points to **buffering events on the device
and syncing to Azure (Event Hubs) opportunistically** when a connection is
available — a practical reliability need, not a data-ownership stance. Flagging
as an open question to confirm, not an assumed ethos.
