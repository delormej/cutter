# Auto Stock Execution Plan and Specification

## Purpose

Add an Auto Stock mode to Cut Planner that can suggest what lumber or sheet goods to buy from a list of required cuts.

The app should support three material planning cases:

1. Sheet goods, such as plywood, MDF, and panels.
2. Exact-width linear lumber, such as buying 1.5 inch trim boards and crosscutting them.
3. Ripped linear lumber, such as buying wider boards, ripping them into narrower strips, then crosscutting those strips.

The highest priority use case is linear trim planning: deciding whether to buy exact-width boards or wider boards that can be ripped down, then producing a buy list and cut plan.

## Current App Context

The existing app is a single-page HTML application in `index.html`.

It already has:

- Manual stock rows.
- Cut rows.
- Width, length, quantity, thickness, and group fields.
- Kerf input.
- Part rotation option.
- A 2D guillotine-style sheet optimizer.
- Compatibility matching based on `thickness` and `group`.

The Auto Stock feature should build on this instead of replacing it.

## User-Facing Goals

The user should be able to:

- Enter all cuts for a project.
- Assign cuts to a material group and thickness.
- Choose Manual Stock or Auto Stock.
- Ask the app to suggest stock to buy.
- Compare exact-width linear stock against wider rippable boards.
- See a ranked set of buying options.
- See a per-board cut plan.
- Convert suggested stock into editable manual stock rows if desired.

Example output:

```text
Buy List

3/4 plywood
2 x 4' x 8' sheets

3/4 painted trim, 1.5" wide
2 x 1x4 x 10'
Rip each board into 2 strips at 1.5"
```

## Terminology

### Cut

A required finished part.

For sheet goods, width and length describe a rectangular part.

For linear lumber, width describes the finished strip width and length describes the crosscut length.

### Stock

Material that can be purchased.

Stock may be:

- A sheet.
- An exact-width linear board.
- A wider board that can be ripped into narrower strips.

### Group

A user-defined material category, such as:

- `plywood`
- `painted trim`
- `oak`
- `maple`

### Thickness

The material thickness. This should be matched within each stock recommendation bucket.

### Linear Width

For trim and boards, this is the required finished width, such as `1.5"`.

### Rip Kerf

Material lost between strips when ripping a wider board.

### Crosscut Kerf

Material lost between length cuts.

Initially, this may reuse the app's existing kerf input. Long term, the UI should expose rip kerf and crosscut kerf separately.

## High-Level Architecture

Auto Stock should be implemented as a layer above the existing optimizers.

```text
cuts
  -> group by material need
  -> choose candidate stock
  -> run appropriate optimizer
  -> rank scenarios
  -> render buy list and cut plan
```

There should be separate optimizers for:

- 2D sheet goods.
- 1D exact-width linear lumber.
- 1D ripped linear lumber.

Do not force linear lumber through the sheet optimizer. It has different constraints, waste math, and output needs.

## Data Model

### Stock Mode

Add a stock mode setting:

```js
const stockMode = "manual"; // "manual" | "auto"
```

Manual mode should preserve the current behavior.

Auto mode should derive candidate stock rows from a stock catalog and the entered cuts.

### Cut Row

The current cut row fields can remain:

```js
{
  id,
  name,
  qty,
  w,
  h,
  thickness,
  group
}
```

For internal planning, normalize this into:

```js
{
  id,
  name,
  qty,
  width,
  length,
  thickness,
  group,
  kind
}
```

Where `kind` is:

```js
"sheet-part" | "linear-part"
```

Initial inference rules:

- If the group or stock catalog maps the group to sheet goods, use `sheet-part`.
- If the group maps to trim or linear lumber, use `linear-part`.
- Later, expose `kind` directly in the UI.

### Stock Catalog

Add a built-in stock catalog.

Example:

```js
const stockCatalog = [
  {
    id: "plywood-34-4x8",
    kind: "sheet",
    name: "3/4 plywood sheet",
    group: "plywood",
    thickness: "3/4",
    width: "4'",
    length: "8'",
    maxQty: 10,
    cost: null
  },
  {
    id: "select-pine-1x2",
    kind: "linear-exact",
    name: "1x2 select pine",
    group: "painted trim",
    thickness: "3/4",
    actualWidth: "1.5",
    lengths: ["8'", "10'", "12'"],
    costByLength: {
      "8'": null,
      "10'": null,
      "12'": null
    }
  },
  {
    id: "select-pine-1x4",
    kind: "linear-rippable",
    name: "1x4 select pine",
    group: "painted trim",
    thickness: "3/4",
    actualWidth: "3.5",
    lengths: ["8'", "10'", "12'"],
    ripKerf: "1/8",
    costByLength: {
      "8'": null,
      "10'": null,
      "12'": null
    }
  }
];
```

### Normalized Stock Candidate

Each stock catalog entry should be parsed into inches internally:

```js
{
  id,
  kind,
  name,
  group,
  thickness,
  width,
  length,
  actualWidth,
  cost,
  maxQty
}
```

## Grouping Rules

First group cuts by:

```text
kind + thickness + group
```

Then for linear cuts, sub-group by required finished width:

```text
kind + thickness + group + width
```

Example:

```text
linear-part + 3/4 + painted trim
  1.5" wide
  2.25" wide
```

For version 1, solve each linear width group independently.

Later, add mixed-width ripping, where a single wider board can produce multiple output strip widths.

## Auto Stock UI Specification

### Sidebar Controls

In the Stock section, add:

```text
Stock Mode
[ Manual ] [ Auto ]
```

In Auto mode, hide or collapse manual stock rows by default, but keep them available through an advanced/manual override affordance later.

Add Auto Stock controls:

```text
Auto Stock Strategy
[ Practical ] [ Cheapest ] [ Least Waste ] [ Fewest Boards ] [ Fewest Rips ]

Linear Stock
[x] Allow exact-width boards
[x] Allow ripping wider boards

Stock lengths
[x] 8'
[x] 10'
[x] 12'
[ ] 16'

Kerf
Rip kerf: 1/8"
Crosscut kerf: 1/8"
```

For the first implementation, it is acceptable to use the existing global kerf for both rip and crosscut kerf.

### Results View

Add a buy list above layout results:

```text
Buy List

3/4 plywood
2 x 4' x 8' sheets

3/4 painted trim, 1.5" wide
Recommended:
2 x 1x4 x 10'
Rip each into 2 strips at 1.5"
```

Show ranked alternatives:

```text
Alternatives

4 x 1x2 x 8'
No ripping
Waste: 39"

1 x 1x8 x 12'
Rip into 4 strips at 1.5"
Waste: 80"
```

### Linear Board Visualization

For exact-width linear boards:

```text
1x2 x 8' Board 1
[ 75" ][ 17 1/8" ][ waste ]
```

For ripped boards:

```text
1x4 x 10' Board 1
Rip: 2 strips @ 1.5"

Strip 1
[ 75" ][ 38 7/8" ][ waste ]

Strip 2
[ 69 3/4" ][ 28 3/8" ][ 17 1/8" ][ waste ]
```

## Sheet Goods Auto Stock

### Scope

Auto stock for sheets should use the existing 2D optimizer.

### Candidate Generation

For each sheet goods group:

1. Find matching sheet stock catalog entries.
2. For each candidate sheet size, generate quantities from `1` through `maxQty`.
3. Run the existing sheet optimizer.
4. Keep the best plans that place every cut.

### Sheet Scenario Shape

```js
{
  kind: "sheet-scenario",
  materialKey,
  stockCandidate,
  qty,
  plan,
  placedCount,
  unplacedCount,
  usedArea,
  wasteArea,
  yieldRate,
  cost,
  score
}
```

### Sheet Scoring

Initial score without prices:

```js
score =
  unplacedCount * 100000000 +
  qty * 10000 +
  wasteArea;
```

With prices:

```js
score =
  unplacedCount * 100000000 +
  totalCost * 100000 +
  wasteArea +
  qty * 1000;
```

## Exact-Width Linear Optimizer

### Scope

This handles buying boards at the exact finished width and crosscutting them to length.

Example:

```text
Need:
4 @ 75"
2 @ 17 1/8"
3 @ 28 3/8"
3 @ 38 7/8"
1 @ 69 3/4"

Stock:
1.5" x 8'
1.5" x 10'
1.5" x 12'
```

### Algorithm

Use a 1D bin-packing heuristic.

1. Expand cuts by quantity.
2. Sort pieces longest first.
3. Generate stock boards from a candidate board length and quantity.
4. Place each piece into the board with the smallest remaining length that fits.
5. Account for crosscut kerf between pieces.
6. Score each candidate quantity and length.
7. Return the best plans.

### Crosscut Kerf Rule

When adding a piece to an empty board, required length is:

```js
piece.length
```

When adding a piece after one or more existing pieces, required length is:

```js
crosscutKerf + piece.length
```

### Exact Linear Scenario Shape

```js
{
  kind: "linear-exact-scenario",
  materialKey,
  requiredWidth,
  stockCandidate,
  stockLength,
  stockQty,
  boards: [
    {
      id,
      name,
      width,
      length,
      cuts: [],
      usedLength,
      wasteLength
    }
  ],
  unplaced: [],
  totalPurchasedLength,
  totalCutLength,
  totalKerfLoss,
  totalWasteLength,
  ripCount: 0,
  cost,
  score
}
```

## Ripped Linear Optimizer

### Scope

This handles buying wider boards, ripping them into strips of the required finished width, then crosscutting those strips.

Example:

```text
Need:
1.5" wide painted trim

Candidate stock:
1x4 actual width 3.5"

Rip kerf:
1/8"
```

Calculate strip count:

```js
stripCount = Math.floor((actualWidth + ripKerf) / (requiredWidth + ripKerf));
```

For a 3.5 inch board ripped into 1.5 inch strips:

```text
1.5 + 1/8 + 1.5 = 3.125"
```

Result:

```text
2 strips
0.375" leftover width
```

### Algorithm

For each rippable stock candidate:

1. Check `actualWidth >= requiredWidth`.
2. Calculate `stripCount`.
3. Ignore candidates where `stripCount < 1`.
4. For each allowed stock length, generate physical board quantities from `1` through `maxQty`.
5. Convert physical boards into virtual strips.
6. Run the same 1D length packer against the virtual strips.
7. Group packed strips back under their source boards.
8. Calculate rip waste, length waste, board count, rip count, and cost.
9. Rank the scenario.

### Virtual Strip Shape

```js
{
  id,
  sourceBoardId,
  stripIndex,
  width: requiredWidth,
  length: stockLength,
  cuts: [],
  usedLength: 0,
  wasteLength: stockLength
}
```

### Ripped Scenario Shape

```js
{
  kind: "linear-ripped-scenario",
  materialKey,
  requiredWidth,
  stockCandidate,
  stockLength,
  stockQty,
  stripsPerBoard,
  leftoverWidth,
  boards: [
    {
      id,
      name,
      actualWidth,
      length,
      strips: []
    }
  ],
  unplaced: [],
  totalPurchasedLength,
  totalStripLength,
  totalCutLength,
  totalCrosscutKerfLoss,
  totalRipKerfLoss,
  totalLengthWaste,
  totalWidthWaste,
  ripCount,
  cost,
  score
}
```

### Rip Count

For a board producing `N` strips, rip count is usually:

```js
Math.max(0, N - 1)
```

If the plan also separates a leftover strip or trims an edge, this may differ. For v1, use `N - 1`.

## Mixed-Width Ripping

Do not implement this in the first version.

The architecture should allow it later.

Mixed-width ripping means one wider board can produce multiple finished widths:

```text
1x6 actual width 5.5"

Possible rip pattern:
2.25" + 1/8" + 1.5" + 1/8" + 1.5" = 5.5"
```

Future implementation would need:

- Rip pattern generation.
- Width bin-packing.
- Length bin-packing for each generated strip.
- Scenario scoring across multiple required widths.

For v1, use one required output width per rippable stock scenario.

## Scenario Generation

For each material group, produce candidate scenarios.

### Sheet Goods

```text
candidate sheet size x quantity
```

Example:

```text
4' x 8' plywood, qty 1
4' x 8' plywood, qty 2
4' x 8' plywood, qty 3
```

### Exact Linear

```text
candidate exact-width board x length x quantity
```

Example:

```text
1x2 x 8', qty 1
1x2 x 8', qty 2
1x2 x 10', qty 1
1x2 x 10', qty 2
```

### Ripped Linear

```text
candidate wider board x length x quantity x required output width
```

Example:

```text
1x4 x 8', ripped into 1.5" strips, qty 1
1x4 x 8', ripped into 1.5" strips, qty 2
1x6 x 8', ripped into 1.5" strips, qty 1
```

## Linear Scenario Scoring

The app should support different optimization priorities.

### Practical Default

Without costs:

```js
score =
  unplacedCount * 100000000 +
  stockQty * 10000 +
  totalLengthWaste * 10 +
  ripCount * 50 +
  totalWidthWaste;
```

With costs:

```js
score =
  unplacedCount * 100000000 +
  cost * 100000 +
  stockQty * 1000 +
  totalLengthWaste * 10 +
  ripCount * 50 +
  totalWidthWaste;
```

### Cheapest

```js
score =
  unplacedCount * 100000000 +
  cost * 100000 +
  totalLengthWaste +
  ripCount * 10;
```

If costs are missing, fall back to Practical.

### Least Waste

```js
score =
  unplacedCount * 100000000 +
  totalLengthWaste * 100 +
  totalWidthWaste * 10 +
  stockQty * 100 +
  ripCount * 10;
```

### Fewest Boards

```js
score =
  unplacedCount * 100000000 +
  stockQty * 100000 +
  totalLengthWaste * 10 +
  ripCount * 50;
```

### Fewest Rips

```js
score =
  unplacedCount * 100000000 +
  ripCount * 100000 +
  stockQty * 1000 +
  totalLengthWaste * 10;
```

## Public Function Plan

Add these functions incrementally.

### Parsing and Grouping

```js
function normalizeCuts(cutRows, unit) {}
function normalizeStockCatalog(stockCatalog, unit) {}
function inferCutKind(cut, catalog) {}
function groupCutsForAutoStock(cuts) {}
```

### Sheet Auto Stock

```js
function suggestSheetStock(cuts, catalogCandidates, options) {}
function buildSheetStockRows(candidate, qty) {}
```

### Linear Exact Stock

```js
function suggestLinearExactStock(cuts, catalogCandidates, options) {}
function packLinearCutsIntoBoards(cuts, boards, crosscutKerf) {}
```

### Linear Ripped Stock

```js
function suggestLinearRippedStock(cuts, catalogCandidates, options) {}
function getStripCount(actualWidth, requiredWidth, ripKerf) {}
function buildVirtualStrips(candidate, stockLength, stockQty, requiredWidth, ripKerf) {}
function groupStripsBySourceBoard(strips) {}
```

### Scenario Ranking

```js
function scoreAutoStockScenario(scenario, options) {}
function rankScenarios(scenarios, options) {}
function chooseRecommendedScenario(scenarios, options) {}
```

### Rendering

```js
function renderBuyList(autoStockPlan) {}
function renderLinearPlan(scenario) {}
function renderLinearBoard(board) {}
function renderLinearStrip(strip) {}
```

## Auto Stock Plan Shape

The top-level result should look like:

```js
{
  mode: "auto",
  groups: [
    {
      materialKey,
      kind,
      thickness,
      group,
      requiredWidth,
      recommended,
      alternatives
    }
  ],
  buyList: [],
  warnings: []
}
```

Buy list item:

```js
{
  materialKey,
  name,
  qty,
  width,
  length,
  thickness,
  group,
  note
}
```

## Validation Rules

Auto Stock should report clear warnings when:

- A cut has no thickness.
- A cut has no group.
- A cut has invalid width or length.
- No stock catalog entry matches a material group.
- A required linear width is wider than all matching stock candidates.
- A cut length exceeds the longest available board length.
- A sheet part does not fit any sheet candidate.
- Ripping is disabled and no exact-width board exists.

Warnings should not be cryptic. Example:

```text
No linear stock matches 3/4 painted trim at 2.25" wide. Add a stock option or allow ripping from wider boards.
```

## Implementation Phases

### Phase 1: Internal Linear Exact Optimizer

Goal:

Support exact-width board recommendations for linear cuts.

Tasks:

- Add internal cut normalization.
- Add a basic stock catalog.
- Add grouping by `kind + thickness + group + width`.
- Implement `packLinearCutsIntoBoards`.
- Generate exact-width board scenarios.
- Rank scenarios.
- Render a simple buy list and text cut plan.

Success criteria:

- Given trim cuts and 1.5 inch stock lengths, the app suggests board quantity and length.
- The app shows which cuts go on each board.
- Crosscut kerf is included.

### Phase 2: Ripped Linear Optimizer

Goal:

Support buying wider boards and ripping them into finished-width strips.

Tasks:

- Add rippable stock catalog entries.
- Implement strip count calculation.
- Convert physical boards into virtual strips.
- Reuse the 1D length packer on strips.
- Group strips back under source boards.
- Add rip waste and rip count calculations.
- Render rip plan and crosscut plan.

Success criteria:

- The app can recommend buying 1x4 boards and ripping them into 1.5 inch strips.
- The output shows strips per board.
- The output compares exact-width and ripped options.

### Phase 3: Auto Stock UI

Goal:

Expose Auto Stock in the app UI.

Tasks:

- Add Manual/Auto stock mode control.
- Add auto-stock strategy control.
- Add exact-width/ripping toggles.
- Add allowed stock length controls.
- Add buy list rendering.
- Add linear board visualization.

Success criteria:

- User can switch to Auto Stock.
- User can run optimization without entering manual stock.
- Results include buy list, recommended plan, and alternatives.

### Phase 4: Sheet Auto Stock

Goal:

Use the existing 2D optimizer to suggest sheet goods.

Tasks:

- Add sheet stock catalog entries.
- Generate quantity scenarios for matching sheets.
- Run existing optimizer against each scenario.
- Rank sheet scenarios.
- Add sheet goods to buy list.

Success criteria:

- User can enter plywood cuts without manually entering stock.
- The app suggests number of sheets.
- Existing sheet layouts still render.

### Phase 5: User-Editable Catalog

Goal:

Let users configure available stock.

Tasks:

- Add catalog UI.
- Allow editing stock names, widths, lengths, and costs.
- Persist catalog in local storage.
- Add reset-to-default catalog option.

Success criteria:

- User can model local store availability.
- Costs influence recommendations.

### Phase 6: Mixed-Width Ripping

Goal:

Allow a single wider board to produce multiple finished widths.

Tasks:

- Generate width rip patterns.
- Pack multiple width groups into shared stock boards.
- Re-run length packing per generated strip.
- Add advanced scoring for mixed plans.

Success criteria:

- The app can recommend ripping a 1x6 into mixed 1.5 inch and 2.25 inch strips.
- The output remains understandable.

## Recommended First Milestone

Build this first:

```text
Auto Stock for Linear Lumber v1

Inputs:
- linear trim cuts
- group
- thickness
- finished width
- allowed exact board sizes
- allowed rippable board sizes
- stock lengths
- kerf

Outputs:
- best exact-width option
- best ripped option
- recommended buy list
- per-board rip and crosscut plan
```

This directly answers the most important project question:

```text
Should I buy exact-width trim boards, or buy wider boards and rip them?
How many boards?
What lengths?
What waste?
What is the cut plan?
```

## Notes and Tradeoffs

- Use inches internally, matching the existing app.
- Keep Manual Stock working exactly as it does now.
- Implement exact-width linear optimization before ripped optimization.
- Keep mixed-width ripping out of v1.
- Treat costs as optional at first.
- Return alternatives, not only one answer, because lumber buying is often a tradeoff between waste, cost, effort, and availability.

