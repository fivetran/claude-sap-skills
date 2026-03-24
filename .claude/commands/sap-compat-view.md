---
name: sap-compat-view
description: >
  Creates a new SAP compatibility view in the fivetran/dbt_sap project. Use this
  when asked to add a compatibility view for an SAP table (e.g. BSAK, BSAD, COEP).
  Covers all required file changes: staging base model, column macro, staging model,
  compatibility view SQL, and YAML/README documentation. Includes known gotchas from
  prior implementation work.
metadata:
  team: data
---

# SAP Compatibility View — Implementation Guide

Use BSAD as the canonical reference throughout. The BSAD files are the closest analogue
to most new views. Always read them before writing anything new.

Reference files:
- `models/staging/base/stg_sap__bsad_bck_tmp.sql`
- `macros/get_column_macros/get_bsad_bck_columns.sql`
- `models/staging/stg_sap__bsad_bck.sql`
- `models/compatibility_views/mart/bsad.sql`

---

## Files to create or edit (in order)

### 1. `dbt_project.yml` — add source var

Under `vars > sap:`, add after the nearest alphabetical neighbor:

```yaml
<table>_bck: "{{ source('sap', '<table>_bck') }}"
```

### 2. `models/staging/src_sap.yml` — add source table

Copy the `bsad_bck` block and adapt it. Key fields to change:
- `name:` and `identifier:` → new table name
- `enabled:` var → `sap_using_<table>_bck`
- Column list → match the actual SAP schema for this table

### 3. `models/staging/base/stg_sap__<table>_bck_tmp.sql` — base model

```sql
{{ config(enabled=var('sap_using_<table>_bck', True)) }}

select *
from {{ var('<table>_bck') }}
```

### 4. `macros/get_column_macros/get_<table>_bck_columns.sql` — column macro

Copy from `get_bsad_bck_columns.sql`. Add/remove columns to match the target SAP table.
Keep alphabetical order. Datatypes follow these rules:
- IDs, codes, indicators, dates stored as strings → `dbt.type_string()`
- Amounts, quantities, percentages → `dbt.type_numeric()`
- Integer counts (e.g. `pendays`) → `dbt.type_int()`
- Fivetran metadata: `_fivetran_deleted` → `boolean`, `_fivetran_synced` → `dbt.type_timestamp()`

**Critical:** `_fivetran_deleted` is defined in the macro but intentionally NOT selected in
the staging model's `final` CTE. Every other column in the macro must appear in `final`.

### 5. `models/staging/stg_sap__<table>_bck.sql` — staging model

Copy from `stg_sap__bsad_bck.sql`. The structure is fixed:
- `base` CTE: applies `remove_slashes_from_col_names`
- `fields` CTE: applies `fivetran_utils.fill_staging_columns` with the column macro
- `final` CTE: explicit column list with casts

**Audit checklist before saving:**
- Every column in the macro (except `_fivetran_deleted`) must appear in `final`
- Cross-check both directions: macro → final, and final → macro
- Common miss: multi-currency suffix columns like `mwst2`, `mwst3`, `sknt2`, `sknt3`
  added late to the macro but missed in the final select list

### 6. `models/compatibility_views/mart/<table>.sql` — compatibility view

Structure: two CTEs (`archived_records` + `cleared_records`) joined by `UNION ALL`.

**`archived_records`** — reads from `stg_sap__<table>_bck where xarch = 'X'`

Columns that don't exist in the archive table must be hardcoded:
```sql
cast('' as {{ dbt.type_string() }}) as kontt,
cast('' as {{ dbt.type_string() }}) as kontl,
cast('00000000' as {{ dbt.type_string() }}) as uebgdat,
cast('00000000' as {{ dbt.type_string() }}) as pernr,
cast('' as {{ dbt.type_string() }}) as vorgn,
cast('' as {{ dbt.type_string() }}) as awtyp,
cast('' as {{ dbt.type_string() }}) as logsystem_sender,
cast('' as {{ dbt.type_string() }}) as bukrs_sender,
cast('' as {{ dbt.type_string() }}) as belnr_sender,
cast('0000' as {{ dbt.type_string() }}) as gjahr_sender,
cast('000' as {{ dbt.type_string() }}) as buzei_sender,
cast('' as {{ dbt.type_string() }}) as j_1tpbupl,
cast(0 as {{ dbt.type_numeric() }}) as fcsl,
cast('' as {{ dbt.type_string() }}) as rfccur,
cast('' as {{ dbt.type_string() }}) as bdgt_account,
cast('' as {{ dbt.type_string() }}) as re_account,
cast('' as {{ dbt.type_string() }}) as payt_rsn,
```

**`cleared_records`** — reads from `stg_sap__bseg i LEFT OUTER JOIN stg_sap__bkpf h`

WHERE clause:
- Vendor tables (BSAK-like): `i.koart = 'K'`
- Customer tables (BSAD-like): `i.koart = 'D'`
- Both: `and not (i.augbl = '') and not (i.h_bstat = 'D') and not (i.h_bstat = 'M')`

**Before referencing any column from `i.` (BSEG) or `h.` (BKPF), verify it exists in that
staging model.** Referencing a column that doesn't exist produces an `invalid identifier`
Snowflake error at runtime — dbt compile will not catch this.

Check with:
```bash
grep -n "<column>" models/staging/stg_sap__bseg.sql
grep -n "<column>" models/staging/stg_sap__bkpf.sql
```

Known BSAK-specific columns that are NOT in `stg_sap__bseg` (hardcode these):

| Column | Hardcode value |
|--------|---------------|
| `zollt`, `zolld` | `''` |
| `diekz` | `''` |
| `xesrd` | `''` (cannot derive — `esrnr` also missing from BSEG) |
| `lnran` | `''` |
| `kzbtr` | `''` |
| `zekkn` | `''` |
| `qsznr` | `''` |
| `penlc1`, `penlc2`, `penlc3` | `0` |
| `penfc` | `0` |
| `pernr` | `'00000000'` |
| `vorgn` | `''` |

Known columns NOT in `stg_sap__bkpf` (hardcode these):

| Column | Hardcode value |
|--------|---------------|
| `penrc` | `''` |

**Type gotcha:** `dtws1`, `dtws2`, `dtws3`, `dtws4` are `dbt.type_string()` — do NOT cast
them as `dbt.type_numeric()` even though the name looks numeric.

### 7. `models/staging/stg_sap.yml` — staging docs

Copy the `stg_sap__bsad_bck` block and adapt column descriptions.

**Checklist:**
- `_fivetran_synced` appears once, near the top of the column list
- `_dataaging` appears once, in its correct position in the data columns (not duplicated near `_fivetran_synced`)
- Every column in the staging model's `final` CTE has a corresponding entry here
- Easy misses: `j_1tpbupl`, `buzei_sender` (vendor-specific, added last)

### 8. `models/compatibility_views/mart/compatibility_views.yml` — compat view docs

Copy the `bsad` block. Insert alphabetically. Adapt column descriptions.

### 9. `README.md` — table entry

Add a row to the Compatibility Views table, alphabetically by table name:
```
| [<table>](link) | <SAP table description> | `sap_using_<table>_bck` |
```

---

## Post-implementation audit

Run these checks after creating all files, before running dbt:

1. **Macro vs final CTE**: list all column names from the macro; confirm each appears in
   the staging model's `final` select (excluding `_fivetran_deleted`)
2. **BSEG column availability**: for every `i.<col>` in `cleared_records`, grep
   `stg_sap__bseg.sql` to confirm it's there
3. **BKPF column availability**: for every `h.<col>` in `cleared_records`, grep
   `stg_sap__bkpf.sql` to confirm it's there
4. **YAML duplicates**: check `stg_sap.yml` for duplicate column entries in the new block
5. **UNION column count**: both CTEs in the compat view must output the same columns in
   the same order

---

## Quick reference: BSAK vs BSAD differences

BSAK (vendor) has these that BSAD (customer) does not:
- `lifnr` instead of `kunnr`
- `ebeln`, `ebelp` (purchase order)
- `xblnr` (comes from archive directly, not BKPF)
- `zollt`, `zolld` (customs)
- `diekz`, `xesrd`, `lnran`, `kzbtr`, `zekkn`
- `qsshb`, `qbshb`, `qsznr`, `qsfbt` (withholding tax detail)
- `penlc1-3`, `penfc`, `pendays`, `penrc` (penalty)
- `pernr`, `vorgn`
- koart filter = `'K'` (vs `'D'` for BSAD)
