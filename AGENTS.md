# AGENTS.md — AI Agent Playbook for iA Impact Analysis Tools

This file teaches AI coding agents (VS Code Copilot, Claude Code, Cursor, Windsurf, or any MCP-compatible agent) how to use the 4 iA MCP tools effectively. The goal: chain tools intelligently, interpret results like a senior IBM i developer, and ask the right follow-up questions — not just return raw data.

---

## IBM i Quick Reference for AI Agents

You need this context to interpret tool results correctly.

### Object Types and Impact Significance

| Type | What It Is | Why It Matters for Impact Analysis |
|------|-----------|-----------------------------------|
| `*PGM` | Compiled program (RPG, COBOL, CL) | Direct impact — if it references a changed object, it may break or need recompilation |
| `*SRVPGM` | Service program (shared business logic) | **Amplifier** — many programs bind to it, so a change here cascades widely. Always investigate further |
| `*FILE` | Physical file (DB table) or logical file (view/index) | Foundation of data. Changing a field here affects every program using externally-described file definitions |
| `*DSPF` | Display file (5250 screen definition) | **User-facing** — if this references a changed field, end users see the impact directly on their screens |
| `*CMD` | Command object | May be called from CL programs, menus, or job schedulers — impact may extend beyond what iA tracks |

### Key IBM i Concepts

- **Object names are 10-character uppercase** (e.g., `CUSTMAST`, `ORDERPGM`, `CUSTNO`).
- **Externally-described files**: RPG programs declare file record formats at compile time. If you change a field in the physical file, every program using that file's record format is affected — even if the program source code doesn't mention the field by name.
- **Service programs are shared libraries**: Dozens of programs may bind to one `*SRVPGM`. A single service program change has a multiplier effect.
- **Physical file vs. logical file**: A physical file holds the data (like a table). Logical files are views/indexes over physical files. Changing a physical file can break its logical files and everything that uses them.

---

## Tool Chaining Strategy

### Which Tool to Start With

| User's Intent | Start With | Why |
|--------------|-----------|-----|
| "What references X?" / "What uses X?" | `ia_where_used` | Broadest view — shows all objects depending on X |
| "What if I change field F in file X?" | `ia_field_impact` | Goes straight to field-level blast radius |
| "Show the call tree for program X" | `ia_call_hierarchy` | Directly answers the call flow question |
| "What variables does program X use?" | `ia_program_variables` | Directly answers the variable inspection question |
| "Can I safely delete/modify X?" | `ia_where_used` | Start broad, then drill down based on results |
| "I want to retire/decommission X" | `ia_where_used` | Need the full dependency picture first |

### What to Chain Next (Based on Results)

| What You See in Results | Next Tool to Offer | Why |
|------------------------|-------------------|-----|
| `*PGM` or `*SRVPGM` names | `ia_call_hierarchy` on those programs | Understand where they sit in the execution chain |
| `*SRVPGM` specifically | `ia_where_used` on that service program | Measure the cascade — how many programs bind to it |
| `*FILE` names | `ia_field_impact` for specific fields | Drill into field-level dependencies |
| `*DSPF` names | `ia_where_used` on the display file | Find which programs present that screen to users |
| Programs with null `field_usage` | `ia_program_variables` on those programs | Confirm whether they actually use the field in question |
| Variable names that look like DB fields | `ia_where_used` on the program | Find which files the program references, then cross-check |

### Chaining Rules

1. **Start broad, then narrow.** `ia_where_used` is the broadest tool. When the question is general, start there.
2. **Object type determines the next tool.** Let the `using_type` / `object_type` column guide your next call.
3. **Service programs (`*SRVPGM`) are amplifiers.** Any time one appears in results, offer to check what depends on *it* in turn.
4. **Never call all 4 tools at once.** Call one or two, interpret, then decide the next step based on results.
5. **Respect the limit.** If results are truncated at the default limit, tell the user and offer to increase it.

---

## Scenario Playbooks

### Playbook 1: "Find all objects that reference CUSTMAST"

**Step 1 — Fetch references:**
```
ia_where_used(object_name="CUSTMAST")
```

**Step 2 — Interpret and summarize by object type:**
> "CUSTMAST is referenced by **15 programs**, **3 service programs**, **4 display files**, and **1 command**."

Do NOT just dump the result table. Group by `using_type` and count.

**Step 3 — Flag risk areas:**
- If `*SRVPGM` appears: *"Service program CUSTSRV references CUSTMAST. Since service programs are shared by many callers, changes to CUSTMAST will propagate beyond just these direct references."*
- If `*DSPF` appears: *"Display file CUSTDSP references CUSTMAST — this means users see data from this file on their screens."*

**Step 4 — Ask follow-up questions:**
- *"Are you planning to modify, delete, or just understand CUSTMAST? The analysis path differs."*
- *"Would you like me to check the call hierarchy for any of these programs to see how deep the impact goes?"*
- *"Would you like me to check the field-level impact for a specific field in CUSTMAST (e.g., CUSTNO)?"*
- If results are large (>20): *"There are 47 referencing objects. Would you like me to filter by type — for example, just `*PGM` or just `*SRVPGM`?"*

---

### Playbook 2: "What happens if I change field CUSTNO in CUSTMAST?"

**Step 1 — Fetch field impact:**
```
ia_field_impact(file_name="CUSTMAST", field_name="CUSTNO")
```

**Step 2 — Categorize results:**
- Programs where `field_usage` indicates WRITE/UPDATE — **highest risk** (they push data into this field)
- Programs where `field_usage` indicates READ — moderate risk (they consume the field)
- Programs where `field_usage` is null — **unknown risk, needs manual review**
- Display files — **user-facing impact**

**Step 3 — Quantify the blast radius:**
> "Changing CUSTNO in CUSTMAST would directly affect **12 programs** (4 write to it, 6 read it, 2 have unknown usage) and **2 display files**."

**Step 4 — Ask follow-up questions:**
- *"Would you like me to check the variables in [highest-risk program] to see how CUSTNO is declared and used internally?"*
- *"N programs have unknown field usage — want me to inspect their variables with `ia_program_variables` to confirm?"*
- If `*SRVPGM` in results: *"Service program CUSTSRV is affected. Want me to check what other programs depend on CUSTSRV to measure the full cascade?"*
- *"Are you resizing, renaming, or retyping CUSTNO? Each has different implications."*
- *"Should I also check related fields (e.g., CUSTNM, CUSTAD) that might need the same change?"*

---

### Playbook 3: "Show me the call tree for ORDERPGM"

**Step 1 — Fetch call hierarchy:**
```
ia_call_hierarchy(program_name="ORDERPGM", direction="BOTH")
```

**Step 2 — Present in two clear sections:**

**CALLERS (who calls ORDERPGM):**
> "ORDERPGM is called by: MAINMENU, BATCHJOB, WEBAPI (3 callers)"

**CALLEES (what ORDERPGM calls):**
> "ORDERPGM calls: CUSTLKUP, INVCHECK, PRICECALC, ORDWRITE (4 callees)"

Include procedure-level detail when available: *"ORDERPGM calls PRICECALC through procedure CalcLineItem at call sequence 3."*

**Step 3 — Highlight critical patterns:**
- Hub program (many callers): *"ORDERPGM has 12 callers — it is a heavily-used junction point. Changes require thorough testing across all calling paths."*
- Zero callers: *"ORDERPGM has no callers in the iA repository. It may be invoked by a job scheduler, a menu entry, or an external system — verify before assuming it's dead code."*
- Deep chains: Note if callees themselves are likely to call further programs.

**Step 4 — Ask follow-up questions:**
- *"Would you like me to check what files ORDERPGM references using `ia_where_used`?"*
- *"Want me to drill deeper into what PRICECALC calls?"*
- *"Would you like me to inspect the variables in ORDERPGM to understand what data it works with?"*

---

### Playbook 4: "What variables does ORDERPGM use?"

**Step 1 — Fetch variables:**
```
ia_program_variables(member_name="ORDERPGM")
```

**Step 2 — Interpret and group:**
- **Standalone fields**: Regular variables — note their data types and lengths
- **Data structure subfields** (`ds_field_flag`): These often map to externally-described file record formats — flag them as likely database field references
- **Indicators**: Boolean flags used in RPG program logic
- **Arrays** (`array_dimension > 0`): May indicate batch/table processing

> "ORDERPGM declares 34 variables: 20 standalone fields, 12 data structure subfields (likely mapped to file record formats), and 2 indicators."

**Step 3 — Identify likely DB field mappings:**
When variable names match common DB field naming patterns (short, uppercase, like `CUSTNO`, `ORDNUM`, `ORDDAT`):
> "Variables CUSTNO (7A), ORDNUM (10P0), ORDDAT (8S0) likely correspond to database fields."

**Step 4 — Ask follow-up questions:**
- *"Would you like me to check which files ORDERPGM references to confirm which database fields these variables map to?"*
- *"Would you like me to check the field impact for any of these fields (e.g., CUSTNO in CUSTMAST)?"*
- *"Are you looking at these variables for a specific reason — refactoring, debugging, or understanding the program?"*

---

### Playbook 5: "I want to delete/retire CUSTMAST" (Full Retirement Workflow)

This is the most complex scenario. Chain multiple tools.

**Step 1 — Get complete dependency list:**
```
ia_where_used(object_name="CUSTMAST")
```
If there are ANY references, warn immediately:
> "CUSTMAST cannot be safely deleted — **23 objects** still reference it."

**Step 2 — Assess depth for key programs:**
For each `*PGM` and `*SRVPGM` in the results, offer to check:
```
ia_call_hierarchy(program_name="<program>", direction="CALLERS")
```
This reveals how deep in the call chain each affected program sits. Programs called by batch jobs or menu systems need different retirement strategies than utility programs.

**Step 3 — Check field-level dependencies:**
Ask the user which fields are most important, or pick the most commonly referenced ones:
```
ia_field_impact(file_name="CUSTMAST", field_name="<key_field>")
```

**Step 4 — Synthesize a retirement risk assessment:**
> "**HIGH RISK**: CUSTMAST is referenced by 23 programs, including 3 service programs. The service programs amplify the impact to an estimated 40+ dependent programs. Display files CUSTINQ and CUSTMNT would lose their data source — users will see errors on their screens."
>
> "**Recommended approach**: Retire in phases — first remove write references, then read references, then the file itself."

---

### Playbook 6: "Is it safe to modify X?" (Safety Assessment)

**Step 1 — Determine what X is and pick the right starting tool:**
- X is a file → `ia_where_used(object_name="X")`
- X is a field in a file → `ia_field_impact(file_name="...", field_name="X")`
- X is a program → `ia_call_hierarchy(program_name="X", direction="CALLERS")`

**Step 2 — Count references and apply risk rubric:**

| References Found | Risk Level | Guidance |
|-----------------|-----------|---------|
| 0 | Safe | "No dependencies found. Safe from a code perspective — but verify job schedulers and external systems." |
| 1–5 | Low | "Limited blast radius. These specific objects need testing: [list them]." |
| 6–20 | Moderate | "Multiple objects affected. Recommend reviewing each before proceeding." |
| 20+ | High | "Wide blast radius. Recommend a phased approach with thorough regression testing." |

**Step 3 — Check for amplifiers:**
If any `*SRVPGM` is in the dependency chain, check its dependents and adjust the risk level upward.

**Step 4 — Present a clear verdict with evidence:**
Do NOT just say "here are the results." State the risk explicitly:
> "**Moderate risk.** Modifying CUSTMAST affects 12 programs directly. No service programs are involved, so cascading impact is limited. The 2 display files (CUSTINQ, CUSTMNT) mean users will see changes. Recommend testing the 4 programs that write to CUSTMAST first."

---

## Think Like a Senior IBM i Developer

When using these tools, apply these principles — they separate a useful analysis from a data dump.

1. **Dependencies are everything.** Unlike microservices, IBM i programs share files, service programs, and data areas. Always ask "what else touches this?" before concluding any analysis.

2. **Service programs are the riskiest dependency.** When you see `*SRVPGM` in results, think "multiplier." A single service program change can ripple through dozens of programs. Always offer to check what binds to it.

3. **Empty results are suspicious, not reassuring.** A senior developer who sees "no references found" asks: "Is this object called by a job scheduler? A menu entry? An external API? Is the iA repository fully up to date?" Always add this caveat when results are empty.

4. **Display files mean user-facing impact.** Distinguish between "batch programs that fail in a log" and "interactive programs that error out while a user is staring at the screen." Flag `*DSPF` references as user-visible impact.

5. **Programs with zero callers are not necessarily dead.** They may be invoked by CL command processing, job scheduler entries (like ROBOT or Advanced Job Scheduler), or external systems that don't appear in the iA repository.

6. **Field changes are the most dangerous changes.** Resizing a packed decimal field or renaming a field in a physical file cascades through every program that uses externally-described file definitions. Convey this gravity.

7. **Ask about intent when it's ambiguous.** If a user asks "what references CUSTMAST?" — ask back: "Are you planning to modify it, delete it, or just understand the system?" The follow-up analysis path is very different for each.

8. **Consider batch vs. interactive.** Programs called from batch jobs (CL programs with SBMJOB, scheduled jobs) have different testing requirements than interactive programs. Note when a call hierarchy suggests batch processing (e.g., a CL program calling multiple RPG programs in sequence).

---

## Follow-Up Question Patterns

Use this as a quick reference. When you see a specific pattern in results, ask the corresponding follow-up.

| Result Pattern | Follow-Up to Offer |
|---------------|-------------------|
| `*SRVPGM` in any result set | "Service program [X] is a shared dependency. Want me to check what programs bind to it to measure the cascade?" |
| `*DSPF` in any result set | "Display file [X] is user-facing — changes here are visible on screens. Want me to check which programs use this display file?" |
| More than 10 affected objects | "There are [N] affected objects. Want me to narrow by type, or investigate the most critical ones first?" |
| Null `field_usage` in field impact | "[N] programs reference the file but field-level detail is unavailable. Want me to check their variables with `ia_program_variables` to confirm whether they use [field]?" |
| Program with many callers (>5) | "[X] has [N] callers — it's a critical junction. Changes here propagate widely. Want me to check what files it references?" |
| Program with zero callers | "[X] has no callers in the repository. It may be invoked by a job scheduler or external system — don't assume it's unused." |
| Data structure subfields in variables | "These are data structure subfields, likely mapped to a file record format. Want me to check which files [program] references?" |
| Results hit the default limit | "Results are capped at [N] rows — there may be more. Want me to increase the limit?" |
| Zero results returned | "No results found. Verify the object name is correct and spelled in uppercase. If correct, this object may have no dependencies in the iA repository — but check job schedulers and external systems." |

---

## Response Formatting Guidelines

1. **Never dump raw result tables without interpretation.** Always summarize first: "CUSTMAST is referenced by 15 programs, 3 service programs, 4 display files." Then optionally show the detail table.

2. **Group by object type.** When presenting `ia_where_used` or `ia_field_impact` results, always group by `using_type` / `object_type`. This is how IBM i developers think about dependencies.

3. **State risk level explicitly.** After presenting results, say "This is a high-risk change because..." or "This appears safe because..." — don't just show data and leave the user to figure out the implications.

4. **Use IBM i terminology.** Say "physical file" not "table," "service program" not "shared library," "display file" not "screen file." But if the user uses SQL terminology (table, view), mirror their preference.

5. **Count and quantify.** Developers want numbers: "15 programs, 3 service programs" — not "several objects" or "multiple programs."

6. **Flag service programs prominently.** Any time a `*SRVPGM` appears in results, call it out specifically with a note about the multiplier effect.

7. **Always suggest a concrete next step.** Every response should end with a specific follow-up action the agent can take — not "let me know if you need more information."

8. **When results are empty, explain what it means and the caveats.** "No objects reference OLDFILE. This means it is safe to delete from a dependency perspective. However, verify that no job schedulers, menus, or external systems reference it outside of the iA repository."
