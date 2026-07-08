# Instance Tree Explorer — Virtualized Rendering Design

Design decisions from grilling session, resuming debug-suite-plan.md Q57+.

## Context
First concrete component to build (MVP #1: Explorer, per Build Priority in
debug-suite-plan.md). "Virtualized" here = UI list virtualization (render only
visible rows), unrelated to the Script Debugger's Fiu VM "script virtualization."

## Library check
Vide (reference/vide, local clone, gitignored) has `indexes()`/`values()` for
keyed reactive list diffing (create/destroy components as a reactive table
changes), but **no built-in viewport windowing** — every item in the source
gets a live component regardless of visibility. Windowing must be hand-rolled.

## Locked decisions

1. **Approach:** flatten-then-window, not native tree-walk windowing (Q59).
2. **Row height:** fixed, uniform across all rows (Q60). No variable-height
   rows — richer per-row info goes in the nested Property Watcher, not the
   tree row itself.
3. **Row key:** the Roblox `Instance` reference itself, not a derived path
   string (Q61). Path strings only matter for the separate Annotations Store
   (persistence across sessions), not this component.
4. **Scroll trigger:** `ScrollingFrame:GetPropertyChangedSignal("CanvasPosition")`,
   event-driven — not Heartbeat/RenderStepped polling (Q62).
5. **Canvas sizing:** absolute `CanvasSize`/`Position` offsets for windowed rows
   (Option A) — not `UIListLayout` + literal spacer frames (Q63).
6. **Overscan buffer:** none by default; add only if profiling shows pop-in
   during fast scroll (Q64). Not assumed necessary given optimized hooks +
   potential parallel Luau rendering.
7. **Row instance pooling:** none — let `indexes()`'s natural create/destroy
   diffing handle it, since windowed-slice membership changes are already
   keyed by Instance ref (Q65). Profile before adding manual recycling.
8. **Expand/collapse default:** collapsed by default, `source(Set<Instance>)`
   of expanded nodes (Q66).

### Flat-array/visible-set structure (refined beyond initial flatten proposal)

User's optimization, adopted over the simpler "filter full precomputed array
on every toggle" approach:

- Maintain a **linked-list-style structure** (prev/next or parent/prev-sibling
  pointers) of currently-*visible* (non-collapsed) entries only — not a flat
  array with absolute indices (Q67, option C).
- Each entry carries its own recursively-expanded `childs`, precomputed and
  kept live via the existing global instance-tracking hooks
  (`Instance.new`/`ChildAdded`/`ChildRemoved` from debug-suite-plan.md's
  Instance Tracking section) — no re-walking the live Roblox tree on toggle.
- **Expand:** splice the node's precomputed `childs` in at its position —
  O(subtree size), not O(N).
- **Collapse:** remove a contiguous range covering the node's own subtree plus
  any independently-expanded descendants. Range length determined by a live
  **"visible descendant count"** cached per entry, incremented/decremented as
  descendants expand/collapse and bubbled up the ancestor chain (Q68, option A
  under Q67 — chosen over force-collapsing descendants on every collapse,
  which would lose nested expand state).
- **No absolute indices stored** — avoids O(N) renumbering on every splice/
  delete. A flat array with real indices is only materialized transiently for
  the *windowed slice* at render time.

## Open / unresolved

- **Q68 — fast scroll-to-arbitrary-position without absolute indices.**
  Walking from the list head is O(N) per scroll change, defeating the point.
  Options on the table:
  - (C) Walk from last-known windowed anchor, delta-only — matches actual
    scroll usage (near-continuous CanvasPosition changes), no extra structure.
  - (C)+(A) — (C) plus a coarse periodic anchor table (index+node cached every
    K visible nodes, refreshed incrementally near splice points) as a fallback
    for long jumps (e.g. scrollbar click-to-position).
  - (B) — full order-statistics tree / skip list for O(log N) both-directions
    lookup. Textbook-correct, more implementation complexity.
  - **Not yet decided** — awaiting user's choice when grilling resumes.
