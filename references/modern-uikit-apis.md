# Modern UIKit: the complete guide for iOS 18 through iOS 26

UIKit's 2025–2026 releases represent its most significant evolution since diffable data sources. **iOS 26 makes UIKit reactive by default** — the Observation framework now drives automatic view updates, a new `updateProperties()` lifecycle method cleanly separates content from layout, and `.flushUpdates` eliminates manual `layoutIfNeeded()` calls during animation. Meanwhile, the scene lifecycle becomes mandatory, Liquid Glass reshapes the interaction layer, and typed notifications bring compile-time safety to `NotificationCenter`. This guide covers every major API change with production-ready Swift code.

---

## 1. Reactive UIKit via the Observation framework

### The UIObservationTrackingEnabled Info.plist key

Apple quietly shipped automatic observation tracking in **iOS 18** as an opt-in feature, then made it the default in iOS 26. When enabled, UIKit wraps key lifecycle methods in Swift Observation tracking contexts. Any `@Observable` property read during those methods registers a dependency — and when that property changes, UIKit automatically invalidates the relevant method.

For apps targeting iOS 18 through iOS 25, add this to your Info.plist:

```xml
<key>UIObservationTrackingEnabled</key>
<true/>
```

On macOS 15+, the equivalent key is `NSObservationTrackingEnabled`. **On iOS 26, the key is ignored** — tracking is always on.

### Which methods are automatically tracked

UIKit wraps these methods with observation tracking, organized by class:

- **UIView**: `updateProperties()` (iOS 26+), `layoutSubviews()`, `updateConstraints()`, `draw(_:)`
- **UIViewController**: `updateProperties()` (iOS 26+), `viewWillLayoutSubviews()`, `viewDidLayoutSubviews()`, `updateViewConstraints()`, `updateContentUnavailableConfiguration(using:)`
- **UICollectionViewCell / UITableViewCell**: `updateConfiguration(using:)`, `configurationUpdateHandler`
- **UIButton**: `updateConfiguration()`, `configurationUpdateHandler`
- **UIPresentationController**: `containerViewWillLayoutSubviews()`, `containerViewDidLayoutSubviews()`

### How automatic tracking works in practice

Reading an `@Observable` property inside any tracked method registers that property as a dependency. UIKit then calls the appropriate invalidation method (`setNeedsLayout()`, `setNeedsUpdateProperties()`, `setNeedsDisplay()`) when the property changes. The dependency set is **dynamic** — only properties actually read during a given execution are tracked:

```swift
@Observable
class Counter {
    var count: Int = 0
}

class CounterViewController: UIViewController {
    let counter = Counter()
    private let label = UILabel()

    override func viewDidLoad() {
        super.viewDidLoad()
        view.addSubview(label)
        // setup constraints, tap gesture, etc.
    }

    // On iOS 18 (with plist key): use viewWillLayoutSubviews
    // On iOS 26: use updateProperties (preferred)
    override func viewWillLayoutSubviews() {
        super.viewWillLayoutSubviews()
        label.text = "Count: \(counter.count)"  // Dependency recorded
    }

    @objc func increment() {
        counter.count += 1  // Triggers re-run of viewWillLayoutSubviews
    }
}
```

Collection view cells benefit enormously — a single `configurationUpdateHandler` replaces manual snapshot updates:

```swift
@Observable class ListItemModel {
    var icon: UIImage?
    var title: String
    var subtitle: String
}

// In cell provider:
cell.configurationUpdateHandler = { cell, state in
    var content = UIListContentConfiguration.subtitleCell()
    content.image = listItemModel.icon       // Tracked
    content.text = listItemModel.title       // Tracked
    content.secondaryText = listItemModel.subtitle // Tracked
    cell.contentConfiguration = content
}
// Change any property while visible → handler re-runs automatically
```

### The updateProperties() lifecycle method (iOS 26)

iOS 26 introduces `updateProperties()` on both `UIView` and `UIViewController`, providing a **dedicated phase for content and styling** that runs before layout but after trait collection updates. The update cycle now proceeds: traits → `updateProperties()` → `layoutSubviews()` → `draw(_:)`.

This separation matters because **invalidating properties no longer forces a layout pass**, and vice versa:

```swift
@Observable
class BadgeModel {
    var count: Int = 0
    var backgroundColor: UIColor = .systemBlue
}

class BadgeView: UIView {
    let model: BadgeModel
    let label = UILabel()

    // Content and styling — runs independently of layout
    override func updateProperties() {
        super.updateProperties()
        label.text = "\(model.count)"
        backgroundColor = model.backgroundColor
    }

    // Geometry only
    override func layoutSubviews() {
        super.layoutSubviews()
        label.frame = bounds.insetBy(dx: 8, dy: 4)
    }
}
```

The full API surface includes `setNeedsUpdateProperties()` for manual invalidation and `updatePropertiesIfNeeded()` for forcing immediate evaluation.

### Avoiding update storms

An update storm occurs when a tracked method writes to an observable property it also reads, creating an infinite invalidation loop. Follow these rules:

**Rule 1 — Never write to observed properties inside tracked methods:**

```swift
// ❌ Writing to model inside tracked method → infinite loop
override func updateProperties() {
    super.updateProperties()
    model.formattedCount = "Count: \(model.count)"  // WRITES to model
    label.text = model.formattedCount
}

// ✅ Only READ observable properties; write to view properties
override func updateProperties() {
    super.updateProperties()
    label.text = "Count: \(model.count)"  // Reads model, writes to label
}
```

**Rule 2 — Don't create cross-dependencies between updateProperties and layoutSubviews:**

```swift
// ❌ layoutSubviews writes to model → triggers updateProperties → invalidates layout
override func layoutSubviews() {
    super.layoutSubviews()
    model.currentWidth = bounds.width  // Writing to @Observable → loop!
}

// ✅ Use non-observable state for layout feedback
private var cachedWidth: CGFloat = 0  // Plain property, not @Observable
override func layoutSubviews() {
    super.layoutSubviews()
    cachedWidth = bounds.width
}
```

**Rule 3 — Properties with layout-invalidating side effects belong in `updateProperties()`**, not `layoutSubviews()`. Apple's documentation states explicitly: "Some properties aren't appropriate to change during `layoutSubviews()`, for example, properties where setting the value has a side-effect of invalidating the view's layout."

---

## 2. The .flushUpdates animation option replaces layoutIfNeeded

Before iOS 26, animating constraint or observable-driven changes required manually calling `layoutIfNeeded()` inside the animation block. This was error-prone — you had to know which views depended on which properties. **`.flushUpdates` eliminates this entirely** by automatically applying all pending updates (traits, properties, and layout) before and after the animation closure.

### Before and after comparison

```swift
// ❌ iOS 18 and earlier — manual approach
UIView.animate(withDuration: 0.3) {
    heightConstraint.constant = 100
    self.view.layoutIfNeeded()  // Must manually reference view and flush
}

// ✅ iOS 26 — automatic approach
UIView.animate(withDuration: 0.3, options: .flushUpdates) {
    heightConstraint.constant = 100
    // No layoutIfNeeded() needed!
}
```

This becomes especially powerful with observable-driven views:

```swift
@Observable class ConfirmationModel {
    var confirmationColor: UIColor = .gray
}

// The view's updateProperties() reads model.confirmationColor
// ❌ Old way: must reference the specific view
UIView.animate(withDuration: 0.3) {
    myModel.confirmationColor = .green
    confirmationView.layoutIfNeeded()  // Must know which view depends on this
}

// ✅ New way: just change the model
UIView.animate(withDuration: 0.3, options: .flushUpdates) {
    myModel.confirmationColor = .green
    // Don't even need to reference confirmationView!
    // UIKit knows it depends on confirmationColor and handles everything.
}
```

### UIViewPropertyAnimator support

```swift
let animator = UIViewPropertyAnimator(duration: 1, curve: .easeInOut)
animator.flushUpdates = true
animator.addAnimations {
    heightConstraint.constant = 100
}
animator.startAnimation()
```

### Real-world keyboard handling with typed notifications

```swift
let keyboardObserver = NotificationCenter.default.addObserver(
    of: UIScreen.self,
    for: .keyboardWillShow
) { message in
    UIView.animate(
        withDuration: message.animationDuration,
        options: .flushUpdates
    ) {
        bottomConstraint.constant = view.bounds.maxY - message.endFrame.minY
    }
}
```

The critical rule: **only make invalidating state changes inside the animation closure**. Don't mix reads or updates with state changes — just set the new values and let UIKit's automatic tracking handle the rest. The `.flushUpdates` option also implicitly applies to nested animation scopes, ensuring consistent behavior in complex animation hierarchies.

---

## 3. UIScene lifecycle becomes mandatory

### The enforcement timeline

Apple has been tightening the screws on scene lifecycle adoption across three releases. In **iOS 18.4**, UIKit began logging a warning: "This process does not adopt UIScene lifecycle." In **iOS 26**, that warning escalates but the app still launches. In **iOS 27** (building with the iOS 27 SDK), **apps that haven't adopted the scene lifecycle will crash on launch** with an assert before any AppDelegate methods fire.

The enforcement is tied to the SDK you build with, not the OS version the app runs on. Existing binaries compiled with older SDKs continue to work. Only adoption of the scene **lifecycle** is required — supporting multiple scenes is optional.

### Info.plist configuration

Add `UIApplicationSceneManifest` to your Info.plist:

```xml
<key>UIApplicationSceneManifest</key>
<dict>
    <key>UIApplicationSupportsMultipleScenes</key>
    <false/>
    <key>UISceneConfigurations</key>
    <dict>
        <key>UIWindowSceneSessionRoleApplication</key>
        <array>
            <dict>
                <key>UISceneConfigurationName</key>
                <string>Default Configuration</string>
                <key>UISceneDelegateClassName</key>
                <string>$(PRODUCT_MODULE_NAME).SceneDelegate</string>
            </dict>
        </array>
    </dict>
</dict>
```

### Window management moves to the scene delegate

The `UIWindow(frame:)` initializer is deprecated in iOS 26. All window creation must use `UIWindow(windowScene:)`:

```swift
class SceneDelegate: UIResponder, UIWindowSceneDelegate {
    var window: UIWindow?

    func scene(
        _ scene: UIScene,
        willConnectTo session: UISceneSession,
        options connectionOptions: UIScene.ConnectionOptions
    ) {
        guard let windowScene = scene as? UIWindowScene else { return }

        window = UIWindow(windowScene: windowScene)
        window?.rootViewController = MyRootViewController()
        window?.makeKeyAndVisible()

        // State restoration: check for previous activity
        if let activity = session.stateRestorationActivity {
            (window?.rootViewController as? MyRootViewController)?
                .restoreState(from: activity)
        }
    }

    func stateRestorationActivity(for scene: UIScene) -> NSUserActivity? {
        return scene.userActivity
    }
}
```

**Critical migration detail**: after adopting UIScene lifecycle, `launchOptions` in `didFinishLaunchingWithOptions:` will be `nil`. Any launch-options logic must move to `scene(_:willConnectTo:options:)` via `connectionOptions`.

### What stays in AppDelegate vs. what moves

The AppDelegate retains `application(_:didFinishLaunchingWithOptions:)` for non-UI setup, APNs callbacks, `application(_:configurationForConnecting:options:)`, and `application(_:didDiscardSceneSessions:)`. Everything else — the four lifecycle methods (`didBecomeActive`, `willResignActive`, `didEnterBackground`, `willEnterForeground`), URL handling, and user activity handling — moves to `UISceneDelegate` equivalents like `sceneDidBecomeActive(_:)`, `scene(_:openURLContexts:)`, etc.

### TN3187 migration steps

Apple Technical Note TN3187 (published May 2025, revised June 2025) outlines the migration process: determine if migration is needed (missing `UIApplicationSceneManifest` or unimplemented `configurationForConnecting`), choose static (Info.plist) or dynamic (AppDelegate callback) configuration, create a `SceneDelegate` class, migrate lifecycle methods, and test in Split View and Stage Manager on iPad.

---

## 4. Liquid Glass brings a new interaction layer

Liquid Glass is iOS 26's defining design element — a translucent, specular material that floats above content for interactive elements. Standard UIKit controls (tab bars, navigation bars, alerts) adopt it automatically when built with Xcode 26. Custom glass effects use `UIGlassEffect` with `UIVisualEffectView`.

```swift
if #available(iOS 26, *) {
    let glassEffect = UIGlassEffect()
    glassEffect.isInteractive = true  // Scales and bounces on tap
    glassEffect.tintColor = .systemBlue

    let effectView = UIVisualEffectView(effect: glassEffect)
    effectView.frame = CGRect(x: 50, y: 100, width: 300, height: 200)

    let label = UILabel(frame: effectView.bounds)
    label.text = "Glass Surface"
    label.textAlignment = .center
    effectView.contentView.addSubview(label)

    view.addSubview(effectView)
} else {
    // Fallback for older iOS
    let blurEffect = UIBlurEffect(style: .systemMaterial)
    effectView.effect = blurEffect
}
```

For grouping multiple glass elements that merge when proximate, use `UIGlassContainerEffect`:

```swift
if #available(iOS 26, *) {
    let containerEffect = UIGlassContainerEffect()
    containerEffect.spacing = 40.0  // Merge distance threshold

    let containerView = UIVisualEffectView(effect: containerEffect)
    let firstGlass = UIVisualEffectView(effect: UIGlassEffect())
    let secondGlass = UIVisualEffectView(effect: UIGlassEffect())

    containerView.contentView.addSubview(firstGlass)
    containerView.contentView.addSubview(secondGlass)
    view.addSubview(containerView)
}
```

**Key implementation guidance**: use glass sparingly and only for interactive elements, remove custom backgrounds on sheets to let glass texture show through, prefer setting `effect = nil` over changing `alpha` for removal (triggers proper dematerialization animation), and note that larger glass surfaces render more opaque while smaller ones are clearer.

---

## 5. Adaptive navigation with floating tabs and sidebar

iOS 26's tab bar floats over content with Liquid Glass, can minimize on scroll, and **automatically adapts between compact tabs (iPhone) and sidebar (iPad)** when you use the `UITab` API with `UITabGroup`:

```swift
let tabBarController = UITabBarController()
tabBarController.mode = .tabSidebar

tabBarController.tabs = [
    UITab(title: "Home", image: UIImage(systemName: "house"),
          identifier: "Home") { _ in HomeViewController() },

    UITabGroup(
        title: "Library",
        image: UIImage(systemName: "books.vertical"),
        identifier: "Library",
        children: [
            UITab(title: "Albums", image: UIImage(systemName: "photo"),
                  identifier: "Albums") { _ in AlbumsViewController() },
            UITab(title: "Artists", image: UIImage(systemName: "music.mic"),
                  identifier: "Artists") { _ in ArtistsViewController() },
        ]
    ) { _ in LibraryViewController() },

    UISearchTab { _ in
        UINavigationController(rootViewController: SearchViewController())
    }
]

// Enable minimize-on-scroll
tabBarController.tabBarMinimizeBehavior = .onScrollDown
```

Tab groups appear as a single tab in the compact tab bar and expand into sections in the sidebar. Individual tabs support `preferredPlacement` options: `.fixed` (always visible), `.sidebarOnly`, and `.pinned` (trailing edge, icon only). The sidebar shows automatically on iPad landscape and can be toggled via `tabBarController.sidebar.isHidden`.

---

## 6. Typed NotificationCenter.Message eliminates userInfo casting

iOS 26 introduces `NotificationCenter.MainActorMessage` and `NotificationCenter.AsyncMessage` protocols that replace untyped `Notification` objects with strongly-typed, concurrency-safe message structs. **No more `userInfo` dictionary casting at runtime.**

### Defining and posting a typed message

```swift
public struct DownloadDidFinish: NotificationCenter.MainActorMessage {
    public typealias Subject = DownloadManager

    // Strongly typed payload — replaces userInfo
    public let fileURL: URL
    public let success: Bool
}

// Post
NotificationCenter.default.post(
    DownloadDidFinish(fileURL: url, success: true),
    subject: DownloadManager.shared
)
```

### Observing with type safety

```swift
let token = NotificationCenter.default.addObserver(
    of: DownloadManager.shared,
    for: DownloadDidFinish.self
) { message in
    // Closure is main-actor isolated — guaranteed main thread
    print("File: \(message.fileURL), success: \(message.success)")
}
```

`MainActorMessage` guarantees main-thread execution; `AsyncMessage` is `Sendable` for cross-concurrency use. Both protocols bridge with legacy `Notification` by implementing `makeMessage(_:)` and `makeNotification(_:object:)`, so old-style posts are received by new-style observers and vice versa. Apple already ships typed messages for system notifications — for example, `UIScreen.keyboardWillShow` replaces `UIResponder.keyboardWillShowNotification`.

---

## 7. Deprecated-to-modern API migration reference

### traitCollectionDidChange → registerForTraitChanges (iOS 17+)

The old `traitCollectionDidChange` fires for *every* trait change. The new API registers for specific traits only:

```swift
// ❌ Deprecated
override func traitCollectionDidChange(_ previous: UITraitCollection?) {
    super.traitCollectionDidChange(previous)
    if previous?.horizontalSizeClass != traitCollection.horizontalSizeClass {
        configureLayout()
    }
}

// ✅ Modern (iOS 17+)
override func viewDidLoad() {
    super.viewDidLoad()
    registerForTraitChanges([UITraitHorizontalSizeClass.self]) {
        (self: Self, _: UITraitCollection) in
        self.configureLayout()
    }
}
```

### Keyboard notifications → UIKeyboardLayoutGuide (iOS 15+)

**One constraint replaces 30+ lines** of notification handling code:

```swift
// ❌ Deprecated: manual notification + frame math + layoutIfNeeded
NotificationCenter.default.addObserver(self, selector: #selector(keyboardWillShow),
    name: UIResponder.keyboardWillShowNotification, object: nil)

// ✅ Modern (iOS 15+)
view.keyboardLayoutGuide.topAnchor.constraint(
    equalToSystemSpacingBelow: textField.bottomAnchor,
    multiplier: 1.0
).isActive = true
```

### textLabel → UIListContentConfiguration (iOS 14+)

```swift
// ❌ Deprecated
cell.textLabel?.text = "Title"
cell.detailTextLabel?.text = "Subtitle"

// ✅ Modern (iOS 14+)
var content = cell.defaultContentConfiguration()
content.text = "Title"
content.secondaryText = "Subtitle"
content.textProperties.font = .preferredFont(forTextStyle: .headline)
cell.contentConfiguration = content
```

### barTintColor → UINavigationBarAppearance (iOS 13+)

```swift
// ❌ Deprecated — breaks in iOS 15 (transparent scrollEdge default)
navigationBar.barTintColor = .systemRed
navigationBar.titleTextAttributes = [.foregroundColor: UIColor.white]

// ✅ Modern (iOS 13+)
let appearance = UINavigationBarAppearance()
appearance.configureWithOpaqueBackground()
appearance.backgroundColor = .systemRed
appearance.titleTextAttributes = [.foregroundColor: UIColor.white]
navigationBar.standardAppearance = appearance
navigationBar.scrollEdgeAppearance = appearance  // Critical: prevents transparent edge
```

### reloadItems → reconfigureItems (iOS 15+)

`reconfigureItems` reuses the existing cell and re-runs the configuration handler. `reloadItems` dequeues a completely new cell. **Reconfigure is dramatically faster** for data-only changes:

```swift
// ❌ Old: creates a new cell
snapshot.reloadItems([itemID])

// ✅ Modern (iOS 15+): reuses existing cell, re-runs config handler
snapshot.reconfigureItems([itemID])
```

Use `reloadItems` only when you need to change the cell type or require `prepareForReuse()` to fire.

### register + string dequeue → CellRegistration (iOS 14+)

```swift
// ❌ Deprecated: string identifiers, force casting, separate register step
collectionView.register(MyCell.self, forCellWithReuseIdentifier: "MyCell")
let cell = collectionView.dequeueReusableCell(withReuseIdentifier: "MyCell",
    for: indexPath) as! MyCell

// ✅ Modern (iOS 14+): type-safe, no strings, auto-registered
let registration = UICollectionView.CellRegistration<UICollectionViewListCell, Item> {
    cell, indexPath, item in
    var content = cell.defaultContentConfiguration()
    content.text = item.name
    cell.contentConfiguration = content
}
// In data source provider:
collectionView.dequeueConfiguredReusableCell(using: registration, for: indexPath, item: item)
```

### UICollectionViewFlowLayout → CompositionalLayout (iOS 13+)

```swift
// ❌ Old: requires delegate methods for sizing, limited to simple grids
let layout = UICollectionViewFlowLayout()
// + UICollectionViewDelegateFlowLayout for sizeForItemAt...

// ✅ Modern (iOS 14+ for lists): two lines for a full table-view replacement
var config = UICollectionLayoutListConfiguration(appearance: .insetGrouped)
config.headerMode = .supplementary
let layout = UICollectionViewCompositionalLayout.list(using: config)
```

CompositionalLayout supports per-section layouts via `sectionProvider`, orthogonal scrolling with `section.orthogonalScrollingBehavior = .continuous`, and declarative sizing with `.fractionalWidth`, `.absolute`, and `.estimated` dimensions — all without subclassing or delegate methods.

---

## Conclusion

The iOS 18–26 era marks UIKit's transition from an imperative framework to a **reactive, declarative-friendly system**. Three changes stand out as the most impactful for existing codebases.

First, **automatic observation tracking** (available today on iOS 18 with a single plist key) eliminates manual invalidation patterns that have plagued UIKit development for over a decade. The new `updateProperties()` phase in iOS 26 makes the separation of content updates from layout geometry both natural and enforced by the framework.

Second, the **mandatory scene lifecycle** in iOS 27 SDK builds means migration is no longer optional. Apps that haven't adopted `UISceneDelegate` by the iOS 27 timeframe will crash on launch. TN3187 provides a clear migration path, but the work should start now.

Third, the **API modernization table** above represents a decade of accumulated improvements. Apps still using `textLabel`, string-based cell registration, or `FlowLayout` are leaving significant performance and maintainability gains on the table. The modern APIs aren't just newer — they're fundamentally better abstractions that produce less code, fewer bugs, and faster cells.