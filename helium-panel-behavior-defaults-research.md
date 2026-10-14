# Panel, routing and shared-pin defaults

Bounded source research for [HITL ticket #6](https://github.com/mia-riezebos/helium/issues/6), checked on 2026-10-14. This report supplies facts, not a ticket resolution or implementation proposal.

Read [Direction][direction] and [Terminology][context] at parent commit `2782ce9661bc1d2f8bcd635c7a1c185b377277b3`. Existing Chromium and Helium behavior remains the baseline. Sidebery supplies references for missing capabilities.

All primary-source links pin Sidebery v5 to `ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18` or Chromium 155.0.8059.79 to `a49e2e3d4addfd84e765b52afe230bedb81096db`. The [native inventory][native-report], [Sidebery inventory][sidebery-report], and [profile investigation][profile-report] supplied leads. The Sidebery checkout is clean at its target SHA. The two cached Chromium selection files match fresh downloads byte-for-byte.

**Evidence boundary.** Facts below describe source paths. No browser was built or launched. Runtime event ordering, races and reliability remain unverified. No browser source or shared planning document changed.

## 1. Panel definitions and window contents

- **Source fact.** Definitions and navigation order live in the extension-wide `sidebar` storage value, without a window key. Managed configuration takes precedence over local storage. Saves broadcast configuration changes to other connected sidebars. Windows therefore share definitions within the same extension/profile context. [Loading][config-load], [saving][config-save], [broadcast][config-broadcast], [listener][config-listener].
- **Source fact.** Each sidebar loads its current window's native tabs and constructs its own panel contents. Active-panel session state uses a window ID. Shared definitions do not mean shared live tabs or a shared active panel. [Window tabs][window-tabs], [window selection][window-panel], [panel contents][panel-contents].
- **Source fact.** `noEmpty=false` applies to the initial default tabs panel and newly created tabs panels. Both factories copy the same defaults. The navigation add-panel action nevertheless creates an initial tab explicitly. That is separate from the persistent `noEmpty` setting. [Defaults][panel-defaults], [initial factory][config-factory], [new factory][panel-factory], [add action][panel-add].
- **Source fact.** Closing the last ordinary tab preserves the panel definition. With `noEmpty=true`, removal requests a replacement with `active:false`. Otherwise, the default `hideEmptyPanels=true` hides empty panels in navigation; the removal handler may switch away. Global pins do not count as panel-local contents. [Removal][panel-last-close], [hidden navigation][panel-nav], [defaults][empty-defaults], [pin partition][panel-contents].
- **Source fact.** Explicit switching to an empty tabs panel creates a tab when any of `noEmpty`, `hideInact`, or `hideEmptyPanels` is true. The caller can suppress creation. All three false permits an empty destination. Deleting a panel instead removes its definition and saves configuration; populated-panel deletion offers attach, bookmark-and-close, close, or leave handling. [Switch][panel-switch], [deletion][panel-delete].

**Configuration dependencies.** Defaults are `noEmpty=false`, `hideEmptyPanels=true`, `hideInact=false`, and `hideDiscardedTabPanels=false`. Empty-panel visibility uses window-local contents. Shared pins are excluded unless pin placement is `panel`. [Panel defaults][panel-defaults], [visibility defaults][empty-defaults], [native defaults][native-defaults], [partition][panel-contents].

**Runtime uncertainty.** Closing the entire window's last native tab also depends on Firefox's window lifecycle. The panel-removal handler does not establish that outcome.

## 2. Pinning, unpinning and new tabs

- **Source fact.** Pinning normally retains `panelId`. A recent native move can temporarily change it; the pin handler restores `prevPanelId` within its one-second recovery condition. `prevPanelId` is move bookkeeping, not an unpin-return destination. Tree normalization clears pinned tabs' `parentId`, depth, parent flag and folded state. Pinning through Sidebery promotes immediate children to the pinned tab's former parent. [Pin handler][pin-handler], [move bookkeeping][move-bookkeeping], [tree reset][pin-tree], [pin command][pin-command].
- **Source fact.** Ordinary global unpinning uses the currently viewed tabs panel, then the previous tabs panel, then the first tabs panel. Panel-local pin placement instead uses the pin's retained panel. The handler assigns the destination and moves to its leading ordinary-tab boundary. It does not restore the former tree parent or old group. Explicit drag/move unpinning has its own supplied destination and bypasses this automatic placement. [Unpin handler][unpin-handler], [move destination][explicit-unpin], [move flag][explicit-unpin-flag].
- **Source fact.** A linked `relGroupId` supplies a special unpin path: the pin moves to the related group tab, replaces it, and removes that group tab. This is linked-group behavior, not restoration of the pin's old ancestry. Automatic groups for pinned openers are disabled by default. Cross-panel results of that special replacement are **unverified** here. [Replacement][pin-group-replace], [auto-group condition][pin-auto-group], [pin defaults][new-tab-defaults].
- **Source fact.** With defaults and no overriding container rule, a tab opened from a pinned opener enters the currently viewed tabs panel at its start. The panel fallback is the previous tabs panel, then a panel near the native tab index. The pinned opener supplies no ordinary tree parent. Optional pinned auto-grouping changes this path. [Destination][new-tab-panel], [placement][pin-opener-position], [parent][pin-opener-parent], [defaults][new-tab-defaults].
- **Source fact.** A normal native new tab without an opener also chooses the viewed tabs panel by default, but enters at its end (`moveNewTab='end'`). Sidebery's new-tab-in-panel command chooses that panel, or the previous tabs panel; its default `general` position also uses the end. Its new-tab button defaults to the end. The active-pin position options apply to relative placement modes, not the default end mode. Later URL routing can move the new tab again. [Destination][new-tab-panel], [command][new-tab-command], [command default][new-tab-key-default], [index calculation][new-tab-index], [defaults][new-tab-defaults], [routing event][routing-event].

**Runtime uncertainty.** External browser pin/move events and queued routing can arrive in different orders. The source guards describe intended handling; this report does not validate those races.

## 3. Panel switching and selection

- **Source fact.** Sidebery defaults both `activateLastTabOnPanelSwitching` and its loaded-only option to true. Restoration consults the destination's active-tab history, then its first ordinary tab. If the candidate is discarded, it uses the first loaded ordinary tab. If none exists, it makes no activation request. An empty panel also has no restoration candidate. [Defaults][selection-defaults], [restoration][panel-restore], [history][panel-history].
- **Source fact.** A global active pin skips that restoration. Panel-local pins do not receive this exemption. Search or a caller's suppression flag can also skip restoration. Switching clears Sidebery multi-selection unless dragging or selection preservation prevents it. Native highlighting defaults on; its delayed update retains the active tab and remaining selected visible tabs. There is no per-panel multi-selection restoration in these paths. [Switch][panel-switch], [selection reset][selection-reset], [highlight update][selection-highlight], [native default][native-defaults].
- **Source fact.** Empty-panel creation occurs before the global-pin restoration guard and still runs with a global pin active. It calls tab creation without `active:false`. Therefore “a global pin always stays active” overstates the switch path. With creation suppressed, or all three empty-panel settings false, no tab is created and no empty-panel candidate is activated. [Switch][panel-switch], [creation request][tab-create].
- **Source fact.** Inactive-panel native-tab hiding defaults off. When enabled, visibility follows the viewed panel while a pin is active, and panel activation requests a visibility update. This does not activate an inactive panel's tab. [Defaults][native-defaults], [visibility update][native-visibility], [panel activation][pin-panel-visibility].

**Chromium cross-check.** Group focusing is disabled by default at this revision. Entering another focused group removes selected tabs outside that group, including pins. Removal clears an active pin; the transition then selects and activates the group's first tab, including split companions when applicable. Separately, validity checks allow selecting a pin while a group is already focused. These are different operations. Leaving focus copies the current selection without restoring an earlier group selection. This is native flat-group focus, not a panel feature. [Feature default][chromium-focus-default], [focus transition][chromium-focus], [removal and pin validity][chromium-selection].

**Runtime uncertainty.** Empty-panel tab activation and delayed highlight timing were not exercised. The source proves the requests and guards, not a tested UI sequence.

## 4. Routing semantics and Chromium URL conventions

### Sidebery reference

- **Source fact.** Each panel owns an ordered rule list. The configuration has `id`, `active`, optional `name`, `url`, `containerId`, and `topLvlOnly`. The initial editor toggle is false, and absent `topLvlOnly` means false. At this SHA, the add handler omits that field even when its toggle is set; editing an existing rule does save it. Cancel/save reset the editor toggle to true. The stored value and effective default must be distinguished from the displayed toggle. [Schema][rule-schema], [editor default][rule-default], [add handler][rule-add], [edit handler][rule-edit].
- **Source fact.** Enabled rules collect in panel order and rule-list order. Rules combining container and URL criteria sort before single-criterion rules. Matching returns the first eligible rule; `topLvlOnly` excludes tabs with depth greater than zero. Plain URL text is a case-sensitive whole-URL substring; `/.../` creates a regex without flags, with literal fallback on an invalid regex. [Compilation][rule-compile], [matching][rule-match].
- **Source fact.** The URL-update routing path checks no manual-move override. The inspected tab schema and generic move path store no durable exemption. A later eligible URL change can route a manually moved tab again. Tree depth, pin state and the rule predicates still affect eligibility. [URL event][routing-event], [tab schema][tab-schema], [move state][move-state], [matching][rule-match].
- **Source fact.** No match normally leaves the tab in place. The current panel can specify `moveExcludedTo` instead, defaulting to `-1`. That fallback requires configured rules on that panel and at least one compiled rule globally; it skips URLs beginning with `a` or `m`. It is not a universal default-panel destination. [Fallback][rule-route], [default][panel-defaults].
- **Source fact.** The inspected configuration-update path recompiles rules and can reorder tabs by existing panel membership. It has no pass matching all existing tab URLs. The editor saves configuration; the background broadcaster excludes the originating sidebar. **Unverified:** immediate compiled-rule refresh in that originating sidebar after an in-sidebar edit. A guaranteed edit-triggered rescan is not established. [Update path][rule-update], [editor save][rule-edit], [broadcast][config-broadcast].

**Runtime uncertainty.** Navigation routing is scheduled after 120 ms, excluding pinned tabs and `about:blank`. The timer reads current tab state when it fires. Redirects, manual moves during that delay, and already queued moves remain untested. [Event][routing-event], [timer][rule-route].

### Existing Chromium conventions, not a product syntax choice

| Owning source | Exact domains and subdomains | Paths |
| --- | --- | --- |
| Extension `URLPattern` | A literal host matches that host. `*.example.com` matches the apex and descendant subdomains at dot boundaries. `*` matches all hosts; embedded host globs are rejected. [Grammar][url-pattern-grammar], [host comparison][url-pattern-host], [targeted tests][url-pattern-tests]. | A path is required and permits `*` globs. `/foo*` matches `/foo` and `/foobar`. `MatchesURL` passes `PathForRequest`, rather than a host substring, into path matching. Default path matching is case-sensitive. [Grammar][url-pattern-grammar], [tests][url-pattern-tests], [URL call][url-pattern-url], [path matching][url-pattern-path]. |
| URLMatcher utility's administrator URL-filter format | Bare `example.com` includes subdomains. Leading-dot `.example.com` means exact host. The whole-host `*` means all hosts. This format does not use the extension format's `*.` marker for subdomain inclusion. [Parser][url-filter-parser], [tests][url-filter-host-tests]. | Paths are case-sensitive prefixes, with query criteria handled separately. `/ruin` also matches `/ruins` and `/ruin/signup`, so prefixes are not segment boundaries. [Condition creation][url-filter-conditions], [path tests][url-filter-path-tests], [case tests][url-filter-case-tests]. |

The utility header identifies its owning administrator-format documentation. The target's URLBlocklist policy declaration also links that documentation and gives exact-host/path examples. These policy and extension conventions establish existing behavior, not panel routing. [Utility contract][url-filter-contract], [policy declaration][url-filter-policy].

**Inference for the parent.** “Use Chromium URL matching” alone does not select one unambiguous domain/path contract. Chromium's existing families differ on bare hosts, subdomain notation and path glob versus prefix behavior. Sidebery's substring matcher supplies neither domain boundaries nor path-only matching. That is the actual remaining product ambiguity surfaced here. No helper, syntax, implementation, or HITL answer is selected.

[direction]: https://github.com/mia-riezebos/helium/blob/2782ce9661bc1d2f8bcd635c7a1c185b377277b3/DIRECTION.md
[context]: https://github.com/mia-riezebos/helium/blob/2782ce9661bc1d2f8bcd635c7a1c185b377277b3/CONTEXT.md
[native-report]: https://github.com/mia-riezebos/helium/blob/bbe2d36f3d4d5f3333657aee2ce842d3646a76da/helium-native-tabs-research.md
[sidebery-report]: https://github.com/mia-riezebos/helium/blob/107047b817835dbabc7e77f2b83cf6fa177850c1/helium-sidebery-behavior-research.md
[profile-report]: https://github.com/mia-riezebos/helium/blob/d92d98fdd8e2ce1a8386e2bfec99966809ffc215/helium-profile-panels-research.md
[config-load]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/sidebar.fg.ts#L185-L205
[config-save]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/sidebar.fg.ts#L909-L921
[config-broadcast]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/storage.bg.ts#L13-L79
[config-listener]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/sidebar.fg.ts#L163-L168
[window-tabs]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.ts#L343-L348
[window-panel]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/sidebar.fg.ts#L1270-L1279
[panel-contents]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/sidebar.fg.ts#L383-L483
[panel-defaults]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/defaults/panels.ts#L57-L80
[config-factory]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/sidebar-config.ts#L33-L73
[panel-factory]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/sidebar.fg.ts#L1783-L1801
[panel-add]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/sidebar/components/bar.navigation.vue#L743-L776
[panel-last-close]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.handlers.ts#L1364-L1418
[panel-nav]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/sidebar/components/bar.navigation.vue#L155-L169
[empty-defaults]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/defaults/settings.ts#L34-L40
[native-defaults]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/defaults/settings.ts#L185-L190
[panel-switch]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/sidebar.fg.ts#L1290-L1323
[panel-delete]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/sidebar.fg.ts#L1698-L1771
[pin-handler]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.handlers.ts#L1032-L1068
[move-bookkeeping]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.handlers.ts#L1486-L1508
[pin-tree]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.ts#L1935-L1967
[pin-command]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.ts#L1221-L1257
[unpin-handler]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.handlers.ts#L983-L1023
[explicit-unpin]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.move.ts#L247-L270
[explicit-unpin-flag]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.move.ts#L341-L349
[pin-group-replace]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.groups.ts#L47-L57
[pin-auto-group]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.handlers.ts#L283-L292
[new-tab-defaults]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/defaults/settings.ts#L120-L134
[new-tab-panel]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.create.ts#L643-L702
[pin-opener-position]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.create.ts#L732-L736
[pin-opener-parent]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.create.ts#L844-L852
[new-tab-command]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/keybindings.fg.ts#L551-L559
[new-tab-key-default]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/defaults/settings.ts#L302-L303
[new-tab-index]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.create.ts#L774-L823
[routing-event]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.handlers.ts#L905-L908
[selection-defaults]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/defaults/settings.ts#L85-L90
[panel-restore]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.ts#L1184-L1215
[panel-history]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.ts#L2411-L2452
[selection-reset]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/selection.fg.ts#L520-L578
[selection-highlight]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/selection.fg.ts#L584-L606
[tab-create]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.create.ts#L81-L125
[native-visibility]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.ts#L1548-L1592
[pin-panel-visibility]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/sidebar.fg.ts#L1257-L1267
[chromium-focus-default]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/ui_features.cc#L554-L556
[chromium-focus]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/tab_strip_model.cc#L261-L324
[chromium-selection]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/tab_strip_model_selection_state.cc#L54-L103
[rule-schema]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/types/sidebar.ts#L237-L252
[rule-default]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/components/popup.tab-move-rules.vue#L102-L106
[rule-add]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/components/popup.tab-move-rules.vue#L237-L257
[rule-edit]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/components/popup.tab-move-rules.vue#L328-L424
[rule-compile]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.move.ts#L742-L803
[rule-match]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.move.ts#L876-L902
[tab-schema]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/types/tabs.ts#L7-L60
[move-state]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.move.ts#L247-L270
[rule-route]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.move.ts#L805-L851
[rule-update]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/sidebar.fg.ts#L1115-L1176
[url-pattern-grammar]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/extensions/common/url_pattern.h#L18-L48
[url-pattern-host]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/extensions/common/url_pattern.cc#L510-L546
[url-pattern-tests]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/extensions/common/url_pattern_unittest.cc#L221-L254
[url-pattern-url]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/extensions/common/url_pattern.cc#L446-L468
[url-pattern-path]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/extensions/common/url_pattern.cc#L586-L618
[url-filter-parser]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/components/url_matcher/url_util.cc#L375-L397
[url-filter-host-tests]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/components/url_matcher/url_util_unittest.cc#L461-L467
[url-filter-conditions]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/components/url_matcher/url_util.cc#L267-L288
[url-filter-path-tests]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/components/url_matcher/url_util_unittest.cc#L345-L377
[url-filter-case-tests]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/components/url_matcher/url_util_unittest.cc#L519-L527
[url-filter-contract]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/components/url_matcher/url_util.h#L76-L105
[url-filter-policy]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/components/policy/resources/templates/policy_definitions/Miscellaneous/URLBlocklist.yaml#L3-L29
