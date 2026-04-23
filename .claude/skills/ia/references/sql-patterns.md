# iA SQL Patterns for execute_sql

**When to use these patterns:**
- **>100 rows expected** — Use `execute_sql` with these patterns instead of dedicated tools
- **Dedicated tool truncates** — If you hit a tool's limit, switch to `execute_sql` with the pattern below
- **No dedicated tool exists** — For queries not covered by `ia_*` tools

**How to use:** Copy the SQL, replace `${IA_LIBRARY}` with `IADEMODEV`, replace `<PARAM>` placeholders with actual values, adjust `FETCH FIRST N ROWS ONLY` as needed.

**Getting correct columns:** If a pattern below doesn't exist for your use case, check the dedicated tool's SQL in `impact-analysis.yaml` and adapt it for `execute_sql`.

---

## Core Analysis

### #1 Where-Used (find all objects referencing X)
**→ Use `ia_find_object_usages`**
```sql
SELECT I.IAOOBJNAM AS using_object, I.IAOOBJTYP AS using_type,
  I.IAOOBJLIB AS using_library, I.IAOOBJATR AS using_attr,
  I.IAOOBJTXT AS using_text, I.IARUSAGES AS usage_type,
  CASE WHEN M.MEMBER_NAME IS NULL THEN 'N' ELSE 'Y' END AS source_exist
FROM ${IA_LIBRARY}.IAALLREFPF I
LEFT JOIN ${IA_LIBRARY}.IAOBJMAP M
  ON I.IAOOBJNAM = M.OBJECT_NAME AND I.IAOOBJTYP = M.OBJECT_TYPE AND I.IAOOBJLIB = M.OBJECT_LIBR
WHERE I.IAROBJNAM = '<OBJECT_NAME>'
ORDER BY I.IAOOBJTYP, I.IAOOBJNAM
FETCH FIRST 200 ROWS ONLY
```

### #2 Call Hierarchy — Callees (what does X call)
**→ Use `ia_call_hierarchy` with `direction=CALLEES`**
```sql
SELECT MBR_NAME AS source_program, CALLED_OBJCT AS target_program,
  FROM_PROC AS calling_procedure, CALLED_PROC AS called_procedure, CALL_SEQ
FROM ${IA_LIBRARY}.IAPGMCALLS
WHERE MBR_NAME = '<PROGRAM_NAME>'
ORDER BY CALL_SEQ
FETCH FIRST 200 ROWS ONLY
```

### #3 Call Hierarchy — Callers (who calls X)
**→ Use `ia_call_hierarchy` with `direction=CALLERS`**
```sql
SELECT MBR_NAME AS calling_program, CALLED_OBJCT AS target,
  FROM_PROC AS calling_procedure, CALLED_PROC AS called_procedure, CALL_SEQ
FROM ${IA_LIBRARY}.IAPGMCALLS
WHERE CALLED_OBJCT = '<PROGRAM_NAME>'
ORDER BY MBR_NAME, CALL_SEQ
FETCH FIRST 200 ROWS ONLY
```

### #4 Field Impact (blast radius of changing a field)
**→ Use `ia_file_field_impact_analysis`** — returns `impact_type`: NEEDS_CHANGE (field in RPG or CL source), NEEDS_RECOMPILE (implicit access or *SRVPGM), STRUCTURAL (*FILE/*DSPF structural dependency)
**→ For full blast radius, also call `ia_file_dependencies` + `ia_find_object_usages` per LF (see QF-1)**
```sql
SELECT DISTINCT ref.OBJECT_NAME AS affected_object, ref.OBJECT_TYPE,
  ref.OBJECT_ATTR, ref.MAPED_FROM AS reference_type,
  fld.FIELD_DATA_TYPE AS field_type, fld.FIELD_LENGTH, fld.FIELD_DEC_POS,
  CASE
    WHEN ref.OBJECT_TYPE IN ('*FILE', '*DSPF') THEN 'STRUCTURAL'
    WHEN ref.OBJECT_TYPE = '*SRVPGM'           THEN 'NEEDS_RECOMPILE'
    WHEN src_rpg.MEMBER_NAME IS NOT NULL        THEN 'NEEDS_CHANGE'
    WHEN src_cl.MEMBER_NAME  IS NOT NULL        THEN 'NEEDS_CHANGE'
    ELSE 'NEEDS_RECOMPILE'
  END AS impact_type
FROM ${IA_LIBRARY}.IAALLREFPF ref
LEFT JOIN ${IA_LIBRARY}.IAFILEDTL fld
  ON fld.FILE_NAME = '<FILE_NAME>' AND fld.FIELD_NAME_INTERNAL = '<FIELD_NAME>'
LEFT JOIN (
  SELECT DISTINCT MEMBER_NAME FROM ${IA_LIBRARY}.IAQRPGSRC
  WHERE SOURCE_DATA LIKE '%<FIELD_NAME>%'
) src_rpg ON src_rpg.MEMBER_NAME = ref.OBJECT_NAME
  AND ref.OBJECT_ATTR IN ('RPGLE', 'SQLRPGLE', 'RPG', 'RPG38', 'RPGII', 'RPGILE')
LEFT JOIN (
  SELECT DISTINCT MEMBER_NAME FROM ${IA_LIBRARY}.IAQCLSRC
  WHERE SOURCE_DATA LIKE '%<FIELD_NAME>%'
) src_cl ON src_cl.MEMBER_NAME = ref.OBJECT_NAME
  AND ref.OBJECT_ATTR IN ('CLLE', 'CLE', 'CL', 'CL38')
WHERE ref.REFERENCED_OBJ = '<FILE_NAME>'
ORDER BY impact_type, ref.OBJECT_TYPE, ref.OBJECT_NAME
FETCH FIRST 500 ROWS ONLY
```

### #5 Program Variables
**→ Use `ia_program_variables`**
```sql
SELECT IAV_VAR AS variable_name, IAV_TYP AS variable_type,
  IAV_DTYP AS data_type, IAV_LEN AS field_length,
  IAV_DEC AS decimal_positions, IAV_DIM AS array_dimension
FROM ${IA_LIBRARY}.IAPGMVARS
WHERE IAV_MBR = '<MEMBER_NAME>'
ORDER BY IAV_VAR
FETCH FIRST 500 ROWS ONLY
```

### #6 Object Lookup (find type/library/attribute)
**→ Use `ia_object_lookup`**
```sql
SELECT DISTINCT IAROBJNAM AS object_name, IAROBJTYP AS object_type,
  IAROBJLIB AS object_library, IAROBJATR AS object_attribute
FROM ${IA_LIBRARY}.IAALLREFPF WHERE IAROBJNAM = '<OBJECT_NAME>'
UNION
SELECT DISTINCT IAOOBJNAM, IAOOBJTYP, IAOOBJLIB, IAOOBJATR
FROM ${IA_LIBRARY}.IAALLREFPF WHERE IAOOBJNAM = '<OBJECT_NAME>'
ORDER BY object_type, object_name
FETCH FIRST 200 ROWS ONLY
```

### #7 List Library Files
**→ Use `ia_library_files`**
```sql
SELECT TABLE_NAME AS file_name, TABLE_TYPE AS file_type,
  TABLE_TEXT AS description, LAST_ALTERED_TIMESTAMP AS last_altered
FROM QSYS2.SYSTABLES
WHERE TABLE_SCHEMA = '${IA_LIBRARY}'
ORDER BY TABLE_NAME
FETCH FIRST 500 ROWS ONLY
```

---

## Extended Analysis

### #8 Physical-to-Logical File Dependencies (IADSPDBR)
```sql
SELECT WHRTYP AS file_type, WHRFI AS file_name, WHRLI AS file_library,
  WHNO AS dependency_count, WHREFI AS dependent_file,
  WHRELI AS dependent_library, WHTYPE AS relation_type, WHCSTN AS constraint_name
FROM ${IA_LIBRARY}.IADSPDBR
WHERE WHRFI = '<FILE_NAME>' AND WHREFI <> ''
ORDER BY WHREFI
FETCH FIRST 200 ROWS ONLY
```
WHTYPE codes: D=Data, I=Access path, O=Owner, V=SQL VIEW, C=Constraint

### #9 Program Source Location (IAOBJMAP)
```sql
SELECT OBJECT_NAME, OBJECT_TYPE, OBJECT_ATTR,
  MEMBER_LIBR, MEMBER_SRCF, MEMBER_NAME, MEMBER_TYPE
FROM ${IA_LIBRARY}.IAOBJMAP
WHERE OBJECT_NAME = '<PROGRAM_NAME>'
ORDER BY OBJECT_TYPE
FETCH FIRST 50 ROWS ONLY
```

### #10 Program-to-File Mapping
```sql
SELECT MEMBER_NAME, MEMBER_TYPE, ACTUAL_FILE, RENAMED_FILE,
  ACTUAL_RCDFMT_NAME, PREFIX, QUALIFIED_LIBRARY
FROM ${IA_LIBRARY}.PROGRAM_TO_FILE_MAPPING_DETAILS
WHERE MEMBER_NAME = '<MEMBER_NAME>'
ORDER BY ACTUAL_FILE
FETCH FIRST 200 ROWS ONLY
```

### #11 Copybook Impact (who includes copybook X)
```sql
SELECT MEMBER_LIBRARY, MEMBER_SOURCE_FILE, MEMBER_NAME, MEMBER_TYPE,
  MEMBER_RRN_NUMBER, COPYBOOK_MEMBER_LIBRARY, COPYBOOK_MEMBER_NAME
FROM ${IA_LIBRARY}.COPYBOOK_MEMBER_DETAIL
WHERE COPYBOOK_MEMBER_NAME = '<COPYBOOK_NAME>'
ORDER BY MEMBER_NAME
FETCH FIRST 200 ROWS ONLY
```

### #12 Service Program Exports
```sql
SELECT OBJECT_NAME, OBJECT_LIBRARY, OBJECT_TYPE, OBJECT_TEXT,
  PROCEDURE_NAME, PROCEDURE_TYPE
FROM ${IA_LIBRARY}.IMPORTED_EXPORTED_PROCEDURE_DETAILS
WHERE OBJECT_NAME = '<SRVPGM_NAME>' AND PROCEDURE_TYPE = 'EXPORT'
ORDER BY PROCEDURE_NAME
FETCH FIRST 200 ROWS ONLY
```

### #13 Procedure-Level Call Graph
```sql
SELECT OBJECT_NAME, PROCEDURE_NAME, REFERENCED_OBJECT_NAME,
  REFERENCED_PROCEDURE_NAME, REFERENCED_OBJECT_TYPE
FROM ${IA_LIBRARY}.PROCEDURE_REFERENCE_DETAIL
WHERE OBJECT_NAME = '<PROGRAM_NAME>'
ORDER BY PROCEDURE_NAME, REFERENCED_PROCEDURE_NAME
FETCH FIRST 500 ROWS ONLY
```

### #14 Program Compile Status (source vs compile date)
**→ `ia_program_info` returns the core metadata; use this SQL for the full compile comparison**
```sql
SELECT PGM_NAME, PGM_LIB, PGM_TYP, MOD_NAME, MOD_ATTRIB,
  SRC_FILE, SRC_LIB, SRC_MBR, MOD_CRTDATE, SRC_UPDDATE, TGT_RLS
FROM ${IA_LIBRARY}.IAPGMINF
WHERE PGM_NAME = '<PROGRAM_NAME>'
ORDER BY MOD_TYP, MOD_NAME
FETCH FIRST 100 ROWS ONLY
```
If SRC_UPDDATE > MOD_CRTDATE, program needs recompilation.

### #15 Dead Code Candidates (never-used objects)
**→ Use `ia_unused_objects` for a faster lookup; this query uses the richer IA_OBJECT_USAGE_SUMMARY_DETAIL view**
```sql
SELECT OBJECT_NAME, OBJECT_LIBRARY, OBJECT_TYPE, OBJECT_ATTRIBUTE,
  OBJECT_SIZE, OBJECT_TEXT, OBJECT_USAGE_CATEGORY
FROM ${IA_LIBRARY}.IA_OBJECT_USAGE_SUMMARY_DETAIL
WHERE OBJECT_USAGE_CATEGORY = 'Never'
ORDER BY OBJECT_SIZE DESC
FETCH FIRST 200 ROWS ONLY
```

---

## Source & Documentation

### #16 RPG Source Text
**→ Prefer #18 (combined source + complexity) for full source retrieval**
```sql
SELECT SOURCE_RRN, SOURCE_DATA
FROM ${IA_LIBRARY}.IAQRPGSRC
WHERE MEMBER_NAME = '<MEMBER_NAME>'
ORDER BY LIBRARY_NAME, SOURCE_RRN
FETCH FIRST 500 ROWS ONLY
```

### #18 Combined RPG Source + Complexity (PREFERRED for full source retrieval)

**Use this instead of calling `ia_rpg_source` + `ia_code_complexity` separately.**

```sql
SELECT R.LIBRARY_NAME, R.SOURCEPF_NAME, R.MEMBER_NAME, R.MEMBER_TYPE,
  R.SOURCE_RRN, R.SOURCE_SEQ, R.SOURCE_SPEC, R.SRCLIN_TYPE, R.SOURCE_DATA,
  C.TOTAL_LINES, C.EXEC_LINES, C.IF_COUNT, C.DO_COUNT, C.FOR_COUNT,
  C.SEL_COUNT, C.SBR_COUNT, C.PROC_COUNT, C.SQL_COUNT, C.GOTO_COUNT,
  C.FILE_COUNT, C.DEVF_COUNT, C.CALL_PGM_COUNT, C.CALLED_BY_PGM
FROM ${IA_LIBRARY}.IAQRPGSRC R
LEFT JOIN ${IA_LIBRARY}.IA_CODE_INFO C ON C.MEMBER_NAME = R.MEMBER_NAME
WHERE R.MEMBER_NAME = '<MEMBER_NAME>'
ORDER BY R.LIBRARY_NAME, R.SOURCE_RRN
FETCH FIRST 10000 ROWS ONLY
```

**Why this is better:**
- Gets source code AND complexity metrics in ONE query
- Higher limit (10000) avoids pagination for most programs
- No need for separate `ia_code_complexity` call to get TOTAL_LINES

**For spec-type filtering:**
```sql
SELECT R.SOURCE_RRN, R.SOURCE_SPEC, R.SOURCE_DATA
FROM ${IA_LIBRARY}.IAQRPGSRC R
WHERE R.MEMBER_NAME = '<MEMBER_NAME>'
  AND R.SOURCE_SPEC = 'P'  -- P=procedures, D=definitions, F=files, C=calc
ORDER BY R.LIBRARY_NAME, R.SOURCE_RRN
FETCH FIRST 5000 ROWS ONLY
```

### #17 CL Source Text
```sql
SELECT SOURCE_RRN, SOURCE_DATA
FROM ${IA_LIBRARY}.IAQCLSRC
WHERE MEMBER_NAME = '<MEMBER_NAME>'
ORDER BY LIBRARY_NAME, SOURCE_RRN
FETCH FIRST 500 ROWS ONLY
```

### #19 Pseudocode Summary
```sql
SELECT SOURCE_RRN, DOCUMENT_SEQ, GENERATED_PSEUDOCODE
FROM ${IA_LIBRARY}.GENERATED_PSEUDOCODE_DETAILS
WHERE MEMBER_NAME = '<MEMBER_NAME>'
ORDER BY SOURCE_RRN, DOCUMENT_SEQ
FETCH FIRST 500 ROWS ONLY
```

### #20 Who Created an Object?
**Use `OBJECT_DETAILS.CREATED_BY_USER`** — not `IA_CODE_INFO.CREATED_BY` (that's iA scan metadata)
```sql
SELECT OBJECT_NAME, OBJECT_TYPE, OBJECT_ATTRIBUTE, CREATED_BY_USER,
       CREATION_DATE, CREATION_TIME, OBJECT_TEXT
FROM ${IA_LIBRARY}.OBJECT_DETAILS
WHERE OBJECT_NAME = '<OBJECT_NAME>'
```

### #21 List All Developers by Objects Created
```sql
SELECT CREATED_BY_USER AS DEVELOPER, COUNT(*) AS OBJECTS_CREATED
FROM ${IA_LIBRARY}.OBJECT_DETAILS
WHERE CREATED_BY_USER <> '' AND CREATED_BY_USER <> '*NONE'
GROUP BY CREATED_BY_USER
ORDER BY OBJECTS_CREATED DESC
FETCH FIRST 50 ROWS ONLY
```
