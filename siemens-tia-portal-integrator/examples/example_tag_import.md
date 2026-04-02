# Example — Batch Tag Import from CSV

Demonstrates Pattern A end-to-end.

---

## Conversation

**User:**
> Import tags from `io_list.csv` into `PLC_1` GlobalTags table.

---

**Agent step 1 — Read file**

```
files_read_csv(filePath="io_list.csv", delimiter=",")
```

Result:
```
42 rows — headers: [Name, DataType, Address, Comment]
```

---

**Agent step 2 — Validate columns**

```python
validate_columns(
    headers  = ["Name", "DataType", "Address", "Comment"],
    required = ["tag_name", "data_type"],
    optional = ["logical_address", "comment"],
)
```

Result:
```json
{
  "mapping": {
    "tag_name":        "Name",
    "data_type":       "DataType",
    "logical_address": "Address",
    "comment":         "Comment"
  },
  "missing":  [],
  "valid":    true
}
```

---

**Agent step 3 — Verify session**

```
projects_get_session_info()
```

Result: Session active, project "Plant_A" open.

---

**Agent step 4 — Confirm tag table**

```
tags_tagtable_list(deviceName="PLC_1")
```

Result: `["GlobalTags", "Safety_Tags"]` — GlobalTags exists ✓

---

**Agent step 5 — Check for duplicates**

```
tags_list(deviceName="PLC_1", tagTableName="GlobalTags")
```

Result: 12 existing tags — none conflict with the 42 import rows ✓

---

**Agent step 6 — Create tags (loop × 42)**

```
tags_create(deviceName="PLC_1", tagTableName="GlobalTags",
            tagName="Motor1_Start", dataType="Bool", logicalAddress="%I0.0")
…
tags_create(…  tagName="Motor1_Start" …)   ← row 27: DUPLICATE — already added in row 1
```

Row 27 fails — agent logs and continues.

---

**Agent step 7 — Save**

```
projects_save()
```

---

**Agent response:**

```
✓ Tag import completed

Total rows : 42
Created    : 41
Failed     : 1

Failed:
  • Motor1_Start (row 27) — Tag already exists in table

Recommendation:
  - Remove duplicate row from io_list.csv

Next steps:
  - Compile software: compilation_software(deviceName="PLC_1")
```
