# Native interaction controls for ticket #7

Research for [Choose keyboard, mouse and horizontal-tab interactions](https://github.com/mia-riezebos/helium/issues/7), checked on 2026-10-15. Source inspection only; no bindings, interaction policy or implementation design selected.

## Scope and exact revisions

| Source | Revision |
| --- | --- |
| Inherited planning | `455a05f71d6daa3eaa2f4ed84cdebc8c4e52caf9`. Read [DIRECTION.md][direction] and [CONTEXT.md][context]. |
| Helium browser baseline | `1fe0fe984484020aae958886adc30618a6ac6dc0`. Local browser patches match this revision. [Version target](https://github.com/imputnet/helium/blob/1fe0fe984484020aae958886adc30618a6ac6dc0/chromium_version.txt), [patch order](https://github.com/imputnet/helium/blob/1fe0fe984484020aae958886adc30618a6ac6dc0/patches/series). |
| Chromium | `155.0.8059.79`, commit `a49e2e3d4addfd84e765b52afe230bedb81096db`. [Version source][version]. |
| Sidebery | `ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18`. All Sidebery references below use this commit. |

The completed [native inventory][native-inventory] and [Sidebery inventory][sidebery-inventory] supplied leads. This report extends their input/presentation gaps using owning source files. No browser build, launch, implementation, upstream contribution or issue edit occurred.
The [panel behavior resolution][behavior] settles both orientations, left colour stripes, recursive named groups, shared pins and selection/subtree semantics. They are requirements, not open choices.
Feature defaults below describe source configuration, not an installed binary.

## Keyboard configuration and action families

`BrowserShortcutService` stores profile overrides in `helium.browser.custom_accelerators`. It supports several accelerators per command, assignment/removal and individual/global resets. `CustomizableBrowserShortcuts` defaults on. [Service and preference][shortcuts].
Settings records a non-modifier key with Shift/Control/Alt/Meta, filters commands and reports conflicts. Browser windows observe changes and reload native accelerators; macOS also updates eligible menu items. [Settings][shortcut-settings], [window wiring][shortcut-wiring], [serialization][shortcuts].
**Inference:** this existing command-ID configuration/dispatch mechanism can carry additional native actions without an independent keyboard binding system. It does not implement unknown actions; assignment rejects unknown command IDs. [Catalogue and assignment][shortcuts].

| Action family | Existing native command or control | Actual remaining gap |
| --- | --- | --- |
| Tab activation | `IDC_SELECT_NEXT/PREVIOUS_TAB`, `IDC_CYCLE_TO_NEXT/PREV_TAB`, numbered/last-tab commands. | Panel-relative activation is not a current command semantic. [Catalogue][shortcuts], [dispatch][commands]. |
| Tab lifecycle | New, duplicate, reopen, close/close-other/close-right, pin and mute. Several are assignable without defaults. | Panel membership is absent; shared-pin behavior follows the settled decision. [Catalogue][shortcuts], [behavior][behavior]. |
| Tab movement | `IDC_MOVE_TAB_NEXT/PREVIOUS`; move-to-new-window uses the selected indices. Native model/drag APIs move selected tabs into a flat group. | Move selected tabs to another panel has no native command. Relative movement moves the active tab or split, with an explicit multi-selection TODO. [Commands][commands], [relative move][relative-move], [selected move][selected-move]. |
| Flat group organization | Create group, new tab in current group, close current group, next/previous group commands; editor naming, colour and ungroup actions; keyboard-focused header collapse/reorder controls. | Not every editor/header action is in Helium's shortcut catalogue. There is no recursive group reparent/indent/outdent or ancestry-navigation command family. [Catalogue][shortcuts], [group commands][group-commands], [editor][group-editor], [header][header], [group contract][group-contract]. |
| Panel organization/navigation | No tab-organization panel commands in the inspected catalogue. Browser side-panel commands are unrelated. | Next/previous/direct/previously-viewed panel, panel lifecycle and panel transfer are absent capabilities. This lists gaps, not selected first-release actions. [Catalogue][shortcuts], [terminology][context]. |
| Browser layout/focus | Helium classic/compact/vertical layouts, vertical-strip collapse and browser pane/toolbar focus. | They do not navigate organizational panels or recursive ancestry. [Catalogue][shortcuts], [layout shortcut][layout-shortcut]. |

Create-new-group creates a **new tab**, then groups it. New-tab-in-group uses the active tab's group; close-group closes that group. Next/previous-group normally activates another group's tab, with wrapping search. Existing focus mode changes that behavior. None is a panel command. [Group command implementations][group-commands].
Native selection already supplies Shift ranges, Ctrl (Windows/Linux) or Command (macOS) toggles, modifier-plus-Shift additive ranges, selected-tab dragging and middle-click close. These handlers supply defaults, not a configurable mouse catalogue. [Tab input][tab-input], [drag handler][drag-handler].
Sidebery fills the missing action-family reference: ordinary versus looping panel navigation, direct/previous panel, selected-tab panel transfer, branch fold/expand and indent/outdent. Its branches are tabs-under-tabs, not this project's named-group model. No parity requirement follows. [Dispatch catalogue][sidebery-keys], [terminology][context].

### Existing defaults and conflicts, without choosing bindings

- Existing browser defaults include Ctrl+Tab/Ctrl+Shift+Tab, numbered tabs, reopen and Ctrl+Shift+PageUp/PageDown movement. Mac also has Command-based tab selection. These remain distinct command families. [Native table][accelerators], [Mac defaults][mac-keys], [Mac menu defaults][mac-menu].
- Windows/Linux group defaults use Alt+Shift+C/P/X/Z/W for new-tab-in-group/create/next/previous/close. Mac's non-menu defaults use Command+Control with those letters. [Native group defaults][accelerators], [Mac group defaults][mac-keys].
- Assigning a modifiable conflict removes the old command's binding; locked conflicts reject assignment. Mac non-menu defaults are locked, including group/tab-selection defaults. Dynamic Command+W/Command+Shift+W close bindings and user-remapped App Shortcuts retain platform ownership. Additional bindings are possible for exposed commands. [Conflict/lock and Mac ownership][shortcuts], [Settings feedback][shortcut-settings].
- Contextual existing bindings matter: Control/Command+S toggles vertical-strip collapse when its default-on preference and vertical layout apply; otherwise it saves. Control/Command+Shift+C defaults to URL copying through a default-on preference, with inspection as the alternate action. [Layout wrapper][layout-shortcut], [copy/inspect wrapper][copy-shortcut].
- Helium's MRU preference defaults false. Its patch routes `SelectNextTab/SelectPreviousTab` through that preference, beyond the setting's Ctrl+Tab wording. Browser conflicts are checked by this service; it is not proof against OS interception or extension/content dispatch conflicts. [MRU patch][mru], [conflict scope][shortcuts], [registration][shortcut-wiring].

## Wheel, horizontal gesture and overflow arbitration

| Native path | Configuration | Existing behavior |
| --- | --- | --- |
| Vertical collection controls | Separate pinned/unpinned vertical scroll views. Group-focus swipe installs only with `TabGroupsFocusing`, default off. | Vertical overflow remains native; horizontal swipe rotates **flat group focus**. Browser-root wheel tab switching explicitly excludes vertical mode. [Scroll setup][strip-scroll], [installation][vertical-region], [feature][focus-feature], [root][root-wheel]. |
| Legacy horizontal controls | Default horizontal path because `TabStripUnification` defaults off. Helium `scroll-tabs=always/never` overrides the platform default. | The inspected old region directly hosts `TabStrip`, with width compression/clipping, not a scroll container. Root wheel selects tabs when enabled. [Factory/old region][horizontal-region], [feature default][unification], [legacy visibility][legacy-container], [flag][wheel-flag]. |
| Collection-backed horizontal controls | `TabStripUnification` enabled; horizontal scrollbars hidden but enabled; vertical scrollbars disabled. | Both wheel axes feed horizontal overflow. Pinned/unpinned containers scroll separately; unpinned pagination buttons depend on overflow and their pin preference, default true. No horizontal panel/group-focus swipe installs here. [Setup][strip-scroll], [layout][strip-layout], [button preference][scroll-prefs], [horizontal region][horizontal-region]. |

### Vertical swipe details

The pre-target handler accepts mouse-wheel offsets on Windows/Linux and scroll events on macOS. It checks the focus feature and suppresses swiping during drag. It has no Shift/Control/Alt/Command filter or remapping. [Platform handlers][swipe].
It accumulates both axes. At 15 units, the strictly larger absolute cumulative axis locks; exact ties remain undecided. A vertical lock passes all subsequent input through. A horizontal lock consumes input, even before the 75-unit rotation threshold. [Arbitration][swipe], [constants][swipe-constants].
It rotates once per stream, using negative X as forward and reversing for RTL. A gap over 300 ms resets the stream. Windows/Linux wheel handling has no momentum-phase classification. Mac consumes horizontal momentum without rotating and resets on momentum end, fling start/cancel or supported scroll end/begin phases. [Swipe state][swipe].
**Source limit:** macOS event conversion still conflates some physical phases into momentum phases; `ScrollEvent` construction leaves scroll phase unset on that path. The handler's phase branches do not establish every device's physical-gesture boundary. [Mac conversion][mac-events], [event construction][event-construction].
Rotation uses visual flat-group order and an unfocused state: after the final group comes unfocused, then the first group. Reverse rotation mirrors it; no groups means no change. This is neither panel-edge behavior nor direct final-to-first group wrapping. [Rotation][focus-rotation].

### Horizontal tab-wheel and overflow details

Root wheel defaults on for Linux and off for Windows/macOS; `scroll-tabs` adds always/default/never. It applies across horizontal paths over the strip or caption/top hit regions. [Platform defaults][browser-defaults], [Helium flag][wheel-flag], [root handler][root-wheel].
Root handling rounds each accumulated axis by `MouseWheelEvent::kWheelDelta`, then sums the whole steps. Opposing diagonals can cancel; there is no dominant-axis lock, modifier filter, one-per-gesture latch or momentum filter. One event triggers at most one tab-selection call; mouse exit clears remainders. [Root algorithm][root-wheel].
With default non-MRU selection, its positional guard prevents wrapping and skips collapsed tabs. At a boundary or below a switching threshold it falls through. Helium's enabled MRU path can choose a different destination while retaining that positional guard; do not promise spatial adjacency under that preference. [Root guard][root-wheel], [MRU override][mru], [MRU ordering][mru-implementation].
For mouse-wheel dispatch, the browser-root override runs before child wheel dispatch. Therefore enabled root tab switching can consume input before horizontal overflow; only its fallback reaches the child. Unhandled scroll events convert to wheel events without axis remapping. [Root override][root-wheel], [child dispatch][root-dispatch], [scroll fallback][widget-scroll].
Collection overflow chooses the larger absolute X/Y **for each event**, without a stream axis lock. Wheel/nonordinal ties prefer X; the scroll-event ordinal call reverses its arguments, so compositor updates prefer Y on ordinal ties. Modifier flags pass through unchanged. Conversion preserves momentum, and the compositor receives inertial-phase metadata without the swipe controller's suppression. [Scroll conversion][scroll-view], [compositor input][compositor].
Overflow positions clamp without wrapping. Wheel scrollbar handling reports handled even when already at the boundary; compositor input also returns handled when its handler exists. Do not infer boundary handoff to another action. Touchscreen gesture handling explicitly does not remap vertical gestures into horizontal scrolling. [Scrollbar][scrollbar], [scroll view][scroll-view], [compositor][compositor].
Native unpinned pagination buttons scroll by a page minus a minimum tab width, clamp to bounds and animate. They depend on overflow plus `tab_scroll_buttons.pinned_to_tabstrip`; this is an existing overflow control, not a panel-switch command. [Buttons][scroll-buttons], [layout][strip-layout], [preference][scroll-prefs].

### Does Shift+wheel already provide horizontal input?

There is **no uniform Shift+wheel-to-X conversion in the inspected native toolbar path**. Windows maps `WM_MOUSEWHEEL` to Y and `WM_MOUSEHWHEEL` to X; X11 maps vertical/horizontal button or valuator axes; Wayland retains supplied axes and modifier flags. These converters do not swap axes for Shift. [Windows][win-events], [X11][x-events], [Wayland][wayland-events].
macOS reads AppKit's supplied X/Y deltas, including precise devices. An OS/device-supplied horizontal delta can reach the handler, but this source does not guarantee Shift supplies it. Native wheel/scroll construction and the swipe handler do not add a swap. [AppKit input][mac-events], [event construction][event-construction], [swipe][swipe].
Collection-backed horizontal overflow accepts **ordinary Y wheel input too**, regardless of Shift, because it explicitly treats both axes as horizontal. That is existing overflow behavior, not evidence for a cross-platform modified panel gesture. [Scroll setup][strip-scroll], [conversion][scroll-view].

### Sidebery reference and the actual ambiguity

Sidebery's sidebar uses nonzero `deltaX`, without a modifier or dominant-axis test. `hScrollAction` defaults `none`; `switch_panels` is optional. Wheel thresholds default off, one-switch-per-scroll defaults false, and panel debounce defaults 128 ms. This is configurable reference behavior, not native Helium behavior. [Sidebar wheel][sidebery-wheel], [settings][sidebery-settings], [threshold helper][sidebery-debounce].
Its wheel panel path ignores hidden panels, does not request looping, and respects `skipOnSwitching`. Separate keyboard actions explicitly request looping. It uses debounce/optional accumulation rather than native phase-aware momentum suppression. [Panel traversal][sidebery-panels], [keyboard dispatch][sidebery-keys], [threshold helper][sidebery-debounce].
Required horizontal toolbar scrolling must switch panels, while overflow stays reachable. The actual conflict is that collection-backed horizontal overflow already consumes that same stream, and enabled root tab-wheel selection can precede it. Native sources settle neither ownership nor panel gesture cadence. [Direction][direction], [root][root-wheel], [overflow][scroll-view].

## Horizontal inline group headers, accessibility and drops

`TabCollectionNode` creates `TabGroupView`; the view passes orientation into its header, layout, drag axes and animation axes. Its controller accepts orientation and installs a horizontal closing helper. This supplies an existing horizontal group reference, not recursive support. [Node factory][node], [group view][group-view], [controller](https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/tabs/common/tab_strip_collection_controller.cc#L61-L77).
Horizontal layout places one header chip before the group's direct children in a single standard-height row. It accounts for tab/header overlap and width allocation; collapsed groups occupy header width only. Vertical layout places children below the header with an inset and group line. [Orientation layouts][group-layout].
Horizontal title width is capped relative to a standard tab width. The collapse icon and separate editor button exist only vertically. Horizontal headers still use click/Space/Enter for collapse and right-click for the editor, except focused-group restrictions. These controls do not show an ancestor path. [Header sizing/input][header].
**Keyboard constraint:** the common header requests `GetReorderCommandForKeyboardEvent(...kVertical)` even in horizontal mode. That means Control/Command+Up/Down, with additional Mac Shift+Home/End handling. The legacy horizontal header instead handles Control/Command+Left/Right. Do not claim the common header already has orientation-matched reorder keys. [Common handler][header], [key utility][reorder-utility], [legacy handler][legacy-header].
The common header has accessible-only focus, a focus ring, editable `kTabList` role, expanded/collapsed state and a name from title/content/state. Windows relies on node collapse state; other platforms also include status text. Hover-card accessibility can replace the naming path. No level/ancestor-path semantics appear in these inspected handlers. [Header accessibility][header].
The collection controller reorders a group as a flat tab range, skips whole neighboring groups and refuses pin-boundary/focused-group moves. Group collections allow tabs and splits, not groups. Recursive ancestry presentation, subtree focus traversal and ancestry-aware accessibility therefore remain unsupported, rather than native behavior to assume. [Controller][group-reorder], [group contract][group-contract].
For drag/drop, native collection targets supply a container range, flat insertion index and optional group ID. Tabs can enter an existing flat group. A dragged header calls `MoveGroupTo` and does **not** apply the target group as a parent. Nested header drop/reparent semantics are absent. [Target resolution and movement][drag-handler].

## Choices that still belong to Mia and the parent

1. Which missing panel/recursive-group action families and mouse triggers enter the first release. Existing native selection and group operations remain the baseline.
2. How required panel gestures share horizontal toolbar input with reachable overflow and the optional native tab-wheel action, including modifiers and diagonal ownership.
3. Panel gesture cadence and boundaries where native swipe and Sidebery references differ: one or repeated switch, momentum treatment and end behavior.
4. How horizontal inline headers communicate recursive ancestry, including focused navigation, accessibility and distinguishing reorder targets from nesting targets.

[direction]: https://github.com/mia-riezebos/helium/blob/455a05f71d6daa3eaa2f4ed84cdebc8c4e52caf9/DIRECTION.md
[context]: https://github.com/mia-riezebos/helium/blob/455a05f71d6daa3eaa2f4ed84cdebc8c4e52caf9/CONTEXT.md
[native-inventory]: https://github.com/mia-riezebos/helium/blob/bbe2d36f3d4d5f3333657aee2ce842d3646a76da/helium-native-tabs-research.md
[sidebery-inventory]: https://github.com/mia-riezebos/helium/blob/107047b817835dbabc7e77f2b83cf6fa177850c1/helium-sidebery-behavior-research.md
[behavior]: https://github.com/mia-riezebos/helium/issues/6#issuecomment-6179294449
[version]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/VERSION
[shortcuts]: https://github.com/imputnet/helium/blob/1fe0fe984484020aae958886adc30618a6ac6dc0/patches/helium/core/custom-keyboard-shortcuts.patch
[shortcut-settings]: https://github.com/imputnet/helium/blob/1fe0fe984484020aae958886adc30618a6ac6dc0/patches/helium/settings/custom-keyboard-shortcuts-page.patch
[shortcut-wiring]: https://github.com/imputnet/helium/blob/1fe0fe984484020aae958886adc30618a6ac6dc0/patches/helium/ui/custom-keyboard-shortcuts-wiring.patch
[commands]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/browser_commands.cc#L1555-L1800
[relative-move]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/tab_strip_model.cc#L4230-L4324
[selected-move]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/tab_strip_model.cc#L1170-L1195
[group-commands]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/browser_commands.cc#L1824-L1940
[tab-input]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/tabs/common/tab_view.cc#L600-L653
[accelerators]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/accelerator_table.cc
[mac-keys]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/global_keyboard_shortcuts_mac.mm#L103-L166
[mac-menu]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/cocoa/accelerators_cocoa.mm#L65-L110
[layout-shortcut]: https://github.com/imputnet/helium/blob/1fe0fe984484020aae958886adc30618a6ac6dc0/patches/helium/ui/layout/shortcuts.patch
[copy-shortcut]: https://github.com/imputnet/helium/blob/1fe0fe984484020aae958886adc30618a6ac6dc0/patches/helium/core/keyboard-shortcuts.patch
[mru]: https://github.com/imputnet/helium/blob/1fe0fe984484020aae958886adc30618a6ac6dc0/patches/helium/core/tab-cycling-mru.patch
[mru-implementation]: https://github.com/imputnet/helium/blob/1fe0fe984484020aae958886adc30618a6ac6dc0/patches/brave/tab-cycling-mru-impl.patch#L42-L86
[horizontal-region]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/frame/horizontal_tab_strip_region_view.cc
[unification]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/features.cc#L91
[vertical-region]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/frame/vertical_tab_strip_region_view.cc#L185-L196
[focus-feature]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/ui_features.cc#L554-L557
[strip-scroll]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/tabs/common/tab_strip_view.cc#L578-L613
[strip-layout]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/tabs/common/tab_strip_view_layout.cc#L150-L201
[scroll-prefs]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/tab_strip_prefs.cc#L21-L28
[legacy-container]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/tabs/tab_container_impl.cc
[swipe]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/tabs/vertical/vertical_tab_strip_focus_swipe_controller.cc#L28-L154
[swipe-constants]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/tabs/vertical/vertical_tab_strip_focus_swipe_controller.h#L17-L37
[focus-rotation]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/tab_strip_model.cc#L326-L366
[browser-defaults]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/defaults.cc#L35-L39
[wheel-flag]: https://github.com/imputnet/helium/blob/1fe0fe984484020aae958886adc30618a6ac6dc0/patches/ungoogled-chromium/add-flag-to-scroll-tabs.patch
[root-wheel]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/frame/browser_root_view.cc#L323-L402
[root-dispatch]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/ui/views/widget/root_view.cc#L688-L698
[widget-scroll]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/ui/views/widget/widget.cc#L2442-L2452
[scroll-view]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/ui/views/controls/scroll_view.cc
[scrollbar]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/ui/views/controls/scrollbar/scroll_bar.cc
[compositor]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/ui/compositor/overscroll/scroll_input_handler.cc#L18-L87
[scroll-buttons]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/tabs/horizontal/tab_scroll_button_container.cc#L267-L323
[win-events]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/ui/events/win/events_win_utils.cc#L402-L408
[x-events]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/ui/events/x/events_x_utils.cc#L648-L680
[wayland-events]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/ui/ozone/platform/wayland/host/wayland_event_source.cc#L1188-L1257
[mac-events]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/ui/events/cocoa/events_mac.mm
[event-construction]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/ui/events/event.cc
[sidebery-keys]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/keybindings.fg.ts#L177-L266
[sidebery-wheel]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/sidebar/sidebar.vue#L310-L330
[sidebery-settings]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/defaults/settings.ts#L229-L249
[sidebery-debounce]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/mouse.fg.ts#L272-L339
[sidebery-panels]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/sidebar.fg.ts#L1367-L1475
[node]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/tabs/common/tab_collection_node.cc#L94-L118
[group-view]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/tabs/common/tab_group_view.cc#L60-L100
[group-layout]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/tabs/common/tab_group_view_layout.cc
[group-editor]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/tabs/groups/tab_group_editor_bubble_view.cc
[header]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/tabs/common/tab_group_header_view.cc
[reorder-utility]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/event_utils.cc#L22-L69
[legacy-header]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/tabs/tab_group_header.cc#L257-L290
[group-reorder]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/tabs/common/tab_strip_collection_controller.cc#L755-L801
[group-contract]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/components/tabs/impl/tab_group_tab_collection.cc#L16-L24
[drag-handler]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/tabs/common/tab_drag_handler.cc
