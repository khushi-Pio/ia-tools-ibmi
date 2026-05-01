# iA Agent Playbook

**Efficiency goal:** Answer most questions in 1-2 tool calls. See [query-flows.md](query-flows.md) for optimal sequences.

**Tool preference:** For queries ≤100 rows, use dedicated `ia_*` tools. **For queries >100 rows, use `execute_sql` directly** — dedicated tools silently truncate.

**Getting SQL:** Use [sql-patterns.md](sql-patterns.md) templates. If no pattern exists, read the tool's SQL from `impact-analysis.yaml` and adapt it.

## IBM i Object Types

| Type | What It Is | Impact Significance |
|------|-----------|---------------------|
| `*PGM` | Compiled program (RPG, COBOL, CL) | Direct — may break if referenced object changes |
| `*SRVPGM` | Service program (shared logic) | **Amplifier** — many programs bind to it, changes cascade widely |
| `*FILE` | Physical file (table) or logical file (view/index) | Foundation — field changes affect all externally-described programs |
| `*DSPF` | Display file (5250 screen) | **User-facing** — changes visible on screens immediately |
| `*CMD` | Command object | May be called from CL, menus, or job schedulers beyond iA tracking |

## Query Chaining Strategy

| What You See in Results | Next Tool (fallback SQL#) | Why |
|------------------------|---------------------------|-----|
| `*PGM` or `*SRVPGM` names | `ia_call_hierarchy` (SQL #2/#3) | Understand where they sit in execution chain |
| `*SRVPGM` specifically | `ia_find_object_usages` on that SRVPGM (SQL #1) | Measure cascade — how many programs bind to it |
| Many refs, just need count | `ia_reference_count` | Lightweight tally grouped by type |
| `*FILE` names | `ia_file_field_impact_analysis` for specific fields (SQL #4) | Drill into field-level dependencies |
| `*FILE` (physical) | `execute_sql` on IADSPDBR (SQL #8) | Find logical files/views built over it |
| `*DSPF` names | `ia_find_object_usages` on display file (SQL #1) | Find which programs present that screen |
| Programs with null field_usage | `ia_program_variables` (SQL #5) | Confirm whether they actually use the field |
| Variable names matching DB fields | `execute_sql` on PROGRAM_TO_FILE_MAPPING_DETAILS (SQL #10) | Cross-check which files the program references |
| Unknown internal structure | `ia_data_structures`, `ia_subroutines` | Inspect DS layouts and subroutine usage |
| Suspect file routing | `ia_file_overrides`, `ia_override_chain` | Detect OVRDBF redirection |
| Stale / stale-looking object | `ia_object_lifecycle` | Check last-used date and days-used count |
| Large or rarely-used objects (capacity/cleanup) | `ia_obj_size` | Rank by size, filter by usage_category (Never/Rare) |
| High-risk program to refactor | `ia_code_complexity` | Get IF/DO/SQL/GOTO counts and line totals |
| Circular call suspicion | `ia_circular_deps` | Detect two-way call pairs |
| Cleanup candidates | `ia_unused_objects` | Objects with zero references |
| Results hit the limit | **Use `execute_sql`** with [sql-patterns.md](sql-patterns.md) | Avoid truncation |

## Chaining Rules

1. **Start broad, then narrow.** SQL#1 (where-used) is the broadest query. Start there for general questions.
2. **Object type determines the next query.** Let `using_type` / `object_type` guide your next step.
3. **Service programs are amplifiers.** Any `*SRVPGM` in results — offer to check what depends on it.
4. **Don't run everything at once.** Run 1-2 queries, interpret, then decide next step based on results.
5. **Respect the limit.** If results are truncated, tell the user and offer to increase it.
6. **Source code: use execute_sql.** For source retrieval, use `execute_sql` with SQL #18 — it returns source + complexity in one query with 10000-row limit. Reserve `ia_object_lookup`/`ia_member_lookup` for metadata validation only.

## Pagination Strategy

**For source code:** Use `execute_sql` with SQL pattern #18 (limit 10000) — this covers 99% of programs without pagination. Only paginate if source exceeds 10000 lines.

**For other tools:** When result count equals the limit, data is likely truncated. Offer to increase limit or switch to `execute_sql`.

**Pro tip:** Use spec-type filtering to reduce data: add `SOURCE_SPEC = 'P'` for procedures only, `SOURCE_SPEC = 'D'` for definitions.

## Playbooks

### P1: "What references X?" (Where-Used)
Call `ia_find_object_usages` → Group by `using_type`, count each → Flag `*SRVPGM` (cascade risk) and `*DSPF` (user-facing) → Ask: "Are you modifying, deleting, or just understanding X?" → For a quick tally, use `ia_reference_count` instead.

### P2: "What if I change field F in file X?" (Field Impact)
**Always run all 3 steps, then synthesize:**
1. `ia_file_field_impact_analysis(file_name=X, field_name=F)` → direct PF references: *PGM, *SRVPGM, *DSPF, *FILE classified by impact_type
2. `ia_file_dependencies(file_name=X)` → LFs / indexes / views over PF (STRUCTURAL — must be rebuilt)
3. `ia_find_object_usages` on each LF found in step 2 → programs using those LFs (also need change/recompile)

Categorize by `impact_type`: NEEDS_CHANGE=source edit required (field name explicit in RPG or CL source), NEEDS_RECOMPILE=recompile only (implicit record-format or *SRVPGM), STRUCTURAL=rebuild/recreate (*FILE or *DSPF inheriting field via DDS REF).
Quantify: "N programs need code changes, M need recompile, K objects are structural dependencies."
**Notes:** `*SRVPGM` always shows NEEDS_RECOMPILE (source check by SRVPGM name is unreliable). CL programs are now checked via IAQCLSRC — those with field in source correctly show NEEDS_CHANGE.
Ask: "Resizing, renaming, or retyping? Each has different implications."

### P3: "Show call tree for program X" (Call Hierarchy)
Call `ia_call_hierarchy` with `direction=BOTH` → Present in two sections: CALLERS / CALLEES → Flag: hub (>5 callers), zero callers (may be scheduler-invoked), deep chains → For parameter inspection at call sites, chain `ia_call_parameters`.

### P4: "What variables does program X use?"
Call `ia_program_variables` → Group: standalone fields, DS subfields (likely DB field refs), indicators, arrays → Flag variable names matching DB patterns (short uppercase like CUSTNO, ORDNUM) → For full DS layouts, chain `ia_data_structures`.

### P5: "Retire/delete X" (Full Retirement)
`ia_find_object_usages` for all refs → If ANY refs exist, warn immediately → `ia_call_hierarchy` on key programs to assess depth → `ia_file_field_impact_analysis` on critical fields → Synthesize risk: "HIGH/MODERATE/LOW — N objects, M service programs, D display files."

### P6: "Is it safe to modify X?" (Safety Assessment)
`ia_find_object_usages` or `ia_file_field_impact_analysis` depending on object vs field → Count refs → Apply risk rubric → Check for `*SRVPGM` amplifiers → For code-level risk, add `ia_code_complexity` → State verdict explicitly with evidence.

### P7: "Logical files over physical X?" (File Dependencies)
`execute_sql` on IADSPDBR (SQL#8) → Interpret WHTYPE: D=data, I=access path, V=SQL VIEW, C=constraint → Chain `ia_find_object_usages` on each logical file to find programs referencing them.

### P8: "Find dead code"
**Compiled objects:** `ia_unused_objects` for objects with zero refs → `ia_object_lifecycle` to confirm last-used date → `ia_dashboard` to cross-check category → Always flag members that may be scheduler-invoked.
**Orphaned sources:** `ia_uncompiled_sources` for source members never compiled into objects → Check LAST_CHANGED date to identify abandoned development.

### P9: "Where does this program actually write?"
`ia_file_overrides` for the member → If overrides exist, also run `ia_override_chain` → Combine with `ia_find_object_usages` on each resolved target file to confirm downstream impact.

### P10: "What programs include this copybook?" (Copybook Impact)
`ia_copybook_impact(copybook_name="CUSTDS")` → Returns all members with /COPY directive + line numbers → Group by member_type (RPGLE, SQLRPGLE, CBLLE) → Flag: high count = high-risk copybook change.

### P11: "What does this service program export?" (API Surface)
`ia_srvpgm_exports(object_name="MYSRVPGM", procedure_type="EXPORT")` → Lists all exported procedures → Chain `ia_procedure_xref` on specific procedures to find callers → For parameter signatures, use `ia_procedure_params`.

### P12: "What procedures call procedure X?" (Procedure-Level Impact)
`ia_procedure_xref(procedure_name="PROCESSORDER", direction="CALLERS")` → More granular than ia_call_hierarchy → Shows procedure-to-procedure relationships → Critical for ILE modular code.

### P13: "Find batch jobs and scheduler calls" (Job Detection)
`ia_cl_jobs(call_type="SBMJOB")` → Returns SBMJOB calls with job queue, hold flag → Critical: programs appearing "unused" may be scheduler-invoked → Cross-reference with ia_unused_objects to avoid false positives.

### P14: "What files does program X use (with prefixes)?"
`ia_program_files(member_name="ORDENTRY", library="AIDEMOLIB")` → Shows files with PREFIX, RENAME, record format details → Use `library` param to scope to a specific library when the same member name exists in multiple libraries → More detailed than ia_find_object_usages for understanding file access patterns.

### P15: "Scope analysis to a project area"
`ia_application_area(area_name="*LIST")` → List all defined areas → Then `ia_application_area(area_name="MYPROJECT")` → Returns objects in that area for scoped analysis.

### P16: "Resolve SQL long name to system name"
`ia_sql_names(name_pattern="STORE%")` → Maps SQL long names ↔ 10-char system names → Essential for SQL procedure/function analysis.

## Risk Rubric

| References | Risk | Guidance |
|-----------|------|---------|
| 0 | Safe | No code dependencies — but verify job schedulers and external systems |
| 1–5 | Low | Limited blast radius — list specific objects that need testing |
| 6–20 | Moderate | Review each affected object before proceeding |
| 20+ | High | Wide blast radius — recommend phased approach with regression testing |

## Follow-Up Patterns

| Result Pattern | Offer |
|---------------|-------|
| `*SRVPGM` in results | "SRVPGM X is shared — check what binds to it?" |
| `*DSPF` in results | "Display file X is user-facing — check which programs use it?" |
| >10 affected objects | "N objects affected — narrow by type or investigate critical ones first?" |
| Null `field_usage` | "N programs have unknown field usage — inspect variables with SQL#5?" |
| Program with >5 callers | "X has N callers — critical junction. Check what files it references?" |
| Program with 0 callers | "X has no callers in repository — may be invoked by scheduler or external system." |
| Results hit limit | "Capped at N rows — switching to execute_sql for full results." |
| Zero results | "No results — verify uppercase spelling. If correct, check schedulers/external systems." |

## Response Rules

1. **Never dump raw tables.** Summarize first: "X is referenced by 15 programs, 3 service programs, 4 display files."
2. **Group by object type.** This is how IBM i developers think about dependencies.
3. **State risk explicitly.** "This is a high-risk change because..." — don't just show data.
4. **Use IBM i terms.** Say "physical file" not "table", "service program" not "shared library" — unless user uses SQL terms.
5. **Always suggest a concrete next step.** Every response ends with a specific follow-up action.

## Tool Efficiency Rules

1. **One query is usually enough.** Before calling a second tool, ask: "Can I answer this from results I already have?"
2. **Use `ia_program_detail` for program anatomy.** It returns calls, files, subroutines, variables, overrides, and call parameters in ONE query — no need for 6 separate calls.
3. **Don't chain redundantly.** `ia_find_object_usages` already gives you everything; don't follow up with `ia_reference_count` on the same object.
4. **Skip intermediate steps.** Don't call `ia_member_lookup` just to get location before `ia_rpg_source_tokens` — go straight to the token analysis.
5. **Batch your thinking.** If results show 5 SRVPGMs, don't call `ia_find_object_usages` on each one individually — ask the user which ones matter first.
6. **Respect the 80/20 rule.** 80% of user questions can be answered with these tools in 1 call:
   - `ia_find_object_usages` — what uses X?
   - `ia_file_field_impact_analysis` — what if I change field F?
   - `ia_call_hierarchy` — call tree for X?
   - `ia_program_detail` — tell me about program X?
   - `ia_code_complexity` — complexity hotspots?
   - `ia_unused_objects` — dead compiled objects?
   - `ia_uncompiled_sources` — orphaned sources?
   - `ia_object_lookup` — find object named X?
   - `ia_dashboard` — repository overview?
   - `ia_copybook_impact` — what includes copybook X?
   - `ia_srvpgm_exports` — what does SRVPGM X export?
   - `ia_procedure_xref` — what calls procedure X?
   - `ia_cl_jobs` — batch job detection?
