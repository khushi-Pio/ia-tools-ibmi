# iA Repository Tables Catalog

For full column details on any table, run:
`describe_sql_object(object_name='TABLE_NAME', schema_name='${IA_LIBRARY}')`

---

## 1. Cross-Reference & Dependencies

| Table | Purpose | Key Columns | Query When |
|-------|---------|-------------|------------|
| `IAALLREFPF` | Master cross-reference: object-to-object relationships | IAOOBJNAM, IAOOBJTYP, IAROBJNAM, IAROBJTYP, IARUSAGES | Where-used, field impact, object lookup |
| `IAPGMCALLS` | Call graph: CALL, CALLP, bound module refs | MBR_NAME, CALLED_OBJCT, FROM_PROC, CALLED_PROC, CALL_SEQ | Call hierarchy (callers/callees) |
| `IADSPDBR` | Physical↔logical file dependencies (DSPDBR output) | WHRFI, WHREFI, WHTYPE(D/I/O/V/C), WHNO | Find logicals over a physical file, constraints |
| `IAOBJREFPF` | Raw DSPPGMREF output: program→object refs with I/O usage | WHPNAM, WHFNAM, WHFUSG(1-8), WHOBJT(F/P/D), WHRFNM | File usage analysis (I/O/U), record format tracking |
| `IAFILEDTL` | Field-level details for database files | MEMBER_NAME, FIELD_NAME_INTERNAL, FIELD_DATA_TYPE, FIELD_USE_MODE | Field impact joins, field type lookups |

## 2. Program & Object Metadata

| Table | Purpose | Key Columns | Query When |
|-------|---------|-------------|------------|
| `IAPGMINF` | Program/SRVPGM description: modules, source, compile dates | PGM_NAME, MOD_NAME, SRC_FILE, SRC_MBR, MOD_CRTDATE, SRC_UPDDATE, TGT_RLS | Source location, compile status, module composition |
| `IAOBJMAP` | Compiled object → source member mapping | OBJECT_NAME, OBJECT_TYPE, MEMBER_LIBR, MEMBER_SRCF, MEMBER_NAME, MEMBER_TYPE | Find source for a compiled object |
| `OBJECT_DETAILS` | Full object inventory (DSPOBJD-level) | OBJECT_NAME, OBJECT_TYPE, OBJECT_SIZE, LASTUSED_DATE, DAYS_USED_COUNT, NO_OF_DEPEND | Object metadata, stale objects, size analysis |
| `IAOBJUSGDP` (IA_OBJECT_USAGE_SUMMARY_DETAIL) | Object size + usage summary with category buckets | OBJECT_NAME, OBJECT_LIBRARY, OBJECT_TYPE, OBJECT_SIZE, OBJECT_USAGE_CATEGORY (Never/Rare/…), DAYS_USED_COUNT, LAST_USED_DATE | Capacity planning, cleanup candidates — used by `ia_obj_size` |
| `IASRCMBRID` | Source member registry with surrogate IDs and code metrics | MBR_SUR_ID, MEMBER_NAME, MEMBER_TYPE, MBR_TTL_LOC, MBR_CMT_LOC | Member ID resolution, code size metrics |
| `MEMBERLIST_DETAILS` (sys: IAMEMBER) | Source member inventory (census of all members) | MEMBER_NAME, SOURCE_FILE, MEMBER_TYPE, NUMBER_OF_RECORD, MBR_CHANGED_DATE | Member existence check, inventory, stale member detection (use `ia_member_lookup`) |

## 3. Variables & Data Flow

| Table | Purpose | Key Columns | Query When |
|-------|---------|-------------|------------|
| `IAPGMVARS` | All variables declared in a program | IAV_MBR, IAV_VAR, IAV_TYP, IAV_DTYP, IAV_LEN, IAV_DIM | Variable inspection, field mapping confirmation |
| `IAVARREL` | Variable operations: assignments, comparisons, BIF calls | MEMBER_NAME, OPCODE_NAME, RESULT_VAL, FACTOR1_VAL, FACTOR2_VAL, BIF | Opcode-level variable analysis, data flow |
| `IAVARTRK` | Variable flow tracking: source→destination at each line | SRC_MBR_ID, SOURCE_RRN, VAR_FRM_ID, VAR_TO_ID, VAR_LEVEL | Data flow tracing (join with IASRCVARID for names) |
| `IASRCVARID` | Variable ID registry: maps surrogate IDs to names | VAR_SUR_ID, VARIABLE_NM, VARIABLE_TYP, VARIABLE_LEN | Resolve variable IDs from IAVARTRK to actual names |
| `IAPGMDS` | Data structure field details (system name: IACPGMREF) | MEMBER_NAME, DATASTR_NAME, DATASTR_TYPE, DS_FLD_NAME, DS_FLD_TYPE, DS_FLD_SIZE | DS inspection, externally-described file mapping |

## 4. Procedures & Parameters

| Table | Purpose | Key Columns | Query When |
|-------|---------|-------------|------------|
| `IAPRCPARM` | Procedure parameter details (PR/PI declarations) | MEMBER_NAME, PROCEDURE_NAME, PR_PI_FLAG, PARAMETER_NAME, PARM_DATA_TYPE, KEYWORDS | Procedure signatures, API analysis |
| `IAENTPRM` | Entry parameter definitions | MEMBER_NAME, PROC_NAME, PARM_SEQ, PARM_NAME | Procedure entry parameters |
| `IACALLPARM` | Call parameter details: params passed at each call site | MBR_NAME, CALLED_OBJECT, CALL_SEQ, PARM_NAME, PARM_TYPE, PARM_LENGTH | Parameter-level call analysis, signature mismatch |
| `PROCEDURE_REFERENCE_DETAIL` | Procedure-level cross-reference (who calls which procedure) | OBJECT_NAME, PROCEDURE_NAME, REFERENCED_OBJECT_NAME, REFERENCED_PROCEDURE_NAME | Procedure-level call graph |
| `PROCEDURE_INFORMATION` | PR/PI declarations at source level with line numbers | MEMBER_NAME, PROCEDURE_NAME, PROCEDURE_TYPE, RRN_NUMBER, EXPORT_OR_IMPORT | Jump-to-declaration, exported API surface |
| `IMPORTED_EXPORTED_PROCEDURE_DETAILS` | Object-level exports/imports for SRVPGMs | OBJECT_NAME, PROCEDURE_NAME, PROCEDURE_TYPE(EXPORT/IMPORT) | Service program API surface discovery |

## 5. Source Code

| Table | Purpose | Key Columns | Query When |
|-------|---------|-------------|------------|
| `IAQRPGSRC` | Full RPG/SQLRPGLE source text, line-by-line | MEMBER_NAME, SOURCE_RRN, SOURCE_DATA, SRCLIN_TYPE, SOURCE_SPEC | Read RPG source, search for keywords |
| `IAQCLSRC` | Full CL/CLLE source text, line-by-line | MEMBER_NAME, SOURCE_RRN, SOURCE_DATA | Read CL source, find CALL/OVRDBF commands |
| `IAQCBLSRC` | Full COBOL source text (if COBOL present) | MEMBER_NAME, SOURCE_RRN, SOURCE_DATA | Read COBOL source |
| `IAQDDSSRC` | DDS source (PF/LF/DSPF/PRTF), line-by-line | MEMBER_NAME, MEMBER_TYPE, SOURCE_RRN, SOURCE_DATA | Read DDS definitions, find field/key declarations |

## 6. Program Structure

| Table | Purpose | Key Columns | Query When |
|-------|---------|-------------|------------|
| `IASUBRDTL` | Subroutine details: EXSR/BEGSR/ENDSR with line positions | MBR_NAME, SR_OPCODE, SR_SBR_NAME, MBR_RRN, SR_CNT | Internal program structure, dead subroutine detection |
| `IAPGMKLIST` | K-list (key list) definitions and key fields | MEMBER_NAME, KLIST_NAME, KFLD_NAME | Key field usage, CHAIN/SETLL access patterns |
| `IAPRFXDTL` | Field prefix mapping: original→prefixed field names | MBR_NAME, FILENAM, OLD_FLD_NAME, NEW_FLD_NAME | Resolve prefixed field references |
| `IAOVRPF` | File overrides (OVRDBF/OVRPRTF) from CL source | IOVR_MBR, IOVR_FROM, IOVR_TO, IOVR_SCP, IOVR_PGM | Runtime file redirection detection |
| `COPYBOOK_MEMBER_DETAIL` | /COPY dependency tracking (sys: IACPYBDTL) | MEMBER_NAME, COPYBOOK_MEMBER_NAME, MEMBER_RRN_NUMBER | Copybook impact: who includes what |

## 7. Inventory & Usage

| Table | Purpose | Key Columns | Query When |
|-------|---------|-------------|------------|
| `IA_DASHBOARD_DETAIL` | Member/object summary with UNUSED/COMPILED category | MEMBER_NAME, MEMBER_CATEGORY, MEMBER_LINES, OBJECT_SIZE, DAYS_USED_COUNT | Complexity hotspots, unused member cleanup |
| `IA_OBJECT_USAGE_SUMMARY_DETAIL` | Usage classification: Never/Rare/Regular | OBJECT_NAME, OBJECT_TYPE, OBJECT_USAGE_CATEGORY, LAST_USED_DATE | Dead code discovery, cleanup candidates |
| `IAEXCOBJS` | Objects excluded from iA analysis (system APIs) | OBJECT_NAME, OBJECT_TYPE, LIBRARY_NAME | Filter system objects from analysis results |
| `IAVARCAL` | CL variable calls with job scheduling details | IVCAL_MBR, IVCAL_VARM, IVCAL_JOBQ, IVCAL_HOLD | CL call graph, batch job detection |
| `IAMODINF` | Module info / procedure list (imported/exported) | MOD_NAME, PROC_NAME, PROC_TYPE(IMP/EXP) | Module procedure inventory |

## 8. Application Areas & Modernization

| Table | Purpose | Key Columns | Query When |
|-------|---------|-------------|------------|
| `APPLICATION_AREA_HEADER` | Named project scopes (sys: IAAPPAHDRP) | APP_AREA_NAME, APP_AREA_DESC, REPO_NAME | List application areas |
| `APPLICATION_AREA_OBJECT_DETAILS` | Objects in each app area (sys: IAAPPAOBJL) | APP_AREA_NAME, OBJECT_NAME, OBJECT_TYPE | Scope analysis to a specific project area |
| `APPLICATION_AREA_RULES` | Filter rules defining area membership (sys: IAAPPRULEP) | APP_AREA_NAME, RULE_SEQ_NO, OBJECT_NAME, OBJECT_NAME_CONDITION | Understand area inclusion rules |
| `IA_LONG_SHORT_NAME` | SQL long↔short name mapping (sys: IALNGNMDTL) | ROUTINE_NAME, EXTERNAL_NAME, SPECIFIC_NAME, LANGUAGE | Resolve cryptic 10-char system names to SQL long names |
| `DDSTODDL_FILE_CONVERSION_DETAILS` | DDS→DDL conversion tracking (sys: IADDSDDLDP) | DDS_OBJECT_NAME, DDL_OBJECT_NAME, OBJ_CONV_STATUS(S/F/P) | Modernization tracking, conversion status |
| `IAPGMREF` | Source word-level index (tokenized RPG source) | MEMBER_NAME, SOURCE_RRN, SOURCE_WORD, WRD_OBJECTV | Cross-member keyword search, opcode analysis |
