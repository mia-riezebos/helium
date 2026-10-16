# Helium tab organization

## Purpose

Build Sidebery-style tab organization into Helium's native tab toolbar and vertical sidebar.
The first release targets the agreed features below rather than complete Sidebery parity.

## Required behavior

- Named groups can contain other groups and tabs recursively.
- Support tree-style tabs. Opening a link in a new tab automatically makes it a child of its source tab.
- Groups contain whole tab trees and other groups. Tabs contain only child tabs; a tree stays within one group and tab panel.
- Ordinary new-tab commands create tabs without a parent. Provide an explicit New child tab action, preserving native placement and focus where compatible.
- Tab panels hold separate tab collections. Horizontal scrolling over the tab toolbar switches panels.
- Panel order is configurable.
- Panel navigation changes one panel per swipe, ignores momentum, and stops at either end.
- Support both horizontal and vertical tab modes, with vertical mode driving the design.
- Horizontal mode shows nested groups in one row with the usual inline expand/collapse controls.
- Route tabs automatically to tab panels using configurable domain or URL-path rules. Routing behavior remains configurable.
- Routing rules default to moving a tab alone, without its group. A rule can instead preserve its full group-name path.
- Group-preserving routing reuses matching destination groups and creates missing ancestors, such as `Clients > Acme`.
- Domain rules include subdomains by default. Path rules match prefixes. An exact host filter narrows domain matching.
- Pinned tabs appear in every tab panel.
- Tabs and groups have configurable colours, shown as a small stripe on the left.
- An explicit tab colour wins over group colours. Other grouped tabs use their group colour.
- Expose every in-scope action to keybinding configuration. Added actions start unassigned; choose their defaults through use.
- Preserve existing Chromium and Helium keybindings.
- Allow middle-click and double-click remapping on tabs and group headers, preserving native defaults. Defer extra mouse gestures.
- Full Sidebery interaction parity remains outside scope.
- Preserve organization across browser restarts and session restoration.

## Constraints

- Windows, macOS, and Linux are required platforms.
- Integrate into the existing native tab toolbar. Keep the extension side panel available independently.
- Browser changes are required; a standalone Chrome extension cannot deliver the intended integration.
- Use cross-platform browser architecture. Platform-exclusive window tabbing is excluded.
- Reuse and extend Chromium and Helium's existing native models, UI, layouts, shortcuts, and behavior. Add only required capability gaps; minimize replacement code and duplicated state to reduce future maintenance.
- Use the existing native implementation appropriate to each tab mode, sharing provided components wherever possible. A common UI implementation is not a separate requirement.
- Existing Chromium and Helium behavior guides unspecified details. Use Sidebery as the reference for required capabilities they lack; check its implementation before asking product questions.

## Stretch goal

Allow each tab panel to use a Chromium profile so different website accounts can coexist in one window.
Tab panels must work without profile assignment. Profile isolation must remain intact when this feature is added.

## Deferred

Bookmarks, history and other browser-view panels are outside the current release scope.
[Define bookmarks and optional sidebar panels](https://github.com/mia-riezebos/helium/issues/10) retains their investigation and provisional future choices.
Existing native bookmark/history access remains available.

## Current state

The Chromium capability, Sidebery behavior, and profile feasibility investigations are complete and indexed in the [Native tab panels and nested groups](https://github.com/mia-riezebos/helium/issues/1) map.
The [native integration decision](https://github.com/mia-riezebos/helium/issues/5) adopts existing Chromium and Helium functionality as the baseline.
[Define panel, group, routing and shared-pin behavior](https://github.com/mia-riezebos/helium/issues/6) settles colours, routing and organization defaults.
[Choose keyboard, mouse and horizontal-tab interactions](https://github.com/mia-riezebos/helium/issues/7) settles bindings, gestures and horizontal group presentation.
Required tab trees supersede the earlier tabs-as-leaves assumption. Groups wrap whole trees; ordinary new tabs have no parent.
[Identify native opener tracking and tab-tree behavior](https://github.com/mia-riezebos/helium/issues/12) is complete. Native source attribution is reusable; temporary opener links do not supply durable tab ancestry.
The fork is `mia-riezebos/helium`, cloned at `~/Development/imputnet/helium`.
The upstream repository is `imputnet/helium`. No browser build or implementation exists yet.
Existing Helium tree-tab proposals provide references, but do not supply recursive named groups or the complete agreed scope.

## Open decisions

- Remaining automatic child placement and branch behavior alongside groups, pins and routing.
- How shared pins behave between panels assigned to different profiles, if the stretch goal proceeds.

## Next step

Resolve [Define tab trees and automatic child placement](https://github.com/mia-riezebos/helium/issues/11) through the wayfinder map.
Then settle restoration and acceptance evidence.
Assess profile-per-panel feasibility only; its implementation remains a stretch goal.
