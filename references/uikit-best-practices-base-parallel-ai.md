# UIKit 2024–2026: The Definitive Playbook for Correct, Fast, Modern Swift UI

## Executive Summary

Modern UIKit development has undergone a radical architectural shift driven by Swift concurrency, reactive data flow, and strict performance mandates. As of the iOS 26 SDK, the `UIScene` lifecycle is no longer optional; apps failing to adopt it will not launch, marking the final deprecation of the legacy `UIApplicationDelegate` window management [guide_summary[1]][1]. Correctness now hinges on compiler-enforced main-thread safety using `@MainActor` and `defaultIsolation`, eliminating the class of race conditions that plagued earlier Objective-C-style code [concurrency_and_main_thread_safety.main_actor_scoping[0]][2].

Performance optimization has moved from manual calculation to declarative definitions. The introduction of `UIKeyboardLayoutGuide` eliminates thousands of lines of brittle notification observation code [keyboard_and_scroll_management.uikeyboardlayoutguide_usage[0]][3]. In lists, `DiffableDataSource` combined with `reconfigureItems` is the only performant path for content updates, rendering `reloadData` obsolete for all but the most drastic state changes [modern_list_and_grid_apis.performance_and_correctness[0]][4]. Furthermore, the integration of Swift's Observation framework allows UIKit views to automatically invalidate and update without manual `setNeedsLayout` calls, a pattern formalized in iOS 26 with the `updateProperties()` lifecycle method [guide_summary[0]][5].

This report details the non-negotiable best practices for writing UIKit code that is correct by default, performant on 120Hz displays, and fully integrated with the modern Swift ecosystem.

## 2024–2026 UIKit Modernization — What’s New and Non‑Negotiable

The convergence of UIKit and SwiftUI patterns has created a "reactive UIKit" where state drives UI updates automatically.

### UIScene Lifecycle Is Mandatory in iOS 26 — Migration Steps that Prevent App Launch Failures
Adopting the `UIScene`-based lifecycle is now mandatory. Apps built with the iOS 26 SDK that rely solely on legacy `UIApplicationDelegate` callbacks for UI setup will fail to launch [guide_summary[1]][1]. Migration requires shifting responsibilities:
* **Manifest Declaration**: Add `UIApplicationSceneManifest` to `Info.plist` to declare scene support.
* **Window Management**: Move all `UIWindow` initialization from `application(_:didFinishLaunchingWithOptions:)` to `scene(_:willConnectTo:options:)` in a `UISceneDelegate`.
* **State Restoration**: Handle deep links and state restoration on a per-scene basis using the connection options or `scene(_:continue:)`.

### Reactive UIKit via Observation and updateProperties() — Back‑Deploy to iOS 18
Modern UIKit embraces Swift's Observation framework to eliminate manual invalidation. By enabling `UIObservationTrackingEnabled` in `Info.plist` (back-deployable to iOS 18), views automatically track dependencies on `@Observable` models [guide_summary[0]][5].
* **The New Lifecycle**: iOS 26 introduces `updateProperties()`, a dedicated method that runs before layout specifically for applying model data to view properties [guide_summary[0]][5].
* **Automatic Invalidation**: Reading an `@Observable` property inside `updateProperties` (or `layoutSubviews` on older iOS) registers a dependency; the system automatically re-runs the method when the data changes [modern_ui_update_pipeline.automatic_observation_tracking[0]][6].
* **Risk Mitigation**: Avoid modifying observed properties inside these methods to prevent "update storms" (infinite invalidation loops) [modern_ui_update_pipeline.automatic_observation_tracking[0]][6].

###.flushUpdates Eliminates Manual layoutIfNeeded() in Animations
The `.flushUpdates` animation option, introduced in iOS 26, fundamentally simplifies animation logic. It automatically applies pending layout and property updates—including those triggered by Observable changes—both before and after the animation block [guide_summary[0]][5]. This removes the need for the error-prone pattern of manually calling `layoutIfNeeded()` inside animation blocks to force frame updates [guide_summary[0]][5].

## View Controller Lifecycle — Correctness-First Patterns that Avoid Jank

Correct view controller management relies on strict adherence to method responsibilities, particularly regarding geometry and cleanup.

### Method Responsibilities with Failure Cases
* **One-Time Setup**: Use `viewDidLoad` for initialization that is independent of view geometry (e.g., adding subviews, setting static constraints) [view_controller_lifecycle_practices.method_responsibilities[0]][7].
* **Geometry-Aware Setup**: Place logic dependent on final frames (e.g., layer gradients, manual positioning) in `viewDidLayoutSubviews`. This is the only point where bounds are guaranteed to be final [view_controller_lifecycle_practices.method_responsibilities[2]][8].
* **Critical Anti-Pattern**: Never call `setNeedsLayout()` or `setNeedsUpdateConstraints()` inside `viewWillLayoutSubviews` or `viewDidLayoutSubviews`. Doing so schedules an immediate subsequent layout pass, creating an infinite loop that freezes the app [auto_layout_performance_and_correctness[0]][9].

### TransitionCoordinator for Size/Rotation — Lockstep Animations
To handle device rotation or Split View resizing smoothly, use `viewWillTransition(to:with:)`. The provided `UIViewControllerTransitionCoordinator` allows you to animate constraint changes alongside the system's rotation animation [view_controller_lifecycle_practices.transition_coordinator_usage[0]][10].
* **Pattern**: Call `coordinator.animate(alongsideTransition:completion:)` to synchronize your custom layout updates with the system transition [animation_api_selection_guide.api_capability_matrix[2]][11].

### Deallocation Verification Playbook
Memory leaks in view controllers are a primary source of resource exhaustion.
* **Verification**: Implement `deinit` with a log statement or breakpoint. If `deinit` is not hit when a view controller is popped, a retain cycle exists [memory_management_and_leak_prevention.delegate_ownership_rules[3]][12].
* **Tooling**: Use Xcode's Memory Graph Debugger to visualize strong reference cycles, particularly looking for closures capturing `self` strongly or delegates not marked `weak` [memory_management_and_leak_prevention.closure_capture_cycles[2]][13].

## Auto Layout Performance & Constraint Management

Performance issues in Auto Layout often stem from "constraint churn"—the expensive process of removing and re-adding constraints.

### Zero-Churn Constraints with Batch Activation
The most performant strategy is to create constraints once and toggle them.
* **Avoid**: Removing all constraints in `updateConstraints` and recreating them. This forces the solver to rebuild the equation system from scratch [auto_layout_performance_and_correctness.constraint_churn_avoidance[0]][14].
* **Adopt**: Create static constraints in `viewDidLoad`. To change layout, toggle the `isActive` property or modify the `.constant` of existing constraints [auto_layout_performance_and_correctness.constraint_churn_avoidance[1]][9].
* **Batching**: Use `NSLayoutConstraint.activate([... ])` to apply multiple changes in a single pass, which is more efficient than activating them individually [auto_layout_performance_and_correctness.batching_updates[2]][9].

### StackView vs Manual Constraints — Where Performance Tips
While `UIStackView` is convenient, it adds overhead.
* **Guidance**: Avoid deeply nested stack views (more than 2 levels) inside reusable cells (`UITableViewCell`/`UICollectionViewCell`). The layout calculation cost accumulates rapidly during scrolling [auto_layout_performance_and_correctness.uistackview_vs_manual_constraints[0]][15].
* **Alternative**: Use manual constraints or a single flat `UIStackView` for performance-critical list items [auto_layout_performance_and_correctness.uistackview_vs_manual_constraints[0]][15].

### Animate Constraints Correctly
To animate layout changes without visual glitches:
1. Update the `constant` of the constraint (or toggle active constraints).
2. Call `view.layoutIfNeeded()` inside the animation block to force the layout engine to interpolate the frames [auto_layout_performance_and_correctness.constraint_churn_avoidance[5]][16].
3. **Modern Path**: On iOS 26+, simply use the `.flushUpdates` option in `UIView.animate` to handle this automatically [guide_summary[0]][5].

## Modern Lists and Grids — Diffable + Compositional as the Standard

The combination of `UICollectionViewDiffableDataSource` and `UICollectionViewCompositionalLayout` is the undisputed standard for all lists and grids.

### Identity Rules that Prevent Crashes and Glitches
Correctness in diffable data sources relies entirely on identifier stability.
* **Stable Identity**: Use a model's stable `UUID` or database primary key as the item identifier, *not* the model struct itself. If the identifier changes when the content changes, the diffing engine perceives it as a delete-and-insert rather than an update, causing animation glitches [modern_list_and_grid_apis.diffable_data_sources[0]][17].
* **Uniqueness**: Identifiers must be globally unique in the snapshot. Duplicates cause the crash `BUG_IN_CLIENT_OF_DIFFABLE_DATA_SOURCE__DUPLICATE_ITEM_IDENTIFIERS` [modern_list_and_grid_apis.diffable_data_sources[0]][17].

### Reconfigure vs Reload — Preserve Cell Instance and State
For content updates (e.g., a heart icon toggling), avoid `reloadItems`.
* **Use `reconfigureItems(_:)`**: This keeps the existing cell, preserves its internal state (like scroll position or text selection), and simply re-runs the configuration handler. It is significantly cheaper than dequeuing a new cell [modern_list_and_grid_apis.performance_and_correctness[0]][4].
* **Initial Load**: Use `applySnapshotUsingReloadData(_:)` for the very first population of a large list to bypass the expensive diffing calculation [modern_list_and_grid_apis.performance_and_correctness[1]][18].

### Compositional Layout Performance Tuning
* **Caching**: Creating `NSCollectionLayoutSection` objects can be expensive. Cache them by section index or type to avoid recreation during every scroll event [modern_list_and_grid_apis.compositional_layout[0]][19].
* **Orthogonal Scrolling**: Use `orthogonalScrollingBehavior` for horizontal rows. Monitor scroll offsets using the `visibleItemsInvalidationHandler` on the section, rather than `UIScrollViewDelegate` [modern_list_and_grid_apis.compositional_layout[0]][19].

## Performant Cell Configuration — Declarative, State-Driven, Reactive

Modern cell configuration decouples state from views using lightweight configuration structs.

### Core Components and State Handling
* **Registration**: Use `UICollectionView.CellRegistration` to register cells in a type-safe manner, eliminating string identifiers and manual casting [performant_cell_configuration.core_components[0]][20].
* **Configuration**: Use `UIListContentConfiguration` for standard layouts. It acts as a view model; you populate it and assign it to `cell.contentConfiguration` [performant_cell_configuration.core_components[2]][21].
* **State Updates**: Implement `configurationUpdateHandler`. This closure is automatically called when cell state changes (e.g., selected, highlighted), allowing you to centralize all appearance logic based on `UICellConfigurationState` [performant_cell_configuration.state_driven_updates[0]][22].

### Reactive Updates with @Observable Models
When `UIObservationTrackingEnabled` is active, accessing an `@Observable` property inside a cell's `configurationUpdateHandler` automatically registers a dependency. If the model changes, UIKit re-runs the handler for that specific cell immediately, enabling SwiftUI-like data flow within UIKit cells [performant_cell_configuration.reactive_updates_with_observation[0]][6].

### Async Image Loading Pattern with Cancellation
To load images in cells without race conditions:
1. Check if the image is cached or present on the model.
2. If not, start a `Task` to fetch it.
3. **Crucial**: Verify the cell's identity has not changed when the task completes before applying the image.
4. **Cancellation**: Cancel any in-flight tasks in `prepareForReuse` to prevent stale images from overwriting new content [performant_cell_configuration.asynchronous_content_loading[0]][22].

## List Prefetching & Cancellation with Swift Concurrency

Prefetching is essential for smooth scrolling, especially with heavy media.

### Delegate Correctness and Priority Handling
Implement `prefetchRowsAt` (or `prefetchItemsAt`). The array of index paths provided is sorted by priority; iterate in order to fetch data for the items closest to the viewport first [list_prefetching_and_cancellation.delegate_implementation[0]][23]. Dispatch expensive work (network/disk) to a background task immediately [list_prefetching_and_cancellation.delegate_implementation[0]][23].

### Concurrency Integration and Task Bookkeeping
* **Pattern**: Store running `Task` handles in a dictionary keyed by the item's stable identifier (not `IndexPath`, which is volatile) [list_prefetching_and_cancellation.swift_concurrency_integration[0]][24].
* **Cancellation**: Implement `cancelPrefetchingForRowsAt`. When called, look up the task by ID and call `task.cancel()`. This prevents wasting bandwidth on cells the user scrolled past quickly [list_prefetching_and_cancellation.swift_concurrency_integration[0]][24].

## UINavigationController — Correct, Safe, and Leak‑Free Navigation

### Safe Stack Mutations and Deep Links
* **Set Stack**: Use `setViewControllers(_:animated:)` to modify the entire navigation stack at once. It intelligently animates the transition (push or pop) to the target state and is the safest way to handle deep links [navigation_controller_best_practices.safe_stack_mutations[0]][25].
* **Constraint**: Never push a `UITabBarController` onto a navigation stack, and ensure the same view controller instance is not pushed twice [navigation_controller_best_practices.safe_stack_mutations[0]][25].

### Prevent Concurrent Transition Crashes
Initiating a transition while another is in progress causes runtime warnings or crashes.
* **Guard**: Check `viewController.transitionCoordinator`. If it is not `nil`, a transition is active.
* **Sequence**: If a transition is active, chain the next navigation action in the `completion` block of `transitionCoordinator.animate(alongsideTransition:completion:)` [navigation_controller_best_practices.concurrent_transition_prevention[0]][26].

### Gesture Interop in iOS 26
iOS 26 introduces `interactiveContentPopGestureRecognizer`, allowing back-swipes from anywhere in the content area. If your app uses custom horizontal pan gestures (e.g., carousels), you must set failure requirements (`require(toFail:)`) against this new system recognizer to prevent conflicts [navigation_controller_best_practices.interactive_gesture_handling[2]][27].

## Animation API Selection — Choose by Control vs Offloading

| API | Best for | Interactivity | Off-main-thread | Works with Constraints | Notes |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **UIView.animate** | Simple one-shot UI changes | No | No | Yes (with.flushUpdates) | Simplest syntax; fire-and-forget [animation_api_selection_guide.api_capability_matrix[0]][28]. |
| **UIViewPropertyAnimator** | Gesture-driven transitions | Yes | No | Yes | Supports pause, scrub, reverse, and velocity handoff [animation_api_selection_guide.api_capability_matrix[0]][28]. |
| **Core Animation (CA)** | High-perf layer effects | Limited | Yes (Render Server) | No | Runs on Render Server; immune to main-thread hitches [animation_api_selection_guide.api_capability_matrix[1]][29]. |

### Interactive Patterns with PropertyAnimator
For gesture-driven animations, use `UIViewPropertyAnimator`. You can pause the animation and scrub it by setting `fractionComplete` based on gesture translation. When the gesture ends, use `continueAnimation` to smoothly settle the view [animation_api_selection_guide.interactive_animations[0]][28].

## Concurrency & Main Thread Safety — Make It Impossible to Be Wrong

Swift 6 enforces thread safety at compile time, making "Main Thread Checker" violations a build-time error.

### MainActor Scoping Strategies
* **Module-Wide**: Enable `defaultIsolation(MainActor.self)` in build settings (Swift 6.2+) to make the main thread the default for all UI code [concurrency_and_main_thread_safety.main_actor_scoping[0]][2].
* **Granular**: Use `@MainActor` on view controller classes and view models to ensure all property access and method calls happen on the main thread [concurrency_and_main_thread_safety.main_actor_scoping[0]][2].

### Deadlock Avoidance & Actor Reentrancy
* **Deadlock**: Never call `DispatchQueue.main.sync` from a background task. This is a classic deadlock source. Replace it with `await MainActor.run {... }` [concurrency_and_main_thread_safety.deadlock_avoidance[0]][30].
* **Reentrancy**: Remember that `await` suspends execution. The app state may change while your function is suspended. Always re-validate assumptions (e.g., `guard !Task.isCancelled`) after an `await` before updating the UI [concurrency_and_main_thread_safety.deadlock_avoidance[0]][30].

### Bridging Legacy Delegates
To safely handle legacy delegates (like `CLLocationManagerDelegate`) that call back on the main thread but lack `@MainActor` annotations:
* **Assume Isolated**: Use `MainActor.assumeIsolated {... }` inside the delegate method to synchronously verify execution is on the main thread and access main-actor isolated state [concurrency_and_main_thread_safety.bridging_legacy_apis[0]][30].

## Memory Management & Leak Prevention

### Closure Captures and When Weak/Unowned Apply
* **Standard**: Use `[weak self]` in escaping closures (like network completion handlers) to prevent retain cycles.
* **Unowned**: Use `[unowned self]` *only* when you can mathematically prove the closure cannot outlive the object (e.g., certain animation blocks). If in doubt, use weak [memory_management_and_leak_prevention.closure_capture_cycles[0]][31].

### Delegates, Observers, Timers — Ownership Rules
* **Delegates**: Must always be `weak` [memory_management_and_leak_prevention.delegate_ownership_rules[0]][32].
* **Timers**: `Timer` and `CADisplayLink` retain their targets. You *must* call `invalidate()` in `viewWillDisappear` or `deinit` to break the cycle [memory_management_and_leak_prevention.observer_and_timer_cleanup[0]][31].
* **Observers**: Block-based `NotificationCenter` observers return a token that must be retained and removed; they do not auto-invalidate like selector-based observers [memory_management_and_leak_prevention.observer_and_timer_cleanup[0]][31].

## UIKit–SwiftUI Interop — Safe, Performant Hybrids

### Embed SwiftUI in UIKit Without Jank
* **Controller**: Use `UIHostingController` to wrap SwiftUI views.
* **Sizing**: Set `sizingOptions =.intrinsicContentSize` (iOS 16+) to ensure the controller correctly resizes itself when the SwiftUI content changes [uikit_swiftui_interoperability.embedding_swiftui_in_uikit[2]][33].
* **Cells**: Use `UIHostingConfiguration` inside `UITableView` or `UICollectionView` cells. This is lighter than a full hosting controller and handles cell states (selection, highlighting) automatically [uikit_swiftui_interoperability.embedding_swiftui_in_uikit[3]][34].

### Embed UIKit in SwiftUI Safely
* **Representables**: Use `UIViewRepresentable` or `UIViewControllerRepresentable`.
* **Lifecycle**: Implement `makeUIView` (create once) and `updateUIView` (update on state change). Keep `updateUIView` lightweight; do not recreate the view here [uikit_swiftui_interoperability.embedding_uikit_in_swiftui[1]][35].
* **Coordination**: Use a `Coordinator` to handle delegates and data flow back to SwiftUI via `@Binding` [uikit_swiftui_interoperability.data_flow_and_coordination[2]][36].

## Image Loading & Caching — Native Stack + Library Tradeoffs

### Native Framework Practices
* **Decoding**: Use `UIImage.byPreparingForDisplay()` (iOS 15+) to decode images on a background thread, preventing main-thread hitches during scrolling [image_loading_and_caching_strategies.native_framework_usage[2]][37].
* **Thumbnails**: Use `UIImage.byPreparingThumbnail(ofSize:)` to downsample images efficiently, saving memory [image_loading_and_caching_strategies.native_framework_usage[5]][38].

### Third-Party Libraries
While native APIs are capable, libraries offer robust caching and cancellation out of the box.

| Library | Strengths | Tradeoffs/Risks |
| :--- | :--- | :--- |
| **Nuke** | Highest performance, built for Swift Concurrency (Actors) [image_loading_and_caching_strategies.third_party_library_comparison[0]][39]. | Newer API surface to learn. |
| **Kingfisher** | Easiest integration, rich feature set [image_loading_and_caching_strategies.third_party_library_comparison[0]][39]. | Generalist approach. |
| **SDWebImage** | Extensive plugin ecosystem (e.g., YYCache) [image_loading_and_caching_strategies.third_party_library_comparison[0]][39]. | Objective-C legacy footprint. |

## Keyboard & Scroll Management — Zero‑Jank with Layout Guides

### Layout Guide Patterns
Replace all keyboard notification logic with `UIKeyboardLayoutGuide` (iOS 15+).
* **Pattern**: Create a single Auto Layout constraint pinning your content's bottom anchor to `view.keyboardLayoutGuide.topAnchor`. The system automatically animates this constraint to match the keyboard's movement [keyboard_and_scroll_management.uikeyboardlayoutguide_usage[0]][3].

### Accessory Views & Edge Cases
* **iPad**: Set `followsUndockedKeyboard = true` on the layout guide to correctly track floating and split keyboards on iPadOS [keyboard_and_scroll_management.uikeyboardlayoutguide_usage[3]][40].
* **Scroll Sync**: Pinning a `UIScrollView` bottom to the keyboard guide automatically handles content insets and scroll indicator insets, ensuring glitch-free resizing [keyboard_and_scroll_management.scroll_view_synchronization[0]][41].

## Adaptive Layout & Trait Handling — Predictable Responses to Change

### Modern Trait Handling
* **Registration**: Use `registerForTraitChanges` (iOS 17+) instead of `traitCollectionDidChange`. This is more performant as it only triggers for specific trait changes you care about [adaptive_layout_and_trait_handling.modern_trait_handling[1]][42].
* **Automatic Tracking**: In iOS 18+, methods like `layoutSubviews` and `drawRect` automatically track trait usage and invalidate themselves when those traits change [adaptive_layout_and_trait_handling.modern_trait_handling[4]][6].

### Dynamic Type and Safe Areas
* **Fonts**: Use `UIFont.preferredFont(forTextStyle:)` for system fonts or `UIFontMetrics` for custom fonts to support Dynamic Type.
* **Labels**: Always set `adjustsFontForContentSizeCategory = true` and `numberOfLines = 0` to allow text to scale and wrap.

## Dark Mode & Appearance — Semantic, Dynamic by Default

### Semantic Colors & Assets
* **System Colors**: Prefer semantic colors like `.label`, `.systemBackground`, and `.separator` which adapt automatically to light/dark mode [dark_mode_and_appearance_management.semantic_colors_and_assets[0]][43].
* **Asset Catalog**: Define custom colors in Asset Catalogs with variants for "Any", "Dark", and "High Contrast". This ensures `UIColor(named:)` returns a dynamic, adaptive color object [dark_mode_and_appearance_management.semantic_colors_and_assets[2]][44].

### UIAppearance Caveats
Avoid `UIAppearance` for properties that need to change at runtime (like theme switching). `UIAppearance` settings are only applied when a view is added to the window; they do not update live views. Use dynamic colors and trait registration instead [dark_mode_and_appearance_management.uiappearance_caveats[0]][45].

## Accessibility — Ship with Inclusive Defaults

### VoiceOver Fundamentals
* **Labels**: Set `accessibilityLabel` to a concise identification (e.g., "Play").
* **Traits**: Use `accessibilityTraits` (e.g., `.button`) so VoiceOver announces the control type automatically ("Play, button") [accessibility_best_practices.voiceover_support[0]][46].
* **Hints**: Use `accessibilityHint` to describe the *result* of the action ("Plays the selected song") [accessibility_best_practices.voiceover_support[0]][46].

### Custom Actions & Rotors
For complex list items (like swipeable rows), use `UIAccessibilityCustomAction`. This allows VoiceOver users to access actions like "Delete" or "Archive" via the Actions rotor without needing to perform complex gestures [accessibility_best_practices.custom_actions_and_rotors[0]][47].

## Common UIKit Anti‑Patterns & How to Fix Them

* **Off-main-thread UI updates**: Updating UI from a background completion handler.
 * *Fix*: Mark UI methods `@MainActor` and use `await` to call them [common_uikit_anti_patterns.corrected_example[0]][48].
* **Constraint Churn**: Removing and re-adding constraints in `updateConstraints`.
 * *Fix*: Toggle `isActive` or change `.constant` on static constraints [auto_layout_performance_and_correctness.constraint_churn_avoidance[0]][14].
* **Layout Loops**: Calling `setNeedsLayout` inside `layoutSubviews`.
 * *Fix*: Remove the call; layout updates should be a result of state change, not a side effect of layout itself [auto_layout_performance_and_correctness[0]][9].
* **Unstable Diffable IDs**: Using a struct as its own identifier.
 * *Fix*: Use a stable `UUID` or database ID [modern_list_and_grid_apis.diffable_data_sources[0]][17].
* **Manual Keyboard Notifications**: Using `keyboardWillShowNotification`.
 * *Fix*: Use `UIKeyboardLayoutGuide` [deprecated_api_replacement_guide.modern_replacement[0]][40].

## Deprecated/Modern Replacement Guide

| Deprecated/Legacy Pattern | Modern Replacement | Version Notes | Migration Steps |
| :--- | :--- | :--- | :--- |
| **Legacy App Lifecycle** | `UIScene` + `SceneDelegate` | Mandatory in iOS 26 [guide_summary[1]][1]. | Add Scene Manifest to Info.plist; move UI setup to SceneDelegate. |
| **traitCollectionDidChange** | `registerForTraitChanges` | Deprecated iOS 17+ [adaptive_layout_and_trait_handling.modern_trait_handling[1]][42]. | Register for specific traits; remove broad overrides. |
| **Keyboard Notifications** | `UIKeyboardLayoutGuide` | iOS 15+ [deprecated_api_replacement_guide.versioning_notes[0]][49]. | Remove observers; pin content to guide. |
| **Manual layoutIfNeeded** | `.flushUpdates` option | iOS 26+ [guide_summary[0]][5]. | Use `UIView.animate(..., options: [.flushUpdates])`. |

## Performance Measurement Toolkit — Prevent and Catch Regressions

### Profiling Stack per Scenario
* **Scroll Jank**: Use **XCTHitchMetric** in tests. In Instruments, use the **Animation Hitches** template to distinguish between Commit Hitches (main thread blocked) and Render Hitches (GPU/Render Server overload) [performance_measurement_toolkit.profiling_stack_per_scenario[0]][50].
* **Launch Time**: Use **XCTApplicationLaunchMetric** to consistently measure cold and warm start times [performance_measurement_toolkit.profiling_stack_per_scenario[0]][50].
* **Custom Intervals**: Use `OSSignposter` to mark intervals (`beginInterval` / `endInterval`) around critical code paths. These appear in the Points of Interest instrument [performance_measurement_toolkit.os_signpost_instrumentation[0]][51].

### CI Gating Strategy
Performance baselines are hardware-dependent. To gate builds on CI:
1. Run performance tests on the CI runner hardware.
2. Commit the generated `.xcbaseline` files to the repository.
3. CI runs will now compare against this hardware-specific baseline, failing only on true regressions.

## iOS 26 Design & Interaction Updates That Affect Code

### Liquid Glass Materials
iOS 26 introduces "Liquid Glass," a translucent material automatically applied to system bars. For custom views, adopt `UIVisualEffectView` with `UIGlassEffect` to match the system aesthetic [ios_26_design_and_features.liquid_glass_material[0]][52].

### Adaptive Navigation
Tab bars on iPhone now float and minimize on scroll by default. On iPad, `UITabBarController` automatically adapts between a tab bar and a sidebar. Ensure your content insets handle these floating elements correctly by respecting the safe area [ios_26_design_and_features.adaptive_navigation[0]][52].

### Typed Notifications
`NotificationCenter` now supports strongly-typed messages (`NotificationCenter.Message`), replacing string-based names and `userInfo` dictionaries. This improves type safety for system events like keyboard updates.

## References

1. *TN3187: Migrating to the UIKit scene-based life cycle | Apple Developer Documentation*. https://developer.apple.com/documentation/technotes/tn3187-migrating-to-the-uikit-scene-based-life-cycle
2. *Explore concurrency in SwiftUI - WWDC25 - Videos - Apple Developer*. https://developer.apple.com/videos/play/wwdc2025/266/
3. *Keyboards and input | Apple Developer Documentation*. https://developer.apple.com/documentation/uikit/keyboards-and-input
4. *reconfigureItems(at:) | Apple Developer Documentation*. https://developer.apple.com/documentation/uikit/uicollectionview/reconfigureitems(at:)
5. *WWDC 2025 What’s new in UIKit - DEV Community*. https://dev.to/arshtechpro/wwdc-2025-whats-new-in-uikit-1kj4
6. *Automatic observation tracking | Apple Developer Documentation*. https://developer.apple.com/documentation/uikit/automatic-observation-tracking
7. *viewDidLoad() | Apple Developer Documentation*. https://developer.apple.com/documentation/uikit/uiviewcontroller/viewdidload()
8. *viewDidLayoutSubviews() | Apple Developer Documentation*. https://developer.apple.com/documentation/uikit/uiviewcontroller/viewdidlayoutsubviews()
9. *Auto Layout Guide: Changing Constraints*. https://developer.apple.com/library/archive/documentation/UserExperience/Conceptual/AutolayoutPG/ModifyingConstraints.html
10. *UIViewController | Apple Developer Documentation*. https://developer.apple.com/documentation/UIKit/UIViewController
11. *animate(alongsideTransition:completion:) | Apple Developer Documentation*. https://developer.apple.com/documentation/uikit/uiviewcontrollertransitioncoordinator/animate(alongsidetransition:completion:)
12. *Easy way to detect a retain cycle in a view controller | Sarunw*. https://sarunw.com/posts/easy-way-to-detect-retain-cycle-in-view-controller/
13. *Understanding Memory Graph Debugger in Xcode - DEV Community*. https://dev.to/arshtechpro/understanding-memory-graph-debugger-in-xcode-your-guide-to-catching-memory-leaks-274
14. *220_Auto Layout Performance*. https://devstreaming-cdn.apple.com/videos/wwdc/2018/220f49ijgby0rma/220/220_high_performance_auto_layout.pdf
15. *UIStackView - NSHipster*. https://nshipster.com/uistackview/
16. *layoutIfNeeded() | Apple Developer Documentation*. https://developer.apple.com/documentation/uikit/uiview/layoutifneeded()
17. *NSDiffableDataSourceSnapshot | Apple Developer Documentation*. https://developer.apple.com/documentation/uikit/nsdiffabledatasourcesnapshot-swift.struct
18. *applySnapshotUsingReloadData(_:) | Apple Developer Documentation*. https://developer.apple.com/documentation/uikit/uicollectionviewdiffabledatasource-9tqpa/applysnapshotusingreloaddata(_:)
19. *Implementing modern collection views*. https://developer.apple.com/documentation/UIKit/implementing-modern-collection-views
20. *UICollectionView.CellRegistration | Apple Developer Documentation*. https://developer.apple.com/documentation/uikit/uicollectionview/cellregistration
21. *UIListContentConfiguration | Apple Developer Documentation*. https://developer.apple.com/documentation/UIKit/UIListContentConfiguration-swift.struct
22. *configurationUpdateHandler*. https://developer.apple.com/documentation/uikit/uicollectionviewcell/configurationupdatehandler-7rqbu
23. *collectionView(_:prefetchItemsAt:) | Apple Developer Documentation*. https://developer.apple.com/documentation/uikit/uicollectionviewdatasourceprefetching/collectionview(_:prefetchitemsat:)
24. *Prefetching collection view data | Apple Developer Documentation*. https://developer.apple.com/documentation/UIKit/prefetching-collection-view-data
25. *UINavigationController | Apple Developer Documentation*. https://developer.apple.com/documentation/uikit/uinavigationcontroller
26. *
    
        A powerful UINavigationController API that you might not know about · Jesse Squires
    
    *. https://www.jessesquires.com/blog/2022/12/15/powerful-navigation-api/
27. *interactiveContentPopGestureRecognizer | Apple Developer Documentation*. https://developer.apple.com/documentation/uikit/uinavigationcontroller/interactivecontentpopgesturerecognizer
28. *UIViewPropertyAnimator | Apple Developer Documentation*. https://developer.apple.com/documentation/uikit/uiviewpropertyanimator
29. *CAAnimation | Apple Developer Documentation*. https://developer.apple.com/documentation/quartzcore/caanimation
30. *assumeIsolated(_:file:line:) | Apple Developer Documentation*. https://developer.apple.com/documentation/swift/mainactor/assumeisolated(_:file:line:)
31. *Practical Memory Management*. https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmPractical.html
32. *Object ownership*. https://developer.apple.com/library/archive/documentation/General/Conceptual/DevPedia-CocoaCore/ObjectOwnership.html
33. *UIHostingControllerSizingOptions | Apple Developer Documentation*. https://developer.apple.com/documentation/swiftui/uihostingcontrollersizingoptions
34. *Use SwiftUI with UIKit - WWDC22 – Vídeos – Apple Developer*. https://developer.apple.com/br/videos/play/wwdc2022/10072/?time=1511
35. *UIViewRepresentable | Apple Developer Documentation*. https://developer.apple.com/documentation/swiftui/uiviewrepresentable
36. *UIViewControllerRepresentable | Apple Developer Documentation*. https://developer.apple.com/documentation/swiftui/uiviewcontrollerrepresentable
37. *prepareForDisplay(completionHandler:)*. https://developer.apple.com/documentation/uikit/uiimage/preparefordisplay(completionhandler:)
38. *prepareThumbnail(of:completionHandler:)*. https://developer.apple.com/documentation/uikit/uiimage/preparethumbnail(of:completionhandler:)
39. *Asynchronously loading images into table and collection ...*. https://developer.apple.com/documentation/UIKit/asynchronously-loading-images-into-table-and-collection-views
40. *followsUndockedKeyboard*. https://developer.apple.com/documentation/uikit/uikeyboardlayoutguide/followsundockedkeyboard
41. *Integrating the virtual keyboard into your app with the keyboard layout guide | WWDC by Sundell & Friends*. https://wwdcbysundell.com/2021/integrating-the-virtual-keyboard-into-your-app
42. *registerForTraitChanges(_:handler:) | Apple Developer Documentation*. https://developer.apple.com/documentation/uikit/uitraitchangeobservable-67e94/registerfortraitchanges(_:handler:)
43. *Color | Apple Developer Documentation*. https://developer.apple.com/design/human-interface-guidelines/color
44. *Dark Mode on iOS 13 - NSHipster*. https://nshipster.com/dark-mode/
45. *UIAppearance | Apple Developer Documentation*. https://developer.apple.com/documentation/UIKit/UIAppearance
46. *accessibilityLabel | Apple Developer Documentation*. https://developer.apple.com/documentation/uikit/uiaccessibilityelement/accessibilitylabel
47. *Delivering an exceptional accessibility experience | Apple Developer Documentation*. https://developer.apple.com/documentation/accessibility/delivering_an_exceptional_accessibility_experience
48. *Diagnosing memory, thread, and crash issues early | Apple Developer Documentation*. https://developer.apple.com/documentation/xcode/diagnosing-memory-thread-and-crash-issues-early
49. *Keep up with the keyboard - WWDC23 - Videos - Apple Developer*. https://developer.apple.com/videos/play/wwdc2023/10281/
50. *Performance Tests | Apple Developer Documentation*. https://developer.apple.com/documentation/xctest/performance-tests
51. *Measuring performance with os_signpost – Donny Wals*. https://www.donnywals.com/measuring-performance-with-os_signpost/
52. *Adopting Liquid Glass | Apple Developer Documentation*. https://developer.apple.com/documentation/TechnologyOverviews/adopting-liquid-glass