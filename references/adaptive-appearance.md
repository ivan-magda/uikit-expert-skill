# UIKit Adaptive UI & Accessibility Reference (iOS 17+)

This reference covers the five pillars of adaptive, accessible UIKit development: the modern trait system, size classes, Dynamic Type, Dark Mode, and accessibility. Every section includes ✅ correct and ❌ incorrect Swift patterns so you can audit your codebase at a glance. All APIs target **iOS 17+** unless noted otherwise.

---

## 1. Trait collections and `registerForTraitChanges`

iOS 17 replaced the monolithic `traitCollectionDidChange(_:)` callback with a surgical registration API. Instead of being called for *every* trait mutation, you now declare exactly which traits you care about — and the system only fires your handler when those specific values change.

### The closure-based registration pattern

The primary API lives on `UITraitChangeObservable` (conformed to by `UIViewController`, `UIView`, `UIWindow`, `UIWindowScene`, and `UIPresentationController`):

```swift
func registerForTraitChanges<Self>(
    _ traits: [UITrait],
    handler: @escaping (Self, UITraitCollection) -> Void
) -> any UITraitChangeRegistration
```

✅ **Correct — register for specific traits with `self: Self`:**

```swift
override func viewDidLoad() {
    super.viewDidLoad()

    registerForTraitChanges(
        [UITraitHorizontalSizeClass.self, UITraitUserInterfaceStyle.self]
    ) { (self: Self, previousTraitCollection: UITraitCollection) in
        self.updateLayout()   // `self` is the parameter, not a capture
    }
}
```

A target-action variant is also available:

```swift
registerForTraitChanges(
    [UITraitHorizontalSizeClass.self],
    action: #selector(updateLayout)
)
```

❌ **Incorrect — using `[weak self]` and the deprecated callback:**

```swift
// ❌ Don't use [weak self] — the first parameter IS the object, not a capture
registerForTraitChanges([UITraitUserInterfaceStyle.self]) { [weak self] (vc: Self, _) in
    self?.updateLayout()   // Unnecessary indirection
}

// ❌ Deprecated in iOS 17 — fires for ALL trait changes, wasting work
override func traitCollectionDidChange(_ prev: UITraitCollection?) {
    super.traitCollectionDidChange(prev)
    if prev?.horizontalSizeClass != traitCollection.horizontalSizeClass {
        updateLayout()
    }
}
```

### Why `[weak self]` is not needed

The closure's first parameter is the observed object itself, **passed by the system at invocation time** — it is not a captured reference. Apple's Tyler Fox at WWDC 2023: *"The object whose traits have changed is passed as the first parameter to the closure. Use this parameter so you don't have to capture a weak reference."* Writing `self: Self` shadows the outer `self` with the parameter, so every use of `self` inside the closure refers to the parameter. The registration is automatically cleaned up when the object deallocates, so no retain cycle forms.

### The handler is NOT called on registration

This is the critical subtlety most developers miss. **The handler fires only on subsequent changes** — not at registration time. If you rely solely on the handler to configure initial state, your UI will be wrong until the first trait change occurs.

✅ **Correct — set initial state in `viewIsAppearing(_:)`:**

```swift
final class ProfileViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        registerForTraitChanges([UITraitHorizontalSizeClass.self]) {
            (self: Self, _: UITraitCollection) in
            self.updateLayout()
        }
    }

    override func viewIsAppearing(_ animated: Bool) {
        super.viewIsAppearing(animated)
        updateLayout()          // ← Initial state set here
    }

    private func updateLayout() {
        let isCompact = traitCollection.horizontalSizeClass == .compact
        stackView.axis = isCompact ? .vertical : .horizontal
    }
}
```

`viewIsAppearing(_:)` is ideal because it fires after traits are finalized, runs once per appearance cycle, and **back-deploys to iOS 13** despite being part of the iOS 17 SDK.

❌ **Incorrect — relying on the handler for initial state:**

```swift
override func viewDidLoad() {
    super.viewDidLoad()
    // ❌ No initial call — UI is misconfigured until a trait actually changes
    registerForTraitChanges([UITraitHorizontalSizeClass.self]) {
        (self: Self, _) in
        self.updateLayout()
    }
}
```

For `UIView` subclasses (which lack `viewIsAppearing`), use `layoutSubviews()`:

```swift
override func layoutSubviews() {
    super.layoutSubviews()
    layer.borderColor = UIColor.separator.cgColor   // Always up to date
}
```

### Convenience trait sets

UIKit provides pre-built arrays so you don't have to enumerate every color-related trait manually:

```swift
registerForTraitChanges(
    UITraitCollection.systemTraitsAffectingColorAppearance,
    action: #selector(resolveLayerColors)
)

registerForTraitChanges(
    UITraitCollection.systemTraitsAffectingImageLookup,
    action: #selector(reloadImages)
)
```

### Custom traits with `UITraitDefinition`

Define a custom trait by conforming to `UITraitDefinition`. The only requirement is a `defaultValue`:

```swift
struct ContainedInSettingsTrait: UITraitDefinition {
    static let defaultValue = false
    static let affectsColorAppearance = false   // optional
}
```

Add convenience accessors on `UITraitCollection` (read-only) and `UIMutableTraits` (read-write):

```swift
extension UITraitCollection {
    var isContainedInSettings: Bool { self[ContainedInSettingsTrait.self] }
}

extension UIMutableTraits {
    var isContainedInSettings: Bool {
        get { self[ContainedInSettingsTrait.self] }
        set { self[ContainedInSettingsTrait.self] = newValue }
    }
}
```

### Propagating custom traits with `traitOverrides`

`traitOverrides` (available on `UIWindowScene`, `UIWindow`, `UIViewController`, `UIPresentationController`, and `UIView`) lets you inject values that cascade down the hierarchy:

```swift
// Set at the window-scene level — every VC and view inherits it
windowScene.traitOverrides.isContainedInSettings = true

// Override at a specific view — only this subtree is affected
detailView.traitOverrides.isContainedInSettings = false

// Check and remove overrides
if view.traitOverrides.contains(ContainedInSettingsTrait.self) {
    view.traitOverrides.remove(ContainedInSettingsTrait.self)
}
```

Overrides flow **parent → child**. A child's own `traitOverrides` are applied on top of inherited values. In iOS 17's unified hierarchy, the chain is: Window Scene → Window → Root VC → Root View → Subviews → Child VC → Child View.

### Bridging to SwiftUI with `UITraitBridgedEnvironmentKey`

A bridged trait flows **bidirectionally** between UIKit's trait collection and SwiftUI's `@Environment`. Three pieces are needed:

```swift
// 1️⃣ UIKit trait (already defined above)
// 2️⃣ SwiftUI EnvironmentKey
struct ContainedInSettingsKey: EnvironmentKey {
    static let defaultValue = false
}

extension EnvironmentValues {
    var isContainedInSettings: Bool {
        get { self[ContainedInSettingsKey.self] }
        set { self[ContainedInSettingsKey.self] = newValue }
    }
}

// 3️⃣ Bridge conformance
extension ContainedInSettingsKey: UITraitBridgedEnvironmentKey {
    static func read(from traitCollection: UITraitCollection) -> Bool {
        traitCollection.isContainedInSettings
    }
    static func write(to mutableTraits: inout UIMutableTraits, value: Bool) {
        mutableTraits.isContainedInSettings = value
    }
}
```

Now a `traitOverrides` change in UIKit automatically updates any SwiftUI `@Environment(\.isContainedInSettings)` in hosted views, and vice-versa.

---

## 2. Size class quick-reference

Size classes abstract away specific point dimensions into two buckets — **Compact** (C) and **Regular** (R) — for width and height independently. The table below covers every mainstream configuration.

| Configuration | Width | Height |
|---|---|---|
| **iPhone — portrait** (all models) | C | R |
| **iPhone — landscape** (standard & Pro) | C | C |
| **iPhone — landscape** (Plus / Max / Air, ≥ 414 pt width) | R | C |
| **iPad — full screen portrait** (all models) | R | R |
| **iPad — full screen landscape** (all models) | R | R |
| **iPad — Slide Over** (any orientation) | C | R |
| **iPad — Split View portrait** (both apps) | C | R |
| **iPad — Split ⅓ landscape** (narrow app) | C | R |
| **iPad — Split ⅔ landscape** (wide app) | R | R |
| **iPad — Split ½ landscape** (non-12.9″ / non-13″) | C | R |
| **iPad — Split ½ landscape** (12.9″ / 13″ only) | **R** | R |

Key observations worth highlighting:

The **½–½ landscape split on 12.9″ and 13″ iPads** is unique: both apps receive **Regular width**. On every other iPad, a ½–½ split gives both apps Compact width. The height class is **always Regular** on iPad regardless of orientation or multitasking mode. The informal breakpoint for Regular vs. Compact horizontal on iPhone landscape is a portrait width of **≥ 414 points** — every Plus, Max, and Air model meets this threshold. Apple does not publish an official single-number breakpoint; size classes are assigned per device model.

With iPadOS 16+ **Stage Manager** and resizable windows, apps can encounter arbitrary window sizes. The system still reports Compact or Regular based on the window's width using the same approximate thresholds (~680 pt for iPad). Always use size classes — never hard-code device idiom checks — for layout decisions.

---

## 3. Dynamic Type

Dynamic Type lets users choose their preferred text size system-wide. Supporting it correctly requires three things: the right font API, proper scaling for custom fonts, and opting in to **live updates**.

### System fonts with `preferredFont`

✅ **Correct — system font with live updates enabled:**

```swift
let label = UILabel()
label.font = UIFont.preferredFont(forTextStyle: .body)
label.adjustsFontForContentSizeCategory = true   // Required for live updates
label.numberOfLines = 0                           // Allow text to reflow
```

❌ **Incorrect — fixed system font that ignores the user's preference:**

```swift
let label = UILabel()
label.font = UIFont.systemFont(ofSize: 17)        // ❌ Never scales
```

The available text styles at the default "Large" size: `.largeTitle` (34 pt), `.title1` (28), `.title2` (22), `.title3` (20), `.headline` (17, semibold), `.body` (17), `.callout` (16), `.subheadline` (15), `.footnote` (13), `.caption1` (12), `.caption2` (11).

### Scaling custom fonts with `UIFontMetrics`

`UIFontMetrics` applies the same scale factor the system uses, but to your own typeface:

✅ **Correct — custom font that scales:**

```swift
guard let merriweather = UIFont(name: "Merriweather-Regular", size: 17) else {
    fatalError("Missing font")
}
let label = UILabel()
label.font = UIFontMetrics(forTextStyle: .body).scaledFont(for: merriweather)
label.adjustsFontForContentSizeCategory = true
```

You can cap the maximum size to prevent extreme growth:

```swift
label.font = UIFontMetrics(forTextStyle: .body)
    .scaledFont(for: merriweather, maximumPointSize: 40)
```

❌ **Incorrect — custom font that never scales:**

```swift
label.font = UIFont(name: "Merriweather-Regular", size: 17)   // ❌ Static size forever
```

### Why `adjustsFontForContentSizeCategory` is critical

This property (available on `UILabel`, `UITextField`, `UITextView`) defaults to **`false`**. Without it, `preferredFont(forTextStyle:)` returns the correct size *at creation time*, but the label never updates when the user changes their text size setting via Control Center or Settings.

❌ **Incorrect — font is right initially but frozen:**

```swift
label.font = UIFont.preferredFont(forTextStyle: .headline)
// adjustsFontForContentSizeCategory defaults to false
// → If the user changes text size mid-session, this label stays the same size
```

✅ **Correct — two-line pattern you should always use:**

```swift
label.font = UIFont.preferredFont(forTextStyle: .headline)
label.adjustsFontForContentSizeCategory = true
```

For the rare case where you cannot use `adjustsFontForContentSizeCategory`, you can observe the notification manually or register for the trait in iOS 17+:

```swift
// iOS 17+ alternative
registerForTraitChanges([UITraitPreferredContentSizeCategory.self]) {
    (self: Self, _) in
    self.bodyLabel.font = UIFontMetrics(forTextStyle: .body)
        .scaledFont(for: self.customFont)
}
```

---

## 4. Dark Mode

UIKit's Dark Mode support rests on **semantic colors** that resolve dynamically, **asset catalog variants**, and careful handling of the few places where the dynamic system breaks down (most notably `CGColor`).

### Semantic system colors

UIKit ships dozens of adaptive colors organized in a clear hierarchy. The most important groupings:

| Category | Colors | Purpose |
|----------|--------|---------|
| **Labels** | `.label`, `.secondaryLabel`, `.tertiaryLabel`, `.quaternaryLabel` | Text hierarchy from primary to lowest emphasis |
| **Backgrounds** | `.systemBackground`, `.secondarySystemBackground`, `.tertiarySystemBackground` | Flat (non-grouped) screens |
| **Grouped backgrounds** | `.systemGroupedBackground`, `.secondarySystemGroupedBackground`, `.tertiarySystemGroupedBackground` | Grouped table view / form screens |
| **Fills** | `.systemFill` through `.quaternarySystemFill` | Shape fills at varying emphasis |
| **Separators** | `.separator` (translucent), `.opaqueSeparator` | Divider lines |
| **Other** | `.link`, `.placeholderText` | Links and placeholder text |
| **Grays** | `.systemGray` through `.systemGray6` | Six adaptive gray levels |

Dark Mode also distinguishes **base** (edge-to-edge) from **elevated** (sheets, popovers) levels. In dark mode, system background colors shift lighter at the elevated level automatically.

✅ **Correct — semantic colors adapt automatically:**

```swift
view.backgroundColor = .systemBackground
titleLabel.textColor = .label
subtitleLabel.textColor = .secondaryLabel
divider.backgroundColor = .separator
```

❌ **Incorrect — hard-coded values that break in dark mode:**

```swift
view.backgroundColor = .white          // ❌ Blinding in dark mode
titleLabel.textColor = .black          // ❌ Invisible on dark background
divider.backgroundColor = UIColor(white: 0.8, alpha: 1)   // ❌ Doesn't adapt
```

### The CGColor trap

`CGColor` is a Core Graphics struct with **fixed** color component values. When you write `UIColor.label.cgColor`, the dynamic `UIColor` is resolved *once* to a frozen `CGColor`. Toggling appearance will leave the layer stuck in the wrong mode.

❌ **Incorrect — frozen CGColor set once:**

```swift
override func viewDidLoad() {
    super.viewDidLoad()
    cardView.layer.borderColor = UIColor.separator.cgColor   // ❌ Resolved once, stuck
    cardView.layer.borderWidth = 1
}
```

✅ **Correct — re-resolve on trait change (iOS 17+):**

```swift
override func viewDidLoad() {
    super.viewDidLoad()
    cardView.layer.borderWidth = 1
    applyLayerColors()

    registerForTraitChanges([UITraitUserInterfaceStyle.self]) {
        (self: Self, _) in
        self.applyLayerColors()
    }
}

override func viewIsAppearing(_ animated: Bool) {
    super.viewIsAppearing(animated)
    applyLayerColors()          // Initial state
}

private func applyLayerColors() {
    cardView.layer.borderColor = UIColor.separator.cgColor
    cardView.layer.shadowColor = UIColor.label.cgColor
}
```

For views, the `layoutSubviews()` approach is equally valid — it fires whenever traits change because the system invalidates layout:

```swift
override func layoutSubviews() {
    super.layoutSubviews()
    layer.borderColor = UIColor.separator.cgColor
}
```

### Asset catalog color and image variants

Xcode's asset catalog supports **Any Appearance + Dark** and an optional **High Contrast** axis, yielding up to four slots: Any, Dark, High Contrast Light, High Contrast Dark. Colors and images loaded from asset catalogs are automatically dynamic — they resolve to the correct variant for the current trait environment with no additional code.

```swift
// Loads a dynamic color/image that adapts without manual trait observation
let cardColor = UIColor(named: "CardBackground")      // adapts automatically
let icon = UIImage(named: "SettingsIcon")              // picks correct variant
```

### Dynamic provider initializer

For colors defined in code rather than asset catalogs, `UIColor` offers a closure-based initializer:

```swift
let adaptiveCardBackground = UIColor { traitCollection in
    switch (traitCollection.userInterfaceStyle, traitCollection.accessibilityContrast) {
    case (.dark, .high):  return UIColor(white: 0.20, alpha: 1)
    case (.dark, _):      return UIColor(white: 0.12, alpha: 1)
    case (_, .high):      return .white
    default:              return UIColor(white: 0.96, alpha: 1)
    }
}
```

A handy factory extension keeps call sites tidy:

```swift
extension UIColor {
    static func dynamic(light: UIColor, dark: UIColor) -> UIColor {
        UIColor { $0.userInterfaceStyle == .dark ? dark : light }
    }
}

// Usage
static let cardBorder = UIColor.dynamic(
    light: UIColor(white: 0.85, alpha: 1),
    dark:  UIColor(white: 0.25, alpha: 1)
)
```

Dynamic provider colors resolve on every draw pass, so they work correctly with `backgroundColor`, `tintColor`, and attributed strings — but they still produce a frozen `CGColor` if you access `.cgColor`.

### `UIAppearance` does not live-update

Appearance proxy invocations are replayed **only when a view is first added to a window** (just before `didMoveToWindow()`). Switching between light and dark mode at runtime **will not** re-apply appearance proxy settings to views already in the hierarchy.

```swift
// These run once per view insertion — NOT on appearance changes
UINavigationBar.appearance().barTintColor = .systemBackground
UILabel.appearance(whenContainedInInstancesOf: [UITableViewCell.self]).textColor = .label
```

If you use dynamic `UIColor` values (`.label`, asset catalog colors), the colors themselves adapt, but the appearance proxy will not re-call the setter. In practice, rely on semantic colors and dynamic providers rather than appearance proxies for Dark Mode.

### Forcing an interface style with `overrideUserInterfaceStyle`

Available on `UIView`, `UIViewController`, and `UIWindow`, this forces a subtree into a specific style:

```swift
// Force this VC's subtree to dark
overrideUserInterfaceStyle = .dark

// Force entire window
window?.overrideUserInterfaceStyle = .light

// Reset to follow system
overrideUserInterfaceStyle = .unspecified
```

The override cascades downward: setting it on a window affects everything inside; setting it on a view affects that view and all subviews. To opt the entire app out, add `UIUserInterfaceStyle = Light` (or `Dark`) to Info.plist.

---

## 5. Accessibility

### VoiceOver fundamentals

**`isAccessibilityElement`** is the gatekeeper. Standard UIKit controls default to `true`, but **custom `UIView` subclasses default to `false`** — the single most common VoiceOver bug is forgetting to flip this flag.

✅ **Correct — custom view fully configured for VoiceOver:**

```swift
class StatusBadge: UIView {
    var status: Status = .active {
        didSet { accessibilityLabel = status.localizedDescription }
    }

    override init(frame: CGRect) {
        super.init(frame: frame)
        isAccessibilityElement = true            // ← Required for custom views
        accessibilityTraits = .image
        accessibilityLabel = status.localizedDescription
    }

    required init?(coder: NSCoder) { fatalError() }
}
```

❌ **Incorrect — label set but view is invisible to VoiceOver:**

```swift
class StatusBadge: UIView {
    override init(frame: CGRect) {
        super.init(frame: frame)
        // ❌ isAccessibilityElement defaults to false — VoiceOver ignores this entirely
        accessibilityLabel = "Active"
    }
}
```

**`accessibilityLabel`** should be concise, localized, and must **not** include the control type (VoiceOver appends it automatically from traits):

```swift
// ✅ VoiceOver reads: "Add new flight — button"
addButton.accessibilityLabel = NSLocalizedString("Add new flight", comment: "")

// ❌ VoiceOver reads: "Add button — button"
addButton.accessibilityLabel = "Add button"
```

**`accessibilityHint`** describes the *result* of an action. Use it only when the label alone is ambiguous. It should begin with a third-person verb and end with a period:

```swift
// ✅ "Opens the flight details."
cell.accessibilityHint = NSLocalizedString("Opens the flight details.", comment: "")

// ❌ Repeats the label / names the gesture
deleteButton.accessibilityHint = "Tap to delete"
```

**`accessibilityTraits`** — always **insert and remove**, never assign outright (assigning overwrites UIKit's defaults):

```swift
// ✅ Preserves the existing .button trait
favoriteButton.accessibilityTraits.insert(.selected)

// ❌ Overwrites .button — VoiceOver no longer announces "button"
favoriteButton.accessibilityTraits = .selected
```

Common traits: `.button`, `.header` (enables the heading rotor), `.selected`, `.notEnabled` (announces "dimmed"), `.adjustable` (requires implementing `accessibilityIncrement()` / `accessibilityDecrement()`), `.image`, `.link`, `.startsMediaSession`. iOS 17 added **`.toggleButton`**, which announces "switch button" with a proper toggle hint.

Container views should have `isAccessibilityElement = false` so their children remain individually focusable. If you set a container to `true`, **all children become invisible** to VoiceOver.

### `UIAccessibilityCustomAction` for swipeable rows

VoiceOver users navigate with horizontal swipes, so they **cannot perform** the standard swipe-to-reveal actions on table rows. `UIAccessibilityCustomAction` exposes these operations through the VoiceOver Actions rotor (swipe up/down to browse, double-tap to activate).

✅ **Correct — custom actions mirror swipe actions:**

```swift
class MessageCell: UITableViewCell {
    func configure(with message: Message) {
        // ... UI setup ...

        accessibilityCustomActions = [
            UIAccessibilityCustomAction(
                name: NSLocalizedString("Delete", comment: "")
            ) { [weak self] _ in
                self?.delete(message)
                return true                    // true = action succeeded
            },
            UIAccessibilityCustomAction(
                name: NSLocalizedString("Archive", comment: "")
            ) { [weak self] _ in
                self?.archive(message)
                return true
            }
        ]
    }
}
```

❌ **Incorrect — swipe actions exist but no custom actions are provided:**

```swift
// ❌ VoiceOver users have NO way to delete or archive
func tableView(_ tableView: UITableView,
               trailingSwipeActionsConfigurationForRowAt indexPath: IndexPath
) -> UISwipeActionsConfiguration? {
    let delete = UIContextualAction(style: .destructive, title: "Delete") { ... }
    return UISwipeActionsConfiguration(actions: [delete])
    // Fix: also set accessibilityCustomActions on the cell
}
```

### Posting accessibility notifications

Three notifications handle nearly every dynamic-UI scenario. Choosing the wrong one confuses VoiceOver users.

| Notification | When to use | Argument |
|---|---|---|
| `.screenChanged` | A new modal or major screen appears | The new screen's view (or `nil`) |
| `.layoutChanged` | Part of the layout changed (error appeared, section reloaded) | The new/changed element (or `nil`) |
| `.announcement` | Transient info with no UI change (toast, background task finished) | A localized `String` |

✅ **Correct — use `.layoutChanged` for partial updates, focus on the new element:**

```swift
func showError(_ message: String) {
    errorLabel.text = message
    errorLabel.isHidden = false
    UIAccessibility.post(notification: .layoutChanged, argument: errorLabel)
}
```

❌ **Incorrect — using `.screenChanged` for a minor layout update:**

```swift
func showBanner() {
    bannerView.isHidden = false
    // ❌ Plays the "new screen" sound — overkill for a banner
    UIAccessibility.post(notification: .screenChanged, argument: bannerView)
}
```

❌ **Incorrect — using `.announcement` when focus should move:**

```swift
func showError() {
    errorLabel.isHidden = false
    // ❌ User hears the error text but focus stays on the old element
    UIAccessibility.post(notification: .announcement, argument: errorLabel.text)
}
```

✅ **Correct — `.screenChanged` for a custom modal with `accessibilityViewIsModal`:**

```swift
func presentOverlay() {
    let overlay = AlertOverlayView()
    overlay.accessibilityViewIsModal = true    // Trap VoiceOver focus inside
    view.addSubview(overlay)
    UIAccessibility.post(notification: .screenChanged, argument: overlay)
}
```

**iOS 17+ announcement priority** lets you control whether an announcement can be interrupted:

```swift
let urgent = NSAttributedString(
    string: "Connection lost",
    attributes: [.accessibilitySpeechAnnouncementPriority: UIAccessibilityPriority.high]
)
UIAccessibility.post(notification: .announcement, argument: urgent)
```

iOS 17 also introduced a Swift-native posting API that works across UIKit, AppKit, and SwiftUI:

```swift
AccessibilityNotification.Announcement("Photos loaded").post()
AccessibilityNotification.LayoutChanged(errorLabel).post()
```

### Grouping and reading order

Use `shouldGroupAccessibilityChildren = true` on a container so VoiceOver reads all its children before moving to the next sibling — useful for card-style layouts where the default left-to-right scan would jump across cards.

For fully custom reading order, override `accessibilityElements` (the container must have `isAccessibilityElement = false`):

```swift
override var accessibilityElements: [Any]? {
    get { [headerLabel, priceLabel, buyButton, disclaimerLabel] }
    set { }
}
```

---

## Conclusion

The iOS 17 trait system is a genuine leap: explicit trait registration eliminates wasted callbacks, `traitOverrides` replaces brittle environment manipulation, and `UITraitBridgedEnvironmentKey` unifies UIKit and SwiftUI state propagation. The price is one new mental-model rule — **the handler is not called on registration** — making `viewIsAppearing(_:)` essential for initial configuration. Across Dark Mode, Dynamic Type, and accessibility, the recurring theme is the same: use the *dynamic* version of every API (`UIColor.label` not `.black`, `preferredFont` + `adjustsFontForContentSizeCategory` not `systemFont(ofSize:)`, `.layoutChanged` not silence) and re-resolve the few things that aren't dynamic (`CGColor`, appearance proxies). Getting these patterns right once means your app adapts correctly to every device size, appearance, text size, and assistive technology — now and as Apple adds new traits in the future.
---

## Summary Checklist

- [ ] `registerForTraitChanges` (iOS 17+) used instead of deprecated `traitCollectionDidChange`
- [ ] Trait handler uses `self: Self` closure pattern (framework manages lifecycle, no `[weak self]` needed)
- [ ] Initial state set in `viewIsAppearing` (handler is NOT called on registration)
- [ ] Dynamic Type: `UIFont.preferredFont(forTextStyle:)` or `UIFontMetrics` for custom fonts
- [ ] `adjustsFontForContentSizeCategory = true` set on labels/text views for live text-size changes
- [ ] `numberOfLines = 0` set on labels to allow wrapping at larger text sizes
- [ ] Dark mode: semantic colors (`.label`, `.systemBackground`) used — not hardcoded colors
- [ ] `layer.borderColor` / `layer.shadowColor` (CGColor) re-resolved on trait changes
- [ ] Custom views set `isAccessibilityElement`, `accessibilityLabel`, `accessibilityTraits`
- [ ] Complex list items use `UIAccessibilityCustomAction` for VoiceOver-accessible actions
- [ ] Accessibility notifications posted: `.screenChanged`, `.layoutChanged`, `.announcement` as needed
