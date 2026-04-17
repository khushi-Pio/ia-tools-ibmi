# iA Tools for IBM i — Impact Analysis MCP Tool Definitions

SQL-based [MCP](https://modelcontextprotocol.io/) tool definitions for **Impact Analysis (iA)** on IBM i. These tools query pre-parsed IBM i source metadata in the iA repository (built by [programmers.io](https://www.programmers.io/)) to provide cross-reference lookups, call hierarchy analysis, field impact analysis, and more.

Use them with **VS Code GitHub Copilot**, **Claude Code**, or any MCP-compatible AI agent to ask natural-language questions about your IBM i codebase.

## What's in this repo

| File | Description |
|------|-------------|
| `impact-analysis.yaml` | MCP tool definitions for iA queries |
| `.claude/skills/ia/` | `/ia` skill for Claude Code — query guidance and SQL patterns |
| `AGENTS.md` | Agent-specific guidance: always use `/ia` skill for iA analysis |
| `.vscode/mcp.json` | VS Code MCP server config (auto-detected on open) |
| `.env.example` | DB2i connection template |
| `LICENSE` | Apache-2.0 |

### `/ia` Skill (for Claude Code)

A token-efficient skill at `.claude/skills/ia/` that teaches AI agents how to query all 35+ iA tables. Invoke with `/ia` in any Claude Code session. The skill includes table schemas, SQL patterns, and query workflows in its `references/` folder.

## Tools (47 custom + 2 built-in)

### Custom iA Tools (defined in `impact-analysis.yaml`)

| # | Tool | Description |
|---|------|-------------|
| 1 | `ia_where_used` | Find all objects referencing a given object |
| 2 | `ia_call_hierarchy` | Program call tree (CALLERS/CALLEES/BOTH) |
| 3 | `ia_field_impact` | Blast radius of changing a field in a file |
| 4 | `ia_program_variables` | All variables declared in a program |
| 5 | `ia_data_structures` | Data structure definitions and subfields |
| 6 | `ia_call_parameters` | Parameters passed at each external call site |
| 7 | `ia_subroutines` | BEGSR/EXSR details with usage counts |
| 8 | `ia_file_overrides` | OVRDBF statements (real file routing) |
| 9 | `ia_file_fields` | Field-level metadata for a database file |
| 10 | `ia_object_list` | Repository inventory filtered by object type and attribute |
| 11 | `ia_program_info` | Program/module metadata (source, compile info) |
| 12 | `ia_program_summary` | Quick program overview with compile info and complexity |
| 13 | `ia_program_detail` | Deep structural analysis (calls, files, subroutines, variables) |
| 14 | `ia_source_code` | Source member location and line counts |
| 15 | `ia_rpg_source_tokens` | Token-level RPG source analysis |
| 16 | `ia_cl_source_tokens` | Token-level CL source analysis |
| 17 | `ia_dashboard` | Repository health summary by member category |
| 18 | `ia_repo_config` | iA repository configuration settings |
| 19 | `ia_exception_log` | iA parser exception log |
| 20 | `ia_dds_to_ddl_status` | DDS→DDL conversion tracking |
| 21 | `ia_reference_count` | Lightweight reference count grouped by type |
| 22 | `ia_unused_objects` | Dead-code candidates (unreferenced objects) |
| 23 | `ia_circular_deps` | Detect circular call chains |
| 24 | `ia_override_chain` | Chained OVRDBF dependencies (A→B→C) |
| 25 | `ia_object_lifecycle` | Creation/change/last-used dates per object |
| 26 | `ia_code_complexity` | Complexity metrics per source member |
| 27 | `ia_library_files` | List all files/tables in the iA library |
| 28 | `ia_object_lookup` | Look up object type, library, and attribute by name |
| 29 | `ia_file_dependencies` | Find LFs, indexes, and views dependent on a physical file |
| 30 | `ia_uncompiled_sources` | Find source members without compiled objects (orphaned sources) |

### Advanced Analysis

| # | Tool | Description |
|---|------|-------------|
| 31 | `ia_copybook_impact` | Find programs including a copybook via /COPY |
| 46 | `ia_member_copybooks` | List copybooks used by a source member |
| 32 | `ia_srvpgm_exports` | List service program exported/imported procedures |
| 33 | `ia_procedure_xref` | Procedure-level cross-reference (callers/callees) |
| 34 | `ia_procedure_params` | Get procedure PR/PI parameter signatures |
| 35 | `ia_cl_jobs` | Detect SBMJOB calls in CL programs |
| 36 | `ia_variable_ops` | Find variable declarations, assignments, BIFs |
| 37 | `ia_klist_usage` | Find KLIST/KFLD key list definitions |
| 38 | `ia_application_area` | List application areas and their objects |
| 39 | `ia_sql_names` | Map SQL long names to system short names |
| 40 | `ia_program_files` | List files used by a program with PREFIX details |
| 41 | `ia_rpg_source` | Read RPG source code with optional spec-type filtering |
| 42 | `ia_rpg_source_search` | Search RPG source across members for keywords |
| 43 | `ia_rpg_source_stats` | Modernization stats: free-format vs fixed-format, comment ratio |
| 44 | `ia_member_lookup` | Look up source member metadata and verify existence |
| 45 | `ia_object_references` | Find what an object references/contains (inverse of ia_where_used) |
| 47 | `ia_obj_size` | Object size and usage metrics — lookup a specific object or rank the largest / unused objects |

### Built-in MCP Server Tools

| Tool | Description | Enabled by |
|------|-------------|------------|
| `execute_sql` | Run any SELECT query the AI agent constructs (read-only by default) | `IBMI_ENABLE_EXECUTE_SQL=true` in `.env` |
| `describe_sql_object` | Get DDL/schema for any table, view, index, or procedure | Always enabled |

> More tools are being developed and will be released incrementally. Contributions welcome!

---

## Quick Start — VS Code + GitHub Copilot

This is the recommended way to use and develop iA tools. VS Code will automatically start the MCP server when you open this folder.

### Prerequisites

| Requirement | Details |
|-------------|---------|
| **VS Code** | With [GitHub Copilot](https://marketplace.visualstudio.com/items?itemName=GitHub.copilot) extension installed (Copilot Chat required) |
| **Node.js 18+** | Download from [nodejs.org](https://nodejs.org/) — required to run the MCP server |
| **IBM i access** | Db2 for i accessible via [Mapepire](https://mapepire.github.io/) on port 8076 |
| **iA repository** | Your IBM i source must be parsed by [programmers.io iA](https://www.programmers.io/) into a library (e.g., `SDK01`) |

### Step 1: Fork and clone

```bash
# Fork this repo on GitHub (click the "Fork" button), then:
git clone https://github.com/<your-username>/ia-tools-ibmi.git
cd ia-tools-ibmi
```

### Step 2: Configure your IBM i connection

```bash
cp .env.example .env
```

Edit `.env` with your IBM i credentials:

```
DB2i_HOST=your-ibmi-hostname
DB2i_USER=your-user-profile
DB2i_PASS=your-password
DB2i_PORT=8076
IA_LIBRARY=SDK01
```

| Variable | Description |
|----------|-------------|
| `DB2i_HOST` | IBM i hostname or IP address |
| `DB2i_USER` | IBM i user profile with `*READ` authority to iA tables |
| `DB2i_PASS` | Password for the user profile |
| `DB2i_PORT` | Mapepire port (default: `8076`) |
| `IA_LIBRARY` | Library where iA repository tables are stored (default: `SDK01`) |
| `IBMI_ENABLE_EXECUTE_SQL` | Set to `true` to enable the built-in `execute_sql` tool (default: `false`) |

> **Security**: `.env` is in `.gitignore` — your credentials are never committed.

### Step 3: Start the MCP server

Install and start the MCP server in a terminal:

```bash
npm install -g @ibm/ibmi-mcp-server
npx @ibm/ibmi-mcp-server --transport http --tools ./impact-analysis.yaml
```

You should see output like:
```
IBM i MCP Server listening on http://localhost:3000
```

> **Keep this terminal running** — the MCP server must stay active while you use VS Code.

> **Windows users**: If `npx` gives "Access is denied", use `node` directly:
> ```bash
> node %APPDATA%\npm\node_modules\@ibm\ibmi-mcp-server\dist\index.js --transport http --tools ./impact-analysis.yaml
> ```

### Step 4: Open in VS Code

```bash
code .
```

VS Code detects `.vscode/mcp.json` and connects to the running MCP server at `http://localhost:3000/mcp`. The 47 iA tools plus built-in SQL tools become available in Copilot Chat.

### Step 5: Use iA tools in Copilot Chat

1. **Open Copilot Chat**: Press `Ctrl+Alt+I` (Windows/Linux) or `Cmd+Alt+I` (Mac)
2. **Switch to Agent mode**: Click the mode dropdown at the top of the chat panel and select **"Agent"**
3. **Verify tools are loaded**: Click the **tools icon** (wrench/hammer) at the top-left of the chat input — you should see the 47 `ia-*` tools plus `execute_sql` and `describe_sql_object` listed under "ibmi-ia-tools"
4. **Ask a question** — the agent will automatically pick the right iA tool:

```
Find all programs that reference CUSTFILE
```
```
Show the call hierarchy for ORDERPGM
```
```
What is the blast radius of changing field CUSTNO in ORDERFILE?
```
```
List all variables in program ORDHIST
```

> **Note**: Copilot will ask for your confirmation before running each MCP tool. Click **"Allow"** (or "Always allow" for this session).

---

## Alternative Setup

### Claude Code

Create a `.mcp.json` file in your project root:

```json
{
  "mcpServers": {
    "ibmi-ia-tools": {
      "command": "npx",
      "args": ["-y", "@ibm/ibmi-mcp-server@latest", "--tools", "/full/path/to/impact-analysis.yaml"],
      "cwd": "."
    }
  }
}
```

Set `DB2i_HOST`, `DB2i_USER`, `DB2i_PASS`, `DB2i_PORT`, and `IA_LIBRARY` as environment variables (or in `.env` in the same directory — the MCP server loads it automatically).

### Command Line (manual testing)

Run the MCP server directly to test without an AI agent:

```bash
# Install the MCP server globally
npm install -g @ibm/ibmi-mcp-server

# Set environment variables
export DB2i_HOST=your-ibmi-host
export DB2i_USER=your-user
export DB2i_PASS=your-password
export DB2i_PORT=8076
export IA_LIBRARY=SDK01

# Start in HTTP mode for manual testing
npx @ibm/ibmi-mcp-server --transport http --tools ./impact-analysis.yaml
# Server starts at http://localhost:3000
```

---

## Contributing

We welcome contributions — new tools, bug fixes, improvements, and questions!

- **File an issue**: Use our [issue templates](https://github.com/PIO-Anurag-Garg/ia-tools-ibmi/issues/new/choose) for bugs, new tool requests, improvements, or questions
- **Submit a PR**: Fork, edit `impact-analysis.yaml`, test against your iA repository, and open a pull request

See **[CONTRIBUTING.md](CONTRIBUTING.md)** for the full guide: setup instructions, tool conventions, testing workflow, and PR checklist.

---

## iA Tables

These tools query the following iA repository tables (pre-parsed IBM i source metadata):

| Table | Purpose |
|-------|---------|
| `IAALLREFPF` | Cross-reference: object-to-object relationships |
| `IAPGMCALLS` | Call graph: CALL, CALLP, bound module references |
| `IAFILEDTL` | Field-level details for database files |
| `IAPGMVARS` | Program variables (standalone, DS subfields, indicators) |
| `IAPGMDS` | Data structure definitions and subfields |
| `IACALLPARM` | Parameters passed at each call site |
| `IASUBRDTL` | Subroutines (BEGSR/EXSR) with usage counts |
| `IAOVRPF` | File overrides (OVRDBF) |
| `IAPGMINF` | Program/module metadata and compile info |
| `IAPGMREF` | RPG source token-level index |
| `IACPGMREF` | CL source token-level index |
| `IAOBJMAP` | Object-to-source member mapping |
| `IAOBJECT` | Object lifecycle (create/change/last-used dates) |
| `IASRCMBRID` | Source member metadata (file, library, type, line counts) |
| `IADSPDBR` | DSPDBR output (logical file/index/view dependencies) |
| `IAEXCPLOG` | iA parser exception log |
| `OBJECT_DETAILS` | Object inventory (type, library, source) |
| `IA_DASHBOARD_DETAIL` | Member categories, line counts, library map |
| `IA_CODE_INFO` | Per-member complexity metrics |
| `REPO_CONFIGURATION` | iA repo configuration settings |
| `DDSTODDL_FILE_CONVERSION_DETAILS` | DDS→DDL modernization tracking |
| `IAMEMBER` | Source member list: file, library, type, timestamps |
| `QSYS2.SYSTABLES` | System catalog: file/table inventory per library |

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `npx: Access is denied` (Windows) | Install globally: `npm install -g @ibm/ibmi-mcp-server`, then use `node %APPDATA%\npm\node_modules\@ibm\ibmi-mcp-server\dist\index.js --transport http --tools ./impact-analysis.yaml` |
| MCP server not starting | Check Node.js version: `node --version` (must be 18+) |
| Tools not showing in Copilot Chat | Make sure you're in **Agent** mode (not Ask or Edit mode). Click the tools icon to enable/disable specific tools |
| Tools not connecting in VS Code | Ensure the MCP server is running in a separate terminal (`http://localhost:3000`) before opening VS Code |
| `Connection refused` on port 8076 | Verify Mapepire is running on your IBM i — check with your system administrator |
| Tools return no data | Check `IA_LIBRARY` in `.env` matches your iA library name. Verify the user profile has `*READ` authority to the iA tables |
| YAML syntax error | Validate the file: `npx js-yaml impact-analysis.yaml` |
| MCP server shows stale tools after edit | Stop (`Ctrl+C`) and restart the MCP server in your terminal |

---

## License

Apache-2.0 — see [LICENSE](LICENSE).
