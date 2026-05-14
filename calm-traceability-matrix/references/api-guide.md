# SAP Cloud ALM API Guide

## API Parameters & Key Fields

### A. get_analytics_requirements
```
filter: project eq '<projectId>'
top: 500
```
| Field | Maps to Column | Notes |
|---|---|---|
| `requirementId` | Requirement | e.g. "3-6742" |
| `name` | Requirement Name | Full text name |
| `statusText` | Requirement Status | Use as-is — already human-readable |
| `process` | Solution Process | **Direct field. Never derive from scopeId.** |
| `scope` | (join key) | UUID — used only for SP status lookup |
| `scopeName` | Scope | e.g. "Finance", "Manufacturing Model Company" |
| `GUID` | (join key) | Used to join features |

### B. get_analytics_features
```
filter: projectId eq '<projectId>'
top: 500
```
| Field | Maps to Column | Notes |
|---|---|---|
| `featureName` | Feature | Feature display name |
| `statusText` | Feature Status | Use as-is |
| `requirementId` | (join key) | Matches req.GUID |

### C. list_tasks
```
project_id: '<projectId>'
limit: 100
(no task_type filter — fetch ALL types)
```
| Field | Maps to Column | Notes |
|---|---|---|
| `title` | Task name | |
| `status` | Task Status | Map via code table below |
| `type` | Task Type | Map via code table below |

**Task Status Code Mapping:**
```
CIPUSOPEN  → Open          CIPUSINP   → In Progress
CIPUSCLOSE → Done          CIPUSBLK   → Blocked
CIPUSNO    → Not Started   CIPTKOPEN  → Open
CIPTKCLOSE → Done          CIPTKINP   → In Progress
CIPTKBLK   → Blocked
```

**Task Type Code Mapping:**
```
CALMUS     → User Story    CALMPT     → Project Task
CALMTMPL   → Project Task  CALMCHKLI  → Checklist Item
CALMQGATE  → Quality Gate
```

### D. get_analytics_tests
```
filter: projectGUID eq '<projectId>'
top: 500
```
| Field | Maps to Column | Notes |
|---|---|---|
| `testcaseName` | Test Case | |
| `type` | Test Case Status | "Manual" or "Automated" |
| `status` | Test Execution Status | Initial/Passed/Failed/In Progress |
| `testPlan` | Test Plan | |
| `testcaseId` | (join key) | UUID for defect join |
| `scopeName` | (join key) | For matching to requirements |

**Deduplication rule**: Do NOT deduplicate by testcaseId. Same TC in
2 test plans = 2 rows in output.

### E. get_analytics_defects
```
filter: projectName eq '<projectName>'
top: 500
```
| Field | Maps to Column | Notes |
|---|---|---|
| `name` | Defect | Defect display name |
| `statusText` | Defect Status | Use as-is: Closed/New/In Progress/Postponed/Retest Required |
| `testcaseId` | (join key) | Primary join to tests |
| `scopeName` | (join key) | Fallback join to requirements |

### F. list_solution_processes
```
filter: projectId eq '<projectId>'
top: 500  (paginate if needed)
```
| Field | Use | Notes |
|---|---|---|
| `scopeId` | Join key | Matches req.scope |
| `solutionProcessVersionName` | Backup SP name | Only used if req.process is blank |
| `statusName` | Solution Process Status | Must validate — see below |

**SP Status Validation** — only accept these exact 5 values:
```
Design | Realization | Production | Maintenance | Obsolete
```
Any other value (blank, null, or unrecognised) → store as empty string.

---

## Index Building Rules

### feat_by_req_guid
```python
feat_by_req_guid = {}
for feat in features:
    rid = feat.get('requirementId')
    if rid:
        feat_by_req_guid[rid] = feat
```

### test_by_scope
```python
test_by_scope = defaultdict(list)
for test in tests:
    test_by_scope[test.get('scopeName', '')].append(test)
```

### defect_by_tc + defect_by_scope
```python
defect_by_tc    = {}
defect_by_scope = {}
for d in defects:
    if d.get('testcaseId'):
        defect_by_tc[d['testcaseId']] = d
    if d.get('scopeName'):
        defect_by_scope.setdefault(d['scopeName'], d)
```

### sp_status_by_scope
```python
VALID_SP = {'Design', 'Realization', 'Production', 'Maintenance', 'Obsolete'}
sp_status_by_scope = {}
for sp in solution_processes:
    sid = sp.get('scopeId')
    status = sp.get('statusName', '')
    if sid and status in VALID_SP:
        if sid not in sp_status_by_scope:  # first valid wins
            sp_status_by_scope[sid] = status
```

---

## Status Sources

**Each entity's status comes ONLY from its own API. Never cross-populate.**

| Column | Source API | Source Field |
|---|---|---|
| Solution Process Status | list_solution_processes | statusName (validated) |
| Requirement Status | get_analytics_requirements | statusText |
| Feature Status | get_analytics_features | statusText |
| Task Status | list_tasks | status (mapped via code table) |
| Test Case Status | get_analytics_tests | type |
| Test Execution Status | get_analytics_tests | status |
| Defect Status | get_analytics_defects | statusText |

---

## Row Building Rules

For each requirement record, build rows in this order:

### Feature rows
```python
feat = feat_by_req_guid.get(req['GUID'])
if feat:
    emit_row(req, feature=feat, task=None, test=None)
```

### Task rows
```python
# Match tasks to requirement by:
# 1. Task title contains requirementId string, OR
# 2. Task title contains keywords from req.name, OR
# 3. Task is linked to same solution process as req (less reliable)
for task in matched_tasks:
    emit_row(req, feature=None, task=task, test=None)
```

### Test rows (one per testcase × testplan combination)
```python
scope_tests = test_by_scope.get(req.get('scopeName', ''), [])
for test in scope_tests:
    # Find defect for this test
    md = defect_by_tc.get(test.get('testcaseId'))
    if not md:
        md = defect_by_scope.get(req.get('scopeName', ''))
    emit_row(req, feature=None, task=None, test=test, defect=md)
```

### No linked entities
```python
if not feat and not matched_tasks and not scope_tests:
    emit_row(req, feature=None, task=None, test=None)
```

### emit_row column values
```python
{
    'Solution Process':        req.get('process', '') or '',
    'Scope':                   req.get('scopeName', '') or '',
    'Solution Process Status': sp_status_by_scope.get(req.get('scope',''), ''),
    'Project':                 project_name,
    'Requirement':             req.get('requirementId', ''),
    'Requirement Name':        req.get('name', ''),
    'Requirement Status':      req.get('statusText', ''),
    'Feature':                 feat.get('featureName','') if feat else '',
    'Feature Status':          feat.get('statusText','') if feat else '',
    'Task name':               task.get('title','') if task else '',
    'Task Status':             map_task_status(task['status']) if task else '',
    'Task Type':               map_task_type(task['type']) if task else '',
    'Test Case':               test.get('testcaseName','') if test else '',
    'Test Case Status':        test.get('type','') if test else '',
    'Test Execution Status':   test.get('status','') if test else '',
    'Test Plan':               test.get('testPlan','') if test else '',
    'Defect':                  defect.get('name','') if defect else '',
    'Defect Status':           defect.get('statusText','') if defect else '',
}
```

---

## Pagination

Always paginate when exactly 500 records are returned:
```python
skip = 0
all_records = []
while True:
    batch = api_call(top=500, skip=skip)
    all_records.extend(batch['value'])
    if len(batch['value']) < 500:
        break
    skip += 500
```
