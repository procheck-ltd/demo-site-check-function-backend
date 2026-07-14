# MP Connect — compliance schema

Data model for the site access check function (demo). Nine tables: three
lookups, two core entities, and the fact/log tables. Compliance is evaluated
at check time and every check is logged with a RAG outcome; failures are
recorded against a controlled list of fail codes.

```mermaid
erDiagram
  worker ||--o{ user_qualification : holds
  qualification_type ||--o{ user_qualification : classifies
  site ||--o{ site_requirement : requires
  qualification_type ||--o{ site_requirement : "required as"
  vehicle_type ||--o{ site_requirement : "scoped to"
  worker ||--o{ access_check : checked
  site ||--o{ access_check : "location of"
  access_check ||--o{ access_check_fail : "fails with"
  fail_code ||--o{ access_check_fail : categorises
  qualification_type ||--o{ access_check_fail : "flagged for"

  qualification_type {
    smallint id PK
    text code "unique"
    text name
    text category "nullable"
    boolean active
  }
  vehicle_type {
    smallint id PK
    text code "unique"
    text name
    boolean active
  }
  fail_code {
    smallint id PK
    text code "unique"
    text label
    text severity "amber/red"
    boolean active
  }
  worker {
    bigint id PK
    text full_name
    text nexus_id "unique"
    boolean active
  }
  site {
    bigint id PK
    text code "unique slug"
    text name
    boolean active
  }
  user_qualification {
    bigint id PK
    bigint worker_id FK
    smallint qualification_type_id FK
    date valid_from
    date valid_to "nullable"
    timestamptz updated_at
  }
  site_requirement {
    bigint id PK
    bigint site_id FK
    smallint qualification_type_id FK
    text visit_type "nullable"
    smallint vehicle_type_id FK "nullable"
    smallint alternative_group "OR-group"
    boolean mandatory
  }
  access_check {
    bigint id PK
    bigint worker_id FK
    bigint site_id FK
    text visit_type
    text vehicle_types "array of codes"
    text outcome "green/amber/red"
    boolean is_override
    text override_reason "nullable"
    boolean information_only
    text checked_by
    text check_method
    text ruleset_version
    timestamptz occurred_at
  }
  access_check_fail {
    bigint id PK
    bigint access_check_id FK
    smallint fail_code_id FK
    smallint qualification_type_id FK "nullable"
    text detail "nullable"
    timestamptz created_at
  }
```

## Tables

**Lookups**
- `qualification_type` — controlled list of qualification codes (CSCS, WAH, MED…). The shared vocabulary both fact tables reference.
- `vehicle_type` — controlled list of vehicle types (tanker, tipper, mixer, van) for driver-scoped requirements.
- `fail_code` — controlled list of failure reasons with a severity (`amber` warning / `red` fail).

**Core**
- `worker` — the person being checked.
- `site` — the site being controlled.

**Compliance (current state)**
- `user_qualification` — the qualifications a worker holds, with a validity window.
- `site_requirement` — what a site requires, scoped by `visit_type` / `vehicle_type`; `alternative_group` expresses OR-logic (any one in a group satisfies it); `mandatory` distinguishes red vs advisory.

**Access (logs)**
- `access_check` — one row per gate check: overall RAG `outcome`, override flag, who/how it was checked, and the `ruleset_version` in force.
- `access_check_fail` — child rows of a check: one per failing/warning requirement, each carrying a `fail_code`.

## Notes
- RAG: `red` = missing/expired mandatory qualification; `amber` = a required qualification expiring within the grace window (default 30 days); `green` = all satisfied.
- Current-state model — qualifications and requirements are edited in place; the check/fail logs are the historical record.
