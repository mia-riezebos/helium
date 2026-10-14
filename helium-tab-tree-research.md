# Native opener tracking and tab-tree behavior

Research for [#12](https://github.com/mia-riezebos/helium/issues/12), unblocking [tree behavior #11](https://github.com/mia-riezebos/helium/issues/11). Checked 2026-10-15.

Chromium supplies source-tab attribution, related-tab ordering and temporary opener-based selection. It does not supply a durable user tab tree.
Sidebery supplies a separate parent model, branch operations and restoration metadata. Its configurable defaults are references for Mia's decisions. [Native model][model], [Sidebery tree][s-tree], [Sidebery storage][s-storage].

## Revisions and evidence boundary

| Source | Revision |
| --- | --- |
| Inherited planning documents | `d653b587440f4a13936bac377233db69b0fdbe97`. Tabs may now have children; recursive named groups and panels remain required. Browser-view panels are deferred. [Direction][direction], [terms][context]. |
| Chromium | `155.0.8059.79`, commit `a49e2e3d4addfd84e765b52afe230bedb81096db`. [Version][c-version]. |
| Helium browser baseline | `1fe0fe984484020aae958886adc30618a6ac6dc0`, targeting that Chromium release. [Target][h-version], [patch series][h-series]. |
| Sidebery | `ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18`. [Defaults][s-defaults]. |

**Evidence boundary.** This report reads pinned public source and Helium patches. It does not build, launch or test a browser.
Helium downloads a prepared Chromium base archive. That archive was not materialized; release source plus checked-in patches define this investigation. [Archive declaration][h-downloads].
The four immutable prior inventories and local feasibility note supplied leads. Every behavior below cites its owning source.
“Verified” means traced source behavior. “Inference” states a conclusion from that trace. Runtime uncertainty remains explicit.
No browser code, issue, shared planning document or existing report changes accompany this artifact. No architecture or keybindings are selected.

## Five relationships that must stay distinct

| Relationship | Owning representation and meaning |
| --- | --- |
| Native ownership | `TabCollection` owns tabs/collections, with recursive iteration producing strip order. Tabs have a parent collection. This is collection ancestry. [Contract][collection]. |
| Structural tab ancestry | A tab being another tab's durable child is absent from the target collection types and `TabModel`. Native tabs remain leaves. [Collection contract][collection], [tab model][tab-model]. |
| Native strip opener | `TabModel::opener_handle_` references a live `TabInterface`. Insertion and close selection consume it; reset heuristics modify it. Ownership does not follow this link. [Handle][tab-model], [insertion/selection][model]. |
| JavaScript `window.opener` | Blink reads the frame opener. Setting it to null disowns that frame relationship. `noopener` suppresses script linkage. [Bindings][dom-opener], [window creation][web-create]. |
| Extension `openerTabId` | Desktop serialization reads the strip opener through `TabListBridge`, then obtains its extension tab ID. It is not a separate durable parent field. [Serialization][extension-tab], [bridge][bridge]. |

Content also records an original-opener main-frame chain even when it suppresses the live script opener.
Source destruction reconnects that attribution chain to the source's predecessor. It does not preserve the original immediate parent. [Creation][original-opener], [chain lifecycle][original-chain].
There is also **Back-to-Opener**. Its controller tracks an initiating frame's `WebContents` independently of the strip opener.
At primary-main-frame ready-to-commit, it requires a user gesture and a live initiator in another `WebContents`. [Controller][back-opener].
It clears the relationship when that source dies or navigates to a different URL, including a fragment change.
Pinning disables the action while retaining its cached relationship. Helium enables this feature by default; Chromium's source default is disabled. [Controller][back-opener], [Helium enablement][h-back].
**Inference.** These are different answers to “who opened this?” None is interchangeable with user-maintained ancestry.

## Creation paths and available source identity

The normal browser paths converge on `Navigate` and `TabStripModel::AddTab`.
`Navigate` initially records explicit `source_contents`, or falls back to the originating browser's active tab.
It preassigns that tab as opener when there is a user gesture or `ADD_INHERIT_OPENER`. [Navigator][navigate].
For an exact `LINK` transition without `ADD_FORCE_INDEX`, `AddTab` requests opener inheritance and native related-tab placement.
`InsertTabAtImpl` then assigns the **currently active destination-model tab**, overwriting a preassigned source when one exists. [Addition][add], [insertion][insert].
**Inference.** Normal active-source link opens have a reusable strip link. Background-script or unusual inactive-source opens require distinguishing explicit source from that final heuristic.

| Actual flow | Source trace and limits |
| --- | --- |
| Link context-menu “Open in new tab” | The handler sends `NEW_BACKGROUND_TAB` plus `LINK` through the source `WebContents::OpenURL`. Context-menu parameters retain source-frame IDs and initiator token. The browser delegate sets `nav_params.source_contents = source`. [Menu][context-menu], [parameters][context-params], [delegate][delegate]. |
| Middle/Ctrl/Command link click | The anchor handler derives policy from the event. Middle click, Ctrl off macOS, or Command on macOS requests a background tab; adding Shift requests foreground. `RenderFrameImpl::OpenURL` sends disposition/initiator data. Content's navigator adds source-frame IDs and `LINK`; the source `WebContents` calls the browser delegate. [Anchor][anchor], [policy][click-policy], [renderer][renderer-open], [content navigator][content-nav], [contents delegation][contents-open]. |
| `target=_blank` | Anchor relation handling defaults `_blank` to `noopener`, unless `rel=opener` applies or the setting changes. The setting's inspected initial value is true. New-frame creation still carries an initiating frame. The browser-side source is not dependent on JavaScript opener access. [Relations][anchor-rel], [setting][blank-default], [window creation][blink-create], [web contents][web-create]. |
| JavaScript `window.open` | `LocalDOMWindow::open` builds a frame request; creation sends the calling frame and requested disposition. Accepted new contents reach `AddNewContents(source, ...)`, then `chrome::AddWebContents` and `Navigate`. Named-target reuse can navigate an existing frame, creating no child tab. Popup blocking can prevent creation. [DOM open][dom-open], [renderer creation][renderer-create], [browser delegate][delegate], [tab insertion][tabstrip], [content creation][web-create]. |
| JavaScript open with `noopener`/`noreferrer` | Feature parsing makes `noreferrer` imply `noopener`. Content creates a separate browsing instance and suppresses the live script opener. It still calls `AddNewContents(this, ...)` and reports the initiating frame through `DidOpenRequestedURL`. [Feature parsing][blink-create], [suppressed path][web-create]. This proves source identity in that accepted path, not universal automatic parenting. |
| Duplicate tab | The command clones the specified contents and inserts beside that tab, preserving its pin/group state. It uses `ADD_ACTIVE \| ADD_INHERIT_OPENER`; inheritance uses the old active tab. Duplicating an inactive context-menu target therefore does not make the strip opener reliable clone provenance. [Duplicate handler][duplicate], [insertion][insert]. |
| Ordinary new-tab command/button | `NewTab` calls `AddAndReturnTabAt` with `TYPED`. Navigator fallback supplies active-tab context. The model also gives end-of-strip typed tabs a temporary opener for quick lookup. This is no page-initiated link. [Command][new-tab], [tab helper][tabstrip], [typed heuristic][add], [navigator][navigate]. |
| Explicit new-tab-to-right | The model calls `AddTabAt` with context-target index+1 and that target's group. The helper uses `TYPED`; navigator fallback supplies the active tab as context. The target position does not identify the new tab's opener. [Command][new-right], [helper][tabstrip], [navigator][navigate]. |
| Link/JavaScript new window or popup | The same source arguments exist, but navigation chooses another browser and forces its insertion index. A preassigned source handle can remain when the destination has no active tab. This does not create cross-window collection ownership or normal same-strip ancestry. [Window disposition/insertion][navigate], [delegate][delegate], [insertion][insert]. |
| Ordinary New window command | The normal-browser command supplies a profile to `NewEmptyWindow`, which creates/restores a window; it supplies no source tab. This differs from link/JavaScript new-window creation. macOS app-specific command branches can launch the app instead. [Command][new-window], [empty-window path][new-empty-window]. |
| Extension create/update | An explicit extension opener goes through `SetOpenerOfTab`, which requires the same browser window. Updates reject self-openers. It sets the native strip link, without changing script opener or collection parentage. [Set/create/update handlers][tabs-api], [bridge][bridge]. |

`AddWebContents` sets `user_gesture = true` after popup blocking for focus handling, retaining the original gesture separately.
Thus an accepted script open may receive a native opener even without an original user click. [Insertion helper][tabstrip].
**Reusable facts.** Context menus, modified anchor clicks and accepted renderer-created windows expose concrete source contents/frame identity. [Menu][context-menu], [content navigator][content-nav], [window creation][web-create].
Duplication exposes the cloned target in its command handler. Ordinary new tabs expose active context, not link provenance. [Duplicate][duplicate], [new-tab][new-tab].
**Runtime uncertainty.** This investigation does not prove event timing, popup disposition or source availability for every sandbox, redirect, app, guest or frame case.
Suppressed script access must remain suppressed. Source attribution is not permission to restore `window.opener`. [Security/creation checks][web-create], [DOM bindings][dom-opener].

## Opener lifetime and native ordering

Current-tab `Navigate` calls `BrowserUiController::UpdateUIForNavigationInTab`, which invokes `TabNavigating`. [Call path][navigate], [UI callback][navigation-ui].

| Operation | Verified native behavior |
| --- | --- |
| Navigation | `TYPED`, `AUTO_BOOKMARK`, `GENERATED`, `KEYWORD` and `AUTO_TOPLEVEL` cause `TabNavigating` to forget **all strip openers**. A new-tab page at strip end gets one-navigation leniency. The predicate does not clear for every navigation type. [Predicate][navigation-predicate], [reset][navigation-reset]. |
| Active selection | User selection between unrelated opener families clears all links. Same-opener siblings and direct source/child transitions receive exceptions. Leaving a tab marked `reset_opener_on_active_tab_change` clears its own link. [Selection reset][selection-reset]. |
| New foreground related tab | Opener inheritance first clears existing strip links, then assigns the old active tab. This can discard links among earlier tabs. [Insertion][insert]. |
| Reorder within a strip | Actual index moves call `FixOpeners` on moved tabs. That redirects tabs which reference the moved tab to its opener; it does not simply erase the moved tab's own opener. Nonadjacent descendants stop participating in the contiguous insertion scan. [Move/fix][move-fix], [multi-move][multi-move], [source test][order-tests]. |
| Detach/window move | Detach fixes inbound links. `OnRemovedFromModel` clears the departing tab's own opener and reset flag. A batch/window move therefore does not preserve a structural opener tree by contract. [Detach][detach], [tab removal][tab-model]. |
| Pin/unpin or group changes | Pinning moves to the pin boundary and removes group membership. Index-changing moves run the same opener repair. The moved tab's own link is not a durable group/pin policy; membership-only changes are not an unconditional opener reset. [Pin/move][pin-move], [multi-move][multi-move]. |
| Source closes | Detach invokes `FixOpeners`: direct children inherit the removed source's opener, or null when that would be self-reference. This is live-link promotion, not stored closed-parent ancestry. [Detach][detach], [repair][fix], [nested-opener source test][nested-test]. |
| Contents replacement/discard | `DiscardWebContents` repairs inbound links before swapping contents inside the same `TabModel`. Its own opener field survives that swap. If the discarded child itself has children, their links are repaired away from it. The extension replacement event reports old/new contents IDs separately. [Discard][discard], [model swap][tab-discard], [event][replace-event]. |
| Restore replacing current tab | This separate path inserts restored contents after the active tab with inherited opener, then closes the old tab. It does not recover the historical opener tree. [Replacement restore][tabrestore]. |

Close selection prefers a tab opened by the closing block, then a sibling through its opener, then the opener.
Same-collection and adjacent fallbacks follow, with collapsed/focused-group restrictions. These are selection heuristics, not branch-closure semantics. [Close selection][close-selection].
Native background links append after the last **contiguous transitive opener descendant** of the active tab.
The scan skips unrelated pinned tabs, stops at unrelated unpinned tabs, and insertion stops before a group discontinuity.
Foreground links insert immediately after the active tab. Group/split contiguity and the pin boundary can adjust the result. [Ordering][order], [descendant scan][descendants], [addition constraints][add].
The pinned source case therefore does not mean a pinned parent owns an unpinned branch. [Ordering][order], [pinned constructor][pinned-collection].

Helium's “Open new tabs next to the active tab” preference defaults to false.
For non-link/default-invalid-index placement, enabling it uses active+1, or the active group's end for a background tab.
It does not replace the ordinary link-placement branch. Its changed index also affects whether the typed-at-end heuristic runs. [Helium placement patch][h-placement], [typed condition][add].
Helium's MRU cycling preference defaults to false and redirects next/previous cycling through MRU selection when enabled.
It adds no parent metadata or opener lifetime promise. [MRU patch][h-mru].
**Inference.** Native opener “trees” are temporary task/order/selection links. Their clearing, promotion and contiguous scans rule out treating them as durable ancestry unchanged.

## Persistence, UI and browser bounds

**Verified session representation.** `SessionTab` records window/tab IDs, visual index, navigation history, pin/group/split state and extra-data maps.
`SessionWindow` holds ordered tabs, one selected-tab index and group/split metadata. Neither typed record stores strip opener or tab-parent ancestry. [Session records][sessions], [commands][session-commands].
Restoration relabels group IDs and calls `AddRestoredTab`. That handler inserts with active/pin flags, without recovering an opener field. [Restore caller][session-restore], [restore insertion][tabrestore].
Recently closed tab/group/split/window entries retain indices, navigation and group/split metadata, plus entry extra data. They contain no durable tab-parent field. [Closed records][closed-types].
Historical capture asks `GetExtraDataForTab`; its target implementation populates Glic data rather than opener/tree ancestry. [Capture][closed-capture], [producer][live-context].
Saved-group reopen may focus, move or open the existing saved group. For group/window restore, its saved state is authoritative. [Saved-group branch][saved-restore].
**Inference.** Extra-data storage is reusable storage capability. It does not establish automatic ancestry capture or blanket crash/reopen/saved-group preservation.

**Verified model/UI limits.** The root accepts pinned/unpinned collections; unpinned accepts groups/splits; groups accept splits; splits accept tabs only.
There is no tab-parent collection type, panel type or group-under-group allowance. [Root][root-collection], [unpinned][unpinned-collection], [pinned][pinned-collection], [group][group-collection], [split][split-collection].
The collection-backed view maps tab/group/split/container types, with no tab-branch view. [View mapping][nodes].
The default horizontal path still uses the older strip; collection-backed horizontal controls require disabled-default `TabStripUnification`.
Vertical controls use collection-backed views. Source presence does not prove Windows/macOS/Linux builds or UI parity. [Horizontal factory][horizontal], [feature default][features], [vertical region][vertical].

**Verified extension semantics.** Tabs expose one index/window/group ID and optional `openerTabId`. Groups expose one window, title, colour and collapsed state.
These schemas have no tab-tree parent/collapse or recursive group ancestry. Group IDs are browser-session identifiers. [Tabs schema][tabs-schema], [groups schema][groups-schema].
Sidebery's practice of clearing extension opener with the tab's own ID is incompatible with Chromium's update handler, which rejects that value. [Sidebery normalization][s-tree], [Chromium update][tabs-api].

**Verified bounds.** `Navigate` clears source contents/opener/site-instance/referrer when source and destination profiles differ.
Drag compatibility requires the same `Profile` and compatible browser/app type. Native grouping skips source tabs outside the destination strip. [Profile clearing][navigate], [drag check][drag], [group setup][tabstrip].
Extension opener mutation requires the same window and resolves IDs through the caller's profile/incognito visibility. [API checks][tabs-api].
Restored contents use the restoring browser's profile. Session service selects original regular profiles; this is no promise to persist private-window organization. [Restore creation][tabrestore], [service scope][session-factory].
**Inference.** Source identity can be observed across a new-window operation without creating ownership across windows, profiles or incognito boundaries.

## Sidebery automatic parenting and branch references

These are **verified source defaults and handlers**, not selected Helium policy. “No opener” does not imply that every possible inserted tab stays root-level.
`onTabCreated` obtains panel/index/parent separately, supports explicit/reopen positions, and normalizes the result against ordered tree state. [Creation handler][s-created], [placement functions][s-create].

| Case | Defaults, dependencies and actual behavior |
| --- | --- |
| Source open from an ordinary tab | `tabsTree=true`, `moveNewTabParent=last_child`, `moveNewTabParentActPanel=false`. Normal source opens use the source panel and become its last child, after its descendant branch. `first_child`, `sibling`, `before`, start/end/default/none settings change this behavior. [Defaults][s-defaults], [panel/index/parent handlers][s-create]. |
| No opener / new-tab button | `moveNewTab=end` and `moveNewTabButton=end` choose panel end without assigning active-tab parent. Other settings can assign active-tab children or siblings. Unknown/external insertion can adopt the next tab's parent; explicit placement and reopen metadata take precedence. [Defaults][s-defaults], [parent function][s-create], [creation normalization][s-created]. |
| Source is pinned | `moveNewTabPin=start`, `pinnedAutoGroup=false`. Default offspring start the active tab panel and do not parent under the pin. Optional automatic grouping uses a separate linked group-page tab. [Defaults][s-defaults], [placement][s-create], [pinned source handling][s-created]. |
| New child under collapsed parent | `ignoreFoldedParent=false`, `autoExpandTabsOnNew=false`. It stays under the parent and invisible, with a parent flash. Enabling ignore outdents; enabling expansion opens the branch. Folded ancestors also make the new tab invisible. [Defaults][s-defaults], [handler][s-created]. |
| Collapse and selection | `autoFoldTabs=false`, `autoExpandTabs=false`, `hideFoldedTabs=false`, `discardFolded=false`. Folding hides descendants in Sidebery and activates the root if an active descendant is inside. Native hiding/discarding require separate settings. Activating an invisible tab expands ancestors. Its action-selection helper selects a folded parent's branch outside search. [Defaults][s-defaults], [fold/expand][s-fold], [activation][s-activation], [selection helper][s-select]. |
| Parent close and promotion | `rmChildTabs=folded`, `treeRmOutdent=branch`, `autoGroupOnClose=false`. Closing a folded parent includes descendants; an expanded parent leaves children promoted to its old parent. `first_child` instead makes the first child parent of surviving siblings; all/none options change descendant closure. Search suppresses automatic descendant inclusion. [Defaults][s-defaults], [close set][s-rm], [removal handler][s-removed]. |
| Close-branch confirmation | `warnOnMultiTabClose=collapsed`. `removeTabs` confirms multiple tabs when invisible descendants are included. Cancellation restores visibility and has a separate already-removed-parent recovery path. This is a reference dependency, not a proposed notice. [Defaults][s-defaults], [confirmation/recovery][s-rm]. |
| Pin/unpin extraction | Sidebery pin commands promote direct children to the pin's former parent. Tree normalization makes pins visible roots with no child/fold state. Global-pin unpinning chooses the active/previous tab panel and its start; linked group-page handling can change replacement behavior. [Pin commands][s-pin], [normalization][s-tree], [unpin handler][s-unpin]. |
| Branch moves | Drag collection includes descendants. The move handler retains internal parent links when both tabs move; it reparents moved roots to the destination parent and promotes left-behind children. Moving active content under a folded parent activates that parent. Dropping into an item in the dragged set is rejected. [Drag gather][s-drag], [move handler][s-move], [drop guard][s-drop]. |
| URL routing / panel boundaries | Active rules match URL substring or regex, optional container and top-level-only filters. URL changes route unpinned non-blank tabs after a delay. Routing passes only that tab to `Tabs.move`, with no destination parent; descendants staying behind are promoted. Parent normalization rejects other-panel parents, pins and parents at/after the child. [Rules][s-rules], [URL handler][s-url], [routing move][s-route], [move][s-move], [normalization][s-tree]. |

Panel selection for initial creation first checks rules without URL matchers. URL matching happens through the subsequent URL-change handler.
`moveNewTabParentActPanel` can stop source-panel placement when another panel is active. `default` placement follows native index rather than guaranteeing source-panel membership. [Panel function][s-create], [rules][s-rules].
**Verified tree constraint.** Parent tabs must precede children, be unpinned and share their panel. Levels and native order normalize the tree.
Even the `none` depth-limit option maps to an internal maximum level of 123; it is not a mathematical unlimited-depth guarantee. [Normalization][s-tree].

Sidebery persists its own `parentId`, `panelId` and `folded`, using `browser.sessions.setTabValue` and a non-private-window cache.
Restore remaps stored IDs, tries matching cache/session metadata, and uses opener as a fallback. [Save/cache][s-storage], [restore][s-restore].
Its manifest targets Gecko. Chromium's sessions API declares recently-closed/device retrieval and restore, without `setTabValue`. These persistence APIs do not transfer unchanged. [Manifest][s-manifest], [Chromium API][sessions-schema].
Sidebery groups are ordinary group-page tabs created with `browser.tabs.create`; their children use the tab-parent machinery.
They are separate from Mia's required recursive named group nodes and Chromium's named group collections. [Group creation][s-groups], [terms][context], [native group constructor][group-collection].

## Two existing Helium proposal leads

| Proposal | Revision-pinned findings and compatibility limits |
| --- | --- |
| [#1832](https://github.com/imputnet/helium/pull/1832) | Inspected `ImTheSquid/helium@d061f7aa5bd1818694627c33ea695076a1c46b29`. Its target is Chromium **150.0.7871.114**. The patch adds `NestedTabCollection`, nested session/reopen records and branch UI/movement. It hooks explicit source arguments in `Browser::OpenURLFromTab` and `Browser::AddNewContents`. Nested owners cannot be pinned or split; groups gain nested children but do not gain group children. [Patch][proposal-1832], [version][proposal-1832-version]. |
| [#1468](https://github.com/imputnet/helium/pull/1468) | Inspected `riomadeit/helium@689b1b63af433c4c8b0a61f16f124c495735582d`. Its target is Chromium **147.0.7727.101**. The patch adds a tree controller, node/parent IDs and collapsed extra data. Automatic insertion parenting checks a `LINK` transition and opener-before-child order, excluding temporary typed new tabs. [Patch][proposal-1468], [version][proposal-1468-version]. |

Both PRs were open at inspection. Their descriptions report macOS testing, not verification on all required platforms. [PR #1832](https://github.com/imputnet/helium/pull/1832), [PR #1468](https://github.com/imputnet/helium/pull/1468).
The target 155 source delegates these browser navigation methods through `BrowserWebContentsDelegate`; #1832 hooks older `Browser` methods.
**Inference.** Neither patch establishes drop-in compatibility, complete recursive named groups/panels, or current-baseline persistence. No patch application or runtime validation occurred. [Target delegate][delegate], [proposal patch][proposal-1832].
The #1468 description specifically reports that reopening closed tabs loses tree position. Its extra-data code alone does not prove end-to-end recovery. [PR #1468](https://github.com/imputnet/helium/pull/1468), [patch][proposal-1468].

## Remaining product choices for #11

1. Define the relationship between tab ancestry and named groups. Decide what happens when source and destination groups or panels differ.
2. Set placement for foreground opens, repeated siblings, no-opener/new-tab commands, duplication and accepted script/noopener opens. Link Open in new tab must remain a child.
3. Decide whether new children expand collapsed parents, and how collapse changes active selection and branch selection.
4. Choose surviving-child promotion versus branch closure, plus pin/unpin extraction and branch move behavior.
5. Decide whether URL routing moves a tab alone or preserves tab ancestry, independently of the agreed named-group-path option. Define restored/reopened parent recovery with #8.

**Resolution.** The prerequisite is answered by source evidence. Native creation handlers provide reusable source identity; strip ordering and lifecycle provide bounded reuse behavior.
Durable tab-parent/collapse representation, tree UI and tree restoration remain absent from the pinned baseline. #11 can choose behavior without treating native opener links as persisted ancestry. [Model][model], [collection contract][collection], [sessions][sessions], [nodes][nodes].

## Sources

[direction]: https://github.com/mia-riezebos/helium/blob/d653b587440f4a13936bac377233db69b0fdbe97/DIRECTION.md
[context]: https://github.com/mia-riezebos/helium/blob/d653b587440f4a13936bac377233db69b0fdbe97/CONTEXT.md
[c-version]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/VERSION
[h-version]: https://github.com/imputnet/helium/blob/1fe0fe984484020aae958886adc30618a6ac6dc0/chromium_version.txt
[h-series]: https://github.com/imputnet/helium/blob/1fe0fe984484020aae958886adc30618a6ac6dc0/patches/series
[h-downloads]: https://github.com/imputnet/helium/blob/1fe0fe984484020aae958886adc30618a6ac6dc0/downloads.ini
[model]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/tab_strip_model.cc
[tab-model]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/tab_model.cc#L98-L124
[collection]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/components/tabs/public/tab_collection.h
[root-collection]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/components/tabs/impl/tab_strip_collection.cc#L58-L66
[unpinned-collection]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/components/tabs/impl/unpinned_tab_collection.cc#L16-L19
[pinned-collection]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/components/tabs/impl/pinned_tab_collection.cc#L9-L12
[split-collection]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/components/tabs/impl/split_tab_collection.cc#L14-L22
[group-collection]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/components/tabs/impl/tab_group_tab_collection.cc#L14-L24
[dom-opener]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/third_party/blink/renderer/core/frame/dom_window.cc#L347-L385
[back-opener]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/back_to_opener/back_to_opener_controller.cc
[h-back]: https://github.com/imputnet/helium/blob/1fe0fe984484020aae958886adc30618a6ac6dc0/patches/helium/core/enable-back-to-opener.patch
[navigate]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/navigator/browser_navigator.cc#L708-L1003
[add]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/tab_strip_model.cc#L1808-L1903
[insert]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/tab_strip_model.cc#L3772-L3800
[context-menu]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/renderer_context_menu/render_view_context_menu.cc#L3799-L3826
[context-params]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/components/renderer_context_menu/render_view_context_menu_base.cc#L514-L548
[delegate]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/browser_web_contents_delegate/browser_web_contents_delegate.cc#L533-L747
[anchor]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/third_party/blink/renderer/core/html/html_anchor_element.cc#L370-L529
[click-policy]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/third_party/blink/renderer/core/loader/navigation_policy.cc#L49-L92
[renderer-open]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/content/renderer/render_frame_impl.cc#L6117-L6192
[content-nav]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/content/browser/renderer_host/navigator.cc#L1070-L1170
[contents-open]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/content/browser/web_contents/web_contents_impl.cc#L6852-L6928
[anchor-rel]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/third_party/blink/renderer/core/html/anchor_element_utils.cc#L171-L192
[blank-default]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/third_party/blink/renderer/core/frame/settings.json5#L380-L383
[blink-create]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/third_party/blink/renderer/core/page/create_window.cc
[web-create]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/content/browser/web_contents/web_contents_impl.cc#L5611-L5907
[original-opener]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/content/browser/web_contents/web_contents_impl.cc#L12648-L12667
[original-chain]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/content/browser/renderer_host/frame_tree_node.cc#L90-L114
[dom-open]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/third_party/blink/renderer/core/frame/local_dom_window.cc#L2455-L2598
[renderer-create]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/content/renderer/render_frame_impl.cc#L7069-L7236
[tabstrip]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/browser_tabstrip.cc#L44-L190
[duplicate]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/browser_commands.cc#L369-L405
[new-tab]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/browser_commands.cc#L1366-L1408
[new-window]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/browser_commands.cc#L1297-L1339
[new-empty-window]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/browser_commands.cc#L916-L989
[new-right]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/tab_strip_model.cc#L2698-L2720
[tabs-api]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/extensions/api/tabs/tabs_api.cc
[extension-tab]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/extensions/extension_tab_util.cc#L454-L462
[bridge]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/tab_list_bridge.cc#L173-L185
[navigation-reset]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/tab_strip_model.cc#L1484-L1501
[navigation-predicate]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/tab_strip_model.cc#L150-L165
[navigation-ui]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/browser_ui_controller/browser_ui_controller.cc#L70-L87
[selection-reset]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/tab_strip_model.cc#L5622-L5669
[move-fix]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/tab_strip_model.cc#L4921-L4959
[multi-move]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/tab_strip_model.cc#L5319-L5475
[order-tests]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/tab_strip_model_unittest.cc#L2106-L2137
[detach]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/tab_strip_model.cc#L998-L1038
[pin-move]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/tab_strip_model.cc#L5401-L5410
[fix]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/tab_strip_model.cc#L5532-L5555
[nested-test]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/tab_strip_model_unittest.cc#L2160-L2225
[discard]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/tab_strip_model.cc#L470-L500
[tab-discard]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/tab_model.cc#L444-L459
[replace-event]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/extensions/api/tabs/tabs_event_router.cc#L602-L624
[order]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/tab_strip_model.cc#L5688-L5730
[close-selection]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/tab_strip_model.cc#L5734-L5915
[descendants]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/tab_strip_model.cc#L1457-L1481
[h-placement]: https://github.com/imputnet/helium/blob/1fe0fe984484020aae958886adc30618a6ac6dc0/patches/helium/core/open-new-tabs-next-to-active-tab-option.patch
[h-mru]: https://github.com/imputnet/helium/blob/1fe0fe984484020aae958886adc30618a6ac6dc0/patches/helium/core/tab-cycling-mru.patch
[sessions]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/components/sessions/core/session_types.h
[session-commands]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/components/sessions/core/session_service_commands.cc
[session-restore]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/sessions/session_restore.cc#L1120-L1139
[tabrestore]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/browser_tabrestore.cc
[closed-types]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/components/sessions/core/tab_restore_types.h
[closed-capture]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/components/sessions/core/tab_restore_service_helper.cc#L1464-L1492
[live-context]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/browser_live_tab_context.cc#L169-L177
[saved-restore]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/browser_live_tab_context.cc#L317-L407
[nodes]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/tabs/common/tab_collection_node.cc#L23-L44
[horizontal]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/frame/horizontal_tab_strip_region_view.cc
[features]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/features.cc
[vertical]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/frame/vertical_tab_strip_region_view.cc
[tabs-schema]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/common/extensions/api/tabs.json
[groups-schema]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/common/extensions/api/tab_groups.json
[sessions-schema]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/common/extensions/api/sessions.json
[drag]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/tabs/dragging/tab_drag_controller.cc#L2988-L3007
[session-factory]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/sessions/session_service_factory.cc#L63-L71
[s-defaults]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/defaults/settings.ts#L85-L188
[s-manifest]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/manifest.json
[s-create]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.create.ts#L643-L908
[s-created]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.handlers.ts#L246-L628
[s-tree]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.ts#L1935-L2040
[s-storage]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.ts#L725-L836
[s-restore]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.ts#L343-L555
[s-fold]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.ts#L1654-L1760
[s-activation]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.handlers.ts#L1783-L1812
[s-select]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/sidebar/components/tab.vue#L721-L729
[s-rm]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.rm.ts#L192-L285
[s-removed]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.handlers.ts#L1220-L1335
[s-pin]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.ts#L1221-L1258
[s-unpin]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.handlers.ts#L972-L1064
[s-drag]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/sidebar/components/tab.vue#L563-L624
[s-move]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.move.ts#L95-L320
[s-drop]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/drag-and-drop.fg.ts#L1131-L1141
[s-rules]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.move.ts#L742-L902
[s-url]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.handlers.ts#L853-L908
[s-route]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.move.ts#L805-L851
[s-groups]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.groups.ts#L113-L127
[proposal-1832]: https://github.com/ImTheSquid/helium/blob/d061f7aa5bd1818694627c33ea695076a1c46b29/patches/helium/core/nested-tabs.patch
[proposal-1832-version]: https://github.com/ImTheSquid/helium/blob/d061f7aa5bd1818694627c33ea695076a1c46b29/chromium_version.txt
[proposal-1468]: https://github.com/riomadeit/helium/blob/689b1b63af433c4c8b0a61f16f124c495735582d/patches/helium/ui/tree-tabs.patch
[proposal-1468-version]: https://github.com/riomadeit/helium/blob/689b1b63af433c4c8b0a61f16f124c495735582d/chromium_version.txt
