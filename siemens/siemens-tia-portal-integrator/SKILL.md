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
| **Project Automation** | `tia-portal-openness-mcp-server` (`mcp_test-tia-port_*`) | Apply changes to a live TIA Portal project |

**Always** invoke the `siemens-awl-stl-programmer` skill when the workflow involves generating or
modifying STL/AWL block source. Treat MCP tool calls as the final "write" step — validate data
completely before applying any changes to the project.

In this environment, the TIA Portal MCP server exposes tool IDs with the `mcp_test-tia-port_`
prefix. Use the exact tool IDs below rather than shorthand aliases.

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
| `mcp_test-tia-port_files_read_csv` | `scripts/read_csv.md` | Read CSV file |
| `mcp_test-tia-port_files_read_excel` | `scripts/read_excel.md` | Read Excel sheet |
| `mcp_test-tia-port_files_list_sheets` | `scripts/list_sheets.md` | List worksheets |
| `mcp_test-tia-port_files_validate_format` | — | Structural CSV/Excel check |
| `mcp_test-tia-port_files_get_info` | — | File metadata |

Use the **Python scripts** only when MCP file tools are unavailable or when custom preprocessing
(e.g., alias-based column mapping) is needed.

---

## TIA Portal MCP Tool Reference

The following list reflects the current tool surface exposed by the TIA Portal MCP server in this
environment.

### Project Lifecycle
| Tool | When to use |
|---|---|
| `mcp_test-tia-port_projects_get_session_info` | Verify TIA Portal session is active before any operation |
| `mcp_test-tia-port_projects_create` | Create a new TIA Portal project |
| `mcp_test-tia-port_projects_open` | Open a project if none is active |
| `mcp_test-tia-port_projects_open_with_upgrade` | Open and upgrade an older project |
| `mcp_test-tia-port_projects_save` | **Always call after every batch write operation** |
| `mcp_test-tia-port_projects_close` | Close project when done |

### Devices
| Tool | When to use |
|---|---|
| `mcp_test-tia-port_devices_list` | Enumerate devices; resolve device name before tag or block operations |
| `mcp_test-tia-port_devices_create` | Create a new PLC or HMI device |
| `mcp_test-tia-port_devices_delete` | Delete an existing device |
| `mcp_test-tia-port_devices_search_catalog` | Validate order number before device creation |
| `mcp_test-tia-port_catalog_search_device_items` | Search device and module catalog entries |
| `mcp_test-tia-port_devices_get_attributes` | Read device properties |
| `mcp_test-tia-port_devices_set_attribute` | Update a writable device attribute |
| `mcp_test-tia-port_devices_get_app_id` | Read the device App ID |
| `mcp_test-tia-port_devices_set_app_id` | Set the device App ID |

### Device Items
| Tool | When to use |
|---|---|
| `mcp_test-tia-port_deviceitems_list` | Enumerate hardware items in a device |
| `mcp_test-tia-port_deviceitems_get_attributes` | Read a specific hardware item's attributes |
| `mcp_test-tia-port_deviceitems_copy` | Copy a hardware item to a new position |
| `mcp_test-tia-port_deviceitems_plug_move` | Move a hardware item to a new position |
| `mcp_test-tia-port_deviceitems_delete` | Delete a hardware item |

### Blocks
| Tool | When to use |
|---|---|
| `mcp_test-tia-port_blocks_list` | Enumerate all PLC blocks in a device |
| `mcp_test-tia-port_blocks_system_types_list` | Enumerate available system-defined data types |
| `mcp_test-tia-port_software_add_block` | Add a new empty block shell |
| `mcp_test-tia-port_software_get_block_hierarchy` | Inspect the PLC block hierarchy |
| `mcp_test-tia-port_blocks_editor_open_block` | Open a block in the TIA Portal editor |
| `mcp_test-tia-port_blocks_editor_open_type` | Open a UDT in the TIA Portal editor |
| `mcp_test-tia-port_blocks_fingerprints_get` | Retrieve block or UDT interface fingerprints |
| `mcp_test-tia-port_blocks_ob_priority_set` | Set an OB execution priority |
| `mcp_test-tia-port_blocks_source_generate_from_block` | Extract an existing block as source text |

### External Sources
| Tool | When to use |
|---|---|
| `mcp_test-tia-port_blocks_external_source_add` | Upload an AWL or STL source file |
| `mcp_test-tia-port_blocks_external_source_delete` | Remove an uploaded external source |
| `mcp_test-tia-port_blocks_source_generate` | Compile one external source into a TIA block |
| `mcp_test-tia-port_blocks_external_source_generate_all` | Batch-compile all uploaded external sources |
| `mcp_test-tia-port_blocks_external_source_generate_in_group` | Batch-compile external sources within a group |
| `mcp_test-tia-port_blocks_external_source_generate_with_options` | Compile external sources with explicit options |
| `mcp_test-tia-port_blocks_udt_delete` | Delete a UDT created from source |

### Tags
| Tool | When to use |
|---|---|
| `mcp_test-tia-port_tags_group_system_get` | Inspect the top-level system tag group |
| `mcp_test-tia-port_tags_group_list` | Enumerate user-defined tag groups |
| `mcp_test-tia-port_tags_group_find` | Resolve a specific tag group |
| `mcp_test-tia-port_tags_group_create` | Create a tag group |
| `mcp_test-tia-port_tags_group_delete` | Delete a tag group |
| `mcp_test-tia-port_tags_tagtable_list` | Confirm which tag tables exist |
| `mcp_test-tia-port_tags_tagtable_get` | Read a tag table's metadata |
| `mcp_test-tia-port_tags_tagtable_create` | Create a missing tag table |
| `mcp_test-tia-port_tags_tagtable_delete` | Delete a tag table |
| `mcp_test-tia-port_tags_tagtable_export` | Export a tag table |
| `mcp_test-tia-port_tags_tagtable_open_editor` | Open a tag table in the editor |
| `mcp_test-tia-port_tags_list` | Enumerate tags; check for duplicates before import |
| `mcp_test-tia-port_tags_create` | Create a single PLC tag |
| `mcp_test-tia-port_tags_constants_system_list` | Enumerate system constants in a tag table |
| `mcp_test-tia-port_tags_constants_user_list` | Enumerate user-defined constants in a tag table |

### HMI
| Tool | When to use |
|---|---|
| `mcp_test-tia-port_hmi_targets_list` | Enumerate HMI targets in the project |
| `mcp_test-tia-port_hmi_targets_get` | Read HMI target metadata |
| `mcp_test-tia-port_hmi_targets_validate` | Confirm a device is an HMI target before HMI-specific operations |

### ProDiag
| Tool | When to use |
|---|---|
| `mcp_test-tia-port_blocks_prodiag_assigned_get` | Read the ProDiag FB assigned to a member |
| `mcp_test-tia-port_blocks_prodiag_assigned_set` | Assign a ProDiag FB to a member |
| `mcp_test-tia-port_blocks_prodiag_attributes_get` | Read ProDiag FB attributes |
| `mcp_test-tia-port_blocks_prodiag_export_csv` | Export ProDiag alarm messages to CSV |

### Compilation
| Tool | When to use |
|---|---|
| `mcp_test-tia-port_compilation_software` | Compile device software after block changes |
| `mcp_test-tia-port_compilation_project` | Compile the full project |

### Utilities
| Tool | When to use |
|---|---|
| `mcp_test-tia-port_utilities_get_project_info` | Get project name, path, TIA version |
| `mcp_test-tia-port_utilities_list_libraries` | Enumerate accessible TIA libraries |
| `mcp_test-tia-port_utilities_elicit_user_input` | Prompt the user for required missing inputs |
| `mcp_test-tia-port_hello_world` | Connectivity smoke test |

### Development Assistance
| Tool | When to use |
|---|---|
| `mcp_test-tia-port_sampling_summarize_project` | High-level project overview |
| `mcp_test-tia-port_sampling_generate_code` | AI-assisted code generation within TIA context |
| `mcp_test-tia-port_sampling_get_suggestions` | Get suggested next steps |

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

1. **Verify session first** — call `mcp_test-tia-port_projects_get_session_info` before any write.
2. **Validate before writing** — run column validation and catalog checks before the first MCP write.
3. **Ask on ambiguity** — if a requirement or target (device, sheet, table, mapping, action) is unclear or missing, ask the user before proceeding; do not assume.
4. **One MCP call per item** — loop individually; no bulk payloads.
5. **Save after each batch** — call `mcp_test-tia-port_projects_save` after every successful write batch.
6. **Compile after block changes** — always call `mcp_test-tia-port_compilation_software` after uploading sources.
7. **Delegate code generation** — never write AWL inline; invoke `siemens-awl-stl-programmer` skill.
8. **Continue on single failures** — collect all failures and report at the end, do not abort early.
