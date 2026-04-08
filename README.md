# Databricks ABAC Examples

Hands-on notebooks for **Attribute-Based Access Control (ABAC)** with Databricks Unity Catalog. Each notebook is self-contained, tested end-to-end, and covers a distinct ABAC pattern.

**GitHub Pages site:** https://abhidatabricks.github.io/DatabricksABACExamples/

---

## Notebooks

| # | Notebook | Pattern | Compliance |
|---|----------|---------|-----------|
| 1 | `ABAC_Tutorial_1_Row_Filtering.ipynb` | Row filtering — regional, healthcare, financial | PCI-DSS |
| 2 | `ABAC_Tutorial_2_Column_Masking.ipynb` | Column masking — 6 techniques + industry examples | PCI-DSS, HIPAA |
| 3 | `ABAC_Tutorial_5_Multi_Domain_Governance.ipynb` | Multi-domain 2D tagging (domain × sensitivity) | CPNI |

---

## Prerequisites

### 1. Databricks Runtime

- **DBR 14.3 LTS or later** (Unity Catalog ABAC requires DBR 14.1+; 14.3 LTS recommended for stability)
- Serverless SQL Warehouse or Classic SQL Warehouse attached to a Unity Catalog metastore

### 2. Unity Catalog Metastore

- A Unity Catalog metastore must be attached to your workspace
- You must have **metastore admin** or **catalog owner** privileges to create governed tags
- All notebooks use the catalog/schema: `abac_tutorial.abac_tut` (configurable via `catalog` and `schema` variables at the top of each notebook)

### 3. Account Groups

The notebooks use these account-level groups. Create them in **Account Console → Groups** before running:

| Group Name | Used In | Purpose |
|------------|---------|---------|
| `abac_tut_admins` | All notebooks | Full access — sees all rows and unmasked data |
| `abac_tut_us_team` | Tutorial 1 | Sees US-region rows only |
| `abac_tut_eu_team` | Tutorial 1 | Sees EU-region rows only |
| `abac_tut_apac_team` | Tutorial 1 | Sees APAC-region rows only |
| `abac_tut_hr_team` | Tutorial 2, 3 | Sees HR-domain unmasked data |
| `abac_tut_finance_team` | Tutorial 2, 3 | Sees Finance-domain unmasked data |
| `abac_tut_clinical_staff` | Tutorial 1, 2 | Healthcare clinical access (time-based emergency override) |
| `abac_tut_compliance_team` | Tutorial 1, 2 | Compliance/audit access |

> **How to create groups:** Account Console → Groups → Add Group → enter name → Save.
> Then add your test users as members of the relevant groups.

### 4. Governed Tags (Tag Policies)

Governed tags are created via the **Databricks REST API** (`POST /api/2.1/tag-policies`). Each notebook's setup cell creates the required tags automatically using your workspace token.

Tags required per notebook:

**Tutorial 1 — Row Filtering:**
| Tag Key | Allowed Values |
|---------|---------------|
| `abac_tut_region` | `us`, `eu`, `apac` |

**Tutorial 2 — Column Masking:**
| Tag Key | Allowed Values |
|---------|---------------|
| `abac_tut_sensitivity` | `pii`, `internal`, `public` |
| `pci_data` | `credit_card`, `ssn`, `income` |
| `phi_level` | `direct_id`, `quasi_id`, `clinical` |

**Tutorial 3 — Multi-Domain Governance:**
| Tag Key | Allowed Values |
|---------|---------------|
| `telco_domain` | `identity`, `financial`, `network` |
| `telco_sensitivity` | `confidential`, `internal` |
| `telco_service_type` | `mobile`, `broadband`, `iot` |

> **Note:** Governed tag creation requires metastore admin or a privilege that allows `POST /api/2.1/tag-policies`. If the API call fails with a 403, ask your metastore admin to create the tags, or use Catalog Explorer → Tags.

### 5. Unity Catalog Permissions

The user running the notebooks needs:

| Permission | Where |
|-----------|-------|
| `USE CATALOG` | On `abac_tutorial` catalog |
| `CREATE SCHEMA` | On `abac_tutorial` catalog |
| `ALL PRIVILEGES` (or `CREATE TABLE`, `MODIFY`) | On `abac_tutorial.abac_tut` schema |
| `CREATE FUNCTION` | On `abac_tutorial.abac_tut` schema |
| `CREATE POLICY` (schema-level) | Requires metastore admin or catalog owner |

> The `CREATE POLICY` privilege at schema scope is a **metastore admin** or **catalog owner** operation. If you are not a catalog owner, ask your admin to grant `CREATE POLICY ON SCHEMA abac_tutorial.abac_tut TO <your-user>`.

### 6. Databricks Personal Access Token

Each notebook calls the REST API to create governed tags. Set the following at the top of each notebook (or via a Databricks secret):

```python
token = "<your-databricks-personal-access-token>"
workspace_url = "https://<your-workspace>.azuredatabricks.net"  # or .cloud.databricks.com
```

Generate a PAT: **User Settings → Developer → Access Tokens → Generate new token**.

---

## Running the Notebooks

### Import to Databricks Workspace

**Option A — Databricks CLI:**
```bash
databricks workspace import \
  --file ABAC_Tutorial_1_Row_Filtering.ipynb \
  --path /Users/<your-email>/ABAC_Tutorial_1_Row_Filtering \
  --format JUPYTER \
  --overwrite
```

**Option B — UI:** Workspace → Import → upload the `.ipynb` file.

### Run Order

Run notebooks independently — each is self-contained. Within each notebook, run cells **top to bottom**. Each notebook starts with a cleanup cell that drops any policies or tables from previous runs.

### Switching User Roles (Testing Access Control)

To test as a non-admin user:
1. Add a test user to one of the groups above (e.g., `abac_tut_eu_team`)
2. Share the notebook or table with that user (grant `SELECT` on the tables)
3. Have them run only the **query cells** (not the setup cells) — the policies apply at query time

---

## Key ABAC Constraints (Gotchas)

These are the hard rules discovered through testing. Violating them produces cryptic errors.

| Rule | Error if violated |
|------|------------------|
| Mask function return type must **exactly** match the column type | `CAST_INVALID_INPUT` |
| `EXCEPT group` clause is only valid in `CREATE POLICY`, **not** in `ALTER TABLE SET MASK` | `PARSE_SYNTAX_ERROR at EXCEPT` |
| `MATCH COLUMNS` only accepts `has_tag()` / `has_tag_value()` — no comparison operators | `UC_INVALID_POLICY_CONDITION` |
| A column used in `ROW FILTER USING COLUMNS` must NOT have a column mask policy | `UC_USING_REF_OF_MASKED_COLUMNS` |
| Only one active row filter per table; only one active column mask per column | `MULTIPLE_ROW_FILTERS` / `MULTIPLE_MASKS` |
| `DROP TABLE IF EXISTS` required before `CREATE OR REPLACE TABLE` when columns have governed tags | `AnalysisException` on column tag |
| Governed tag values are validated — tagging a column with an undefined value fails | `INVALID_PARAMETER_VALUE` |

---

## Catalog & Schema

All notebooks default to:
- **Catalog:** `abac_tutorial`
- **Schema:** `abac_tut`

Change the `catalog` and `schema` variables in the first cell of each notebook to use a different location.

---

## License

Apache 2.0. See [LICENSE](LICENSE).
