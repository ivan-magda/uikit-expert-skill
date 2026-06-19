# Photo Grid: Efficient Large-Image Display

## Problem/Feature Description

Your team is building an iOS app for a photography studio that needs to showcase a portfolio of high-resolution photos in a scrollable grid. The app currently loads images directly from disk and displays them full-size — testers have reported that the app crashes on older devices when browsing more than a dozen photos, and scrolling is choppy even on modern hardware.

You've been asked to rewrite the photo grid screen from scratch. The grid will display many photos at once in a compact thumbnail layout, so memory efficiency is critical. The implementation must scroll smoothly, even when the user flicks rapidly through hundreds of thumbnails. Cells will be recycled as the user scrolls, and the loading logic must handle that gracefully — a thumbnail that was loading for a cell that's been recycled should not end up in the wrong cell once the recycle is done.

The data model is simple: each photo has a stable UUID and a URL pointing to a local image file. You can synthesize a set of test photos programmatically (using `UIGraphicsImageRenderer` or similar to draw solid-color images to temp files) to exercise the grid — no external downloads needed. The grid itself should use a 3-column layout.

## Output Specification

Produce a self-contained Swift source file named `PhotoGridViewController.swift` containing the full implementation. The file should include:

- `PhotoGridViewController`: the main view controller
- `PhotoCell`: a `UICollectionViewCell` subclass used by the grid
- Any supporting types (model, cache, etc.) needed to make it work

The app does not need to compile and run in isolation (no `AppDelegate` or `SceneDelegate` required), but the implementation should be complete enough that a reviewer can assess correctness from the source alone.

Do not leave any large temporary image files (>50 MB) on disk when finished.
