# Redshift → Unity Catalog Policy Sync

A Databricks notebook that reads the access-control **privileges (GRANTs)** from an Amazon
Redshift database and reproduces them as equivalent **Unity Catalog grants** on the
**Lakehouse Federation foreign catalog** that federates that Redshift database.

**Why:** When you federate Redshift into Unity Catalog, the foreign catalog mirrors the
databases, schemas, and tables — but **not** the access grants. Users who could read
specific schemas/tables in native Redshift have no access when querying the same data through
Databricks until it is re-granted. This notebook automates that re-granting: extract from
Redshift, translate to UC, generate reviewable SQL, and (optionally) apply.

> **Notebook:** [`Redshift_to_UC_Policy_Sync_v1.ipynb`](./Redshift_to_UC_Policy_Sync_v1.ipynb)

---

## Quickstart for the SWA team

**What it does:** reads GRANTs from Redshift system views → translates them to Unity Catalog
grants (catalog / schema / table) → writes a reviewable `.sql` → optionally applies them.

**Before you run (one-time):**
1. A Lakehouse Federation **foreign catalog** for the Redshift database must exist and be
   visible to your compute.
2. Compute must be able to reach Redshift on **port 5439**.
3. Create a **secret scope** with your Redshift credentials:
   ```bash
   databricks secrets create-scope <scope>
   databricks secrets put-secret <scope> host
   databricks secrets put-secret <scope> user
   databricks secrets put-secret <scope> password
   ```
4. Target Databricks users/groups should already exist — grants map **by name**
   (a Redshift group maps to a Databricks group of the same name).

**Run it:**
1. Import the notebook and attach compute.
2. Set the widgets: `secret_scope`, `redshift_database`, `uc_foreign_catalog`. Leave
   `apply_grants = False`.
3. Run top to bottom. Use the **connection test** cell near the top first.
4. Review the generated grants (STEP 5) and the exported `.sql` file.
5. When you're happy, set `apply_grants = True` and re-run the final cell to apply.

**Switches you'll likely touch** (full details below):
- `INCLUDE_SCHEMAS` — empty = all schemas; set a Python list to scope/speed up.
- `PRINCIPAL_OVERRIDES` — only if a Redshift name differs from the Databricks name.
- `CAPTURE_OWNER_PRIVILEGES` / `EFFECTIVE_PRIVILEGES_MODE` — enable to also capture *implicit*
  access (see [the three modes](#the-three-privilege-capture-modes)).

**Note:** the `SVV_*` views report **only explicit grants**. If access relies on object
ownership or group membership, use owner-capture or effective-privileges mode. Foreign
catalogs are read-only, so only read privileges are synced.

---

## Table of contents
- [How it works](#how-it-works)
- [Prerequisites](#prerequisites)
- [One-time setup: secret scope](#one-time-setup-secret-scope)
- [Running the notebook](#running-the-notebook)
- [Configuration reference (every switch)](#configuration-reference-every-switch)
- [Privilege mapping](#privilege-mapping)
- [Identity (principal) mapping](#identity-principal-mapping)
- [The three privilege-capture modes](#the-three-privilege-capture-modes)
- [Testing](#testing)
- [Verification](#verification)
- [Limitations & caveats](#limitations--caveats)
- [Security notes](#security-notes)

---

## How it works

The notebook runs top to bottom in seven logical steps:

1. **Configure** – read connection + target settings; credentials come from a Databricks
   secret scope (no plaintext).
2. **Preflight** – verify the foreign catalog is visible, the password is present, and
   Redshift is reachable — fail fast with a clear message instead of cascading errors.
3. **Extract** – query Redshift system views for the actual privileges.
4. **Discover** – list schemas in the foreign catalog (informational).
5. **Transform** – map Redshift privileges & principals → Unity Catalog grants.
6. **Generate** – print the grants and export them to a `.sql` file for review.
7. **Apply** *(opt-in)* – execute the grants against Unity Catalog.

It connects to Redshift with the pure-Python `redshift_connector` (no JDBC JAR) and issues UC
grants with `spark.sql`.

---

## Prerequisites

| # | Requirement | Notes |
|---|---|---|
| 1 | A Lakehouse Federation **foreign catalog** for the Redshift database | e.g. `dkushari_swa_rs`. Must exist and be visible to the compute running the notebook (`USE CATALOG`). |
| 2 | **Network path** from the Databricks compute to Redshift on port **5439** | Security group / VPC peering / PrivateLink. Serverless works if the path is open. |
| 3 | A Redshift user with **system-view access** | Superuser recommended for full visibility of the `SVV_*` views. |
| 4 | Target Databricks **principals already exist** | Users/groups referenced by the grants must exist in the Databricks account (see [Identity mapping](#identity-principal-mapping)). |
| 5 | `redshift_connector` on the cluster | Installed by the notebook via `%pip install`. |

---

## One-time setup: secret scope

Credentials are read from a Databricks secret scope so nothing sensitive lives in the
notebook. Create it once with the Databricks CLI:

```bash
databricks secrets create-scope swa_redshift
databricks secrets put-secret   swa_redshift host        # Redshift endpoint hostname
databricks secrets put-secret   swa_redshift user        # Redshift username
databricks secrets put-secret   swa_redshift password    # Redshift password
```

The scope name is configurable via the `secret_scope` widget (default `swa_redshift`). Secret
values are shown as `[REDACTED]` in notebook output.

---

## Running the notebook

Run the cells top to bottom. Only two cells ever need your attention:

1. **CONFIGURATION** – set the widgets and (optionally) the Python knobs (see the reference
   below). Leave `apply_grants = False` for the first pass.
2. **STEP 7 – Apply** – runs in **review-only** mode first. When you are happy with the
   generated SQL, set the `apply_grants` widget to `True` and re-run just this cell to apply.

Everything else — install, connection test, preflight, extract, transform, export — is just
"run the cell". The connection-test cell near the top lets you confirm connectivity before
doing any real work.

---

## Configuration reference (every switch)

All switches live in the **CONFIGURATION** cell. There are two kinds: **widgets** (shown at
the top of the notebook) and **Python knobs** (variables inside the cell).

### Widgets

| Widget | Default | Example | What it does / why |
|---|---|---|---|
| `apply_grants` | `False` | `True` | **Review vs. apply.** `False` = generate + export SQL only, change nothing (safe default). `True` = execute the grants against Unity Catalog. Always review at `False` first. |
| `secret_scope` | `swa_redshift` | `prod_redshift` | Name of the Databricks secret scope holding `host`/`user`/`password`. Lets you point at different credential sets per environment. |
| `uc_foreign_catalog` | `dkushari_swa_rs` | `swa_redshift_fc` | The Lakehouse Federation foreign catalog the grants are applied to. This is the UC-side target. |
| `redshift_database` | `oetrta` | `analytics` | The Redshift database to read privileges from. Extraction is scoped to this database only. |
| `output_path` | `/Workspace/Users/.../redshift_grants.sql` | any workspace path | Where the generated `.sql` file is written for review. Directory is auto-created; falls back to `/tmp` if unwritable. |

### Python knobs

| Knob | Default | Example | What it does / why |
|---|---|---|---|
| `PRINCIPAL_OVERRIDES` | `{}` | `{"rs_analysts": "SWA Analysts"}` | **Name mapping** for principals whose Redshift name differs from the Databricks name. Format: `{"redshift_name": "databricks_name"}`. Leave empty when names match 1:1. |
| `MAP_PUBLIC_TO_ACCOUNT_USERS` | `True` | `False` | Maps Redshift `PUBLIC` → the built-in Databricks group `account users`. Set `False` to skip PUBLIC grants entirely. |
| `INCLUDE_SCHEMAS` | `[]` (all) | `["amenon", "finance"]` | **Schema scope.** Empty list = ALL user schemas (default). Populate a Python list to restrict the sync to specific schemas — useful for testing or phased rollout, and much faster than scanning every schema. System schemas are always excluded. |
| `CAPTURE_OWNER_PRIVILEGES` | `False` | `True` | Also capture **implicit owner** privileges. Redshift object owners have implicit USAGE/SELECT that the `SVV_*` views do NOT report. When `True`, each schema/table owner is added as a grantee. Default `False` because owners are often the admin, whose matching UC principal may not exist. |
| `EFFECTIVE_PRIVILEGES_MODE` | `False` | `True` | **Most complete mode.** Resolves each user's *effective* access — explicit + all implicit paths (ownership, group/role membership, PUBLIC) — using `HAS_SCHEMA_PRIVILEGE` / `HAS_TABLE_PRIVILEGE`. Produces per-user grants. Slower (evaluates users × objects). See [modes](#the-three-privilege-capture-modes). |
| `EFFECTIVE_EXCLUDE_SUPERUSERS` | `True` | `False` | Only relevant when `EFFECTIVE_PRIVILEGES_MODE = True`. Skips superusers (who implicitly have everything) so you don't grant SELECT-on-everything to admins. |
| `SYSTEM_SCHEMAS` | `["information_schema", "pg_catalog", "pg_internal", "pg_automv", "catalog_history"]` | — | Schemas always excluded as noise. Rarely needs changing. |

---

## Privilege mapping

Only the privileges relevant to reading federated data are translated (foreign catalogs are
read-only in UC):

| Redshift source view | Redshift privilege | → | Unity Catalog grant |
|---|---|---|---|
| `SVV_DATABASE_PRIVILEGES` | any (implies access) | → | `GRANT USE CATALOG ON CATALOG <catalog>` |
| `SVV_SCHEMA_PRIVILEGES` | `USAGE` | → | `GRANT USE SCHEMA ON SCHEMA <catalog>.<schema>` |
| `SVV_RELATION_PRIVILEGES` | `SELECT` | → | `GRANT SELECT ON TABLE <catalog>.<schema>.<table>` |

`USE CATALOG` is added automatically for every mapped principal. `USE SCHEMA` is added for any
schema that has a table-level SELECT even if no explicit schema `USAGE` row exists.

**Intentionally not synced:** write privileges (`INSERT`/`UPDATE`/`DELETE`/`CREATE`) — foreign
catalogs are read-only; column-level grants — limited UC support; default/future-object grants
(`SVV_DEFAULT_PRIVILEGES`).

---

## Identity (principal) mapping

Principals are mapped **by name**, regardless of whether they are a Redshift user, group, or
role:

- **1:1 by name (default):** Redshift `swa_analysts` → Databricks `swa_analysts`.
- **Group → group:** a Redshift group maps to a Databricks **account group** of the same name.
- **`PUBLIC` → `account users`:** controlled by `MAP_PUBLIC_TO_ACCOUNT_USERS`.
- **Overrides:** use `PRINCIPAL_OVERRIDES` when a name differs between the two systems.

All principals are backtick-quoted in the generated SQL (required for names with spaces, e.g.
`` `account users` ``).

> **Important:** the Apply step only succeeds if the target principal **exists in the
> Databricks account** (and, for groups, is assigned to the workspace). Unmapped or
> non-existent principals are reported at apply time (❌) without stopping the run; the rest
> still apply.

---

## The three privilege-capture modes

| Mode | Switch | Captures | Grant granularity | Cost | Apply requires |
|---|---|---|---|---|---|
| **Explicit** (default) | *(none)* | Explicit `GRANT`s only | user / group / public as-granted | fast | matching UC principals |
| **+ Owner** | `CAPTURE_OWNER_PRIVILEGES=True` | + object ownership | + owner as a user | fast | + owner UC users |
| **Effective** | `EFFECTIVE_PRIVILEGES_MODE=True` | **everything** (explicit + ownership + group/role membership + PUBLIC) | **per-user, fully resolved** | slow (users × objects) | matching UC **users** |

**Recommendation:** start with **Explicit + group mirroring** — it keeps grants at the group
level and matches how access is usually administered. Use **Effective** mode when you need a
provable "no one loses access" guarantee during a cutover, accepting that it flattens
everything to per-user grants that require matching UC users.

---

## Testing

The first code cell (`TEST SETUP ONLY`) contains commented Redshift statements to seed test
privileges. Run them in the **Redshift Query Editor** (not the notebook):

```sql
-- Option A: grant to PUBLIC (maps to UC `account users`, always applies)
GRANT USAGE ON SCHEMA <schema> TO PUBLIC;
GRANT SELECT ON ALL TABLES IN SCHEMA <schema> TO PUBLIC;

-- Option B: grant to a GROUP (maps 1:1 to a Databricks account group)
CREATE GROUP swa_analysts;
GRANT USAGE  ON SCHEMA <schema>                TO GROUP swa_analysts;
GRANT SELECT ON ALL TABLES IN SCHEMA <schema>  TO GROUP swa_analysts;
```

For Option B, create a Databricks **account group** `swa_analysts` (assigned to the workspace)
so the apply resolves it.

---

## Verification

After applying, confirm the grants in Unity Catalog (the notebook includes `%sql` cells for
this):

```sql
SHOW GRANTS ON CATALOG `<catalog>`;
SHOW GRANTS ON SCHEMA  `<catalog>`.`<schema>`;
SHOW GRANTS ON TABLE   `<catalog>`.`<schema>`.`<table>`;
```

Federated objects also appear as `TABLE_TYPE = FOREIGN` in `information_schema.tables`, and
applied grants are visible in `information_schema.{catalog,schema,table}_privileges`.

---

## Limitations & caveats

- **`SVV_*_PRIVILEGES` show only explicit grants** — not owner-implicit, membership-implicit,
  or default/future privileges. Use `CAPTURE_OWNER_PRIVILEGES` or `EFFECTIVE_PRIVILEGES_MODE`
  to go beyond explicit grants.
- **Foreign catalogs are read-only** in Unity Catalog, so only read-oriented privileges are
  meaningful; write privileges are intentionally skipped.
- **`SVV_RELATION_PRIVILEGES` can be slow** across all schemas — scope with `INCLUDE_SCHEMAS`
  when possible.
- **Superusers** are not synced (they implicitly have everything); handle admin access
  separately.
- **Column-level grants** are out of scope (limited UC foreign-catalog support).
- The Apply step is **idempotent** (re-running is safe) but only lands grants for principals
  that exist in the Databricks account.

---

## Security notes

- Redshift credentials are stored in a **Databricks secret scope**, never in the notebook.
- The notebook defaults to **review-only** (`apply_grants=False`); applying is a deliberate,
  separate action.
- The generated `.sql` file is a reviewable, auditable artifact suitable for change
  management.
