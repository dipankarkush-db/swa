# Usage — Redshift → Unity Catalog Policy Sync

A quick, practical guide for running the notebook. For the full reference (every switch,
examples, and the three capture modes), see [README.md](./README.md).

**What it does:** reads GRANTs from Redshift system views → translates them to Unity Catalog
grants (catalog / schema / table) → writes a reviewable `.sql` → optionally applies them.

---

## Before you run (one-time)

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

---

## Run it

1. Import `Redshift_to_UC_Policy_Sync_v1.ipynb` and attach compute.
2. Set the widgets: `secret_scope`, `redshift_database`, `uc_foreign_catalog`. Leave
   `apply_grants = False`.
3. Run top to bottom. Use the **connection test** cell near the top first.
4. Review the generated grants (STEP 5) and the exported `.sql` file.
5. When you're happy, set `apply_grants = True` and re-run the final cell to apply.
6. Verify with the `SHOW GRANTS` cells at the end.

---

## Switches you'll likely touch

| Switch | Use it to… |
|---|---|
| `INCLUDE_SCHEMAS` | Empty = all schemas. Set a Python list (e.g. `["amenon"]`) to scope the sync and speed it up. |
| `PRINCIPAL_OVERRIDES` | Map a Redshift name to a different Databricks name (only when they differ). |
| `MAP_PUBLIC_TO_ACCOUNT_USERS` | Map Redshift `PUBLIC` → Databricks `account users` (on by default). |
| `CAPTURE_OWNER_PRIVILEGES` | Also include implicit **owner** privileges. |
| `EFFECTIVE_PRIVILEGES_MODE` | Resolve each user's full **effective** access (explicit + all implicit). |

---

## Good to know

- `apply_grants = False` (the default) changes nothing — it only generates and exports the
  SQL. Applying is a deliberate second pass.
- The `SVV_*` views report **only explicit grants**. If access relies on object ownership or
  group membership, enable owner-capture or effective-privileges mode.
- Foreign catalogs are **read-only** in Unity Catalog, so only read privileges are synced.
- A grant only applies successfully if its target principal **exists in the Databricks
  account** (groups must be assigned to the workspace); unmatched principals are reported and
  skipped without stopping the run.
