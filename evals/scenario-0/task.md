# Photo Grid Screen

## Problem/Feature Description

Your team is building a travel photo discovery app in UIKit. The app needs a full-screen photo gallery where users can browse a large collection of destination photos. The photos live on a remote server and must be downloaded on demand as the user scrolls. Initial loading should be fast and the grid should scroll smoothly without jank, even on older devices.

A previous attempt at this screen was scrapped because it consumed too much memory — the app was crashing on devices with 2GB RAM when browsing large photo libraries. The root cause was traced back to how images were decoded and cached: full-resolution bitmaps were being held in memory even for thumbnail-sized cells. The new implementation needs to be memory-efficient from the ground up.

The grid should display at least 20 photos in a multi-column layout. Images must load asynchronously so the UI stays responsive. The implementation must handle rapid scrolling correctly: cells that scroll offscreen should not continue doing work, and a cell that is reused for a new photo must not momentarily show the wrong image. The data layer should be designed so that updating a single photo's metadata (e.g. adding a "liked" badge) does not unnecessarily reload every other cell.

## Output Specification

Produce Swift source files (`.swift`) that implement the photo grid screen. You may split the code across as many files as makes sense. At minimum, provide:

- A view controller that sets up and manages the collection view
- A cell class for displaying individual photos
- Any supporting types (image loader, cache, model, etc.) as separate files if appropriate

Also write a short `summary.md` (plain text or Markdown) explaining the key design decisions you made — particularly around image loading, memory management, and collection view data management.

The agent should not run the app or require a simulator; only the written source files will be reviewed.
