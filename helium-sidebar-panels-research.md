# Bookmarks and optional sidebar panels

Research prerequisites for [issue #10](https://github.com/mia-riezebos/helium/issues/10), checked 2026-10-15. Native bookmarks already have profile-owned storage, a manager, menus, editing controls, and a compact side-panel view. Their opening behavior differs by entry point. [Model factory][bookmark-factory], [manager][manager-ui], [compact view][bookmark-list], [opening][bookmark-open].
Full history exists. The older Journeys panel exists but Helium disables its feature by default. The newer by-date panel is disabled by default and its app template is empty at the target. [History UI][history-ui], [cluster view][cluster-template], [Helium gate][helium-journeys], [by-date gate][history-feature], [empty template][history-template].
The current side-panel host replaces one entry with another. Showing bookmarks there replaces an extension entry; the separate tab region does not establish a second bookmark host. [Host replacement][coordinator], [browser children][browser-view].

## Exact revisions and evidence boundary

| Source | Pinned revision |
| --- | --- |
| Parent scope and terminology | `1e361f9e3492970a715666a15e6bb7ab218f7df4`. Bookmarks is required at the left edge; history remains optional. Native reuse and independent extension availability are constraints. [Direction][direction], [terms][context]. |
| Chromium | `155.0.8059.79`, commit `a49e2e3d4addfd84e765b52afe230bedb81096db`. [Helium target][version-target], [Chromium version][version-source]. |
| Helium browser patches | `1fe0fe984484020aae958886adc30618a6ac6dc0`, including its ungoogled-chromium patches and ordered patch series. [Series][series]. |
| Sidebery | `ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18`. [Pinned defaults][side-settings]. |

**Evidence method.** Read owning source files and applicable patches. Prior inventories supplied leads only. The prepared Chromium archive was not materialized. [Archive declaration][downloads].
**Boundary.** Source facts describe these revisions, not an installed binary's flags or observed interactions. No builds, launches, runtime checks, browser-code edits, issues, or PRs form part of this research.

## Native component inventory

| Existing component | Source fact and Helium status |
| --- | --- |
| `BookmarkModel` / `BookmarkNode` | URL/folder hierarchy, permanent bookmark-bar/other/mobile roots, IDs, UUIDs, URL lookup, mutations, and observers. Account nodes are a distinct node subset. The factory loads the model from the profile path. [Model][bookmark-model], [nodes][bookmark-node], [factory][bookmark-factory]. |
| Storage and merged presentation | `BookmarkStorage` schedules native serialization; `BookmarkMergedSurfaceService` presents underlying permanent folders and ordering. Its factory uses the same bookmark model. Synthetic side-panel root IDs are presentation IDs, not additional bookmark records. [Storage][bookmark-storage], [merged service][merged-service], [merged factory][merged-factory], [synthetic IDs][bookmark-ui-contract]. |
| Full Bookmark Manager | `BookmarksUI` supplies `chrome://bookmarks` resources. Its folder-tree control and item list have selection, keyboard navigation, and commands. This is a full-page manager, not evidence of a ready-made tab-sidebar host. [Manager UI][manager-ui], [folder tree][manager-tree], [items][manager-items], [commands][manager-commands]. |
| Compact bookmarks panel | `BookmarksSidePanelCoordinator` registers a window entry and creates a profile-bound `SidePanelWebUIViewT<BookmarksSidePanelUI>`. Its list drills into folders and sends URL clicks to `BookmarksPageHandler`; the handler observes the native merged service. [Coordinator][bookmark-coordinator], [list][bookmark-list], [handler ownership][bookmark-handler]. |
| Views controls and menus | `BookmarkBarView`, URL buttons, folder menus, `BookmarkMenuDelegate`, and `BookmarkContextMenuController` exist. `BookmarkEditorView` has a native `TreeView` for choosing/editing folders. An editor's folder tree is not a bookmark browsing panel. [Bar][bookmark-bar], [menu][bookmark-menu], [context commands][bookmark-context], [editor][bookmark-editor]. |
| Helium availability versus visibility | Helium defaults `kShowBookmarkBar` to false and enables the additional bar-visibility settings. It retains bookmarks actions while removing unrelated toolbar actions. The ungoogled patch hides the “open side panel” item in the All Bookmarks submenu. None of these changes removes the bookmark model or compact view. [Bar default][helium-bar], [visibility settings][helium-bar-settings], [actions][helium-actions], [hidden submenu item][hidden-bookmark-entry]. |

**Inference.** The inspected model, manager, menu, and compact-view paths can share native bookmark records. A second bookmark database is not a prerequisite. This does not establish that a view can move between hosts unchanged. [Factory][bookmark-factory], [merged factory][merged-factory], [handler][bookmark-handler].

## Profile and incognito ownership

- **Source fact.** Regular incognito bookmark access redirects to the original profile's model and merged service. Bookmarks are shared with that regular profile, rather than isolated per incognito window. [Bookmark factory][bookmark-factory], [merged factory][merged-factory].
- **Source fact.** History uses a different contract. `HistoryServiceFactory` redirects regular off-record access to the original service; it also distinguishes explicit access from automatic access under the saving-history policy. `HistoryTabHelper` returns no service for off-record browsing, so that access contract does not mean incognito visits are recorded. [History factory][history-factory], [visit guard][history-recording].
- **Source fact.** `ShowHistory` refuses the full history UI in off-record mode and shows the native disclaimer dialog. The cluster panel excludes regular-primary incognito and guest sessions, and its service factory uses `kOriginalOnly` for regular profiles. [History command][history-command], [cluster eligibility][cluster-eligibility], [cluster factory][cluster-factory].
- **Inference.** Preserve those separate native boundaries. Neither an isolated incognito bookmark store nor automatic exposure of the redirected regular history service follows from native reuse. A panel-assigned profile would raise ownership questions beyond the existing window-profile binding. [Bookmark coordinator][bookmark-coordinator], [history coordinator][history-coordinator].

## Bounded history candidate inventory

| Candidate | Actual UI versus service availability |
| --- | --- |
| Full `chrome://history` | A real full-page History WebUI with list rows and `BrowsingHistoryHandler`. The handler queries `HistoryService` through explicit access and `BrowsingHistoryService`. Helium retains this path; it changes the synced-device presentation and disables Google-backed web-history integration. A history service alone is not an embeddable UI. [UI][history-ui], [rows][history-row-template], [queries][history-handler], [Helium device changes][helium-history-devices], [Google integration changes][helium-history-google]. |
| History-cluster / Journeys panel | A real compact view with a search field and `<history-clusters>`, plus its coordinator and WebUI. Registration requires eligible, enabled-and-visible Journeys and absence of the by-date variant. Helium changes Journeys' desktop default to disabled. Retained source is not default-on availability. [Template][cluster-template], [coordinator][cluster-coordinator], [eligibility][cluster-eligibility], [registration][entry-registration], [Helium default][helium-journeys]. |
| New by-date history panel | `HistorySidePanelCoordinator::IsSupported` checks `kByDateHistoryInSidePanel`, whose default is disabled. Its WebUI still requires an enabled Journeys service, despite always registering the ordinary history handler. The target's app template contains no rendered content. This is a shell, not a completed by-date list to reuse. [Coordinator][history-coordinator], [default][history-feature], [WebUI gate][history-panel-ui], [template][history-template]. |

**Inference.** Full history provides real UI evidence, and the older cluster panel provides compact-view evidence. Neither proves an independent right-edge organization-panel presentation. The empty by-date shell materially limits that candidate at this revision. No history inclusion decision follows.

## Opening behavior, active page, focus, and commands

| Entry point | Owning native behavior |
| --- | --- |
| Bookmark bar/menu URL | The bar passes event dispositions to its controller; the menu delegates to `OpenAllIfAllowed`. Plain opening uses current-tab disposition. Middle/Ctrl, or Command on macOS, requests a background tab; adding Shift requests a foreground tab. Shift alone requests a window. Alt requests save; Ctrl/Command+Alt without Shift requests a split. These are the target's mappings, not newly selected defaults. [Bar][bookmark-bar], [menu][bookmark-menu], [mapping][dispositions], [current-tab default][disposition-contract]. |
| Folder menus and open-all | Normal folder interaction exposes a menu. Menu subfolders can trigger open-all through middle click or the source's Command event flag. `GetURLsToOpen` includes immediate URL children only. Context commands cover tabs, windows, incognito, native groups, and split views, subject to eligibility. Later URLs normally open in background tabs after the initial disposition. [Menu handling][bookmark-menu], [folder traversal][bookmark-open], [context commands][bookmark-context]. |
| Compact bookmark list | In browsing mode, folder clicks drill into children and focus the first rendered child. Left/middle URL clicks forward mouse/modifier flags to the native disposition helper and open in the window browser. Plain URL opening targets the current tab. The row handler also blurs its event target. [Row handling][bookmark-list], [event forwarding](https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/resources/side_panel/bookmarks/power_bookmark_row_item.ts#L158-L179), [opening handler][bookmark-handler], [mapping][dispositions]. |
| Full Bookmark Manager | Single click selects; modifier click changes selection. Double click/Enter opens a URL in a new foreground tab, or selects a folder. Middle click opens a URL in a background tab; Shift+middle uses the foreground opening command. Middle-clicking a folder does not open it. Bulk expansion includes immediate URL children. [Item handlers][manager-items], [manager commands][manager-commands]. |
| Full history list | Each ordinary history result is an anchor to its URL. Click and auxiliary-click handlers record metrics without canceling navigation. It inherits browser hyperlink behavior; these handlers add no organization-panel destination. [Anchor][history-row-template], [click handlers][history-row]. |
| Cluster history visits | The visit control prevents default navigation and forwards modifiers to `OpenHistoryUrl`. That handler chooses current-tab opening in the cluster side panel and a new foreground tab in full History WebUI. “Open visits in group” opens background tabs, filters illegal visits, and caps the group operation at 32 visits. [Visit control][cluster-visit], [handler][cluster-handler]. |

**Source fact.** Native open-all prompts at the helper's threshold; Helium adds an enabled-default flag described as confirmation for 15 or more bookmarks. Disabling it bypasses that helper's prompt. Bookmark Manager has its own WebUI confirmation path; the Helium patch does not alter that path. [Native helper][bookmark-open], [Helium flag][helium-open-confirmation], [manager confirmation][manager-commands].
**Source fact.** Native foreground/background dispositions already set tab activation. The navigator removes `ADD_ACTIVE` for background tabs and adds it for foreground tabs; an empty browser can promote current/background opening to foreground. [Navigator][navigator].
**Source fact.** The bookmarks handler's “bookmark current tab in folder” uses the window browser's active page and native model. Opening the side panel does not create a separate active page. On first showing, its coordinator requests content focus. Closing returns focus to the active page when stored focus lies within the side panel. [Current-page action][bookmark-handler], [show/close focus][coordinator].
**Source fact.** Native commands already cover bookmark current/all tabs, bar visibility, manager, bookmarks side panel, full history, and cluster side panel. Helium's configurable-shortcut catalogue includes current/all bookmarks, bar, manager, bookmarks-bar focus, and full history. Those catalogue entries do not supply organization-panel navigation. [Browser command dispatch][browser-commands], [Helium catalogue][helium-shortcuts].
**Runtime uncertainty.** Foreground activation, DOM blur, and explicit close-focus requests are distinct. The sources do not prove a universal “entry opening always returns keyboard focus to the page” contract in a new host.
**New question.** Native dispositions settle tab/window/incognito intent. A destination tab panel, whether an explicit destination yields to URL routing, and whether opening returns the viewed sidebar to tabs exceed those contracts. [Native opening][bookmark-open], [agreed routing scope][direction].

## Independent extension coexistence and host constraints

- **Source fact.** `BrowserView` owns one `SidePanel` and separate horizontal/vertical tab regions. The vertical region owns tab controls and surrounding buttons; the base region initializes/resets the tab collection view and resolves focused tab indices through that view. Helium changes the side-panel minimum content width to 200, plus its host border. [Browser ownership][browser-view], [vertical region][vertical-region], [base region][base-region], [Helium sizing][helium-side-host].
- **Source fact.** A `SidePanelEntry` obtains a single owned content view or transfers its cached view. The registry assigns browser/tab scope, rejects duplicate keys, and clears cached views. Its scope must outlive entry content. [Entry contract][entry-contract], [content transfer][entry-content], [registry][registry].
- **Source fact.** `PopulateSidePanel` removes/caches the prior view, adds one replacement, sets its current key, and sends shown/hidden notifications. Bookmarks and extensions use that same entry system. Extension content uses an extension view host, with browser- or tab-scoped ownership. [Replacement][coordinator], [bookmarks entry][bookmark-coordinator], [extension entry][extension-coordinator], [extension content][extension-view].
- **Source fact.** Bookmark content binds the window profile and browser interface. `SidePanelWebUIView` installs the wrapper host, marks its WebContents as side-panel content, wires browser/tab identity, and calls `WasShown` when added. Generic wrapper URL opening forwards to its host; the base host implementation returns null. A full-page anchor-based view therefore has host dependencies beyond its data service. [Bookmarks binding][bookmark-coordinator], [WebView lifetime][side-webview], [wrapper navigation][wrapper].
- **Source fact.** Tab changes choose a contextual entry before a window entry, otherwise close the side panel. Deregistering a visible entry closes it. Independent availability must preserve this existing extension lifecycle rather than assume its content stays fixed through every active-tab change. [Tab-change handling][coordinator], [deregistration][registry].

**Bounded inference.** Existing tab-region and side-panel ownership are separate, making coexistence a plausible investigation. The current bookmarks/history entry cannot simply share the current side-panel slot with an open extension entry. Scope, view ownership, visibility notifications, focus, and profile binding remain concrete reuse constraints. These findings select no hosting design and prove no unchanged embedding or cross-platform guarantee.

## Sidebery references where native organization panels are absent

| Behavior | Exact source fact and dependencies; no adoption implied |
| --- | --- |
| Panel order | Actual panel order follows the saved `nav` sequence, skipping utility entries. Fresh configuration has one tabs panel and utility buttons, with no bookmarks/history panel and no fixed bookmarks-left/history-right order. [Order][side-order], [fresh configuration][side-config]. |
| Bookmarks configuration | Defaults are root `root________`, tree view, `lockedPanel=false`, `tempMode=false`, `skipOnSwitching=false`, and `autoConvert=false`. Its unfiltered alternative “history” view sorts bookmarks by date added; it is not browsing history. Loading requires bookmark permission and reads Firefox's bookmark tree. [Defaults][side-panel-defaults], [alternate view][side-bookmark-view], [load][side-bookmarks]. |
| Non-tab switching | `switchToPanel` activates sidebar content. Its tab creation/last-active-tab branches run only for tab panels, so opening bookmarks/history leaves the active page unchanged in this path. Last-tab activation defaults true; a global active pin suppresses that tab change. [Switch][side-switch], [defaults][side-settings]. |
| Shared global pins | Placement defaults to `top`; `top`, `left`, and `right` render outside the panel-content box. They therefore remain visible beside non-tab content and read the same global pinned IDs. `panel` placement is a separate configuration. [Settings][side-settings], [layout][side-layout], [IDs][side-pins]. |
| Return to tabs | Leaving a tab panel records `prevTabsPanelId`. The recent-tab-panel helper falls back to the active tab's panel, then the first tabs panel. A non-tab panel in `tempMode` returns after 300 ms on mouse leave when search is inactive; bookmark `tempMode` defaults false. A newly activated ordinary tab can switch the sidebar back, with `switchPanelAfterSwitchingTab='always'`, unless locks/subpanel state block it. [Remember/fallback][side-switch], [helper][side-recent], [temporary return][side-leave], [defaults][side-panel-defaults], [activation guards][side-tab-activation], [setting][side-settings]. |
| Bookmark opening | Left click defaults to `open_in_act`; middle click defaults to `open_in_new`. Both activation flags and middle-click removal default false. New tabs target the recent tabs panel, with `default` position using its placement helpers. Existing-open-tab activation and highlighting default false. Ctrl/Shift participate in selection instead of the native URL disposition mapping. [Defaults][side-settings], [mouse handler][side-bookmark-node], [destination/opening][side-bookmark-open]. |
| Folder opening | Left click expands/collapses. Middle click opens the folder into the recent tab panel and activates the first opened item. Folder traversal includes descendants and represents folder structure in opened items, unlike Chromium's immediate-URL-child open-all. [Folder handlers][side-bookmark-node], [traversal][side-bookmark-open]. |

**Runtime uncertainty.** No source trace here verifies queued routing, focus after opening, mouse-leave timing, or restored non-tab content in a running browser.

## Actual choices for the parent

1. Whether history joins the first release, and whether bookmarks/history have horizontal-mode access. Bookmarks-left and bounded one-step swipes are already required. [Direction][direction].
2. While bookmarks or optional history is viewed, whether global pins remain visible and whether selecting/activating a tab returns the viewed sidebar to its tab panel. Native side-panel focus behavior and Sidebery's separate state provide evidence, not a decision. [Native host][coordinator], [Sidebery switching][side-switch], [activation][side-tab-activation].
3. Which tab panel receives newly opened entries, and how explicit destinations interact with already-agreed URL routing. Current-tab opening and native window/incognito dispositions should retain their entry point's native meaning. [Native opening][bookmark-open], [Sidebery destinations][side-bookmark-open], [routing scope][direction].
4. Which added viewed-panel and last-tab-panel state must survive restoration. Native side-panel view caching and Sidebery's saved active-panel ID are evidence of separate lifetimes, not a restoration guarantee. Existing view behavior remains the reuse baseline. [Caching][entry-content], [active-panel save][side-switch], [restoration scope][direction].

The action consequence is already settled: expose every in-scope action to configurable bindings, preserve native bindings, and leave added actions unassigned. Profile assignment remains feasibility only. [Direction][direction].
[direction]: https://github.com/mia-riezebos/helium/blob/1e361f9e3492970a715666a15e6bb7ab218f7df4/DIRECTION.md
[context]: https://github.com/mia-riezebos/helium/blob/1e361f9e3492970a715666a15e6bb7ab218f7df4/CONTEXT.md
[version-target]: https://github.com/mia-riezebos/helium/blob/1fe0fe984484020aae958886adc30618a6ac6dc0/chromium_version.txt
[version-source]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/VERSION
[series]: https://github.com/mia-riezebos/helium/blob/1fe0fe984484020aae958886adc30618a6ac6dc0/patches/series
[downloads]: https://github.com/mia-riezebos/helium/blob/1fe0fe984484020aae958886adc30618a6ac6dc0/downloads.ini
[bookmark-model]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/components/bookmarks/browser/bookmark_model.h
[bookmark-node]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/components/bookmarks/browser/bookmark_node.h
[bookmark-factory]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/bookmarks/bookmark_model_factory.cc#L32-L104
[bookmark-storage]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/components/bookmarks/browser/bookmark_storage.cc#L209-L305
[merged-service]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/bookmarks/bookmark_merged_surface_service.cc
[merged-factory]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/bookmarks/bookmark_merged_surface_service_factory.cc#L17-L62
[bookmark-ui-contract]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/webui/side_panel/bookmarks/bookmarks_side_panel_ui.h#L36-L50
[manager-ui]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/webui/bookmarks/bookmarks_ui.cc
[manager-tree]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/resources/bookmarks/folder_node.ts
[manager-items]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/resources/bookmarks/item.ts#L213-L270
[manager-commands]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/resources/bookmarks/command_manager.ts
[bookmark-coordinator]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/side_panel/bookmarks/bookmarks_side_panel_coordinator.cc#L56-L85
[bookmark-list]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/resources/side_panel/bookmarks/power_bookmarks_list.ts#L1070-L1114
[bookmark-handler]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/webui/side_panel/bookmarks/bookmarks_page_handler.cc
[bookmark-bar]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/bookmarks/bookmark_bar_view.cc#L1397-L1415
[bookmark-menu]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/bookmarks/bookmark_menu_delegate.cc#L412-L496
[bookmark-context]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/bookmarks/bookmark_context_menu_controller.cc
[bookmark-editor]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/bookmarks/bookmark_editor_view.cc
[bookmark-open]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/bookmarks/bookmark_utils_desktop.cc#L95-L425
[dispositions]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/ui/base/window_open_disposition_utils.cc#L12-L49
[disposition-contract]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/ui/base/window_open_disposition_utils.h#L15-L34
[navigator]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/navigator/browser_navigator.cc#L420-L460
[browser-commands]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/browser_command_controller.cc
[helium-bar]: https://github.com/mia-riezebos/helium/blob/1fe0fe984484020aae958886adc30618a6ac6dc0/patches/helium/core/disable-bookmarks-bar.patch
[helium-bar-settings]: https://github.com/mia-riezebos/helium/blob/1fe0fe984484020aae958886adc30618a6ac6dc0/patches/helium/core/enable-bookmark-bar-settings.patch
[helium-actions]: https://github.com/mia-riezebos/helium/blob/1fe0fe984484020aae958886adc30618a6ac6dc0/patches/helium/ui/remove-dead-toolbar-actions.patch#L20-L69
[helium-side-host]: https://github.com/mia-riezebos/helium/blob/1fe0fe984484020aae958886adc30618a6ac6dc0/patches/helium/ui/side-panel.patch#L317-L323
[hidden-bookmark-entry]: https://github.com/mia-riezebos/helium/blob/1fe0fe984484020aae958886adc30618a6ac6dc0/patches/ungoogled-chromium/remove-uneeded-ui.patch#L539-L555
[helium-open-confirmation]: https://github.com/mia-riezebos/helium/blob/1fe0fe984484020aae958886adc30618a6ac6dc0/patches/helium/core/bookmark-open-all-confirmation-flag.patch
[helium-shortcuts]: https://github.com/mia-riezebos/helium/blob/1fe0fe984484020aae958886adc30618a6ac6dc0/patches/helium/core/custom-keyboard-shortcuts.patch#L278-L289
[history-factory]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/history/history_service_factory.cc
[history-recording]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/history/history_tab_helper.cc#L645-L653
[history-command]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/chrome_pages.cc#L337-L361
[history-ui]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/webui/history/history_ui.cc
[history-handler]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/webui/history/browsing_history_handler.cc#L508-L517
[history-row-template]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/resources/history/history_item.html.ts#L41-L59
[history-row]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/resources/history/history_item.ts#L369-L388
[helium-history-devices]: https://github.com/mia-riezebos/helium/blob/1fe0fe984484020aae958886adc30618a6ac6dc0/patches/helium/ui/show-tabs-from-non-account-backends.patch
[helium-history-google]: https://github.com/mia-riezebos/helium/blob/1fe0fe984484020aae958886adc30618a6ac6dc0/patches/helium/core/sync/disable-google-backed-sync-integrations.patch
[cluster-template]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/resources/side_panel/history_clusters/app.html.ts#L12-L56
[cluster-coordinator]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/side_panel/history_clusters/history_clusters_side_panel_coordinator.cc
[cluster-eligibility]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/side_panel/history_clusters/history_clusters_side_panel_utils.cc#L15-L23
[cluster-factory]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/history_clusters/history_clusters_service_factory.cc#L59-L74
[entry-registration]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/side_panel/side_panel_helper.cc#L34-L61
[cluster-visit]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/ui/webui/resources/cr_components/history_clusters/url_visit.ts#L129-L241
[cluster-handler]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/webui/history_clusters/history_clusters_handler.cc
[helium-journeys]: https://github.com/mia-riezebos/helium/blob/1fe0fe984484020aae958886adc30618a6ac6dc0/patches/helium/core/disable-history-clusters.patch
[history-coordinator]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/side_panel/history/history_side_panel_coordinator.cc
[history-feature]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/ui_features.cc#L373
[history-panel-ui]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/webui/side_panel/history/history_side_panel_ui.cc
[history-template]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/resources/side_panel/history/app.html.ts#L9-L13
[browser-view]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/frame/browser_view.cc#L974-L996
[vertical-region]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/frame/vertical_tab_strip_region_view.cc
[base-region]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/frame/base_tab_strip_region_view.cc
[entry-contract]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/side_panel/side_panel_entry.h#L142-L170
[entry-content]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/side_panel/side_panel_entry.cc#L65-L79
[registry]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/side_panel/side_panel_registry.cc
[coordinator]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/side_panel/side_panel_coordinator.cc#L215-L405
[side-webview]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/side_panel/side_panel_web_ui_view.cc#L32-L103
[wrapper]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/webui/top_chrome/webui_contents_wrapper.cc
[extension-coordinator]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/extensions/extension_side_panel_coordinator.cc
[extension-view]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/extensions/extension_side_panel_coordinator_views.cc#L51-L65
[side-order]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/sidebar.fg.ts#L872-L907
[side-config]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/sidebar-config.ts#L66-L72
[side-panel-defaults]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/defaults/panels.ts#L5-L20
[side-bookmark-view]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/sidebar/components/panel.bookmarks.vue
[side-bookmarks]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/bookmarks.fg.ts#L331-L391
[side-switch]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/sidebar.fg.ts#L1205-L1323
[side-settings]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/defaults/settings.ts
[side-layout]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/sidebar/sidebar.vue#L74-L121
[side-pins]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/sidebar/components/bar.pinned-tabs.vue#L25-L29
[side-recent]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/sidebar.fg.ts#L2762-L2775
[side-leave]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/sidebar/sidebar.vue#L347-L365
[side-tab-activation]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/tabs.fg.handlers.ts#L1763-L1776
[side-bookmark-node]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/components/bookmark-node.vue#L99-L300
[side-bookmark-open]: https://github.com/mbnuqw/sidebery/blob/ca1e2b386288a0e8dd95d8c9f902e2fe4b5ead18/src/services/bookmarks.fg.ts#L590-L757
