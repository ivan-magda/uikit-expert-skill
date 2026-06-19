# Article Card View Component

## Problem/Feature Description

Your team is building a news reader app and needs a polished, reusable `ArticleCardView` component — a `UIView` subclass that displays an article's title and a short summary. The card should feel at home on any user's device: it must look correct in both light and dark mode (including on displays that use system-adaptive coloring), automatically resize its text for users who rely on larger accessibility font sizes, and be fully usable with VoiceOver for blind and low-vision users.

The design team also wants the card to have a rounded-corner border that shimmers with a subtle animation while content is loading. Because this shimmer needs to be perfectly synchronized to screen refreshes for a smooth 60/120 fps effect, the animation loop must be driven at the display's native frame rate. A known pitfall here is that the card may need to stay in memory for a while (e.g., in a scroll view offscreen), so the animation timer must not prevent the card from being released when it's no longer needed.

## Output Specification

Produce a single Swift source file named `ArticleCardView.swift` containing:

1. **`ArticleCardView`** — a `UIView` subclass with:
   - A title label and a summary label, each correctly configured for dynamic text sizing
   - A rounded-corner border whose color adapts automatically when the system appearance changes
   - Accessibility support so VoiceOver users get a meaningful, descriptive experience
   - A shimmer animation on the border driven by a display-link-based animation loop

2. **`ArticleCardViewController`** — a minimal `UIViewController` that hosts one `ArticleCardView` and demonstrates its use (adds it to the view hierarchy with constraints)

Do not include any third-party dependencies. Write the file so it compiles as part of a standard UIKit iOS project (no SwiftUI, no SPM packages required). The file should be self-contained and ready for a developer to drop into an Xcode project.
