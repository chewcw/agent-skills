---
name: siemens-tia-portal-integrator
description: >
  Orchestrate data-driven Siemens TIA Portal workflows by reading structured data from CSV and Excel
  files, generating STL/AWL code via the siemens-awl-stl-programmer skill, and applying changes to a
  live TIA Portal project via the tia-portal-openness-mcp-server. Trigger this skill whenever the user
  asks to import, generate, sync, or batch-apply PLC blocks, tags, devices, or data blocks from a
  spreadsheet or CSV source. Also trigger when the user wants to extract existing project data from
  TIA Portal and export it to a file.
---

# Siemens TIA Portal Integrator Skill

---

## Overview

This skill bridges three capabilities:

| Layer | Tool | Purpose |
|---|---|---|
| **File I/O** | Python scripts (embedded below) | Read CSV / Excel into structured dicts |
| **Code Generation** | `siemens-awl-stl-programmer` skill | Produce compilable STL/AWL blocks |
| **Project Automation** | `tia-portal-openness-mcp-server` MCP tools | Create/update devices, tags, and blocks in TIA Portal |

Always activate the **siemens-awl-stl-programmer** skill when the workflow requires generating or
modifying STL block source code. Treat MCP tool calls as the final "write" step — generate and
validate data first, then apply changes.

---

## Python Helper Scripts

These scripts are self-contained and should be executed in a sandboxed Python environment (e.g.,
Claude's bash tool) when the user does not already have a file-reading pipeline. Copy the relevant
script into a temp file and run it; capture `stdout` as JSON for further processing.

---

### Script 1 — Read CSV (`read_csv.py`)

```python
#!/usr/bin/env python3
"""
Read a CSV file and return rows as a JSON array.
Usage: python read_csv.py <filepath> [delimiter]
Output: {"headers": [...], "rows": [{...}, ...], "row_count": N}
"""
import csv
import json
import sys

def read_csv(filepath: str, delimiter: str = ",") -> dict:
    rows = []
    with open(filepath, newline="", encoding="utf-8-sig") as fh:
        reader = csv.DictReader(fh, delimiter=delimiter)
        headers = reader.fieldnames or []
        for row in reader:
            # Skip entirely empty rows
            if any(v.strip() for v in row.values()):
                rows.append({k: v.strip() for k, v in row.items()})
    return {"headers": list(headers), "rows": rows, "row_count": len(rows)}

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print(json.dumps({"error": "Usage: read_csv.py <filepath> [delimiter]"}))
        sys.exit(1)
    filepath  = sys.argv[1]
    delimiter = sys.argv[2] if len(sys.argv) > 2 else ","
    try:
        result = read_csv(filepath, delimiter)
        print(json.dumps(result, ensure_ascii=False, indent=2))
    except Exception as e:
        print(json.dumps({"error": str(e)}))
        sys.exit(1)
```

---

### Script 2 — Read Excel (`read_excel.py`)

```python
#!/usr/bin/env python3
"""
Read one sheet from an Excel (.xlsx) file and return rows as a JSON array.
Usage: python read_excel.py <filepath> [sheet_name_or_index]
Output: {"sheet": "...", "headers": [...], "rows": [{...}, ...], "row_count": N}
Requires: openpyxl  (pip install openpyxl --break-system-packages)
"""
import json
import sys

def read_excel(filepath: str, sheet: str | int = 0) -> dict:
    try:
        import openpyxl
    except ImportError:
        return {"error": "openpyxl not installed. Run: pip install openpyxl --break-system-packages"}

    wb = openpyxl.load_workbook(filepath, data_only=True)

    # Resolve sheet
    if isinstance(sheet, int):
        ws = wb.worksheets[sheet]
    else:
        ws = wb[sheet]

    rows_iter = ws.iter_rows(values_only=True)
    headers = [str(c) if c is not None else f"col_{i}" for i, c in enumerate(next(rows_iter, []))]

    rows = []
    for raw in rows_iter:
        row = {headers[i]: (str(v).strip() if v is not None else "") for i, v in enumerate(raw)}
        if any(v for v in row.values()):   # skip blank rows
            rows.append(row)

    return {"sheet": ws.title, "headers": headers, "rows": rows, "row_count": len(rows)}

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print(json.dumps({"error": "Usage: read_excel.py <filepath> [sheet_name_or_index]"}))
        sys.exit(1)
    filepath = sys.argv[1]
    sheet    = sys.argv[2] if len(sys.argv) > 2 else 0
    # Try to coerce to int index if purely numeric
    if isinstance(sheet, str) and sheet.isdigit():
        sheet = int(sheet)
    try:
        result = read_excel(filepath, sheet)
        print(json.dumps(result, ensure_ascii=False, indent=2))
    except Exception as e:
        print(json.dumps({"error": str(e)}))
        sys.exit(1)
```

---

### Script 3 — List Excel Sheets (`list_sheets.py`)

```python
#!/usr/bin/env python3
"""
List all worksheet names in an Excel file.
Usage: python list_sheets.py <filepath>
Output: {"sheets": [...], "sheet_count": N}
"""
import json
import sys

def list_sheets(filepath: str) -> dict:
    try:
        import openpyxl
    except ImportError:
        return {"error": "openpyxl not installed. Run: pip install openpyxl --break-system-packages"}
    wb = openpyxl.load_workbook(filepath, read_only=True, data_only=True)
    return {"sheets": wb.sheetnames, "sheet_count": len(wb.sheetnames)}

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print(json.dumps({"error": "Usage: list_sheets.py <filepath>"}))
        sys.exit(1)
    try:
        print(json.dumps(list_sheets(sys.argv[1]), indent=2))
    except Exception as e:
        print(json.dumps({"error": str(e)}))
        sys.exit(1)
```

---

### Script 4 — Validate & Map Columns (`validate_columns.py`)

```python
#!/usr/bin/env python3
"""
Validate that required columns are present in a list of headers,
using flexible alias matching. Prints a JSON mapping report.

Usage: import and call validate_columns() in-process, or embed inline.
"""
import json

# Default alias maps — extend as needed
ALIAS_MAP = {
    "tag_name":        ["Name", "TagName", "Tag", "PLC_Tag", "Signal"],
    "data_type":       ["DataType", "Type", "DataTypeName", "DType"],
    "logical_address": ["LogicalAddress", "Address", "Addr", "HWAddress"],
    "comment":         ["Comment", "Description", "Remark", "Note"],
    "device_name":     ["DeviceName", "Device", "PLC", "Controller"],
    "order_number":    ["OrderNumber", "MLFB", "ArticleNumber", "PartNumber", "OrderNo"],
    "position":        ["Position", "Location", "Slot", "Rack"],
    "block_name":      ["BlockName", "Block", "FBName", "FCName", "OBName"],
    "block_type":      ["BlockType", "Type", "Kind"],
    "db_name":         ["DBName", "InstanceDB", "DataBlock"],
}

def validate_columns(headers: list[str], required: list[str], optional: list[str] = None) -> dict:
    """
    headers   : column names from CSV / Excel
    required  : canonical field names that MUST be resolved
    optional  : canonical field names that are nice-to-have

    Returns {
        "mapping":  { canonical_name: actual_header },
        "missing":  [ canonical_names_not_found ],
        "warnings": [ canonical_names_optional_not_found ],
        "valid":    bool
    }
    """
    optional = optional or []
    mapping  = {}
    missing  = []
    warnings = []

    lower_headers = {h.lower(): h for h in headers}

    def resolve(canonical: str) -> str | None:
        aliases = ALIAS_MAP.get(canonical, [canonical])
        for alias in aliases:
            if alias.lower() in lower_headers:
                return lower_headers[alias.lower()]
        return None

    for field in required:
        match = resolve(field)
        if match:
            mapping[field] = match
        else:
            missing.append(field)

    for field in optional:
        match = resolve(field)
        if match:
            mapping[field] = match
        else:
            warnings.append(field)

    return {
        "mapping":  mapping,
        "missing":  missing,
        "warnings": warnings,
        "valid":    len(missing) == 0,
    }

if __name__ == "__main__":
    # Quick self-test
    headers = ["TagName", "DataType", "Addr", "Comment"]
    result  = validate_columns(headers, required=["tag_name", "data_type"],
                                        optional=["logical_address", "comment"])
    print(json.dumps(result, indent=2))
```

---

## Orchestration Patterns

### Pattern A — Import Tags from CSV → TIA Portal

```
1. read_csv.py  <tags.csv>
       ↓ JSON rows
2. validate_columns()  [required: tag_name, data_type]
       ↓ mapping
3. MCP: tags_tagtable_list  (confirm target table exists)
4. MCP: tags_create  (loop, one call per row)
5. Report summary
```

**Agent pseudocode:**

```python
# Step 1 – read
rows = run_script("read_csv.py tags.csv")["rows"]

# Step 2 – validate
mapping = validate_columns(rows[0].keys(),
                           required=["tag_name","data_type"],
                           optional=["logical_address","comment"])
if not mapping["valid"]:
    raise ValueError(f"Missing columns: {mapping['missing']}")

# Step 3 – ensure table exists
tables = mcp("tags_tagtable_list", deviceName="PLC_1")
if "GlobalTags" not in [t["name"] for t in tables]:
    mcp("tags_tagtable_create", deviceName="PLC_1", tagTableName="GlobalTags")

# Step 4 – create tags
created, failed = [], []
for row in rows:
    try:
        mcp("tags_create",
            deviceName     = "PLC_1",
            tagTableName   = "GlobalTags",
            tagName        = row[mapping["mapping"]["tag_name"]],
            dataType       = row[mapping["mapping"]["data_type"]],
            logicalAddress = row.get(mapping["mapping"].get("logical_address",""), ""))
        created.append(row[mapping["mapping"]["tag_name"]])
    except Exception as e:
        failed.append({"tag": row[mapping["mapping"]["tag_name"]], "error": str(e)})
```

---

### Pattern B — Import Devices from Excel → TIA Portal

```
1. list_sheets.py  <devices.xlsx>        (show user available sheets)
2. read_excel.py   <devices.xlsx>  <sheet>
       ↓ JSON rows
3. validate_columns()  [required: device_name, order_number]
4. MCP: devices_search_catalog  (validate each order number)
5. MCP: devices_create           (loop)
6. Report summary
```

---

### Pattern C — Generate STL Blocks from Excel → TIA Portal

This pattern chains the **siemens-awl-stl-programmer** skill with MCP block upload tools.

```
1. read_excel.py  <blocks.xlsx>  <sheet>
       ↓ rows describing blocks (name, type, inputs, outputs, logic description)
2. For each row → invoke siemens-awl-stl-programmer skill
       ↓ produces .awl source text
3. write each .awl to a temp file:  /tmp/<BlockName>.awl
4. MCP: blocks_external_source_add    (upload source file)
5. MCP: blocks_source_generate        (compile source into TIA block)
6. MCP: compilation_software          (compile full software)
7. Report summary
```

**Expected Excel schema for block generation:**

| BlockName | BlockType | Inputs | Outputs | StaticVars | LogicDescription |
|---|---|---|---|---|---|
| PumpControl | FB | RunCmd:BOOL,Fault:BOOL | PumpOut:BOOL,Alarm:BOOL | RunLatch:BOOL | Latch on RunCmd, reset on Fault |
| ScaleFlow | FC | RawVal:INT,HiLim:REAL,LoLim:REAL | ScaledVal:REAL | — | Scale 0-27648 to engineering units |

The agent reads each row, builds a natural-language specification, passes it to the
**siemens-awl-stl-programmer** skill, and receives a complete AWL source string in return.

---

### Pattern D — Export Project Blocks to Excel (Reverse Direction)

```
1. MCP: blocks_list                         (enumerate all blocks)
2. MCP: blocks_source_generate_from_block   (get AWL source per block)
3. Parse VAR sections to extract I/O signatures
4. Write summary rows to Excel via openpyxl (see script below)
5. Present file to user
```

**Script 5 — Write Excel Summary (`write_excel_summary.py`):**

```python
#!/usr/bin/env python3
"""
Write a list of dicts to an Excel file.
Usage: import and call write_summary(rows, headers, filepath)
Requires: openpyxl
"""
import json, sys

def write_summary(rows: list[dict], headers: list[str], filepath: str) -> dict:
    try:
        import openpyxl
        from openpyxl.styles import Font, PatternFill, Alignment
    except ImportError:
        return {"error": "openpyxl not installed."}

    wb = openpyxl.Workbook()
    ws = wb.active
    ws.title = "Summary"

    # Header row
    header_font = Font(bold=True, color="FFFFFF")
    header_fill = PatternFill("solid", fgColor="1F4E79")
    for col_idx, header in enumerate(headers, start=1):
        cell = ws.cell(row=1, column=col_idx, value=header)
        cell.font   = header_font
        cell.fill   = header_fill
        cell.alignment = Alignment(horizontal="center")

    # Data rows
    for row_idx, row in enumerate(rows, start=2):
        for col_idx, header in enumerate(headers, start=1):
            ws.cell(row=row_idx, column=col_idx, value=row.get(header, ""))

    # Auto-width columns
    for col in ws.columns:
        max_len = max((len(str(c.value or "")) for c in col), default=10)
        ws.column_dimensions[col[0].column_letter].width = min(max_len + 4, 60)

    wb.save(filepath)
    return {"success": True, "filepath": filepath, "rows_written": len(rows)}

if __name__ == "__main__":
    # Self-test
    sample = [{"BlockName": "PumpControl", "BlockType": "FB", "Inputs": "RunCmd:BOOL"},
              {"BlockName": "ScaleFlow",   "BlockType": "FC", "Inputs": "RawVal:INT"}]
    result = write_summary(sample, ["BlockName","BlockType","Inputs"], "/tmp/test_output.xlsx")
    print(json.dumps(result))
```

---

## MCP Tool Quick Reference

The following tools from `tia-portal-openness-mcp-server` are most relevant to this skill.
Refer to the MCP server documentation for full parameter schemas.

### Project Lifecycle
| Tool | When to use |
|---|---|
| `projects_open` | Ensure a project is open before any write operation |
| `projects_save` | Always save after a batch write operation |
| `projects_get_session_info` | Verify TIA Portal session is active |

### Devices
| Tool | When to use |
|---|---|
| `devices_list` | Enumerate devices; resolve device name before tag/block ops |
| `devices_create` | Create a new PLC or HMI device |
| `devices_search_catalog` | Validate an order number before `devices_create` |
| `devices_get_attributes` | Read device properties |

### Tags
| Tool | When to use |
|---|---|
| `tags_tagtable_list` | Confirm target tag table exists |
| `tags_tagtable_create` | Create missing tag table |
| `tags_create` | Create a single PLC tag |
| `tags_list` | Enumerate tags; check for duplicates before import |

### Blocks
| Tool | When to use |
|---|---|
| `blocks_list` | Enumerate all blocks in a device |
| `blocks_external_source_add` | Upload an AWL/STL source file |
| `blocks_source_generate` | Compile an external source into a TIA block |
| `blocks_source_generate_from_block` | Extract existing block as AWL source |
| `blocks_external_source_generate_all` | Batch-generate all external sources |
| `compilation_software` | Compile after all block changes |

### Utilities
| Tool | When to use |
|---|---|
| `utilities_get_project_info` | Get project name, path, version |
| `sampling_summarize_project` | Get high-level project overview |
| `sampling_get_suggestions` | Get AI suggestions for next steps |

---

## End-to-End Example Conversations

### Example 1 — Batch Tag Import

**User:** Import tags from `io_list.csv` into `PLC_1` GlobalTags table.

**Agent workflow:**
```
1. [bash] python read_csv.py io_list.csv
   → 42 rows, headers: [Name, DataType, Address, Comment]

2. validate_columns → mapping resolved, all required fields present.

3. [MCP] projects_get_session_info → session active, project "Plant_A" open.

4. [MCP] tags_tagtable_list(deviceName="PLC_1")
   → ["GlobalTags", "Safety_Tags"] — GlobalTags exists.

5. [MCP] tags_list(deviceName="PLC_1", tagTableName="GlobalTags")
   → 12 existing tags — none conflict with import.

6. Loop: tags_create × 42
   → 41 created, 1 failed (duplicate: "Motor1_Start" already existed).

7. [MCP] projects_save

8. Report:
   ✓ 41 tags created in PLC_1 → GlobalTags
   ✗ 1 skipped: Motor1_Start (already exists)
```

---

### Example 2 — Generate FB from Excel Row, Upload to TIA Portal

**User:** Generate a function block for each row in `blocks_spec.xlsx` and upload to `PLC_1`.

**Agent workflow:**
```
1. [bash] python list_sheets.py blocks_spec.xlsx
   → sheets: ["FB_Specs", "FC_Specs"]

2. [bash] python read_excel.py blocks_spec.xlsx FB_Specs
   → 3 rows: PumpControl, ValveControl, ConveyorDrive

3. For each row:
   a. Build spec string from columns (BlockName, Inputs, Outputs, LogicDescription)
   b. Invoke siemens-awl-stl-programmer skill → AWL source text
   c. [bash] write AWL to /tmp/<BlockName>.awl

4. [MCP] blocks_external_source_add(deviceName="PLC_1",
         sourcePath="/tmp/PumpControl.awl")   × 3

5. [MCP] blocks_source_generate(deviceName="PLC_1",
         sourceName="PumpControl")            × 3

6. [MCP] compilation_software(deviceName="PLC_1")
   → 0 errors, 2 warnings (review recommended)

7. [MCP] projects_save

8. Report:
   ✓ PumpControl (FB) — generated, compiled
   ✓ ValveControl (FB) — generated, compiled
   ✓ ConveyorDrive (FB) — generated, 2 warnings (see compiler output)
```

---

### Example 3 — Export Block Inventory to Excel

**User:** Give me an Excel overview of all blocks in `PLC_1`.

**Agent workflow:**
```
1. [MCP] blocks_list(deviceName="PLC_1")
   → 18 blocks: [OB1, FB10, FB11, FC20, FC21, DB100, ...]

2. [MCP] blocks_source_generate_from_block × 18
   → AWL source strings; parse VAR_INPUT / VAR_OUTPUT sections

3. Build rows: [{BlockName, BlockType, InputCount, OutputCount, StaticVarCount}]

4. [bash] python write_excel_summary.py  (inline call with rows)
   → /tmp/PLC1_Block_Inventory.xlsx

5. Present file to user.
```

---

### Example 4 — Device Import with Catalog Validation

**User:** Import hardware from `hardware_list.xlsx`, validate against catalog first.

**Agent workflow:**
```
1. [bash] python list_sheets.py hardware_list.xlsx
   → sheets: ["Devices"]

2. [bash] python read_excel.py hardware_list.xlsx Devices
   → 8 rows, headers: [DeviceName, OrderNumber, Position, Comment]

3. validate_columns → device_name, order_number resolved.

4. Pre-validation loop: devices_search_catalog × 8
   → 7 found, 1 not found (row 6: 6ES7 999-0000-0AB0)

   Report to user:
   ⚠ Row 6 order number not found in catalog. Continue with 7 valid devices? (yes/no)

5. [User: yes]

6. Loop: devices_create × 7 — all succeed.

7. [MCP] projects_save

8. Report:
   ✓ 7 devices created
   ✗ 1 skipped (catalog miss — row 6)
```

---

## Validation Rules

Apply these checks before any MCP write call:

**Tag names:**
- Max 24 characters
- First character must be a letter or underscore
- Allowed: letters, digits, underscore — no spaces or special characters

**Data types (TIA Portal):**
- `Bool`, `Byte`, `Word`, `DWord`, `Int`, `DInt`, `Real`, `LReal`, `Time`, `Char`, `String`
- Case-sensitive in some TIA versions — use the exact capitalisation above

**Logical addresses:**
- Inputs: `%I<byte>.<bit>` or `%IW<byte>` for words
- Outputs: `%Q<byte>.<bit>` or `%QW<byte>`
- Memory: `%M<byte>.<bit>` or `%MW<byte>` or `%MD<byte>`

**Order numbers:**
- Pattern: `[0-9][A-Z0-9]{3} [0-9A-Z]{3}-[0-9A-Z]{4,6}-[0-9][A-Z0-9]{3}`
- Example: `6ES7 516-3AN02-0AB0`

**Block names:**
- Max 24 characters
- No special characters except underscore
- Must be unique within the device

---

## Skill Interaction Map

```
siemens-tia-portal-integrator
         │
         ├── File I/O layer
         │       ├── read_csv.py
         │       ├── read_excel.py
         │       ├── list_sheets.py
         │       ├── validate_columns.py
         │       └── write_excel_summary.py
         │
         ├── Code Generation (delegate)
         │       └── siemens-awl-stl-programmer skill
         │               └── produces .awl / .db source text
         │
         └── Project Automation (delegate)
                 └── tia-portal-openness-mcp-server MCP tools
                         ├── projects_*
                         ├── devices_*
                         ├── tags_*
                         ├── blocks_*
                         └── compilation_*
```

---

## Error Handling Reference

| Situation | Agent Action |
|---|---|
| File not found | Report path, ask user to verify |
| Missing required column | Show found columns, ask for mapping |
| Order number not in catalog | Warn, ask to skip or abort |
| Tag already exists | Skip with warning, continue loop |
| Block compile error | Show compiler output, do not save |
| No TIA Portal session | Call `projects_open` or instruct user |
| openpyxl not installed | Run `pip install openpyxl --break-system-packages` |

---

## Best Practices

1. **Always open and verify the project first** — call `projects_get_session_info` or `utilities_get_project_info` before any write.
2. **Validate before writing** — run `validate_columns` and catalog checks before the first MCP write call; prevent partial imports.
3. **One MCP call per item** — loop individually; never construct bulk payloads unless the tool explicitly supports them.
4. **Save after each batch** — call `projects_save` after every successful write batch.
5. **Compile after block changes** — always call `compilation_software` after uploading external sources.
6. **Delegate code generation** — do not write AWL inline; always invoke the `siemens-awl-stl-programmer` skill for any block source.
7. **Continue on single failures** — do not abort an import loop on one bad row; collect all failures and report at the end.
8. **Preserve accumulator order** — when reviewing or patching generated AWL, verify subtraction and division instruction order before upload (see `reference/STL_Core_Instructions.md`).
