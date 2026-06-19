# Chat Compose Screen

## Problem/Feature Description

Your team is building a messaging feature for a mobile app. The product manager wants a dedicated compose screen where users can scroll through their message history and type new messages using a text input bar pinned to the bottom of the screen. A key requirement is that when the software keyboard appears, the compose bar must slide up automatically to stay visible — users should never have to hunt for the input field behind the keyboard.

The design also needs to work correctly on iPad, where users sometimes detach the keyboard and float it anywhere on screen. In that case, the compose bar should still respond correctly to where the floating keyboard actually is.

The entire screen should be built programmatically in UIKit — no storyboards, no XIBs. The implementation should follow modern iOS best practices.

## Output Specification

Write a Swift file named `ChatComposeViewController.swift` containing a `ChatComposeViewController: UIViewController` implementation with:

- A scrollable area (using `UIScrollView` or `UICollectionView`) showing a list of chat messages (you may use hardcoded sample messages)
- A compose bar at the bottom containing a `UITextField` or `UITextView` for composing new messages and a send button
- Automatic keyboard avoidance: the compose bar should move up to remain visible when the keyboard appears or changes position, and return to the bottom when the keyboard is dismissed
- Support for iPad's floating (undocked) keyboard

The implementation must be entirely programmatic. Deliver only the Swift source file.
