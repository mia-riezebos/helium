# Helium tab organization

## Purpose

Build Sidebery-style tab organization into Helium's native tab toolbar.
The first release targets the agreed features below rather than complete Sidebery parity.

## Required behavior

- Named groups can contain other groups and tabs recursively.
- Panels hold separate tab collections. Horizontal scrolling over the tab toolbar switches panels.
- Support both horizontal and vertical tab modes, with vertical mode driving the design.
- Route tabs automatically to panels using configurable domain or URL-path rules. Routing behavior remains configurable.
- Pinned tabs appear in every panel.
- Tabs and groups have configurable colours, shown as a small stripe on the left.
- Support configurable keybindings and mouse actions. The full Sidebery interaction catalogue is unnecessary.
- Preserve organization across browser restarts and session restoration.

## Constraints

- Windows, macOS, and Linux are required platforms.
- Integrate into the existing native tab toolbar. Keep the extension side panel available independently.
- Browser changes are required; a standalone Chrome extension cannot deliver the intended integration.
- Use cross-platform browser architecture. Platform-exclusive window tabbing is excluded.
- Reuse Sidebery code where appropriate. An internal WebUI hosted in the native toolbar is a candidate, not a validated decision.
- Sidebery's existing behavior and options are the reference for unspecified details. Check its implementation before asking product questions; ask when the proposed behavior differs or research leaves a real ambiguity.

## Stretch goal

Allow each panel to use a Chromium profile so different website accounts can coexist in one window.
Panels must work without profile assignment. Profile isolation must remain intact when this feature is added.

## Current state

Feasibility research exists in `helium-feasibility.md`.
The fork is `mia-riezebos/helium`, cloned at `~/Development/imputnet/helium`.
The upstream repository is `imputnet/helium`. No browser build or implementation exists yet.
Existing Helium tree-tab proposals provide references, but do not supply recursive named groups or the complete agreed scope.

## Open decisions

- The initial configurable keyboard and mouse action catalogue, derived from Sidebery's existing actions.
- How shared pins behave between panels assigned to different profiles, if the stretch goal proceeds.

## Next proof

Embed the tab interface in Helium's toolbar and connect it to real tabs.
Verify recursive grouping, panel switching, and restoration before broadening the implementation.
