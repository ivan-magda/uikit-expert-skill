# Preferences Screen for a Productivity App

## Problem Description

The mobile team at a productivity startup is building a Settings screen for their iOS app. Users need a clean, professional preferences panel that works beautifully across all device conditions: light and dark appearances, any Dynamic Type size (including accessibility sizes), and full VoiceOver navigation for users who rely on screen readers.

The screen should be a scrollable list of preference categories, implemented as a UICollectionView. It must contain at least three distinct sections — for example: **Profile**, **Appearance**, and **Notifications** — with multiple rows in each section. The layout should use the standard inset-grouped list appearance that users expect in a native iOS settings screen.

The list should include at least one row that has a swipe-to-delete or other swipe action (such as "Reset to defaults" for a section). VoiceOver users must also be able to trigger this action without swiping, since they cannot perform standard swipe gestures on list items.

At the bottom of the screen there should be a text input area — for example a "Display Name" field or a short bio — that adjusts correctly when the software keyboard appears, so the input is never obscured.

The team wants the screen to look polished and respond correctly to:
- The user switching between light and dark mode at runtime
- The user changing their preferred text size in iOS Settings
- VoiceOver being active

The screen must be wired into a UINavigationController so the navigation bar is visible.

## Output Specification

Produce the following Swift source files in your working directory:

- `SettingsViewController.swift` — the main UICollectionView-based settings screen
- `SettingsCell.swift` — any custom cell type(s) used in the collection view
- `AppDelegate.swift` — minimal app entry point wiring the settings screen into a UINavigationController
- `accessibility_report.md` — a short report (markdown) documenting:
  - Which `accessibilityLabel`, `accessibilityTraits`, and `accessibilityHint` values are set on each custom or non-standard element
  - How swipe actions on list rows are made accessible to VoiceOver users who cannot perform standard swipe gestures
  - How CALayer color properties (e.g. `borderColor`, `shadowColor`) are kept in sync with dark mode trait changes at runtime

All Swift files should be compilable and self-contained (no external dependencies beyond UIKit/SwiftUI). You do not need to run or build the project — just produce the source files.

Clean up any downloaded or generated files larger than 50 MB before finishing.
