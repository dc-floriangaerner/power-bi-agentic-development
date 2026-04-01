# PBIP Validation Hooks

PostToolUse hooks that validate PBIR and TMDL files after Write, Edit, and Bash tool use.

## Hook files

| Hook | Trigger | Scope |
|---|---|---|
| `validate-pbir.sh` | Write, Edit, Bash | .json/.pbir files in .Report/, .SemanticModel/, .Dataset/ |
| `validate-report-binding.sh` | Write, Edit, Bash | definition.pbir binding validation (byPath/byConnection) |
| `validate-tmdl.sh` | Write, Edit, Bash | .tmdl files in .SemanticModel/ or .Dataset/ |

## Checks

All checks are toggleable via `config.yaml`. Set any key to `false` to disable.

| Config key | Check | Hooks |
|---|---|---|
| `json_syntax` | JSON syntax (`jq empty`) | validate-pbir |
| `folder_spaces` | Folder names with spaces (won't render) | validate-pbir |
| `required_fields` | Required fields per file type (from Microsoft JSON schemas) | validate-pbir |
| `schema_url` | `$schema` URL matches expected pattern | validate-pbir |
| `name_format` | Visual/page name is word chars and hyphens only | validate-pbir |
| `bypath_exists` | byPath target directory exists locally | validate-report-binding |
| `fab_exists` | byConnection model exists in Fabric (via `fab exists`) | validate-report-binding |
| `tmdl_syntax` | TMDL structural syntax (via `tmdl-validate`) | validate-tmdl |

## Required fields (from schemas)

Derived from Microsoft's published JSON schemas at [github.com/microsoft/json-schemas](https://github.com/microsoft/json-schemas/tree/main/fabric/item/report/definition).

| File | Schema | Required |
|---|---|---|
| `visual.json` | visualContainer/2.7.0 | `$schema`, `name`, `position` + oneOf(`visual`, `visualGroup`) |
| `page.json` | page/2.1.0 | `$schema`, `name`, `displayName`, `displayOption` |
| `report.json` | report/3.2.0 | `$schema`, `themeCollection` |
| `definition.pbir` | definitionProperties/2.0.0 | `$schema`, `version`, `datasetReference` |

## Graceful degradation

- If `jq` is not installed, all hooks skip silently
- If `fab` CLI is not installed or not authenticated, the `fab_exists` check skips silently
- If `tmdl-validate` binary is not found, TMDL hooks skip silently
- If `config.yaml` is missing, all checks default to enabled

## Updating required fields when schemas change

1. Fetch the latest schema:
   ```
   https://raw.githubusercontent.com/microsoft/json-schemas/main/fabric/item/report/definition/{schemaType}/{version}/schema.json
   ```
   For definition.pbir: `.../definitionProperties/{version}/schema.json`

2. Find the top-level `"required"` array. Check `oneOf`/`anyOf` constraints and nested `$ref` definitions.

3. Update the jq extraction and MISSING checks in `validate-pbir.sh`.

4. Update the table above.

5. Test: `echo '{"tool_name":"Write","tool_input":{"file_path":"<path>"}}' | bash plugins/pbip/hooks/validate-pbir.sh`

## Constraints

- 10-second timeout per hook; jq calls are consolidated (one per file type)
- Must work on bash 3.2 (macOS) and bash 4+ (Git Bash / Linux); no `mapfile`, no associative arrays
- `.gitattributes` enforces `eol=lf` so scripts work on Windows checkout
- Only exit 2 + stderr surfaces in Claude Code; stdout exit 0 is invisible
- Full semantic validation (visual types, expressions, field references) belongs in pbir-cli
