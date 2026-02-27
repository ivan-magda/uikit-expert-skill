# UIKit best practices for 2024–2025 in Swift

This guide covers the correctness, performance, and modern API patterns every UIKit developer needs to write production-quality iOS apps targeting **iOS 15–18**. Each section focuses on practical pitfalls that cause real bugs — not theoretical architecture — with annotated Swift code showing both wrong and correct patterns. The emphasis is on what has changed since iOS 15, what Apple recommends as of WWDC 2023/2024, and what AI code generators consistently get wrong.

---

## 1. View controller lifecycle: the correct mental model

The single most important thing to understand about `UIViewController` is the **ordering and guarantees** of each lifecycle method. Getting this wrong causes geometry bugs, redundant work, and crashes.

### Complete lifecycle ordering

```
init(nibName:bundle:) / init(coder:)
    ↓
loadView()                          ← Create root view (override only for custom root views)
    ↓
viewDidLoad()                       ← One-time setup; geometry NOT final
    ↓
viewWillAppear(_:)                  ← Every appearance; view NOT in hierarchy
    ↓
viewIsAppearing(_:)                 ← iOS 13+ (backdeployed); geometry IS final  ★
    ↓
viewWillLayoutSubviews()            ← May fire multiple times
    ↓
viewDidLayoutSubviews()             ← May fire multiple times
    ↓
viewDidAppear(_:)                   ← View visible and interactive
    ↓
viewWillDisappear(_:)
    ↓
viewDidDisappear(_:)
```

**`viewIsAppearing(_:)`** is the key addition announced at WWDC 2023. Although it was formally introduced with iOS 17, **it back-deploys to iOS 13+**, making it immediately usable in virtually all production apps. It fires after the view is added to the hierarchy with accurate traits and geometry, but before the view is visible on screen — making it the correct place for geometry-dependent setup.

| State | viewWillAppear | viewIsAppearing |
|---|:---:|:---:|
| View in hierarchy | ✘ | ✓ |
| Traits finalized | ✘ | ✓ |
| Geometry accurate | ✘ | ✓ |
| Transition coordinator available | ✓ | ✘ |

### What belongs where

**`loadView()`** — Override only when providing a completely custom root view. Never call `super.loadView()` when overriding. Never call `loadView()` directly.

```swift
// ✅ Correct loadView override
override func loadView() {
    view = ProfileView() // Do NOT call super
}
```

**`viewDidLoad()`** — One-time setup: add subviews, activate constraints, configure delegates, register cells. Do **not** perform geometry-dependent work here.

```swift
// ❌ Wrong: geometry is unreliable in viewDidLoad
override func viewDidLoad() {
    super.viewDidLoad()
    gradientLayer.frame = view.bounds // Likely incorrect dimensions
}

// ✅ Correct: constraints don't depend on final geometry
override func viewDidLoad() {
    super.viewDidLoad()
    view.addSubview(tableView)
    tableView.translatesAutoresizingMaskIntoConstraints = false
    NSLayoutConstraint.activate([
        tableView.topAnchor.constraint(equalTo: view.safeAreaLayoutGuide.topAnchor),
        tableView.leadingAnchor.constraint(equalTo: view.leadingAnchor),
        tableView.trailingAnchor.constraint(equalTo: view.trailingAnchor),
        tableView.bottomAnchor.constraint(equalTo: view.bottomAnchor)
    ])
}
```

**`viewIsAppearing(_:)`** — Geometry-dependent work, trait-based layout decisions, scrolling to a selected item. Called once per appearance transition (unlike `viewDidLayoutSubviews` which fires multiple times).

```swift
override func viewIsAppearing(_ animated: Bool) {
    super.viewIsAppearing(animated)
    gradientLayer.frame = view.bounds
    updateLayoutForSizeClass(traitCollection.horizontalSizeClass)
    if let selected = selectedIndexPath {
        tableView.scrollToRow(at: selected, at: .middle, animated: false)
    }
}
```

**`viewDidLayoutSubviews()`** — Fires every time `layoutSubviews()` runs. Use only for lightweight frame-based layer adjustments. Never add subviews or constraints here.

**`viewWillAppear(_:)`** — Now limited in usefulness. Use only for transition coordinator animations and balanced notification subscriptions.

Always call `super` in every lifecycle override. Forgetting `super.viewWillAppear(animated)` silently breaks UIKit internal state.

---

## 2. Auto Layout: batch, prioritize, and never churn

### Batch activation is not optional

**`NSLayoutConstraint.activate(_:)` is significantly more efficient** than setting `.isActive = true` individually. Internally, batch activation calls `NSISEngine.withAutomaticOptimizationsDisabled`, preventing the constraint solver from performing intermediate calculations between each constraint.

```swift
// ❌ Triggers intermediate solver work per constraint
label.topAnchor.constraint(equalTo: view.topAnchor).isActive = true
label.leadingAnchor.constraint(equalTo: view.leadingAnchor).isActive = true

// ✅ Single solver pass
NSLayoutConstraint.activate([
    label.topAnchor.constraint(equalTo: view.topAnchor),
    label.leadingAnchor.constraint(equalTo: view.leadingAnchor),
    label.trailingAnchor.constraint(equalTo: view.trailingAnchor),
    label.bottomAnchor.constraint(equalTo: view.bottomAnchor)
])
```

### translatesAutoresizingMaskIntoConstraints

Every programmatically created view defaults to `translatesAutoresizingMaskIntoConstraints = true`. This generates autoresizing-mask-based constraints that conflict with your manual constraints. **Set it to `false` before activating constraints.** Views created in Interface Builder with constraints have this set automatically.

### Priority management

Never change a constraint's priority from or to `.required` (1000) while it's active — this crashes. If you need to toggle priorities at runtime, start at **999** instead of 1000.

```swift
let widthConstraint = label.widthAnchor.constraint(equalToConstant: 200)
widthConstraint.priority = UILayoutPriority(999) // Breakable, modifiable at runtime
```

Content hugging (`.defaultHigh` = 750) controls resistance to growing. Compression resistance (`.defaultLow` = 250) controls resistance to shrinking. Set these explicitly when multiple views compete for space.

### Constraint animation pattern

```swift
// 1. Flush pending layout
view.layoutIfNeeded()

// 2. Update constraints OUTSIDE animation block
heightConstraint.constant = 200

// 3. Animate layoutIfNeeded on the SUPERVIEW
UIView.animate(withDuration: 0.35) {
    self.view.layoutIfNeeded()
}
```

### Debugging ambiguous layouts

Set a symbolic breakpoint on `UIViewAlertForUnsatisfiableConstraints` in Xcode. Name constraints for readable console output:

```swift
topConstraint.identifier = "ProfileHeader-Top"
```

In LLDB: `expr -l objc++ -O -- [[UIWindow keyWindow] _autolayoutTrace]` — views marked `AMBIGUOUS` have insufficient constraints.

### Reducing constraint churn in reusable cells

Never remove and re-add constraints per reuse cycle. Create all constraint sets once in `init`, then toggle `.isActive` or modify `.constant` values during configuration.

---

## 3. Modern collection and table view APIs

### Diffable data sources and the identity trap

**The golden rule: use stable identifiers (UUIDs, database IDs) as your item type** — not full model objects.

```swift
// ✅ Use ID type as the item identifier
var dataSource: UICollectionViewDiffableDataSource<Section, Recipe.ID>!

// In the cell provider, fetch the current model from your backing store
dataSource = UICollectionViewDiffableDataSource(collectionView: collectionView) {
    collectionView, indexPath, recipeID -> UICollectionViewCell? in
    let recipe = self.store.recipe(with: recipeID)
    return collectionView.dequeueConfiguredReusableCell(
        using: self.cellRegistration, for: indexPath, item: recipe
    )
}
```

The most common diffable data source bug is implementing `Hashable` incorrectly. If you hash by ID but `==` compares all fields, changing a mutable field makes the old and new item unequal despite identical hashes. The data source sees this as delete + insert, not an update — causing visual glitches or crashes with `reloadItems`.

### reconfigureItems vs reloadItems (iOS 15+)

**`reconfigureItems`** updates cells in-place without calling `prepareForReuse` — significantly better performance and smoother animations. **`reloadItems`** dequeues entirely new cells. Prefer `reconfigureItems` for data changes; use `reloadItems` only when the cell type itself must change.

```swift
var snapshot = dataSource.snapshot()
snapshot.reconfigureItems([changedItemID]) // In-place update
dataSource.apply(snapshot, animatingDifferences: true)
```

For initial data load, use `applySnapshotUsingReloadData(_:)` to skip diff computation entirely.

### UICollectionView.CellRegistration (iOS 14+)

Replaces string-based `register`/`dequeue` with a type-safe, generics-based pattern:

```swift
let cellRegistration = UICollectionView.CellRegistration<MyCell, Recipe> { cell, indexPath, recipe in
    var content = UIListContentConfiguration.subtitleCell()
    content.text = recipe.title
    content.secondaryText = recipe.subtitle
    cell.contentConfiguration = content
}

// In cell provider:
return collectionView.dequeueConfiguredReusableCell(
    using: cellRegistration, for: indexPath, item: recipe
)
```

No `register()` call needed. No string identifiers. No force casting.

### Compositional layout essentials

`UICollectionViewCompositionalLayout` (iOS 13+) replaces `UICollectionViewFlowLayout` for any non-trivial layout. Compose sections independently with different scroll behaviors:

```swift
let layout = UICollectionViewCompositionalLayout { sectionIndex, environment in
    switch Section(rawValue: sectionIndex) {
    case .featured: return self.makeOrthogonalSection()
    case .grid:     return self.makeGridSection()
    case .list:     return self.makeListSection()
    case nil:       return nil
    }
}
```

For UITableView-style lists, `UICollectionLayoutListConfiguration` provides `.plain`, `.grouped`, `.insetGrouped`, `.sidebar`, and `.sidebarPlain` appearances — with built-in swipe actions, accessories, and self-sizing.

### Self-sizing cells

For UITableView: set `rowHeight = UITableView.automaticDimension` and `estimatedRowHeight = 60`. For compositional layout: use `.estimated(44)` in the layout size. In both cases, constraints must form an **unambiguous chain from top to bottom** of `contentView`.

---

## 4. UINavigationController patterns that avoid crashes

### The mid-transition crash

Pushing or popping while another animated transition is in progress causes `"Can't add self as subview"` — a crash. Guard against this by tracking animation state via `UINavigationControllerDelegate.navigationController(_:didShow:animated:)`.

### UINavigationBarAppearance (iOS 13+)

Since iOS 15, `scrollEdgeAppearance` defaults to transparent. This caused widespread visual bugs when apps upgraded. Always configure all four appearance slots:

```swift
let appearance = UINavigationBarAppearance()
appearance.configureWithOpaqueBackground()
appearance.backgroundColor = .systemBlue
appearance.titleTextAttributes = [.foregroundColor: UIColor.white]

navigationBar.standardAppearance = appearance
navigationBar.scrollEdgeAppearance = appearance      // ← Don't forget this
navigationBar.compactAppearance = appearance
navigationBar.compactScrollEdgeAppearance = appearance // iOS 15+
```

**Set appearance on `navigationItem`** (per-VC, takes precedence) or on the `navigationBar` (global). The #1 mistake is setting it on the wrong object.

### Large titles

Set `prefersLargeTitles = true` **once** on the navigation bar. Control per-VC behavior with `navigationItem.largeTitleDisplayMode` (`.always`, `.never`, `.automatic`). Never toggle `prefersLargeTitles` in `viewWillAppear` — it causes glitchy transitions.

### Back button

`backBarButtonItem` is set on the **pushing** VC, not the displayed VC. This is the #1 source of confusion. iOS 14+ added `backButtonTitle` and `backButtonDisplayMode` for cleaner APIs. iOS 16+ added `backAction` to intercept back navigation without replacing the button's appearance.

---

## 5. UIView animation: from basic to interactive

### UIView.animate best practices

Always check the `finished` parameter in completion handlers — `false` means the animation was interrupted. Since **iOS 8, UIView animations are additive by default**, meaning overlapping animations on the same property blend smoothly rather than replacing each other.

```swift
UIView.animate(withDuration: 0.3, delay: 0, options: [.curveEaseOut, .allowUserInteraction],
    animations: { self.cardView.transform = .identity },
    completion: { finished in
        guard finished else { return }
        self.cardView.removeFromSuperview()
    }
)
```

Note: `.allowUserInteraction` enables touches during animation, but **hit testing uses the model layer** (final position), not the presentation layer (current visual position).

### UIViewPropertyAnimator for interactive animations (iOS 10+)

`UIViewPropertyAnimator` supports pausing, reversing, scrubbing, and interrupting animations — essential for gesture-driven transitions:

```swift
let animator = UIViewPropertyAnimator(duration: 0.5, dampingRatio: 0.8) {
    self.detailView.transform = .identity
}

// Scrub with a gesture
@objc func handlePan(_ gesture: UIPanGestureRecognizer) {
    let progress = gesture.translation(in: view).y / view.bounds.height
    switch gesture.state {
    case .changed:
        animator.fractionComplete = progress
    case .ended:
        animator.isReversed = progress < 0.5
        animator.continueAnimation(withTimingParameters: nil, durationFactor: 0)
    default: break
    }
}
```

The animator state machine: **inactive → active → stopped**. Calling `stopAnimation(true)` returns to inactive. Calling `stopAnimation(false)` requires a subsequent `finishAnimation(at:)` call. Violating the state machine crashes.

### Spring animations

The classic API uses `usingSpringWithDamping` (0–1, where 1 = no bounce). **iOS 17+ introduced** `UIView.animate(springDuration:bounce:)` with a more intuitive `bounce` parameter (-1 to 1) that aligns with SwiftUI's `.spring(duration:bounce:)`.

### When to use CAAnimation

Use `CABasicAnimation` for layer properties not exposed by UIView: `cornerRadius`, `shadowOpacity`, `shadowRadius`, `borderWidth`, 3D transforms. Always set the **model value** before adding the animation — `CAAnimation` only affects the presentation layer:

```swift
let anim = CABasicAnimation(keyPath: "cornerRadius")
anim.fromValue = view.layer.cornerRadius
anim.toValue = 25.0
anim.duration = 0.3
view.layer.cornerRadius = 25.0  // ← Set model value first!
view.layer.add(anim, forKey: "cornerRadius")
```

---

## 6. Memory management: retain cycles hide in plain sight

### [weak self] vs [unowned self]

**Default to `[weak self]`** in all escaping closures. Use `[unowned self]` only when lifetime is provably guaranteed (e.g., `lazy var` closures).

```swift
// ✅ Network callback — always [weak self]
service.fetch { [weak self] result in
    guard let self else { return }
    self.updateUI(with: result)
}
```

### The four retain cycle traps

**Timer (target-selector)**: `Timer.scheduledTimer(timeInterval:target:selector:...)` **strongly retains its target**. The run loop retains the timer. If `self` stores the timer, the cycle is: `self → timer → (run loop) → self`. Fix: use the block-based API with `[weak self]`, and invalidate in `viewWillDisappear`:

```swift
timer = Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true) { [weak self] _ in
    self?.tick()
}
```

**NotificationCenter (closure API)**: The returned token retains the closure. If the closure captures `self` strongly, and `self` stores the token: `self → token → closure → self`. Fix: `[weak self]` in the closure, remove observer in `deinit`.

**CADisplayLink**: Like Timer, it strongly retains its target with no block-based API. Fix: use a weak proxy object that forwards the selector and invalidates if the target is nil.

**Nested closures**: If an outer closure captures `[weak self]`, uses `guard let self`, and then passes that strong `self` into a stored inner closure — the inner closure holds `self` alive. Re-apply `[weak self]` in nested escaping closures.

### Delegates must be weak

Always declare delegates as `weak var delegate: SomeDelegate?` on a protocol constrained to `AnyObject`.

---

## 7. UIKit and SwiftUI interoperability

### UIHostingController embedding

The correct pattern requires full child view controller containment: `addChild` → `addSubview` → `didMove(toParent:)`. **The hosting controller must be retained** as a stored property — creating a local variable and only adding its view causes deallocation that breaks the SwiftUI-UIKit bridge.

```swift
let hostingController = UIHostingController(rootView: MySwiftUIView())
addChild(hostingController)
hostingController.view.translatesAutoresizingMaskIntoConstraints = false
view.addSubview(hostingController.view)
NSLayoutConstraint.activate([/* pin edges */])
hostingController.didMove(toParent: self)
```

**`sizingOptions` (iOS 16+)** controls how the hosting controller tracks its SwiftUI content's ideal size. Set `.intrinsicContentSize` when embedding in Auto Layout containers (cells, stack views). Set `.preferredContentSize` for popovers. Without this, SwiftUI content won't size properly.

### UIViewRepresentable lifecycle

The order is: `makeCoordinator()` → `makeUIView(context:)` → `updateUIView(_:context:)` (called on every SwiftUI state change). SwiftUI **reuses** the underlying UIView — properties set only in `makeUIView` won't update. Always set mutable state in `updateUIView`.

```swift
func updateUIView(_ textView: UITextView, context: Context) {
    if textView.text != text { textView.text = text } // Guard against loops
}
```

### Sizing pitfalls

UIViewRepresentable views expand to fill available space by default. Solutions: override `intrinsicContentSize` on the UIView, use `.fixedSize()` in SwiftUI, set content hugging priorities, or apply an explicit `.frame()`.

---

## 8. Modern cell configuration replaces textLabel forever

### UIContentConfiguration (iOS 14+)

`UIListContentConfiguration` replaces the deprecated `textLabel`, `detailTextLabel`, and `imageView` properties. It works identically on both `UITableViewCell` and `UICollectionViewCell`.

```swift
// ❌ Deprecated
cell.textLabel?.text = item.title
cell.detailTextLabel?.text = item.subtitle
cell.imageView?.image = item.icon

// ✅ Modern (iOS 14+)
var content = cell.defaultContentConfiguration()
content.text = item.title
content.secondaryText = item.subtitle
content.image = item.icon
content.textProperties.font = .preferredFont(forTextStyle: .headline)
content.imageProperties.cornerRadius = 8
cell.contentConfiguration = content
```

Factory methods: `.cell()`, `.subtitleCell()`, `.valueCellConfiguration()`, `.sidebarCell()`, `.header()`, `.footer()`, and more.

### UIBackgroundConfiguration

```swift
var bg = UIBackgroundConfiguration.listGroupedCell()
bg.backgroundColor = .systemBackground
bg.cornerRadius = 10
bg.strokeColor = .separator
cell.backgroundConfiguration = bg
```

### configurationUpdateHandler (iOS 15+)

Replaces subclassing for state-dependent styling. Called whenever cell state changes (selection, highlight, etc.):

```swift
cell.configurationUpdateHandler = { cell, state in
    var content = UIListContentConfiguration.cell().updated(for: state)
    content.text = "Item"
    if state.isSelected {
        content.textProperties.font = .boldSystemFont(ofSize: 16)
    }
    cell.contentConfiguration = content
}
```

### Custom content configurations

For layouts beyond what `UIListContentConfiguration` offers, conform to `UIContentConfiguration` with `makeContentView()` and `updated(for:)`, then implement `UIContentView` on your custom view.

---

## 9. Swift concurrency and UIKit main thread safety

### UIViewController is @MainActor

`UIViewController` is annotated `@MainActor` in Apple's SDK headers. All subclass methods are implicitly `@MainActor`. You do **not** need to add `@MainActor` to your view controller classes.

### Task lifecycle management

Always store task references and cancel them when the view disappears. **`deinit` alone is unreliable** — if a `Task` captures `self` strongly (implicit in `@MainActor` context), it keeps the VC alive, so `deinit` never fires.

```swift
class DataViewController: UIViewController {
    private var loadTask: Task<Void, Never>?

    override func viewIsAppearing(_ animated: Bool) {
        super.viewIsAppearing(animated)
        loadTask = Task {
            let items = try? await api.fetchItems()
            guard !Task.isCancelled else { return }
            updateUI(with: items)
        }
    }

    override func viewDidDisappear(_ animated: Bool) {
        super.viewDidDisappear(animated)
        loadTask?.cancel()
    }
}
```

### Task.detached does not inherit actor isolation

```swift
// ❌ Data race — not on MainActor
Task.detached { self.label.text = data.title }

// ✅ Hop back explicitly
Task.detached {
    let data = try await self.fetchData()
    await MainActor.run { self.label.text = data.title }
}
```

### Swift 6 strict concurrency

The main migration challenge is the **cascading `@MainActor` effect** — protocols your VC conforms to and its dependencies may need `@MainActor` annotation. Use `nonisolated` to opt out for methods that don't touch UI. Types crossing actor boundaries must be `Sendable`.

---

## 10. Image loading: decoded size is the real cost

### The memory math

A **12-megapixel** photo compressed to 3 MB on disk consumes **~48 MB of RAM** when decoded: `4032 × 3024 × 4 bytes = 48.8 MB`. File size is irrelevant; decoded bitmap dimensions determine memory.

### Downsampling with ImageIO

Decode only at the display size — never load the full bitmap and resize:

```swift
func downsample(imageAt url: URL, to pointSize: CGSize,
                scale: CGFloat = UIScreen.main.scale) -> UIImage? {
    let sourceOptions = [kCGImageSourceShouldCache: false] as CFDictionary
    guard let source = CGImageSourceCreateWithURL(url as CFURL, sourceOptions) else { return nil }
    let maxDimension = max(pointSize.width, pointSize.height) * scale
    let options: CFDictionary = [
        kCGImageSourceCreateThumbnailFromImageAlways: true,
        kCGImageSourceShouldCacheImmediately: true,
        kCGImageSourceCreateThumbnailWithTransform: true,
        kCGImageSourceThumbnailMaxPixelSize: maxDimension
    ] as CFDictionary
    guard let cgImage = CGImageSourceCreateThumbnailAtIndex(source, 0, options) else { return nil }
    return UIImage(cgImage: cgImage)
}
```

This reduces a 3648×5472 image displayed in a 200×300pt view on @3x from **~80 MB to ~2 MB**.

### iOS 15+ high-level APIs

`UIImage.byPreparingForDisplay()` and `byPreparingThumbnail(of:)` provide async decoding without manual queue management:

```swift
let thumbnail = await largeImage.byPreparingThumbnail(of: CGSize(width: 120, height: 120))
imageView.image = thumbnail
```

### Solving the cell reuse race condition

The classic bug: cell A starts loading image X, gets reused for row B, image X completes and displays in the wrong cell. The fix requires three elements — **cancel on reuse, clear immediately, verify identity**:

```swift
class ImageCell: UICollectionViewCell {
    private var loadTask: Task<Void, Never>?
    private var currentURL: URL?

    func configure(with url: URL) {
        currentURL = url
        imageView.image = nil
        loadTask = Task { [weak self] in
            let image = try? await ImageLoader.shared.image(for: url)
            guard !Task.isCancelled, self?.currentURL == url else { return }
            self?.imageView.image = image
        }
    }

    override func prepareForReuse() {
        super.prepareForReuse()
        loadTask?.cancel()
        imageView.image = nil
        currentURL = nil
    }
}
```

Use `NSCache` with `totalCostLimit` based on decoded bitmap size (not file size) for the in-memory cache.

---

## 11. Keyboard handling: use the layout guide

### Keyboard layout guide (iOS 15+) — the preferred approach

Every `UIView` has a `keyboardLayoutGuide` that tracks the keyboard automatically, including animation:

```swift
NSLayoutConstraint.activate([
    inputBar.bottomAnchor.constraint(equalTo: view.keyboardLayoutGuide.topAnchor),
    inputBar.leadingAnchor.constraint(equalTo: view.leadingAnchor),
    inputBar.trailingAnchor.constraint(equalTo: view.trailingAnchor)
])
```

When the keyboard is hidden, the guide tracks the bottom safe area. When visible, it tracks the keyboard's top edge. No notification handling, no coordinate conversion, no manual animation.

### Notification-based handling (pre-iOS 15)

Two critical details most implementations get wrong: **always convert the keyboard frame** from screen coordinates to view coordinates using `view.convert(_:from: view.window)`, and **subtract `view.safeAreaInsets.bottom`** to avoid double-insetting on devices with a home indicator.

```swift
@objc func adjustForKeyboard(notification: Notification) {
    guard let frameEnd = (notification.userInfo?[UIResponder.keyboardFrameEndUserInfoKey] as? NSValue)?.cgRectValue,
          let duration = notification.userInfo?[UIResponder.keyboardAnimationDurationUserInfoKey] as? Double,
          let curve = notification.userInfo?[UIResponder.keyboardAnimationCurveUserInfoKey] as? UInt
    else { return }
    
    let frameInView = view.convert(frameEnd, from: view.window)
    let height = notification.name == UIResponder.keyboardWillHideNotification
        ? 0 : frameInView.height - view.safeAreaInsets.bottom
    
    UIView.animate(withDuration: duration, delay: 0,
                   options: UIView.AnimationOptions(rawValue: curve << 16)) {
        self.scrollView.contentInset.bottom = height
    }
}
```

---

## 12. Trait collections: iOS 17 changed everything

### registerForTraitChanges replaces traitCollectionDidChange

`traitCollectionDidChange(_:)` is **deprecated in iOS 17**. The replacement, `registerForTraitChanges`, registers for *specific* traits — better performance since the old method fired for every trait change.

```swift
// ✅ iOS 17+
override func viewDidLoad() {
    super.viewDidLoad()
    registerForTraitChanges([UITraitHorizontalSizeClass.self]) { (self: Self, _) in
        self.configureLayout()
    }
    registerForTraitChanges([UITraitUserInterfaceStyle.self]) { (self: Self, _) in
        self.updateColors()
    }
}

override func viewIsAppearing(_ animated: Bool) {
    super.viewIsAppearing(animated)
    configureLayout() // Handler is NOT called on registration
    updateColors()
}
```

The first closure parameter uses `self: Self` — no `[weak self]` needed; the framework handles the lifecycle. The handler is **not** called on registration, so set initial state in `viewIsAppearing`.

### Custom traits (iOS 17+)

Define custom traits with `UITraitDefinition`, add convenience extensions on `UITraitCollection` and `UIMutableTraits`, set via `traitOverrides` on any view/VC/window, and observe with `registerForTraitChanges`. Custom traits propagate down the hierarchy and can bridge to SwiftUI environment keys via `UITraitBridgedEnvironmentKey`.

### Size classes for adaptive layout

| Configuration | Horizontal | Vertical |
|---|---|---|
| iPhone portrait | Compact | Regular |
| iPhone landscape | Compact (Regular on Plus/Max) | Compact |
| iPad full screen | Regular | Regular |
| iPad narrow split | Compact | Regular |

---

## 13. Accessibility: Dynamic Type and VoiceOver

### Dynamic Type with custom fonts

```swift
// System font — simplest
label.font = UIFont.preferredFont(forTextStyle: .body)
label.adjustsFontForContentSizeCategory = true

// Custom font — scale with UIFontMetrics
let custom = UIFont(name: "MyFont-Regular", size: 17)!
label.font = UIFontMetrics(forTextStyle: .body).scaledFont(for: custom)
label.adjustsFontForContentSizeCategory = true
```

**`adjustsFontForContentSizeCategory = true`** is essential — without it, the font is correct at creation time but doesn't respond to live text-size changes.

### VoiceOver essentials

Standard UIKit controls (`UILabel`, `UIButton`) are accessible by default. Custom views need explicit configuration:

```swift
customView.isAccessibilityElement = true
customView.accessibilityLabel = "Play"           // What it is
customView.accessibilityHint = "Plays the track" // What it does
customView.accessibilityTraits = [.button, .startsMediaSession]
```

For container views with individually accessible sub-elements, set `isAccessibilityElement = false` on the container and populate `accessibilityElements` with `UIAccessibilityElement` instances.

Post notifications after dynamic UI changes: `.screenChanged` for major transitions (new VC), `.layoutChanged` for element additions/removals, `.announcement` for status messages.

---

## 14. Dark mode: the CGColor trap

### Dynamic colors resolve automatically — mostly

`UIColor.systemBackground`, `.label`, `.secondaryLabel`, and other semantic colors adapt automatically. Custom dynamic colors work via `UIColor { traitCollection in ... }`.

**The #1 dark mode bug is CGColor.** `CGColor` is a Core Graphics primitive with no dynamic behavior. Setting `layer.borderColor = UIColor.label.cgColor` in `viewDidLoad` freezes the color in whatever mode was active at that moment.

```swift
// ❌ Frozen in initial appearance mode
view.layer.borderColor = UIColor.label.cgColor

// ✅ Re-resolve when traits change
private func updateLayerColors() {
    view.layer.borderColor = UIColor.label.cgColor
    view.layer.shadowColor = UIColor.separator.cgColor
}

// iOS 17+:
registerForTraitChanges([UITraitUserInterfaceStyle.self]) { (self: Self, _) in
    self.updateLayerColors()
}
```

**Why `UIView.backgroundColor` works but `CALayer.borderColor` doesn't:** UIView stores the dynamic UIColor and re-resolves it automatically. CALayer stores the resolved CGColor — a static value.

### Asset catalog color and image variants

Add "Any, Dark" appearances in Xcode's Asset Catalog for both colors and images. Reference with `UIColor(named:)` and `UIImage(named:)` — they adapt automatically.

### overrideUserInterfaceStyle

Force a specific mode per window, VC, or view. Set to `.unspecified` to follow the system. Use sparingly.

---

## 15. Anti-patterns AI code generators consistently produce

Based on extensive analysis, AI-generated UIKit code makes these mistakes most frequently:

**Targeting obsolete iOS versions.** Generated code uses `traitCollectionDidChange` instead of `registerForTraitChanges`, misses `viewIsAppearing`, uses manual cell configuration instead of `UIContentConfiguration`, and ignores `keyboardLayoutGuide`.

**Geometry work in `viewDidLoad`.** Layer frames, size-class checks, and scroll-to-item calls placed in `viewDidLoad` where geometry is not yet accurate. These belong in `viewIsAppearing`.

**Missing task cancellation.** Tasks launched from lifecycle methods without stored references or cancellation in `viewDidDisappear`. This leaks view controllers and causes stale UI updates.

**Forgotten `translatesAutoresizingMaskIntoConstraints = false`.** The single most common Auto Layout bug. Results in `NSAutoresizingMaskLayoutConstraints` conflicts in the console.

**Individual constraint activation.** Using `.isActive = true` per constraint instead of `NSLayoutConstraint.activate([])`.

**Strong self in escaping closures.** Timer callbacks, NotificationCenter closures, and network completions without `[weak self]`.

**No `prepareForReuse` implementation.** Cells that load images asynchronously without cancelling in-flight requests or clearing stale images on reuse.

**Direct navigation bar property setting.** Using `barTintColor`/`isTranslucent` instead of `UINavigationBarAppearance`, which breaks on iOS 15+ due to the transparent `scrollEdgeAppearance` default.

**Expensive work in `viewWillAppear`.** Network fetches or heavy computation that fires on every tab switch, every navigation pop, every modal dismissal.

**Using `Task.detached` for UI work.** `Task.detached` does not inherit `@MainActor` isolation. UI updates inside it are data races.

---

## 16. Deprecated APIs and their modern replacements

| Deprecated API | Deprecated In | Modern Replacement |
|---|---|---|
| `traitCollectionDidChange(_:)` | iOS 17 | `registerForTraitChanges(_:handler:)` |
| `UITableViewCell.textLabel` | iOS 14 | `UIListContentConfiguration.text` via `cell.contentConfiguration` |
| `UITableViewCell.detailTextLabel` | iOS 14 | `UIListContentConfiguration.secondaryText` |
| `UITableViewCell.imageView` | iOS 14 | `UIListContentConfiguration.image` |
| `UITableViewCell(style:reuseIdentifier:)` styles | iOS 14 | `UIListContentConfiguration` factory methods |
| `UIApplication.shared.setStatusBarStyle` | iOS 9 | Override `preferredStatusBarStyle` on UIViewController |
| `UIApplication.shared.setStatusBarHidden` | iOS 9 | Override `prefersStatusBarHidden` on UIViewController |
| `reloadItems(_:)` on snapshot | — | `reconfigureItems(_:)` (iOS 15+) for in-place updates |
| Manual `performBatchUpdates` | — | `NSDiffableDataSourceSnapshot.apply(_:)` |
| `register(_:forCellWithReuseIdentifier:)` + string dequeue | — | `UICollectionView.CellRegistration` (iOS 14+) |
| Navigation bar `barTintColor`/`isTranslucent` | iOS 13 | `UINavigationBarAppearance` with all four appearance slots |
| `UICollectionViewFlowLayout` (complex layouts) | Not deprecated | `UICollectionViewCompositionalLayout` (iOS 13+) |

**iOS 17+ migration priority:** Replace `traitCollectionDidChange` with `registerForTraitChanges`. Adopt `viewIsAppearing` (back-deploys to iOS 13). Use `traitOverrides` instead of `overrideTraitCollection`. These three changes address the most impactful API evolution in recent UIKit history.

---

## Conclusion

The most consequential shifts in modern UIKit are not new features but **corrections to long-standing friction points**: `viewIsAppearing` finally provides a reliable lifecycle callback with accurate geometry, `registerForTraitChanges` eliminates the wasteful all-traits notification pattern, and content configurations replace the rigid cell style system that predated Auto Layout. The common thread across all 16 topics is that **the modern APIs exist specifically because the old patterns caused persistent, hard-to-diagnose bugs** — transparent navigation bars on iOS 15, stale cells from incorrect diffable data source hashing, image loading race conditions, CGColor dark mode freezing, and retain cycles in timers and notification observers.

Three practices yield the highest impact for code quality: **always use `NSLayoutConstraint.activate` with `translatesAutoresizingMaskIntoConstraints = false`**, **always cancel Tasks in `viewDidDisappear` and use `[weak self]` in escaping closures**, and **always use `viewIsAppearing` instead of `viewDidLoad` for geometry-dependent work**. These three patterns alone prevent the majority of UIKit bugs seen in production apps and AI-generated code alike.