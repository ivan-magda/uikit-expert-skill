# Expandable FAQ Cards

## Problem/Feature Description

Your team is building a FAQ screen for a customer-facing iOS app. Users have complained that the previous screen was overwhelming — it showed all answer text at once, making it hard to scan. The new design calls for a list of question cards where each card shows only the question title by default. Tapping a card should smoothly reveal (or hide) the full answer text with an animated transition. Multiple cards can be expanded simultaneously.

The implementation must be done entirely in UIKit with programmatic Auto Layout — no Storyboards or XIBs. The screen should work cleanly without any runtime layout warnings appearing in the debug console. The engineering lead has emphasized that this kind of expanding/collapsing layout is a classic source of Auto Layout performance issues, so the constraint setup must be efficient and the animations must be smooth.

## Output Specification

Produce a Swift source file named `FAQViewController.swift` that contains a complete, runnable UIKit view controller implementation of the FAQ screen. The file should include:

- A `FAQViewController` class that manages a list of at least 4 FAQ items
- A custom card view or cell class for displaying each FAQ item with its expand/collapse behavior
- All Auto Layout setup and constraint management code
- Animation logic for the expand/collapse transition

The file should be self-contained — someone reviewing it should be able to understand the full implementation by reading it alone.
