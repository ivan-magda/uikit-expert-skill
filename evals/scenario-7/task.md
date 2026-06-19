# News Feed View Controller

## Problem/Feature Description

The Readly app has a news feed screen that shows a scrollable list of articles fetched from the backend. The team's backend engineer has exposed a Swift async function that returns an array of `Article` values — the function simulates real latency by sleeping for a couple of seconds before returning results.

The iOS team needs a `FeedViewController` that wires this fetch into the UIKit view lifecycle correctly. A recurring complaint from QA is that the app occasionally shows stale data or crashes when a user quickly navigates into and then back out of the feed screen before loading completes. The fix requires the view controller to properly manage the async work: start it when the screen appears, stop it when the user leaves, and make sure the UI is only updated when the data actually arrived for the current session. The implementation should be plain UIKit with Swift Concurrency — no Combine, no DispatchQueue-based workarounds.

The company's minimum deployment target is iOS 15. If any iOS 26-specific concurrency features are used, they must degrade gracefully on older OS versions.

## Output Specification

Produce a single Swift source file named `FeedViewController.swift` that contains:

- An `Article` struct or class (with at least `id: UUID` and `title: String` fields)
- A standalone async function `fetchArticles() async -> [Article]` that simulates a network fetch with an artificial delay
- A `FeedViewController: UIViewController` subclass that:
  - Displays the articles in a `UICollectionView` or `UITableView`
  - Starts fetching articles when the view appears
  - Cancels the fetch if the user navigates away before it finishes
  - Only updates the UI once loading is complete and the user is still on the screen
  - Uses `UICollectionViewDiffableDataSource` or a table view data source as appropriate
