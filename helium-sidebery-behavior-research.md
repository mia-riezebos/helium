# Sidebery behavior references for native tab organization

Sidebery provides useful references for panels, nested group tabs, routing, shared pins, and configurable actions. Its groups are real browser tabs that head tab subtrees. They are not independent named collection nodes. Automatic routing and the different reopening paths also have narrower guarantees than "preserve the organization" suggests. [Group creation][group-create], [routing][route-move], [reopening][closed-open]

This report resolves [research ticket #3](https://github.com/mia-riezebos/helium/issues/3) for [map #1](https://github.com/mia-riezebos/helium/issues/1). It inventories behavior, not a browser implementation. Chromium remains the default wherever it already provides the required capability, as specified in [Direction][direction].

## Scope, revisions, and confidence

| Source | Recorded baseline |
| --- | --- |
| Helium checkout | `d2ff2d2ad38cba5fbb91f0a63ad691eaabbcc172`; [Direction][direction] and [terminology][context] read at this revision. |
| Chromium target | `155.0.8059.79`, from this checkout's [version file][chromium-target]. This report makes no Chromium capability claims. Target-version native capability evidence belongs to [ticket #2](https://github.com/mia-riezebos/helium/issues/2). |
| Sidebery | `mbnuqw/sidebery`, branch `v5`, resolved on 2026-10-14 to [`ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18`](https://github.com/mbnuqw/sidebery/commit/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18). Its [manifest][manifest] identifies version 5.6.1 and Firefox minimum 153.0. |

All Sidebery citations below use that immutable commit. Source files were read from a checkout at that SHA. No newer Sidebery or Chromium main behavior is mixed into this baseline. `helium-feasibility.md` supplied leads only; its embedding hypothesis does not guide this report.

Unless marked **Inference**, statements describe source behavior. Source inspection does not establish runtime reliability. No browser was built, launched, or tested. No browser source was changed.

## Panels and the three different kinds of nesting

Sidebery gives each tabs panel an ID, name, colour, icon, move rules, skip-on-switching flag, and optional no-empty behavior. Unpinned tabs have one `panelId`. Panel recalculation partitions the window's tab list into consecutive panel ranges. These are tab collections within a window. [Panel configuration][panel-config], [tab properties][tab-model], [panel ranges][panel-ranges]

| Concept | Sidebery evidence | Meaning for this map |
| --- | --- | --- |
| Tabs under tabs | Ordinary tabs have `parentId`, depth, and folded state. Tree normalization requires a preceding, unpinned parent in the same panel. | This capability is broader than Helium's required groups containing groups/tabs. It is not an additional requirement. [Tree normalization][tree] |
| Group tabs | Grouping creates a browser tab showing Sidebery's internal group page. Selected tabs and their descendants move beneath its tab ID. The group name is encoded in the page URL and can be renamed. | A named grouping behavior reference with real tab/page semantics. [Creation][group-create], [group URL][group-url], [rename][group-name] |
| Recursive named groups | A group tab can have another group tab as parent. The group page explicitly recognizes its group parent and nested subgroup entries. | Evidence of recursive named grouping through group tabs. It does not establish a separate collection-node model. [Nested groups][group-nesting] |

The default tree setting is enabled with a user-facing depth option of `none`. The normalization path still uses an internal maximum depth of 123. "Unlimited recursive groups" would overstate this implementation. [Tree defaults][tree-settings], [depth options][tree-options], [internal depth][tree]

Collapsing a branch hides descendants in Sidebery and activates the parent if a descendant was active. Optional settings also hide native tabs or discard folded descendants. Closing a parent includes descendants according to `rmChildTabs`, whose values are `all`, `folded`, and `none`; the default is `folded`. Group tabs share these tab-tree operations. [Collapse][fold], [close behavior][close-tree], [defaults][tree-settings], [options][tree-options]

**Inference:** Nested group behavior is relevant; treating a group as an activatable web page is not settled by Helium's terminology. Chromium's group behavior and the recursive-group requirement must settle selection, collapse, close, and ungroup semantics. Tabs-under-tabs, group overview pages, auto-folding, and automatic discarding do not follow from the agreed scope.

## Routing and when rules run

| Aspect | Verified behavior |
| --- | --- |
| Configuration | Each panel owns an ordered rule list. Rules have an enabled flag, optional name, URL expression, optional Firefox container, and `topLvlOnly`. There is no creation-only trigger field in the rule schema or editor. [Schema][rule-schema], [editor][rule-editor] |
| URL matching | Plain text is a case-sensitive substring of the complete URL. A string enclosed by `/` becomes a regular expression without flags. Invalid regular expressions fall back to literal substring matching. There is no separate domain/path matcher here. [Compilation][rule-compile], [matching][rule-match] |
| Overlap | Enabled rules are collected in panel order. Rules with both container and URL criteria sort ahead of single-criterion rules. The first matching rule wins; `topLvlOnly` is a filter, not a priority score. [Compilation and ordering][rule-compile], [matching][rule-match] |
| Navigation timing | The changed-URL handler schedules routing after 120 ms for unpinned tabs, excluding `about:blank`. Repeated scheduling replaces that tab's pending timeout. This covers navigation of existing tabs. [URL handler][url-routing], [timer][route-move] |
| Creation timing | New-tab panel selection consults container-only rules. That lookup deliberately skips rules containing URL criteria. General URL routing is attached to URL updates rather than a dedicated creation-only option. [New-tab selection][new-tab-panel], [container-only lookup][rule-match] |
| No matching rule | A panel can specify `moveExcludedTo`. The routing function can move an unmatched tab there, excluding URLs starting with `a` or `m`. It first requires at least one compiled move rule globally. [Fallback destination][route-move] |
| Destination and focus | Routed tabs enter the destination panel at its start or end, tied to the existing `moveNewTabParent` setting. `tabsPanelSwitchActMoveAuto` controls following an active routed tab; its default is true. [Destination][route-move], [default][tab-defaults] |
| Tree effects | Automatic routing queues the matched tab alone, with no destination parent. The generic move path can promote children left behind to that tab's former parent. It does not automatically transfer the whole subtree. [Routing call][route-move], [orphan handling][move-orphans] |

**Inference:** A literal domain string can also match a path or query parameter. Sidebery's substring behavior therefore does not define Helium's requested domain/path semantics. Container criteria are Firefox-specific and are not evidence for profile-per-panel support.

**Inference:** No persistent manual-routing exemption appears in the inspected rule or tab fields. A manual move does not establish immunity to a later matching URL update. Rule edits rebuild the compiled list, but the inspected paths do not establish a whole-window URL rescan after editing. Initial URL-only creation without a subsequent URL event, redirects, restore timing, and queued moves need runtime evidence before promising exact results. [Rule fields][rule-schema], [tab fields][tab-model], [recalculation call sites][rule-refresh], [event source][url-routing]

## Shared pins

Sidebery supports `panel`, `top`, `left`, and `right` pin placement, defaulting to `top`. The non-panel positions render a global bar outside the panel content. That bar reads the same pinned tab IDs for every panel. These are shared views of existing tabs, not one duplicate per panel. [Default][pin-settings], [positions][tree-options], [global placement][sidebar-layout], [bar data][pin-bar], [pinned ID collection][pin-collection]

When a global pin is active, switching panels retains that active tab. For ordinary tabs, panel switching can activate the panel's last active tab. It can also create a tab in an empty destination, depending on `noEmpty`, inactive-tab hiding, and empty-panel hiding. [Switch behavior][panel-switch]

Pinned tabs cannot be tree parents in the normalized tree. The separate `pinnedAutoGroup` option can create or reuse an unpinned group linked to a pin for its opened children; it defaults to false. This is optional Sidebery behavior, outside the shared-pin requirement. [Tree rules][tree], [linked group behavior][pin-group], [default][pin-settings]

**Inference:** Same-window shared pins match the agreed behavior reference. This source provides no answer for shared pins across full Chromium profiles. That question remains with [profile research #4](https://github.com/mia-riezebos/helium/issues/4).

## Colour behavior and the requested stripe

| Layer | Sidebery behavior |
| --- | --- |
| Explicit tab/group-tab colour | Context-menu choices set a persisted `customColor` on selected tab IDs. Group tabs use the same tab field and renderer. The menu also offers clearing the custom colour. [Menu][color-menu], [persistence][colors], [tab/group fields][tab-model] |
| Automatic tab colour | Optional domain-derived or Firefox-container colour. Disabled by default. [Calculation][colors], [defaults][color-defaults] |
| Automatic branch colour | Optional root-URL or root-domain colour propagates through the root branch. Disabled by default. [Calculation][colors], [defaults][color-defaults] |
| Precedence | Explicit custom colour wins, then enabled branch colour, then enabled automatic tab colour. [Renderer][color-render] |
| Inheritance | `inheritCustomColor` defaults to true. New children can copy a parent's colour. Moving beneath a coloured parent can overwrite moved tabs' custom colours. This is stored copying on these paths, not a live inherited style. [Default][color-defaults], [new child][color-new], [move][color-move] |
| Geometry | Custom/automatic colour paints a full-row colour layer. Firefox container colour has a separate narrow indicator on the right of ordinary tabs. These are different visual signals. [Colour layer][color-layer], [container indicator][container-stripe] |

**Inference:** Helium's small left stripe is an explicit project requirement, not a claim about Sidebery's default rendering. The remaining colour question is whether group colour supplies a descendant default, copies over descendant choices, or colours only the group. Automatic domain colouring is an available reference, not required parity.

## Configurable keyboard and mouse actions

Sidebery loads browser commands, validates shortcut syntax and duplicate assignments, and updates bindings through `browser.commands`. Its command handler requires a focused window. This establishes configurability in Firefox, not native shortcut availability on Windows, macOS, and Linux. [Bindings][keys-config], [dispatch][keys-dispatch]

| Relevant action family | Existing command references |
| --- | --- |
| Panel navigation | Next/previous panel, explicit looping variants, direct indexed panel, previous panel. |
| Organization | Move selected tabs to a panel or its start/end, group selection, rename, indent/outdent, move up/down, flatten. |
| Tree and pin interaction | Fold/expand/toggle branch, select branch/children, select parent, new child, pin/unpin. |

These are an inventory for [interaction ticket #7](https://github.com/mia-riezebos/helium/issues/7), not a selected initial catalogue. The actual dispatch distinguishes looping from non-looping panel commands and includes tree-specific actions that Helium has not required. [Command dispatch][keys-dispatch]

Mouse settings are mappings for specific targets and gestures. Tab double-click and long left/right click can map to actions such as reload, pin, mute, duplicate, rename, or child creation. Middle-click has separate normal-tab, pinned-tab, Ctrl, Shift, and close-button choices. Panel background clicks can create a tab, undo close, navigate, or fold/expand, depending on gesture. [Gesture options][mouse-options], [tab handlers][mouse-tab], [panel handlers][mouse-panel]

Defaults include middle-click close for ordinary tabs, middle-click discard for pinned tabs, and no tab double-click action. Panel background double-click and middle-click create a tab. These are reference defaults, not adopted Helium defaults. [Mouse defaults][mouse-defaults]

## Horizontal wheel panel switching

Sidebery's `hScrollAction` offers `switch_panels`, `switch_act_tabs`, or `none`, defaulting to `none`. The panel-content container handles nonzero `deltaX`; positive switches forward and negative backward. Horizontal input temporarily blocks the vertical wheel debouncer. This handler is attached to the panel box, not the entire native browser toolbar. [Default][wheel-settings], [choices][wheel-options], [handler][wheel-handler], [target][sidebar-layout]

`onePanelSwitchPerScroll` restarts the panel-switch pause for continuing input in the same direction. `navSwitchPanelsDelay` defaults to 128 ms. The wheel call skips hidden panels, honors `skipOnSwitching`, and does not request wraparound. Empty/discarded-panel hiding can change eligible targets. [Wheel handler][wheel-handler], [panel switching][panel-cycle], [defaults][wheel-settings]

`wheelThreshold` defaults to false. When enabled, the debouncer uses separate X/Y thresholds and optional accumulation. The defaults are X=10, Y=60, with accumulation enabled. It resets its buffer after 500 ms and passes non-pixel delta modes through. Separately, `navSwitchPanelsWheel` maps vertical scrolling over panel navigation to panel switching. [Debouncer][wheel-debounce], [defaults][wheel-settings], [navigation wheel][nav-wheel]

**Inference:** These controls provide gesture references for Helium's required toolbar wheel behavior. Trackpad inertia, diagonal input, Shift-wheel translation, browser/page navigation interference, and device consistency remain untested. Sidebery's horizontal navigation-bar option is not evidence for horizontal native tabs. Its manifest declares an extension sidebar, and its tabs panel renders a scrolling tab list. [Manifest][manifest], [tab panel][tabs-panel]

## Persistence, restoration, and reopening

| Path | What the inspected implementation retains or attempts |
| --- | --- |
| Panel configuration | Saves sidebar navigation order and panel configurations to extension storage. The active panel ID has separate per-window session storage, excluding private windows. [Save][panel-save], [load][panel-load], [active panel][active-panel-save] |
| Live tab metadata | Session values contain old ID, panel ID, parent ID, folded state, custom title, and custom colour. A separate delayed local cache also records URL and pin information; it skips private windows. [Session save][session-save], [cache][cache-save] |
| Startup/session restore | Tries a matching cache, otherwise reads per-tab session values. Restoration remaps old parent IDs, restores folded/title/colour metadata, and chooses fallbacks for missing panels. Panel order and tree visibility are recalculated. This depends on the browser restoring native tabs. [Restore selection][session-restore], [metadata and fallback][restore-tab] |
| Native reopen closed tab | `undoRmTab` calls Firefox's recently-closed session restore. Creation handling recognizes reopening, checks session data, and attempts the former panel/parent/position. Bulk session restore has a separate metadata-detection and reload path. [Native undo][native-undo], [reopened-tab handling][reopen-native], [bulk restore][restore-bulk] |
| Sidebery multi-tab undo | Notification undo uses captured tab information. It attempts the original index and parent, otherwise the end of the existing parent branch, otherwise the end of the panel. [Undo capture][undo-capture], [position fallback][undo-position] |
| Recently closed subpanel | Stores a reduced URL/title/container/tree record. Opening a tab or branch creates tabs in the currently active panel at its configured new-tab position. That record/open path does not carry custom colour, folded state, or the original panel. This differs from native session reopen. [Reduced record][closed-record], [open destination][closed-open] |
| Snapshots | Capture panel configurations, tab URLs/titles, relative depth, pin state, folded state, and custom title/colour. Reopening adapts panels, reconstructs parent links from depth, and creates windows. This is structural reopening, not demonstrated restoration of live page state. [Capture][snapshot-save], [open][snapshot-open] |

Cache matching compares native URL/pin sequences and tolerates loading blank tabs; it is not simply a stable tab-ID lookup. The source also has explicit recovery fallbacks. **Inference:** These paths show intended recovery behavior, not a guarantee of crash-proof or lossless restoration. [Cache matcher][cache-match], [fallbacks][restore-tab]

## Decisions this evidence makes concrete

These questions belong to the existing decision tickets. They do not authorize additional Sidebery parity.

1. **Group behavior, #6.** What happens when a group is selected, collapsed, removed, or closed with nested groups? Preserve Chromium's applicable behavior; decide only the recursive cases it leaves open. Sidebery's group-page activation and tabs-under-tabs are separate options, not implied requirements.
2. **Routing, #6.** Define domain/path boundaries, overlap priority, and configurable timing. Does routing run on creation, navigation, restore, or rule edits? Does a manual move override it? Does routing a grouped tab detach it, preserve membership, or move a subtree? Sidebery supplies concrete precedents, not a complete answer.
3. **Shared pins and colours, #6.** Choose panel-switch activation while a shared pin is active. Decide whether explicit child colours survive group-colour changes and reparenting. Across-profile shared pins remain bounded feasibility work in #4.
4. **Inputs and horizontal mode, #7.** Select the initial configurable actions and their targets. Decide wheel wraparound, empty-panel eligibility, one-switch-per-gesture behavior, and interaction with tab-strip overflow. The horizontal-mode rendering of recursive groups remains a Helium design question.
5. **Recovery contract, #8.** Distinguish browser restart, crash recovery, native reopen, closed subtree reopen, and missing-parent/panel fallbacks. State whether each restores identity, nesting, colour, collapse, selection, and order. Snapshot export/import and Firefox container behavior are outside the agreed requirement unless explicitly selected.

Decision tickets are [panel/group/routing #6](https://github.com/mia-riezebos/helium/issues/6), [interactions #7](https://github.com/mia-riezebos/helium/issues/7), and [restoration #8](https://github.com/mia-riezebos/helium/issues/8). The preceding sections provide the evidence for each question.

## Remaining runtime uncertainty

This investigation does not validate event ordering during redirects or restore, queued routing after manual moves/pinning, crash durability, duplicate-title reopen matching, or device-specific wheel behavior. Those need later runtime evidence against the selected behavior contract. Sidebery's source does not prove native toolbar integration, independent extension-side-panel coexistence, or cross-platform Helium behavior. These remain native-integration and acceptance questions, not findings to infer from an extension sidebar.

[direction]: https://github.com/mia-riezebos/helium/blob/d2ff2d2ad38cba5fbb91f0a63ad691eaabbcc172/DIRECTION.md
[context]: https://github.com/mia-riezebos/helium/blob/d2ff2d2ad38cba5fbb91f0a63ad691eaabbcc172/CONTEXT.md
[chromium-target]: https://github.com/mia-riezebos/helium/blob/d2ff2d2ad38cba5fbb91f0a63ad691eaabbcc172/chromium_version.txt
[manifest]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/manifest.json#L1-L62
[panel-config]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/defaults/panels.ts#L57-L75
[tab-model]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/types/tabs.ts#L5-L60
[panel-ranges]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/sidebar.fg.ts#L449-L483
[tree]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.ts#L1935-L2011
[group-create]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.groups.ts#L62-L142
[group-url]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/utils.ts#L556-L572
[group-name]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.groups.ts#L378-L391
[group-nesting]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.groups.ts#L177-L218
[tree-settings]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/defaults/settings.ts#L139-L161
[tree-options]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/defaults/settings.ts#L341-L352
[fold]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.ts#L1654-L1706
[close-tree]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.rm.ts#L192-L247
[rule-schema]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/types/sidebar.ts#L237-L252
[rule-editor]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/components/popup.tab-move-rules.vue#L27-L47
[rule-compile]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.move.ts#L742-L803
[rule-match]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.move.ts#L876-L902
[route-move]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.move.ts#L805-L851
[url-routing]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.handlers.ts#L853-L908
[new-tab-panel]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.create.ts#L643-L659
[tab-defaults]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/defaults/settings.ts#L85-L108
[move-orphans]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.move.ts#L196-L208
[rule-refresh]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/sidebar.fg.ts#L1115-L1120
[pin-settings]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/defaults/settings.ts#L131-L137
[sidebar-layout]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/sidebar/sidebar.vue#L69-L122
[pin-bar]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/sidebar/components/bar.pinned-tabs.vue#L25-L29
[pin-collection]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/sidebar.fg.ts#L385-L431
[panel-switch]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/sidebar.fg.ts#L1290-L1321
[pin-group]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.handlers.ts#L283-L292
[color-menu]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/menu.fg.options.tabs.ts#L532-L550
[colors]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.colors.ts#L34-L120
[color-defaults]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/defaults/settings.ts#L163-L168
[color-render]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/sidebar/components/tab.vue#L131-L144
[color-new]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.handlers.ts#L604-L617
[color-move]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.move.ts#L305-L311
[color-layer]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/styles/sidebar/tab.styl#L255-L275
[container-stripe]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/styles/sidebar/tab.styl#L791-L802
[keys-config]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/keybindings.fg.ts#L40-L155
[keys-dispatch]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/keybindings.fg.ts#L169-L282
[mouse-options]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/defaults/settings.ts#L320-L334
[mouse-tab]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/sidebar/components/tab.vue#L418-L443
[mouse-panel]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/sidebar/components/panel.tabs.vue#L83-L208
[mouse-defaults]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/defaults/settings.ts#L250-L272
[wheel-settings]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/defaults/settings.ts#L229-L249
[wheel-options]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/defaults/settings.ts#L317-L318
[wheel-handler]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/sidebar/sidebar.vue#L310-L330
[panel-cycle]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/sidebar.fg.ts#L1367-L1475
[wheel-debounce]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/mouse.fg.ts#L272-L339
[nav-wheel]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/sidebar/components/bar.navigation.vue#L438-L443
[tabs-panel]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/sidebar/components/panel.tabs.vue#L1-L19
[panel-save]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/sidebar-config.fg.ts#L8-L11
[panel-load]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/sidebar-config.ts#L17-L30
[active-panel-save]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/sidebar.fg.ts#L1270-L1279
[session-save]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.ts#L791-L832
[cache-save]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.ts#L725-L753
[session-restore]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.ts#L343-L400
[restore-tab]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.ts#L502-L555
[native-undo]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.rm.ts#L512-L517
[reopen-native]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.handlers.ts#L294-L403
[restore-bulk]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.handlers.ts#L110-L238
[undo-capture]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.rm.ts#L302-L350
[undo-position]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.rm.ts#L392-L460
[closed-record]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.rm.ts#L131-L186
[closed-open]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/sidebar/components/sub-panel.closed-tabs.vue#L196-L225
[snapshot-save]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/snapshots.bg.ts#L33-L138
[snapshot-open]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/snapshots.bg.ts#L399-L480
[cache-match]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.ts#L637-L719
