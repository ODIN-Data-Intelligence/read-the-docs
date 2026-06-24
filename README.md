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
  - [Vocabulary & AI](#vocabulary--ai)
  - [Terms of Use (ODRL)](#terms-of-use-odrl)
  - [OpenLineage](#openlineage)
- [Capabilities Matrix](capabilities.md) — full feature inventory with status (available / partial / planned)
- [Database ERDs](erd.md) — full schema diagrams for all six databases
- [Services](#services-1)
  - [Inventory Service](#inventory-service)
  - [Harvest Service](#harvest-service)
  - [Lineage Service](#lineage-service)
  - [Search Service](#search-service)
  - [AI Service](#ai-service)
  - [Identity Service](#identity-service)
  - [Policy Service](#policy-service)
- [Authentication & User Management](#authentication--user-management)
  - [Roles](#roles)
  - [Default Users](#default-users)
  - [Inviting Users](#inviting-users)
  - [Keycloak Configuration](#keycloak-configuration)
- [API Reference](#api-reference)
  - [Authentication](#authentication)
  - [Inventory API](#inventory-api)
  - [Harvest API](#harvest-api)
  - [Lineage API](#lineage-api)
  - [Search API](#search-api)
  - [AI API](#ai-api)
  - [Policy API](#policy-api)
- [Deployment](#deployment)
  - [Docker Compose](#docker-compose)
  - [Environment Variables](#environment-variables)
  - [Kubernetes (Raw Manifests)](#kubernetes-raw-manifests)
  - [Kubernetes (Helm / MicroK8s)](#kubernetes-helm--microk8s)
- [Security Hardening](#security-hardening)
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
- **AI metadata enrichment** — per-element classification, description, vocabulary concept, and PII/direct-identifier recommendations, all owner-reviewed before acceptance. Vocabulary concept chips are individually selectable; PII and direct-identifier flags are AI-detected using W3C DPV-PD vocabulary IRIs and field name heuristics.
- **FCRA-aware policy derivation** — when any published logical model element is flagged as personal information or a direct identifier, the `HAS_PII_ELEMENTS` signal fires and ODRL terms are automatically strengthened with POLICY_STRICT rules (restricted distribution, audit obligation).
- **ODRL terms of use + ODRE enforcement** — access-level policy (OPEN → HIGHLY\_RESTRICTED) derived automatically from element classifications and vocabulary concepts; the dedicated policy-service evaluates ODRL policies at request time using the ODRE enforcement algorithm, returning machine-readable access decisions.
- **Accountable data ownership** — role-based dataset ownership with a proposal-and-approval transfer workflow and a full audit history. The governance dashboard surfaces pending tasks and an activity feed for every user.
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
| **DPV** | W3C | Data Privacy Vocabulary — personal data categories, processing purposes, and legal bases. DPV-PD concept IRIs drive AI-based PII and direct-identifier detection on logical model elements |
| **ODRL** | W3C | Terms of use policies — permissions, prohibitions, obligations derived from element classifications and vocabulary concepts |
| **ODRE** | Academic (Cimmino et al., 2025) | Open Digital Rights Enforcement — Algorithm 1 evaluation engine; policy-service implements A-Level (static ODRL) and B1-Level (runtime variable injection) |

> **Alpha notice:** ODIN is currently in private alpha. APIs and database schemas may change between releases. Not recommended for production workloads yet.

---

### Quick Start

Get a full ODIN stack running locally in under five minutes using Docker Compose.

**1. Clone and configure**

```bash
git clone https://github.com/ODIN-Data-Intelligence/odin.git
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

ODIN follows Domain-Driven Design with a database-per-service pattern. Seven Spring Boot 3.3 microservices communicate via Kafka events. Traefik routes external HTTP traffic.

```
Browser / CLI
    │
    ▼
Traefik (port 80/443)
    ├── catalog.local/          → consumer-frontend (nginx, port 3001)
    ├── manage.catalog.local/   → producer-frontend (nginx, port 3000)
    └── api.catalog.local/      → services (ports 8001–8007)
            │
            ├── inventory-service  :8001  PostgreSQL :5433
            ├── harvest-service    :8002  PostgreSQL :5434 + MinIO :9000
            ├── lineage-service    :8003  PostgreSQL+AGE :5435
            ├── search-service     :8004  OpenSearch :9200
            ├── ai-service         :8005  PostgreSQL+pgvector :5437
            ├── identity-service   :8006  PostgreSQL :5436 + Keycloak :8180
            └── policy-service     :8007  PostgreSQL :5438
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
| **inventory-service** | 8001 | PostgreSQL 16 | DCAT/DPROD/CSV-W metadata, logical models, vocabulary mappings, Kafka event publisher |
| **harvest-service** | 8002 | PostgreSQL 16 + MinIO | Spring Batch crawlers for Snowflake, AWS Glue, Teradata, DCAT HTTP; Quartz scheduler |
| **lineage-service** | 8003 | PostgreSQL + Apache AGE | OpenLineage REST ingestion, DDL parsing via Calcite, Cypher graph queries |
| **search-service** | 8004 | OpenSearch 2.x | Full-text + semantic indexing, FIBO facets, autocomplete suggestions |
| **ai-service** | 8005 | PostgreSQL + pgvector | Spring AI RAG pipeline, embeddings, SSE chat streaming, agentic proposer/reviewer review, Ollama / OpenAI |
| **identity-service** | 8006 | PostgreSQL 16 | Keycloak OAuth2/OIDC, role-based access (Administrator, Data Owner, Steward, Governance), user provisioning with Keycloak sync, API keys, tenant management |
| **policy-service** | 8007 | PostgreSQL 16 | ODRL policy registry and ODRE enforcement engine (PDP). Evaluates A-Level and B1-Level ODRL policies at request time; syncs policies from dataset change events via Kafka; persists evaluation log. |

---

### Event Topology

All inter-service communication uses Kafka with an envelope schema that carries tenant, event type, and schema version on every message.

| Topic | Producer | Consumers | Compacted |
|---|---|---|---|
| `inventory.datasets.changes` | inventory-service | search-service, ai-service, policy-service | Yes |
| `inventory.data-products.changes` | inventory-service | search-service, ai-service | Yes |
| `harvest.entities.discovered` | harvest-service | inventory-service | No |
| `harvest.ddl.discovered` | harvest-service | lineage-service | No |
| `lineage.graph.updated` | lineage-service | search-service | No |
| `policy.records.changes` | policy-service | — (reserved for future PEPs) | No |
| `policy.evaluations.completed` | policy-service | API gateway / consumer (planned) | No |

**Event envelope:**

```json
{
  "eventId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "eventType": "DatasetCreated",
  "schemaVersion": "1.0",
  "producerService": "inventory-service",
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
| Bearer JWT (OIDC) | `Authorization: Bearer <token>` | User sessions via Keycloak (producer UI, consumer UI) |
| API Key | `X-API-Key: <key>` | Service-to-service calls, CI pipelines, curl |
| Dev key | `X-API-Key: dev-*` | Local development only — bypasses auth entirely |

**Tenant isolation:** Every resource row carries a `tenant_id` UUID extracted from the JWT `tenant_id` claim. Rows from other tenants are never returned. The `X-Tenant-Id` header is no longer injected by nginx — the tenant is carried in the JWT itself.

> **Critical:** Never use `X-API-Key: dev-*` in production. It grants unrestricted admin access to all tenants.

See the [Authentication & User Management](#authentication--user-management) section for role definitions, user setup, and Keycloak configuration.

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
                           └── VocabularyMapping (SKOS)
                                   ▲
Physical     Distribution (DCAT)  │ logicalDataElementId FK
               └── CSVWTable → CSVWSchema → CSVWColumn ──────────────┘

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

Physical columns are created automatically during harvest. Each `CSVWColumn` carries an optional `logicalDataElementId` FK that, when set, creates the logical–physical binding. A single `LogicalDataElement` may be bound by multiple physical columns across different distributions or schema versions.

---

### Logical Models

A **LogicalModel** belongs to a Dataset and provides the business-oriented view of its structure. It contains **LogicalDataElements** — each representing a named business concept with an optional binding to a physical column and zero or more vocabulary mappings.

| Field | Type | Description |
|---|---|---|
| `name` | string | Technical element name (from harvest or manual entry) |
| `label` | string | Human-readable business name shown in the Model tab: *Trade Amount*, *Settlement Currency* |
| `description` | string | Plain-English business description, curated by a steward or accepted from an AI recommendation |
| `logicalType` | string | Semantic type: `MonetaryAmount`, `Identifier`, `Date`, `Party` |
| `classification` | string | Accepted data sensitivity level: `PUBLIC`, `INTERNAL`, `CONFIDENTIAL`, `HIGH_CONFIDENTIAL` |
| `recommendedClassification` | string | AI-suggested classification pending data owner review; cleared on accept/reject |
| `classificationReasoning` | string | One-sentence rationale produced alongside the AI classification recommendation |
| `physicalColumnIds` | UUID[] | IDs of bound `csvw_columns` rows (FK lives on the column side) |
| `isIdentifier` | boolean | True if this element forms part of the logical primary key |
| `isNullable` | boolean | Whether the business concept permits absence of a value |
| `isPersonalInformation` | boolean | True if this element holds personal data about a natural person (DPV-PD category match or field name heuristic) |
| `isDirectIdentifier` | boolean | True if this element directly identifies a natural person (name, email, SSN, LEI for sole traders, etc.) |
| `recommendedIsPersonalInformation` | boolean? | AI-suggested value pending data owner review; cleared on accept/reject |
| `recommendedIsDirectIdentifier` | boolean? | AI-suggested value pending data owner review; cleared on accept/reject |
| `piiRecommendationReasoning` | string? | One-sentence rationale for the AI PII/identifier recommendation |
| `piiRecommendedAt` | timestamp? | When the current pending PII recommendation was generated |

**Bind a physical column:**

```bash
curl -X POST \
  http://localhost:8001/api/v1/logical-data-elements/{elementId}/bind \
  -H "Content-Type: application/json" \
  -H "X-API-Key: dev-local" \
  -d '{ "physicalColumnId": "b3f1a2e4-..." }'
```

**AI description recommendation:** For each element the AI service can generate a plain-English business description grounded in the element's name, label, logical type, and vocabulary mappings. The result is stored in `recommendedDescription` and surfaced as an inline suggestion in the Model tab — a steward accepts or dismisses it. Accepting writes the text to `description` and clears `recommendedDescription`.

```bash
# Request a description recommendation for one element
curl -s -X POST \
  http://localhost:8001/api/v1/logical-data-elements/{elementId}/recommend-description \
  -H "X-API-Key: dev-local"

# Accept the recommendation
curl -s -X POST \
  http://localhost:8001/api/v1/logical-data-elements/{elementId}/accept-description \
  -H "X-API-Key: dev-local"

# Bulk — all elements in a model
curl -s -X POST \
  http://localhost:8001/api/v1/logical-models/{modelId}/recommend-descriptions \
  -H "X-API-Key: dev-local"
```

**AI PII and direct-identifier recommendation:** For each element the AI service detects whether the element holds personal data (`isPersonalInformation`) or directly identifies a natural person (`isDirectIdentifier`). Detection uses DPV-PD and schema.org vocabulary concept IRIs already on the element, plus field name and dataset keyword heuristics. Results are stored as pending `recommendedIs*` fields and surfaced in the Model tab for data owner review.

```bash
# Request PII/identifier recommendation for one element
curl -s -X POST \
  http://localhost:8001/api/v1/logical-data-elements/{elementId}/recommend-pii \
  -H "X-API-Key: dev-local"

# Accept the recommendation
curl -s -X POST \
  http://localhost:8001/api/v1/logical-data-elements/{elementId}/accept-pii \
  -H "X-API-Key: dev-local"

# Reject the recommendation
curl -s -X POST \
  http://localhost:8001/api/v1/logical-data-elements/{elementId}/reject-pii \
  -H "X-API-Key: dev-local"

# Bulk — all elements in a model
curl -s -X POST \
  http://localhost:8001/api/v1/logical-models/{modelId}/recommend-pii \
  -H "X-API-Key: dev-local"
# → {"jobId": "...", "status": "RUNNING"}

# Poll job status
curl -s http://localhost:8001/api/v1/logical-models/recommend-pii/jobs/{jobId} \
  -H "X-API-Key: dev-local"
```

**Auto-scaffold from harvest:** When a harvest run discovers columns for a dataset that has no published LogicalModel, ODIN automatically generates a *draft* LogicalModel with one `LogicalDataElement` per `CSVWColumn`. Each harvested column has its `logicalDataElementId` set to the newly created element, and its `logicalType` is inferred from the source datatype.

---

### Vocabulary & FIBO

ODIN ships with seven system vocabularies pre-loaded. Additional RDF vocabularies can be registered at any time.

**Pre-loaded vocabularies:**

| Vocabulary | Prefix | Type | Base IRI |
|---|---|---|---|
| schema.org | `schema` | general | `https://schema.org/` |
| FIBO FND | `fibo-fnd` | financial | `https://spec.edmcouncil.org/fibo/ontology/FND/` |
| FIBO FBC | `fibo-fbc` | financial | `https://spec.edmcouncil.org/fibo/ontology/FBC/` |
| FIBO SEC | `fibo-sec` | financial | `https://spec.edmcouncil.org/fibo/ontology/SEC/` |
| FIBO MD | `fibo-md` | financial | `https://spec.edmcouncil.org/fibo/ontology/MD/` |
| SKOS | `skos` | general | `http://www.w3.org/2004/02/skos/core#` |
| DPV / DPV-PD | `dpv` | privacy | `https://w3id.org/dpv#` / `https://w3id.org/dpv/pd#` |

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

### Semantic Types

A **semantic type** is the terminal fragment of a vocabulary concept IRI, extracted automatically from `exactMatch` and `closeMatch` mappings on published logical models. Semantic types provide a lightweight, human-readable classification at the dataset level — without duplicating the raw IRI.

| Vocabulary concept IRI | Semantic type |
|---|---|
| `.../FBC/ProductsAndServices/ClientsAndAccounts/Customer` | `Customer` |
| `.../FBC/ProductsAndServices/ClientsAndAccounts/DebitCardAccount` | `DebitCardAccount` |
| `https://schema.org/LoanOrCredit` | `LoanOrCredit` |

The `GET /api/v1/datasets/{id}/semantic-context` endpoint returns:

| Field | Description |
|---|---|
| `semanticTypes` | IRI fragments from exactMatch/closeMatch (published models only) |
| `vocabConceptLabels` | All concept labels across all elements |
| `vocabConceptIris` | All concept IRIs |
| `fiboConcepts` | Subset of IRIs containing `edmcouncil.org/fibo` |
| `logicalElementNames` | Business names of all logical elements |
| `logicalTypes` | Logical type values (MonetaryAmount, Date, …) |
| `acceptedTags` | AI-recommended semantic tags accepted and persisted by a steward |

**AI recommendations:** `POST /api/v1/datasets/{id}/recommend-semantic-context` sends the full semantic context to the LLM and returns typed `RecommendedSemanticType` objects with a rationale and optional vocabulary hint. Recommendations can be accepted (persisting a `DatasetSemanticTag`) or dismissed.

```bash
# Get AI recommendations for a dataset
curl -X POST http://localhost:8001/api/v1/datasets/{id}/recommend-semantic-context \
  -H "X-API-Key: dev-local"

# Accept a recommendation
curl -X POST http://localhost:8001/api/v1/datasets/{id}/semantic-tags \
  -H "Content-Type: application/json" -H "X-API-Key: dev-local" \
  -d '{"type": "Customer", "vocabularyIri": "https://spec.edmcouncil.org/fibo/.../Customer"}'
```

---

### Vocabulary & AI

Semantic vocabularies are the missing layer between your data and AI. Standard concept IRIs — schema.org and FIBO — transform how language models reason over catalog metadata.

**Ambiguity is the root cause of AI failure.** RAG pipelines retrieve chunks of text. Without semantic grounding, a question about "settlement amount" returns every table that mentions the word "amount." A SKOS `exactMatch` binding to `fibo-fnd-acc-cur:MonetaryAmount` makes retrieval precise — the model finds the right element, not the most popular one.

**Standard IRIs are native to foundation models.** schema.org and FIBO IRIs appear extensively in the training corpora of every major LLM. Annotating a data element with `https://schema.org/price` or `fibo-md-temx-ex:MarketPrice` puts it in semantic proximity to everything the model already knows about that concept — zero prompt engineering required.

**Agents need contracts, not descriptions.** A vocabulary mapping is a contract: this column contains a `LegalEntityIdentifier`, not "some kind of ID." Agents that operate on contracts are auditable; agents that operate on descriptions are not.

**Your metadata becomes a knowledge graph.** ODIN's vocabulary mappings, logical models, and lineage edges form a traversable knowledge graph stored in Apache AGE. From a regulatory report, upstream through lineage to source systems, sideways through vocabulary to equivalent concepts in other datasets.

**FIBO: regulatory-grade semantics, pre-loaded.**

| FIBO concept IRI (abbreviated) | Meaning |
|---|---|
| `fibo-fnd-acc-cur:MonetaryAmount` | Monetary value with attached currency |
| `fibo-fnd-acc-cur:Currency` | ISO 4217 currency code |
| `fibo-fbc-fi-fi:FinancialInstrument` | General financial instrument |
| `fibo-fbc-fct-rga:LegalEntityIdentifier` | LEI — unique legal entity identifier |
| `fibo-md-temx-ex:MarketPrice` | Exchange-quoted market price |
| `fibo-sec-eq-eq:Share` | Equity share / stock |

**Cross-system equivalence without ETL.** Different source systems use different column names for the same concept: `trade_ccy`, `SETTL_CURR`, `SettlementCurrency`. All three mapped to `fibo-fnd-acc-cur:Currency` with `exactMatch` become interchangeable to any AI agent — without moving a byte of data. Find all matching datasets via the search API:

```bash
curl "http://localhost:8004/api/v1/search?fibo_concept=fibo-fnd-acc-cur%3ACurrency" \
  -H "X-API-Key: dev-local"
```

---

### Terms of Use (ODRL)

ODIN derives an [ODRL](https://www.w3.org/TR/odrl-model/) terms-of-use policy for every dataset automatically — no manual authoring required. The policy is inferred from two signals already present in the catalog:

1. **Element classifications** — the most restrictive accepted `classification` across all published logical model elements determines the access level.
2. **Vocabulary concept mappings** — FIBO namespace prefixes (`fibo-fbc`, `fibo-sec`, `fibo-md`) and dataset keywords (`mifid`, `emir`, `gdpr`, `basel`, `finrep`, `fcra`) identify applicable regulatory frameworks.
3. **PII element signals** — when any published element has `isPersonalInformation = true` or `isDirectIdentifier = true`, the `HAS_PII_ELEMENTS` signal fires. This triggers FCRA-style POLICY_STRICT rules: the derived ODRL policy gains a prohibition on distribution without explicit consent and an audit obligation, strengthening the base access level rules.

**Access levels:**

| Effective classification | Access level | Typical restrictions |
|---|---|---|
| `PUBLIC` | OPEN | Redistribute with attribution |
| `INTERNAL` | INTERNAL_ONLY | No external distribution |
| `CONFIDENTIAL` | RESTRICTED | Analytics only; notify data owner before AI/ML use |
| `HIGH_CONFIDENTIAL` | HIGHLY_RESTRICTED | Explicit approval required; full audit trail |

**Retrieve derived terms:**

```bash
curl http://localhost:8001/api/v1/datasets/{id}/terms-of-use \
  -H "X-API-Key: dev-local"
```

The response includes `effectiveClassification`, `accessLevel`, `permissions`, `prohibitions`, `obligations`, `applicableRegulations`, `odrlPolicy` (full ODRL JSON), `policySource` (`derived` / `explicit` / `fallback`), and `derivationDetails` (element counts, matched signals).

**Accept and lock a policy (data owner only):**

```bash
# Lock the derived policy as the official terms for this dataset
curl -X POST http://localhost:8001/api/v1/datasets/{id}/terms-of-use/accept \
  -H "X-API-Key: dev-local"

# Reset back to dynamic derivation
curl -X DELETE http://localhost:8001/api/v1/datasets/{id}/terms-of-use/policy \
  -H "X-API-Key: dev-local"
```

**Accept pre-condition:** The "Accept Policy" button in the producer Governance tab is enabled only when every element in the published logical model has both an accepted `classification` and at least one vocabulary concept mapping. ODIN shows per-element readiness hints while conditions are unmet.

**Consumer visibility:** The consumer discovery UI exposes a **Terms** tab on every dataset drawer, showing the access level badge, rule sections, applicable regulations as pills, and a collapsible ODRL JSON block for technical consumers.

#### ODRE Policy Enforcement (policy-service)

The **policy-service** (port 8007) is the platform's Policy Decision Point (PDP). It implements ODRE Algorithm 1 (Cimmino et al., *Computers & Security*, 2025) — a concrete enforcement layer on top of ODRL that produces machine-readable `UsageDecision` tuples at request time.

**Policy levels supported:**

| Level | Description |
|---|---|
| A-Level | Pure ODRL JSON-LD. Static constraint evaluation — dateTime, numeric, string comparisons. All policies generated by `TermsOfUseService` are A-Level. |
| B1-Level | Variable injection: `[=varName]` placeholders resolved from the request `M` map at evaluation time. Use to inject caller role, caller ID, or any runtime value without embedding it in the stored policy. |

**Register a policy (upsert):**

```bash
curl -X PUT http://localhost:8007/api/v1/policies/{datasetId} \
  -H "X-API-Key: dev-local" -H "Content-Type: application/json" \
  -d '{
    "policyJson": "{\"@context\":\"http://www.w3.org/ns/odrl.jsonld\",\"@type\":\"Set\",\"uid\":\"urn:example\",\"permission\":[{\"target\":\"dataset:x\",\"action\":\"read\"}]}",
    "policyLevel": "A"
  }'
```

**Evaluate access (PDP call):**

```bash
curl -X POST http://localhost:8007/api/v1/policies/{datasetId}/evaluate \
  -H "X-API-Key: dev-local" -H "Content-Type: application/json" \
  -d '{"M": {"callerRole": "DATA_OWNER"}, "F": {}}'
# → {"granted": true, "policyLevel": "A", "decisions": [{"action":"read","result":"true","delegated":false}]}
```

The `granted` field is `true` when the decision set contains a `(read, "true")` tuple (non-delegated permission). An empty decision set means all constraints failed — access denied.

**Kafka sync:** policy-service subscribes to `inventory.datasets.changes`. When a dataset event carries a non-null `hasPolicy` field (set after a data owner accepts terms), the policy is automatically upserted into the policy registry — no manual `PUT` required.

**Evaluation log:** Every `POST /evaluate` call is persisted to `evaluation_log` and a `PolicyEvaluationResultPayload` event is published to `policy.evaluations.completed` for downstream enforcement points. Retrieve recent decisions with `GET /api/v1/policies/{datasetId}/evaluations`.

**Policy composition (component pieces):** A dataset's effective policy is not a single hand-written document — it is assembled from reusable, typed fragments called **policy pieces**. Each piece has a `pieceType` of `CLASSIFICATION` (derived from the data's sensitivity), `REGULATION` (e.g. FCRA-aware rules), or `CONTRACTUAL` (terms-of-use obligations), keyed by a dimension value. `dataset_policy_links` records which pieces apply to which dataset, and the registry composes them into the effective `policy_records` document. Inspect the breakdown with:

```bash
curl http://localhost:8007/api/v1/policies/{datasetId}/components \
  -H "X-API-Key: dev-local"
# → { "components": { "CLASSIFICATION": {...}, "REGULATION": {...}, "CONTRACTUAL": {...} },
#     "assembled": { ...effective ODRL policy... } }
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
# Step 1 — resolve namespace/name to a lineage UUID
LINEAGE_ID=$(curl -s "http://localhost:8003/api/v1/datasets/lookup?namespace=REGULATORY_DB.BCBS239&name=RISK_AGGREGATION" \
  -H "X-API-Key: dev-local" | jq -r .id)

# Step 2 — upstream lineage, 4 hops (using lineage UUID)
curl "http://localhost:8003/api/v1/datasets/$LINEAGE_ID/lineage?direction=upstream&depth=4" \
  -H "X-API-Key: dev-local"

# Downstream impact analysis
curl "http://localhost:8003/api/v1/datasets/$LINEAGE_ID/impact" \
  -H "X-API-Key: dev-local"
```

Lineage is stored in an Apache AGE property graph on PostgreSQL. Cypher queries traverse `DERIVED_FROM`, `READ_BY`, and `WRITES_TO` edges. Column-level lineage uses `COLUMN_LINEAGE` edges.

---

### Governance & Audit

Every dataset has an optional **data owner** and an immutable **audit history**. Ownership only changes hands through a proposal workflow, so a transfer is never unilateral.

**Assigning an owner:** `PUT /api/v1/datasets/{id}/owner` with `{"userId":"..."}`. Rejected with `409` if the dataset already has an owner — use the transfer workflow instead.

**Ownership transfer workflow:**
1. **Propose** — `POST /api/v1/datasets/{id}/ownership-proposals` with `{"proposedOwnerId":"..."}` creates a `PENDING` proposal.
2. **Approve / reject** — only the current owner (or a catalog admin) may call `.../ownership-proposals/{proposalId}/approve` or `.../reject`.
3. **Pending queue** — `GET .../ownership-proposals/pending` returns the open proposal, or `204` if none.

**AI action gating:** Accepting or rejecting AI-generated recommendations is restricted to the dataset's data owner:

| Action | Who can perform it |
|---|---|
| Accept / reject AI classification | Data Owner, Administrator |
| Accept / reject AI description | Data Owner, Administrator |
| Accept / reject AI vocabulary concept recommendations | Data Owner, Administrator |
| Accept / reset ODRL terms of use policy | Data Owner only |
| Accept physical schema AI mappings | Data Owner only |
| Approve / reject ownership transfer | Current Data Owner, Administrator |
| Direct owner assignment (unowned dataset) | Administrator |
| Propose ownership transfer | Data Governance Officer, Data Steward, current Data Owner |

**Audit history:** `GET /api/v1/datasets/{id}/history` returns a paginated, reverse-chronological log. Tracked events: `CREATED`, `UPDATED`, `DELETED`, `OWNER_ASSIGNED`, `OWNER_TRANSFER_PROPOSED`, `OWNER_TRANSFER_APPROVED`, `OWNER_TRANSFER_REJECTED`.

**Governance dashboard:** The producer UI dashboard provides a personal governance view — stat cards (owned datasets and data products), Outstanding Tasks (pending transfers directed at the user), My Proposals (proposals submitted or nominated for), and My Changes (a chronological event feed). Served by `GET /api/v1/dashboard/summary` and `GET /api/v1/dashboard/activity` on the inventory-service.

---

## Services

### Inventory Service

**Port:** 8001 | **Database:** PostgreSQL 16 (port 5433)

The inventory-service is the primary metadata store. It owns all DCAT, DPROD, CSV-W, logical model, and vocabulary resources. All other services treat it as the source of truth.

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

The harvest-service crawls external data sources, normalises their metadata, and publishes it to Kafka for the inventory-service to ingest. Jobs are scheduled with Quartz and executed as Spring Batch jobs.

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

**Embedding pipeline:** The ai-service listens on `inventory.datasets.changes` and `inventory.data-products.changes`. On each event it produces three chunks per dataset:

| Chunk | Content |
|---|---|
| 0 | Title + description + keywords + physical column names |
| 1 | Logical element names + vocab concept labels (business meaning layer) |
| 2 | Semantic types + vocab concept labels + logical element names (type layer — enables "which datasets contain Customer data?" queries) |

All three chunks are embedded and upserted into pgvector. During RAG retrieval the top-8 chunks from the semantic similarity search are injected as catalog context into the system prompt.

#### Agentic Review (proposer/reviewer)

Beyond one-shot recommendations, the ai-service runs a **two-agent proposer/reviewer loop** over a logical model to produce higher-quality, self-critiqued metadata enrichment. A **proposer** agent drafts per-element descriptions, classifications, vocabulary concept mappings, and PII / direct-identifier flags; a **reviewer** agent then audits that draft against the dataset's full DCAT context and returns a verdict (`APPROVE` / `REJECT`) with per-issue comments. The proposer revises on each `REJECT`. The loop is **capped at 10 iterations**, and a **long-term review memory** carries lessons from past reviews into new runs to speed convergence. When the loop converges (or hits the cap), the final recommendation is persisted to the model's elements for a data owner to accept or reject in the producer Governance tab.

Progress streams live over **Server-Sent Events** — each `data:` line is a JSON `AgenticEvent`: phase markers (`CONTEXT`, `MEMORY`, `PROPOSING`, `REVIEWING`, `LOCKED`), the proposer's `PROPOSAL`, the reviewer's `REVIEW` (verdict + comments + summary), and a terminal `DONE` / `MAX_REACHED` / `ERROR`.

```bash
# Run the agentic review over one logical model and stream progress (SSE)
curl -N -X POST http://localhost:8005/api/v1/agentic-review \
  -H "Content-Type: application/json" \
  -H "X-API-Key: dev-local" \
  -H "Accept: text/event-stream" \
  -d '{"datasetId": "3fa85f64-5717-4562-b3fc-2c963f66afa6", "modelId": "3fa85f64-5717-4562-b3fc-2c963f66afa6"}'
```

> Swagger UI cannot render SSE — use `curl -N` or the producer UI to observe the stream.

---

### Identity Service

**Port:** 8006 | **Database:** PostgreSQL 16 (port 5436) + Keycloak (port 8180)

The identity-service manages organisations, users, roles, and access policies. It integrates with Keycloak 24 for OIDC token issuance and validation.

**Responsibilities:**
- User provisioning (invite, list, enable/disable) backed by the Keycloak Admin REST API — changes made in the producer Admin › Users UI are written directly to Keycloak and synced to the local `catalog_users` table
- Bidirectional Keycloak sync — on startup the service imports any existing Keycloak users into the local catalog database so that user references (e.g. `ownerId`) resolve correctly
- Organisation and domain management
- Long-lived API key issuance (stored as SHA-256 hashes)
- JWT issuer — all services trust `http://keycloak:8180/realms/datacatalog`

The Keycloak realm (`datacatalog`) is auto-imported from `infra/keycloak/datacatalog-realm.json` on first startup. Subsequent configuration changes must be applied via the Keycloak Admin Console (`http://localhost:8180`) or the Admin REST API — the import file is only read on a fresh database.

> Keycloak 24 uses `KEYCLOAK_ADMIN` and `KEYCLOAK_ADMIN_PASSWORD` environment variables. The old `KC_BOOTSTRAP_ADMIN_*` variables are not supported.

---

### Policy Service

**Port:** 8007 | **Database:** PostgreSQL 16 (port 5438)

The policy-service is the platform's **Policy Decision Point (PDP)**. It holds the ODRL policy registry for all datasets and evaluates them on demand using an internal implementation of the ODRE enforcement algorithm (Cimmino et al., *Computers & Security*, 2025).

**Responsibilities:**
- Maintain a `policy_records` registry keyed by `(dataset_id, tenant_id)` — policies are automatically synced from `inventory.datasets.changes` Kafka events when a data owner accepts terms
- Evaluate ODRL policies at request time via `POST /api/v1/policies/{datasetId}/evaluate`, running Algorithm 1 against the A-Level or B1-Level policy
- Persist an `evaluation_log` for every evaluation request
- Publish `PolicyEvaluationResultPayload` to the `policy.evaluations.completed` Kafka topic so downstream Policy Enforcement Points (API gateway, consumer layer) can act on decisions without coupling to policy logic

**ODRE Algorithm 1 (internal engine):**

The internal `OdreEngine` parses ODRL JSON-LD with Jackson and walks the `permission`, `prohibition`, and `obligation` rule arrays. For each rule it evaluates `constraint` triples using a three-tier comparison strategy (OffsetDateTime → numeric → String). Supported operators: `lt`, `lteq`, `eq`, `neq`, `gt`, `gteq` (namespace prefix stripped automatically). The engine supports `odrl:dateTime` as a built-in operand that resolves to the current instant.

**UsageDecision semantics:**

| Tuple | Meaning |
|---|---|
| `(action, "true")` | Permission granted — caller may perform the action |
| `(action, action)` | Delegated — engine cannot perform the action; caller must handle (e.g. show attribution notice) |
| Absent | All constraints failed — access denied |

```bash
# List all policy records for the current tenant
curl http://localhost:8007/api/v1/policies/{datasetId} \
  -H "X-API-Key: dev-local"

# Evaluate with B1-Level variable injection
curl -X POST http://localhost:8007/api/v1/policies/{datasetId}/evaluate \
  -H "X-API-Key: dev-local" -H "Content-Type: application/json" \
  -d '{"M": {"callerRole": "DATA_OWNER", "callerId": "user-uuid"}, "F": {}}'

# Fetch evaluation history
curl "http://localhost:8007/api/v1/policies/{datasetId}/evaluations?page=0&size=20" \
  -H "X-API-Key: dev-local"
```

---

## Authentication & User Management

### Overview

ODIN uses Keycloak 24 as its identity provider. The **producer UI** (`http://localhost:3000`) requires login before any content is shown. The **consumer UI** (`http://localhost:3001`) is read-only and does not require a login.

Authentication flow:
1. User visits the producer UI → redirected to the Keycloak login page
2. User logs in with email and password
3. Keycloak issues an OIDC access token (JWT) via the Authorization Code + PKCE flow
4. The producer app stores the token in memory and includes it as a `Bearer` header on every API call
5. Backend services validate the JWT signature against the Keycloak JWKS endpoint
6. The token is refreshed automatically every 30 seconds before it expires

---

### Roles

Four roles are defined in the `datacatalog` realm:

| Role | Keycloak name | Description | Admin nav visible |
|---|---|---|---|
| **Administrator** | `administrator` | Full platform access — manages users, harvest sources, settings, all data assets | All items |
| **Data Governance** | `data-governance` | Governs data quality and compliance across domains | Domains only |
| **Data Owner** | `data-owner` | Owns and manages data products and datasets in their domain | Hidden |
| **Data Steward** | `data-steward` | Curates metadata, semantic annotations, and logical models | Hidden |

Each role carries a `permissions` JWT claim that controls backend API access:

| Role | Backend permissions |
|---|---|
| Administrator | `catalog:read`, `catalog:write`, `catalog:admin` |
| Data Governance | `catalog:read`, `catalog:write` |
| Data Owner | `catalog:read`, `catalog:write` |
| Data Steward | `catalog:read`, `catalog:write` |

---

### Default Users

The following users are pre-configured in the default realm:

| Email | Password | Role |
|---|---|---|
| `admin@datacatalog.local` | `admin` | Administrator |
| `governance@datacatalog.local` | `password` | Data Governance |
| `owner@datacatalog.local` | `password` | Data Owner |
| `steward@datacatalog.local` | `password` | Data Steward |

> Change all passwords before exposing the stack to a network.

---

### Inviting Users

Users are managed through the Keycloak Admin Console. To add a new user:

1. Open the admin console → log in as `admin` / `admin`
   - Docker Compose: `http://localhost:8180`
   - MicroK8s: `http://keycloak.catalog.local/admin` (add the host to `/etc/hosts` as printed by `deploy.sh`)
2. Select the **datacatalog** realm
3. Go to **Users** → **Add user**
4. Set email, first name, last name; click **Create**
5. On the **Credentials** tab set a temporary password
6. On the **Role mapping** tab assign one of the four catalog roles
7. On the **Attributes** tab add:
   - Key `tenant_id` → value `00000000-0000-0000-0000-000000000001`
   - Key `permissions` → values matching the role's permission set (`catalog:read`, `catalog:write`, `catalog:admin` — see table above, one value per row)

The `tenant_id` and `permissions` attributes are mapped into the JWT by Keycloak protocol mappers and are required for the backend services to accept the token.

> The realm sets `unmanagedAttributePolicy: ADMIN_EDIT`, so the `tenant_id` and `permissions` attributes always save correctly from the Admin Console (or the Admin REST API). This policy ships with the realm import, so it applies automatically to any fresh deployment.

---

### Keycloak Configuration

**Realm:** `datacatalog`  
**Admin console:** `http://localhost:8180`  
**Admin credentials:** `admin` / `admin` (configured via `KEYCLOAK_ADMIN` / `KEYCLOAK_ADMIN_PASSWORD` env vars)

**Clients:**

| Client ID | Type | Used by |
|---|---|---|
| `catalog-frontend` | Public (PKCE) | Producer and consumer browser apps |
| `identity-service` | Confidential (service account) | Backend M2M calls to Keycloak Admin API |
| `catalog-api` | Bearer-only | Backend resource server JWT validation |

**Protocol mappers** (on `catalog-scopes` client scope):

| Mapper | Claim | Purpose |
|---|---|---|
| `tenant_id` | `tenant_id` (String) | Tenant scoping for all backend queries |
| `permissions` | `permissions` (JSON array) | Backend Spring Security authority grants |
| `realm_roles` | `realm_access.roles` (String array) | Frontend role-based UI visibility |

---

### Logging Out

Click **Sign out** in the sidebar footer of the producer UI. This terminates the Keycloak session — revisiting the app requires a fresh login.

To force-expire all sessions for a user (e.g., after a role change):
```
Keycloak Admin Console → Users → <user> → Sessions → Sign out all sessions
```

---

### Token Lifetimes

| Token | Lifetime |
|---|---|
| Access token | 1 hour (configurable in Keycloak realm settings) |
| Refresh token | Session-scoped (expires when browser closes) |
| Client-side refresh | Every 30 seconds (proactive, 60 s before access token expiry) |

---

## API Reference

### Authentication

| Header | Required | Description |
|---|---|---|
| `Authorization: Bearer <jwt>` | One of these two | Keycloak OIDC access token — tenant extracted from `tenant_id` JWT claim |
| `X-API-Key: <key>` | One of these two | Long-lived API key issued by identity-service |

When using a JWT, the `tenant_id` claim in the token provides tenant scoping automatically. No `X-Tenant-Id` header is needed.

Any key starting with `dev-` bypasses authentication and grants `catalog:admin` scope for local development:

```bash
curl -H "X-API-Key: dev-local" http://localhost:8001/api/v1/datasets
```

> **Do not use dev keys in production.** They grant unrestricted admin access to all tenants.

---

### Inventory API

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

**Semantic Types & Tags**

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/v1/datasets/{id}/semantic-context` | Dataset-level semantic summary: types, vocab concepts, accepted AI tags |
| `POST` | `/api/v1/datasets/{id}/semantic-tags` | Persist an accepted AI-recommended semantic tag |
| `DELETE` | `/api/v1/datasets/{id}/semantic-tags/{tagId}` | Remove a persisted semantic tag |
| `POST` | `/api/v1/datasets/{id}/recommend-semantic-context` | AI recommendation of semantic types for this dataset |

**Ownership & Governance**

| Method | Path | Description |
|---|---|---|
| `PUT` | `/api/v1/datasets/{id}/owner` | Assign an owner directly (`{userId}`) |
| `GET` | `/api/v1/datasets/{id}/history` | Paginated audit history for a dataset |
| `POST` | `/api/v1/datasets/{id}/ownership-proposals` | Propose an ownership transfer |
| `POST` | `/api/v1/datasets/{id}/ownership-proposals/{proposalId}/approve` | Approve a pending transfer |
| `POST` | `/api/v1/datasets/{id}/ownership-proposals/{proposalId}/reject` | Reject a pending transfer |
| `GET` | `/api/v1/datasets/{id}/ownership-proposals/pending` | Fetch the current pending proposal |

**Logical Models**

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/v1/datasets/{id}/logical-models` | List logical models for a dataset |
| `POST` | `/api/v1/datasets/{id}/logical-models` | Create a logical model |
| `GET` | `/api/v1/logical-models/{id}/elements` | List data elements |
| `POST` | `/api/v1/logical-data-elements/{id}/bind` | Bind element to a physical column |
| `POST` | `/api/v1/logical-data-elements/{id}/vocab-mappings` | Add a SKOS vocabulary mapping |
| `POST` | `/api/v1/logical-data-elements/{id}/recommend-classification` | AI-recommended data classification (PUBLIC / INTERNAL / CONFIDENTIAL / HIGH_CONFIDENTIAL) |
| `POST` | `/api/v1/logical-data-elements/{id}/accept-classification` | Accept the pending recommended classification |
| `POST` | `/api/v1/logical-data-elements/{id}/reject-classification` | Reject the pending recommended classification |
| `POST` | `/api/v1/logical-data-elements/{id}/recommend-description` | AI-generated element description |
| `POST` | `/api/v1/logical-data-elements/{id}/accept-description` | Accept the pending description |
| `POST` | `/api/v1/logical-data-elements/{id}/reject-description` | Reject the pending description |
| `POST` | `/api/v1/logical-models/{id}/recommend-classifications` | Bulk classification recommendation job for all elements in a model |
| `GET` | `/api/v1/logical-models/recommend-classifications/jobs/{jobId}` | Poll bulk classification job status |
| `POST` | `/api/v1/logical-data-elements/{id}/recommend-vocab-concepts` | AI-suggested SKOS concept mappings (up to 5) for an element |
| `POST` | `/api/v1/logical-data-elements/{id}/accept-vocab-concepts` | Accept recommendations — optional body `{"iris":[...]}` accepts only selected concepts; without body, accepts all; all recommendations cleared regardless |
| `POST` | `/api/v1/logical-data-elements/{id}/reject-vocab-concepts` | Reject recommendations without creating mappings |
| `POST` | `/api/v1/logical-models/{id}/recommend-vocab-concepts` | Bulk vocabulary concept recommendation job |
| `GET` | `/api/v1/logical-models/recommend-vocab-concepts/jobs/{jobId}` | Poll bulk vocab recommendation job status |
| `POST` | `/api/v1/logical-data-elements/{id}/recommend-pii` | AI-recommended `isPersonalInformation` and `isDirectIdentifier` values for one element |
| `POST` | `/api/v1/logical-data-elements/{id}/accept-pii` | Accept the pending PII recommendation |
| `POST` | `/api/v1/logical-data-elements/{id}/reject-pii` | Reject the pending PII recommendation |
| `POST` | `/api/v1/logical-models/{id}/recommend-pii` | Bulk PII/identifier recommendation job for all elements in a model |
| `GET` | `/api/v1/logical-models/recommend-pii/jobs/{jobId}` | Poll bulk PII recommendation job status |

**Terms of Use (ODRL)**

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/v1/datasets/{id}/terms-of-use` | Derive ODRL policy from element classifications + vocabulary concepts |
| `POST` | `/api/v1/datasets/{id}/terms-of-use/accept` | Lock derived policy as `hasPolicy` on the dataset |
| `DELETE` | `/api/v1/datasets/{id}/terms-of-use/policy` | Clear explicit policy; revert to dynamic derivation |

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
| `GET` | `/api/v1/datasets/lookup?namespace=&name=` | Resolve namespace/name → lineage service UUID |
| `GET` | `/api/v1/datasets/{id}/lineage` | Graph traversal by lineage UUID. Params: `direction`, `depth` |
| `GET` | `/api/v1/datasets/{id}/impact` | Downstream impact analysis by lineage UUID |
| `GET` | `/api/v1/catalog-datasets/{catalogId}/lineage-identity` | Resolve catalog UUID → lineage UUID, namespace, name |
| `PUT` | `/api/v1/datasets/{namespace}/{name}/catalog-link` | Link a catalog dataset UUID to a lineage dataset |
| `POST` | `/api/v1/ddl/submit` | Submit DDL for Calcite parsing |

---

### Search API

**Base URL:** `http://localhost:8004`

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/v1/search` | Full-text search. Params: `q`, `type`, `domainId`, `lifecycleStatus`, `format`, `hasLineage`, `keyword`, `theme`, `vocabConcept`, `vocab`, `semanticType`, `page`, `size` |
| `GET` | `/api/v1/search/suggest` | Autocomplete. Param: `q`. Returns up to 10 suggestions |
| `POST` | `/api/v1/admin/reindex` | Trigger full reindex from inventory-service |

The `semanticType` parameter filters by IRI terminal fragment (e.g. `?semanticType=Customer`). The search response includes a `semanticTypes` facet alongside the existing type/domain/format facets.

---

### AI API

**Base URL:** `http://localhost:8005`

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/v1/conversations` | List conversations for the current user |
| `POST` | `/api/v1/conversations` | Create a new conversation |
| `POST` | `/api/v1/conversations/{id}/messages` | Send a message. Set `Accept: text/event-stream` for SSE. Optional `focusDatasetId` body field pins context to a specific dataset |
| `POST` | `/api/v1/agentic-review` | Run the proposer/reviewer agent loop over a logical model and stream `AgenticEvent` progress (SSE). Body: `{"datasetId": "...", "modelId": "..."}`. The converged result is persisted to the model's elements for accept/reject |
| `POST` | `/api/v1/semantic-search` | Vector similarity search |
| `POST` | `/api/v1/admin/embeddings/refresh` | Re-embed all documents |

> **Reasoning model support:** When using qwen3 or other reasoning models, `<think>...</think>` blocks are stripped server-side before tokens are delivered to the client. The thinking phase is invisible to the SSE consumer.

---

### Policy API

**Base URL:** `http://localhost:8007`

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/v1/policies/{datasetId}` | Retrieve the stored ODRL policy record for a dataset |
| `PUT` | `/api/v1/policies/{datasetId}` | Upsert a policy. Body: `{"policyJson": "...", "policyLevel": "A"}` |
| `DELETE` | `/api/v1/policies/{datasetId}` | Remove the policy record for a dataset |
| `POST` | `/api/v1/policies/{datasetId}/evaluate` | Evaluate the policy. Body: `{"M": {}, "F": {}}`. Returns `{"granted": bool, "policyLevel": "A", "decisions": [...]}` |
| `GET` | `/api/v1/policies/{datasetId}/components` | Component breakdown of the assembled policy, keyed by piece type (`CLASSIFICATION`, `REGULATION`, `CONTRACTUAL`) and dimension, plus the assembled document |
| `GET` | `/api/v1/policies/{datasetId}/evaluations` | Paginated evaluation log. Params: `page`, `size` |

**`M` map (B1-Level variable injection):** Pass any runtime context that the stored policy references via `[=varName]` placeholders. Common keys: `callerRole`, `callerId`, `purpose`. Keys not referenced in the policy are silently ignored.

**`F` map (C-Level coded functions):** Reserved for future extension. Pass `{}` for all current policies.

```bash
# B1-Level policy: allow read only for DATA_OWNER role
POLICY='{
  "@context": "http://www.w3.org/ns/odrl.jsonld", "@type": "Set",
  "uid": "urn:example:b1",
  "permission": [{
    "target": "dataset:x", "action": "read",
    "constraint": [{
      "leftOperand": {"@value": "[=callerRole]", "@type": "xsd:string"},
      "operator": "eq",
      "rightOperand": {"@value": "DATA_OWNER", "@type": "xsd:string"}
    }]
  }]
}'

curl -X PUT "http://localhost:8007/api/v1/policies/{datasetId}" \
  -H "X-API-Key: dev-local" -H "Content-Type: application/json" \
  -d "{\"policyJson\": $(echo $POLICY | jq -Rc .), \"policyLevel\": \"B1\"}"

curl -X POST "http://localhost:8007/api/v1/policies/{datasetId}/evaluate" \
  -H "X-API-Key: dev-local" -H "Content-Type: application/json" \
  -d '{"M": {"callerRole": "DATA_STEWARD"}, "F": {}}'
# → {"granted": false, "decisions": []}

curl -X POST "http://localhost:8007/api/v1/policies/{datasetId}/evaluate" \
  -H "X-API-Key: dev-local" -H "Content-Type: application/json" \
  -d '{"M": {"callerRole": "DATA_OWNER"}, "F": {}}'
# → {"granted": true, "decisions": [{"action":"read","result":"true","delegated":false}]}
```

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
| `make logs svc=inventory-service` | Tail logs for a specific service |

**Profiles:**

| Profile | Additional services |
|---|---|
| (default) | All 7 services + Kafka + PostgreSQL × 6 + OpenSearch + MinIO + Keycloak |
| `ai` | Adds ai-service and Ollama |

```bash
# Start including AI features
docker compose --profile ai up -d
```

**Rebuilding a single image:**

```bash
# Always build then up — `restart` reuses the old image
docker compose build inventory-service
docker compose up -d inventory-service
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
| `POLICY_DB_PASSWORD` | policy | `policy` | PostgreSQL password for the policy-service database |
| `OLLAMA_BASE_URL` | ai | `http://ollama:11434` | Ollama base URL |
| `OPENAI_API_KEY` | ai | (empty) | OpenAI key; takes precedence over Ollama if set |
| `AI_CHAT_MODEL` | ai | `llama3` | Chat model name |
| `AI_EMBED_MODEL` | ai | `nomic-embed-text` | Embedding model (must be 768-dim) |
| `SNOWFLAKE_ACCOUNT` | harvest | — | Snowflake account identifier |
| `AWS_ACCESS_KEY_ID` | harvest | — | AWS credentials for Glue connector |
| `AWS_SECRET_ACCESS_KEY` | harvest | — | AWS credentials for Glue connector |
| `AWS_REGION` | harvest | `us-east-1` | AWS region for Glue |
| `JAVA_TOOL_OPTIONS` | All Java services | (empty) | JVM options injected at startup. Set to `-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005` to enable remote debugging. See [Local Development](#local-development) for port mappings. |

---

### Kubernetes (Raw Manifests)

Raw Kubernetes manifests live under `infra/kubernetes/`. Fourteen numbered YAML files cover everything from namespace creation to ingress rules. `deploy.sh` applies them in dependency order via `kubectl` and `envsubst`.

**Prerequisites**

| Tool | Purpose |
|---|---|
| `kubectl` | Cluster access (configured kubeconfig) |
| `envsubst` | Image registry injection — `apt install gettext-base` |
| `curl`, `jq` | Health checks and seed script |
| NGINX Ingress | `microk8s enable ingress` or install via manifest |

**1. Build and push images**

```bash
export IMAGE_REGISTRY=localhost:32000/   # MicroK8s built-in registry

for svc in inventory-service harvest-service lineage-service search-service ai-service identity-service policy-service; do
  docker build -t ${IMAGE_REGISTRY}odin/${svc}:latest services/${svc}/
  docker push ${IMAGE_REGISTRY}odin/${svc}:latest
done
```

**2. Deploy**

```bash
IMAGE_REGISTRY=localhost:32000/ ./infra/kubernetes/deploy.sh
```

The script applies all manifests in order, then waits for StatefulSets, Jobs, and Deployments to reach ready state.

| Flag | Description |
|---|---|
| `--dry-run` | Render manifests via `envsubst` without applying — review before applying |
| `--delete` | Tear down all resources; PVCs are preserved to protect data |

**3. Update /etc/hosts**

```bash
NODE_IP=$(kubectl get nodes \
  -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
echo "$NODE_IP  catalog.local manage.catalog.local api.catalog.local" \
  | sudo tee -a /etc/hosts
```

**4. Load sample data**

```bash
./infra/kubernetes/seed.sh --namespace odin-catalog
```

The seed script establishes `kubectl port-forward` tunnels to each service, waits for readiness, then delegates to `infra/seed/seed.sh` — the same Meridian Capital financial dataset scenario used with Docker Compose.

| Flag | Default | Description |
|---|---|---|
| `--namespace` | `odin-catalog` | Kubernetes namespace |
| `--api-key` | `dev-local` | `X-API-Key` header value |
| `--context` | (current context) | `kubectl` context |
| `--timeout` | `120` | Seconds to wait per service health check |

**5. Access**

| App | URL |
|---|---|
| Consumer (discovery) | `http://catalog.local` |
| Producer (management) | `http://manage.catalog.local` |
| API health | `http://api.catalog.local/inventory/actuator/health` |

**Manifest inventory**

| File | Contents |
|---|---|
| `00-namespace.yaml` | Namespace with `pod-security.kubernetes.io/enforce: privileged` |
| `01-serviceaccount.yaml` | Default service account |
| `02-secrets.yaml` | PostgreSQL passwords, MinIO credentials, Keycloak admin password |
| `03-configmaps.yaml` | Common Spring config, Kafka topic script, OpenSearch index mapping, Keycloak realm |
| `10-postgres.yaml` | Six StatefulSets: inventory, harvest, lineage (Apache AGE), identity, ai (pgvector), policy |
| `11-kafka.yaml` | KRaft-mode Kafka StatefulSet (no ZooKeeper) |
| `12-opensearch.yaml` | OpenSearch with privileged sysctl init container (`vm.max_map_count=262144`) |
| `13-minio.yaml` | MinIO Deployment for harvest snapshots |
| `14-redis.yaml` | Redis for Quartz scheduler clustering |
| `15-keycloak.yaml` | Keycloak 24 with realm auto-import |
| `20-backend-services.yaml` | Seven Spring Boot Deployments + ClusterIP Services |
| `21-frontends.yaml` | Producer and Consumer frontend Deployments + Services |
| `22-ingress.yaml` | NGINX Ingress for all three virtual hosts |
| `30-jobs.yaml` | One-shot Jobs: Kafka topic creation and OpenSearch index initialisation |

> **Port collision:** Stop any local Docker Compose stack before running `seed.sh`. It uses ports 8001–8004 for port-forwarding.

---

### Kubernetes (Helm / MicroK8s)

A Helm umbrella chart is provided under `infra/helm/` for clusters managed with Helm 3. A MicroK8s-specific `deploy.sh` under `infra/microk8s/` wraps the Helm install with sensible defaults.

```bash
# MicroK8s quick deploy
./infra/microk8s/deploy.sh

# Reduced resources for machines with < 16 GB RAM
./infra/microk8s/deploy.sh --reduced-resources

# Standard Helm install
helm install odin infra/helm \
  --namespace odin-catalog --create-namespace \
  -f infra/microk8s/values.yaml

# Upgrade after image rebuild
helm upgrade odin infra/helm \
  --namespace odin-catalog \
  -f infra/microk8s/values.yaml
```

See [microk8s-deployment.md](microk8s-deployment.md) for the full MicroK8s setup guide including snap installation, addon enablement, and image registry configuration.

---

## Security Hardening

The default Docker Compose stack is intentionally permissive for local development. Before deploying to any shared or internet-facing environment, apply the following mitigations.

### Environment credentials

All passwords in `docker-compose.yml` now default to weak fallbacks (`${VAR:-fallback}`). **Override every variable** in your deployment `.env` or secrets manager:

| Variable | What it protects |
|---|---|
| `INVENTORY_DB_PASSWORD` … `POLICY_DB_PASSWORD` | Six PostgreSQL databases |
| `MINIO_ROOT_USER` / `MINIO_ROOT_PASSWORD` | MinIO object store |
| `KEYCLOAK_ADMIN` / `KEYCLOAK_ADMIN_PASSWORD` | Keycloak admin console |
| `KEYCLOAK_CLIENT_SECRET` | Identity-service M2M Keycloak client |
| `OPENSEARCH_INITIAL_ADMIN_PASSWORD` | OpenSearch admin account |

### Dev API key bypass

The `X-API-Key: dev-*` bypass is controlled by `APP_DEV_AUTH_ENABLED`. It is enabled in `docker-compose.yml` for local convenience. **Set `APP_DEV_AUTH_ENABLED=false` (or omit the variable) in any non-local deployment** to disable it entirely.

### Keycloak

- Use `start --optimized` (not `start-dev`) in production; configure `KC_HOSTNAME`, a TLS terminator, and an external PostgreSQL database.
- Rotate the default realm client secret (`identity-service` client) via `KEYCLOAK_CLIENT_SECRET`.
- Seed users are created with `temporary: true` passwords — they will be prompted to change on first login. Remove or replace them before going to production.

### OpenSearch

`DISABLE_SECURITY_PLUGIN: "true"` is set for local development. Enable the security plugin in production and configure TLS + authentication. Rotate `OPENSEARCH_INITIAL_ADMIN_PASSWORD` to a strong secret.

### Kafka & Redis

Kafka runs on PLAINTEXT with no authentication. Redis runs with no password. For production:
- Kafka: enable SASL/SSL (`KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: SASL_SSL:SASL_SSL`).
- Redis: set `requirepass` via a `REDIS_PASSWORD` environment variable and update the harvest-service config.

### Network exposure

All infrastructure ports (`5433–5437`, `9092`, `6379`, `9200`, `9000`, `8180`) are bound to `0.0.0.0` in the Compose file for local convenience. In production, either remove the `ports:` mappings entirely (containers communicate over `catalog-net`) or bind to `127.0.0.1`.

### HTTPS / HSTS

The nginx containers serve over HTTP. Terminate TLS at your load balancer or Traefik, then uncomment the `Strict-Transport-Security` header line in both `frontend/producer/nginx.conf` and `frontend/consumer/nginx.conf`.

---

## Contributing

### Local Development

**Build all services:**

```bash
./gradlew build                              # compile + test all services
./gradlew :services:inventory-service:bootRun  # run one service locally
```

**Build the frontends:**

```bash
cd frontend/shared && pnpm install && pnpm build
cd ../producer  && pnpm install && pnpm dev   # http://localhost:3000
cd ../consumer  && pnpm install && pnpm dev   # http://localhost:3001
```

**Remote debugging:**

Enable JDWP on all Java services by setting `JAVA_TOOL_OPTIONS` in `.env` — no rebuild required:

```dotenv
JAVA_TOOL_OPTIONS=-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005
```

Then restart the services (`docker compose up -d`). Each service exposes a unique debug port on the host:

| Service | HTTP port | Debug host port |
|---|---|---|
| `inventory-service` | 8001 | **5001** |
| `harvest-service` | 8002 | **5002** |
| `lineage-service` | 8003 | **5003** |
| `search-service` | 8004 | **5004** |
| `ai-service` | 8005 | **5005** |
| `identity-service` | 8006 | **5006** |
| `policy-service` | 8007 | **5007** |

Connect your IDE to `localhost:<debug-port>`. The JVM logs `Listening for transport dt_socket at address: 5005` on startup when active. Leave `JAVA_TOOL_OPTIONS=` (empty) to disable with zero overhead.

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
