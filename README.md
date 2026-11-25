# NexoMX

**Community Disaster Intelligence & Resilience Network**

A platform that connects official hazard information in Mexico with what citizens and civil protection teams need to do in a specific place: understand local risk, get prepared, report damage and coordinate the response.

> **Status:** definition. There is no code yet: the repo stays at this README and the MIT license until the project is approved. This README sets the scope, the domain rules and the session plan for the first stage.

---

## Contents

1. [What it is and what it isn't](#what-it-is-and-what-it-isnt)
2. [Origin](#origin)
3. [First stage: one complete vertical slice](#first-stage-one-complete-vertical-slice)
4. [Non-negotiable domain rules](#non-negotiable-domain-rules)
5. [Architecture](#architecture)
6. [Development environment and stack](#development-environment-and-stack)
7. [Proposed repository structure](#proposed-repository-structure)
8. [Session plan](#session-plan)
9. [What success looks like](#what-success-looks-like)
10. [How I work with the assistant in this repo](#how-i-work-with-the-assistant-in-this-repo)
11. [Data sources](#data-sources)

---

## What it is and what it isn't

NexoMX complements official channels; it doesn't replace them. It is not a new alerting authority, it doesn't issue evacuation orders, and it doesn't replace SMN, CONAGUA, CENAPRED, Civil Protection, SINAPROC, SIAT-CT or Cell Broadcast.

It works in the gap between the official warning and the action:

- **Citizens:** see what's happening in their municipality in plain language, with the source and the time it was last updated, and report damage such as flooding, fallen trees or power poles, blocked roads or damaged homes.
- **Civil Protection and local government:** receive those reports, verify them, assign them and follow them up with an audited timeline.

## Origin

The project started as **NexoMX Edge**: nodes that keep public weather data close to users and keep serving it when the origin goes down. A forecast alone isn't enough if people don't know what it means for their area or who to report what they see to, and if local teams have no structured way to receive and verify reports. So it grew into a community intelligence platform. Edge stays as a resilience component inside a larger architecture.

## First stage: one complete vertical slice

One municipality, one hazard (heavy rain and flooding) and one end-to-end flow. The hard part goes first: Edge nodes that keep working while cut off, and a simulator that proves it.

1. Run origin, Edge nodes, clients and failures inside a seeded, deterministic simulator.
2. Accept citizen reports and operator transitions on an Edge node while it's offline, as events with hybrid logical clocks.
3. Merge those events on reconnect with domain rules and send real conflicts to an operator queue.
4. Check convergence and the invariants below under random kills, partitions, packet loss and clock skew.
5. Ingest weather data from a fixture faithful to the SMN schema, with provenance, timestamps and supersede.
6. Expose an API with an OpenAPI contract, including as-of queries.
7. Show a web view of the local situation and a report form.
8. Review reports and change their status from an operator panel, with every transition audited.

**Out of scope for now:** national coverage, more hazards, in-house prediction, integration with official alerting, real message delivery and a pilot with authorities.

## Non-negotiable domain rules

These rules apply to all code, tests and UI:

1. **Information types stay separate.** The UI and the API always distinguish between original official data, data normalized by NexoMX, NexoMX analysis, approved recommendations, unverified citizen reports and verified reports. NexoMX analysis never looks like a government alert.
2. **Every piece of data shows its source and time.** Without a source and a timestamp, data is not published.
3. **Data expires.** Event time, publication time and reception time are different things. Data past its validity is shown as stale or stops being presented as current; never as a current warning.
4. **A report is not a fact.** No report becomes confirmed on its own, and no single report triggers an alert or an evacuation.
5. **Every transition is audited** with actor, time, previous state and new state.
6. **Permissions from day one.** Every sensitive operation checks the role of whoever runs it.
7. **Privacy by default.** Store only what's needed; names, phone numbers, faces and personal locations are never exposed in public views, and original evidence has restricted access.
8. **No made-up routes, shelters or protocols.** They are only shown when they have a registered source and validation.

## Architecture

```text
Official sources (fixtures in this stage)
             |
             v
Ingestion & data adapters
             |
             v
Normalization, provenance and validation
             |
             v
Risk context
      /      |       \
     v       v        v
Protocols  Reports  Incidents
      \      |       /
       v     v      v
         Messaging
             |
     +-------+-------+
     v               v
Citizen view    Operator panel

Edge: offline reads and writes, an event log
with hybrid logical clocks, and merge on reconnect.
```

It starts as a **modular monolith** with clear domain boundaries. No microservices until there is an operational reason for them. Business rules don't live in HTTP handlers.

### Data provenance

Each record stores, depending on what the source publishes:

```text
source_id, source_name, source_record_id
observed_at, published_at, received_at, normalized_at
valid_from, valid_until
schema_version, normalization_version
raw_payload_reference, quality_status
```

### Report workflow

```text
Received → Triaged → UnderReview → Verified → Assigned → InProgress → Resolved → Closed
                                 ↘ Rejected
```

`Duplicate`, `NeedsMoreInformation` or `Escalated` may be added. Any transition outside this graph is rejected.

### Edge

Outages hit hardest when reports matter most, so an Edge node keeps working while it's cut off: it serves what it already has and keeps taking new reports and operator transitions.

- Stores in SQLite and serves over local HTTP.
- Every write is an event in a local log, stamped with a hybrid logical clock (HLC) and the node id.
- Reports carry a client UUID and an idempotency key, so merging them is a union with no duplicates.
- On reconnect the origin replays transitions in HLC order. The first valid one wins. A transition whose previous state no longer matches is rejected, recorded in the audit log as a conflict and sent to the operator queue. Nothing is resolved silently.
- Forecasts are monotonic: the record with the latest issue time (model run, then `published_at`) is the current one. Older versions are kept only for as-of queries and are never shown as current. If the newest one is past `valid_until`, it's shown as stale; it never falls back to an older one.
- Sync is incremental and idempotent, with a version per record, and exposes when each record was synced.

**Limit:** a disconnected node can only serve what it already received. It can't get new official data during a full outage, and it can't alert devices without an independent channel.

| Scenario | Expected behavior |
| --- | --- |
| Normal connectivity | Periodic sync and local serving |
| Slow origin | Answer from valid cache and flag its age |
| Origin down | Serve synced data that is still valid and keep accepting reports and transitions |
| Intermittent connectivity | Retries with backoff and idempotent sync |
| Recovery | Merge offline events without duplicates; stale transitions go to the conflict queue |
| Older forecast arrives late | Store it for as-of queries; keep serving the newest one |
| Expired data | Show it as stale or stop presenting it as current |

### Simulator

Origin, Edge nodes, clients, links and failures all run in one process on a virtual clock. A seed controls node and origin kills, partitions, packet loss, delay and clock skew, so the same seed replays the same run and any convergence bug can be reproduced. The node code is the same one that runs over real HTTP; only the transport changes.

A checker runs after every simulated run and asserts the invariants: no lost or duplicated reports, no expired data served as current, the same state hash on every node after reconnecting, and every rejected transition present in the conflict queue.

## Development environment and stack

The project is developed on a hosted Linux desktop **with no internet, no GPU, no Docker and no apt-get**. Packages install from a mirror that may lack new or unusual versions. UIs are built as web apps that open in the desktop's browser.

Available tools the project uses: Node 22 with npm, PostgreSQL, SQLite, Chrome and Playwright, git.

### Stack

| Layer | Choice | Why |
| --- | --- | --- |
| Simulator, API, ingestion worker and Edge agent | TypeScript on Node 22 | Fully available in the environment; it's my daily stack |
| Web | React + TypeScript | Citizen view and operator panel in the browser |
| Central database | PostgreSQL with the `pg` driver | Pure JavaScript driver, no native build |
| Edge | SQLite with `node:sqlite` | Built into Node 22; avoids native drivers that need to download headers |
| Migrations | Versioned SQL files and a small in-house runner | Reproducible and reviewable, no external tooling |
| Contracts | OpenAPI | Document and validate the API |
| Tests | Vitest, Playwright and seeded chaos runs | Unit, integration, end-to-end and resilience |
| Metrics | Prometheus-format endpoint per service | Tests read the metrics directly |

**Few dependencies, stable versions.** Before adding a package, check that it's in the mirror. If a version is missing, loosen the range instead of forcing it.

### What doesn't work in this environment and how it's handled

| Limitation | Decision |
| --- | --- |
| **Go with the standard library only, no modules.** No PostgreSQL or SQLite driver. | The original plan was Go for the backend and Edge. It moves to TypeScript. If Go modules become available, the Edge agent is the first candidate to move back to Go. |
| **PostGIS is not guaranteed.** | Geometries stored as GeoJSON; containment and distance computed in the application. Enough for one municipality. |
| **No internet.** No live SMN API and no map tiles. | Fixtures faithful to the SMN schema; municipal boundaries as local GeoJSON; map without a basemap. |
| **No Docker or network tools.** | The simulator runs in-process on a virtual clock, so nodes, links and failures need no containers. Over real HTTP, outages are simulated by stopping the origin process or with an in-house proxy that delays or drops requests. Local PostgreSQL is started by a script in the repo. |
| **No Prometheus or Grafana server.** | Each service exposes `/metrics`; tests read and validate them. |

## Proposed repository structure

A monorepo with npm workspaces, organized by domain. This is a proposal; it gets adjusted in session 1.

```text
nexomx/
├── apps/
│   ├── api/          # HTTP API: situation, reports, incidents
│   ├── worker/       # Ingestion and normalization
│   ├── edge/         # Edge agent: SQLite, local API, offline writes
│   └── web/          # Citizen view and operator panel
├── packages/
│   ├── domain/       # Domain rules: states, permissions, validity
│   ├── sync/         # Event log, HLC, merge and conflict rules
│   ├── sim/          # Virtual clock, seeded faults, transport, checker
│   └── db/           # SQL migrations and PostgreSQL access
├── fixtures/
│   ├── smn/          # Responses faithful to the SMN schema, valid and invalid
│   └── geo/          # Municipal boundaries as GeoJSON
├── api/openapi/      # OpenAPI contract
├── scripts/          # PostgreSQL startup, seed and demo
├── docs/
│   ├── decisions/    # ADRs
│   └── architecture/
└── tests/
    ├── e2e/          # Playwright
    └── chaos/        # Seeded runs: outages, partitions, skew and recovery
```

## Session plan

Each session ends with passing tests and a concrete way to check the result. The hard part goes first; the app sits on top of it.

| # | Session | Checked by |
| --- | --- | --- |
| 1 | Monorepo structure, simulator and transport: virtual clock, seeded loss, delay, skew and kills, same node code over HTTP | The same seed produces the same event trace twice; a node killed mid-run restarts from its SQLite state |
| 2 | Event log and HLC on the Edge node: offline reports and transitions, idempotency keys, versioned records | Resending the same report with the same key doesn't duplicate it; HLC order holds under ±5 min skew |
| 3 | Merge and conflicts: report state machine on replay, precondition rejects, forecast supersede, conflict queue, audit log | A stale transition is rejected and lands in the conflict queue with an audit record; an older forecast never replaces a newer one |
| 4 | Checker and chaos run: invariants, state hash and the numbers in [What success looks like](#what-success-looks-like) | The 30-minute run passes every target; the same seed gives the same state hash and conflict list twice in a row |
| 5 | PostgreSQL schema and migrations, provenance model, SMN fixtures and ingestion with validation and idempotent upserts | Migrations run from scratch twice; ingesting the same fixture twice doesn't duplicate records; an invalid fixture is rejected |
| 6 | Public API with OpenAPI: situation by municipality, validity fields, as-of queries, pagination and consistent errors | Responses include source and validity; expired data comes back as stale; an as-of query returns what was current at that time |
| 7 | Auth and RBAC (citizen, operator, admin) on the origin and the Edge node, rate limits | A citizen can't change states, online or offline |
| 8 | Citizen view: situation page, report form and labels by information type | Playwright test: a citizen sees the situation with source and time, and submits a report while the node is offline |
| 9 | Operator panel: review queue, conflict queue, map with local GeoJSON, transitions and timeline | Playwright test: an operator takes a report from received to closed and resolves a conflict |
| 10 | ADRs, diagrams, metrics, documentation and a reproducible demo | Someone new can run the chaos run and the demo following only the README |

**Milestones:**

- **Milestone 1 (sessions 1–4):** Edge nodes take writes offline, merge on reconnect and pass the seeded chaos run.
- **Milestone 2 (sessions 5–7):** fixture data flows to the API with provenance, validity and as-of queries, with permissions.
- **Milestone 3 (sessions 8–10):** both UIs work on top, including offline, and the demo is reproducible.

## What success looks like

Not the number of screens. The core is done when a seeded chaos run passes:

- **Setup:** 1 origin and 5 Edge nodes, 30-minute runs with random node and origin kills, partitions, 20% packet loss and ±5 min clock skew, while 500 reports and 2,000 transitions go in.
- **0** lost or duplicated reports.
- **0** expired records served as current.
- **100%** of still-valid records served during outages.
- Every node reaches the **same state hash within 60 s** of reconnecting.
- Every rejected transition shows up in the conflict queue and the audit log.
- Edge reads under **50 ms at p95**.
- The **same seed replays to the same state hash and the same conflict list, twice in a row**, so any failure can be debugged.

The first stage is done when, on top of that:

- Ingestion is repeatable and rejects invalid data.
- The API returns data with its source, validity and coverage, and answers as-of queries.
- A citizen submits a report, online or offline, and an operator takes it to closed with an audit trail.
- Wrong permissions are rejected.
- The system tells official data, its own analysis and unverified reports apart.

These numbers are test goals, not results.

## How I work with the assistant in this repo

- Read the relevant files and this README before changing code.
- Small changes with a clear scope; nothing outside what the session asks for.
- Every new feature comes with its tests, and they run before anything is called done.
- Decisions that change the plan are recorded as ADRs in `docs/decisions/`.
- If an environment limitation blocks something, it gets documented here instead of being worked around silently.

## Data sources

This stage works with fixtures. Before connecting any real source, verify its endpoints, schema, units, frequency, license and terms of use against its current documentation.

- [SMN — Web Service API](https://smn.conagua.gob.mx/es/web-service-api)
- [INEGI — ENDUTIH 2024](https://www.inegi.org.mx/programas/dutih/2024/)
- [Datos Abiertos — Free Public Internet Access Points (PAIG)](https://www.datos.gob.mx/dataset/puntos_acceso_internet_gratuitos_habilitados_paig)
- [Datos Abiertos — Natural Hazard Alerts 2026](https://www.datos.gob.mx/dataset/alertamiento_fenomenos_naturales_2026)
- [CENAPRED — Risk communication at the community level](https://www.gob.mx/cenapred/articulos/comunicacion-del-riesgo-en-el-ambito-comunitario)
- [CENAPRED — Early warning and early action](https://www.gob.mx/cenapred/es/articulos/alerta-temprana-acciones-tempranas)
- [DOF — National Civil Protection Program 2026–2030](https://sidof.segob.gob.mx/notas/docFuente/5787683)

---

Fernando Acuña Martínez · Personal project
