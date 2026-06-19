# Dashboard Container: Embed and Swap Panels

## Problem/Feature Description

Your team is building the main dashboard screen for an iOS analytics app. The dashboard needs to show two panels side by side: a statistics summary on the left and a visualization on the right. Both panels are full view controllers managed by a central `DashboardViewController`. The design team has confirmed that the right panel must be swappable at runtime — users can switch between a chart view and a map view depending on which data dimension they select.

Everything must be built programmatically using UIKit, without storyboards or XIBs. The app targets iOS 15+. The dashboard will be used as the root content of a `UINavigationController`, so the view hierarchy must be robust and memory-safe.

## Output Specification

Write a single Swift file named `DashboardViewController.swift` that contains:

- `DashboardViewController` — the container, with:
  - A `StatsViewController` embedded on the left half of the view
  - A `ChartViewController` embedded on the right half of the view, initially
  - A method `replaceRightPanel(with:)` that removes the current right panel and installs a new child view controller in its place
  - All layout done with Auto Layout constraints
- Stub implementations of `StatsViewController`, `ChartViewController`, and any additional child VC class you use in your demo of `replaceRightPanel(with:)`

The file should be complete and syntactically valid Swift. Include a brief comment at the top with your name and the date.
