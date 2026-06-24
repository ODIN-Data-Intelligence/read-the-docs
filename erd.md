# ODIN Catalog — Database ERD

Six databases, one per service. All primary keys are `UUID`. Foreign keys reference PKs in the same database only; cross-service references use soft IDs stored as `UUID` columns without a database-level FK constraint.

---

## 1. inventory-service · PostgreSQL 16

Owns the DCAT / DPROD / CSV-W metadata model. `resources` is a polymorphic base table; every typed row (catalog, dataset, distribution, etc.) has a matching row there via a shared PK.

```mermaid
erDiagram
    resources {
        uuid        id              PK
        varchar     resource_type
        text        iri             UK
        uuid        tenant_id
        uuid        domain_id
        text        title
        text        description
        timestamptz issued
        timestamptz modified
        text_arr    language
        text_arr    keywords
        text_arr    themes
        text        license
        text        rights_statement
        text        access_rights
        text_arr    conforms_to
        uuid        creator_id
        uuid        publisher_id
        jsonb       contact_points
        text        source_uri
        jsonb       extra
        boolean     is_deleted
        timestamptz created_at
        timestamptz updated_at
    }

    catalogs {
        uuid     resource_id     PK  "FK → resources"
        text     homepage
        uuid_arr has_part
    }

    datasets {
        uuid        resource_id         PK  "FK → resources"
        uuid        catalog_id
        text        accrual_periodicity
        timestamptz temporal_start
        timestamptz temporal_end
        float       spatial_resolution_m
        text        temporal_resolution
        text        version
        text        version_notes
        uuid        is_version_of       FK  "self → datasets"
    }

    distributions {
        uuid    resource_id         PK  "FK → resources"
        uuid    dataset_id          FK
        text    access_url
        text    download_url
        text    media_type
        text    format
        bigint  byte_size
        varchar checksum_algorithm
        text    checksum_value
        text    compress_format
        text    package_format
        text    availability
        uuid    csvw_table_id       FK  "deferred"
    }

    data_services {
        uuid     resource_id          PK  "FK → resources"
        text     endpoint_url
        text     endpoint_description
        uuid_arr serves_dataset
        text     protocol
        text     security_schema_type
    }

    data_products {
        uuid    resource_id             PK  "FK → resources"
        varchar lifecycle_status
        uuid    owner_id
        text    purpose
        varchar information_sensitivity
        jsonb   has_policy
    }

    data_product_ports {
        uuid        id              PK
        uuid        data_product_id FK
        varchar     port_type
        uuid        data_service_id FK
        uuid        dataset_id      FK
        uuid        distribution_id FK
        timestamptz created_at
    }

    catalog_records {
        uuid        resource_id       PK  "FK → resources"
        uuid        catalog_id
        uuid        primary_topic_id
        timestamptz listing_date
        timestamptz modification_date
        text        harvest_source
    }

    csvw_tables {
        uuid        id              PK
        uuid        distribution_id FK
        text        url
        text        title
        text        description
        jsonb       dialect
        boolean     suppress_output
        text        table_direction
        text        notes
        timestamptz created_at
        timestamptz updated_at
    }

    csvw_table_schemas {
        uuid     id           PK
        uuid     table_id     FK
        text_arr primary_key
        text     about_url
        text     property_url
        text     value_url
    }

    csvw_columns {
        uuid    id                      PK
        uuid    schema_id               FK
        int     ordinal
        text    name
        text_arr titles
        text    datatype
        boolean required
        boolean virtual
        boolean suppress_output
        text    lang
        text    default_value
        text    property_url
        text    value_url
        text    about_url
        text    description
        uuid    logical_data_element_id FK  "nullable"
    }

    vocabularies {
        uuid        id              PK
        text        name
        text        prefix          UK
        text        base_iri        UK
        varchar     vocabulary_type
        text        description
        text        version
        text        homepage
        boolean     is_system
        timestamptz created_at
    }

    dataset_vocabulary_profiles {
        uuid        id            PK
        uuid        dataset_id    FK
        uuid        vocabulary_id FK
        boolean     is_primary
        text_arr    domain_tags
        timestamptz created_at
    }

    logical_models {
        uuid        id          PK
        uuid        dataset_id  FK
        text        name
        text        description
        text        version
        varchar     status
        timestamptz created_at
        timestamptz updated_at
    }

    logical_data_elements {
        uuid        id               PK
        uuid        logical_model_id FK
        text        name
        text        label
        text        description
        text        logical_type
        boolean     is_required
        boolean     is_identifier
        boolean     is_nullable
        int         ordinal
        timestamptz created_at
        timestamptz updated_at
    }

    logical_element_vocab_mappings {
        uuid        id                 PK
        uuid        logical_element_id FK
        uuid        vocabulary_id      FK
        text        concept_iri
        text        concept_label
        text        concept_definition
        varchar     match_type
        timestamptz created_at
    }

    cross_model_mappings {
        uuid        id           PK
        varchar     source_type
        uuid        source_id
        varchar     target_type
        uuid        target_id
        text        mapping_type
        timestamptz created_at
    }

    resources      ||--o|  catalogs                     : "extends"
    resources      ||--o|  datasets                     : "extends"
    resources      ||--o|  distributions                : "extends"
    resources      ||--o|  data_services                : "extends"
    resources      ||--o|  data_products                : "extends"
    resources      ||--o|  catalog_records              : "extends"

    datasets       ||--o{  distributions                : "has"
    datasets       o|--o|  datasets                     : "isVersionOf"
    datasets       ||--o{  dataset_vocabulary_profiles  : "profiles"
    datasets       ||--o{  logical_models               : "models"

    distributions  o|--o|  csvw_tables                  : "describes"

    csvw_tables    ||--o{  csvw_table_schemas            : "schema"
    csvw_table_schemas ||--|{ csvw_columns              : "columns"

    data_products  ||--o{  data_product_ports            : "ports"
    data_product_ports o{--o| data_services             : "via"
    data_product_ports o{--o| datasets                  : "via"
    data_product_ports o{--o| distributions             : "via"

    vocabularies   ||--o{  dataset_vocabulary_profiles   : "used in"
    vocabularies   ||--o{  logical_element_vocab_mappings : "mapped by"

    logical_models ||--|{  logical_data_elements         : "elements"

    logical_data_elements ||--o{ csvw_columns           : "bound by"
    logical_data_elements ||--o{ logical_element_vocab_mappings : "mappings"
```

---

## 2. harvest-service · PostgreSQL 16

Tracks harvest sources, scheduled jobs, run history, and per-entity results.

```mermaid
erDiagram
    harvest_sources {
        uuid        id              PK
        uuid        tenant_id
        text        name
        varchar     source_type
        text        base_url
        text        region
        text        database_name
        text_arr    schema_filter
        text        credential_ref
        jsonb       extra_config
        timestamptz created_at
        timestamptz updated_at
    }

    harvest_credentials {
        uuid        id                PK
        uuid        source_id         FK
        varchar     credential_type
        text        encrypted_payload
        timestamptz created_at
    }

    harvest_jobs {
        uuid        id              PK
        uuid        source_id       FK
        text        name
        text        schedule_cron
        boolean     full_refresh
        boolean     enabled
        timestamptz created_at
        timestamptz updated_at
    }

    harvest_runs {
        uuid        id                  PK
        uuid        job_id              FK
        uuid        source_id           FK
        varchar     status
        varchar     triggered_by
        timestamptz started_at
        timestamptz completed_at
        int         entities_discovered
        int         entities_created
        int         entities_updated
        int         entities_failed
        text        snapshot_path
        text        error_message
        boolean     full_refresh
        timestamptz created_at
    }

    harvest_run_items {
        uuid        id                 PK
        uuid        run_id             FK
        varchar     entity_type
        text        source_key
        uuid        canonical_id
        varchar     action
        jsonb       raw_payload
        jsonb       normalized_payload
        text        error_detail
        timestamptz created_at
    }

    harvest_sources   ||--o{  harvest_credentials  : "credentials"
    harvest_sources   ||--o{  harvest_jobs         : "jobs"
    harvest_jobs      ||--o{  harvest_runs         : "runs"
    harvest_sources   ||--o{  harvest_runs         : "runs"
    harvest_runs      ||--o{  harvest_run_items    : "items"
```

---

## 3. lineage-service · PostgreSQL 16 + Apache AGE

Relational tables persist OpenLineage events and column-level lineage. Apache AGE mirrors the same relationships as a Cypher-queryable property graph for multi-hop traversal.

```mermaid
erDiagram
    lineage_jobs {
        uuid    id            PK
        text    namespace
        text    name
        jsonb   facets
        bigint  age_vertex_id
    }

    lineage_datasets {
        uuid    id                  PK
        text    namespace
        text    name
        jsonb   facets
        jsonb   schema_facet
        uuid    catalog_resource_id
        bigint  age_vertex_id
    }

    lineage_runs {
        uuid        id                 PK
        text        run_id             UK
        uuid        job_id             FK
        jsonb       facets
        timestamptz nominal_start_time
        timestamptz nominal_end_time
    }

    lineage_run_events {
        uuid        id          PK
        uuid        run_id      FK
        varchar     event_type
        timestamptz event_time
        text        producer
        text        schema_url
        jsonb       inputs
        jsonb       outputs
        jsonb       raw_event
        timestamptz created_at
    }

    column_lineage {
        uuid    id                  PK
        uuid    run_event_id        FK
        uuid    output_dataset_id   FK
        text    output_column
        uuid    input_dataset_id    FK
        text    input_column
        text    transformation_type
    }

    lineage_jobs       ||--o{  lineage_runs        : "runs"
    lineage_runs       ||--o{  lineage_run_events  : "events"
    lineage_run_events ||--o{  column_lineage      : "columns"
    column_lineage     o{--o|  lineage_datasets    : "output dataset"
    column_lineage     o{--o|  lineage_datasets    : "input dataset"
```

> **Apache AGE graph** (`lineage_graph`):
> - Vertices: `Job {namespace, name}`, `Dataset {namespace, name}`, `Column {dataset, name}`
> - Edges: `READ_BY` (Dataset→Job), `WRITES_TO` (Job→Dataset), `DERIVED_FROM` (Dataset→Dataset), `COLUMN_LINEAGE` (Column→Column)
> - `lineage_jobs.age_vertex_id` and `lineage_datasets.age_vertex_id` link relational rows to their AGE vertex IDs.

---

## 4. ai-service · PostgreSQL 16 + pgvector

Stores chat conversations and the vector embedding corpus used for RAG.

```mermaid
erDiagram
    conversations {
        uuid        id          PK
        uuid        tenant_id
        uuid        user_id
        text        title
        timestamptz created_at
    }

    conversation_messages {
        uuid        id              PK
        uuid        conversation_id FK
        varchar     role
        text        content
        int         token_count
        text        model_used
        timestamptz created_at
    }

    embedding_documents {
        uuid        id          PK
        uuid        tenant_id
        varchar     entity_type
        uuid        entity_id
        int         chunk_index
        text        content
        vector768   embedding
        text        model_name
        jsonb       metadata
        timestamptz created_at
    }

    conversations ||--|{ conversation_messages : "messages"
```

> **pgvector index**: `embedding_documents.embedding` uses an `IVFFlat` index with cosine distance (`vector_cosine_ops`, 100 lists) for k-NN similarity search.
> The `(entity_id, chunk_index, model_name)` composite key ensures idempotent upserts on re-embedding.

---

## 5. identity-service · PostgreSQL 16

Manages organizations (tenants), domain hierarchy, users, and API keys. Keycloak is the authoritative identity provider; `catalog_users` mirrors relevant attributes and holds ABAC roles/permissions.

```mermaid
erDiagram
    organizations {
        uuid        id           PK
        text        name         UK
        text        display_name
        text        description
        varchar     plan
        boolean     active
        timestamptz created_at
    }

    domains {
        uuid        id               PK
        uuid        tenant_id
        text        name
        text        description
        uuid        parent_domain_id FK  "self → domains"
        uuid        owner_id
        timestamptz created_at
        timestamptz updated_at
    }

    catalog_users {
        uuid        id               PK
        uuid        tenant_id
        text        email            UK
        text        first_name
        text        last_name
        text        keycloak_user_id UK
        boolean     active
        text_arr    roles
        text_arr    permissions
        timestamptz created_at
        timestamptz updated_at
    }

    api_keys {
        uuid        id           PK
        uuid        tenant_id
        uuid        owner_id
        text        key_hash     UK
        text        description
        boolean     active
        timestamptz expires_at
        text_arr    scopes
        timestamptz created_at
        timestamptz last_used_at
    }

    organizations  ||--o{  domains       : "tenancy"
    organizations  ||--o{  catalog_users : "members"
    organizations  ||--o{  api_keys      : "keys"
    domains        o|--o{  domains       : "parent"
    catalog_users  ||--o{  api_keys      : "owns"
```

---

## 6. policy-service · PostgreSQL 16

Owns the ODRL policy registry and the ODRE evaluation audit trail. `policy_records` holds the assembled, effective ODRL policy per dataset, while `policy_pieces` are reusable, typed policy fragments (CLASSIFICATION / REGULATION / CONTRACTUAL) that `dataset_policy_links` composes onto datasets. `dataset_id` is a soft cross-service UUID (no FK to inventory-service).

```mermaid
erDiagram
    policy_records {
        uuid        id              PK
        uuid        dataset_id          "UK with tenant_id"
        uuid        tenant_id           "UK with dataset_id"
        varchar     policy_level
        jsonb       policy_json
        timestamptz created_at
        timestamptz updated_at
    }

    evaluation_log {
        uuid        id              PK
        uuid        dataset_id
        uuid        tenant_id
        varchar     action
        boolean     granted
        jsonb       request_context
        timestamptz created_at
    }

    policy_pieces {
        uuid        id              PK
        uuid        tenant_id           "UK with piece_type, dimension_key"
        varchar     piece_type          "CLASSIFICATION | REGULATION | CONTRACTUAL"
        varchar     dimension_key
        text        label
        jsonb       policy_json
        varchar     policy_level
        timestamptz created_at
        timestamptz updated_at
    }

    dataset_policy_links {
        uuid        id              PK
        uuid        dataset_id          "UK with tenant_id, piece_id"
        uuid        tenant_id
        uuid        piece_id        FK  "→ policy_pieces, ON DELETE CASCADE"
        timestamptz applied_at
    }

    policy_pieces ||--o{ dataset_policy_links : "applied to datasets"
```

> Each evaluation call (`POST /api/v1/policies/{datasetId}/evaluate`) appends one row to `evaluation_log`. The effective `policy_records.policy_json` for a dataset is the composition of every `policy_pieces` row linked via `dataset_policy_links`.
