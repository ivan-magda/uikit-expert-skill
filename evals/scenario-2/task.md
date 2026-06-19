# ShopFlow: Multi-Screen Navigation with Branded Styling

## Problem/Feature Description

The ShopFlow team is rebuilding their iOS shopping app from scratch after a brand refresh. Their product manager has defined three core screens: a **Home** screen (the entry point featuring prominently-sized branding and featured products), a **Category** screen (a mid-level browsing view with a standard navigation bar), and a **Product Detail** screen (a full-bleed image experience where the navigation bar should appear transparent so the hero image bleeds beneath it).

The app's marketing team requires consistent branding across all navigation bar states — including when the user scrolls, when the device is in compact height (landscape), and the intersection of compact + scrolled. Past implementations only styled the "normal" state and shipped with jarring color flashes as users scrolled or rotated their phone; this time the bar must look correct in every state.

Additionally, the CS team reports that push notification deep links currently drop users at the home screen instead of the intended product detail screen, because the old navigation code pushed screens one at a time and frequently triggered crashes from rapid taps. The new implementation must support atomically navigating to a specific product detail (skipping animation jank from intermediate screens) and must gracefully handle rapid navigation triggers without crashing.

## Output Specification

Write Swift source files that implement the three-screen navigation flow described above. Structure your code however you see fit — UIKit UINavigationController-based, programmatic layouts, split into multiple files or a single file, your choice.

Produce the following output files:

- **`NavigationApp.swift`** (or multiple `.swift` files) — the UIKit implementation including all three view controllers and the navigation controller setup
- **`deeplink.swift`** — a standalone function or method (can reference the above VCs) that demonstrates how to deep-link directly to the Product Detail screen for a given product ID, bypassing intermediate push animations for the intermediate screens
- **`design_notes.md`** — a short document (200–500 words) explaining:
  - How navigation bar appearance is configured and why you made the choices you did
  - How the large/standard title split between screens is managed
  - How the deep-link entry point works and why you chose that approach
  - How the implementation handles situations where a navigation gesture or push is already in progress
