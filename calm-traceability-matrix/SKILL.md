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
