# Helium feasibility evidence

The current product scope and terminology are in `DIRECTION.md` and `CONTEXT.md`. This note records feasibility findings from the broader initial exploration.

## Confirmed Sidebery configuration

Checked upstream `v5` source on 2026-10-14:

- Panel move rules have an enabled state, URL substring or regular-expression matching, optional container matching, and a top-level-only filter. Panels also have a destination for tabs that no longer match. Rules are evaluated on URL changes for unpinned tabs. These mechanisms establish existing configurable routing; they do not establish a dedicated new-tab-only trigger option. Mia requires configurable routing behavior in the port. [Move-rule implementation](https://github.com/mbnuqw/sidebery/blob/v5/src/services/tabs.fg.move.ts#L742), [URL-change handler](https://github.com/mbnuqw/sidebery/blob/v5/src/services/tabs.fg.handlers.ts#L905), [panel defaults](https://github.com/mbnuqw/sidebery/blob/v5/src/defaults/panels.ts).
- Pin placement supports panel-local placement and global top/left/right positions. Mia selected global pins, visible in every panel. [Settings and option values](https://github.com/mbnuqw/sidebery/blob/v5/src/defaults/settings.ts).
- Horizontal scrolling already has a `switch_panels` action. The settings also include one-switch-per-scroll and wheel accumulation controls. Use these existing behaviors as the reference. [Mouse settings](https://github.com/mbnuqw/sidebery/blob/v5/src/defaults/settings.ts#L229).

Checked 2026-10-14. Scope now requires a near-feature-complete Sidebery-style system built into Helium on Windows, macOS, and Linux. Nested grouping is a minimum requirement. Full profile switching inside one window is a stretch goal. This is source research, with no builds or runtime validation.

## Conclusion

A built-in implementation is plausible. The required scope includes named groups inside groups, with tabs at any level. Helium already modifies Chromium's native vertical-tab UI. Existing tree-tab proposals provide useful references, but neither establishes a near-feature-complete Sidebery port.

## Porting Sidebery itself

Sidebery's source uses TypeScript and Vue, and its repository declares the MIT license. Its own group creation code creates group tabs and assigns parent IDs. Reusing this UI and behavior in a browser-owned sidebar is a credible direction; embedding it and connecting its browser operations still needs a prototype. [Package](https://github.com/mbnuqw/sidebery/blob/v5/package.json), [repository and license](https://github.com/mbnuqw/sidebery), [group creation](https://github.com/mbnuqw/sidebery/blob/v5/src/services/tabs.fg.groups.ts).

An existing Chromium port is a reference, not evidence of full parity. Its migration plan explicitly excludes containers, per-container rules, tab hiding, Firefox theme integration, native context-menu overrides, and full screenshot previews. The Helium fork could provide browser integration for missing operations, but those capabilities need individual implementation and validation. [Chromium port migration plan](https://github.com/7rah/sidebery/blob/manifest-v3/docs/superpowers/plans/2026-03-28-chrome-core-migration.md), [Sidebery API requirements](https://github.com/mbnuqw/sidebery/blob/v5/src/manifest.json).

For the profile stretch goal, keep each profile's `Browser`, `TabStripModel`, services, and `WebContents` together. A shared host that switches complete profile-bound contexts is more credible than mixing profiles inside one Chromium tab model. This is an architectural inference, not a demonstrated cross-platform implementation.

## Existing Helium work

The inspected Helium commit is `e294de285951c27fdaff2ba3df5703b88971084b`, targeting Chromium 155.0.8059.79. Its layout patches modify native `BrowserView` layout and `VerticalTabStripStateController`; `HeliumLayoutStateController` uses profile preferences. This offers an existing native integration point. [Chromium version](https://github.com/imputnet/helium/blob/e294de285951c27fdaff2ba3df5703b88971084b/chromium_version.txt), [vertical layout patch](https://github.com/imputnet/helium/blob/e294de285951c27fdaff2ba3df5703b88971084b/patches/helium/ui/layout/vertical.patch), [layout controller patch](https://github.com/imputnet/helium/blob/e294de285951c27fdaff2ba3df5703b88971084b/patches/helium/ui/layout/core.patch).

Two proposals remain open:

- [PR #1832](https://github.com/imputnet/helium/pull/1832), `ImTheSquid/helium` branch `nested-tabs`, introduces `NestedTabCollection`. Its author describes parent/child tabs, collapse, subtree dragging, session restore, and reopening closed trees. The author states that groups may contain trees but remain top-level. Tabs-under-tabs therefore has a concrete lead; groups-under-groups still needs development in that proposal. The author lists drag limitations and possible crashes. I verified the current PR description and open state; I did not independently review this patch's implementation.
- [PR #1468](https://github.com/imputnet/helium/pull/1468) proposes indentation, collapse, reparenting, and basic tree-state restoration. Its description explicitly says reopening closed tabs loses tree position and identifies grouped/pinned/split interactions for review. These are author-reported capabilities, not runtime findings.

## The isolation boundary

These concepts solve different problems:

| Concept | Boundary |
| --- | --- |
| Workspace or panel | Organizes tabs. Organization alone creates no storage or service isolation. |
| Storage container | Can separate website state through storage partitions, while retaining the same profile-level services and preferences. |
| Full Chromium profile | A distinct `Profile`/`BrowserContext`, profile directory, preferences, and profile-keyed services. |

`Profile` inherits `BrowserContext` and exposes its data path and preferences. Chromium's profile architecture documents profile-keyed services and their startup/shutdown dependencies. `BrowserContext` can own multiple storage partitions; partitions expose cookies, network contexts, IndexedDB, service workers, and cache storage. A partition therefore does not by itself reproduce a full profile. [Profile declaration](https://github.com/chromium/chromium/blob/8e88c7782761350ce57319a78d8cca97b8f3a901/chrome/browser/profiles/profile.h), [profile architecture](https://www.chromium.org/developers/design-documents/profile-architecture/), [BrowserContext](https://github.com/chromium/chromium/blob/8e88c7782761350ce57319a78d8cca97b8f3a901/content/public/browser/browser_context.h), [StoragePartition](https://github.com/chromium/chromium/blob/8e88c7782761350ce57319a78d8cca97b8f3a901/content/public/browser/storage_partition.h).

## Chromium's window constraints

Current Chromium main, inspected at `8e88c7782761350ce57319a78d8cca97b8f3a901`, binds `Browser` to a constant profile pointer and its own model. Its constructor passes that profile into `TabStripModel`; `BrowserView` also has a constant owning browser pointer. This is more than a sidebar rendering change. [Browser ownership](https://github.com/chromium/chromium/blob/8e88c7782761350ce57319a78d8cca97b8f3a901/chrome/browser/ui/browser.h#L258), [construction](https://github.com/chromium/chromium/blob/8e88c7782761350ce57319a78d8cca97b8f3a901/chrome/browser/ui/browser.cc#L265), [BrowserView ownership](https://github.com/chromium/chromium/blob/8e88c7782761350ce57319a78d8cca97b8f3a901/chrome/browser/ui/views/frame/browser_view.h#L1094).

Tab dragging explicitly rejects another browser with a different profile. `WebContents` receives its context at creation and must die before that context. The model also uses its own profile for services such as reading lists and tab groups. These support preserving separate models; they do not prove mixed-profile insertion is universally guarded. [Drag restriction](https://github.com/chromium/chromium/blob/8e88c7782761350ce57319a78d8cca97b8f3a901/chrome/browser/ui/views/tabs/dragging/tab_drag_controller.cc#L3103), [WebContents lifetime](https://github.com/chromium/chromium/blob/8e88c7782761350ce57319a78d8cca97b8f3a901/content/public/browser/web_contents.h#L159), [model](https://github.com/chromium/chromium/blob/8e88c7782761350ce57319a78d8cca97b8f3a901/chrome/browser/ui/tabs/tab_strip_model.cc).

## Plurium precedent, excluded implementation path

I inspected Plurium commit `af4888575f7bc5cf9257dacb2678495b13d07c7b`. Patch 0001 calls AppKit `addTabbedWindow:ordered:`. Patch 0004 retains each window's browser/model and stores presentation entries as `(window, tab)` pairs. Activation selects the owning model's tab and then its native window. Moving across profiles reopens the URL in the target profile and closes the source. Patch 0009 renders this presentation vertically. [Patch 0001](https://github.com/indapublic/plurium/blob/af4888575f7bc5cf9257dacb2678495b13d07c7b/patches/0001-Mac-join-profile-windows-into-one-native-window-tab-.patch), [patch 0004](https://github.com/indapublic/plurium/blob/af4888575f7bc5cf9257dacb2678495b13d07c7b/patches/0004-Mac-one-tab-strip-with-the-tabs-of-all-profiles.patch), [patch 0009](https://github.com/indapublic/plurium/blob/af4888575f7bc5cf9257dacb2678495b13d07c7b/patches/0009-Mac-the-unified-strip-also-works-with-vertical-tabs.patch).

**Reject Plurium's architecture for this project.** It depends on macOS window tabbing and retains multiple native windows internally. Keep only its useful separation between presentation and profile-bound models.

## Untested assumptions and next proof

Cross-platform hosting of multiple complete browser contexts remains unproven. Focus, command routing, extension UI, permissions, downloads, modal dialogs, shutdown, and session restoration must follow the selected profile. Profile isolation is not separate-OS-process isolation.

The next proof should embed Sidebery's existing sidebar and connect real Helium tabs, recursive grouping, and persisted panel trees. Verify relaunch and closed-tree restoration. Then prove horizontal panel gestures without interfering with page navigation. Use the native tree-tab proposals as references for browser integration. Defer profile switching to a separate cross-platform hosting experiment after the required model works. No time or effort estimate is justified by this research alone.
