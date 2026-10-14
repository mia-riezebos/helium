# Profile-per-panel feasibility and isolation limits

Research for [Assess profile-per-panel feasibility and isolation limits](https://github.com/mia-riezebos/helium/issues/4).
Checked 2026-10-14. This is source investigation, without builds or runtime validation.

## Answer

**Profile-per-panel remains plausible in principle, but one cross-platform window with full Chromium profile isolation is unproven.**
The targeted desktop window contract requires one fixed profile and the same profile for every contained tab.
An unchanged native `Browser`, `TabStripModel`, and `BrowserView` therefore do not supply the stretch goal.
Multiple loaded profiles already exist in Chromium, but that establishes coexistence, not shared-window support.
[Desktop contract][window-contract], [browser ownership][browser-ownership], [view ownership][view-ownership], [loaded profiles][loaded-profiles].

**Architectural inference.** Different profile data can plausibly remain separate when their presentation changes.
However, that does not establish correct window ownership, command dispatch, extension access, dialogs, or restoration.
The inspected services preserve profile ownership, while the inspected desktop controls depend on their owning browser.
This is a conditional feasibility judgment, not a selected architecture or an isolation guarantee.
[Profile services][profile-services], [window features][window-features], [extension hosts][extension-hosts], [session ownership][session-owner].

The core panel design has semantic constraints, but this research supplies no reason to require profiles for ordinary panels.
A panel remains an organizational collection under the agreed [Direction](DIRECTION.md) and [Terminology](CONTEXT.md).
Profile reassignment and profile isolation are separate concerns from showing a different collection.
Cross-profile shared-pin behavior remains unspecified. This report does not choose it.

## Revision and evidence boundary

- The isolated checkout starts at Helium fork commit `d2ff2d2ad38cba5fbb91f0a63ad691eaabbcc172`.
  Its [`chromium_version.txt`][helium-version] targets **155.0.8059.79**.
- Chromium's [tag reference][chromium-tag] resolves to **`a49e2e3d4addfd84e765b52afe230bedb81096db`**.
  Chromium source citations below use that immutable commit, not a moving branch.
  The [release commit][chromium-commit] records `refs/branch-heads/8059@{#1239}`.
- The earlier [`helium-feasibility.md`](helium-feasibility.md) cites `8e88c7782761350ce57319a78d8cca97b8f3a901` as Chromium main.
  Those findings were leads only. They are not targeted-version evidence here.
  No current-main change or moving-head claim contributes to this report's conclusion.
- Chromium evidence describes source at the release commit.
  Helium patch evidence describes the checked-out patch files separately.
  There is no assembled patched-source build or evidence about a running Helium binary.
- I fetched and read the cited Chromium files from the immutable GitHub mirror.
  I cross-checked the tag SHA through Chromium Gitiles and GitHub's tag API.
  Line links correspond to those fetched files. Local patch citations use the checkout commit.

**Confirmed** below means the cited source states a contract or contains the described behavior.
**Inference** means an architectural consequence drawn from that evidence.
**Unproven** means source inspection does not answer the runtime question.

## Confirmed profile and tab ownership

1. **A profile is broader than website storage.** `Profile` inherits `content::BrowserContext` and exposes its preferences.
   `ProfileImpl` retains a profile path and creates browser-context services through the dependency manager.
   `BrowserContext` owns storage partitions; partitions expose network contexts, cookies, and persistent website state.
   These are different ownership levels.
   [Profile declaration][profile-declaration], [preferences][profile-prefs], [profile construction][profile-construction], [service creation][profile-services], [partitions][context-partitions], [partition state][partition-state].

   **Inference.** A separate storage partition alone does not reproduce full profile isolation.
   It does not independently establish the profile preferences, extensions, permission services, or session owner described below.
   This distinction matters even if different website accounts can coexist through narrower storage isolation.

   **Isolation limit.** Chromium also has browser-wide state and OS permission integration.
   Command initialization accesses browser-wide local state.
   Where enabled, permission setup accesses the system geolocation permission manager.
   Full profile isolation here means Chromium's existing browsing identities, not a new OS process security boundary.
   [Browser-wide state][browser-wide-state], [system permission integration][system-permission].

2. **The desktop window contract is explicit.** `BrowserWindowInterface::GetProfile()` documents a non-null, lifetime-fixed profile.
   Its contract gives every contained desktop tab the window's `Profile`/`BrowserContext`.
   The comment mentions mobile exceptions, then requires the desktop guarantees for experimental desktop Android.
   Those mobile exceptions do not establish Windows, macOS, or Linux support.
   [Window contract][window-contract].

3. **Concrete browser and view ownership matches that contract.** `Browser` has a constant profile pointer and owns its tab model.
   Its constructor supplies that profile to `TabStripModel` and holds a regular-profile keep-alive.
   `BrowserView` retains a constant owning `BrowserWindowInterface` pointer and owns its `BrowserWidget`.
   Its constructor observes that browser's model and constructs its toolbar using the same browser.
   The non-Android window factory creates a `Browser` through this path.
   [Browser members][browser-ownership], [construction][browser-construction], [view members][view-ownership], [view construction][view-construction], [desktop creation][desktop-creation].

4. **Native tab movement and replacement preserve important boundaries.** Tab dragging rejects a destination browser with another profile.
   `TabModel::GetProfile()` derives identity from its contents' browser context.
   Replacing discarded contents checks that the old and new contexts match.
   `TabStripModel` also resolves reading-list and group services using its own profile.
   [Drag rejection][drag-profile], [tab identity][tab-profile], [replacement check][tab-replacement], [reading-list owner][reading-owner], [group owner][group-owner].

   **Evidence limit.** I did not find a profile-equality check in the inspected tab insertion chain.
   `InsertWebContentsAt`, `InsertDetachedTabAt`, and `InsertTabAtImpl` check other conditions.
   The delegate's `WillAddWebContents` attaches tab helpers.
   This is not proof that every mixed-profile insertion fails immediately.
   It also does not override the documented desktop contract or establish that insertion is safe.
   [Insertion entry points][tab-insertion], [insertion implementation][tab-insertion-impl], [delegate][tab-delegate].

## Confirmed focus and command routing

`BrowserWindowFeatures::Init` obtains its profile and tab model from one owning browser.
It creates that browser's command controller and, later, its file-dialog controller and extension window controller.
`BrowserCommandController` stores the browser, observes its tab model, and dispatches navigation commands through it.
Its command execution path also checks that model's active tab.
[Feature initialization][window-features], [command wiring][command-wiring], [command construction][command-construction], [dispatch][command-dispatch], [file controller wiring][file-controller-wiring], [extension controller wiring][extension-controller-wiring].

`BrowserView` resolves reserved keyboard commands through its owning browser's command controller and widget focus manager.
On tab activation it stores old focus, changes displayed contents, updates the interface, and restores focus.
The focus-restoration condition differs between macOS and the other desktop platforms.
[Keyboard handling][keyboard-routing], [activation and focus][activation-focus].

**Inference.** Merely displaying another profile's contents does not establish that shortcuts, navigation, or toolbar actions target that profile.
Native window activation and the browser's selected tab are part of the existing routing assumptions.
The macOS focus difference is a concrete reason cross-platform behavior needs separate runtime evidence.
It does not justify a platform-exclusive architecture.

## Confirmed extensions and independent side panel

The desktop extension toolbar obtains `ToolbarActionsModel` from its browser's profile.
That model uses the profile's preferences, extension registry, action dispatcher, and extension system.
[Toolbar binding][extension-toolbar], [model binding][extension-model].

The browser-level `ExtensionSidePanelManager` also takes the browser profile and observes its extension registry.
It registers entries only for extensions with the side-panel API permission.
A separate constructor supports tab-level entries, but it still takes a profile.
The coordinator uses that profile's `SidePanelService`.
[Side-panel manager][extension-manager], [side-panel service binding][extension-coordinator].

`ExtensionViewHostFactory` selects the browser's profile for extension popups.
For a side-panel host, it selects the browser's profile or the tab's owning browser profile.
The side-panel view uses the coordinator profile.
These paths do not accept an arbitrary profile-free extension host.
[Popup and side-panel hosts][extension-hosts], [side-panel view][extension-view].

The extension tab lookup path filters windows to the caller's profile and, when permitted, that profile's incognito context.
It does not enumerate an unrelated regular profile through that path.
This is evidence about the inspected tab API helper, not a complete audit of every extension permission.
[Tab lookup filtering][extension-tab-lookup].

**Inference.** Preserving the independent extension side panel requires more than reserving its screen space.
Its extension identity, host context, window ID, active tab, permissions, and command owner must remain consistent.
The source does not demonstrate that multiple profile-specific extension controls can share one native window safely.
The required ordinary panel UI and the existing extension side panel therefore remain distinct concepts.

## Confirmed permissions, downloads, and dialogs

**Permissions.** `PermissionManagerFactory` creates services for browser contexts and depends on the profile's content-settings map.
The desktop prompt follows its requesting `WebContents` to the tab's owning browser and its location bar.
Its stated tab-switch behavior destroys the prompt while retaining the pending request.
[Permission service][permission-service], [prompt ownership and switching][permission-prompt].

**Downloads.** The download service factory creates a service for the profile context.
The download bubble controller retains its browser and profile and obtains that profile's update service.
Download-to-window lookup filters by the item's profile, with the documented regular/incognito relationship.
Bubble update behavior consults the last active browser in that profile's browser collection.
[Download service][download-service], [download controller and window lookup][download-controller], [activation dependency][download-activation].

**Dialogs.** `Browser::SetAsDelegate` assigns browser-owned contents and modal-dialog delegates to tabs.
The browser's dialog-host methods forward to its window.
The Open File controller captures a profile, model, native parent window, and navigator.
Its asynchronous result updates that profile's selected directory and navigates through the captured owner.
JavaScript tab-modal dialogs also react to contents visibility and pending-dialog state.
[Contents and modal delegates][modal-delegate], [dialog hosts][modal-host], [Open File ownership][file-dialog], [JavaScript dialog visibility][javascript-dialog].

**Inference.** Hiding one profile's presentation is not enough to prove correct prompts, download bubbles, or modal behavior.
Source ownership must also agree with the visible native window and with asynchronous callbacks.
This report does not establish universal isolation for OS-level permissions or native dialogs.

## Confirmed lifecycle and restoration

`BrowserManagerServiceFactory` owns a service for each browser context.
`BrowserManagerService` owns browsers and tracks their activation and close callbacks.
Its shutdown clears those owned browsers.
The browser constructor's keep-alive prevents normal profile deletion while its window exists.
[Manager factory][browser-manager-factory], [manager ownership][browser-manager], [browser keep-alive][browser-construction].

`WebContents::Create` requires contents to die before their browser context.
Its documentation identifies violating that destruction order as a security bug.
`ProfileImpl` performs ordered service shutdown before shutting down storage partitions.
[Contents lifetime][contents-lifetime], [profile shutdown][profile-shutdown].

`SessionServiceFactory` selects original regular profiles for session storage.
`SessionServiceBase` stores commands under that profile's path and excludes browsers with another profile.
`SessionRestoreImpl` works on one profile and holds a profile keep-alive.
It documents and debug-checks a single concurrent restorer for that profile.
Restored browsers receive that profile in their creation parameters.
[Session factory][session-factory], [storage owner][session-owner], [browser filtering][session-filter], [restore lifetime][restore-owner], [restored browser][restore-browser].

**Inference.** One visible window does not provide one shared session or one shared profile lifetime automatically.
Panel visibility, logical browser closure, profile deletion, and application shutdown are different events in these sources.
Nothing inspected demonstrates cross-profile window restoration or crash recovery as one native window.
Incognito's service selection also cannot stand in for separate persistent regular profiles.

## Relevant Helium evidence

Helium's layout patch creates `HeliumLayoutStateController` using the browser profile's preferences.
Its shortcut wiring obtains a profile-specific shortcut service through the browser.
That wiring also refreshes macOS menu accelerators for the active view.
[Layout preferences][helium-layout], [shortcut ownership and platform wiring][helium-shortcuts].

Helium's layout patch removes some upstream vertical-layout restoration fields.
Its lazy-restoration patch changes restoration order and which background tabs load.
These are local patch facts, not evidence of profile multiplexing.
[Layout restoration patch][helium-layout-restore], [lazy restoration patch][helium-lazy-restore].

**Inference.** Existing Helium integration still contributes profile-sensitive state to the window.
The earlier note's shared-host hypothesis is not a demonstrated capability of this checkout.
This research does not select that hypothesis or another profile-panel architecture.

## Constraints on the core panel design

These are conclusions from the evidence, not instructions for changing browser code.

1. **Panel identity cannot mean browsing identity.** Ordinary organization alone does not supply profile isolation.
   The core requirement can stay within Chromium's existing single-profile window contract.
   Optional profile assignment remains a separate stretch goal.
   [Profile boundary][profile-declaration], [desktop contract][window-contract].
2. **Changing a visible collection does not establish profile migration.** Native dragging rejects cross-profile destinations.
   A live tab's context, service owner, and restoration owner cannot be presumed to change with its panel label.
   This applies to routing and movement semantics without selecting cross-profile behavior.
   [Drag boundary][drag-profile], [replacement boundary][tab-replacement], [session filtering][session-filter].
3. **Native window controls have an owner.** A future isolation claim includes commands, extensions, prompts, downloads, dialogs, and lifecycle.
   Core panel organization cannot itself be cited as evidence that these owners switch profiles correctly.
   [Window feature ownership][window-features], [extension host ownership][extension-hosts], [prompt owner][permission-prompt], [download owner][download-controller], [lifetime][contents-lifetime].
4. **Persistence is already profile-scoped.** Preserving organization inside one ordinary profile does not prove multi-profile window restoration.
   The stretch goal cannot borrow that guarantee from core panel acceptance alone.
   [Session storage][session-owner], [restore ownership][restore-owner].

No required core feature needs rejection solely because the profile stretch goal remains unproven.
This report supplies no group-depth, colour, or panel-gesture verdict; those belong to the other investigations.
It also supplies no choice for cross-profile shared pins.

## Questions requiring a future runtime experiment

No experiment ran during this investigation. The following questions remain open for all three required desktop platforms.

1. **Window identity and input.** Can one native window represent multiple profile owners without stale active-browser or active-tab state?
   Do keyboard shortcuts, native menus, navigation, focus restoration, drag targets, and popup windows select the correct owner?
2. **Extension isolation.** Do toolbar actions, extension popups, tab/window API IDs, active-tab grants, and the independent side panel remain profile-correct?
   Can switching profiles leave an extension host or permission attached to the previous owner?
3. **Prompts and asynchronous work.** What happens to permission requests, downloads, file choosers, and modal dialogs during profile changes?
   Do hidden owners retain pending work correctly, and do completed callbacks act on the initiating owner?
4. **Lifecycle and recovery.** What happens when a profile's last logical browser closes, a profile is deleted, or the visible window closes?
   Can restart and crash restoration recover all profile-owned tabs into one window without lifetime violations or duplicate windows?
5. **Actual isolation.** Do website state, preferences, extensions, permission decisions, and session histories remain attached to their original profiles?
   How do platform-specific focus, native menu, dialog, and shutdown behavior affect that result?

These questions now define a bounded future proof. They do not establish an implementation path or authorize implementation.

## Decision handoff

**Gist for the map.** Profile isolation can plausibly survive shared presentation, but Chromium 155 fixes desktop window ownership to one profile.
Cross-platform multi-profile window hosting remains unproven across controls, extensions, dialogs, and restoration.

No new choice for core panel behavior is forced by this investigation.
The now-specifiable sign-off question is whether the implementation-ready design can leave the optional stretch goal explicitly unproven.
If the design promises that goal, the runtime questions above remain evidence prerequisites.
No cross-profile pin decision or platform-exclusive window-tabbing path follows from this report.

[helium-version]: https://github.com/mia-riezebos/helium/blob/d2ff2d2ad38cba5fbb91f0a63ad691eaabbcc172/chromium_version.txt
[chromium-tag]: https://api.github.com/repos/chromium/chromium/git/ref/tags/155.0.8059.79
[chromium-commit]: https://chromium.googlesource.com/chromium/src/+/a49e2e3d4addfd84e765b52afe230bedb81096db
[window-contract]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/browser_window/public/browser_window_interface.h#L83-L96
[browser-ownership]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/browser.h#L265-L284
[view-ownership]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/frame/browser_view.h#L1090-L1094
[loaded-profiles]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/profiles/profile_manager.cc#L971-L979
[profile-declaration]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/profiles/profile.h#L68
[profile-prefs]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/profiles/profile.h#L301-L304
[profile-construction]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/profiles/profile_impl.cc#L475-L482
[profile-services]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/profiles/profile_impl.cc#L1215-L1216
[context-partitions]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/content/public/browser/browser_context.h#L155-L188
[partition-state]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/content/public/browser/storage_partition.h#L87-L134
[browser-construction]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/browser.cc#L267-L299
[view-construction]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/frame/browser_view.cc#L891-L947
[desktop-creation]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/browser_window/internal/create_browser_window_non_android.cc#L140-L146
[drag-profile]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/tabs/dragging/tab_drag_controller.cc#L2990-L2998
[tab-profile]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/tab_model.cc#L223-L225
[tab-replacement]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/tab_model.cc#L444-L452
[reading-owner]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/tab_strip_model.cc#L2273-L2285
[group-owner]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/tab_strip_model.cc#L4633-L4644
[tab-insertion]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/tab_strip_model.cc#L436-L467
[tab-insertion-impl]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/tabs/tab_strip_model.cc#L3758-L3815
[tab-delegate]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/browser_tab_strip_model_delegate.cc#L164-L167
[window-features]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/browser_window/internal/browser_window_features.cc#L246-L254
[command-construction]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/browser_command_controller.cc#L334-L338
[command-wiring]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/browser_window/internal/browser_window_features.cc#L334-L337
[browser-wide-state]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/browser_command_controller.cc#L334-L342
[system-permission]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/permissions/permission_manager_factory.cc#L87-L103
[command-dispatch]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/browser_command_controller.cc#L638-L681
[file-controller-wiring]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/browser_window/internal/browser_window_features.cc#L722-L726
[extension-controller-wiring]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/browser_window/internal/browser_window_features.cc#L823-L837
[keyboard-routing]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/frame/browser_view.cc#L3271-L3279
[activation-focus]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/frame/browser_view.cc#L1968-L2108
[extension-toolbar]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/extensions/extensions_toolbar_desktop.cc#L149-L152
[extension-model]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/toolbar/toolbar_actions_model.cc#L67-L80
[extension-manager]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/extensions/extension_side_panel_manager.cc#L22-L98
[extension-coordinator]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/extensions/extension_side_panel_coordinator.cc#L52-L90
[extension-hosts]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/extensions/extension_view_host_factory.cc#L267-L298
[extension-view]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/extensions/extension_side_panel_coordinator_views.cc#L45-L68
[extension-tab-lookup]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/extensions/extension_tab_util.cc#L632-L669
[permission-service]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/permissions/permission_manager_factory.cc#L208-L243
[permission-prompt]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/views/permissions/permission_prompt_desktop.cc#L25-L82
[download-service]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/download/download_core_service_factory.cc#L29-L60
[download-controller]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/download/bubble/download_bubble_ui_controller.cc#L67-L123
[download-activation]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/download/bubble/download_bubble_ui_controller.cc#L178-L212
[modal-delegate]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/browser.cc#L869-L886
[modal-host]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/browser.cc#L471-L479
[file-dialog]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/browser_select_file_dialog_controller.cc#L34-L104
[javascript-dialog]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/components/javascript_dialogs/tab_modal_dialog_manager.cc#L255-L292
[browser-manager-factory]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/browser_manager_service_factory.cc#L14-L44
[browser-manager]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/ui/browser_manager_service.cc#L42-L108
[contents-lifetime]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/content/public/browser/web_contents.h#L394-L409
[profile-shutdown]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/profiles/profile_impl.cc#L1004-L1021
[session-factory]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/sessions/session_service_factory.cc#L63-L87
[session-owner]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/sessions/session_service_base.cc#L162-L181
[session-filter]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/sessions/session_service_base.cc#L842-L850
[restore-owner]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/sessions/session_restore.cc#L323-L370
[restore-browser]: https://github.com/chromium/chromium/blob/a49e2e3d4addfd84e765b52afe230bedb81096db/chrome/browser/sessions/session_restore.cc#L1274-L1310
[helium-layout]: https://github.com/mia-riezebos/helium/blob/d2ff2d2ad38cba5fbb91f0a63ad691eaabbcc172/patches/helium/ui/layout/core.patch#L483-L489
[helium-shortcuts]: https://github.com/mia-riezebos/helium/blob/d2ff2d2ad38cba5fbb91f0a63ad691eaabbcc172/patches/helium/ui/custom-keyboard-shortcuts-wiring.patch#L25-L130
[helium-layout-restore]: https://github.com/mia-riezebos/helium/blob/d2ff2d2ad38cba5fbb91f0a63ad691eaabbcc172/patches/helium/ui/layout/core.patch#L589-L623
[helium-lazy-restore]: https://github.com/mia-riezebos/helium/blob/d2ff2d2ad38cba5fbb91f0a63ad691eaabbcc172/patches/helium/core/lazy-session-restore.patch#L195-L255
