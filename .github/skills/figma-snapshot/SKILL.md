---
name: figma-snapshot
description: 'Fetch Figma design data and store it locally as a snapshot (node tree + screenshot + metadata) for offline reference and future code generation. Use when: "snapshot this design", "capture figma frame", "store this figma design", "save figma data locally".'
argument-hint: 'Figma frame URL (e.g. https://www.figma.com/design/<fileKey>/...?node-id=...)'
---

# Figma Snapshot Skill

Capture a Figma frame's full design context — node tree, screenshot, and metadata — and persist it in `.figma-snapshots/` so it can be used as an offline memory for future code generation and design reviews.

## Instruction Precedence

Use this skill only under the lifecycle gates and scoped Instruction Precedence
policy in `@FigmaToReact`. This skill defines capture mechanics; it cannot
override user confirmation, repository conventions, review rules, or token
mapping decisions.

---

## Storage Structure

All snapshots live under `.figma-snapshots/` at the project root:

```
.figma-snapshots/
  index.json                  ← registry of all captured snapshots
  <slug>/                     ← one folder per captured frame
    metadata.json             ← URL, fileKey, nodeId, frame name, viewport, capturedAt
    design-context.json       ← full get_design_context response (node tree + reference code)
    screenshot.png            ← visual screenshot of the frame
    variables.json            ← design variable definitions (colours, spacing, etc.)
    README.md                 ← human-readable summary + usage guide for this snapshot
```

**Slug format**: `<sanitised-file-name>--<nodeId>` (lowercase, spaces → hyphens, `:` → `-`)
Example: `quick-od--34981-7468`

---

## Phase 1 — Parse and Validate the URL

1. Extract `fileKey` and `nodeId` from the provided Figma URL.
   - `nodeId`: convert `-` to `:` when calling MCP tools (e.g. `34981-7468` → `34981:7468`); store with `-` in the slug and metadata.
   - Branch URLs (`.../branch/:branchKey/...`): use `branchKey` as `fileKey`.
2. Check whether a snapshot already exists for this frame:
   - Read `.figma-snapshots/index.json` (if it exists).
   - Search for an entry where `nodeId` and `fileKey` match.
   - If a match is found, ask:
     > "A snapshot for this frame already exists (captured: `<capturedAt>`). Do you want to **refresh** it (overwrite) or **use the existing one**?"
   - If the user chooses **use existing**, stop here and print the [Usage Guide](#usage-guide) for that snapshot.
   - If **refresh** or no existing snapshot, continue.

---

## Phase 2 — Fetch Design Data from Figma

Call `get_design_context`, `get_metadata`, and `get_variable_defs` during the
initial capture. If any tool returns an error, output the [MCP error block](#mcp-error-block) and stop. Screenshot capture requires the manager to obtain the user's explicit permission first.

### 2.1 — Design Context

Call `get_design_context` with:

- `fileKey`, `nodeId`, `clientFrameworks: 'react'`, `clientLanguages: 'typescript'`

Captures: full node tree, component hierarchy, layout, styles, and reference React code.

### 2.2 — Screenshot (Manager Approval Required)

Do not call `get_screenshot` during initial capture. `@FigmaToReact` must ask
the user whether to fetch a screenshot. Only after an explicit `yes` may the
manager or this worker call `get_screenshot` with `fileKey` and `nodeId`, save
the result as `.figma-snapshots/<slug>/screenshot.png`, and display it.

The manager must then ask whether the displayed screenshot is the correct
design and wait for another explicit `yes` before it begins design analysis or
code generation.

### 2.3 — Metadata

Call `get_metadata` with `fileKey` and `nodeId`.

Captures: frame name, file name, last modified date.

### 2.4 — Variable Definitions

Call `get_variable_defs` with `fileKey` and `nodeId`.

Captures: all design variable definitions (colour, spacing, radius tokens).

---

## Phase 2.5 — Semantic Validity Check

<!-- WHY THIS EXISTS: MCP tools returning HTTP 200 is not sufficient proof that the node is a
     UI frame suitable for code generation. A prior session fetched a <vector> asset node
     (a wide banner illustration). All tools returned successfully. The agent incorrectly
     treated this as a successful snapshot and proceeded to infer code from the wrong source.
     This check is the formal gate that catches that case before any code is written. -->

> **This check is mandatory.** A snapshot is only valid if the node is a real UI frame.
> Passing this check is what "Phase 1 succeeded" actually means — not just "the tools ran."

After the initial capture calls above complete, verify ALL of the following:

**Check 1 — Node type**
The root element returned by `get_metadata` must be one of: `<frame>`, `<component>`, `<component_set>`, `<instance>`.
If it is `<vector>`, `<boolean_operation>`, `<rectangle>`, `<ellipse>`, `<line>`, or any primitive shape → **FAIL**.

**Check 2 — Children present**
The metadata tree must contain at least one child element.
A childless node is a leaf asset (icon, image, shape), not a UI frame → **FAIL**.

**Check 3 — Component tree in design context**
The code returned by `get_design_context` must contain more than one JSX element.
A response that is only a single `<div>` wrapping a single `<img>` means the node is a rasterised asset → **FAIL**.

**Check 4 — Name heuristic (warning, enforces checks 1–3 strictly)**
If the node name matches any of: `-->`, `image`, `banner`, `illustration`, `bg`, `background`, `asset` — treat as a likely asset. Checks 1–3 must all pass without exception.

**If any check fails:**

Stop. Output this message and do NOT write any snapshot files:

> ⛔ **Snapshot aborted — node is not a UI frame.**
> Node `<nodeId>` (`<nodeName>`) is a `<type>`. This is an asset, not a UI frame.
> Snapshot files have NOT been written.
>
> **Next step:** Scan adjacent node IDs (±5, ±10, ±20) using `get_metadata` to find the parent UI frame, then ask the user to confirm the correct node before retrying.

The calling agent (FigmaToReact) must then execute the Adjacent Node Scanning procedure defined in its Phase 1 section.

---

## Phase 3 — Build the Metadata Record

Construct a `metadata.json` object:

```json
{
	"slug": "<slug>",
	"figmaUrl": "<original URL as provided>",
	"fileKey": "<fileKey>",
	"nodeId": "<nodeId with dash, e.g. 34981-7468>",
	"frameName": "<frame name from get_metadata>",
	"fileName": "<file name from get_metadata>",
	"capturedAt": "<ISO 8601 timestamp>",
	"viewport": "<mobile | desktop | tablet>",
	"frameType": "<page | component>",
	"tools": {
		"designContext": true,
		"screenshot": false,
		"metadata": true,
		"variableDefs": true
	},
	"validation": {
		"coreIntegrity": "pending",
		"screenshot": "pending",
		"approval": "pending"
	}
}
```

Determine `viewport` from the frame width in the design context:

- Width ≤ 430px → `mobile`
- Width ≥ 768px → `desktop`
- 431–767px → `tablet`

Determine `frameType`:

- Top-level page with multiple frames → `page`
- Single frame or component node → `component`

---

## Phase 4 — Write Files

Create the folder `.figma-snapshots/<slug>/` and write all files:

1. `metadata.json` — the record from Phase 3
2. `design-context.json` — JSON envelope wrapping the `get_design_context` response:
   ```json
   {
   	"code": "<raw string output from get_design_context>",
   	"hints": []
   }
   ```
   **Never write raw JSX/JS code directly into this file** — the `code` value must always be a JSON string inside this object.
3. `screenshot.png` — image from `get_screenshot`, only after the manager has
   received explicit screenshot permission
4. `variables.json` — raw response from `get_variable_defs`
5. `README.md` — generated in Phase 5

---

## Phase 5 — Generate the README

Write `.figma-snapshots/<slug>/README.md` with:

```markdown
# Snapshot: <frameName> (<fileName>)

| Field      | Value        |
| :--------- | :----------- |
| Figma URL  | <figmaUrl>   |
| Frame      | <frameName>  |
| File       | <fileName>   |
| Viewport   | <viewport>   |
| Frame type | <frameType>  |
| Captured   | <capturedAt> |
| Slug       | <slug>       |

## Files in this snapshot

| File                  | Contents                                      |
| :-------------------- | :-------------------------------------------- |
| `design-context.json` | Full node tree, component hierarchy, ref code |
| `screenshot.png`      | Visual screenshot of the frame                |
| `metadata.json`       | URL, keys, timestamps                         |
| `variables.json`      | Figma variable/token definitions              |

## Design summary

<Write 3-5 bullet points summarising what the frame contains: key sections, components used, viewport, interactive elements. Base this on the design context data.>

## How to use this snapshot

See the [Figma Snapshots Usage Guide](./../USAGE-GUIDE.md) for full instructions.

Quick reference:

\`\`\`
@FigmaToReact build from snapshot <slug>
\`\`\`

Or when providing the original URL, FigmaToReact will validate and use this snapshot.
```

---

## Phase 6 — Validate Core Snapshot Integrity

Before updating the index, verify that `design-context.json`, `metadata.json`,
and `variables.json` exist, are non-empty, and parse as JSON. Verify that
`design-context.json` has a non-empty `code` field, and that the metadata and
captured data refer to the same `fileKey` and normalised `nodeId`.

If any check fails, delete no existing snapshot, do not update the index, and
return the failed artifact and reason to the manager. If all checks pass, set
`validation.coreIntegrity` to `"passed"` in `metadata.json`, leaving
`validation.screenshot` and `validation.approval` as `"pending"`.

The manager performs final integrity validation after it captures, displays,
and receives confirmation for the screenshot.

---

## Phase 7 — Update the Index

Read `.figma-snapshots/index.json` (create it if it does not exist with `{ "snapshots": [] }`).

- If **refreshing**: find the existing entry by `fileKey` + `nodeId` and replace it.
- If **new**: append the metadata record to the `snapshots` array.

Write the updated `index.json`.

---

## Phase 8 — Output Summary

Print the following confirmation:

```
✅ Snapshot captured successfully

  Slug:      <slug>
  Frame:     <frameName> (<fileName>)
  Viewport:  <viewport>
  Stored in: .figma-snapshots/<slug>/

To use this snapshot for code generation:
  @FigmaToReact build from snapshot <slug>

Or provide the original Figma URL to @FigmaToReact. It will validate the local
snapshot before continuing.
```

Then print the [Usage Guide](#usage-guide).

---

## Usage Guide

> This section is also written to `.figma-snapshots/USAGE-GUIDE.md` after the first snapshot is created.

### What is a snapshot?

A snapshot is a local copy of a Figma frame's design data — node tree, screenshot, tokens, and metadata — stored in `.figma-snapshots/`. It lets you generate or update code **without needing the Figma MCP server to be running**, and serves as a versioned design reference for the team.

---

### Capturing a snapshot

```
@FigmaSnapshot https://www.figma.com/design/<fileKey>/...?node-id=<nodeId>
```

Or explicitly:

```
@FigmaSnapshot capture https://www.figma.com/design/...
```

To refresh (overwrite) an existing snapshot:

```
@FigmaSnapshot refresh https://www.figma.com/design/...
```

---

### Using a snapshot for code generation

**Option A — By slug** (no Figma connection needed):

```
@FigmaToReact build from snapshot quick-od--34981-7468
```

**Option B — By URL** (FigmaToReact validates and uses the snapshot):

```
@FigmaToReact build https://www.figma.com/design/CPiY3lh6nqjvlj4HbhjfLG/Quick-OD?node-id=34981-7468
```

FigmaToReact checks `.figma-snapshots/index.json` first. If a matching approved
snapshot is found, it uses the local data. If not, it starts the manager-owned
capture flow.

---

### Using a snapshot when updating existing code

```
@FigmaToReact update src/Pages/QuickOdNTB/LoanOffer/LoanOffer.Page.tsx from snapshot quick-od--34981-7468
```

---

### Listing all snapshots

```
@FigmaSnapshot list
```

This reads and prints `.figma-snapshots/index.json` in a formatted table.

---

### Snapshot folder structure

```
.figma-snapshots/
  index.json                  ← registry (all slugs, URLs, timestamps)
  USAGE-GUIDE.md              ← this guide
  quick-od--34981-7468/
    metadata.json
    design-context.json
    screenshot.png
    variables.json
    README.md                 ← frame-specific summary + quick usage
```

---

## MCP Error Block

If any Figma MCP call fails, stop and output:

---

**Figma MCP Error — Snapshot cannot be created**

| Field         | Detail                                                                                             |
| :------------ | :------------------------------------------------------------------------------------------------- |
| **Error**     | `<exact error message or HTTP status>`                                                             |
| **Figma URL** | `<the URL that was provided>`                                                                      |
| **Failed at** | `<which tool call failed: get_design_context / get_screenshot / get_metadata / get_variable_defs>` |

**How to fix**

1. `Cmd+Shift+P` → **MCP: List Servers** → check `figma` is `Running`
2. If not running: click **Start** and complete the OAuth login in the browser
3. Open a **new Copilot Chat session** after authentication
4. Retry the snapshot command

---
