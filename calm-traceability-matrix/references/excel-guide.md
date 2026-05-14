# Excel Formatting Guide

## Workbook Structure

Two sheets:
1. `Traceability Matrix` — main data (tab colour #2E75B6)
2. `Coverage Summary`    — scope-level summary (tab colour #1F3864)

---

## Sheet 1: Traceability Matrix

### Colour Constants
```python
DARK_BLUE = '1F3864'   # title bar
MID_BLUE  = '2E75B6'   # group headers, column headers
ALT_ROW   = 'EBF3FB'   # even data rows
WHITE     = 'FFFFFF'    # odd data rows
```

### Row Layout

| Row | Content | Height |
|-----|---------|--------|
| 1 | Merged title (A1:R1) | 28 |
| 2 | Empty spacer | 5 |
| 3 | Group headers | 20 |
| 4 | Column headers with auto-filter | 32 |
| 5+ | Data rows | 38 |

### Row 1 — Title
```python
ws.merge_cells('A1:R1')
c = ws['A1']
c.value = f'Traceability Matrix TO BE (CALM-based export) — {project_name}'
c.font  = Font(name='Arial', bold=True, size=14, color='FFFFFF')
c.fill  = PatternFill('solid', fgColor='1F3864')
c.alignment = Alignment(horizontal='left', vertical='center', indent=1)
```

### Row 3 — Group Headers
```python
GROUPS = [
    ('A3:D3', 'Solution Process'),
    ('E3:G3', 'Requirement'),
    ('H3:I3', 'Feature'),
    ('J3:L3', 'Task / User Story'),
    ('M3:P3', 'Test'),
    ('Q3:R3', 'Defect'),
]
# Each: bold Arial 10, white text, MID_BLUE fill, medium border
```

### Row 4 — Column Headers (18 columns A–R)
```
A: Solution Process       B: Scope
C: Solution Process Status  D: Project
E: Requirement            F: Requirement Name
G: Requirement Status     H: Feature
I: Feature Status         J: Task name
K: Task Status            L: Task Type
M: Test Case              N: Test Case Status
O: Test Execution Status  P: Test Plan
Q: Defect                 R: Defect Status
```
Style: bold Arial 9, white text, MID_BLUE fill, thin border, wrap text,
centre-aligned, auto-filter on A4:R4, freeze panes at A5.

### Column Widths
```python
WIDTHS = {
    'A': 36,  # Solution Process
    'B': 22,  # Scope
    'C': 20,  # SP Status
    'D': 16,  # Project
    'E': 12,  # Requirement (ID)
    'F': 40,  # Requirement Name
    'G': 20,  # Requirement Status
    'H': 30,  # Feature
    'I': 14,  # Feature Status
    'J': 36,  # Task name
    'K': 14,  # Task Status
    'L': 14,  # Task Type
    'M': 40,  # Test Case
    'N': 16,  # Test Case Status
    'O': 20,  # Test Execution Status
    'P': 20,  # Test Plan
    'Q': 28,  # Defect
    'R': 14,  # Defect Status
}
```

### Data Row Styling
```python
# Alternating row background
bg = 'EBF3FB' if row_index % 2 == 0 else 'FFFFFF'

# Default cell style
cell.font      = Font(name='Arial', size=9)
cell.fill      = PatternFill('solid', fgColor=bg)
cell.border    = thin_border
cell.alignment = Alignment(vertical='top', wrap_text=True)

# Status cells override with colour from STATUS_COLORS table
if field in STATUS_FIELDS and value in STATUS_COLORS:
    bg_c, fg_c = STATUS_COLORS[value]
    cell.fill = PatternFill('solid', fgColor=bg_c)
    cell.font = Font(name='Arial', size=9, color=fg_c, bold=True)
```

### STATUS_FIELDS
```python
STATUS_FIELDS = {
    'Solution Process Status',
    'Requirement Status',
    'Feature Status',
    'Task Status',
    'Test Case Status',
    'Test Execution Status',
    'Defect Status',
}
```

### STATUS_COLORS
```python
STATUS_COLORS = {
    # Solution Process Status (5 valid values)
    'Design':                  ('DDEEFF', '0C447C'),
    'Realization':             ('FFF2CC', '7D6608'),
    'Production':              ('C6EFCE', '276221'),
    'Maintenance':             ('FCE4D6', '833C00'),
    'Obsolete':                ('F2F2F2', '595959'),

    # Requirement Status
    'In Refinement':           ('E2EFDA', '375623'),
    'In Planning':             ('DDEBF7', '1F497D'),
    'In Realization':          ('FFF2CC', '7D6608'),
    'In Approval':             ('FCE4D6', '833C00'),
    'In Testing':              ('FFEB9C', '9C5700'),
    'Approved for Deployment': ('C6EFCE', '276221'),
    'Successfully Tested':     ('C6EFCE', '276221'),
    'Not Planned':             ('F2F2F2', '595959'),
    'Blocked':                 ('FFCCCC', '9C0006'),
    'Confirmed':               ('DDEBF7', '1F497D'),

    # Feature Status
    'In Preparation':          ('DDEBF7', '1F497D'),
    'Prepared':                ('C6EFCE', '276221'),

    # Task Status
    'Open':                    ('DDEBF7', '1F497D'),
    'In Progress':             ('FFEB9C', '9C5700'),
    'Done':                    ('C6EFCE', '276221'),
    'Closed':                  ('C6EFCE', '276221'),
    'Not Started':             ('F2F2F2', '595959'),

    # Test Case Status (type field)
    'Manual':                  ('EEF2FF', '3730A3'),
    'Automated':               ('E2EFDA', '375623'),

    # Test Execution Status
    'Passed':                  ('C6EFCE', '276221'),
    'Failed':                  ('FFCCCC', '9C0006'),
    'Initial':                 ('F2F2F2', '595959'),

    # Defect Status
    'New':                     ('FFCCCC', '9C0006'),
    'Postponed':               ('F2F2F2', '595959'),
    'Retest Required':         ('FCE4D6', '833C00'),
}
```

### Border Helpers
```python
from openpyxl.styles import Border, Side

thin_side   = Side(style='thin',   color='BFBFBF')
medium_side = Side(style='medium', color='9DC3E6')

def thin_border():
    return Border(left=thin_side, right=thin_side,
                  top=thin_side,  bottom=thin_side)

def medium_border():
    return Border(left=medium_side, right=medium_side,
                  top=medium_side,  bottom=medium_side)
```

### Page Setup
```python
ws.page_setup.orientation = 'landscape'
ws.page_setup.fitToPage   = True
ws.page_setup.fitToWidth  = 1
```

---

## Sheet 2: Coverage Summary

### Layout
```
Row 1: Merged title bar (A1:H1) — dark blue, white bold 12pt
Row 2: Merged note bar (A2:H2) — grey italic 8pt, light grey fill
Row 3: Column headers — MID_BLUE, white bold 9pt
Row 4+: One row per scope (+ "(No Scope)" row at bottom)
```

### Note text (row 2)
```
"SP Status sourced from list_solution_processes API only.
Valid values: Design | Realization | Production | Maintenance | Obsolete.
Blank = no solution process assigned to that scope in this tenant."
```

### Summary Columns (A–H)
```
A: Scope              B: Unique Reqs
C: Features           D: User Stories
E: Project Tasks      F: Test Cases
G: Defects            H: SP Status
```

### Column Widths
```python
{'A': 32, 'B': 14, 'C': 10, 'D': 14, 'E': 14, 'F': 12, 'G': 10, 'H': 16}
```

### SP Status cell (column H) colouring
Apply STATUS_COLORS lookup for the SP status value in column H.

### Coverage % colouring (column B — unique reqs with SP name)
No coverage % needed — just counts. The SP Status column (H) is the
health indicator for each scope.

### Freeze panes
```python
ws2.freeze_panes = 'A4'
```

---

## Output Path

```python
safe_name = project_name.replace(' ', '_').replace('/', '-')
out_path  = f'/mnt/user-data/outputs/{safe_name}_Traceability_Matrix.xlsx'
wb.save(out_path)
```

Then call `present_files([out_path])`.
