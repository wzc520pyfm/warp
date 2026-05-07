# Tech Spec: Multi-session tabs inside a single terminal/agent pane

**Issue:** [warpdotdev/warp#10367](https://github.com/warpdotdev/warp/issues/10367)
**Product spec:** `specs/GH10367/product.md`

## Context

### Single-session pane today

Today every terminal/agent pane is a `TerminalPane` that owns exactly
one `TerminalView` plus its `TerminalManager`. The relevant types and
call-sites that this spec touches:

- **`app/src/pane_group/pane/terminal_pane.rs`**
  - `pub struct TerminalPane` (`:73-87`) — fields:
    `model_event_sender`, `uuid: Vec<u8>`,
    `pane_configuration: ModelHandle<PaneConfiguration>`,
    `view: ViewHandle<TerminalPaneView>`. The struct is built around
    *one* `view` field; "the pane's session" is read off this single
    view via `terminal_view(ctx)` (`:193-195`) and
    `terminal_manager(ctx)` (`:203-208`).
  - `TerminalPane::new(...)` (`:159-184`) — builds the
    `TerminalPaneView` and stores it in `view`.
  - `attach(...)` (`:262-363`) and `detach(...)` (`:365-421`) wire up
    subscriptions and registrations against that single
    `terminal_view`.
  - `snapshot(...)` (`:423-519`) — emits exactly one
    `LeafContents::Terminal(TerminalPaneSnapshot)` /
    `LeafContents::AmbientAgent(AmbientAgentPaneSnapshot)`.
  - `pub type TerminalPaneView = PaneView<TerminalView>` (`:70`) — wraps
    a `TerminalView` and a `PaneStack<TerminalView>` for the existing
    push/pop "agent view stack" (which is unrelated to session tabs).
- **`app/src/terminal/view/pane_impl.rs`**
  - `TerminalView::pane_header_overflow_menu_items(...)` (`:647-713`) —
    today returns only shared-session and split-pane entries; this is
    where the new "New session" item goes.
  - `BackingView::render_header_content(...)` (`:729-738`) —
    `TerminalView` renders the pane's three-column header
    (`render_terminal_pane_header`, `:573-607`); this is the slot the
    tab strip will swap into when there are ≥2 tabs.
  - `update_pane_configuration(...)` (`:117-153`) — derives the per-tab
    title used by the strip; reused unchanged.
- **`app/src/pane_group/pane/mod.rs`**
  - `pub struct PaneStack<P: BackingView>` (`:895-983`) — a stack of
    backing views with `push` / `pop` / `active_view` /
    `entries`. Currently used as a *navigation stack* (one visible at
    a time, e.g. the agent-view fullscreen subtree). It is not a tab
    group: only the topmost entry is rendered, and pop never returns
    the last item.
  - `pub struct PaneConfiguration` (`:688+`) — the data source for the
    pane header (title, secondary text, overflow menu items, share
    object). Populated by the active `TerminalView`'s
    `update_pane_configuration` chain.
- **`app/src/app_state.rs`**
  - `pub enum LeafContents { Terminal(TerminalPaneSnapshot), … }`
    (`:120-143`) — the persisted pane content.
  - `pub struct TerminalPaneSnapshot` (`:193-206`) — one
    snapshot per pane: `uuid`, `cwd`, `shell_launch_data`,
    `is_active`, `is_read_only`, `input_config`,
    `llm_model_override`, `active_profile_id`,
    `conversation_ids_to_restore`, `active_conversation_id`. There is
    no list-of-sessions in the schema today.
- **`app/src/code/view.rs`** (reference, not modified)
  - `TabData` (`:209-228`), `tab_group: Vec<TabData>` (`:230`),
    `active_tab_index` (`:232`), and the open / promote / focus
    helpers (`:264, :339, :656, :706, :918`) — the existing model that
    the issue cites as prior art. The shape of `tab_group` is what we
    mirror inside `TerminalPane`.
- **`crates/integration/`**
  - The integration runner (`crates/integration/src/bin/integration.rs`)
    and `tests/integration/ui_tests.rs` are the harness for end-to-end
    UI flows; the existing `code_view_overflow_menu` test (currently
    untracked at `crates/integration/src/test/code_view_overflow_menu.rs`)
    is the closest pattern for the new pane-overflow-menu integration
    coverage.

### Why a new tab group instead of reusing `PaneStack`

`PaneStack` is the existing "multiple `TerminalView` per pane"
abstraction, but its semantics are *stack* (one rendered, push/pop,
last-item pop is rejected). Generalizing it into a tab group would
require:

- Allowing more than one entry to be considered "visible" at once for
  drag/reorder semantics.
- Decoupling "active entry" from "topmost entry" (tabs reorder).
- Removing the `pop` last-item refusal, so closing a tab can still
  emit `ViewRemoved`.
- Untangling consumers that today rely on stack semantics (the
  fullscreen-agent-view back stack uses `depth() > 1` as "in nav
  stack" — see `pane_impl.rs:246-250`).

The cost of widening `PaneStack` outweighs the duplication of having
a sibling tab-group concept inside `TerminalPane`. We keep `PaneStack`
as-is for the navigation stack, and add a new `TerminalSessionTabs`
container on `TerminalPane` for the new tab semantics. (Considered and
rejected: see "Alternatives considered" below.)

### Why heterogeneous tabs (terminal + agent in one pane)

Per product spec invariant 32 / non-goal #3, a tab is just a
`TerminalView`. `TerminalView` already swaps between terminal and
fullscreen agent mode in place
(`agent_view_controller.is_fullscreen()`); we don't need a new
"tab kind" enum — each tab carries its own `TerminalView` and renders
whichever mode that view is in.

## Proposed changes

The implementation is grouped as: feature flag → data model →
rendering → actions/events → persistence → wiring → tests. Each block
is independent enough to land as its own commit on the same branch
once the spec is approved.

### 1. Feature flag

Per `.claude/skills/add-feature-flag/SKILL.md`:

- Add `multi_session_tabs_in_pane = []` to the `[features]` section of
  `app/Cargo.toml` (not under `default`).
- Add `FeatureFlag::MultiSessionTabsInPane` to
  `crates/warp_core/src/features.rs` and the `#[cfg(feature = …)]`
  match in `app/src/lib.rs`.
- Do *not* add it to `DOGFOOD_FLAGS` initially; the rollout is staged
  separately (see Follow-ups).

All new behavior is wrapped in
`if FeatureFlag::MultiSessionTabsInPane.is_enabled()`. With the flag
off, `TerminalPane` stays in the single-tab code path; the new
data structures collapse to a 1-element vector and the strip
renderer is bypassed.

### 2. Data model: `TerminalPane` becomes a tab container

Refactor `TerminalPane` (`app/src/pane_group/pane/terminal_pane.rs`)
from "owns one `view`" to "owns a tab group whose active entry is the
current `view`". Net shape:

```rust
pub struct TerminalPane {
    model_event_sender: Option<SyncSender<ModelEvent>>,

    /// The pane's tabs. Always non-empty (a pane with zero tabs
    /// closes itself, see `close_tab_or_pane`). When
    /// `MultiSessionTabsInPane` is disabled, length is always 1.
    tabs: Vec1<TerminalSessionTab>,

    /// Index into `tabs` of the currently-active session.
    active_tab_index: usize,

    /// The pane-level configuration that drives the pane header /
    /// strip. Always points at the active tab's per-tab configuration
    /// (re-aimed when the active tab changes).
    pane_configuration: ModelHandle<PaneConfiguration>,
}

pub(crate) struct TerminalSessionTab {
    /// Per-tab UUID (matches today's `TerminalPane.uuid` for
    /// session-restoration identity).
    uuid: Vec<u8>,
    /// The tab's `PaneView<TerminalView>` (same as today's
    /// `TerminalPane.view`). Owns its `TerminalManager`.
    view: ViewHandle<TerminalPaneView>,
    /// Per-tab pane configuration (title, indicators, overflow items
    /// for that specific session).
    pane_configuration: ModelHandle<PaneConfiguration>,
}
```

Helper API on `TerminalPane`:

- `pub fn active_tab(&self) -> &TerminalSessionTab` /
  `active_tab_mut`
- `pub(crate) fn terminal_view(&self, ctx) -> ViewHandle<TerminalView>` —
  unchanged signature; returns `active_tab().view.as_ref(ctx).child(ctx)`.
- `pub(crate) fn terminal_manager(&self, ctx) -> ModelHandle<…>` —
  unchanged signature; returns the active tab's manager.
- `pub fn session_uuid(&self) -> Vec<u8>` — backwards-compat: returns
  the active tab's `uuid`. Existing call-sites that key off the pane's
  uuid (block deletion, model-event sender) continue to work without
  understanding tabs.
- `pub fn tabs(&self) -> &[TerminalSessionTab]` — for the strip
  renderer and persistence.
- `pub fn open_new_session_tab(&mut self, ctx) -> usize` — creates a
  fresh `TerminalView` + `TerminalManager` (using
  `PaneGroup`'s existing factory; details below) and appends it.
  Returns the new tab's index.
- `pub fn switch_to_tab(&mut self, index: usize, ctx)` —
  re-aims `pane_configuration`, calls `redetermine_global_focus` on
  the new active tab's view, fires
  `Event::ActiveSessionChanged`.
- `pub fn close_tab(&mut self, index: usize, ctx) -> CloseTabOutcome`
  where `CloseTabOutcome` is `{ ClosedTab, ClosedLastTabClosePane }`.
  Routes through the existing `close_pane_with_confirmation` path
  scoped to that tab; on confirmation it removes the tab from
  `tabs` and emits `PaneStackEvent::ViewRemoved` for the underlying
  `TerminalPaneView`. If `tabs` becomes empty, returns
  `ClosedLastTabClosePane` and the caller (`PaneGroup`) closes the
  pane via the existing `close_pane` path.
- `pub fn reorder_tab(&mut self, from: usize, to: usize, ctx)` —
  swaps positions in `tabs`, adjusting `active_tab_index` if it
  moved. Emits a `PaneStackEvent::TabReordered { from, to }` (new
  event variant — see Events below).

Critical correctness points:

- **Drop order.** Today `TerminalManager` must drop before
  `TerminalView` to avoid event-loop deadlock (see existing comment on
  `TerminalPane.view`). With `Vec1<TerminalSessionTab>`, each tab is
  dropped via `Vec1`'s `Drop` (which drops `TerminalSessionTab` in
  insertion order). `TerminalSessionTab` mirrors today's TerminalPane
  field order so the manager-before-view invariant holds *per tab*.
  The pane drops its tabs in declaration order; we keep `tabs` as the
  last field of `TerminalPane` and document the requirement next to
  the field.
- **`PaneId`.** The pane's identity (`PaneId::from_terminal_pane_view`)
  is computed off the active tab's view. Switching tabs would change
  the `PaneId`, which is a stability hazard for `PaneGroup`'s
  `pane_contents` map. Resolution: `PaneId` keeps a *pane-scoped*
  identity that does not change on tab switches. We introduce a
  `PaneId::TerminalTabs(EntityId)` variant keyed off
  `TerminalPane`'s own `EntityId` (allocated via `ctx.add_typed_action_view`
  on the *outer* `TerminalPane` model). Existing
  `PaneId::from_terminal_pane_view` (currently the only constructor)
  is renamed to `PaneId::from_terminal_pane(pane_handle)` and built
  from the outer pane handle. Migration is mechanical; the active
  `TerminalView`'s id is still reachable via `terminal_view(ctx).id()`
  for any caller that genuinely wants the inner-view id (e.g.
  `LLMPreferences::get_base_llm_override`).

### 3. Rendering: tab strip in the pane header slot

The pane header is currently rendered by `TerminalView`'s
`BackingView::render_header_content` (only the active view does so).
With multi-tabs, the header has to know about *all* sibling tabs.

Approach: introduce a new render layer above `TerminalView`'s
header. Concretely, in `TerminalPane`:

1. Add a `header: ViewHandle<TerminalPaneHeader>` field, where
   `TerminalPaneHeader` is a new view type (in
   `app/src/pane_group/pane/view/terminal_pane_header.rs`).
2. The pane group asks the *pane's* header renderer for the strip
   when `tabs.len() >= 2`. When `tabs.len() == 1` the renderer
   delegates to the active tab's existing
   `TerminalView::render_terminal_pane_header` → no change in
   single-tab visuals.
3. `TerminalPaneHeader` reads `tabs` and `active_tab_index` and
   renders:
   - Left: optional back button (mirrors
     `maybe_render_header_back_button` for the active tab).
   - Center / fill: a horizontally-scrollable `Flex::row` of
     per-tab elements (`render_session_tab(tab, is_active)`).
   - Right: the existing right-side action cluster
     (`render_header_actions` of the active tab — share, info,
     overflow `…`, pane close).
4. `render_session_tab` produces the same title pieces today's
   pane header centers (agent indicator, share indicator, terminal
   title / conversation title), but constrained to a tab's width and
   followed by a close `×` button. The close `×` dispatches a new
   `TerminalAction::CloseSessionTab { index }` (see Actions below);
   left-click on the tab body dispatches
   `TerminalAction::SwitchSessionTab { index }`.
5. Per `warp-ui-guidelines`: do **not** introduce a new
   `ActionButtonTheme`. The active-tab styling reuses the
   design-system tokens already used by `CodeView`'s tab strip; the
   close `×` reuses `NakedTheme` with the existing close icon.

The tab strip's height equals `PANE_HEADER_HEIGHT` (`:22`) so neighbor
panes don't shift when transitioning between single-tab and
multi-tab. The single-tab code path is untouched, satisfying product
invariant 1.

### 4. Actions and events

#### Overflow menu entry

In `TerminalView::pane_header_overflow_menu_items`
(`pane_impl.rs:647`), when
`FeatureFlag::MultiSessionTabsInPane.is_enabled()`, prepend a "New
session" entry (with a `Separator` between it and the existing
shared-session items). The action is a new
`TerminalAction::OpenNewSessionTabInPane`.

#### Action variants (new)

In the `TerminalAction` enum (`app/src/terminal/view/mod.rs` or
wherever `TerminalAction` lives — to confirm during implementation):

```rust
TerminalAction::OpenNewSessionTabInPane,
TerminalAction::SwitchSessionTab { index: usize },
TerminalAction::CloseSessionTab { index: usize },
TerminalAction::ReorderSessionTab { from: usize, to: usize },
```

`TerminalView::handle_action` routes these to the pane via a new
`Event::PaneTabsAction(PaneTabsAction)` so the *pane group* (which
owns the pane and its tabs) is the actor:

```rust
pub enum PaneTabsAction {
    OpenNewSession,
    Switch(usize),
    Close(usize),
    Reorder { from: usize, to: usize },
}
```

`PaneGroup::handle_terminal_view_event` (`terminal_pane.rs:642`)
gets a new arm for `Event::PaneTabsAction(_)` that translates each
variant into the corresponding `TerminalPane` API call (§2). The
existing `TerminalAction::ToggleMaximizePane` arm is unaffected.

#### `PaneStackEvent`

Add `PaneStackEvent::TabReordered { from, to }` so subscribers (e.g.
`PaneGroup`) can react to reorder for layout/persistence purposes.
Existing `ViewAdded` / `ViewRemoved` continue to fire on
`open_new_session_tab` / `close_tab`. The current handler
(`handle_pane_stack_event`, `terminal_pane.rs:620-640`) gets a
`PaneStackEvent::TabReordered` arm that calls
`on_pane_state_change` so the active terminal view re-evaluates its
title/state.

### 5. Persistence

Extend `TerminalPaneSnapshot` (`app/src/app_state.rs:193`) with two
new fields, both serde-defaulted so old snapshots load unchanged:

```rust
pub struct TerminalPaneSnapshot {
    // ...existing fields...
    /// Sibling tabs (excluding self). Defaults to empty for old
    /// snapshots.
    #[serde(default)]
    pub sibling_tabs: Vec<TerminalPaneSnapshot>,
    /// Index into `[self, ...sibling_tabs]` of the active tab on
    /// restore. Defaults to 0 (the historical single-tab behavior).
    #[serde(default)]
    pub active_tab_index: usize,
}
```

Why `sibling_tabs: Vec<TerminalPaneSnapshot>` rather than a wrapping
`tabs: Vec<TerminalPaneSnapshot>`:

- Preserves the existing wire format. Old readers still parse the
  outer `TerminalPaneSnapshot` and ignore the new fields; the pane
  restores as a single-tab pane.
- New readers see `sibling_tabs.is_empty() && active_tab_index == 0`
  for old snapshots (single-tab) and proceed identically.
- The SQLite `pane_leaves` row's content blob is one
  `TerminalPaneSnapshot`; we don't need a new table or migration.

`TerminalPane::snapshot(...)` (`terminal_pane.rs:423-519`) is updated
to:

1. Compute `(active_snapshot, sibling_snapshots)` by calling the
   existing per-tab snapshot logic on every tab.
2. Place `active_snapshot` at index 0 of the conceptual restore list
   (i.e. it is the outer `TerminalPaneSnapshot`), and the rest as
   `sibling_tabs`. `active_tab_index = 0` because the active tab was
   placed first.
3. *Or*, equivalently, place tabs in their original order with
   `active_tab_index` pointing into the array. Either choice round-trips;
   the spec recommends "keep original order, set
   `active_tab_index`" because it preserves the user's tab order
   across restart for product invariant 23.

`AmbientAgentPaneSnapshot` is untouched; if the pane's active tab is
an ambient-agent session, the existing
`LeafContents::AmbientAgent(AmbientAgentPaneSnapshot)` branch is
chosen for the *outermost* snapshot but its sibling tabs are still
`TerminalPaneSnapshot`. Mixing variants per tab is supported by
storing the per-tab snapshot type in a thin wrapper (a new
`TerminalTabSnapshot` enum):

```rust
#[derive(Clone, Debug, PartialEq)]
pub enum TerminalTabSnapshot {
    Terminal(TerminalPaneSnapshot),
    AmbientAgent(AmbientAgentPaneSnapshot),
}
```

Adjusted snapshot fields:

```rust
pub struct TerminalPaneSnapshot {
    // ...existing fields...
    #[serde(default)]
    pub sibling_tabs: Vec<TerminalTabSnapshot>,
    #[serde(default)]
    pub active_tab_index: usize,
}
pub struct AmbientAgentPaneSnapshot {
    // ...existing fields...
    #[serde(default)]
    pub sibling_tabs: Vec<TerminalTabSnapshot>,
    #[serde(default)]
    pub active_tab_index: usize,
}
```

Restore path (in `app/src/persistence/...` — exact module location to
confirm during implementation; the workspace's
`restore_terminal_pane` / `restore_ambient_agent_pane` are the
entry points):

1. Build the active tab as today.
2. For each sibling tab: dispatch on `TerminalTabSnapshot::{Terminal, AmbientAgent}`
   to the same per-tab restore helper. Append to `tabs`.
3. After all tabs are built, set `active_tab_index` and call
   `switch_to_tab(active_tab_index, ctx)` so the pane wakes up
   pointing at the right tab.

#### Feature-flag-off downgrade

When the flag is disabled at startup, `restore_terminal_pane`
discards `sibling_tabs` after restoring the active tab. This
implements product invariant 24 ("collapse to active tab on
disable"). `tech.md`'s open-questions section calls out whether to
log a warning toast in that case (current default: no toast,
silently collapse — matches the way other flag-gated state
collapses today).

### 6. Wiring updates inside `PaneGroup`

`PaneGroup` is the manager of pane lifecycle. Each call-site that
today operates on the pane's single view needs a small, well-bounded
update:

- **Adding tabs.** `PaneGroup::add_terminal_pane` /
  `insert_terminal_pane` (existing) still create *new* panes, not new
  tabs; `PaneTabsAction::OpenNewSession` calls the new
  `TerminalPane::open_new_session_tab` directly. The factory used by
  that helper reuses `add_terminal_pane`'s existing
  TerminalManager+TerminalView construction logic; we extract it into
  a private `PaneGroup::build_terminal_session_tab(...)` helper so
  both paths share code.
- **Closing the last tab.** `PaneGroup::handle_terminal_view_event`'s
  existing `Event::Exited` and `Event::CloseRequested` arms
  (`terminal_pane.rs:656-673`) translate into "close *the active tab*"
  via `close_tab(active_tab_index)`. If `close_tab` returns
  `ClosedLastTabClosePane`, the pane is closed via the existing
  `close_pane(_with_confirmation)` path. No new pane-close path
  exists — confirmation logic is reused as-is.
- **Block deletion / model events.**
  `TerminalPane::delete_blocks` (`:211-226`) keys off the *active
  tab's* `uuid` and is now called once per tab during
  `detach(DetachType::Closed)` so each tab's blocks are cleaned up
  (today's behavior is "one pane = one uuid"; multi-tab needs
  iteration).
- **Active-view registrations.** `attach`/`detach` register the
  active terminal view with `ActiveAgentViewsModel`,
  `CLIAgentSessionsModel`, `BlocklistAIHistoryModel`, and the
  shared-session manager. We move those registrations to a
  `TerminalSessionTab::attach(...)` / `detach(...)` method that runs
  per tab. The pane-level `attach` calls each tab's `attach` in turn
  (and likewise for detach). This isolates per-tab subscription
  bookkeeping in one place.
- **Drag/drop.** Pane drag-to-split (`is_pane_being_dragged`,
  `:588-590`) targets the pane as a whole — unchanged. Tab drag
  inside the strip is local to `TerminalPaneHeader` and never
  promotes to a pane drag (the strip swallows the drag while the
  cursor is over a tab body).

### 7. Sync inputs and synced-session helpers

`SyncedInputState::should_sync_this_pane_group` (call-site
`terminal_pane.rs:288`) keys off `pane_group_id, window_id`, so it
keeps working unchanged. Inside `attach`, the call to
`group.send_sync_event_to_session(terminal_pane_id, &event, ctx)` is
intentionally left targeting *the active tab's underlying session*,
which is `terminal_pane_id`'s active tab. Because
`group.active_session_view(ctx)` already returns the active view for
the focused pane, no change is required for the sync path. Inactive
tabs do not receive synced input (product invariant 29).

### 8. Block / persistence hooks for restore

Existing `BlocklistAIHistoryModel::clear_conversations_in_terminal_view`
calls in `detach` (`:374-378`) iterate per-tab in the new model:

```rust
for tab in &self.tabs {
    let view_id = tab.view.as_ref(ctx).child(ctx).id();
    BlocklistAIHistoryModel::handle(ctx).update(ctx, |hist, ctx| {
        hist.clear_conversations_in_terminal_view(view_id, ctx);
    });
    tab.delete_blocks(ctx);
}
```

The existing per-pane `delete_blocks` becomes a per-tab method on
`TerminalSessionTab`.

## End-to-end flows

### Spawn → switch → close

```
User clicks "New session" in pane P's overflow:
  TerminalView::handle_action(OpenNewSessionTabInPane)
    → emits Event::PaneTabsAction(OpenNewSession) up to the pane group
  PaneGroup::handle_terminal_view_event arm:
    → P.update(|pane, ctx| pane.open_new_session_tab(ctx))
      → builds TerminalSessionTab (build_terminal_session_tab)
      → pushes onto P.tabs
      → switches active to new tab
      → emits PaneStackEvent::ViewAdded
      → emits Event::ActiveSessionChanged
  Render:
    → P.tabs.len() == 2, TerminalPaneHeader takes over the header slot.

User clicks tab 0:
  TerminalAction::SwitchSessionTab { index: 0 }
    → Event::PaneTabsAction(Switch(0))
  PaneGroup arm:
    → P.update(|pane, ctx| pane.switch_to_tab(0, ctx))
      → re-aims pane_configuration to tab 0's PaneConfiguration
      → focus follows: redetermine_global_focus on tab 0's TerminalView
      → emits Event::ActiveSessionChanged

User clicks × on tab 1 (busy with a running command):
  TerminalAction::CloseSessionTab { index: 1 }
    → Event::PaneTabsAction(Close(1))
  PaneGroup arm calls P.close_tab(1, ctx):
    → tab 1's TerminalView is busy → confirmation modal opens
    → on confirm: tab 1 is removed; PaneStackEvent::ViewRemoved fires
    → P.tabs.len() returns to 1; strip dissolves; single-tab header
      returns; close_tab returns ClosedTab
    → on cancel: nothing changes.
```

### Persist → reload

```
On save:
  PaneGroup::snapshot walks each pane.
  TerminalPane::snapshot() returns LeafContents::Terminal({
      // tab 0 (the active tab if the user selected it 0) fields...
      sibling_tabs: vec![TerminalTabSnapshot::Terminal(tab1_snap), ...],
      active_tab_index: 0,
  })

On restore:
  restore_terminal_pane sees TerminalPaneSnapshot.
  - Build tab 0 from outer fields.
  - For each sibling: dispatch on TerminalTabSnapshot variant.
  - Build TerminalPane with all tabs in order.
  - Call switch_to_tab(active_tab_index).
  - Run attach() once on the pane; per-tab attach() runs for each tab.
```

### Single-tab fallback (flag off)

```
TerminalPane is constructed with a single tab.
TerminalPane::tabs.len() == 1 forever (no menu entry to spawn more).
TerminalPaneHeader::render delegates to the active tab's
TerminalView::render_terminal_pane_header → identical to today.
Snapshot path writes sibling_tabs: [], active_tab_index: 0 →
   structurally identical content to today's payload, so SQLite
   reads/writes are bytewise the same.
```

## Testing and validation

Each numbered behavior in `specs/GH10367/product.md` maps to at least
one test below.

### Unit tests

Sibling tests file `app/src/pane_group/pane/terminal_pane_tests.rs`
(per repo convention `#[cfg(test)] #[path = "..."]` if not already
present) covering the data-model layer:

| Test | Guards invariants |
|---|---|
| `single_tab_panel_unchanged_when_flag_off` | 1, 2, 3 |
| `open_new_session_tab_appends_and_activates` | 4, 5, 6 |
| `switch_to_tab_reaims_pane_configuration_and_focus` | 13, 14 |
| `close_tab_removes_and_picks_neighbor` | 17, 19 |
| `close_last_tab_returns_close_pane_outcome` | 18 |
| `reorder_tab_keeps_active_index_consistent` | 21 |
| `snapshot_round_trip_with_multi_tab_uses_active_index` | 23 |
| `snapshot_old_format_loads_as_single_tab` | 24 |
| `snapshot_round_trip_collapses_when_flag_off` | 24 |
| `pane_id_stable_across_tab_switches` | (correctness, supports 13) |
| `inactive_tab_continues_to_receive_events` | 15 |

For the snapshot tests, build a fixed `TerminalPaneSnapshot` payload
representative of "today's" wire format (with no `sibling_tabs` /
`active_tab_index` fields) using a JSON literal and confirm
`serde_json::from_str` populates the new fields with their `default`
values. Pair with a forward-compat test that takes a multi-tab
snapshot, drops the `sibling_tabs` + `active_tab_index` keys
post-serialization, and asserts the resulting payload deserializes
cleanly into the old struct (only useful as a sanity check; we don't
have a "downlevel" parser).

For overflow-menu coverage, add to
`app/src/terminal/view/pane_impl_tests.rs` (sibling test file):

| Test | Guards invariants |
|---|---|
| `overflow_menu_includes_new_session_when_flag_on` | 3 |
| `overflow_menu_excludes_new_session_when_flag_off` | 2, 3 |
| `new_session_action_is_disabled_in_unsupported_states` | 3 (edge cases) |

### Integration tests

Per `.claude/skills/warp-integration-test/SKILL.md`, integration tests
live under `crates/integration/`. Add a new file
`crates/integration/src/test/multi_session_tabs_in_pane.rs` and wire
it into `crates/integration/src/test.rs` and the manual runner
binary. The headline tests:

| Test | Flow | Guards invariants |
|---|---|---|
| `terminal_pane_overflow_new_session_creates_second_tab` | Open terminal pane → click overflow → click "New session" → assert two tabs in strip, second tab active | 3, 4, 5 |
| `agent_pane_overflow_new_session_creates_second_tab` | Same, starting from a pane in fullscreen agent view | 3, 4, 5, 32 |
| `clicking_inactive_tab_switches_active_session` | Two tabs → click first → assert pane body, focus, and pane title all reflect first tab | 13, 14, 16 |
| `closing_active_tab_picks_neighbor` | Three tabs, second active → close second → assert third becomes active | 17, 19 |
| `closing_last_tab_closes_pane` | Single tab → close × → assert pane no longer in pane group | 18 |
| `pane_close_button_summarizes_busy_tabs` | Two tabs, one running a long command → click pane close → assert single confirmation modal mentioning 1 busy tab | 20, 33 |
| `reorder_tabs_within_strip` | Two tabs → drag tab 0 past tab 1 → assert order swapped, active tab follows the drag | 21 |
| `cross_pane_drag_no_op_in_mvp` | Two tabs in pane A → drag tab out of strip → assert tab snaps back, no new pane | 22 |
| `restart_restores_multi_tab_pane` | Use the persistence harness to round-trip the workspace → assert tab order and active tab survive | 23 |
| `restart_collapses_multi_tab_when_flag_off` | Save with flag on → reload with flag off → assert single-tab pane containing the previously-active tab | 24 |
| `inactive_tab_receives_terminal_output` | Run a command in tab 0 → switch to tab 1 → switch back → assert the output landed in tab 0 in real time, not on switch-back | 15 |

### Manual verification

Per `CONTRIBUTING.md`, manual screenshots/recording for the
visual/UX-heavy invariants:

- **Strip layout (8, 10, 11):** screenshots at 1×, 2×, 4× pane widths.
- **Drag-to-reorder feedback (12, 21):** short screen recording.
- **Truncation (11):** narrow the pane until tab titles ellipsize.
- **Notifications on inactive tab (16):** trigger an agent message in
  an inactive tab; screenshot the unread indicator.
- **Maximize / share / split coexistence (26-29):** walk through each
  with a multi-tab pane.

### Regression bar

Run with the flag *off* and re-execute the existing terminal-pane
integration tests under `crates/integration/` to confirm zero
regression. Then re-run with the flag *on* but only one tab created
to confirm the "1 tab == today" invariant.

CI commands per `CONTRIBUTING.md`:
- `cargo fmt`
- `cargo clippy --workspace --all-targets --all-features --tests -- -D warnings`
- `cargo nextest run -p warp_app -E 'test(pane_group::pane::terminal_pane) + test(terminal::view::pane_impl)'`
- `cargo nextest run -p integration -E 'test(multi_session_tabs_in_pane)'`
- `./script/presubmit` before pushing.

## Risks and mitigations

- **Risk: `PaneId` stability regressions.** Switching from
  `PaneId::from_terminal_pane_view(&view)` (keyed off the inner
  `TerminalView`'s `EntityId`) to a pane-keyed id changes which
  `EntityId` is consulted in many call-sites
  (`PaneGroup.pane_contents`, `child_agent_panes`, etc.).
  *Mitigation:* the change is a renaming refactor that keeps the type
  signature; all existing call-sites compile-fail at the rename and
  are updated. The unit test `pane_id_stable_across_tab_switches`
  guards the new invariant. We also keep
  `terminal_view(ctx).id()` available for any caller that genuinely
  needs the inner-view id.
- **Risk: subscription duplication.** Today the pane subscribes once
  to `BlocklistAIHistoryModel`, `Manager`, and `model_event_sender`;
  with multi-tabs we run those subscriptions per tab. Duplicate
  events into the same handler would cause double-processing.
  *Mitigation:* subscriptions are scoped to per-tab `terminal_view_id`
  filters (already the dominant pattern in `handle_ai_history_event`),
  so a tab only reacts to its own view's events. We add a unit test
  that creates two tabs, fires an `AppendedExchange` for tab 0's
  conversation, and asserts only tab 0 reacts.
- **Risk: shared `PaneConfiguration` writes contend.** Today's
  `pane_configuration` is one object; multiple `TerminalView`s
  writing into it would overwrite each other's titles. *Mitigation:*
  each tab owns its own `PaneConfiguration` (per-tab field on
  `TerminalSessionTab`); the *pane-level* `pane_configuration` is a
  thin pass-through that re-aims at the active tab's configuration on
  switch.
- **Risk: persistence schema migration.** A v1→v2 SQLite migration
  would slow down older builds and complicate rollback.
  *Mitigation:* serde-default new fields keep the wire format
  forward- and backward-compatible; no migration is needed.
- **Risk: scope creep into cross-pane drag.** Reorder-only is a fuzzy
  boundary; it's tempting to "just also" support drag to a sibling
  pane. *Mitigation:* the drag handler explicitly cancels with a
  spring-back when the cursor leaves the strip's bounds, and
  invariant 22 has a dedicated regression test.

## Alternatives considered

- **(A — chosen)** Refactor `TerminalPane` to own a tab vector.
  Mechanical refactor, single owner, persistence is local.
  *Trade-off:* `TerminalPane` accumulates tab-management logic.
- **(B)** New `TerminalTabsPane` wrapper that wraps `Vec<TerminalPane>`
  via the `PaneContent` trait. Cleaner separation, but every consumer
  of `PaneId::from_terminal_pane_view` and every match arm on
  `LeafContents` has to learn the new variant; the migration ripples
  through dozens of files in `pane_group/`, `workspace/`, and
  `persistence/`.
- **(C)** Generalize `PaneStack` into a tab group. Reuses existing
  infra, but rewrites `PaneStack`'s semantics (single-visible →
  multi-visible) and breaks the "in nav stack" check at
  `pane_impl.rs:246-250`. Higher risk for less code savings than (A).

We pick (A) because it minimizes API churn for the rest of the
codebase, keeps the persistence schema additive, and lets us iterate
on the strip UI without re-touching every `PaneId` call-site.

## Follow-ups

- **Cross-pane / cross-window tab drag-and-drop.** Out of MVP per
  invariant 22. Likely lands as its own spec once the in-pane
  reorder is stable; will need to coordinate with the pane group's
  drag-to-split logic.
- **Keyboard shortcuts.** `Cmd+Shift+T` for new session in active
  pane, `Cmd+Shift+[/]` for tab switch, `Cmd+W` retargeting to
  active tab, `Cmd+1..9` for direct switch. Each is gated on the same
  feature flag once shipped, with `.with_enabled(|| FeatureFlag::…)`
  per the `add-feature-flag` skill's keybindings guidance.
- **"New agent session" entry.** A submenu split (separate "New
  terminal session" and "New agent session") lands once the MVP UX
  is validated.
- **Unread / busy indicators on inactive tabs.** Spec invariant 16
  describes the surface; the exact mapping from underlying events to
  unread-state needs a follow-up design pass.
- **DOGFOOD_FLAGS / preview / stable rollout.** Use the
  `promote-feature` skill once dogfood feedback is positive.
- **Workspace tab vs pane tab disambiguation.** A small UX research
  task to document the mental model in user-facing docs once the
  feature is enabled in preview.
