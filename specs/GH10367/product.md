# Product Spec: Multi-session tabs inside a single terminal/agent pane

**Issue:** [warpdotdev/warp#10367](https://github.com/warpdotdev/warp/issues/10367)
**Figma:** none provided.

## Summary

Today a terminal pane and an agent pane each host exactly one session. A
user who wants two concurrent sessions in the same visual area has to
split the pane or open another workspace tab. The code pane already
solves the same problem with file tabs: a single pane hosts multiple
file tabs, the active tab fills the body, and a small tab strip lives at
the top of the pane.

This change brings the same shape to terminal and agent panes. A
terminal/agent pane can host more than one session as a tab group. When
the pane has two or more session tabs, the existing pane header is
replaced by a tab strip; the active tab's session fills the pane body.
A "New session" overflow-menu entry creates a new tab. Closing a tab
runs through the existing pane-close confirmation logic; closing the
last tab closes the pane.

The whole feature ships behind a feature flag so we can iterate without
disturbing users on the single-session model.

## Problem

The single-session pane is inconsistent with the code pane and forces
users into split panes or extra workspace tabs whenever they want two
related sessions side-by-side in the same visual region. Examples:

- Running a long shell job in one tab and prepping the next command in
  another, without losing screen real estate to a split.
- Comparing two agent conversations on the same problem within a single
  pane, switching between them with one click instead of swapping the
  full pane content.
- Drilling into a child cloud-agent run while keeping the parent
  conversation reachable in a sibling tab of the same pane.

Today users work around this with split panes (which halve the pane
body) or workspace tabs (which leave the pane group entirely). The
overflow menu currently only surfaces share-session and
maximize/minimize entries (`TerminalView::pane_header_overflow_menu_items`,
`app/src/terminal/view/pane_impl.rs:647`); there is no entry point for
spawning another session in the same pane.

## Goals

- A terminal pane and an agent pane can each hold N session tabs.
- The "New session" entry point is available from the pane's overflow
  menu and creates a fresh terminal session as a new tab in the same
  pane.
- The active session fills the pane body. Switching to another tab
  switches the body content with a single click.
- Single-tab panes look and behave exactly like today (no visual or
  behavioral regression for the default case).
- Multi-tab panes restore across app restarts, preserving tab order and
  active-tab selection.
- All existing per-pane features (split, maximize, share session, agent
  view, sync inputs, drag-pane-to-split) continue to work; per-pane
  semantics target the active tab where the choice is ambiguous.
- The whole feature ships behind a feature flag so dogfooding,
  preview, and stable rollout can be staged.

## Non-goals

- Cross-pane and cross-window tab drag-and-drop. MVP only supports
  reordering tabs within the same pane; tearing a tab out into a new
  split, dropping into a sibling pane, or moving between windows is
  out of scope and tracked as a follow-up.
- New keyboard shortcuts for tab switching (`Cmd+1..9`,
  `Cmd+Shift+[/]`) and tab close (`Cmd+W` retargeted to a tab). MVP
  ships only the "New session" command; switching/closing happen via
  click and the overflow menu. Shortcuts are a follow-up so we can
  observe usage first and avoid colliding with workspace-tab
  bindings.
- Spawning an agent session directly from "New session". The default
  new tab is a fresh terminal session; the user enters agent mode
  using the existing in-tab affordances if they want one. (The
  underlying data model already lets a single `TerminalView` swap
  between terminal mode and fullscreen agent mode, so a tab can become
  an agent tab after creation; we just don't expose a separate "New
  agent session" entry point in this MVP.)
- Mixing a notebook / code / settings / network-log pane and a session
  in the same tab group. The tab group is restricted to terminal and
  agent (i.e. `TerminalPane`-backed) sessions only.
- Renaming or otherwise re-skinning the existing per-tab title
  derivation (`TerminalView::update_pane_configuration`). The new tab
  strip reuses the existing title computation as-is.
- Changing how the Maximize/Toggle Pane interaction works at the
  split-pane level. Maximize still toggles whether the host pane
  expands to cover its split siblings; it does not affect the tab
  strip.

## Behavior (testable invariants)

The numbered behaviors below are the contract for this feature. Each
one is exercised by automated and/or manual tests in the corresponding
section of `tech.md`.

### Single-tab compatibility (no regression)

1. **One tab equals today.** When a terminal/agent pane hosts exactly
   one session tab, its visual chrome is identical to today: the
   existing three-column pane header (back button on the left,
   centered title with agent/share indicators, right-side actions and
   overflow). No tab strip is rendered.
2. **Feature flag off equals today.** With the multi-session-tabs
   feature flag disabled, the pane behaves exactly as it does today:
   the overflow menu does not include "New session", and the data
   model still holds a single session per pane. Disabling the flag at
   any point reverts to single-session behavior; multi-tab persisted
   state created while the flag was on is collapsed to its active tab
   on next launch (see invariant 23).

### Spawning a new session

3. **Overflow entry.** With the feature flag enabled, the
   terminal/agent pane's overflow menu (`…`) includes a "New session"
   item. The item is enabled whenever the pane is a terminal/agent
   pane (regardless of shared-session, agent-view, or split-pane
   state) and is hidden for non-terminal pane types (code, notebook,
   settings, etc.).
4. **Spawn behavior.** Selecting "New session" creates a new terminal
   session tab, appends it to the right of the existing tabs, and
   makes it the active tab. The new tab opens in the user's default
   shell, with the same startup directory the workspace would use for
   `Cmd+T` (matching `PaneGroup`'s existing
   `startup_path_for_new_session` resolution).
5. **First spawn promotes the strip.** Spawning a second session in a
   single-tab pane causes the existing pane header to be replaced by
   the tab strip in the same vertical slot (no extra row); both tabs
   are visible, the new tab is active, and the previous tab keeps its
   title and indicators. Any pane-level chrome that was on the right
   side of the header (overflow `…`, close button, share controls)
   remains in the same position, now beside the tab strip.
6. **Idempotent semantics.** Spawning a third, fourth, etc. session
   simply appends another tab; the strip already exists. There is no
   hard-coded upper bound on tab count; behavior is well-defined for
   at least 16 tabs (beyond that, the strip scrolls horizontally; UX
   for very large counts is not in scope here).
7. **Hidden child-agent panes are unaffected.** The hidden pane that
   the orchestrator creates for a child agent
   (`insert_ambient_agent_pane_hidden_for_child_agent`) is still a
   separate single-tab pane and is not folded into a sibling's tab
   group automatically. A user-driven "Open in new tab" of a child
   agent (today emitting
   `Event::OpenChildAgentInNewTab`) continues to land on a workspace
   tab, not on a session tab — workspace tabs and session tabs are
   different surfaces.

### Tab strip presentation

8. **Tab strip layout.** The tab strip occupies the same slot as the
   single-tab pane header. From left to right: an optional back button
   (when applicable, mirroring today's `maybe_render_header_back_button`
   logic for the active tab), the tabs themselves (each rendering the
   active tab's existing title computation: agent icon / share
   indicator / shell indicator + title text + close `×`), and the
   right-side action cluster (share, info, overflow `…`, pane close).
   The total strip height equals today's `PANE_HEADER_HEIGHT` so split
   neighbors do not shift.
9. **Tab content.** Each tab in the strip shows the same primary text
   that the single-tab pane header would show for that session:
   conversation title for an active agent conversation, terminal
   title otherwise, with the same fallback chain
   `selected_cli_agent_title → terminal_title → conversation_title →
   "New agent conversation"` driven by
   `TerminalView::update_pane_configuration`.
10. **Active tab indicator.** The active tab is visually distinct
    (background, accent underline, etc.) using the existing
    design-system theme colors used by the code pane's tab strip; no
    new theme is introduced. Inactive tabs render with the same sub
    text/secondary background tokens already used by the code pane.
11. **Truncation.** When the strip is narrower than the sum of its
    tabs' minimum widths, tabs ellipsize their titles using the same
    `ClipConfig::ellipsis()` policy used in the single-tab header. The
    active tab gets first claim on width, then tabs to its left and
    right share the remainder.
12. **Drag indicator.** Pane drag-to-split (which today targets the
    pane header) continues to work from the tab strip's empty space
    but must not start a drag when the cursor begins on a tab body or
    on the close `×` (those handle tab interactions instead).

### Switching tabs

13. **Click switches.** Left-clicking a non-active tab makes it the
    active tab. Switching activates the tab's `TerminalView` (focus
    and presence), calls
    `TerminalView::on_pane_state_change`, and updates the pane-level
    `PaneConfiguration` so any consumers that read the pane title
    (window title, persistence, etc.) see the new active tab's title.
14. **Focus follows tab.** Switching tabs moves keyboard focus to the
    newly-active tab's input area (terminal grid, agent message bar,
    etc.) — same as the existing `redetermine_global_focus` behavior
    when a single-tab pane gains focus.
15. **No background work loss.** Inactive tabs continue to receive
    their backing terminal/agent events (output, conversation
    streaming, share-session updates, sync-input). Only the rendered
    surface changes when a tab becomes/leaves active. Long-running
    commands and streaming agent exchanges in inactive tabs are not
    paused, throttled, or torn down.
16. **Active tab drives pane indicators.** Notification badges, the
    pane's contribution to the workspace tab title, and the
    "active session" highlight in the workspace's session navigator
    all reflect the active tab. Inactive tabs surface their
    notifications via the tab itself (e.g. an unread dot on the tab),
    using the same notification model as today's per-pane unread state.

### Closing tabs

17. **Per-tab close button.** Each tab in the strip has a close `×`.
    Clicking it requests close of that specific tab, going through
    the same confirmation path as
    `Event::CloseRequested → close_pane_with_confirmation` does today
    for a single-tab pane, including the "running command / busy agent"
    confirmation modal when applicable. Cancelling the modal leaves
    the tab open.
18. **Last tab closes the pane.** Closing the last remaining tab
    closes the entire pane, identically to closing a single-tab pane
    today. There is no "empty pane with a New session button" state.
19. **Active tab follows close.** When the active tab is closed,
    focus moves to the tab to the right of the closed tab; if the
    closed tab was rightmost, focus moves to the new rightmost tab.
20. **Pane-level close still closes the pane.** The pane header's
    right-side close button (visible inside a split pane) still
    closes the entire pane, not just the active tab. If any tab is
    busy, a single confirmation summarizes affected tabs (e.g.
    "Close pane? 2 tabs have running commands.") rather than
    cascading per-tab modals.

### Reordering tabs

21. **Drag within strip.** Dragging a tab horizontally within the
    same strip reorders it; the dragged tab follows the cursor, the
    other tabs slide to make room, and dropping commits the new
    order. There is no animation requirement beyond what the design
    system already provides for code-pane tabs.
22. **No cross-pane drag in MVP.** Dropping outside the strip's
    bounds cancels the reorder (no cross-pane move, no detach to a
    new split). Cross-pane drag is captured as a follow-up, not part
    of this MVP.

### Persistence

23. **Multi-tab restore.** When session restoration is enabled
    (`GeneralSettings.restore_session` and
    `AppExecutionMode::can_save_session()`), reopening the workspace
    restores every session tab, in the same order, with the same
    active tab highlighted. Each tab's content is restored using the
    same per-session machinery used today (cwd, shell launch data,
    input config, conversation IDs, agent view active conversation —
    see `TerminalPaneSnapshot` in
    `app/src/app_state.rs:193`), one snapshot per tab.
24. **Backward-compatible snapshot format.** Persisted state written
    by older builds (a single `TerminalPaneSnapshot` per pane) is
    still loadable: it restores into a single-tab pane. Persisted
    state written by this build is forward-compatible in the sense
    that *if* the feature flag is later disabled, the launcher
    collapses the persisted multi-tab pane to a single-tab pane
    containing whichever tab was marked active. (No silent data loss
    for terminal output: the inactive tabs' shell histories live in
    the existing per-session block store; collapsing only drops the
    *tab membership* metadata.)
25. **No new SQLite migration on disable.** Disabling the feature
    flag does not require a SQLite migration; the new tab-membership
    fields are read opportunistically and, when absent, default to a
    single tab.

### Interaction with existing per-pane features

26. **Split panes.** Splitting a multi-tab pane (today's
    `Direction::{Left,Right,Up,Down}` split actions) creates a fresh
    sibling pane with a single empty session, exactly like splitting
    a single-tab pane today. The original pane keeps its tabs.
27. **Maximize.** "Toggle Maximize Pane" applies to the pane as a
    whole (not the active tab), keeping today's semantics. The tab
    strip is visible in the maximized state.
28. **Share session.** Each tab can independently be a sharer or
    viewer; the share controls in the right-side action cluster
    follow the active tab. Tabs that are sharers or viewers display
    a small share indicator next to their title in the strip.
29. **Sync inputs.** When a pane group is in synced-input mode, the
    sync target is the active tab of each pane (mirroring the current
    behavior where the active session is the sync target). Inactive
    tabs do not receive synced input.
30. **Workspace tabs unaffected.** Workspace-level tabs and their
    keyboard shortcuts (`Cmd+T`, `Cmd+1..9`, `Cmd+Shift+[/]`) are not
    repurposed. They still create/switch workspace tabs; pane-level
    session tabs are a different surface.

### Error and edge states

31. **Spawn failure.** If "New session" cannot create a terminal
    (e.g. the underlying shell launch fails), the pane shows the same
    error chrome it does today for a failed shell session, in the new
    tab. The other tabs are unaffected and the strip remains visible.
32. **Empty conversation tab.** A tab whose `TerminalView` is in
    agent mode with an empty conversation displays the default
    "New agent conversation" / "New cloud agent" title in its
    strip entry, matching the single-tab header fallback.
33. **Error confirmation race.** If a tab is closed while its
    confirmation modal is already open (e.g. via the pane's overall
    close button), the modal is dismissed and the pane proceeds with
    a single combined confirmation — never two stacked modals for the
    same close intent.
34. **Accessibility.** Each tab is focusable via keyboard navigation
    of the pane header (consistent with how the code pane's tabs are
    navigated). Screen readers announce the tab as
    "tab N of M, <title>, active/inactive". The close `×` has its
    own accessible label (e.g. "Close session tab <title>").

## Validation

The specifics live in `tech.md` under `Testing and validation`. At a
glance:

- **Unit tests** for the new `TerminalPane` tab-group operations
  (insert, switch, close, reorder, restore-from-snapshot) and for
  `pane_header_overflow_menu_items` including the new "New session"
  entry.
- **Integration tests** under `crates/integration/` for the headline
  flows (1, 3, 4, 13, 17, 18, 21, 23) following the
  `warp-integration-test` skill: open pane, click "New session",
  assert two tabs, switch, close, reorder, persist + reload.
- **Manual verification** for the visual/UX-heavy invariants (8, 10,
  11, 12, 21) using `./script/run` per `CONTRIBUTING.md`'s manual
  testing requirement.
- **Regression bar** for invariants 1, 2, 26–30: full single-tab
  walk-through with the flag off, then with the flag on but only one
  tab created, then with maximize / share / split flows on a
  multi-tab pane.

## Open questions

- Should the "New session" entry have a default keyboard shortcut now
  (e.g. `Cmd+Shift+T` scoped to the focused pane) or stay
  click-/menu-only until usage data informs the choice? Current spec
  defers it.
- For agent panes, should "New session" optionally offer "New agent
  conversation" alongside "New terminal session" (a small two-item
  submenu)? Current spec keeps it to a single entry that always
  spawns a terminal tab; user converts to agent mode in-tab. Worth
  revisiting after dogfood.
- Notification surface for inactive tabs: an unread dot is the
  obvious choice, but the exact set of events that mark a tab unread
  (new agent message, command finished, error, etc.) deserves its
  own pass before unhide.
