# Activity Monitor View Controller

## Problem/Feature Description

Your team is building an iOS fitness tracking app. One of the key screens is an "Activity Monitor" that shows real-time workout stats: elapsed time since the session started, a smoothly animated progress ring showing current effort level, and live status messages that reflect app lifecycle events (such as when the app is backgrounded and then foregrounded again). The screen also needs to report session milestones (like every 30 seconds) back to a parent coordinator object via a custom delegate, and it must load the user's personal record data from the server when the monitor first appears.

This screen is presented modally during active workout sessions and may be pushed and popped frequently as the user navigates. It must not leak memory across presentations — your QA lead has already flagged retain cycles in a prototype that caused the app to balloon in memory after several workout sessions. The engineering goal is to implement the `ActivityMonitorViewController` with correct memory management for every concurrency primitive in use.

## Output Specification

Produce the following Swift source files in your working directory:

- `ActivityMonitorViewController.swift` — the main UIKit view controller implementing the activity monitor screen. It should include:
  - A `Timer` that fires every second to update the elapsed time display
  - A `CADisplayLink` driving a smooth progress ring animation
  - A `NotificationCenter` observer tracking `UIApplication.didBecomeActiveNotification` and `UIApplication.didEnterBackgroundNotification`
  - A custom delegate protocol and a `delegate` property for reporting milestone events to a parent object
  - An async `Task` that fetches personal record data from a remote endpoint (you may use `URLSession.shared.data(from:)` with any placeholder URL — the grader will not execute the code, only read it)
  - A label or text element displaying elapsed time, a simple view or layer representing the progress ring, and status text showing the last app lifecycle event received

- `memory_analysis.md` — a markdown document that explicitly describes:
  - For each memory-management pattern used (Timer, CADisplayLink, NotificationCenter, delegate, async Task, any escaping closures), how retain cycles are prevented
  - Where `[weak self]` appears and why it is needed in each location
  - Where Tasks are stored and when they are cancelled
  - Where the NotificationCenter observer token is removed
  - Where the Timer is invalidated and the CADisplayLink is removed from its run loop
