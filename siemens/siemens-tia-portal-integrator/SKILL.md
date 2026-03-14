---
name: siemens-tia-portal-integrator
description: >
  Orchestrate data-driven Siemens TIA Portal workflows: read structured data from CSV/Excel files,
  generate STL/AWL code via the siemens-awl-stl-programmer skill, and apply changes to a live TIA
  Portal project via the tia-portal-openness-mcp-server. Trigger when the user asks to import,
  generate, sync, or batch-apply PLC blocks, tags, devices, or data blocks from a spreadsheet or CSV
  source — or to export existing TIA Portal project data into a file.
---

# Siemens TIA Portal Integrator Skill

## Overview

This skill bridges three layers:

| Layer | Tool | Purpose |
|---|---|---|
| **File I/O** | Python scripts + MCP file tools | Read/write CSV and Excel |
| **Code Generation** | `siemens-awl-stl-programmer` skill | Produce compilable STL/AWL source |
| **Project Automation** | `tia-portal-openness-mcp-server` | Apply changes to a live TIA Portal project |

**Always** invoke the `siemens-awl-stl-programmer` skill when the workflow involves generating or
modifying STL/AWL block source. Treat MCP tool calls as the final "write" step — validate data
completely before applying any changes to the project.

---

## Skill Map

```
siemens-tia-portal-integrator/
│
├── SKILL.md                         ← you are here (index + quick-reference)
│
├── scripts/
│   ├── read_csv.md                  ← Read CSV → JSON rows
│   ├── read_excel.md                ← Read Excel sheet → JSON rows
│   ├── list_sheets.md               ← List worksheet names
│   ├── validate_columns.md          ← Alias-based column mapping & validation
│   └── write_excel_summary.md       ← Write rows back to an Excel file
│
├── patterns/
│   ├── import_tags_csv.md           ← Pattern A: CSV tags → TIA Portal
│   ├── import_devices_excel.md      ← Pattern B: Excel devices → TIA Portal
│   ├── generate_blocks_excel.md     ← Pattern C: Excel spec → STL → TIA Portal
│   └── export_blocks_excel.md       ← Pattern D: TIA Portal blocks → Excel
│
└── examples/
    ├── example_tag_import.md        ← Full conversation: tag batch import
    ├── example_device_import.md     ← Full conversation: device import + catalog check
    ├── example_block_generation.md  ← Full conversation: FB generation from Excel
    └── example_block_export.md      ← Full conversation: export block inventory
```

---

## MCP File Tools (prefer over Python scripts when available)

The MCP server already exposes file tools. Use these **first** before falling back to Python scripts.

| MCP Tool | Equivalent Python Script | Use When |
|---|---|---|
| `files_read_csv` | `scripts/read_csv.md` | Read CSV file |
| `files_read_excel` | `scripts/read_excel.md` | Read Excel sheet |
| `files_list_sheets` | `scripts/list_sheets.md` | List worksheets |
| `files_validate_format` | — | Structural CSV/Excel check |
| `files_get_info` | — | File metadata |

Use the **Python scripts** only when MCP file tools are unavailable or when custom preprocessing
(e.g., alias-based column mapping) is needed.

---

## MCP Tool Quick Reference

### Project Lifecycle
| Tool | When to use |
|---|---|
| `projects_get_session_info` | Verify TIA Portal session is active before any operation |
| `projects_open` | Open a project if none is active |
| `projects_save` | **Always call after every batch write operation** |
| `projects_close` | Close project when done |

### Devices
| Tool | When to use |
|---|---|
| `devices_list` | Enumerate devices; resolve device name before tag/block ops |
| `devices_create` | Create a new PLC or HMI device |
| `devices_search_catalog` | Validate order number before `devices_create` |
| `devices_get_attributes` | Read device properties |

### Tags
| Tool | When to use |
|---|---|
| `tags_tagtable_list` | Confirm target tag table exists |
| `tags_tagtable_create` | Create missing tag table |
| `tags_list` | Enumerate tags; check for duplicates before import |
| `tags_create` | Create a single PLC tag |

### Blocks
| Tool | When to use |
|---|---|
| `blocks_list` | Enumerate all blocks in a device |
| `blocks_external_source_add` | Upload an AWL/STL source file |
| `blocks_source_generate` | Compile external source into a TIA block |
| `blocks_source_generate_from_block` | Extract existing block as AWL source |
| `blocks_external_source_generate_all` | Batch-compile all uploaded sources |
| `software_add_block` | Add a new empty block shell |
| `compilation_software` | Compile device software after block changes |

### Utilities
| Tool | When to use |
|---|---|
| `utilities_get_project_info` | Get project name, path, TIA version |
| `sampling_summarize_project` | High-level project overview |
| `sampling_generate_code` | AI-assisted code generation within TIA context |
| `sampling_get_suggestions` | Get suggested next steps |

---

## Validation Rules (apply before any MCP write call)

**Tag names**
- Max 24 characters; first character must be a letter or underscore
- Allowed: letters, digits, underscore — no spaces or special characters

**Data types**
- `Bool` `Byte` `Word` `DWord` `Int` `DInt` `Real` `LReal` `Time` `Char` `String`
- Capitalisation is exact — use these spellings

**Logical addresses**
- Inputs: `%I<byte>.<bit>` or `%IW<byte>`
- Outputs: `%Q<byte>.<bit>` or `%QW<byte>`
- Memory: `%M<byte>.<bit>` or `%MW<byte>` or `%MD<byte>`

**Order numbers**
- Example: `6ES7 516-3AN02-0AB0`
- Pattern: `[Family 4 chars][Space][Article]-[SubArticle]-[Variant]`

**Block names**
- Max 24 characters; no special characters except underscore; unique within the device

---

## General Best Practices

1. **Verify session first** — call `projects_get_session_info` before any write.
2. **Validate before writing** — run column validation and catalog checks before the first MCP write.
3. **Ask on ambiguity** — if a requirement or target (device, sheet, table, mapping, action) is unclear or missing, ask the user before proceeding; do not assume.
4. **One MCP call per item** — loop individually; no bulk payloads.
5. **Save after each batch** — call `projects_save` after every successful write batch.
6. **Compile after block changes** — always call `compilation_software` after uploading sources.
7. **Delegate code generation** — never write AWL inline; invoke `siemens-awl-stl-programmer` skill.
8. **Continue on single failures** — collect all failures and report at the end, do not abort early.
