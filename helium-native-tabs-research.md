# Chromium's native tab capabilities

Research for [ticket #2](https://github.com/mia-riezebos/helium/issues/2), part of [Native tab panels and nested groups](https://github.com/mia-riezebos/helium/issues/1). Checked on 2026-10-14.

Chromium already supplies collection-backed tab ownership, flat named groups, pins, selection, movement, native controls, and restoration. The targeted source also contains experimental group focus and horizontal swipe rotation over vertical tabs. Helium adds browser shortcut configuration. Recursive named groups and separate tab panels exceed the inspected behavior. The inventory below distinguishes source evidence from inference; it chooses no integration architecture. [Collection contract][collection], [model][model], [focus feature][ui-features], [swipe handler][swipe], [Helium shortcuts][helium-shortcuts].

## Revisions and evidence boundary

| Item | Exact revision and evidence |
| --- | --- |
| Isolated checkout at research start | Helium fork commit `d2ff2d2ad38cba5fbb91f0a63ad691eaabbcc172`. Its direction documents contain the native-reuse decision. [Direction][direction], [terminology][context]. |
| Browser source baseline in that checkout | Helium commit `1fe0fe984484020aae958886adc30618a6ac6dc0`. The checkout's later commits change documentation; `chromium_version.txt`, `downloads.ini`, and `patches/` match this baseline. [Helium baseline][helium-baseline]. |
| Targeted Chromium release | `155.0.8059.79`, as recorded by Helium. [Version target][helium-version]. |
| Targeted Chromium source commit | `a49e2e3d4addfd84e765b52afe230bedb81096db`. The release tag resolves directly to this commit, whose `chrome/VERSION` records the same release. [Tag resolution][chromium-tag], [version source][chromium-version]. |
| Source acquisition | Read revision-pinned Chromium files and the checked-in Helium patch series. Helium downloads a prepared base archive, which was not downloaded or materialized here. [Archive declaration][helium-downloads], [patch order][helium-series]. |

Every Chromium source link below uses the targeted commit. Every Helium patch link uses the browser baseline. No current-main capability is substituted for target-version evidence. The earlier `helium-feasibility.md` supplied leads only; its embedding hypothesis and newer-main citations do not establish capabilities in this inventory.

This is source investigation, not runtime verification. No browser was built or launched. No browser code, shared direction document, or glossary was changed. Source feature defaults establish defaults in the inspected files, not the effective flags of any installed Helium binary.

## Collections and nesting constraints

**Verified.** `TabCollection` is an n-ary ownership tree containing tabs and child collections. Its iterator and recursive indices expose a flattened tab order. Child collections have one parent, and insertion checks each collection's allowed child types. A general tree data structure therefore does not establish arbitrary user-facing nesting. [Collection documentation][tabs-readme], [collection API][collection], [recursive traversal][collection-impl].

The targeted constructors impose these constraints:

| Collection | Direct tabs allowed? | Allowed child collection types | Evidence |
| --- | --- | --- | --- |
| Tab strip root | No | Pinned, unpinned | [Root constructor][strip-collection] |
| Pinned | Yes | Split | [Pinned constructor][pinned-collection] |
| Unpinned | Yes | Group, split | [Unpinned constructor][unpinned-collection] |
| Group | Yes | Split | [Group constructor][group-collection] |
| Split | Yes | None | [Split constructor][split-collection] |

**Verified invariants.** A native group cannot contain another native group. A pinned collection cannot contain groups. An ordinary tab is a leaf; the inspected model has no tabs-under-tabs relationship. The collection type enum has tab strip, pinned, unpinned, group, and split types, with no panel type. Native splits are structural collections, not named recursive groups or tab panels. [Allowed-type check and enum][collection], [constructors][group-collection], [split constructor][split-collection].

**Verified.** `TabStripModel` creates one root collection and one selection state. Existing APIs address tabs through global model indices and a single optional group ID. `TabModel` derives its pinned, group, and split properties from collection ancestry, but exposes one group ID rather than a group ancestry path. [Model construction and API][model], [ancestry-derived properties][tab-model].

**Missing capability.** The inspected contracts do not represent separate panels, panel membership, or recursive named group ownership. **Inference.** Retaining native tab ownership is plausible because ownership is already hierarchical. That observation does not prove that adding recursive groups or panels preserves every existing consumer's assumptions. Global indices, one-group tab metadata, and allowed child types remain material constraints. [Collection contract][collection], [model API][model-header], [tab metadata][tab-model].

## Native groups and colour

**Verified.** Group visual data contains a title, one colour ID, and collapsed state. The editor has title and colour controls, new-tab-in-group, move-to-window, ungroup, and close actions. Save/share controls depend on saved-group services and eligibility. [Visual data][group-visuals], [group editor][group-editor].

The palette has nine IDs: grey, blue, red, yellow, green, pink, purple, cyan, and orange. These IDs are persisted; the source forbids reusing their numeric values. Groups start with the least-used colour from the model's ordering. [Colour contract][group-colours], [group colour selection][group-model].

**Verified invariants.** Live native groups contain a contiguous range of tabs. The live group registry describes groups with at least one tab and requires removal when a group becomes empty. Saved groups are a separate lifecycle, so this statement is about live tab-strip groups. Grouping pinned tabs moves them into an unpinned group; pinning a grouped tab removes its group membership. [Model movement contract][model-header], [live group registry][group-model-header], [group creation and pinning][model].

**Verified.** Native group views already paint colour lines differently by orientation: a vertical line treatment and a horizontal line/active-tab overline. Helium's vertical patch adjusts this existing line drawing. These are group-level visuals. The inspected tab metadata has no independent user-chosen tab colour. [Group line painting][group-line], [Helium line changes][helium-vertical], [tab metadata][tab-model-header].

**Missing capability.** Configurable per-tab colour and a common left-stripe treatment for tabs/groups in both modes are not supplied by those contracts. The existing group palette and line painting cover part of the requested colour behavior. **Inference.** Exact stripe placement and palette flexibility remain presentation requirements, not proof that a separate colour system is necessary. [Group visual data][group-visuals], [line painting][group-line].

## Horizontal and vertical native controls

**Verified.** `BrowserView` owns horizontal and vertical tab-strip regions and switches their initialized controls when orientation changes. Its side panel is a separate child. Helium keeps this mechanism, adds its layout preference and right alignment, and avoids rebuilding the strip for alignment-only changes. [Browser view][browser-view], [Helium layout controller patch][helium-layout], [orientation/alignment patch][helium-vertical].

There are two horizontal control paths at this revision. The default path uses the older `TabStrip` and `BrowserTabStripController`. `TabStripUnification` selects the newer collection-backed horizontal region, but its source default is disabled. Vertical controls use the collection-backed region. The new common node/control code includes views for tabs, groups, splits, and pinned/unpinned containers, with an orientation-aware controller. [Horizontal region factory][horizontal-region], [feature default][tabs-features], [common node/view mapping][node], [collection controller][collection-controller], [vertical region][vertical-region].

**Verified.** Common controls have separate pinned and unpinned scroll containers. Vertical orientation uses vertical scrolling. The new horizontal path enables horizontal scrolling and treats all scroll events as horizontal. This is native tab overflow behavior, not panel switching. [Scroll configuration][strip-view].

**Verified.** The collection module identifies itself as cross-platform. The native control build targets use Views and shared tab components; platform-specific branches remain in interaction code. Source contains Windows, macOS, and Linux gesture handlers. **Runtime limit.** This establishes cross-platform source paths, not successful builds or equivalent behavior on all three platforms. [Module scope][tabs-readme], [common control build target][common-build], [vertical control build target][vertical-build], [gesture branches][swipe].

**Inference.** The targeted checkout has both existing native control paths and a disabled-default unification path. A design inventory must account for this distinction before claiming that horizontal and vertical mode already use identical controls. No choice between those paths is made here. [Horizontal factory][horizontal-region], [unification default][tabs-features].

## Group focus, wheel input, and shared pins

**Verified, experimental.** `TabGroupsFocusing` is disabled by default in the targeted source. It focuses one existing flat group within the same model. The unpinned layout hides other groups and ungrouped tabs; the separate pinned container remains outside that filter. Selection validation accepts tabs in the focused group and pinned tabs. [Feature default][ui-features], [unpinned visibility filter][unpinned-layout], [pinned/unpinned containers][strip-view], [focused selection validity][selection-state].

The feature also has a focus-freezing parameter that defaults to true. This inventory verifies the parameter, not its runtime freezing effects. [Feature parameters][ui-features].

Group rotation follows visual group order. It cycles through groups and an unfocused state. This differs from a list of independent panels. Focusing a group filters the current selection and may activate the group's first tab; it does not establish independent per-panel selections. Creating another group or certain outside-group operations exits focus. Relative tab cycling skips collapsed tabs and tabs outside focus, while permitting pins. [Focus and rotation][model], [selection state][selection-state].

**Verified, experimental.** `VerticalTabStripFocusSwipeController` already consumes horizontal gestures over the vertical strip and rotates group focus. Windows/Linux use mouse-wheel offsets; macOS uses scroll events. It locks the axis, leaves vertical scrolling alone, suppresses rotations while dragging, triggers once per gesture, handles macOS momentum, and reverses direction for RTL. Source constants are a 15-unit axis lock, 75-unit swipe threshold, and 300 ms stream reset. Those are implementation constants, not device-tested distances. [Swipe behavior][swipe], [constants and platform declarations][swipe-header], [feature-gated installation][vertical-region].

**Verified.** The ordinary browser-root wheel handler selects adjacent tabs over a horizontal strip when enabled. It explicitly excludes vertical mode and accumulates both axes. Helium's `scroll-tabs` flag adds always/default/never control to that behavior. It switches tabs, not panels. [Browser-root wheel handling][browser-root], [Helium scroll flag][helium-wheel].

**Verified invariants.** Pins occupy the prefix of the flattened model. Moves retain the pinned/unpinned boundary. Pins are existing tab objects, not copies for different views. Native focused-group behavior already permits pins across group focus changes, but there are no panels in that model. [Pinned ordering contract][model-header], [root pinned container][strip-collection], [focused validity][selection-state].

**Missing capability.** The required separate panels and their horizontal wheel switching do not exist as such. **Inference.** The targeted focus/filter/swipe/pin mechanisms are relevant native precedents. They do not demonstrate persistent independent panels, groups within each panel, or profile-spanning shared pins. Profile feasibility belongs to ticket #4.

## Selection, commands, dragging, and movement

**Verified.** Native selection records an active tab, anchor, selected tabs, and optional focused group. The common controller delegates selection and close operations to the model. Clicking activates tabs; Shift extends selection; Ctrl on Windows/Linux or Command on macOS toggles selection; modifier-plus-Shift adds a range. Middle click closes a tab. [Selection state][selection-header], [controller operations][collection-controller], [mouse handling][tab-view].

Group headers already support collapse/expand through click and Space/Enter, an editor through right click, keyboard reordering, and header dragging. Collapsing the active group selects another expanded tab, or creates a fallback tab when necessary. Collapse hides child controls without deleting their tabs. [Header interactions][group-header], [collapse selection behavior][collection-controller], [child visibility][group-view].

Native command/menu infrastructure includes new tabs, reload, duplicate, pin/unpin, mute, grouping/ungrouping, movement to another window, and close variants. Default accelerators include tab selection/cycling, movement, reopen, and group actions. Platform branches differ. [Tab menus][tab-menu], [browser commands][browser-commands], [accelerator table][accelerators].

Helium also adds a preference for MRU tab cycling and an option to open new tabs next to the active tab. These existing choices affect tab ordering and selection behavior in the target checkout. [MRU option][helium-mru], [new-tab placement][helium-placement].

**Verified.** Movement APIs preserve pin boundaries and group contiguity. They support selected-tab moves, whole-group moves, and detached group/split insertion without destroying those collections. Dragging supports tab selections and group headers, including movement across compatible windows. The drag target check rejects different profiles, windows without tab strips, and incompatible browser/app types. [Movement and detached collections][model-header], [drag controller][drag].

**Verified invariants.** This is movement of flat groups and selected tab/split collections. It does not establish nested-group subtree movement, reparenting between nested groups, or panel transfer. Focus mode also constrains moves into/out of the focused group. [Allowed group children][group-collection], [focused movement restrictions][collection-controller], [model movement][model].

## Configurable keyboard and mouse actions

**Verified Helium addition.** This checkout already supplies customizable browser shortcuts, enabled by default in its patch. `BrowserShortcutService` builds the native command catalogue, assigns/unassigns shortcuts, handles conflicts and locked shortcuts, resets defaults, and persists changes in a profile preference dictionary. Its catalogue includes tab creation, duplicate, pin, close variants, tab/group movement and focus commands, and Helium layout commands. [Shortcut service and catalogue][helium-shortcuts].

The Settings page exposes shortcut recording, removal, filtering, and reset. Browser windows observe changes and reload native accelerators. macOS wiring also updates menu accelerators. [Settings page][helium-shortcuts-page], [window/platform wiring][helium-shortcuts-wiring].

**Missing capability.** New panel and nested-group actions are absent from this existing catalogue. The inspected mouse, header, swipe, and wheel handlers prescribe their own actions. They do not expose a general configurable mouse-action catalogue. **Inference.** The keyboard requirement is partly satisfied by existing Helium functionality; it should not be counted as an entirely new subsystem. Configurable mouse bindings remain a separate gap. [Catalogue][helium-shortcuts], [mouse actions][tab-view], [group actions][group-header], [swipe][swipe].

## Persistence, restoration, and reopening

**Verified.** The native session format records ordered tabs, a selected tab index, pin state, optional group/split IDs, navigation history, and group visual metadata. Group metadata serializes title, colour, collapsed state, and a saved-group GUID. Window/tab extra-data maps also exist. These maps are storage facilities; they establish no panel or recursive-group semantics. [Session records][session-types], [serialization][session-commands].

Session restore relabels group IDs, adds tabs with their stored membership and pin state, then reapplies group visual metadata. It also restores focused-group state when the feature is enabled and vertical strip collapsed state/width from window data. Helium changes restored-tab loading policy through its lazy restore patch. [Restore path][session-restore], [Helium loading changes][helium-lazy-restore].

**Verified.** Recently closed entries have tab, split, group, and window types. Historical tabs retain their original strip index, pins, group/split metadata, and extra data. A group entry contains ordered tabs rather than recursively nested group entries. Reopening a tab searches for its original group's current window; otherwise it falls back to its browser/context, with end-of-strip fallback for invalid indices or specified dispositions. [Closed-entry records][restore-types], [historical capture and reopen behavior][restore-helper].

**Verified caveat.** Saved-group reopening is not always a historical snapshot. When a saved group still exists, `BrowserLiveTabContext` opens, focuses, or moves that saved group. Group/window restoration treats its saved state as authoritative. Restoring an individual tab can then add it to that group. Unsaved/deleted-group paths add restored tabs and recover visual metadata. [Saved-group restoration branches][live-context].

**Missing capability.** The typed records contain no panel membership, parent-group relationship, per-panel active selection, or per-tab user colour. Native persistence therefore does not currently restore the required organization. **Inference.** Existing extra-data maps do not prove that custom state would survive every crash, reopen, saved-group, or version-migration path. Those guarantees remain unresolved. [Session schema][session-types], [closed-entry schema][restore-types], [saved-group branches][live-context].

## Extension-visible behavior and the independent side panel

**Verified.** `chrome.tabs` exposes one window/index, one group ID, pin state, active/highlighted state, and split-view ID per tab. It supplies create/query/update/move/group/ungroup and related events. `chrome.tabGroups` exposes title, palette colour, collapsed state, window ID, and collaboration `shared` state. Group IDs are unique within a browser session. Neither schema represents panels, parent groups, or per-tab user colours. The tab schema has no general `hidden` property or hide/show action. [Tabs schema][tabs-api-schema], [groups schema][groups-api-schema].

**Verified.** Extension queries iterate all tabs in matching windows. The desktop `TabListBridge` exposes the complete model count/order. Queries apply API filters, not tab-strip collapse/focus visibility. Collapsed or focus-hidden tabs therefore remain queryable with matching filters and permissions. Sensitive URL/title/favicon information is scrubbed when permission rules require it. [Query implementation][tabs-api], [desktop tab list][tab-list], [tab serialization and scrubbing][extension-tab].

Group/tab movement APIs maintain native window/index and pin/group rules. Group movement between windows rejects different profiles. Showing one tab in several panel views is not an extension concept; extension-visible identity is still one tab ID and owning window. The `shared` group property describes collaboration, not the requested shared pin. [Movement checks][groups-api], [tab identity][extension-tab], [group schema][groups-api-schema].

**Verified.** `chrome.sidePanel` hosts extension content independently of the tab strip. It supports default and per-tab options, window/tab opening contexts, action-click behavior, alignment information, and open/close events. `open()` requires a user action. `BrowserView` creates the side panel and tab-strip regions separately, and Helium's layout patch accounts for their adjacent geometry. [Side-panel API][side-panel-api], [distinct native children][browser-view], [Helium geometry][helium-vertical].

**Inference.** Existing native tab controls and extension side-panel hosting are distinct systems, which supports the agreed independence requirement. Source structure alone does not prove simultaneous usability in every window width, alignment, or display mode. Existing extensions also have no native vocabulary for future nested-group ancestry or panels; that is an interoperability question for the design, not a decision to expose a new API.

## Requirement coverage

| Required behavior | Verified coverage at this checkout | Remaining gap or limit |
| --- | --- | --- |
| Recursive named groups | Native collections; flat named, coloured, collapsible groups | Group-under-group nesting is prohibited. Tabs remain leaves. [Types][collection], [group constructor][group-collection]. |
| Separate tab panels | Experimental flat-group focus/filter | No panel ownership or membership contract. [Model][model], [focus][unpinned-layout]. |
| Horizontal wheel switches panels | Experimental cross-platform focus swipe in vertical mode; ordinary horizontal tab wheel navigation | Existing gestures rotate groups or select tabs, not panels. [Swipe][swipe], [ordinary wheel][browser-root]. |
| Vertical and horizontal modes | Native regions and Helium layout settings | Two horizontal control paths; unification defaults off. Runtime parity unverified. [Factory][horizontal-region], [flags][tabs-features], [Helium layout][helium-layout]. |
| Configurable URL/domain routing | Existing tab navigation/query/movement operations | No panel destination or configurable routing contract in the inspected model/control/API files. This is a bounded negative finding. [Model API][model-header], [tabs API][tabs-api-schema]. |
| Pins visible in every panel | Native pin prefix; pins remain valid during group focus | Independent panels do not exist. Cross-profile sharing is outside this ticket. [Pin contract][model-header], [focus validity][selection-state]. |
| Configurable colour stripes | Group title/colour state and existing native group lines | No individual user tab colour; requested stripe presentation is not established. [Metadata][group-visuals], [tab metadata][tab-model-header], [painting][group-line]. |
| Configurable keyboard/mouse actions | Helium browser shortcut configuration; existing native commands | Panel/nested-group commands and general mouse remapping are absent from inspected handlers/catalogue. [Shortcuts][helium-shortcuts], [mouse handlers][tab-view]. |
| Persistence and reopening | Native sessions, recently closed entries, flat group visuals, pins, splits, strip state | No typed recursive hierarchy or panel state. Saved-group reopening can use current saved state. [Session schema][session-types], [reopen][live-context]. |
| Windows/macOS/Linux | Shared native model/Views code and explicit platform input paths | No builds or runtime checks in this evidence-only task. [Module][tabs-readme], [handlers][swipe]. |
| Native toolbar and independent extension side panel | Separate native regions, layout code, side-panel API | Interaction and geometry under the new required organization remain unproven. [Browser view][browser-view], [side-panel API][side-panel-api]. |

## Unresolved runtime evidence

1. Effective feature flags and prepared-archive contents in an actual Helium build, especially group focusing and horizontal unification.
2. Gesture delivery across Windows/Linux wheel hardware and macOS mouse/trackpad input, including thresholds and momentum.
3. Dragging, collapse, selection, and side-panel geometry across both horizontal paths and vertical mode, including platform-specific window handling.
4. Crash/restart/reopen ordering, saved-group reconciliation, and extension event ordering under future organization state. Existing extra-data storage is not a durability guarantee.

These are source-to-runtime limits, not failed tests. The report contains no browser implementation proposal.

## Newly specifiable design questions

These follow from verified constraints. The parent can reconcile them with ticket #3's Sidebery evidence before asking product questions.

1. Does a named group survive removal of its last child? Chromium's live group lifecycle removes empty groups. [Live group registry][group-model-header].
2. What selection survives panel switching: one native active/anchor/selection set, or remembered state per panel? Existing focus has one set. [Selection state][selection-header], [focus transition][model].
3. What should existing extensions observe for nested groups while their API exposes one group ID and no parent? [Tab schema][tabs-api-schema], [group schema][groups-api-schema].
4. Should reopening a saved group restore its current saved state or its closed snapshot? Chromium's saved-group branch already chooses the former. [Reopen behavior][live-context].

## Sources

Links beside claims resolve to the exact revisions listed above. The GitHub tag API verifies the release-to-commit mapping.

[direction]: https://github.com/mia-riezebos/helium/blob/d2ff2d2ad38cba5fbb91f0a63ad691eaabbcc172/DIRECTION.md
[context]: https://github.com/mia-riezebos/helium/blob/d2ff2d2ad38cba5fbb91f0a63ad691eaabbcc172/CONTEXT.md
[helium-baseline]: https://github.com/imputnet/helium/commit/1fe0fe984484020aae958886adc30618a6ac6dc0
[helium-version]: https://github.com/imputnet/helium/blob/1fe0fe984484020aae958886adc30618a6ac6dc0/chromium_version.txt
[helium-downloads]: https://github.com/imputnet/helium/blob/1fe0fe984484020aae958886adc30618a6ac6dc0/downloads.ini
[helium-series]: https://github.com/imputnet/helium/blob/1fe0fe984484020aae958886adc30618a6ac6dc0/patches/series
[chromium-tag]: https://api.github.com/repos/chromium/chromium/git/ref/tags/155.0.8059.79
[chromium-version]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/VERSION
[tabs-readme]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/components/tabs/README.md
[collection]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/components/tabs/public/tab_collection.h
[collection-impl]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/components/tabs/impl/tab_collection.cc
[strip-collection]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/components/tabs/impl/tab_strip_collection.cc
[pinned-collection]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/components/tabs/impl/pinned_tab_collection.cc
[unpinned-collection]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/components/tabs/impl/unpinned_tab_collection.cc
[group-collection]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/components/tabs/impl/tab_group_tab_collection.cc
[split-collection]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/components/tabs/impl/split_tab_collection.cc
[model]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/tab_strip_model.cc
[model-header]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/tab_strip_model.h
[tab-model]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/tab_model.cc
[tab-model-header]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/tab_model.h
[group-model]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/tab_group_model.cc
[group-model-header]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/tab_group_model.h
[group-visuals]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/components/tab_groups/tab_group_visual_data.h
[group-colours]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/components/tab_groups/tab_group_color.h
[group-editor]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/tabs/groups/tab_group_editor_bubble_view.cc
[group-line]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/tabs/common/tab_group_line_view.cc
[group-header]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/tabs/common/tab_group_header_view.cc
[group-view]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/tabs/common/tab_group_view.cc
[node]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/tabs/common/tab_collection_node.cc
[collection-controller]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/tabs/common/tab_strip_collection_controller.cc
[strip-view]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/tabs/common/tab_strip_view.cc
[unpinned-layout]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/tabs/common/unpinned_tab_container_view_layout.cc
[tab-view]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/tabs/common/tab_view.cc
[browser-view]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/frame/browser_view.cc
[browser-root]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/frame/browser_root_view.cc
[horizontal-region]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/frame/horizontal_tab_strip_region_view.cc
[vertical-region]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/frame/vertical_tab_strip_region_view.cc
[tabs-features]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/features.cc
[ui-features]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/ui_features.cc
[swipe]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/tabs/vertical/vertical_tab_strip_focus_swipe_controller.cc
[swipe-header]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/tabs/vertical/vertical_tab_strip_focus_swipe_controller.h
[selection-state]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/tab_strip_model_selection_state.cc
[selection-header]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/tab_strip_model_selection_state.h
[tab-menu]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/tab_menu_model.cc
[browser-commands]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/browser_commands.cc
[accelerators]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/accelerator_table.cc
[drag]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/tabs/dragging/tab_drag_controller.cc
[common-build]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/tabs/common/BUILD.gn
[vertical-build]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/tabs/vertical/BUILD.gn
[session-types]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/components/sessions/core/session_types.h
[session-commands]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/components/sessions/core/session_service_commands.cc
[session-restore]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/sessions/session_restore.cc
[restore-types]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/components/sessions/core/tab_restore_types.h
[restore-helper]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/components/sessions/core/tab_restore_service_helper.cc
[live-context]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/browser_live_tab_context.cc
[tabs-api-schema]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/common/extensions/api/tabs.json
[groups-api-schema]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/common/extensions/api/tab_groups.json
[tabs-api]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/extensions/api/tabs/tabs_api.cc
[groups-api]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/extensions/api/tab_groups/tab_groups_api.cc
[tab-list]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/tab_list_bridge.cc
[extension-tab]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/extensions/extension_tab_util.cc
[side-panel-api]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/common/extensions/api/side_panel.idl
[helium-layout]: https://github.com/imputnet/helium/blob/1fe0fe984484020aae958886adc30618a6ac6dc0/patches/helium/ui/layout/core.patch
[helium-vertical]: https://github.com/imputnet/helium/blob/1fe0fe984484020aae958886adc30618a6ac6dc0/patches/helium/ui/layout/vertical.patch
[helium-wheel]: https://github.com/imputnet/helium/blob/1fe0fe984484020aae958886adc30618a6ac6dc0/patches/ungoogled-chromium/add-flag-to-scroll-tabs.patch
[helium-shortcuts]: https://github.com/imputnet/helium/blob/1fe0fe984484020aae958886adc30618a6ac6dc0/patches/helium/core/custom-keyboard-shortcuts.patch
[helium-shortcuts-page]: https://github.com/imputnet/helium/blob/1fe0fe984484020aae958886adc30618a6ac6dc0/patches/helium/settings/custom-keyboard-shortcuts-page.patch
[helium-shortcuts-wiring]: https://github.com/imputnet/helium/blob/1fe0fe984484020aae958886adc30618a6ac6dc0/patches/helium/ui/custom-keyboard-shortcuts-wiring.patch
[helium-lazy-restore]: https://github.com/imputnet/helium/blob/1fe0fe984484020aae958886adc30618a6ac6dc0/patches/helium/core/lazy-session-restore.patch
[helium-mru]: https://github.com/imputnet/helium/blob/1fe0fe984484020aae958886adc30618a6ac6dc0/patches/helium/core/tab-cycling-mru.patch
[helium-placement]: https://github.com/imputnet/helium/blob/1fe0fe984484020aae958886adc30618a6ac6dc0/patches/helium/core/open-new-tabs-next-to-active-tab-option.patch
