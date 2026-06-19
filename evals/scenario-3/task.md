# Stopwatch Screen with Delegate Notifications and Lap Callbacks

## Problem Description

Your team is building a fitness tracking app and needs a `StopwatchViewController` that lets users time their workouts. The stopwatch runs in the background and notifies a coaching component whenever a milestone is reached (every 10 seconds) — for example, to trigger an audio cue or update a remote session log. Users can also record laps during a run, and each lap recording triggers a callback that internally formats and stores the split time using another closure.

The engineering lead has flagged a critical concern: previous screens in the app were leaking memory when dismissed, causing the app to accumulate stale background timers and observers. This stopwatch must be implemented cleanly so that when the user navigates away, all resources are properly released and no background activity continues. The deallocation must be verifiable in the Xcode console during development.

The stopwatch screen should support a clean architecture that separates the milestone notification responsibility via a delegate, while the lap feature uses a stored callback closure.

## Output Specification

Implement the `StopwatchViewController` as a single Swift source file named `StopwatchViewController.swift`. The file should include:

- `StopwatchDelegate` protocol with at least one method called when a milestone is reached
- `StopwatchViewController` class that:
  - Starts a repeating timer on `viewDidAppear` (or similar) that fires every second
  - Notifies the delegate every 10 seconds
  - Exposes a `lapCallback` property (a stored closure) that can be set externally; when invoked, it internally creates and calls another closure to format/store the lap time
  - Stops the timer appropriately when the screen is no longer visible
  - Logs to the console when it is deallocated
  - Calls `super` in all overridden lifecycle methods

The file should compile without errors against UIKit/Foundation (no external dependencies needed).
