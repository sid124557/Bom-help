# Automotive Headlamp BOM Analysis + "BOM Architect" UI Blueprint

## 1) Input Dataset (Parsed)

| Level | Material  | Description                      | Type | Qty   |
|------:|-----------|----------------------------------|------|-------|
| 0     | LAMP-001  | Finished Headlamp Assembly       | FERT | 1     |
| 1     | HSG-100   | Plastic Housing Sub-Assy         | HALB | 1     |
| 2     | SCRW-99   | Mounting Screw M5                | ROH  | 4     |
| 2     | SEAL-02   | Rubber Gasket                    | ROH  | 1     |
| 1     | OPTIC-200 | Reflector Module                 | HALB | 1     |
| 2     | MIRROR-X  | Chrome Inner Lens                | HALB | 1     |
| 3     | BMC-RAW   | Bulk Molding Compound            | ROH  | 0.5kg |
| 3     | COAT-UV   | UV Protective Coating            | ROH  | 10g   |
| 1     | BULB-H7   | Halogen Bulb 24V                 | ROH  | 1     |

## 2) BOM Hierarchy (Tree)

```text
L0  LAMP-001  Finished Headlamp Assembly (FERT)  Qty: 1
├── L1  HSG-100    Plastic Housing Sub-Assy (HALB)  Qty: 1
│   ├── L2  SCRW-99   Mounting Screw M5 (ROH)        Qty: 4
│   └── L2  SEAL-02   Rubber Gasket (ROH)            Qty: 1
├── L1  OPTIC-200  Reflector Module (HALB)           Qty: 1
│   └── L2  MIRROR-X  Chrome Inner Lens (HALB)       Qty: 1
│       ├── L3  BMC-RAW  Bulk Molding Compound (ROH) Qty: 0.5 kg
│       └── L3  COAT-UV  UV Protective Coating (ROH) Qty: 10 g
└── L1  BULB-H7    Halogen Bulb 24V (ROH)            Qty: 1
```

## 3) Critical Nodes (HALB Sub-assemblies)

1. **HSG-100 (L1 HALB):** Structural enclosure branch; controls mechanical closure (screw/gasket path).
2. **OPTIC-200 (L1 HALB):** Optical module parent; key for photometric behavior.
3. **MIRROR-X (L2 HALB):** Deep optical transformation node; parent of the deepest raw-material branch.

These are "critical" because a HALB node is a dependency container: spec, process, and cost changes propagate both upward (assembly roll-up) and downward (child parts).

## 4) Roll-up Analysis (Deepest ROH Dependencies)

Deepest raw materials (Level 3+) in this dataset:

- **BMC-RAW (L3, ROH, 0.5 kg)** under `MIRROR-X → OPTIC-200 → LAMP-001`
- **COAT-UV (L3, ROH, 10 g)** under `MIRROR-X → OPTIC-200 → LAMP-001`

Interpretation:
- These items are the most nested and process-sensitive materials.
- Any change to MIRROR-X routing, yield, composition, or coating method will likely affect both of these ROH lines.

## 5) Change Impact: MIRROR-X (L2) Engineering Change

### Upstream impact
- **L1 affected:** `OPTIC-200` (direct parent)
- **L0 affected:** `LAMP-001` (finished good)

### Downstream impact
- **L3 children affected:** `BMC-RAW`, `COAT-UV`

### Typical business effects
- Revalidation of optical module behavior (`OPTIC-200`).
- Potential PPAP/change-control path at finished lamp level (`LAMP-001`).
- Cost re-roll and potential supplier/spec updates for molding compound and UV coating.

## 6) Human-readable Summary

This lamp is assembled from three branches:

1. **Housing branch:** Plastic housing with screws and a gasket.
2. **Optical branch:** Reflector module containing a mirror/lens sub-assembly made from molded compound plus UV coating.
3. **Light source branch:** One halogen bulb.

In plain language: *a shell + an optical core + a bulb* build the complete headlamp.

---

# 7) "BOM Architect" UI Design (Implementation-ready)

## A. Input & Process Layer (Control Center)

### 1) Raw Data Paste Bin
- One large textarea accepting:
  - CSV
  - Tab-delimited
  - Spreadsheet paste from Excel/Google Sheets
- UX hint examples shown inline for expected headers: `Level, Material, Description, Type, Qty`.

### 2) Auto-Map Button
- Heuristics map incoming headers to canonical fields:
  - `Level`: `level`, `lvl`, `bom_level`
  - `Material`: `material`, `matnr`, `part`, `code`
  - `Qty`: `qty`, `quantity`, `component_qty`
- If confidence is low, show a manual mapping dropdown for each detected column.

### 3) Hierarchy Engine
Transform flat rows into nested JSON using stack logic:

```ts
// Pseudocode
const stack: Node[] = [];
for (const row of rows) {
  const node = toNode(row); // {id, level, type, qty, children: []}

  while (stack.length && stack[stack.length - 1].level >= node.level) {
    stack.pop();
  }

  if (stack.length === 0) {
    roots.push(node);
  } else {
    stack[stack.length - 1].children.push(node);
    node.parentId = stack[stack.length - 1].id;
  }

  stack.push(node);
}
```

This directly enforces the rule: higher level after lower = child; lower/equal level = sibling or ancestor branch continuation.

## B. Output Layer (Live BOM Workspace)

### Left Pane: Navigation Tree
- Expandable/collapsible hierarchy.
- Visual rules:
  - Level 0 root
  - Level 1 bold
  - Level 2+ indented
- Color badges:
  - 🔵 `HALB` (expandable node)
  - 🟢 `ROH` (leaf/purchase part)
- Drag/drop behavior:
  - Allow repositioning a node (e.g., move L3 from one L2 to another).
  - On drop, recalculate `parentId`, display new path, and trigger impact refresh.

### Right Pane: Detail & Edit Panel
When a node is selected:
- **Price Editor:** modify moving price.
- **Roll-up Calculator:** instantly recompute rolled cost to all ancestors.
  - Flash **red** if total increased.
  - Flash **green** if total decreased.
- **Material Specs:** description, UOM, material type, quantity, path.

## C. Exploded Schematic View (Graph Mode)
- Toggle from tree to radial/node graph.
- Center = finished good.
- Outer rings = deeper levels.
- Optional path trace: hover a node to highlight full parent chain to root.

## D. Three Required UX/Logic Rules

1. **Level Shifting**
- Render with level-based indentation:

```css
.tree-row {
  padding-left: calc(var(--level) * 20px);
}
```

2. **Parent Highlighting**
- Hover/select a node to highlight all ancestors (`L(n-1)...L0`) so users never lose context.

3. **Smart Aggregation (Commonality Report)**
- Group by material code across all branches.
- Output total demand and where-used count.

Example for this BOM:
- `SCRW-99` currently appears once with total demand 4.
- If reused in multiple branches, total demand should auto-sum across all occurrences.

## E. Suggested Technical Stack
- **Framework:** React (or Vue)
- **Tree-grid:** TanStack Table (custom tree rows) or AG-Grid tree data mode
- **State:** Zustand/Redux (React) or Pinia (Vue)
- **Data model:** canonical `BOMNode { id, level, material, type, qty, parentId, children[] }`
- **Computation engine:** memoized selectors for:
  - rolled cost
  - ancestor path
  - commonality report

## F. MVP Build Sequence
1. Paste + parse + auto-map.
2. Build nested JSON via hierarchy engine.
3. Render tree with level indentation and badges.
4. Add detail panel with inline price editing.
5. Add real-time roll-up and color delta feedback.
6. Add drag/drop restructuring.
7. Add commonality report and graph toggle.

This sequence delivers visible user value early while preserving extensibility for deeper engineering workflows.
