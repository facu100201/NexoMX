# NexoMX

**Community Disaster Intelligence & Resilience Network**

A platform that connects official hazard information in Mexico with what citizens and civil protection teams need to do in a specific place: understand local risk, get prepared, report damage and coordinate the response.

> **Status:** definition. There is no code yet. This README sets the scope, the domain rules and the session plan for the first stage.

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

One municipality, one hazard (heavy rain and flooding) and one end-to-end flow:

1. Ingest weather data from a fixture faithful to the SMN schema.
2. Normalize and store every record with provenance and timestamps.
3. Expose an API with an OpenAPI contract.
4. Query the situation by municipality and time window.
5. Show a web view of the local situation.
6. Create citizen reports.
7. Review reports and change their status from an operator panel.
8. Audit every transition.
9. Serve already-synced data from an Edge node with SQLite during a simulated outage of the origin.

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

Edge: sync, local storage and serving of
previously received information.
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

- Incremental, idempotent sync, with a version or sequence number per record.
- Never overwrites newer data with an older version.
- Stores in SQLite and serves over local HTTP.
- Applies expiry policies and exposes when each record was synced.

**Limit:** a disconnected node can only serve what it already received. It can't get new data during a full outage, and it can't alert devices without an independent channel.

| Scenario | Expected behavior |
| --- | --- |
| Normal connectivity | Periodic sync and local serving |
| Slow origin | Answer from valid cache and flag its age |
| Origin down | Serve synced data that is still valid |
| Intermittent connectivity | Retries with backoff and idempotent sync |
| Recovery | Detect changes and sync without duplicates |
| Expired data | Show it as stale or stop presenting it as current |

## Development environment and stack

The project is developed on a hosted Linux desktop **with no internet, no GPU, no Docker and no apt-get**. Packages install from a mirror that may lack new or unusual versions. UIs are built as web apps that open in the desktop's browser.

Available tools the project uses: Node 22 with npm, PostgreSQL, SQLite, Chrome and Playwright, git.

### Stack

| Layer | Choice | Why |
| --- | --- | --- |
| API, ingestion worker and Edge agent | TypeScript on Node 22 | Fully available in the environment; it's my daily stack |
| Web | React + TypeScript | Citizen view and operator panel in the browser |
| Central database | PostgreSQL with the `pg` driver | Pure JavaScript driver, no native build |
| Edge | SQLite with `node:sqlite` | Built into Node 22; avoids native drivers that need to download headers |
| Migrations | Versioned SQL files and a small in-house runner | Reproducible and reviewable, no external tooling |
| Contracts | OpenAPI | Document and validate the API |
| Tests | Vitest, Playwright and disconnection tests | Unit, integration, end-to-end and resilience |
| Metrics | Prometheus-format endpoint per service | Tests read the metrics directly |

**Few dependencies, stable versions.** Before adding a package, check that it's in the mirror. If a version is missing, loosen the range instead of forcing it.

### What doesn't work in this environment and how it's handled

| Limitation | Decision |
| --- | --- |
| **Go with the standard library only, no modules.** No PostgreSQL or SQLite driver. | The original plan was Go for the backend and Edge. It moves to TypeScript. If Go modules become available, the Edge agent is the first candidate to move back to Go. |
| **PostGIS is not guaranteed.** | Geometries stored as GeoJSON; containment and distance computed in the application. Enough for one municipality. |
| **No internet.** No live SMN API and no map tiles. | Fixtures faithful to the SMN schema; municipal boundaries as local GeoJSON; map without a basemap. |
| **No Docker.** | Local PostgreSQL started by a script in the repo. Outages are simulated by stopping the origin process or with an in-house proxy that delays or drops requests. |
| **No Prometheus or Grafana server.** | Each service exposes `/metrics`; tests read and validate them. |

## Proposed repository structure

A monorepo with npm workspaces, organized by domain. This is a proposal; it gets adjusted in session 1.

```text
nexomx/
├── apps/
│   ├── api/          # HTTP API: situation, reports, incidents
│   ├── worker/       # Ingestion and normalization
│   ├── edge/         # Edge agent: sync, SQLite and local API
│   └── web/          # Citizen view and operator panel
├── packages/
│   ├── domain/       # Domain rules: states, permissions, validity
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
    └── resilience/   # Outages, intermittency and recovery
```

## Session plan

Each session ends with passing tests and a concrete way to check the result.

| # | Session | Checked by |
| --- | --- | --- |
| 1 | Monorepo structure, PostgreSQL startup script, schema and migrations, provenance model and SMN fixtures | Migrations run from scratch twice without errors; fixtures load and validate |
| 2 | Ingestion adapter: validation, normalization, idempotent upserts and retries | Ingesting the same fixture twice doesn't duplicate records; an invalid fixture is rejected with a clear error |
| 3 | Public API with OpenAPI: situation and forecasts by municipality, validity fields, pagination and consistent errors | Responses include source and validity; expired data comes back marked as stale |
| 4 | Auth and RBAC (citizen, operator, admin); report submission with validation, idempotency keys and rate limits | A citizen can't change states; resending the same report with the same key doesn't duplicate it |
| 5 | Report state machine, audit log and duplicate detection | An invalid transition is rejected; every valid transition leaves an audit record |
| 6 | Citizen view: situation page, report form and labels by information type | Playwright test: a citizen sees the situation with source and time, and submits a report |
| 7 | Operator panel: review queue, map with local GeoJSON, transitions and timeline | Playwright test: an operator takes a report from received to closed |
| 8 | Edge agent: incremental sync with versions, SQLite, local API and expiry | Older data doesn't overwrite newer data; the node answers from SQLite |
| 9 | Resilience: origin down, intermittent connection, recovery, expired data, metrics and a load test | During the outage the node serves every valid record; on reconnect it syncs without duplicates |
| 10 | ADRs, diagrams, documentation and a reproducible demo | Someone new can run the demo following only the README |

**Milestones:**

- **Milestone 1 (sessions 1–3):** fixture data flows all the way to the API with provenance and validity.
- **Milestone 2 (sessions 4–7):** the report workflow works end to end with permissions and audit.
- **Milestone 3 (sessions 8–10):** Edge survives a simulated outage and the demo is reproducible.

## What success looks like

Not the number of screens. The first stage is done when:

- Ingestion is repeatable and rejects invalid data.
- The API returns data with its source, validity and coverage.
- A citizen submits a report and an operator takes it to closed, with an audit trail.
- Wrong permissions are rejected.
- The system tells official data, its own analysis and unverified reports apart.
- The Edge node serves every synced, still-valid record during an outage, and syncs without duplicates when it reconnects.
- Tests catch network failures, invalid data, duplicates and wrong permissions.

**Metrics:** data freshness, API latency (p50/p95/p99), report processing time, share of valid records available during the outage, and consistency after sync.

**Experimental targets** (test goals, not results): serve 100% of previously stored, still-valid records during an origin outage, and 100% consistency on the test set after sync.

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
