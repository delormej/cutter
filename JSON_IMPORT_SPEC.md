# JSON Import Specification

## Purpose

Add a JSON import capability so users can load a project containing cuts and stock types.

The import should support:

- Project metadata.
- Units.
- Kerf settings.
- Stock types, defined by thickness and material group.
- Optional stock catalog entries.
- Required cuts.
- Future Auto Stock workflows.

The app should also include a button that copies the supported JSON schema/example to the clipboard.

## Current App Context

The current app stores stock and cuts as UI rows with these fields:

```js
{
  name,
  qty,
  w,
  h,
  thickness,
  group
}
```

The import format should map cleanly into these fields while leaving room for future capabilities from `AUTO_STOCK_SPEC.md`, including:

- Sheet goods.
- Exact-width linear lumber.
- Ripped linear lumber.
- Stock catalogs.
- Auto Stock mode.

## User-Facing Goals

The user should be able to:

- Click `Import JSON`.
- Select or paste a JSON document.
- Preview validation errors before applying import.
- Import cuts into the app.
- Import stock types such as `3/4 plywood` or `3/4 painted trim`.
- Optionally import manual stock rows.
- Optionally import stock catalog entries for Auto Stock.
- Click `Copy JSON Schema` to copy the supported schema/example to the clipboard.

## Terminology

### Stock Type

A material need shared by cuts.

It is primarily defined by:

```text
thickness + material
```

Examples:

```text
3/4 + plywood
3/4 + painted trim
1/2 + baltic birch
```

In the app, stock type maps to:

```js
thickness -> thickness
material -> group
```

### Stock Catalog Entry

A purchasable stock option, such as:

- `3/4 plywood, 4' x 8'`
- `1x2 select pine, 8'`
- `1x4 select pine, 10'`, rippable

Stock catalog entries are optional. They are mainly for Auto Stock.

### Manual Stock

Specific available stock rows to use immediately, equivalent to the current Stock UI rows.

## Import Format Version

Every import document should include:

```json
{
  "schemaVersion": 1
}
```

The importer should reject unknown future major versions, but it may ignore unknown fields within a supported version.

## Top-Level JSON Shape

```json
{
  "schemaVersion": 1,
  "project": {
    "name": "Built-in cabinet",
    "notes": "Optional project notes"
  },
  "settings": {
    "units": "in",
    "kerf": "1/8",
    "ripKerf": "1/8",
    "crosscutKerf": "1/8",
    "allowRotate": true,
    "stockMode": "auto"
  },
  "stockTypes": [],
  "stockCatalog": [],
  "manualStock": [],
  "cuts": []
}
```

Required top-level fields:

- `schemaVersion`
- `stockTypes`
- `cuts`

Optional top-level fields:

- `project`
- `settings`
- `stockCatalog`
- `manualStock`

## Field Conventions

### Units

Supported values:

```json
"in"
"mm"
```

If omitted, default to:

```json
"in"
```

### Measurements

Measurements may be strings or numbers.

Strings should support the same formats the app already parses:

```json
"3/4"
"1 1/2"
"75"
"75\""
"4'"
"8 ft"
"1220 mm"
```

Numbers are interpreted in the document's `settings.units`.

Recommended style:

```json
"75"
```

for inches documents, and:

```json
"1905 mm"
```

for metric documents.

### IDs

IDs should be stable strings.

IDs may contain letters, numbers, hyphens, and underscores.

Example:

```json
"painted-trim-34"
```

IDs are used to connect cuts, stock types, stock catalog entries, and manual stock.

## Stock Types

`stockTypes` defines the material buckets used by cuts.

```json
{
  "id": "painted-trim-34",
  "name": "3/4 painted trim",
  "material": "painted trim",
  "thickness": "3/4",
  "kind": "linear"
}
```

Required fields:

- `id`
- `material`
- `thickness`
- `kind`

Optional fields:

- `name`
- `notes`

Allowed `kind` values:

```json
"sheet"
"linear"
```

Mapping to current app fields:

```text
material -> group
thickness -> thickness
```

## Cuts

Cuts define the required finished pieces.

### Sheet Cut

```json
{
  "name": "Side panel",
  "stockTypeId": "plywood-34",
  "qty": 2,
  "width": "15 3/4",
  "length": "34 1/2"
}
```

### Linear Cut

```json
{
  "name": "Long trim",
  "stockTypeId": "painted-trim-34",
  "qty": 4,
  "width": "1 1/2",
  "length": "75"
}
```

Required fields:

- `name`
- `stockTypeId`
- `qty`
- `width`
- `length`

Optional fields:

- `id`
- `notes`
- `grain`
- `canRotate`
- `canRip`

Allowed `grain` values:

```json
"none"
"width"
"length"
```

Defaults:

```json
{
  "qty": 1,
  "grain": "none",
  "canRotate": true,
  "canRip": true
}
```

For current app compatibility:

```text
width -> w
length -> h
stockTypes[stockTypeId].thickness -> thickness
stockTypes[stockTypeId].material -> group
```

## Manual Stock

Manual stock imports directly into the current Stock UI rows.

```json
{
  "name": "Plywood sheet",
  "stockTypeId": "plywood-34",
  "qty": 2,
  "width": "4'",
  "length": "8'"
}
```

Required fields:

- `name`
- `stockTypeId`
- `qty`
- `width`
- `length`

Optional fields:

- `id`
- `cost`
- `notes`

Mapping:

```text
width -> w
length -> h
stockTypes[stockTypeId].thickness -> thickness
stockTypes[stockTypeId].material -> group
```

If `manualStock` is present and `settings.stockMode` is omitted, the importer should default to manual stock mode.

## Stock Catalog

`stockCatalog` defines purchasable options for Auto Stock.

### Sheet Catalog Entry

```json
{
  "id": "plywood-34-4x8",
  "stockTypeId": "plywood-34",
  "kind": "sheet",
  "name": "3/4 plywood sheet",
  "width": "4'",
  "length": "8'",
  "maxQty": 10,
  "cost": 64.5
}
```

### Exact-Width Linear Catalog Entry

```json
{
  "id": "select-pine-1x2",
  "stockTypeId": "painted-trim-34",
  "kind": "linear-exact",
  "name": "1x2 select pine",
  "actualWidth": "1 1/2",
  "lengths": ["8'", "10'", "12'"],
  "costByLength": {
    "8'": 8.5,
    "10'": 11.25,
    "12'": 14.0
  }
}
```

### Rippable Linear Catalog Entry

```json
{
  "id": "select-pine-1x4",
  "stockTypeId": "painted-trim-34",
  "kind": "linear-rippable",
  "name": "1x4 select pine",
  "actualWidth": "3 1/2",
  "lengths": ["8'", "10'", "12'"],
  "ripKerf": "1/8",
  "costByLength": {
    "8'": 12.0,
    "10'": 16.0,
    "12'": 20.0
  }
}
```

Required common fields:

- `id`
- `stockTypeId`
- `kind`
- `name`

Allowed `kind` values:

```json
"sheet"
"linear-exact"
"linear-rippable"
```

For `sheet`, required fields:

- `width`
- `length`

For `linear-exact`, required fields:

- `actualWidth`
- `lengths`

For `linear-rippable`, required fields:

- `actualWidth`
- `lengths`

Optional fields:

- `maxQty`
- `cost`
- `costByLength`
- `ripKerf`
- `notes`

Default `maxQty`:

```json
10
```

## Complete Example

```json
{
  "schemaVersion": 1,
  "project": {
    "name": "Cabinet and trim project"
  },
  "settings": {
    "units": "in",
    "kerf": "1/8",
    "ripKerf": "1/8",
    "crosscutKerf": "1/8",
    "allowRotate": true,
    "stockMode": "auto"
  },
  "stockTypes": [
    {
      "id": "plywood-34",
      "name": "3/4 plywood",
      "material": "plywood",
      "thickness": "3/4",
      "kind": "sheet"
    },
    {
      "id": "painted-trim-34",
      "name": "3/4 painted trim",
      "material": "painted trim",
      "thickness": "3/4",
      "kind": "linear"
    }
  ],
  "stockCatalog": [
    {
      "id": "plywood-34-4x8",
      "stockTypeId": "plywood-34",
      "kind": "sheet",
      "name": "3/4 plywood sheet",
      "width": "4'",
      "length": "8'",
      "maxQty": 10,
      "cost": 64.5
    },
    {
      "id": "select-pine-1x2",
      "stockTypeId": "painted-trim-34",
      "kind": "linear-exact",
      "name": "1x2 select pine",
      "actualWidth": "1 1/2",
      "lengths": ["8'", "10'", "12'"],
      "costByLength": {
        "8'": 8.5,
        "10'": 11.25,
        "12'": 14.0
      }
    },
    {
      "id": "select-pine-1x4",
      "stockTypeId": "painted-trim-34",
      "kind": "linear-rippable",
      "name": "1x4 select pine",
      "actualWidth": "3 1/2",
      "lengths": ["8'", "10'", "12'"],
      "ripKerf": "1/8",
      "costByLength": {
        "8'": 12.0,
        "10'": 16.0,
        "12'": 20.0
      }
    }
  ],
  "cuts": [
    {
      "name": "Side panel",
      "stockTypeId": "plywood-34",
      "qty": 2,
      "width": "15 3/4",
      "length": "34 1/2"
    },
    {
      "name": "Top",
      "stockTypeId": "plywood-34",
      "qty": 1,
      "width": "30",
      "length": "18"
    },
    {
      "name": "Long trim",
      "stockTypeId": "painted-trim-34",
      "qty": 4,
      "width": "1 1/2",
      "length": "75"
    },
    {
      "name": "Short trim",
      "stockTypeId": "painted-trim-34",
      "qty": 2,
      "width": "1 1/2",
      "length": "17 1/8"
    }
  ]
}
```

## JSON Schema

The app should expose a JSON Schema document for validation and clipboard copy.

Use JSON Schema Draft 2020-12.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://cut-planner.local/schema/project-import.v1.json",
  "title": "Cut Planner Project Import",
  "type": "object",
  "required": ["schemaVersion", "stockTypes", "cuts"],
  "properties": {
    "schemaVersion": {
      "const": 1
    },
    "project": {
      "type": "object",
      "properties": {
        "name": { "type": "string" },
        "notes": { "type": "string" }
      },
      "additionalProperties": true
    },
    "settings": {
      "type": "object",
      "properties": {
        "units": { "enum": ["in", "mm"] },
        "kerf": { "$ref": "#/$defs/measure" },
        "ripKerf": { "$ref": "#/$defs/measure" },
        "crosscutKerf": { "$ref": "#/$defs/measure" },
        "allowRotate": { "type": "boolean" },
        "stockMode": { "enum": ["manual", "auto"] }
      },
      "additionalProperties": true
    },
    "stockTypes": {
      "type": "array",
      "minItems": 1,
      "items": { "$ref": "#/$defs/stockType" }
    },
    "stockCatalog": {
      "type": "array",
      "items": { "$ref": "#/$defs/stockCatalogEntry" }
    },
    "manualStock": {
      "type": "array",
      "items": { "$ref": "#/$defs/manualStockEntry" }
    },
    "cuts": {
      "type": "array",
      "minItems": 1,
      "items": { "$ref": "#/$defs/cut" }
    }
  },
  "additionalProperties": true,
  "$defs": {
    "measure": {
      "oneOf": [
        { "type": "string", "minLength": 1 },
        { "type": "number", "exclusiveMinimum": 0 }
      ]
    },
    "id": {
      "type": "string",
      "pattern": "^[A-Za-z0-9_-]+$"
    },
    "stockType": {
      "type": "object",
      "required": ["id", "material", "thickness", "kind"],
      "properties": {
        "id": { "$ref": "#/$defs/id" },
        "name": { "type": "string" },
        "material": { "type": "string", "minLength": 1 },
        "thickness": { "$ref": "#/$defs/measure" },
        "kind": { "enum": ["sheet", "linear"] },
        "notes": { "type": "string" }
      },
      "additionalProperties": true
    },
    "cut": {
      "type": "object",
      "required": ["name", "stockTypeId", "qty", "width", "length"],
      "properties": {
        "id": { "$ref": "#/$defs/id" },
        "name": { "type": "string", "minLength": 1 },
        "stockTypeId": { "$ref": "#/$defs/id" },
        "qty": { "type": "integer", "minimum": 1 },
        "width": { "$ref": "#/$defs/measure" },
        "length": { "$ref": "#/$defs/measure" },
        "grain": { "enum": ["none", "width", "length"] },
        "canRotate": { "type": "boolean" },
        "canRip": { "type": "boolean" },
        "notes": { "type": "string" }
      },
      "additionalProperties": true
    },
    "manualStockEntry": {
      "type": "object",
      "required": ["name", "stockTypeId", "qty", "width", "length"],
      "properties": {
        "id": { "$ref": "#/$defs/id" },
        "name": { "type": "string", "minLength": 1 },
        "stockTypeId": { "$ref": "#/$defs/id" },
        "qty": { "type": "integer", "minimum": 1 },
        "width": { "$ref": "#/$defs/measure" },
        "length": { "$ref": "#/$defs/measure" },
        "cost": { "type": ["number", "null"], "minimum": 0 },
        "notes": { "type": "string" }
      },
      "additionalProperties": true
    },
    "stockCatalogEntry": {
      "type": "object",
      "required": ["id", "stockTypeId", "kind", "name"],
      "properties": {
        "id": { "$ref": "#/$defs/id" },
        "stockTypeId": { "$ref": "#/$defs/id" },
        "kind": { "enum": ["sheet", "linear-exact", "linear-rippable"] },
        "name": { "type": "string", "minLength": 1 },
        "width": { "$ref": "#/$defs/measure" },
        "length": { "$ref": "#/$defs/measure" },
        "actualWidth": { "$ref": "#/$defs/measure" },
        "lengths": {
          "type": "array",
          "minItems": 1,
          "items": { "$ref": "#/$defs/measure" }
        },
        "maxQty": { "type": "integer", "minimum": 1 },
        "cost": { "type": ["number", "null"], "minimum": 0 },
        "costByLength": {
          "type": "object",
          "additionalProperties": {
            "type": ["number", "null"],
            "minimum": 0
          }
        },
        "ripKerf": { "$ref": "#/$defs/measure" },
        "notes": { "type": "string" }
      },
      "allOf": [
        {
          "if": {
            "properties": { "kind": { "const": "sheet" } },
            "required": ["kind"]
          },
          "then": {
            "required": ["width", "length"]
          }
        },
        {
          "if": {
            "properties": { "kind": { "enum": ["linear-exact", "linear-rippable"] } },
            "required": ["kind"]
          },
          "then": {
            "required": ["actualWidth", "lengths"]
          }
        }
      ],
      "additionalProperties": true
    }
  }
}
```

## Import Process

### Entry Points

Add two buttons:

```text
Import JSON
Copy JSON Schema
```

Recommended location:

- In a new `Project` section near the top of the sidebar, or
- In the existing controls section below Stock/Cuts.

### Import Methods

Version 1 should support file import:

```html
<input id="jsonImportFile" type="file" accept="application/json,.json" hidden>
```

The `Import JSON` button should trigger the hidden file input.

Future enhancement:

- Add a paste/import modal with a textarea.

### Import Steps

1. User clicks `Import JSON`.
2. App opens file picker.
3. User selects a JSON file.
4. App reads the file as text.
5. App parses JSON.
6. App validates structural requirements.
7. App validates references and measurements.
8. App shows errors if any exist.
9. App asks for confirmation if import will replace current rows.
10. App clears current rows.
11. App applies settings.
12. App creates stock rows from `manualStock` when present.
13. App creates cut rows from `cuts`.
14. App stores imported stock catalog for Auto Stock if present.
15. App runs optimizer.

### Replacement Behavior

For version 1, importing should replace current cuts and stock.

Before applying import, if the app has existing rows, show:

```text
Importing will replace current stock and cuts.
```

Buttons:

```text
Cancel
Import
```

Future enhancement:

- Add merge mode.

### Validation Layers

#### JSON Parse Validation

If JSON parsing fails:

```text
The selected file is not valid JSON.
```

Include the parse error message when available.

#### Schema-Like Validation

Validate:

- `schemaVersion` is `1`.
- `stockTypes` is a non-empty array.
- `cuts` is a non-empty array.
- IDs are unique within `stockTypes`.
- Each cut references an existing `stockTypeId`.
- Each manual stock row references an existing `stockTypeId`.
- Each catalog entry references an existing `stockTypeId`.
- Quantities are positive integers.
- Measurements parse to positive values.
- Catalog entries have fields required by their `kind`.

#### Semantic Validation

Validate:

- A `sheet` stock type should not be used with `linear-exact` catalog entries.
- A `linear` stock type should not be used with `sheet` catalog entries unless intentionally allowed later.
- Linear cuts should have a meaningful `width`.
- Sheet cuts should have both width and length.
- Duplicate catalog IDs should be rejected.

### Error Reporting

Show all validation errors at once when practical.

Example:

```text
Import failed:
- cuts[3] references unknown stockTypeId "oak-trim-34".
- stockCatalog[1] is linear-rippable but is missing actualWidth.
- cuts[5].length must be greater than zero.
```

## Mapping Imported Data Into Current UI

### Cuts

For each imported cut:

1. Look up `stockTypeId`.
2. Create a cut row:

```js
{
  name: cut.name,
  qty: cut.qty,
  w: cut.width,
  h: cut.length,
  thickness: stockType.thickness,
  group: stockType.material
}
```

### Manual Stock

For each imported manual stock row:

1. Look up `stockTypeId`.
2. Create a stock row:

```js
{
  name: stock.name,
  qty: stock.qty,
  w: stock.width,
  h: stock.length,
  thickness: stockType.thickness,
  group: stockType.material
}
```

### Settings

Map settings:

```text
settings.units -> #units
settings.kerf -> #kerf
settings.allowRotate -> #allowRotate
settings.stockMode -> future stock mode control
```

If `ripKerf` and `crosscutKerf` exist before the app has separate UI fields, store them in an internal imported project object for later Auto Stock use.

## Clipboard Schema Button

### Button Behavior

Add:

```html
<button class="btn" id="copyJsonSchema" type="button">Copy JSON Schema</button>
```

On click:

1. Serialize a schema/example string.
2. Copy it to the clipboard with `navigator.clipboard.writeText`.
3. Show success or failure feedback.

Suggested copied content:

- Prefer the complete JSON Schema.
- Include a short example in a top-level comment is not possible in JSON, so do not mix comments into the schema.
- Alternative: copy the complete example document instead and label the button `Copy JSON Template`.

Recommendation:

Provide two buttons eventually:

```text
Copy JSON Schema
Copy JSON Template
```

For the requested feature, implement `Copy JSON Schema` first.

### Clipboard Function

```js
async function copyJsonSchemaToClipboard() {
  const text = JSON.stringify(projectImportJsonSchema, null, 2);
  try {
    await navigator.clipboard.writeText(text);
    renderMessages(["JSON schema copied to clipboard."]);
  } catch (error) {
    renderMessages(["Could not copy schema to clipboard. Select and copy it manually."]);
  }
}
```

### Clipboard Fallback

Some browsers only allow clipboard access from secure contexts.

Fallback behavior:

1. Open a modal or textarea containing the schema.
2. Select the text.
3. Instruct the user through a short message:

```text
Clipboard access was blocked. The schema is selected below.
```

## Suggested Internal Constants

```js
const PROJECT_IMPORT_SCHEMA_VERSION = 1;
const projectImportJsonSchema = {};
const projectImportJsonTemplate = {};
```

Do not fetch the schema over the network. Keep it bundled in the app for offline use.

## Suggested Functions

### Import UI

```js
function setupJsonImportControls() {}
function openJsonImportPicker() {}
function readSelectedJsonFile(file) {}
```

### Parsing and Validation

```js
function parseProjectImportJson(text) {}
function validateProjectImportDocument(document) {}
function validateStockTypes(stockTypes) {}
function validateCuts(cuts, stockTypeMap) {}
function validateManualStock(manualStock, stockTypeMap) {}
function validateStockCatalog(stockCatalog, stockTypeMap) {}
function validateImportMeasurements(document, unit) {}
```

### Applying Import

```js
function applyProjectImport(document) {}
function applyImportedSettings(settings) {}
function clearCurrentProjectRows() {}
function addImportedCuts(cuts, stockTypeMap) {}
function addImportedManualStock(manualStock, stockTypeMap) {}
function storeImportedStockCatalog(stockCatalog, stockTypeMap) {}
```

### Clipboard

```js
function copyJsonSchemaToClipboard() {}
function showSchemaCopyFallback(schemaText) {}
```

## Implementation Phases

### Phase 1: JSON Template and Clipboard Schema

Goal:

Expose the import format before import is implemented.

Tasks:

- Add `projectImportJsonSchema` constant.
- Add `Copy JSON Schema` button.
- Implement clipboard copy behavior.
- Add fallback textarea/modal if clipboard fails.

Success criteria:

- User can copy the schema from the app.
- Copied schema is valid JSON.

### Phase 2: Basic Import for Cuts and Manual Stock

Goal:

Import a project into the current app fields.

Tasks:

- Add `Import JSON` button and hidden file input.
- Parse selected JSON file.
- Validate required fields.
- Validate stock type references.
- Validate measurements.
- Replace current rows after confirmation.
- Create cut rows.
- Create manual stock rows.
- Apply units, kerf, and rotation settings.
- Run optimizer.

Success criteria:

- A valid JSON project populates the app.
- Invalid files show actionable errors.
- Current manual workflow still works.

### Phase 3: Import Stock Catalog for Auto Stock

Goal:

Support imported purchasable stock options.

Tasks:

- Validate `stockCatalog`.
- Store imported stock catalog in app state.
- Use imported catalog in Auto Stock.
- Show imported catalog entries in future catalog UI.

Success criteria:

- Auto Stock can suggest from imported sheet, exact linear, and rippable linear catalog entries.

### Phase 4: Paste Import and Export

Goal:

Improve project portability.

Tasks:

- Add paste/import modal.
- Add export current project as JSON.
- Add `Copy JSON Template` button.

Success criteria:

- User can move projects in and out without manually editing files.

## Recommended First Version

Implement this first:

```text
Copy JSON Schema
Import JSON file
stockTypes
cuts
manualStock
settings.units
settings.kerf
settings.allowRotate
```

This gives immediate value and maps cleanly to the current app.

Then add `stockCatalog` integration when Auto Stock is implemented.

