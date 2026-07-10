# Instance Tree Explorer — Implementation Task Breakdown

Splits the MVP (debug-suite-plan.md #1, design locked in
explorer-virtualization-design.md) into smaller, sequenced build tasks.
Each task should be independently buildable/testable before the next starts.

## 1. Instance tracking hooks (prerequisite, shared)

- Global hooks: `Instance.new`, `ChildAdded`, `ChildRemoved`
- Startup retroactive walk (`game:GetDescendants()`), tag pre-existing
  instances as "no creation record"
- Exposes a live per-instance `childs` list other tasks consume
- Lune-testable in `src/core/` per debug-suite-plan.md's core/roblox split

## 2. Static flat-list rendering (no windowing, no collapse)

- Precompute full flattened visible list from the instance tree
- Render every row via Vide `indexes()`, fixed row height, keyed by
  `Instance` reference (not path string)
- Goal: prove the row component + Vide keyed-diffing works before adding
  windowing complexity

## 3. Viewport windowing

- `ScrollingFrame:GetPropertyChangedSignal("CanvasPosition")` listener
  (event-driven, no polling)
- Compute visible slice from scroll offset + fixed row height
- Absolute `CanvasSize`/`Position` offsets per windowed row (no
  `UIListLayout` + spacers)
- No overscan buffer initially; add only if profiling shows pop-in
- No manual row pooling; rely on `indexes()` create/destroy diffing

## 4. Expand/collapse with splice-based structure

- Replace the flat precomputed array from Task 2 with the linked-list-style
  visible-entries structure (prev/next pointers)
- Each entry carries precomputed `childs` (from Task 1), kept live via hooks
- Expand: splice node's `childs` in at position (O(subtree size))
- Collapse: remove contiguous range via a cached "visible descendant count"
  bubbled up the ancestor chain
- Default: collapsed, tracked via `source(Set<Instance>)` of expanded nodes

## 5. Scroll-to-position (Q68 — still undecided)

- Start with simplest option: delta-walk from last-known windowed anchor
- Defer periodic anchor table / order-statistics tree until delta-walk is
  proven insufficient for long jumps (e.g. scrollbar click-to-position)

## 6. Search / filter

- Filter the live visible structure by name/class match
- Jump-to-instance: expand ancestor chain + scroll to target row

## 7. Highlight-in-3D + focus mode

- Reversible focus mode: "now focusing X" hint, 3D viewport preview
- Isolated preview (cloned instance, auto-framed) by default; toggle to
  live world-context camera
- Wire into Navigation system (`Navigation.selectInstance`, history stack)

## 8. Property Watcher (nested panel)

- Pin instances, watch property changes live, inline in the tree row
- Last — depends on stable row/tree structure from Tasks 2-4

---

Dependencies: 1 blocks everything; 2 before 3; 3 before 4; 4 before 5-8.
6-8 can proceed in parallel once 4 is stable.
