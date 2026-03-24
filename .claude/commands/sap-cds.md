---
name: sap-cds
description: >
  SAP CDS View expert skill. Use for: translating SAP HANA SQL to ANSI SQL (BigQuery,
  Snowflake, Databricks), understanding the 8-phase extraction pipeline, managing the
  Fivetran connector on sapidess4, working with ignore_association logic, debugging
  connector issues, understanding the 4 custom ABAP function modules (Z_CDS_SQL_VIEWS,
  Z_VIEW_DDL, Z_CDS_DEPENDENCIES, Z_VIEW_GET_META), and any SAP CDS view related task.
metadata:
  team: data
---

# SAP CDS View — Expert Context

You are an expert on SAP CDS views, the Fivetran extraction pipeline, HANA-to-ANSI
SQL translation, and the custom ABAP function modules. Use this context for any task
involving CDS views, the connector, SQL translation, or SAP-side RFCs.

---

## Architecture Overview

### Production Environment

- **Remote server**: `ssh root@sapidess4`
- **Production directory**: `/usr/sap/cds_sql_only/`
- **Local Mac copy**: `/Users/antonio.carbone/cds_enhancement_20260210/github_release/src/`
- **GCS bucket**: `gs://sap_cds_dbt/sap_cds_views/cds_sql_only_<N>/`
- **BigQuery dataset**: `sap_cds_views`
- **SAP client number**: `100`
- **Documentation**: `/Users/antonio.carbone/claude/SAP_CDS_SQL_Translation_Guide.md`

### Directory Structure (sapidess4:/usr/sap/cds_sql_only/)

```
connector.py                  # Python entry point (Fivetran SDK)
drivers/
  installation.sh             # Bootstrap: installs JDK, sets env, writes .env
  sapjco3.jar                 # SAP JCo library
  libsapjco3.so               # SAP JCo native library (Linux)
  gson.jar                    # Google Gson
  SimpleDependencyTable.class                          # Phase 1
  SimpleDependencyTablePhase2Recursive.class           # Phase 2
  SimpleDependencyTablePhase3Descriptions.class        # Phase 3
  SimpleDependencyTablePhase4FieldMetadataFixed.class  # Phase 4
  SimpleDependencyTablePhase5SqlDefinitions.class      # Phase 5
  SimpleDependencyTablePhase6SqlOnlyViews.class        # Phase 6
  SimpleDependencyTablePhase8ChainResolution.class     # Phase 8
  DependencyResolver.class, DependencyNode.class, DependencyGraph.class
  *.java                      # Source files for compilation
```

### Pipeline Flow

```
Java Phase (on SAP via JCo RFC) → CSV files → connector.py → op.upsert() → Fivetran SDK → GCS → BigQuery
```

### Compilation (on sapidess4)

```bash
ssh root@sapidess4
cd /usr/sap/cds_sql_only/drivers
javac -cp .:sapjco3.jar:gson.jar <ClassName>.java
```

### Syncing Files

```bash
# Download from production
scp root@sapidess4:/usr/sap/cds_sql_only/connector.py /tmp/
scp root@sapidess4:/usr/sap/cds_sql_only/drivers/<file> /tmp/

# Upload to production
scp /tmp/<file> root@sapidess4:/usr/sap/cds_sql_only/drivers/
scp /tmp/<file> root@sapidess4:/usr/sap/cds_sql_only/

# Sync full directory
rsync -av root@sapidess4:/usr/sap/cds_sql_only/ /Users/antonio.carbone/cds_enhancement_20260210/github_release/src/
```

---

## The 8-Phase Extraction Pipeline

| Phase | Java Class | Output | SAP RFC |
|-------|-----------|--------|---------|
| 1 | SimpleDependencyTable | *_DEPENDENCIES.csv | (standard DDLDEPENDENCY read) |
| 2 | SimpleDependencyTablePhase2Recursive | Enhanced dependencies | Z_CDS_DEPENDENCIES |
| 3 | SimpleDependencyTablePhase3Descriptions | *_DESCRIPTIONS.csv | Z_CDS_DEPENDENCIES |
| 4 | SimpleDependencyTablePhase4FieldMetadataFixed | *_metadata files | (DD03L/DD04T directly) |
| 5 | SimpleDependencyTablePhase5SqlDefinitions | *_SQL_DEFINITION.csv | Z_VIEW_DDL |
| 6 | SimpleDependencyTablePhase6SqlOnlyViews | *_ABAP/HANA_SQL_DEFINITION.csv | Z_VIEW_DDL, Z_CDS_DEPENDENCIES |
| 7 | SimpleDependencyTablePhase7ViewMetadata | SQL_NAME_MAPPING.csv, DD02L, etc. | Z_VIEW_GET_META |
| 8 | SimpleDependencyTablePhase8ChainResolution | SQL_ONLY_RESOLUTION.csv, SQL_ONLY_OBJECTS.csv, SQL_ONLY_NAME_MAPPING.csv | Z_CDS_SQL_VIEWS, Z_VIEW_DDL, Z_VIEW_GET_META |

---

## installation.sh (Critical for Fivetran)

The Fivetran container has NO Java. `installation.sh` runs before connector.py and:
1. Downloads portable OpenJDK 11 from Adoptium to `$HOME/jdk/` (x86_64 or aarch64)
2. Verifies sapjco3.jar, libsapjco3.so, gson.jar exist
3. Verifies all critical .class files (30+)
4. Sets LD_LIBRARY_PATH, JAVA_HOME, PATH, CLASSPATH
5. Writes `drivers/.env` for connector.py to source

connector.py loads `.env` at startup. If Java still not found, it runs installation.sh.

---

## ignore_association Column (sql_only_resolution table)

### Purpose
Distinguishes real data dependencies from CDS association pointers (lazy joins that are
NOT materialized in HANA SQL).

### Decision Logic

1. Parse BASEINFO block from CDS DDL source:
   ```json
   "FROM": ["VIEW_A", "VIEW_B"],
   "ASSOCIATED": ["VIEW_A", "VIEW_C", "VIEW_D"]
   ```

2. Compute: `association_only = ASSOCIATED - FROM` → `{VIEW_C, VIEW_D}`

3. Classify:
   - Child in `association_only` set → `ignore_association = YES`
   - Child NOT in set (or found via parseFromJoin) → `ignore_association = NO`
   - BASEINFO missing or fallback discovery → `NO` (conservative default)

### Impact on Translation
- `NO` → must exist in target database
- `YES` → safely exclude from translation (not in compiled SQL)
- Filtering by YES can cut the dependency tree in half

---

## HANA SQL → ANSI SQL Translation Rules

### Always Apply (mechanical)

| SAP/HANA | Target | Action |
|----------|--------|--------|
| `"SAPHANADB"."VIEW"` | `"VIEW"` or `schema.VIEW` | Remove/replace schema |
| `N'text'` | `'text'` | Remove N prefix |
| `MANY TO MANY JOIN` / `MANY TO ONE JOIN` | `JOIN` | Strip cardinality hints |
| `WITH READ ONLY` | (remove) | Not needed |
| `SESSION_CONTEXT('CDS_CLIENT')` | `'100'` | Replace with SAP client |
| `"=A0"` generated aliases | `"A0"` | Remove = from aliases |
| ABAP table names (ILDGRCOMPCOCR) | CDS names (I_LEDGERCOMPANYCODECRCYROLES) | Use SQL_NAME_MAPPING |

### Function Translation

| HANA Function | BigQuery | Snowflake | Databricks |
|--------------|----------|-----------|------------|
| `DATS_TIMS_TO_TSTMP(date, time, tz, ...)` | `SAFE.PARSE_TIMESTAMP('%Y%m%d%H%M%S', CONCAT(date, time))` | `TRY_TO_TIMESTAMP_NTZ(CONCAT(date, time), 'YYYYMMDDHH24MISS')` | `TO_TIMESTAMP(CONCAT(date, time), 'yyyyMMddHHmmss')` |
| `ABAP_SYSTEM_TIMEZONE(...)` | `'UTC'` | `'UTC'` | `'UTC'` |
| `TSTMP_TO_DATS(ts, tz, ...)` | `FORMAT_TIMESTAMP('%Y%m%d', ts)` | `TO_CHAR(ts, 'YYYYMMDD')` | `DATE_FORMAT(ts, 'yyyyMMdd')` |
| `FLTP_TO_DEC(f, p, s)` | `CAST(f AS NUMERIC)` | `CAST(f AS NUMBER(p,s))` | `CAST(f AS DECIMAL(p,s))` |
| `CURRENCY_CONVERSION(...)` | **Cannot translate** — needs TCURR tables | Same | Same |
| `UNIT_CONVERSION(...)` | **Cannot translate** — needs T006 tables | Same | Same |

### Dialect-Specific

| Aspect | BigQuery | Snowflake | Databricks |
|--------|----------|-----------|------------|
| Quoting | Backticks `` `col` `` | Double quotes `"COL"` | Backticks `` `col` `` |
| Schema | `` `project.dataset.view` `` | `"SCHEMA"."VIEW"` | `` `catalog`.`schema`.`view` `` |
| VARCHAR | `STRING` | `VARCHAR(n)` | `STRING` |
| DECIMAL | `NUMERIC` / `BIGNUMERIC` | `NUMBER(p,s)` | `DECIMAL(p,s)` |
| SUBSTRING | `SUBSTR` | `SUBSTRING` or `SUBSTR` | `SUBSTRING` |
| SAP dates | `PARSE_DATE('%Y%m%d', val)` | `TO_DATE(val, 'YYYYMMDD')` | `TO_DATE(val, 'yyyyMMdd')` |

---

## CDS View Name Layers

A single view has three names:

| Layer | Example | Where Used |
|-------|---------|-----------|
| CDS Entity Name | `I_Product` | DDL source, developer-facing |
| ABAP Dictionary Name | `IPRODUCT` | DD02L, ABAP programs, Phase 2 |
| HANA SQL View Name | `"IPRODUCT"` | HANA catalog, ABAP SQL definition |

Use `SQL_NAME_MAPPING.csv` (Phase 7/8) for ABAP ↔ CDS name translation.

---

## Translation Strategy (Hybrid — Recommended)

1. **Filter dependencies**: Exclude rows where `ignore_association = YES`
2. **Resolve dependency order**: Leaf tables first, then intermediate views, then top-level
3. **Translate commonly-used intermediate views** as actual views in target DB
4. **Inline simple/single-use views** (flatten to base tables)
5. **Apply mechanical rules** (schema, N'', cardinality, aliases, MANDT)
6. **Translate functions** (DATS_TIMS_TO_TSTMP etc.) per target dialect
7. **Flag untranslatable** (CURRENCY_CONVERSION, UNIT_CONVERSION) for manual review

### Translation Feasibility

- ~75% auto-translatable (mechanical rules only)
- ~15% need function translation rules
- ~10% need manual intervention (currency/unit conversion, complex nesting)

---

## Quick Reference: Key Tables in BigQuery

| Table | Content |
|-------|---------|
| `sql_only_resolution` | Parent-child dependencies with ignore_association |
| `sql_only_objects` | All discovered objects with type (VIEW/TABLE/STOB) |
| `sql_only_name_mapping` | DDL name ↔ ABAP name |
| `*_sql_definition` | Phase 5: HANA CREATE VIEW SQL |
| `*_abap_sql_definition` | Phase 6: SQL_ONLY HANA CREATE VIEW |
| `*_hana_sql_definition` | Phase 6: SQL_ONLY CDS DDL source |

---

## Custom ABAP Function Modules (Z_*)

These 4 custom function modules run inside SAP (IDES S/4HANA) and are called remotely
via JCo RFC from the Java extraction phases.

**To edit**: SAP transaction SE37 on the IDES system.
**ABAP source locations**:
- `~/Downloads/Z_CDS_SQL_VIEWS.abap`
- `sapidess4:/usr/sap/sap-fivetran-connector-databricks/src/abap/Z_VIEW_DDL.abap`
- `sapidess4:/usr/sap/sap-fivetran-connector-databricks/src/abap/Z_VIEW_GET_META.abap`
- Z_CDS_DEPENDENCIES: no local ABAP file (maintained directly in SE37)

### Phase-to-RFC Mapping

```
Phase 1  → (no custom Z_* — uses standard DDLDEPENDENCY read)
Phase 2  → Z_CDS_DEPENDENCIES  (identify SQL-only views)
Phase 3  → Z_CDS_DEPENDENCIES  (check if SQL-only)
Phase 4  → (no custom Z_* — uses DD03L/DD04T directly)
Phase 5  → Z_VIEW_DDL           (HANA CREATE VIEW SQL)
Phase 6  → Z_VIEW_DDL           (ABAP SQL) + Z_CDS_DEPENDENCIES (HANA CDS source)
Phase 7  → Z_VIEW_GET_META      (full metadata: DD02L, columns, descriptions)
Phase 8  → Z_CDS_SQL_VIEWS      (authoritative deps) + Z_VIEW_DDL (SQL parsing)
           + Z_VIEW_GET_META     (fallback metadata)
```

---

### Z_CDS_SQL_VIEWS

**Purpose**: Authoritative dependency resolver for CDS views. Reads DDL source with
BASEINFO metadata block (FROM + ASSOCIATED arrays) and returns all referenced objects.
Primary RFC for Phase 8 chain resolution.

**Interface**:
```
IMPORTING:  IV_DDLNAME    TYPE DDLNAME           " DDL source name
EXPORTING:  EV_SOURCE     TYPE STRING             " DDL source (with BASEINFO)
            EV_ABAP_VIEW  TYPE SOBJ_NAME          " ABAP view name
            ET_REFERENCES TYPE ZDDLDEPENDENCY_TAB " All referenced objects
EXCEPTIONS: DDL_NOT_FOUND, READ_ERROR
```

**ET_REFERENCES fields**: DDLNAME, OBJECTNAME, OBJECTTYPE (`VIEW`/`STOB`), STATE

**ABAP Logic**:
1. Check DDDDLSRC for DDL name; if not found, reverse-lookup via DDLDEPENDENCY
2. Read DDL source with `get_internals = abap_true` (includes BASEINFO block):
   ```abap
   lo_handler = cl_dd_ddl_handler_factory=>create_internal( ).
   lo_handler->read( name = lv_ddlname  get_state = 'A'  get_internals = abap_true ).
   ```
3. Get self-mapping from DDLDEPENDENCY (STOB + VIEW entries)
4. Parse BASEINFO FROM + ASSOCIATED arrays from DDL source
5. Look up each reference name in DDLDEPENDENCY → append to ET_REFERENCES

**SAP Tables**: DDDDLSRC, DDLDEPENDENCY
**SAP Classes**: `cl_dd_ddl_handler_factory`, `if_dd_ddl_handler_internal`
**Called by**: Phase 8 (main BFS loop), TestZCdsSqlViews.java

---

### Z_VIEW_DDL

**Purpose**: Extracts the HANA SQL CREATE VIEW definition by calling HANA system
procedure `SYS.GET_OBJECT_DEFINITION`. Returns the compiled, executable SQL.

**Function Group**: LZ_VIEW_DDL

**Interface**:
```
IMPORTING:  VIEW_NAME            TYPE CSMSYSGUID         " ABAP view name (uppercase)
EXPORTING:  LV_OBJECT_DEFINITION TYPE EHHSS_EXTERNAL_NOTE " Full CREATE VIEW SQL
EXCEPTIONS: 4 (not found), 8 (database error)
```

**ABAP Logic**:
1. Uppercase the input
2. Call HANA via ADBC:
   ```abap
   lo_sql = NEW cl_sql_statement( ).
   lv_sql = |CALL SYS.GET_OBJECT_DEFINITION(NULL, '{ view_name }')|.
   lo_result = lo_sql->execute_query( lv_sql ).
   ```
   NULL schema → default SAPHANADB. Column 5 (COL5) contains the DDL as a LOB.
3. Return first row's COL5

**SAP Procedures**: `SYS.GET_OBJECT_DEFINITION`
**SAP Classes**: `cl_sql_statement`, `cl_sql_result_set`
**Called by**: Phase 5, Phase 6, Phase 8

**Output example**:
```sql
CREATE VIEW "SAPHANADB"."IPRODUCT" ("MANDT", "PRODUCT", ...) AS (
  SELECT "MARA"."MANDT" AS "MANDT", "MARA"."MATNR" AS "PRODUCT", ...
  FROM "MARA" "MARA"
  LEFT OUTER MANY TO ONE JOIN "EPRODUCT" "=A0" ON (...)
  WHERE ("MARA"."MANDT" = SESSION_CONTEXT('CDS_CLIENT'))
) WITH READ ONLY
```

---

### Z_CDS_DEPENDENCIES

**Purpose**: Determines if a DDL source is SQL-only (no ABAP Dictionary entry) and
retrieves the HANA CDS source code line-by-line. Maps DDL names to ABAP names.

**Note**: No local ABAP file — maintained directly in SE37.

**Interface**:
```
IMPORTING:  I_DDLNAME  TYPE DDLNAME   " DDL source name
EXPORTING:  E_IS_ABAP  TYPE CHAR1     " 'X' if ABAP Dict entry exists, '' if SQL-only
            E_CDS_NAME TYPE STRING     " ABAP name (if available)
            E_MESSAGE  TYPE STRING     " Status/error message
            E_DDTEXT   TYPE STRING     " Description text from DD02T
TABLES:     ET_SOURCE  STRUCTURE SOLI  " HANA CDS source (255 chars/line)
```

**ABAP Logic** (inferred from Java callers):
1. Look up DDL name in DDDDLSRC / DDLDEPENDENCY
2. Check DD02L for ABAP Dict entry → set E_IS_ABAP
3. Retrieve HANA CDS source from `_SYS_REPO.ACTIVE_OBJECT` → split into ET_SOURCE lines
4. Get description from DD02T → E_DDTEXT

**Java usage (Phase 2 — identify SQL-only)**:
```java
function.getImportParameterList().setValue("I_DDLNAME", objectName);
function.execute(destination);
String eIsAbap = function.getExportParameterList().getString("E_IS_ABAP");
// Empty E_IS_ABAP → SQL-only view
```

**Java usage (Phase 6 — extract HANA CDS source)**:
```java
JCoTable etSource = function.getTableParameterList().getTable("ET_SOURCE");
StringBuilder hanaSource = new StringBuilder();
for (int i = 0; i < etSource.getNumRows(); i++) {
    etSource.setRow(i);
    hanaSource.append(etSource.getString("LINE"));
}
```

**SAP Tables**: DDDDLSRC, DDLDEPENDENCY, DD02L, DD02T, _SYS_REPO.ACTIVE_OBJECT
**Called by**: Phase 2, Phase 3, Phase 6

---

### Z_VIEW_GET_META

**Purpose**: Comprehensive metadata extractor. Returns dictionary header (DD02L),
description (DD02T), base table references (DD26S), column definitions (DD03L with
DDTEXT from DD04T).

**Interface**:
```
IMPORTING:  VIEW_NAME     TYPE VIEWNAME  " ABAP view/table name
EXPORTING:  DDL_STATEMENT TYPE STRING    " DDL (via cl_cds_ddl_extractor)
            METADATA      TYPE STRING    " Formatted metadata string
            DD02L         TYPE DD02L     " Table/view header
            DDLDEPENDENCY TYPE STRING    " Description text (confusingly named)
            DD02T         TYPE DD02T     " Table/view description
TABLES:     ET_DD26S      STRUCTURE DD26S " Base table references
            ET_COLUMNS    STRUCTURE DD03L " Column definitions with DDTEXT
```

**ABAP Logic**:
1. Get DDL via `cl_cds_ddl_extractor=>get_ddl( iv_name = view_name )`
2. `SELECT * FROM dd26s WHERE viewname = view_name AND as4local = 'A'`
3. `SELECT d3~*, d4~ddtext FROM dd03l AS d3 LEFT OUTER JOIN dd04t AS d4 ON d3~rollname = d4~rollname ... ORDER BY d3~position`
4. `SELECT SINGLE * FROM dd02l WHERE tabname = view_name AND as4local = 'A'`
   - Key: `DD02L-TABCLASS` = `TRANSP` (table), `VIEW` (view), `INTTAB` (structure)
5. `SELECT SINGLE * FROM dd02t WHERE tabname = view_name AND ddlanguage = 'E'`

**SAP Tables**: DD02L, DD02T, DD26S, DD03L, DD04T
**SAP Classes**: `cl_cds_ddl_extractor`
**Called by**: Phase 7, Phase 8 (fallback)

**Output files** (via Java callers): `{VIEW}_DD02L.csv`, `{VIEW}_DD02T.csv`, `{VIEW}_ET_DD26S.csv`, `{VIEW}_ET_COLUMNS.csv`, `{VIEW}_DDLDEPENDENCY.csv`

---

### Custom Table Type

| Type | Definition |
|------|-----------|
| ZDDLDEPENDENCY_TAB | Table type of DDLDEPENDENCY structure — must exist in SE11 before Z_CDS_SQL_VIEWS can activate |

---

## Common Operations

### Check connector logs on sapidess4
```bash
ssh root@sapidess4 "ls -la /usr/sap/cds_sql_only/files/"
```

### Compare datasets in GCS
```bash
gsutil ls gs://sap_cds_dbt/sap_cds_views/cds_sql_only_<N>/sql_only_resolution/
```

### Backup before changes
```bash
ssh root@sapidess4 "cp -r /usr/sap/cds_sql_only /usr/sap/cds_sql_only_backup_$(date +%Y%m%d)"
cp -r ~/cds_enhancement_20260210/github_release/src ~/cds_enhancement_20260210/github_release/src_backup_$(date +%Y%m%d)
```

### Full deploy cycle
1. Edit Java/Python locally or in /tmp
2. Upload to sapidess4
3. Compile Java on sapidess4
4. Sync local copy
5. Deploy connector (user does manually via Fivetran)

### Test RFC from Java
```bash
ssh root@sapidess4
cd /usr/sap/cds_sql_only/drivers
java -cp .:sapjco3.jar:gson.jar -Djava.library.path=. TestZCdsSqlViews <DDL_NAME>
```

### Test RFC from SE37 (SAP GUI)
1. Transaction SE37 → enter function module name → Execute (F8)
2. Fill import parameters (e.g. IV_DDLNAME = 'I_PRODUCT')
3. Execute → check export parameters and tables

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| "Java not found" in Fivetran | installation.sh didn't run or .env not loaded | Ensure connector.py loads drivers/.env and falls back to running installation.sh |
| Empty dataset (schema only, no data) | Java failed silently | Check if .class files exist, verify JCo libraries |
| Duplicate rows (N x 2) | Connector ran twice | Expected behavior — stop extra run in Fivetran |
| `ignore_association` all NO | BASEINFO not found in DDL | Check if DDL source is available for that view |
| Compilation error | Missing dependency class | Compile with full classpath: `-cp .:sapjco3.jar:gson.jar` |
| Z_CDS_SQL_VIEWS empty ET_REFERENCES | DDL name not in DDDDLSRC | Check spelling, try ABAP name instead |
| Z_VIEW_DDL returns empty | View has no HANA representation | Pure ABAP structure, not a real view |
| Z_CDS_DEPENDENCIES E_IS_ABAP = '' | Expected for SQL-only views | Not an error — this identifies SQL-only views |
| Z_VIEW_GET_META empty DD02L-TABNAME | No ABAP Dictionary entry | Object is SQL-only, use Z_CDS_SQL_VIEWS instead |
| BASEINFO missing from EV_SOURCE | `get_internals` not set | Verify Z_CDS_SQL_VIEWS uses `get_internals = abap_true` |
