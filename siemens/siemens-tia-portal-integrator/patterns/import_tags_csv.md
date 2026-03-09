# Pattern A — Import Tags from CSV → TIA Portal

Import PLC tags from a CSV or Excel file into a tag table in a live TIA Portal project.

## Trigger Phrases

- "Import tags from CSV / Excel"
- "Load tags from a file"
- "Create tags from spreadsheet"
- "Batch import tags"

---

## Step-by-Step Orchestration

### 1. Read the file

**Prefer MCP tools** when available:

```
files_read_csv(filePath="<path>", delimiter=",")
```

or for Excel:

```
files_list_sheets(filePath="<path>")   ← ask user to pick sheet if > 1
files_read_excel(filePath="<path>", sheetName="<sheet>")
```

Fall back to `scripts/read_csv.md` or `scripts/read_excel.md` if MCP tools are unavailable.

---

### 2. Validate columns

Call `validate_columns()` (see `scripts/validate_columns.md`):

```python
result = validate_columns(
    headers  = <headers from step 1>,
    required = ["tag_name", "data_type"],
    optional = ["logical_address", "comment"],
)
```

If `result["valid"]` is `False` → report missing columns to user and stop.  
If warnings exist → note optional fields are absent and continue.

---

### 3. Verify TIA Portal session

```
projects_get_session_info()
```

If no session → call `projects_open()` or instruct the user to open a project.

---

### 4. Resolve target device and tag table

```
devices_list()
```

Ask user which device to target if not already specified.

```
tags_tagtable_list(deviceName="<device>")
```

- If target table exists → proceed
- If target table does not exist → offer to create it:

```
tags_tagtable_create(deviceName="<device>", tagTableName="<name>")
```

---

### 5. Check for duplicates (optional but recommended)

```
tags_list(deviceName="<device>", tagTableName="<table>")
```

Compare existing tag names with import rows. Report conflicts and ask:
- Skip duplicates (recommended default)
- Overwrite (if supported)
- Abort

---

### 6. Create tags (loop)

```python
created, failed = [], []
mapping = result["mapping"]

for row in rows:
    tag_name = row[mapping["tag_name"]]
    try:
        tags_create(
            deviceName     = device,
            tagTableName   = table,
            tagName        = tag_name,
            dataType       = row[mapping["data_type"]],
            logicalAddress = row.get(mapping.get("logical_address", ""), ""),
        )
        created.append(tag_name)
    except Exception as e:
        failed.append({"tag": tag_name, "error": str(e)})
```

Continue on individual failures — do not abort the loop.

---

### 7. Save and report

```
projects_save()
```

Report summary:

```
✓ Tag import completed

Total rows : 42
Created    : 41
Failed     : 1

Failed:
  • Motor1_Start — Tag already exists in table

Next steps:
  - Review failed tags
  - Compile software: compilation_software(deviceName="PLC_1")
```

---

## Validation Rules for Tags

| Field | Rule |
|---|---|
| Tag name | Max 24 chars; letters, digits, underscore only; first char not a digit |
| Data type | Exact spelling: `Bool` `Byte` `Word` `DWord` `Int` `DInt` `Real` `LReal` `Time` `Char` `String` |
| Logical address | Format: `%I0.0` `%IW2` `%Q0.0` `%QW4` `%M10.0` `%MW20` `%MD100` |

---

## Error Handling

| Situation | Agent Action |
|---|---|
| File not found | Report path; ask user to verify |
| Missing required column | Show found columns; ask for correct mapping |
| No TIA Portal session | Call `projects_open` or instruct user |
| Device not found | List available devices; ask user to choose |
| Tag table not found | Offer to create it with `tags_tagtable_create` |
| Tag already exists | Skip with warning; continue loop |
| Invalid data type | Suggest correct capitalisation; skip row |
