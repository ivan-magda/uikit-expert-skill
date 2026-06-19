# Profile Screen with Photo Strip and Collapsible Bio

## Problem Description

The mobile team at a photo-sharing app needs a profile screen built entirely in code (no Storyboard or XIB). The screen shows a horizontally scrollable strip of the user's photos at the top, followed by a bio section that can be expanded or collapsed by the user. When navigating to a profile from a photo detail view, the app should restore context by scrolling the photo strip to the last photo the user was viewing.

The profile screen is used from multiple entry points: the main feed taps into it mid-scroll, and deep links from push notifications can pre-select a specific photo. The current placeholder implementation loses the user's place every time the screen appears, which is jarring. It also triggers layout warnings in the console that make it hard to debug constraint issues in production crash logs.

The engineering team has agreed on a few requirements: the bio section must animate smoothly when toggling between expanded and collapsed states, the animation must not cause layout thrashing, and the code must be clean enough for a junior engineer to follow which constraint is which from the crash logs alone.

## Output Specification

Implement the `ProfileViewController` as a single Swift file named `ProfileViewController.swift`. The view controller must:

- Be fully programmatic (no Storyboard/XIB)
- Accept an initializer parameter for the index of the initially-selected photo (e.g., `init(selectedPhotoIndex: Int)`)
- Display a horizontally scrolling collection view of photos (use placeholder images or solid-color cells — actual photo loading is not required for this task)
- Show a bio section below the photo strip that can be expanded and collapsed by tapping a button; the transition must be animated
- Scroll the photo strip to the selected photo index when the screen opens
- Use at least 8 sample photos so the scroll behavior is testable

The file should be a self-contained UIKit view controller implementation. Do not produce a full Xcode project — just the Swift source file.
