# Helium tab organization

Terms for the tab organization built into Helium's toolbar.

## Language

**Panel**:
A destination in the tab toolbar or vertical sidebar's navigation.
A panel can show a tab collection or another browser view, such as bookmarks.

**Tab panel**:
A panel containing tab trees and nested groups shown together.
_Avoid_: Profile, account

**Tab**:
A page represented in a browser window. In a tab tree, a tab can have child tabs.

**Tab tree**:
The parent-child organization of tabs in a tab panel.

**Parent tab**:
The immediate tab ancestor of another tab in the tab tree.

**Opener tab**:
The tab from which a new tab is opened.

**Bookmarks panel**:
A panel showing the browser profile's bookmarks.

**Profile**:
A Chromium browsing identity with its own website sessions, settings, and profile data.
_Avoid_: Panel

**Group**:
A named node that can contain tabs and other groups recursively.
_Avoid_: Flat Chrome group

**Group path**:
The ordered names of a group's ancestors and the group itself within a tab panel, such as `Clients > Acme`.

**Shared pin**:
A pinned tab available in every tab panel.

**Routing rule**:
A domain or URL-path match that assigns a tab to a tab panel.
