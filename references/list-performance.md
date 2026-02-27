# Mastering UICollectionView and UITableView scroll performance in Swift

Buttery-smooth scrolling in UIKit demands a disciplined approach across prefetching, cell configuration, layout, and measurement. **The single most impactful insight: iOS 15+ gives you automatic cell prefetching for free just by building with the SDK**, but squandering that headroom with synchronous image decoding, constraint churn, or stale-image race conditions negates the gain entirely. This guide covers every layer of the scroll performance stack — from initiating prefetch requests to profiling hitches in Instruments — with concrete Swift code showing both the traps and the fixes. All guidance draws from Apple's WWDC 2020–2025 sessions, official sample code, and battle-tested community patterns.

---

## 1. Prefetching data before cells appear on screen

`UICollectionViewDataSourcePrefetching` (iOS 10+) provides advance warning of upcoming cells so you can start async work early. The protocol has two methods: `prefetchItemsAt` (required) starts work, and `cancelPrefetchingForItemsAt` (optional) stops it when the user reverses scroll direction.

**The index paths arrive already sorted by geometric distance** — closest items first. You do not need to re-sort them. Apple's documentation states explicitly: *"The order of the index paths provided represents the priority."*

A critical subtlety: `prefetchItemsAt` is **not called for every cell**. Your `cellForItemAt` must independently handle three scenarios: data already loaded, data in-flight, or data never requested.

```swift
// ✅ GOOD: Kick off async work; index paths are already priority-ordered
func collectionView(_ collectionView: UICollectionView,
                    prefetchItemsAt indexPaths: [IndexPath]) {
    for indexPath in indexPaths {
        guard let item = dataSource.itemIdentifier(for: indexPath) else { continue }
        imagePrefetcher.startFetch(for: item.id, url: item.imageURL)
    }
}

func collectionView(_ collectionView: UICollectionView,
                    cancelPrefetchingForItemsAt indexPaths: [IndexPath]) {
    for indexPath in indexPaths {
        guard let item = dataSource.itemIdentifier(for: indexPath) else { continue }
        imagePrefetcher.cancelFetch(for: item.id)
    }
}
```

```swift
// ❌ BAD: Sorting index paths yourself (redundant — Apple already priority-orders them)
func collectionView(_ collectionView: UICollectionView,
                    prefetchItemsAt indexPaths: [IndexPath]) {
    let sorted = indexPaths.sorted()  // Unnecessary work
    for indexPath in sorted { /* ... */ }
}

// ❌ BAD: Blocking the main thread with synchronous work
func collectionView(_ collectionView: UICollectionView,
                    prefetchItemsAt indexPaths: [IndexPath]) {
    for indexPath in indexPaths {
        let image = UIImage(contentsOfFile: paths[indexPath.row])  // Blocks main thread!
        cache[indexPath] = image
    }
}
```

**Do** in prefetch: start `URLSession` data tasks, begin Core Data fetches, kick off `UIImage.prepareForDisplay()` (iOS 15+). **Don't** in prefetch: decode images synchronously, perform UI updates, or assume prefetch will fire for every cell.

Starting with **iOS 15**, simply building with the SDK enables **automatic cell prefetching** — the system detects spare time between commits and pre-creates upcoming cells. This gave apps roughly **2× more time** to prepare each cell with zero code changes (WWDC 2021, session 10252).

---

## 2. Swift Concurrency and the stable-ID task dictionary

IndexPaths are ephemeral. Insert one row above your prefetched content and every stored `IndexPath` key silently points to the wrong item. The fix: **key your `Task` dictionary by stable item identifiers**, never by `IndexPath`.

```swift
// ✅ GOOD: Full prefetch controller using Swift Concurrency with stable IDs
@MainActor
final class ImagePrefetchController<ItemID: Hashable & Sendable> {

    private var tasks: [ItemID: Task<UIImage?, Never>] = [:]
    private var cache: [ItemID: UIImage] = [:]

    func startPrefetching(for items: [(id: ItemID, url: URL)]) {
        for item in items {
            guard cache[item.id] == nil, tasks[item.id] == nil else { continue }

            tasks[item.id] = Task {
                do {
                    let (data, _) = try await URLSession.shared.data(from: item.url)
                    try Task.checkCancellation()
                    guard let image = UIImage(data: data) else { return nil }
                    let prepared = await image.byPreparingForDisplay()
                    try Task.checkCancellation()
                    cache[item.id] = prepared
                    tasks[item.id] = nil          // Clean up handle
                    return prepared
                } catch {
                    tasks[item.id] = nil
                    return nil
                }
            }
        }
    }

    func cancelPrefetching(for ids: [ItemID]) {
        for id in ids {
            tasks[id]?.cancel()
            tasks[id] = nil
        }
    }

    func cachedImage(for id: ItemID) -> UIImage? { cache[id] }

    func image(for id: ItemID) async -> UIImage? {
        if let cached = cache[id] { return cached }
        return await tasks[id]?.value
    }
}
```

The `@MainActor` annotation is crucial here. Because unstructured `Task { }` inherits the caller's actor context, both the task body and the call site access the same `tasks` dictionary on the main actor — **no data races**. Apple demonstrated this exact pattern at WWDC 2021 ("Explore Structured Concurrency in Swift").

Bridge the IndexPath-based delegate to stable IDs in one line via the diffable data source:

```swift
// ✅ GOOD: Convert IndexPath → stable ID at the protocol boundary
func collectionView(_ collectionView: UICollectionView,
                    prefetchItemsAt indexPaths: [IndexPath]) {
    let items = indexPaths.compactMap { dataSource.itemIdentifier(for: $0) }
                          .map { (id: $0.id, url: $0.imageURL) }
    prefetchController.startPrefetching(for: items)
}
```

```swift
// ❌ BAD: Keying tasks by IndexPath — breaks on any data mutation
var tasks: [IndexPath: Task<UIImage?, Never>] = [:]

func collectionView(_ collectionView: UICollectionView,
                    prefetchItemsAt indexPaths: [IndexPath]) {
    for ip in indexPaths {
        tasks[ip] = Task { await loadImage(for: ip) }
        // After an insertion at row 0, tasks[IndexPath(row:3,...)]
        // now refers to the WRONG item.
    }
}
```

Always call `Task.checkCancellation()` before expensive steps (image decode, network parse) so cooperative cancellation takes effect promptly. And always nil out finished task handles — otherwise the dictionary leaks `Task` objects indefinitely.

---

## 3. The cancel-clear-verify pattern that prevents stale images

The most common scroll bug in UIKit: cell A starts loading image X, gets recycled to display item Y, and then the completion handler from image X fires — writing the wrong image into a cell the user now associates with Y. The flash of incorrect content erodes trust.

```swift
// ❌ BAD: Classic race condition — stale image on recycled cell
func collectionView(_ collectionView: UICollectionView,
                    cellForItemAt indexPath: IndexPath) -> UICollectionViewCell {
    let cell = collectionView.dequeueReusableCell(
        withReuseIdentifier: "ImageCell", for: indexPath) as! ImageCell
    let item = items[indexPath.row]

    Task {
        let image = await downloadImage(from: item.imageURL)
        cell.imageView.image = image   // 💥 cell may now represent a DIFFERENT item
    }
    return cell
}
```

The fix is a three-step discipline — **cancel, clear, verify**:

```swift
// ✅ GOOD: Cancel-clear-verify pattern with Swift Concurrency
final class ImageCell: UICollectionViewCell {
    let imageView = UIImageView()
    private var loadTask: Task<Void, Never>?
    private var representedID: String?

    override func prepareForReuse() {
        super.prepareForReuse()
        loadTask?.cancel()          // (a) Cancel in-flight work
        loadTask = nil
        imageView.image = nil       // (b) Clear stale content
        representedID = nil
    }

    func configure(with item: Item, prefetcher: ImagePrefetchController<String>) {
        representedID = item.id

        // Fast path: image already cached
        if let cached = prefetcher.cachedImage(for: item.id) {
            imageView.image = cached
            return
        }

        // Slow path: async load with identity verification
        loadTask = Task { [weak self] in
            let image = await prefetcher.image(for: item.id)
            // (c) Verify cell still represents the same item
            guard self?.representedID == item.id else { return }
            self?.imageView.image = image
        }
    }
}
```

Step **(a)** — cancelling in `prepareForReuse` — prevents wasted CPU and network bandwidth. Step **(b)** — clearing the image view — eliminates the visual flash of the old image. Step **(c)** — the identity guard after `await` — is the critical safety net. Even if cancellation doesn't propagate in time, the stale result is silently discarded.

Apple's own sample code ("Prefetching Collection View Data") uses a `representedIdentifier` string on the cell for exactly this guard. In the modern diffable data source world, the item's stable `id` is the natural choice.

An alternative Apple-endorsed approach (WWDC 2021) avoids touching cells directly from async callbacks altogether: **download the image, cache it, then call `reconfigureItems`** to re-run the cell registration handler with the now-cached image already available synchronously.

---

## 4. Why reconfigureItems almost always beats reloadItems

`reconfigureItems` (iOS 15+, on `NSDiffableDataSourceSnapshot`) updates a cell's content **in-place** by re-invoking your cell registration handler on the existing cell instance. **`prepareForReuse` is not called.** The cell's internal state — running animations, scroll positions in embedded scroll views, expanded/collapsed toggles — survives intact.

`reloadItems`, by contrast, dequeues a **fresh cell** from the reuse pool: full lifecycle with `prepareForReuse`, configuration from scratch, and loss of all transient state. Worse, it **discards prefetched cells** that the system already prepared, wasting the automatic cell prefetching work iOS 15 did for you.

| Behavior | `reconfigureItems` | `reloadItems` |
|---|---|---|
| Dequeues new cell | No — reuses existing | Yes — full lifecycle |
| Calls `prepareForReuse` | No | Yes |
| Preserves cell state / animations | Yes | No |
| Preserves prefetched cells | Yes | No — discards them |
| Can change cell type | No | Yes |
| Self-sizes after update | Yes | Yes |

Apple's guidance is unambiguous: *"For optimal performance, choose to reconfigure items instead of reloading items unless you have an explicit need to replace the existing cell with a new cell."*

```swift
// ✅ GOOD: Content-only update via reconfigureItems — preserves cell + prefetch work
func markPostAsRead(_ postID: Post.ID) {
    postStore[postID]?.isRead = true
    var snapshot = dataSource.snapshot()
    snapshot.reconfigureItems([postID])             // Re-runs cell registration handler
    dataSource.apply(snapshot, animatingDifferences: true)
}
```

```swift
// ❌ BAD: Using reloadItems when only content changed — wasteful
func markPostAsRead(_ postID: Post.ID) {
    postStore[postID]?.isRead = true
    var snapshot = dataSource.snapshot()
    snapshot.reloadItems([postID])    // Dequeues fresh cell, discards prefetched cells
    dataSource.apply(snapshot)
}
```

The `reconfigureItems` pattern shines for async image loading. Your cell registration handler checks the cache: if the image is there, set it; if not, show a placeholder and kick off a download whose completion triggers another `reconfigureItems` call:

```swift
let cellRegistration = UICollectionView.CellRegistration<PhotoCell, Photo.ID> {
    [weak self] cell, indexPath, photoID in
    guard let self, let photo = self.photoStore[photoID] else { return }

    cell.titleLabel.text = photo.title

    if let image = self.imageCache[photoID] {
        cell.imageView.image = image
    } else {
        cell.imageView.image = UIImage(systemName: "photo")
        self.downloadImage(for: photo) {
            // When download completes, reconfigure — NOT direct cell mutation
            var snap = self.dataSource.snapshot()
            snap.reconfigureItems([photoID])
            self.dataSource.apply(snap, animatingDifferences: false)
        }
    }
}
```

Use `reloadItems` only when you need to **swap cell types** entirely (e.g., switching from a compact cell to an expanded media cell backed by a different registration).

One more iOS 15 data source fix worth noting: `apply(snapshot, animatingDifferences: false)` previously behaved like `reloadData()`, discarding and recreating all cells. **In iOS 15+, it correctly diffs**, making non-animated updates cheap. If you truly need the old "nuke everything" behavior, call the explicit `applySnapshotUsingReloadData(_:)`.

---

## 5. Performance traps hiding inside your cell provider

The cell provider closure runs on the main thread during scroll. Every millisecond spent there is a millisecond closer to a hitch. Three categories of traps appear repeatedly.

### Expensive objects created per-cell

`DateFormatter` initialization costs **3–5 ms** per instance. Creating one for every cell during a fast scroll adds up fast.

```swift
// ❌ BAD: New DateFormatter for every cell — ~3-5ms overhead each
let cellRegistration = UICollectionView.CellRegistration<PostCell, Post> {
    cell, indexPath, post in
    let formatter = DateFormatter()              // Expensive allocation
    formatter.dateStyle = .medium
    formatter.timeStyle = .short
    cell.dateLabel.text = formatter.string(from: post.date)
}
```

```swift
// ✅ GOOD: Static cached formatter — allocated once, reused forever
extension DateFormatter {
    static let mediumDateTime: DateFormatter = {
        let f = DateFormatter()
        f.dateStyle = .medium
        f.timeStyle = .short
        return f
    }()
}

// In cell provider:
cell.dateLabel.text = DateFormatter.mediumDateTime.string(from: post.date)
```

The same principle applies to `NSAttributedString` construction, `NumberFormatter`, and `MeasurementFormatter`. **Pre-compute formatted strings in your view model** so the cell provider does nothing but assign pre-built values:

```swift
// ✅ GOOD: View model pre-computes everything expensive
struct PostViewModel {
    let attributedTitle: NSAttributedString
    let formattedDate: String
    let thumbnailURL: URL

    init(post: Post) {
        let title = NSMutableAttributedString(
            string: post.title,
            attributes: [.font: UIFont.boldSystemFont(ofSize: 16)])
        let sub = NSAttributedString(
            string: "\n\(post.subtitle)",
            attributes: [.font: UIFont.systemFont(ofSize: 13),
                         .foregroundColor: UIColor.secondaryLabel])
        title.append(sub)
        self.attributedTitle = title
        self.formattedDate = DateFormatter.mediumDateTime.string(from: post.date)
        self.thumbnailURL = post.thumbnailURL
    }
}

// Cell provider becomes trivially cheap:
cell.titleLabel.attributedText = viewModel.attributedTitle
cell.dateLabel.text = viewModel.formattedDate
```

### Constraint churn during configuration

Adding and removing Auto Layout constraints in `cellForItemAt` forces a full constraint-system re-solve — potentially **O(n³)** for complex hierarchies.

```swift
// ❌ BAD: Removing and re-adding constraints every configuration
func configure(with post: Post) {
    imageView.constraints.forEach { imageView.removeConstraint($0) }
    if post.hasImage {
        NSLayoutConstraint.activate([
            imageView.heightAnchor.constraint(equalToConstant: 200)
        ])
    } else {
        NSLayoutConstraint.activate([
            imageView.heightAnchor.constraint(equalToConstant: 0)
        ])
    }
}
```

```swift
// ✅ GOOD: Pre-create both constraint sets once in init, toggle isActive
final class PostCell: UICollectionViewCell {
    private var imageVisibleHeight: NSLayoutConstraint!
    private var imageHiddenHeight: NSLayoutConstraint!

    override init(frame: CGRect) {
        super.init(frame: frame)
        imageVisibleHeight = imageView.heightAnchor.constraint(equalToConstant: 200)
        imageHiddenHeight  = imageView.heightAnchor.constraint(equalToConstant: 0)
    }

    func configure(with post: Post) {
        imageVisibleHeight.isActive = post.hasImage
        imageHiddenHeight.isActive  = !post.hasImage
        imageView.isHidden = !post.hasImage
    }
}
```

Even better: **use separate cell registrations** for structurally different layouts (media cells vs. text-only cells) so the constraint graph is fixed per registration.

### Deeply nested UIStackViews

UIStackView translates its properties into Auto Layout constraints. Nesting stack views **3–4 levels deep** causes an explosion in constraint count. Apple's own radar (rdar://24206043) documented severe scroll degradation from nested stack views in collection view cells.

```swift
// ❌ BAD: 4 levels of nested stack views — layout cost explodes
let outer = UIStackView(arrangedSubviews: [
    UIStackView(arrangedSubviews: [          // Level 2
        UIStackView(arrangedSubviews: [      // Level 3
            UIStackView(arrangedSubviews: [  // Level 4 — very expensive
                likesLabel, commentsLabel
            ])
        ])
    ])
])
```

```swift
// ✅ GOOD: Flatten to ≤2 levels, or use manual layout for hot cells
override func layoutSubviews() {
    super.layoutSubviews()
    let bounds = contentView.bounds
    let pad: CGFloat = 12
    thumbnailView.frame = CGRect(x: pad, y: pad, width: 60, height: 60)
    let labelX = thumbnailView.frame.maxX + pad
    let labelW = bounds.width - labelX - pad
    titleLabel.frame = CGRect(x: labelX, y: pad, width: labelW, height: 22)
    subtitleLabel.frame = CGRect(x: labelX, y: 38, width: labelW, height: 18)
}
```

Manual frame layout in `layoutSubviews` is **O(1)** with zero constraint-solving overhead — the right choice for your highest-frequency cells.

One final trap: **creating cell registrations inside the cell provider closure**. iOS 15 added a runtime exception for this because it causes cells to never be reused, leaking memory linearly with scroll distance. Always create registrations once, outside the closure.

---

## 6. Self-sizing cells without layout thrashing

Self-sizing cells use Auto Layout to compute height on demand. The system calls `systemLayoutSizeFitting` — which creates a **throwaway Auto Layout engine**, solves it, returns the size, and discards it. This is expensive. Done naively, it causes visible scroll-bar jitter and content jumps.

### Accurate estimates prevent jump cuts

`estimatedRowHeight` (UITableView) and `estimatedItemSize` (UICollectionViewFlowLayout) tell the system the approximate height of off-screen cells. The system uses this to calculate total content size and scroll indicator position. **Wildly inaccurate estimates** cause the scroll indicator to jump as real heights replace estimates.

```swift
// ✅ GOOD: Provide accurate estimates based on your typical cell
tableView.estimatedRowHeight = 88.0    // Measured average of your actual cells
tableView.rowHeight = UITableView.automaticDimension

// For Dynamic Type, scale the estimate with font size
let baseEstimate: CGFloat = 88
let bodyFont = UIFont.preferredFont(forTextStyle: .body)
tableView.estimatedRowHeight = baseEstimate * (bodyFont.pointSize / 17.0)
```

For UICollectionView with flow layout, set `estimatedItemSize` to a representative size rather than `automaticSize` when you know the approximate dimensions.

### Cache heights to avoid redundant computation

Since `systemLayoutSizeFitting` discards its engine after each call, caching is essential for scrolling cells you've already measured:

```swift
// ✅ GOOD: Height cache keyed by stable item ID
final class HeightCache {
    private var cache: [String: CGFloat] = [:]

    func height(for id: String) -> CGFloat? { cache[id] }

    func store(_ height: CGFloat, for id: String) { cache[id] = height }

    func invalidate() { cache.removeAll() }          // On rotation, Dynamic Type change
    func invalidate(id: String) { cache[id] = nil }  // On content update
}
```

Use this cache in `preferredLayoutAttributesFitting` for collection view cells:

```swift
// ✅ GOOD: Custom sizing with caching in preferredLayoutAttributesFitting
override func preferredLayoutAttributesFitting(
    _ layoutAttributes: UICollectionViewLayoutAttributes
) -> UICollectionViewLayoutAttributes {
    guard let itemID = currentItemID else { return layoutAttributes }

    if let cached = heightCache.height(for: itemID) {
        layoutAttributes.frame.size.height = cached
        return layoutAttributes
    }

    setNeedsLayout()
    layoutIfNeeded()
    let targetSize = CGSize(width: layoutAttributes.size.width,
                            height: UIView.layoutFittingCompressedSize.height)
    let computedSize = contentView.systemLayoutSizeFitting(
        targetSize,
        withHorizontalFittingPriority: .required,
        verticalFittingPriority: .fittingSizeLevel)

    layoutAttributes.frame.size.height = ceil(computedSize.height)
    heightCache.store(layoutAttributes.frame.size.height, for: itemID)
    return layoutAttributes
}
```

**Invalidate the cache** on trait collection changes (Dynamic Type, device rotation) and when specific item content changes. iOS 16 introduced `selfSizingInvalidation` on cells, which can detect Auto Layout changes in the content view and automatically trigger re-sizing — set it to `.enabledIncludingConstraints` for automatic invalidation when constraints change.

For maximum performance on hot cells, consider computing height **manually** from content (e.g., `NSString.boundingRect` for text) instead of using `systemLayoutSizeFitting` at all.

---

## 7. Profiling scroll hitches with Instruments step by step

A **hitch** is a frame displayed later than expected. Apple measures hitches not in FPS (which is misleading) but in **hitch time ratio**: milliseconds of delay per second of scrolling.

| Hitch ratio | Rating | Meaning |
|---|---|---|
| **< 5 ms/s** | ✅ Good | Imperceptible to most users |
| **5–10 ms/s** | ⚠️ Warning | Users notice interruptions |
| **> 10 ms/s** | 🔴 Critical | Severely degraded experience |

Two hitch categories exist. **Commit hitches** occur when your app's main thread takes too long in the commit phase (layout → display → prepare → commit). The render server receives the layer tree late and has nothing to composite. **Render hitches** occur when the GPU/render server can't finish compositing in time — typically caused by offscreen rendering passes from shadows without `shadowPath`, complex masks, or excessive layer blending.

### Concrete profiling workflow

**Step 1.** In Xcode, press ⌘I (Product → Profile). Select the **Animation Hitches** template.

**Step 2.** Choose a **physical device** as the target. Simulator results are unreliable for performance measurements.

**Step 3.** Click Record, then scroll through your content in the app for 10–15 seconds. Stop recording.

**Step 4.** Examine the **Hitches** track. Red/orange bars indicate detected hitches. Click any bar to see its duration, type (commit or render), and the acceptable latency window it exceeded.

**Step 5.** For **commit hitches**: expand the Commits track, find your app's process, drill into the main thread. The integrated Time Profiler shows exactly which methods consumed the frame budget — look for `layoutSubviews`, image decoding (`ImageIO`), date formatting, or Auto Layout solving in the heaviest frames.

**Step 6.** For **render hitches**: check the Buffer Count column (values above 2 indicate the system resorted to triple buffering). Use Xcode's View Debugger with **Editor → Show Layers** and enable **"Show Performance Optimizations"** to highlight offscreen-rendered layers.

**Step 7.** Fix the identified bottleneck, re-profile, and verify the hitch ratio dropped below **5 ms/s**.

Key Instruments tips from Apple: prefer `setNeedsLayout()` over `layoutIfNeeded()` — the latter forces immediate layout within the current transaction, expanding commit duration. Use `isHidden` instead of removing/adding views. Ensure views only invalidate themselves or children, never siblings or parents.

---

## 8. Automated hitch detection with XCTHitchMetric and performance tests

Manual profiling catches problems during development. Automated scroll performance tests catch **regressions** in CI. Apple provides `XCTOSSignpostMetric` (Xcode 12+) and the newer `XCTHitchMetric` for exactly this purpose.

### Writing a scroll performance test

```swift
// ✅ GOOD: Automated scroll hitch test with baseline capability
func testScrollPerformance() throws {
    let app = XCUIApplication()
    app.launch()

    let collection = app.collectionViews.firstMatch
    let options = XCTMeasureOptions()
    options.invocationOptions = [.manuallyStop]

    measure(metrics: [XCTOSSignpostMetric.scrollingAndDecelerationMetric],
            options: options) {
        collection.swipeUp(velocity: .fast)
        stopMeasuring()
        collection.swipeDown(velocity: .fast)    // Reset state for next iteration
    }
}
```

The `manuallyStop` + `stopMeasuring()` pattern ensures the reset swipe-down isn't counted in the measurement. The `measure` block runs **5 iterations** by default (plus one warm-up), reporting duration, hitch count, total hitch time, hitch time ratio, frame rate, and frame count.

Using the dedicated `XCTHitchMetric` (available in recent Xcode versions):

```swift
func testScrollHitchRate() {
    let app = XCUIApplication()
    app.launch()

    let scrollView = app.collectionViews.firstMatch

    measure(metrics: [XCTHitchMetric()]) {
        scrollView.swipeUp(velocity: .fast)
    }
}
```

For custom animations, instrument your code with `os_signpost` and create a targeted metric:

```swift
// In app code:
import os.signpost
let log = OSLog(subsystem: "com.app", category: .pointsOfInterest)

os_signpost(.animationBegin, log: log, name: "CardExpand")
UIView.animate(withDuration: 0.3) { /* ... */ } completion: { _ in
    os_signpost(.end, log: log, name: "CardExpand")
}

// In test code:
func testCardExpandHitches() {
    let metric = XCTOSSignpostMetric(
        subsystem: "com.app", category: "PointsOfInterest", name: "CardExpand")
    measure(metrics: [metric]) {
        app.cells.firstMatch.tap()
    }
}
```

### Test scheme configuration is mandatory

Performance tests run in a debug build with sanitizers enabled will produce meaningless results. Apple explicitly requires a **dedicated test scheme**:

- Build Configuration → **Release**
- Uncheck **Debug executable**
- Disable **Automatic Screenshots**
- Disable **Code Coverage**
- Turn off **all** diagnostics: Address Sanitizer, Thread Sanitizer, Undefined Behavior Sanitizer, Zombie Objects, Guard Malloc

### Setting and tracking baselines

After running a performance test on a physical device, click the gray diamond icon in the test navigator to view metrics. Select a metric (e.g., "Hitch Time Ratio") and click **Set Baseline**. Baselines are device-model-specific and stored in `xcshareddata/` so they can be committed to source control. Future test runs that exceed the baseline plus allowed deviation will **fail automatically**, catching regressions in CI.

In production, **MetricKit** (iOS 14+) collects hitch data from real user devices, and Xcode Organizer displays scroll hitch metrics across app versions — closing the loop from development to production monitoring.

---

## Conclusion

Scroll performance in UIKit is not a single optimization but a **layered discipline**. At the data layer, key your prefetch tasks by stable item IDs and implement cooperative cancellation with `Task.checkCancellation()`. At the cell layer, enforce cancel-clear-verify to prevent stale images, prefer `reconfigureItems` over `reloadItems` to preserve cell state and prefetch work, and pre-compute all expensive formatting in view models. At the layout layer, provide accurate size estimates, cache computed heights, and flatten deep stack view hierarchies — or use manual layout for your hottest cells. At the measurement layer, target a **hitch ratio below 5 ms/s**, automate regression detection with `XCTOSSignpostMetric` or `XCTHitchMetric`, and profile commit vs. render hitches separately in Instruments.

The iOS 15–18 era brought genuinely impactful automatic improvements — cell prefetching, efficient snapshot diffing, `byPreparingForDisplay()`, self-sizing invalidation, and 2–3× faster collection view internals in iOS 17. But these improvements only help if your cell provider doesn't squander the headroom. The code patterns in this guide represent the current state of the art for UIKit scroll performance — tested against Apple's own WWDC sessions (2020 session 10077, 2021 session 10252, 2021 Tech Talks 10855–10857, 2023 session 10055), official sample code, and production-hardened community practice.
---

## Summary Checklist

- [ ] `UICollectionViewDataSourcePrefetching` implemented for media-heavy lists
- [ ] Prefetch tasks stored by stable item ID (not IndexPath) in a dictionary
- [ ] `cancelPrefetchingForItemsAt` cancels tasks for scrolled-past items
- [ ] `prepareForReuse` cancels in-flight image loading tasks
- [ ] `prepareForReuse` clears `imageView.image` to prevent stale content flash
- [ ] Image loading completion verifies cell identity before applying (cancel/clear/verify pattern)
- [ ] No heavy computation in cell provider / `cellForItemAt` — offload to prefetch or background
- [ ] `reconfigureItems` used instead of `reloadItems` for in-place content updates
- [ ] UIStackView nesting kept to ≤2 levels in cells for scroll performance
- [ ] Estimated dimensions set on layout for self-sizing cells
