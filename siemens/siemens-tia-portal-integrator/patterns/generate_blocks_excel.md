# Pattern C — Generate STL Blocks from Excel → TIA Portal

Read a block specification table from Excel, delegate AWL/STL code generation to the
`siemens-awl-stl-programmer` skill, upload each source file, compile, and save.

## Trigger Phrases

- "Generate function blocks from Excel"
- "Create PLC blocks from spreadsheet spec"
- "Build FB/FC from specification table"
- "Auto-generate blocks from file"

---

## Expected Excel Schema

First row must be a header row. Required and optional columns:

| Column (canonical) | Aliases | Required |
|---|---|---|
| `block_name` | BlockName, Block, FBName, FCName | ✓ |
| `block_type` | BlockType, Type, Kind | ✓ |
| `inputs` | Inputs, VarInput, IN | optional |
| `outputs` | Outputs, VarOutput, OUT | optional |
| `static_vars` | StaticVars, Static, VarStatic | optional |
| `logic` | LogicDescription, Logic, Description | optional |
| `db_name` | DBName, InstanceDB | optional (FB only) |

**Example row:**

| BlockName | BlockType | Inputs | Outputs | StaticVars | LogicDescription |
|---|---|---|---|---|---|
| PumpControl | FB | RunCmd:BOOL, Fault:BOOL | PumpOut:BOOL, Alarm:BOOL | RunLatch:BOOL | Latch on RunCmd, reset on Fault |

---

## Step-by-Step Orchestration

### 1. Read and validate the sheet

```
files_list_sheets(filePath="<path>")
files_read_excel(filePath="<path>", sheetName="<sheet>")
```

```python
result = validate_columns(
    headers  = <headers>,
    required = ["block_name", "block_type"],
    optional = ["inputs", "outputs", "static_vars", "logic", "db_name"],
)
```

---

### 2. Verify TIA Portal session

```
projects_get_session_info()
devices_list()   ← confirm target device
```

---

### 3. For each row: build spec and invoke siemens-awl-stl-programmer skill

Construct a natural-language specification string from the row columns:

```python
spec = f"""
Block name : {row['BlockName']}
Block type : {row['BlockType']}
Inputs     : {row.get('Inputs', 'none')}
Outputs    : {row.get('Outputs', 'none')}
Static vars: {row.get('StaticVars', 'none')}
Logic      : {row.get('LogicDescription', 'implement basic structure')}
"""
```

Pass `spec` to the **siemens-awl-stl-programmer** skill.  
Receive back complete AWL source text.

---

### 4. Write AWL source to a temp file

```python
source_path = f"/tmp/{row['BlockName']}.awl"
with open(source_path, "w") as f:
    f.write(awl_source)
```

---

### 5. Upload source to TIA Portal

```
blocks_external_source_add(
    deviceName = "<device>",
    sourcePath = "/tmp/<BlockName>.awl"
)
```

---

### 6. Compile source into a TIA block

```
blocks_source_generate(
    deviceName = "<device>",
    sourceName = "<BlockName>"
)
```

If compilation returns errors → present compiler output to user; do not proceed to `projects_save`.

---

### 7. Compile full software and save

After all blocks are uploaded:

```
compilation_software(deviceName="<device>")
projects_save()
```

---

### 8. Report

```
✓ Block generation completed

Blocks processed : 3
Generated        : 3
Compile warnings : 2  (see details below)
Compile errors   : 0

  • PumpControl  (FB)  — ✓ compiled
  • ValveControl (FB)  — ✓ compiled, 2 warnings
  • ScaleFlow    (FC)  — ✓ compiled

Next steps:
  - Review compiler warnings for ValveControl
  - Download to PLC
```

---

## Error Handling

| Situation | Agent Action |
|---|---|
| Missing block_name or block_type column | Stop; report to user |
| siemens-awl-stl-programmer returns empty | Retry with more detailed spec; report if still failing |
| blocks_external_source_add fails | Log; skip block; continue loop |
| blocks_source_generate compile error | Show output; do not save; ask user to review |
| No TIA Portal session | Call `projects_open` or instruct user |
