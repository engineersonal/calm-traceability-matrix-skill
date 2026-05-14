---
name: calm-traceability-matrix
description: >
  Generates a SAP Cloud ALM Requirement Traceability Matrix Excel file for a
  given project. Use this skill whenever the user mentions any of the following:
  "traceability matrix", "traceability excel", "SAP Cloud ALM traceability",
  "CALM traceability", "generate traceability for [project]", "requirement
  traceability report", "traceability for SAP project". Also triggers when the
  user provides a project name and asks for an Excel export combining requirements,
  features, user stories, test cases, and defects from SAP Cloud ALM.
  The skill produces an .xlsx file that exactly matches the structure of SAP
  CALM's built-in Requirement Traceability report, enriched with additional
  columns (Scope, Solution Process Status, Defect, Defect Status) not present
  in the standard CALM export.
compatibility:
  tools:
    - sap-cloud-alm MCP server (list_projects, get_analytics_requirements,
      get_analytics_features, get_analytics_tests, get_analytics_defects,
      list_tasks, list_solution_processes)
    - bash_tool (Python + openpyxl for Excel generation)
    - present_files
---
 
# SAP Cloud ALM — Requirement Traceability Matrix Skill
 
Generates a professional, colour-coded Requirement Traceability Matrix Excel
workbook for any SAP Cloud ALM project. The output matches CALM's own
Requirement Traceability report row-for-row, enriched with Scope, Solution
Process Status, and Defects columns.
 
Read `references/api-guide.md` for full field mappings, status code tables,
and pagination rules before executing any API calls.
Read `references/excel-guide.md` for exact column layout, colour codes, and
formatting rules before generating the Excel file.
 
---
 
## Workflow
 
### STEP 1 — Identify the Project
 
Call `list_projects`. Find the entry matching the user's project name
(case-insensitive). Extract its `id` (UUID). Use this UUID in every
subsequent API call. If no match is found, tell the user and stop.
 
### STEP 2 — Fetch All Data in Parallel
 
Call all six APIs simultaneously. See `references/api-guide.md` for exact
parameters, field names, and pagination rules.
 
| # | API | Filter |
|---|-----|--------|
| A | `get_analytics_requirements` | `project eq '<id>'` |
| B | `get_analytics_features` | `projectId eq '<id>'` |
| C | `list_tasks` | `project_id='<id>'`, no task_type filter |
| D | `get_analytics_tests` | `projectGUID eq '<id>'` |
| E | `get_analytics_defects` | `projectName eq '<name>'` |
| F | `list_solution_processes` | `projectId eq '<id>'` |
 
**Pagination**: if a response returns exactly 500 records, fetch the next
page with `$skip=500`, then `$skip=1000`, until fewer than 500 are returned.
 
### STEP 3 — Build Lookup Indexes
 
After fetching, build these five in-memory indexes:
 
```
feat_by_req_guid[feat.requirementId]      → feature record
test_by_scope[test.scopeName]             → [list of test records]
defect_by_tc[defect.testcaseId]           → defect record
defect_by_scope[defect.scopeName]         → defect record (fallback)
sp_status_by_scope[sp.scopeId]            → validated SP status string
```
 
See `references/api-guide.md` → "Index Building Rules" for join logic and
the SP status validation list.
 
### STEP 4 — Build the Rows
 
**One row per linked entity per requirement** — this is the critical rule
that makes the output match CALM's report.
 
For each requirement, emit separate rows for:
- Each linked **Feature** (joined via feat_by_req_guid[req.GUID])
- Each linked **Task / User Story** (matched to req by solution process or
  requirement name keywords)
- Each **Test Case × Test Plan** combination (same TC in 2 plans = 2 rows)
- If no entities are linked → 1 row with entity columns blank
Every row carries the requirement's base columns repeated:
Solution Process, Scope, Solution Process Status, Project, Requirement,
Requirement Name, Requirement Status.
 
See `references/api-guide.md` → "Row Building Rules" for full column
population logic per row type (feature row, task row, test row).
 
### STEP 5 — Generate the Excel File
 
Use `openpyxl` in `bash_tool`. See `references/excel-guide.md` for the
exact column layout (18 columns, A–R), group header merges, colour codes
for all status values, alternating row shading, and Coverage Summary sheet.
 
Save to `/mnt/user-data/outputs/<ProjectName>_Traceability_Matrix.xlsx`.
Call `present_files` with that path.
 
---
 
## Critical Rules (Never Violate)
 
1. **Solution Process name** → always `req.process` field directly.
   Never derive from a `scopeId` join. This field is the authoritative
   direct link CALM stores internally per requirement.
2. **Solution Process Status** → only from `list_solution_processes`
   `.statusName`, validated against exactly 5 values:
   `Design | Realization | Production | Maintenance | Obsolete`.
   If blank or not in that list → store empty string. Never substitute
   any other entity's status.
3. **Each entity's status comes only from its own API** — never
   cross-populate. See `references/api-guide.md` → "Status Sources".
4. **Row count matches CALM**: 1 row per linked entity, not 1 row per
   requirement. A req with 4 user stories + 2 test cases = 6 rows.
5. **Fetch ALL task types** from `list_tasks` — not just `CALMUS`.
   `CALMPT` (Project Task) entries must also appear in the output.
6. **Do not deduplicate test cases** by `testcaseId`. Each test plan
   execution is a separate row.
7. **Defects attach only to test rows**, never to requirement-only or
   task-only rows. Primary join: `defect.testcaseId = test.testcaseId`.
   Fallback: `defect.scopeName = req.scopeName`.
 
