# Siemens AWL/STL Programmer Skill

Purpose
-------
This skill helps an agent generate Siemens S7 AWL (also called STL) code snippets and stub files for PLC projects. It's intended for producing Function (`FC`), Function Block (`FB`), Organization Block (`OB`), and UDT examples in AWL format compatible with Step7-style projects.

File Structure
--------------
```
siemens-awl-stl-programmer/
├── SKILL.md                    # Skill metadata and core instructions
├── references/
│   ├── FORMATTING.md           # Formatting rules
│   ├── REFERENCE.md             # Block templates
│   └── STL_Core_Instructions.md # Full instruction reference
└── assets/
    └── templates/              # Standalone block templates
        ├── Function.awl
        ├── FunctionBlock.awl
        ├── OrganizationBlock.awl
        ├── UserDefinedType.udt
        └── ...
```

Templates available
-------------------
- `assets/templates/Function.awl` — sample `FUNCTION` AWL file
- `assets/templates/FunctionBlock.awl` — sample `FUNCTION_BLOCK` AWL file
- `assets/templates/OrganizationBlock.awl` — sample `ORGANIZATION_BLOCK` AWL file
- `assets/templates/UserDefinedType.udt` — sample user-defined type

Conventions
-----------
- Include `{ S7_Optimized_Access := 'TRUE' }` and `VERSION : 0.1`
- Use quoted symbolic names (`"BlockName"`) — do not use numeric block references
- Use `VAR_INPUT`, `VAR_OUTPUT`, `VAR_TEMP` (or `VAR`) sections followed by `BEGIN` / `END_FUNCTION`
- Add `NETWORK` blocks with `TITLE = ...` and `//` comments

Notes for contributors
---------------------
- Keep generated AWL readable and follow the style in `references/FORMATTING.md`
- Reference files are in `references/` directory per Agent Skills spec
- Template files for direct use go in `assets/templates/`

See also
--------
- `SKILL.md` — skill metadata and core instructions
- `references/FORMATTING.md` — detailed formatting rules
- `references/REFERENCE.md` — block templates
- `references/STL_Core_Instructions.md` — full instruction reference
