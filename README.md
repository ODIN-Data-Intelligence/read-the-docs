<p align="center">
  <img src="https://www.odin-catalog.com/logo-light.png" />
</p>

# ODIN Catalog — Documentation

> **v0.0.1-alpha** — APIs and database schemas may change between releases.

---

## Table of Contents

- [Getting Started](#getting-started)
  - [Introduction](#introduction)
  - [Quick Start](#quick-start)
  - [Prerequisites](#prerequisites)
  - [Configuration](#configuration)
- [Architecture](#architecture)
  - [Overview](#overview)
  - [Services](#services)
  - [Event Topology](#event-topology)
  - [Security Model](#security-model)
- [Data Model](#data-model)
  - [Metamodel Overview](#metamodel-overview)
  - [DCAT Datasets](#dcat-datasets)
  - [DPROD Data Products](#dprod-data-products)
  - [CSV-W Physical Schema](#csv-w-physical-schema)
  - [Logical Models](#logical-models)
  - [Vocabulary & FIBO](#vocabulary--fibo)
  - [OpenLineage](#openlineage)
- [Services](#services-1)
  - [Catalog Service](#catalog-service)
  - [Harvest Service](#harvest-service)
  - [Lineage Service](#lineage-service)
  - [Search Service](#search-service)
  - [AI Service](#ai-service)
  - [Identity Service](#identity-service)
- [API Reference](#api-reference)
  - [Authentication](#authentication)
  - [Catalog API](#catalog-api)
  - [Harvest API](#harvest-api)
  - [Lineage API](#lineage-api)
  - [Search API](#search-api)
  - [AI API](#ai-api)
- [Deployment](#deployment)
  - [Docker Compose](#docker-compose)
  - [Environment Variables](#environment-variables)
  - [Kubernetes (Helm)](#kubernetes-helm)
- [Contributing](#contributing)
  - [Local Development](#local-development)
  - [Contribution Guide](#contribution-guide)

---

## Getting Started

### Introduction

ODIN Catalog is an open-source data catalog built on W3C and OMG standards. It bridges the gap between raw technical metadata and business understanding — giving data teams a semantic layer, end-to-end lineage, and AI-powered discovery out of the box.

**What makes ODIN different:**

- **Semantic vocabulary mappings** — every data element can be bound to a concept in FIBO, schema.org, or your own ontology using SKOS match types.
- **Live lineage graph** — OpenLineage events and SQL DDL are parsed into an Apache AGE property graph queryable by Cypher.
- **Data product governance** — the DPROD standard gives every dataset a business owner, lifecycle stage, and access policy.
- **AI-powered Q&A** — a Spring AI RAG pipeline runs over your metadata corpus using Ollama (local) or OpenAI.
- **Zero lock-in** — all metadata is exportable as DCAT 3.0 JSON-LD.

**Standards at the core:**

| Standard | Body | Role in ODIN |
|---|---|---|
| **DCAT 3.0** | W3C | Catalog, Dataset, Distribution, DataService resources |
| **DPROD** | OMG | DataProduct, Port, lifecycle, access policy |
| **CSV-W** | W3C | Physical schema (table, column, datatype) harvested from source systems |
| **OpenLineage** | Linux Foundation | Job/Run/Dataset lineage events ingested via REST |
| **FIBO** | EDM Council | Pre-loaded financial ontology vocabulary (FND, FBC, SEC, MD) |
| **SKOS** | W3C | Mapping properties: exactMatch, closeMatch, relatedMatch |

> **Alpha notice:** ODIN is currently in private alpha. APIs and database schemas may change between releases. Not recommended for production workloads yet.

---

### Quick Start

Get a full ODIN stack running locally in under five minutes using Docker Compose.

**1. Clone and configure**

```bash
git clone https://github.com/odin-catalog/odin.git
cd odin
cp .env.example .env          # review and edit credentials
```

**2. Start the stack**

```bash
make up
# or: docker compose up -d

# Watch services come healthy:
docker compose ps
```

Services start in dependency order. Allow ~60 seconds for Kafka, PostgreSQL, and OpenSearch to initialise before the Spring Boot services become healthy.

**3. Create your first dataset**

```bash
curl -s -X POST http://localhost:8001/api/v1/datasets \
  -H "Content-Type: application/json" \
  -H "X-API-Key: dev-local" \
  -H "X-Tenant-Id: 00000000-0000-0000-0000-000000000001" \
  -d '{
    "title": "Trade Blotter",
    "description": "Intraday trade records from the front-office OMS.",
    "keywords": ["trading", "blotter", "positions"],
    "accrualPeriodicity": "daily"
  }' | jq .
```

**4. Open the frontend**

| App | URL | Purpose |
|---|---|---|
| Producer (management) | `http://localhost:3000` | Publish, govern, harvest |
| Consumer (discovery) | `http://localhost:3001` | Search, explore, ask AI |

> **Dev API key:** Any value starting with `dev-` (e.g. `X-API-Key: dev-local`) grants full `catalog:admin` scope and bypasses Keycloak. Use it for local smoke testing only.

**5. Load sample data**

```bash
make seed        # loads financial services sample data
make reindex     # pushes all datasets into OpenSearch
```

The seed script creates 12 financial datasets, 5 data products, logical models with FIBO vocabulary mappings, and OpenLineage pipeline events for a BCBS 239 risk aggregation scenario.

---

### Prerequisites

**Runtime requirements**

| Requirement | Minimum version | Notes |
|---|---|---|
| Docker | 25.0 | Docker Desktop or Docker Engine on Linux |
| Docker Compose | v2.24 | Bundled with Docker Desktop; `docker compose` (v2 plugin) |
| RAM | 12 GB available | OpenSearch and Kafka are the largest consumers |
| Disk | 8 GB free | Container images + volumes |

**Development requirements**

| Requirement | Version | Notes |
|---|---|---|
| Java | 21 (LTS) | Required to build services; virtual threads (Project Loom) |
| Gradle | 8.x | Wrapper included; run `./gradlew` |
| Node.js | 20 LTS | Required for frontend builds |
| pnpm | 9.x | `npm install -g pnpm` |

**Optional — AI features**

```bash
# Install Ollama for local LLM inference
curl -fsSL https://ollama.ai/install.sh | sh
ollama pull nomic-embed-text   # embedding model (768 dimensions)
ollama pull llama3             # chat model

# Then start the AI profile:
docker compose --profile ai up -d
```

Without Ollama, the ai-service will not start. All other services function normally. You can also configure an OpenAI key in `.env` instead.

---

### Configuration

All runtime configuration is driven by environment variables. Copy `.env.example` to `.env` and edit before running `make up`.

**Core variables**

| Variable | Default | Description |
|---|---|---|
| `POSTGRES_PASSWORD` | `odin` | Shared password for all Postgres instances |
| `KEYCLOAK_ADMIN` | `admin` | Keycloak admin username |
| `KEYCLOAK_ADMIN_PASSWORD` | `admin` | Keycloak admin password |
| `MINIO_ROOT_USER` | `minio` | MinIO root access key |
| `MINIO_ROOT_PASSWORD` | `minio123` | MinIO root secret key |
| `JWT_SECRET` | — | HS256 secret for dev API key validation (32+ chars) |

**AI variables**

| Variable | Default | Description |
|---|---|---|
| `OLLAMA_BASE_URL` | `http://ollama:11434` | Ollama inference endpoint |
| `OPENAI_API_KEY` | (empty) | If set, OpenAI is used instead of Ollama |
| `AI_CHAT_MODEL` | `llama3` | Ollama model name for chat completions |
| `AI_EMBED_MODEL` | `nomic-embed-text` | Embedding model; must produce 768-dimension vectors |

> **Warning:** The default `.env.example` values are intentionally weak. Change all passwords and secrets before exposing any port to a network.

---

## Architecture

### Overview

ODIN follows Domain-Driven Design with a database-per-service pattern. Six Spring Boot 3.3 microservices communicate via Kafka events. Traefik routes external HTTP traffic.

```
Browser / CLI
    │
    ▼
Traefik (port 80/443)
    ├── catalog.local/          → consumer-frontend (nginx, port 3001)
    ├── manage.catalog.local/   → producer-frontend (nginx, port 3000)
    └── api.catalog.local/      → services (ports 8001–8006)
            │
            ├── catalog-service  :8001  PostgreSQL :5433
            ├── harvest-service  :8002  PostgreSQL :5434 + MinIO :9000
            ├── lineage-service  :8003  PostgreSQL+AGE :5435
            ├── search-service   :8004  OpenSearch :9200
            ├── ai-service       :8005  PostgreSQL+pgvector :5437
            └── identity-service :8006  PostgreSQL :5436 + Keycloak :8180
                    │
                    └─── Apache Kafka :9092 (KRaft, no ZooKeeper)
```

**Design principles:**

- **API-first** — every capability is a versioned REST endpoint before any UI is built on top.
- **Database-per-service** — no service shares a database with another. Cross-service reads go through REST or Kafka events.
- **Event-driven** — state changes publish Kafka events on log-compacted topics. Downstream services maintain their own read models.
- **Standards-based exports** — the catalog exports DCAT 3.0 JSON-LD via Apache Jena; the lineage service accepts OpenLineage JSON.

---

### Services

| Service | Port | Database | Responsibility |
|---|---|---|---|
| **catalog-service** | 8001 | PostgreSQL 16 | DCAT/DPROD/CSV-W metadata, logical models, vocabulary mappings, Kafka event publisher |
| **harvest-service** | 8002 | PostgreSQL 16 + MinIO | Spring Batch crawlers for Snowflake, AWS Glue, Teradata, DCAT HTTP; Quartz scheduler |
| **lineage-service** | 8003 | PostgreSQL + Apache AGE | OpenLineage REST ingestion, DDL parsing via Calcite, Cypher graph queries |
| **search-service** | 8004 | OpenSearch 2.x | Full-text + semantic indexing, FIBO facets, autocomplete suggestions |
| **ai-service** | 8005 | PostgreSQL + pgvector | Spring AI RAG pipeline, embeddings, SSE chat streaming, Ollama / OpenAI |
| **identity-service** | 8006 | PostgreSQL 16 | Keycloak OAuth2/OIDC integration, ABAC policies, API keys, tenant management |

---

### Event Topology

All inter-service communication uses Kafka with an envelope schema that carries tenant, event type, and schema version on every message.

| Topic | Producer | Consumers | Compacted |
|---|---|---|---|
| `catalog.datasets.changes` | catalog-service | search-service, ai-service | Yes |
| `catalog.data-products.changes` | catalog-service | search-service, ai-service | Yes |
| `harvest.entities.discovered` | harvest-service | catalog-service | No |
| `harvest.ddl.discovered` | harvest-service | lineage-service | No |
| `lineage.graph.updated` | lineage-service | search-service | No |

**Event envelope:**

```json
{
  "eventId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "eventType": "DatasetCreated",
  "schemaVersion": "1.0",
  "producerService": "catalog-service",
  "tenantId": "00000000-0000-0000-0000-000000000001",
  "timestamp": "2026-05-18T10:23:00Z",
  "payload": { }
}
```

---

### Security Model

**Authentication methods**

| Method | Header | Use case |
|---|---|---|
| Bearer JWT (OIDC) | `Authorization: Bearer <token>` | User sessions via Keycloak |
| API Key | `X-API-Key: <key>` | Service-to-service, CI pipelines, curl |
| Dev key | `X-API-Key: dev-*` | Local development only — bypasses auth entirely |

**Tenant isolation:** Every resource row carries a `tenant_id` UUID. The `X-Tenant-Id` header (set by Traefik or the frontend nginx) scopes all queries. Rows from other tenants are never returned.

> **Critical:** Never use `X-API-Key: dev-*` in production. It grants unrestricted admin access to all tenants.

---

## Data Model

### Metamodel Overview

ODIN's metamodel has three tiers: conceptual (business), logical (semantic), and physical (technical).

```
Conceptual   DataProduct (DPROD)
               ├── InputPort  → DataService
               └── OutputPort → DataService → Distribution

Logical      Dataset (DCAT)
               ├── VocabularyProfile → Vocabulary (FIBO / schema.org)
               └── LogicalModel
                     └── LogicalDataElement
                           ├── physicalColumnId ──FK──┐
                           └── VocabularyMapping      │ (SKOS)
                                                      │
Physical     Distribution (DCAT)                      │
               └── CSVWTable → CSVWSchema → CSVWColumn ◄┘

Lineage      OpenLineage Job ─[READS_FROM/WRITES_TO]→ Dataset
             (stored in Apache AGE graph, linked to DCAT Dataset)
```

| Layer | Key entities | Purpose |
|---|---|---|
| Conceptual | DataProduct, Port, DataService | Business ownership, governance, lifecycle (Ideation → Consume) |
| Logical | LogicalModel, LogicalDataElement, VocabularyMapping | Business meaning, semantic annotations, vocabulary alignment |
| Physical | Distribution, CSVWTable, CSVWColumn | Technical structure as harvested from source systems |

---

### DCAT Datasets

ODIN models datasets and distributions using [DCAT 3.0](https://www.w3.org/TR/vocab-dcat-3/). The full catalog can be exported as DCAT JSON-LD.

| Field | DCAT property | Type | Notes |
|---|---|---|---|
| `title` | `dct:title` | string | Human-readable name |
| `description` | `dct:description` | string | Free-text description |
| `keywords` | `dcat:keyword` | string[] | Used for search facets |
| `themes` | `dcat:theme` | string[] | Domain classification IRIs |
| `accrualPeriodicity` | `dct:accrualPeriodicity` | string | e.g. `daily`, `hourly` |
| `license` | `dct:license` | URI | License IRI |
| `conformsTo` | `dct:conformsTo` | URI[] | Standards this dataset conforms to |

**DCAT export:**

```bash
curl http://localhost:8001/api/v1/catalogs/{id}/export \
  -H "Accept: application/ld+json" \
  -H "X-API-Key: dev-local"
```

---

### DPROD Data Products

Data products are modelled using the [OMG DPROD standard](https://www.omg.org/spec/DPROD/). A data product represents a business-owned, governed unit of data with a defined lifecycle.

**Lifecycle stages:**

| Stage | Description |
|---|---|
| **Ideation** | Concept identified; no data yet |
| **Design** | Schema and SLA being defined |
| **Build** | Pipeline under development |
| **Deploy** | Running in production, not yet published |
| **Consume** | Publicly available for consumers |

**Create a data product:**

```bash
curl -X POST http://localhost:8001/api/v1/data-products \
  -H "Content-Type: application/json" \
  -H "X-API-Key: dev-local" \
  -d '{
    "title": "Trade Risk Data Product",
    "description": "Aggregated risk metrics for regulatory reporting.",
    "lifecycleStatus": "Consume",
    "keywords": ["risk", "trading", "BCBS239"],
    "informationSensitivity": "Internal"
  }'
```

---

### CSV-W Physical Schema

The physical layer is modelled using [CSV on the Web (CSV-W)](https://www.w3.org/TR/tabular-data-primer/). Each distribution harvested from a source system produces a `CSVWTable` with a `CSVWSchema` containing one `CSVWColumn` per field.

| Field | Type | Description |
|---|---|---|
| `name` | string | Column name as it appears in the source system |
| `titles` | string[] | Alternate names / aliases |
| `datatype` | string | Source system type: `DECIMAL(18,4)`, `VARCHAR(50)`, etc. |
| `required` | boolean | Whether the column is NOT NULL |
| `description` | string | Column comment from the source DDL |
| `propertyUrl` | URI | Linked Data property IRI if available |

Physical columns are created automatically during harvest. A `physicalColumnId` UUID is assigned to each and can be referenced from a `LogicalDataElement` to create the logical–physical binding.

---

### Logical Models

A **LogicalModel** belongs to a Dataset and provides the business-oriented view of its structure. It contains **LogicalDataElements** — each representing a named business concept with an optional binding to a physical column and zero or more vocabulary mappings.

| Field | Type | Description |
|---|---|---|
| `name` | string | Business name: *Trade Amount*, *Settlement Currency* |
| `logicalType` | string | Semantic type: `MonetaryAmount`, `Identifier`, `Date`, `Party` |
| `physicalColumnId` | UUID (nullable) | FK to harvested `csvw_columns` row; null until harvest or manual bind |
| `isIdentifier` | boolean | True if this element forms part of the logical primary key |
| `isNullable` | boolean | Whether the business concept permits absence of a value |

**Bind a physical column:**

```bash
curl -X POST \
  http://localhost:8001/api/v1/logical-data-elements/{elementId}/bind \
  -H "Content-Type: application/json" \
  -H "X-API-Key: dev-local" \
  -d '{ "physicalColumnId": "b3f1a2e4-..." }'
```

**Auto-scaffold from harvest:** When a harvest run discovers columns for a dataset that has no published LogicalModel, ODIN automatically generates a *draft* LogicalModel with one `LogicalDataElement` per `CSVWColumn`. Each element has its `physicalColumnId` pre-bound and its `logicalType` inferred from the source datatype.

---

### Vocabulary & FIBO

ODIN ships with six system vocabularies pre-loaded. Additional RDF vocabularies can be registered at any time.

**Pre-loaded vocabularies:**

| Vocabulary | Prefix | Type | Base IRI |
|---|---|---|---|
| schema.org | `schema` | general | `https://schema.org/` |
| FIBO FND | `fibo-fnd` | financial | `https://spec.edmcouncil.org/fibo/ontology/FND/` |
| FIBO FBC | `fibo-fbc` | financial | `https://spec.edmcouncil.org/fibo/ontology/FBC/` |
| FIBO SEC | `fibo-sec` | financial | `https://spec.edmcouncil.org/fibo/ontology/SEC/` |
| FIBO MD | `fibo-md` | financial | `https://spec.edmcouncil.org/fibo/ontology/MD/` |
| SKOS | `skos` | general | `http://www.w3.org/2004/02/skos/core#` |

**SKOS match types:**

| Match type | When to use |
|---|---|
| `exactMatch` | The element represents precisely the same concept |
| `closeMatch` | Very similar but not identical (e.g. trade date ↔ `schema:startDate`) |
| `relatedMatch` | Related but distinct concepts |
| `broadMatch` | The vocabulary concept is broader / more general |
| `narrowMatch` | The vocabulary concept is narrower / more specific |

**Add a vocabulary mapping:**

```bash
curl -X POST \
  http://localhost:8001/api/v1/logical-data-elements/{elementId}/vocab-mappings \
  -H "Content-Type: application/json" \
  -H "X-API-Key: dev-local" \
  -d '{
    "vocabularyId": "...",
    "conceptIri": "https://spec.edmcouncil.org/fibo/ontology/FND/Accounting/CurrencyAmount/MonetaryAmount",
    "conceptLabel": "MonetaryAmount",
    "matchType": "exactMatch"
  }'
```

---

### OpenLineage

ODIN's lineage-service exposes an OpenLineage-compatible HTTP endpoint. Any tool that emits OpenLineage events (Spark, dbt, Airflow, Flink) can send lineage directly to ODIN.

**Send a lineage event:**

```bash
curl -X POST http://localhost:8003/api/v1/lineage \
  -H "Content-Type: application/json" \
  -H "X-API-Key: dev-local" \
  -d '{
    "eventType": "COMPLETE",
    "eventTime": "2026-05-18T10:00:00Z",
    "producer": "https://github.com/OpenLineage/OpenLineage/tree/main/integration/spark",
    "schemaURL": "https://openlineage.io/spec/1-0-5/OpenLineage.json#/definitions/RunEvent",
    "run": { "runId": "3fa85f64-5717-4562-b3fc-2c963f66afa6" },
    "job": { "namespace": "TRADING_DB", "name": "risk_aggregation_job" },
    "inputs":  [{ "namespace": "TRADING_DB.BLOTTER", "name": "TRADE_BLOTTER" }],
    "outputs": [{ "namespace": "REGULATORY_DB.BCBS239", "name": "RISK_AGGREGATION" }]
  }'
```

**Query the lineage graph:**

```bash
# Upstream lineage, 4 hops
curl "http://localhost:8003/api/v1/datasets/REGULATORY_DB.BCBS239/RISK_AGGREGATION/lineage?direction=upstream&depth=4" \
  -H "X-API-Key: dev-local"

# Downstream impact analysis
curl "http://localhost:8003/api/v1/datasets/TRADING_DB.BLOTTER/TRADE_BLOTTER/impact" \
  -H "X-API-Key: dev-local"
```

Lineage is stored in an Apache AGE property graph on PostgreSQL. Cypher queries traverse `DERIVED_FROM`, `READ_BY`, and `WRITES_TO` edges. Column-level lineage uses `COLUMN_LINEAGE` edges.

---

## Services

### Catalog Service

**Port:** 8001 | **Database:** PostgreSQL 16 (port 5433)

The catalog-service is the primary metadata store. It owns all DCAT, DPROD, CSV-W, logical model, and vocabulary resources. All other services treat it as the source of truth.

**Responsibilities:**
- Persist and version DCAT Datasets, Distributions, DataServices, Catalogs
- Persist DPROD DataProducts, Ports, and lifecycle transitions
- Store CSV-W tables and columns (populated by harvest events)
- Manage LogicalModels and LogicalDataElements with physical column bindings
- Maintain the vocabulary registry and per-dataset vocabulary profiles
- Export the full catalog as DCAT 3.0 JSON-LD via Apache Jena
- Publish `catalog.*.changes` Kafka events on all mutations

**Key tables:** `resources`, `datasets`, `distributions`, `data_products`, `csvw_columns`, `logical_models`, `logical_data_elements`, `vocabularies`, `logical_element_vocab_mappings`

---

### Harvest Service

**Port:** 8002 | **Database:** PostgreSQL 16 (port 5434) + MinIO

The harvest-service crawls external data sources, normalises their metadata, and publishes it to Kafka for the catalog-service to ingest. Jobs are scheduled with Quartz and executed as Spring Batch jobs.

**Supported connectors:**

| Connector | Source type | What it harvests |
|---|---|---|
| `dcat_http` | Any DCAT HTTP endpoint | Datasets, distributions via Apache Jena (JSON-LD, Turtle, RDF/XML) |
| `aws_glue` | AWS Glue Data Catalog | Databases, tables, columns, partitions via AWS SDK v2 |
| `snowflake` | Snowflake | `SHOW TABLES`, `DESCRIBE TABLE`, `GET DDL` |
| `teradata` | Teradata | `DBC.TablesV`, `DBC.ColumnsV`, `DBC.ShowSQL` |

**Configure a source:**

```bash
curl -X POST http://localhost:8002/api/v1/sources \
  -H "Content-Type: application/json" \
  -H "X-API-Key: dev-local" \
  -d '{
    "name": "Snowflake Production",
    "sourceType": "snowflake",
    "baseUrl": "orgname-accountname.snowflakecomputing.com",
    "databaseName": "TRADING_DB",
    "schemaFilter": ["BLOTTER", "RISK"],
    "credentialRef": "vault://snowflake/prod"
  }'
```

---

### Lineage Service

**Port:** 8003 | **Database:** PostgreSQL + Apache AGE (port 5435)

The lineage-service ingests OpenLineage events and DDL, persists them to PostgreSQL, and builds a property graph in Apache AGE for multi-hop Cypher traversal.

**DDL lineage — submit raw DDL without running a pipeline:**

```bash
curl -X POST http://localhost:8003/api/v1/ddl/submit \
  -H "Content-Type: application/json" \
  -H "X-API-Key: dev-local" \
  -d '{
    "dialect": "SNOWFLAKE",
    "ddl": "CREATE VIEW RISK_DB.MARKET_RISK.DAILY_POSITIONS AS SELECT t.*, p.close_price FROM TRADING_DB.BLOTTER.TRADE_BLOTTER t JOIN kafka://prices-realtime p ON t.instrument_id = p.instrument_id"
  }'
```

Apache Calcite parses the DDL across Snowflake, Teradata, and Hive dialects. A `DERIVED_FROM` edge is created in the AGE graph between each source table and the view.

---

### Search Service

**Port:** 8004 | **Database:** OpenSearch 2.x (port 9200)

The search-service maintains an OpenSearch index enriched with logical model data, vocabulary concept labels, and FIBO IRIs. It consumes Kafka events to stay in sync with the catalog.

```bash
# Full-text search with filters
curl "http://localhost:8004/api/v1/search?q=trade&type=dataset&domain=Finance&hasLineage=true" \
  -H "X-API-Key: dev-local"

# FIBO concept facet search
curl "http://localhost:8004/api/v1/search?fibo_concept=MonetaryAmount" \
  -H "X-API-Key: dev-local"

# Autocomplete
curl "http://localhost:8004/api/v1/search/suggest?q=trad" \
  -H "X-API-Key: dev-local"

# Full reindex
curl -X POST http://localhost:8004/api/v1/admin/reindex \
  -H "X-API-Key: dev-local"
```

---

### AI Service

**Port:** 8005 | **Database:** PostgreSQL + pgvector (port 5437)

The ai-service provides a RAG pipeline over your metadata corpus using Spring AI. It can run fully on-premises with Ollama or use the OpenAI API.

> The ai-service is optional. Start it with `docker compose --profile ai up -d ai-service ollama`.

```bash
# Create a conversation
CONV=$(curl -s -X POST http://localhost:8005/api/v1/conversations \
  -H "X-API-Key: dev-local" -H "Content-Type: application/json" \
  -d '{"title": "My session"}' | jq -r .id)

# Ask a question (SSE streaming)
curl -N -X POST http://localhost:8005/api/v1/conversations/$CONV/messages \
  -H "Content-Type: application/json" \
  -H "X-API-Key: dev-local" \
  -H "Accept: text/event-stream" \
  -d '{"content": "Which datasets contain monetary amounts mapped to FIBO?"}'
```

**Embedding pipeline:** The ai-service listens on `catalog.datasets.changes` and `catalog.data-products.changes`. On each event it fetches the enriched entity from catalog-service, chunks the text (title + description + element names + vocabulary labels), embeds the chunks using the configured model, and upserts into the pgvector store.

---

### Identity Service

**Port:** 8006 | **Database:** PostgreSQL 16 (port 5436) + Keycloak (port 8180)

The identity-service manages organisations, users, roles, and access policies. It integrates with Keycloak 24 for OIDC token issuance and validation.

A realm export is included at `infra/keycloak/realm-export.json` and imported automatically on first startup.

> Keycloak 24 uses `KEYCLOAK_ADMIN` and `KEYCLOAK_ADMIN_PASSWORD` environment variables. The old `KC_BOOTSTRAP_ADMIN_*` variables are not supported.

---

## API Reference

### Authentication

| Header | Required | Description |
|---|---|---|
| `Authorization: Bearer <jwt>` | One of these two | Keycloak OIDC access token |
| `X-API-Key: <key>` | One of these two | API key from identity-service |
| `X-Tenant-Id: <uuid>` | Yes | Tenant scoping — set by Traefik/nginx in production |

Any key starting with `dev-` bypasses authentication and grants `catalog:admin` scope:

```bash
# These are all equivalent for local development:
curl -H "X-API-Key: dev-anything" ...
curl -H "X-API-Key: dev-local" ...
```

---

### Catalog API

**Base URL:** `http://localhost:8001`

**Datasets**

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/v1/datasets` | List datasets. Params: `page`, `size`, `domain`, `format` |
| `POST` | `/api/v1/datasets` | Create a new dataset |
| `GET` | `/api/v1/datasets/{id}` | Get dataset by ID |
| `PATCH` | `/api/v1/datasets/{id}` | Partial update |
| `DELETE` | `/api/v1/datasets/{id}` | Soft-delete |

**Data Products**

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/v1/data-products` | List. Params: `lifecycleStatus`, `domain` |
| `POST` | `/api/v1/data-products` | Create a data product |
| `PATCH` | `/api/v1/data-products/{id}/lifecycle` | Transition lifecycle: `{"status":"Deploy"}` |

**Logical Models**

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/v1/datasets/{id}/logical-models` | List logical models for a dataset |
| `POST` | `/api/v1/datasets/{id}/logical-models` | Create a logical model |
| `GET` | `/api/v1/logical-models/{id}/elements` | List data elements |
| `POST` | `/api/v1/logical-data-elements/{id}/bind` | Bind element to a physical column |
| `POST` | `/api/v1/logical-data-elements/{id}/vocab-mappings` | Add a SKOS vocabulary mapping |

**Vocabularies**

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/v1/vocabularies` | List all registered vocabularies |
| `GET` | `/api/v1/vocabularies/{id}/concepts/search` | Search concepts. Param: `q=price&limit=20` |
| `GET` | `/api/v1/catalogs/{id}/export` | Export as DCAT JSON-LD (`Accept: application/ld+json`) |

---

### Harvest API

**Base URL:** `http://localhost:8002`

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/v1/sources` | List all harvest sources. Filter by `type` |
| `POST` | `/api/v1/sources` | Register a new source |
| `POST` | `/api/v1/sources/{id}/test` | Test connectivity |
| `GET` | `/api/v1/jobs` | List harvest jobs |
| `POST` | `/api/v1/jobs` | Create a scheduled or on-demand job |
| `POST` | `/api/v1/jobs/{id}/trigger` | Trigger an immediate harvest run |
| `POST` | `/api/v1/jobs/{id}/cancel` | Cancel a running job |
| `GET` | `/api/v1/runs/{id}/items` | Inspect per-entity results of a run |

---

### Lineage API

**Base URL:** `http://localhost:8003`

| Method | Path | Description |
|---|---|---|
| `POST` | `/api/v1/lineage` | Ingest an OpenLineage RunEvent |
| `GET` | `/api/v1/datasets/{ns}/{name}/lineage` | Graph traversal. Params: `direction`, `depth` |
| `GET` | `/api/v1/datasets/{ns}/{name}/column-lineage` | Column-level lineage |
| `GET` | `/api/v1/datasets/{ns}/{name}/impact` | Downstream impact analysis |
| `POST` | `/api/v1/ddl/submit` | Submit DDL for Calcite parsing |
| `GET` | `/api/v1/jobs/{ns}/{name}/runs` | Run history. Filter by `state`, `start`, `end` |

---

### Search API

**Base URL:** `http://localhost:8004`

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/v1/search` | Full-text search. Params: `q`, `type`, `domain`, `lifecycleStatus`, `format`, `hasLineage`, `fibo_concept`, `page`, `size` |
| `GET` | `/api/v1/search/suggest` | Autocomplete. Param: `q`. Returns up to 10 suggestions |
| `POST` | `/api/v1/search/saved` | Save a search query |
| `POST` | `/api/v1/admin/reindex` | Trigger full reindex from catalog-service |

---

### AI API

**Base URL:** `http://localhost:8005`

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/v1/conversations` | List conversations for the current user |
| `POST` | `/api/v1/conversations` | Create a new conversation |
| `POST` | `/api/v1/conversations/{id}/messages` | Send a message. Set `Accept: text/event-stream` for SSE |
| `POST` | `/api/v1/semantic-search` | Vector similarity search |
| `POST` | `/api/v1/admin/embeddings/refresh` | Re-embed all documents |

---

## Deployment

### Docker Compose

The repository ships a `docker-compose.yml` for the full stack and a `docker-compose.override.yml` for development hot-reload overrides.

**Makefile targets:**

| Target | Description |
|---|---|
| `make up` | Start all services in detached mode |
| `make down` | Stop and remove containers (preserves volumes) |
| `make destroy` | Stop containers and delete all volumes |
| `make migrate` | Run Flyway migrations manually |
| `make seed` | Load financial services sample data |
| `make reindex` | Full OpenSearch reindex |
| `make build` | Build all Docker images from source |
| `make test` | Run all service tests in Docker |
| `make logs svc=catalog-service` | Tail logs for a specific service |

**Profiles:**

| Profile | Additional services |
|---|---|
| (default) | All 6 services + Kafka + PostgreSQL × 4 + OpenSearch + MinIO + Keycloak |
| `ai` | Adds ai-service and Ollama |

```bash
# Start including AI features
docker compose --profile ai up -d
```

**Rebuilding a single image:**

```bash
# Always build then up — `restart` reuses the old image
docker compose build catalog-service
docker compose up -d catalog-service
```

---

### Environment Variables

| Variable | Service | Default | Description |
|---|---|---|---|
| `POSTGRES_PASSWORD` | All | `odin` | PostgreSQL password (shared) |
| `KEYCLOAK_ADMIN` | identity | `admin` | Keycloak admin username |
| `KEYCLOAK_ADMIN_PASSWORD` | identity | `admin` | Keycloak admin password |
| `JWT_SECRET` | All | — | HS256 signing secret for dev API keys |
| `MINIO_ROOT_USER` | harvest | `minio` | MinIO access key |
| `MINIO_ROOT_PASSWORD` | harvest | `minio123` | MinIO secret key |
| `OPENSEARCH_PASSWORD` | search | `admin` | OpenSearch admin password |
| `OLLAMA_BASE_URL` | ai | `http://ollama:11434` | Ollama base URL |
| `OPENAI_API_KEY` | ai | (empty) | OpenAI key; takes precedence over Ollama if set |
| `AI_CHAT_MODEL` | ai | `llama3` | Chat model name |
| `AI_EMBED_MODEL` | ai | `nomic-embed-text` | Embedding model (must be 768-dim) |
| `SNOWFLAKE_ACCOUNT` | harvest | — | Snowflake account identifier |
| `AWS_ACCESS_KEY_ID` | harvest | — | AWS credentials for Glue connector |
| `AWS_SECRET_ACCESS_KEY` | harvest | — | AWS credentials for Glue connector |
| `AWS_REGION` | harvest | `us-east-1` | AWS region for Glue |

---

### Kubernetes (Helm)

Helm charts are provided under `infra/helm/charts/` for each service. A top-level umbrella chart is planned for GA.

> **Warning:** Kubernetes deployment is in early stages. The Helm charts are functional but not hardened for production. Use Docker Compose for evaluation and development.

```bash
helm install odin-catalog infra/helm/charts/catalog-service \
  --namespace odin --create-namespace \
  --set postgresql.password=changeme \
  --set kafka.brokers=kafka:9092
```

---

## Contributing

### Local Development

**Build all services:**

```bash
./gradlew build                              # compile + test all services
./gradlew :services:catalog-service:bootRun  # run one service locally
```

**Build the frontends:**

```bash
cd frontend/shared && pnpm install && pnpm build
cd ../producer  && pnpm install && pnpm dev   # http://localhost:3000
cd ../consumer  && pnpm install && pnpm dev   # http://localhost:3001
```

---

### Contribution Guide

**Before you start:**
- Open an issue describing the bug or feature before submitting a PR.
- One logical change per PR — keep diffs reviewable.
- All new API endpoints need an integration test that runs against a real database (no mocks).

**Code conventions:**
- **Java** — hexagonal architecture. Domain classes have no Spring annotations; all infrastructure concerns live in the `infrastructure/` package.
- **TypeScript** — no `any`; shared API types live in `frontend/shared/src/types/`.
- **SQL** — all schema changes via numbered Flyway migrations (`V{n}__description.sql`).

**License:** ODIN Catalog is released under the **Apache 2.0 License**. By contributing you agree your changes will be licensed under the same terms.
