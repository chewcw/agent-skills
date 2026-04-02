# Siemens AWL/STL Programmer Skill

Purpose
-------
This skill helps an agent generate Siemens S7 AWL (also called STL) code snippets and stub files for PLC projects. It's intended for producing Function (`FC`), Function Block (`FB`), Organization Block (`OB`), and UDT examples in AWL format compatible with Step7-style projects.

Examples included
-----------------
- `example_blocks/Function.awl` — sample `FUNCTION` AWL file.
- `example_blocks/FunctionBlock.awl` — sample `FUNCTION_BLOCK` AWL file.
- `example_blocks/OrganizationBlock.awl` — sample `ORGANIZATION_BLOCK` AWL file.
- `example_blocks/UserDefinedType.udt` — sample user-defined type.

Conventions and templates
-------------------------
- Include header fields: `{ S7_Optimized_Access := 'TRUE' }`, `NAME`, and `VERSION`.
- Use `VAR_INPUT`, `VAR_OUTPUT`, `VAR_TEMP` (or `VAR`) sections followed by `BEGIN` / `END_FUNCTION` (or `END_FUNCTION_BLOCK` / `END_ORGANIZATION_BLOCK`).
- Add `NETWORK` blocks with `TITLE` comments for clarity.

Quick generation example
------------------------
Create a file with a structure like:

```
FUNCTION "MyFunction" : Void
{ S7_Optimized_Access := 'TRUE' }
NAME : '1'
VERSION : 0.1

VAR_INPUT
   IN : Int;
END_VAR

VAR_OUTPUT
   OUT : Real;
END_VAR

BEGIN
NETWORK
TITLE = Example
   L #IN;
   ITD;
   DTR;
   T #OUT;

END_FUNCTION
```

Notes for contributors
----------------------
- Keep generated AWL readable and follow the style used in `example_blocks` for consistency.
- If adding more reference material, put it under `reference/`.

See also
--------
- `SKILL.md` for the skill's metadata and internal guidance.
