# Roblox Debug Suite — Consolidated Plan

## Purpose

An in-game script suite for **contracted, authorized vulnerability assessment** of Roblox games. The user is hired by game owners (friends/clients) to find and report exploits in `.rbxl` files they've been given explicit access to test — this is authorized security work, not testing games without permission. The actual goal is understanding unfamiliar codebases and **discovering flaws/exploits** in them, using tooling that emulates real exploit-executor capability — built as legitimate code baked into the game being tested (no actual injection, no ToS risk), since the user has full authorized access to the file anyway.

**Explicitly out of scope:** the suite does not handle reporting/writeup generation for clients — its job ends at finding and understanding flaws; producing the actual client deliverable happens outside the tool.

Manual execution/loading is out of scope for this plan — the user loads it themselves, however they choose.

---

## Build Priority

1. Instance Tree Explorer (MVP)
2. Script Debugger
3. Findings Panel
4. Event Monitor
5. AI Chat Panel
6. Asset Explorer (last)

---

## Core Modules (Panels)

1. **Instance Tree Explorer**
   - Live hierarchy view, search/filter, jump-to-instance, highlight in 3D
   - Nested: **Property Watcher** — pin instances, watch property changes live, inline in the tree
   - Fully suite-mediated tracking (see Instance Tracking below)

2. **Script Debugger**
   - Always-on `hookmetamethod` trap layer (lightweight, always available)
   - Opt-in per-script **virtualization** for real breakpoints/step/pause/halt (see Virtualization below)
   - Watch-expression / immediate-eval on a paused script's live environment (no standalone console)
   - AI: error explainer (scoped to open script, setting-gated), breakpoint condition assistant, watch expression suggestions

3. **Event Monitor**
   - Fire/listen log for RemoteEvents, RemoteFunctions, BindableEvents, with payload inspection
   - Filter mode: hard-filter or highlight-in-context, user-toggleable
   - Payload `Instance` references are clickable → navigate to Tree Explorer
   - AI: payload schema inference + auto parameter naming; "Analyse" button per remote triggers AI-assisted fuzzing skill (see AI System)

4. **Asset Explorer**
   - Instance-tree scan (browsable "what's referenced now" list) + `ContentProvider` hook layer (load times/failures) — combined views in one panel
   - Data-only: no preview viewport (cut — polish, not flaw-discovery; sounds still passively observed for live playback state, never triggers playback)

5. **Findings Panel** (new)
   - **Not a data store** — a categorized index/reference layer over annotations that live with their owning objects (see Annotations Store)
   - Categories (non-fixed, evolving): flaws/exploits, codebase notes, inferred schemas, bookmarks, analysis history, confidence/status tags
   - Every entry links back to its real object via the Navigation system
   - Every flaw entry tagged by **Exploit Pattern Library** category

6. **AI Chat Panel**
   - General-purpose conversational surface for the same unified agent core used elsewhere
   - Full skills/tool access into the suite

*(Cut from scope: Command Console, standalone Network/Replication Inspector, standalone Player/State Inspector, standalone Performance Profiler, standalone Memory/Leak Tracker, standalone Physics Visualizer.)*

---

## UI / Layout Architecture

- **Docking:** Fully free/rearrangeable panels (VS Code/Zed-style), not fixed zones
- **Layout persistence:** Global (not per-game), panels reopen exactly where last closed; first-ever open uses a default position
- **Master toggle:** Persistent square button, top-center of screen (Zed-style) → dropdown listing every tool → click to open/close
- **Visual style:** Zed Editor–inspired aesthetic (clean, minimal, dark-first)
- **UI framework:** Vide (reactive UI library)

---

## Cross-Panel Integration Model

Two distinct mechanisms — kept conceptually separate:

1. **Explicit navigation** (user-driven, moves focus)
   - Triggered by: double-click, context menu action, clickable instance paths in script source, clickable `Instance` values in payloads
   - Implemented as ergonomic functions (e.g. `Navigation.openScript(...)`, `Navigation.selectInstance(...)`) that internally set shared **Vide reactive state** — no separate event bus
   - **Universal navigation history** (browser-style back/forward) across all panels, built on top of these functions as a recorded stack
   - Tree Explorer additionally gets a **reversible focus mode**: focusing an instance shows a visual "now focusing X" hint and a 3D viewport preview, with easy return to prior state
     - Viewport defaults to **isolated preview** (cloned instance, auto-framed); toggleable to **live world-context** camera mode

2. **Ambient state reflection** (passive, never steals focus)
   - Panels independently observe the same underlying Vide reactive state and react visually (badges, pulses, highlights) — not direct panel-to-panel calls
   - Example: a RemoteEvent fires → Event Monitor logs it → the same instance in Tree Explorer gets a passive visual hint

**State management:** Shared Vide reactive `source()`s are the sole cross-panel communication mechanism (chosen over a dedicated event bus — typed, fast, reactive, avoids maintaining two parallel systems).

---

## Instance Tracking

- **Fully suite-mediated** (not just passive signal-reading): global hooks on `Instance.new`, `ChildAdded`, `ChildRemoved`, etc., giving a complete audit trail — chosen deliberately to leave room for future third-party module integration, even though a simpler hybrid approach would cover the user's own immediate needs
- **Startup:** Full retroactive walk (`game:GetDescendants()`) on suite load, tagging pre-existing instances as "no creation record" (honest about what's unknowable), then hooks forward for everything new

---

## Instance Soft-Delete

- Deleting an instance doesn't truly `:Destroy()` it — it's `Parent = nil`'d and stashed in a suite-managed holding area, recoverable via undo
- **Session-scoped**: held instances persist for the suite's current session, auto-clear on unload/restart — no manual cleanup needed, no unbounded accumulation risk

---

## Script Virtualization & Debugging

**Architecture:**
- Each script gets its own **closure with controlled source**, giving full access to source/state via an emulated `decompile`-style function
- **VM base:** `Fiu` (the Luau bytecode interpreter underlying vLuau), used **directly**, not through vLuau's `loadstring` convenience wrapper — needed because the project requires deeper hook access (real step/pause/halt/breakpoints) than that abstraction offers
- **Compiler:** LuauCeption (source → bytecode) as the front-end feeding Fiu

**Two complementary hooking layers (always both available, can overlap on the same script):**
1. **`hookmetamethod`** — always-on (or opt-in per script) lightweight trap layer, general-purpose metatable/call interception (`__index`, `__newindex`, `__namecall`, etc.), independent of virtualization
2. **Virtualization** — heavier, deliberate per-script escalation: original script's `Disabled` set true, real execution halted, virtualized closure (via Fiu) takes over under the *same* tree node, giving true breakpoint/step/pause/halt control

**Experimental "virtualize-all" mode:**
- Every script (or a user-selected scoped subset — folder, tag, manual multi-select) can be virtualized from suite startup instead of opt-in per-script
- Enables safely pausing multiple scripts' VM threads together (no state loss, unlike disabling native scripts) — full scope and scoped-subset variant both supported

**Freeze/pause mechanisms** (independently toggleable, each with disclosed scope/limits — no method achieves a true engine-wide freeze):
1. **Physics freeze** — anchor tracking (careful restore of pre-existing anchored state)
2. **RunService throttle** — suppress Heartbeat/Stepped/RenderStepped firing for suite-tracked connections
3. **Hook-based yield** — using the always-on `hookmetamethod` layer, yield at any trapped call site that's actually yieldable (Luau distinguishes yieldable vs. non-yieldable contexts)
   - Non-yieldable status determined via a **static, precomputed known-list** for now (dynamic/runtime detection deferred until proven necessary)
   - Non-yieldable points get a **visual hint only** (no logging) — best-effort freeze, honestly labeled
4. **Disable-other-scripts freeze** — only available/safe under the virtualize-all(-or-scoped) mode, since it pauses VM threads rather than destroying native script state

Emulated exploit function set (`decompile`, `getscripthash`, `getconnections`, etc.) is **not pre-listed** — grows organically as each panel/skill actually needs a given function (avoids over- or under-scoping).

---

## Filesystem Emulation (fs layer)

- Emulates real-executor fs functions: `writefile`, `readfile`, `isfile`, `listfiles`, `delfile`, etc.
- **Backing store:** `DataStoreService`
- **Structure:** Virtual filesystem index (path → chunk IDs) + chunked storage — handles nested paths, long names, and files beyond DataStore's per-key size limit
- **Write consistency:** Write-then-commit pattern — new/changed chunks written under temp keys first, index only updated after all succeed, so a failure mid-write leaves the prior version intact
- **Versioning:** Rolling last-N (5–10) versions per file, auto-pruned — makes file writes genuinely reversible (not just theoretically)
- This layer is the shared persistence backbone for: layout state, the Annotations Store (Findings panel), and the Exploit Pattern Library

---

## Testing & CI

- **Testable by Lune** (standalone Luau runtime, no Roblox globals)
- **Split:** `src/core/` (pure Luau, zero Roblox API references, Lune-testable) vs. `src/roblox/` (adapters, UI, anything touching `game`/`Instance`/DataStore)
- **Boundary enforcement:** CI lint script scans `src/core/` for forbidden Roblox globals, fails build on violation
- **Test framework:** `frktest` (already proven in this exact Lune/Luau ecosystem via LuauCeption's own test suite)
- **CI:** Full GitHub Actions pipeline — runs `frktest` suite and boundary-lint script on every push/PR, hard-blocking on failure

---

## AI System

**Provider architecture:**
- Direct `HttpService` calls to provider APIs (no proxy/relay layer)
- Ships with a **built-in library of pre-configured providers** (endpoint, auth format, model list) so the user only ever pastes an API key
- First working provider: **OpenCode** (OpenAI-compatible API shape)
- Provider config format is pluggable/extensible by design, for adding more providers incrementally later

**Model selection:** Global default model + optional per-feature override in Settings

**UX for unconfigured AI features:** Visually present but inactive (never hidden) — hover shows "needs setup" tooltip, click redirects to Settings

**Integration philosophy:** AI is embedded *into* existing panels contextually (not standalone "Code Analysis"/"Code Generation" panels), plus one dedicated **AI Chat panel** for general-purpose/exploratory agentic use

**Unified agent core (one architecture, two invocation modes):**
- **Simple single-shot calls** for narrow text-transform tasks with no context-gathering need (e.g. breakpoint condition assistant)
- **Full agentic loops with tool/skill access** for context-variable tasks (e.g. remote param naming, natural-language instance search, Chat panel) — agent gets a starter context payload relevant to the current focus, plus tools to actively gather more as needed, rather than a fixed context dump

**Action-gating (skills that mutate suite/game state):**
- Reversibility-based rule: reversible actions (navigate, read, select, toggle hooks, **write/overwrite files** — reversible now thanks to fs versioning, **delete instances** — reversible now thanks to soft-delete) execute freely
- Only remaining confirm-required action: **script virtualization/disabling** (destroys native script state, no undo possible)
- **Bypass-permissions mode:** user-toggleable in Settings; when active, the hard confirmation gate is skipped, but the agent still carries system-prompt guidance to exercise judgment around irreversible-ish actions
  - Agent's self-judged-risk behavior is itself configurable in Settings (ask-in-chat / proceed-and-flag / tiered bypass), **defaulting to "ask in chat before proceeding"**

**Data Scoping (Per-Project vs. Global):**

Since the user tests multiple different client games (each a separate engagement), data is split by scope:

- **Per-project (scoped to the game's PlaceId/file):** Findings, annotations, pattern-match history — testing Game A never mixes with Game B
- **Global (shared across all engagements):** Exploit Pattern Library, layout persistence, Settings (providers, models, bypass mode config)

**Pattern Library promotion:** Every approved pattern (built-in or AI-discovered within a specific engagement) always promotes to the shared global library, available for all future engagements — reflects the user's accumulated professional expertise growing with each job, no per-pattern scoping choice needed.

**Findings Panel — Annotations Store:**
- Findings/notes/schemas/bookmarks are **not** centrally stored data — they're attached to their owning object (remote, script, instance) via a **separate annotations store**, keyed by stable object identity, independent from the Q22 audit-trail tracking system
- Persisted via the fs emulation layer (JSON files under suite-managed paths), gaining versioning for free
- Findings panel is purely an indexed/categorized *view* over this store; clicking an entry uses Navigation to jump to the real object

**Exploit Pattern Library:**
- A shared, reusable taxonomy of known vulnerability/exploit patterns (e.g. "missing ownership check," "client-trusted value," "unvalidated remote argument") that any AI skill can detect against
- Every Finding gets tagged with the pattern(s) it matches, enabling category-based browsing in the Findings panel
- **Growth model:** built-in starter set + user-authored patterns + AI-proposed patterns (from recurring cross-session observations)
- AI-proposed patterns land in a **review queue**, require explicit user approval before becoming active/referenceable

**Confirmed AI feature set (embedded per panel):**
- Script Debugger: error explainer (scoped to open script, setting-gated), breakpoint condition assistant, watch expression suggestions
- Tree Explorer: natural-language instance search (context-integrated across panels, not isolated fuzzy name-matching)
- Event Monitor: payload schema inference, auto remote parameter naming, "Analyse" button per remote (triggers fuzzing/analysis skill — especially valuable for RemoteFunctions, whose return values/errors often leak server-side implementation details)
- Cross-cutting: Exploit Pattern Library detection, Findings categorization, AI-proposed pattern discovery

*(Cut: standalone Code Analysis/Generation panels, failed-asset AI triage, "Explain this panel" AI summarizer, AI-authored dev-facing reports — replaced by: structured onboarding flow (non-AI) and the Findings panel as the real "understanding" surface.)*

---

## Open / Deferred Items

- Exact Exploit Pattern Library starter set (content, not architecture)
- Dynamic (runtime-detected) non-yieldable hook points — only if the static list proves insufficient
- Second AI provider adapter (native, non-OpenAI-compatible format) — deferred until OpenCode integration is solid
- Onboarding flow — flagged as needed, not yet designed
