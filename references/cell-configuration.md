# UIKit modern cell configuration: the definitive Swift guide

Apple's content and background configuration system, introduced in iOS 14 and refined through iOS 18, is now the **only recommended way** to style cells in UICollectionView and UITableView. The legacy `textLabel`, `detailTextLabel`, and `imageView` properties are formally deprecated. This guide covers every configuration API in depth — from factory methods and state handling to custom configurations and reactive `@Observable` updates — with correct and incorrect code patterns throughout.

The configuration system revolves around lightweight **value-type structs** that describe appearance and content, then get applied to cells in a single assignment. This design eliminates entire classes of state bugs common with imperative cell subclassing and enables powerful state-driven styling with minimal code.

---

## 1. UIListContentConfiguration and its factory methods

`UIListContentConfiguration` is a struct (iOS 14+) providing pre-styled content layouts for list cells, headers, and footers. Each factory method returns a configuration with default styling matching a particular cell style — you then customize properties and assign it to the cell.

### Cell configurations

| Factory Method | Layout | Equivalent Legacy Style |
|---|---|---|
| `.cell()` | Image &#124; Primary text | `UITableViewCell.CellStyle.default` |
| `.subtitleCell()` | Image &#124; Primary text / Secondary text (stacked) | `.subtitle` |
| `.valueCell()` | Image &#124; Primary text … Secondary text (side-by-side) | `.value1` |
| `.sidebarCell()` | Sidebar-styled cell with prominent icon | iPadOS sidebar |
| `.sidebarSubtitleCell()` | Sidebar cell with subtitle | iPadOS sidebar |
| `.accompaniedSidebarCell()` | Accompanied sidebar (split view) | — |
| `.accompaniedSidebarSubtitleCell()` | Accompanied sidebar with subtitle | — |

### Header and footer configurations

| Factory Method | Since | Notes |
|---|---|---|
| `.plainHeader()` / `.plainFooter()` | iOS 14 | Plain list style |
| `.groupedHeader()` / `.groupedFooter()` | iOS 14 | Grouped list style |
| `.sidebarHeader()` | iOS 14 | Sidebar list style |
| `.prominentInsetGroupedHeader()` | iOS 15 | Larger, bolder header for inset grouped |
| `.extraProminentInsetGroupedHeader()` | iOS 15 | Even more prominent |
| **`.header()`** / **`.footer()`** | **iOS 18** | Style-agnostic; auto-adapts via `UIListEnvironment` trait |

**iOS 18 key change:** The new `.header()` and `.footer()` methods — plus the existing `.cell()`, `.subtitleCell()`, and `.valueCell()` — now automatically adapt their appearance based on the **`UIListEnvironment`** trait in the cell's trait collection. This means a single configuration works correctly whether the cell appears in a plain, grouped, inset grouped, or sidebar list.

> ⚠️ **Note:** The correct factory method name is **`.valueCell()`**, not `.valueCellConfiguration()`.

### Configuring text, image, and layout properties

✅ **Correct — modern content configuration:**

```swift
var content = cell.defaultContentConfiguration() // or UIListContentConfiguration.subtitleCell()

// Text
content.text = "Documents"
content.secondaryText = "23 items"

// Text properties
content.textProperties.font = .preferredFont(forTextStyle: .headline)
content.textProperties.color = .label
content.textProperties.numberOfLines = 2
content.textProperties.adjustsFontForContentSizeCategory = true

content.secondaryTextProperties.font = .preferredFont(forTextStyle: .subheadline)
content.secondaryTextProperties.color = .secondaryLabel

// Image
content.image = UIImage(systemName: "folder.fill")
content.imageProperties.tintColor = .systemBlue
content.imageProperties.preferredSymbolConfiguration = UIImage.SymbolConfiguration(
    pointSize: 24, weight: .medium, scale: .default
)
content.imageProperties.cornerRadius = 6
content.imageProperties.maximumSize = CGSize(width: 40, height: 40)

// Spacing
content.imageToTextPadding = 12
content.textToSecondaryTextVerticalPadding = 4
content.directionalLayoutMargins = NSDirectionalEdgeInsets(top: 12, leading: 16, bottom: 12, trailing: 16)

cell.contentConfiguration = content
```

❌ **Incorrect — using deprecated properties:**

```swift
// ❌ Deprecated since iOS 14; do NOT use
cell.textLabel?.text = "Documents"
cell.detailTextLabel?.text = "23 items"
cell.imageView?.image = UIImage(systemName: "folder.fill")
cell.imageView?.tintColor = .systemBlue
```

Key `TextProperties` include `font`, `color`, `numberOfLines`, `lineBreakMode`, `alignment`, `adjustsFontSizeToFitWidth`, `minimumScaleFactor`, and `transform` (`.uppercase`, `.lowercase`, `.capitalized`). Key `ImageProperties` include `tintColor`, `preferredSymbolConfiguration`, `cornerRadius`, `maximumSize`, and **`reservedLayoutSize`** — useful for aligning columns of cells with different-width images by reserving a fixed space.

---

## 2. UIBackgroundConfiguration for state-aware cell backgrounds

`UIBackgroundConfiguration` controls corner radius, stroke, fill color, and visual effects for cell backgrounds. Like content configurations, it's a value-type struct applied in one shot.

### Factory methods

| Factory Method | Since | Notes |
|---|---|---|
| `.listPlainCell()` | iOS 14 | Plain list cell background |
| `.listGroupedCell()` | iOS 14 | Grouped list cell (rounded corners in inset grouped) |
| `.listSidebarCell()` | iOS 14 | Sidebar cell background |
| `.listAccompaniedSidebarCell()` | iOS 14 | Accompanied sidebar cell |
| `.listPlainHeaderFooter()` | iOS 14 | **Deprecated in iOS 18** |
| `.listGroupedHeaderFooter()` | iOS 14 | **Deprecated in iOS 18** |
| `.listSidebarHeader()` | iOS 14 | **Deprecated in iOS 18** |
| `.clear()` | iOS 14 | Fully transparent, no default styling |
| **`.listCell()`** | **iOS 18** | Auto-adapts to `UIListEnvironment` |
| **`.listHeader()`** | **iOS 18** | Replaces deprecated header methods |
| **`.listFooter()`** | **iOS 18** | Replaces deprecated footer methods |

### Customizing backgrounds per state

✅ **Correct — background configuration with custom fill, stroke, and corner radius:**

```swift
var background = UIBackgroundConfiguration.listGroupedCell()
background.cornerRadius = 12
background.backgroundColor = .secondarySystemGroupedBackground
background.strokeColor = .systemGray3
background.strokeWidth = 1.0
background.strokeOutset = 0  // positive = outset, negative = inset
background.visualEffect = nil // or UIBlurEffect(style: .systemMaterial) for blur

cell.backgroundConfiguration = background
```

✅ **Correct — using `configurationUpdateHandler` for per-state backgrounds (iOS 15+):**

```swift
cell.configurationUpdateHandler = { cell, state in
    var background = UIBackgroundConfiguration.listGroupedCell()
    
    if state.isHighlighted {
        background.backgroundColor = .systemGray4
    } else if state.isSelected {
        background.backgroundColor = .systemBlue.withAlphaComponent(0.2)
        background.strokeColor = .systemBlue
        background.strokeWidth = 2.0
    } else if state.isDisabled {
        background.backgroundColor = .systemGray6
    } else {
        background.backgroundColor = .secondarySystemGroupedBackground
    }
    
    background.cornerRadius = 10
    cell.backgroundConfiguration = background
}
```

❌ **Incorrect — mixing legacy and modern background APIs:**

```swift
// ❌ Setting backgroundConfiguration resets backgroundColor and backgroundView to nil.
// Never mix old and new APIs on the same cell.
cell.backgroundConfiguration = UIBackgroundConfiguration.listGroupedCell()
cell.backgroundColor = .red  // ❌ Conflicts — will be overwritten
cell.backgroundView = myCustomView // ❌ Conflicts
```

The `backgroundColorTransformer` property offers a functional alternative — a `UIConfigurationColorTransformer` closure that transforms the resolved color. This is useful for tinting based on external state without creating an entirely new configuration each time. The system automatically calls `updated(for:)` on background configurations when `automaticallyUpdatesBackgroundConfiguration` is `true` (the default), applying platform-standard highlight and selection colors.

---

## 3. configurationUpdateHandler replaces subclassing for state changes

Introduced in **iOS 15**, `configurationUpdateHandler` is a closure property on `UICollectionViewCell`, `UITableViewCell`, and `UITableViewHeaderFooterView`. It fires whenever the cell's **configuration state changes** — selection, highlight, swipe, drag, editing, focus, and trait collection changes (including Dark Mode).

### Closure signature

```swift
// UICollectionViewCell
var configurationUpdateHandler: ((_ cell: UICollectionViewCell, _ state: UICellConfigurationState) -> Void)?

// UITableViewCell
var configurationUpdateHandler: ((_ cell: UITableViewCell, _ state: UICellConfigurationState) -> Void)?
```

### UICellConfigurationState properties

The `UICellConfigurationState` struct exposes these properties:

- **`isSelected`** — cell is selected
- **`isHighlighted`** — cell is highlighted (touch down)
- **`isFocused`** — cell has focus (tvOS, iPadOS keyboard)
- **`isDisabled`** — cell is disabled
- **`isEditing`** — cell is in editing mode
- **`isSwiped`** — cell has a visible swipe action
- **`isExpanded`** — cell is expanded (outlines)
- **`isReordering`** — cell is being reordered
- **`cellDragState`** — enum: `.none`, `.lifting`, `.dragging`
- **`cellDropState`** — enum: `.none`, `.notTargeted`, `.targeted`
- **`traitCollection`** — current `UITraitCollection`

> ⚠️ There is no `isDragging` Bool. Drag state uses the **`cellDragState`** enum instead.

✅ **Correct — inline state handling without subclassing:**

```swift
let registration = UICollectionView.CellRegistration<UICollectionViewListCell, Item> { cell, indexPath, item in
    cell.configurationUpdateHandler = { cell, state in
        // Content
        var content = UIListContentConfiguration.subtitleCell().updated(for: state)
        content.text = item.title
        content.secondaryText = item.subtitle
        content.image = UIImage(systemName: item.iconName)
        
        if state.isHighlighted || state.isSelected {
            content.textProperties.color = .white
            content.imageProperties.tintColor = .white
        }
        
        if state.isSwiped {
            content.textProperties.color = .secondaryLabel
        }
        
        cell.contentConfiguration = content
        
        // Background
        var background = UIBackgroundConfiguration.listGroupedCell().updated(for: state)
        if state.isSelected {
            background.backgroundColor = .systemBlue
        }
        cell.backgroundConfiguration = background
    }
}
```

❌ **Incorrect — subclassing just for state-dependent styling (pre-iOS 15 pattern):**

```swift
// ❌ Unnecessary subclass just to override updateConfiguration(using:)
class MyCell: UICollectionViewListCell {
    var item: Item?
    
    override func updateConfiguration(using state: UICellConfigurationState) {
        super.updateConfiguration(using: state)
        var content = defaultContentConfiguration().updated(for: state)
        content.text = item?.title
        // ... same logic that could be inline
        contentConfiguration = content
    }
}
```

### Custom state keys

You can define custom state that participates in configuration updates via **`UIConfigurationStateCustomKey`**:

```swift
// ✅ Correct: custom state key with typed accessor and configurationState override
// 1. Declare the key
extension UIConfigurationStateCustomKey {
    static let isArchived = UIConfigurationStateCustomKey("com.myapp.isArchived")
}

// 2. Add a typed accessor
extension UICellConfigurationState {
    var isArchived: Bool {
        get { self[.isArchived] as? Bool ?? false }
        set { self[.isArchived] = newValue }
    }
}

// 3. Override configurationState in a cell subclass to inject the value
class ArchivableCell: UICollectionViewListCell {
    var isArchived = false {
        didSet { if oldValue != isArchived { setNeedsUpdateConfiguration() } }
    }
    
    override var configurationState: UICellConfigurationState {
        var state = super.configurationState
        state.isArchived = isArchived
        return state
    }
}

// 4. Read it in the handler
cell.configurationUpdateHandler = { cell, state in
    var content = UIListContentConfiguration.cell().updated(for: state)
    content.text = item.title
    content.textProperties.color = state.isArchived ? .tertiaryLabel : .label
    cell.contentConfiguration = content
}
```

### iOS 18 enhancement — automatic trait tracking

In iOS 18, UIKit now **automatically tracks trait reads** inside `configurationUpdateHandler`. If the closure reads `state.traitCollection.userInterfaceStyle`, UIKit records that dependency and re-invokes the handler when the trait changes. No manual `registerForTraitChanges` call is needed.

---

## 4. updated(for:) versus configurationUpdateHandler

Both `UIContentConfiguration` and `UIBackgroundConfiguration` expose an `updated(for:)` method that returns a **new configuration** with system-default styling applied for a given state. This is complementary to — not a replacement for — `configurationUpdateHandler`.

### How updated(for:) works

```swift
// Returns a new configuration with system defaults for the given state
let base = UIListContentConfiguration.cell()
let styled = base.updated(for: state)
// `styled` now has appropriate text colors for selected/highlighted/disabled states
```

When **`automaticallyUpdatesContentConfiguration`** is `true` (the default), the cell automatically calls `updated(for:)` on the current content configuration whenever the state changes, without any code from you. The same applies to **`automaticallyUpdatesBackgroundConfiguration`** for background configurations.

### When to use each

| Scenario | Use |
|---|---|
| System-default state styling is sufficient | Set `automaticallyUpdatesContentConfiguration = true` (default) — `updated(for:)` is called for you |
| Custom per-state styling beyond defaults | Set the handler, call `updated(for:)` inside it, then customize further |
| Full manual control | Set `automaticallyUpdatesContentConfiguration = false`, use `configurationUpdateHandler` exclusively |

✅ **Correct — combining both approaches:**

```swift
cell.configurationUpdateHandler = { cell, state in
    // Start with system defaults for this state
    var content = UIListContentConfiguration.subtitleCell().updated(for: state)
    content.text = item.title
    content.secondaryText = item.detail
    
    // Then apply custom overrides
    if state.isSelected {
        content.textProperties.color = .white
    }
    
    cell.contentConfiguration = content
}
```

❌ **Incorrect — ignoring updated(for:) and losing system state styling:**

```swift
cell.configurationUpdateHandler = { cell, state in
    // ❌ Creates a fresh config that ignores the current state
    var content = UIListContentConfiguration.subtitleCell() // missing .updated(for: state)
    content.text = item.title
    cell.contentConfiguration = content
    // Selected/highlighted cells won't get system-default color adjustments
}
```

---

## 5. Building a custom UIContentConfiguration from scratch

When `UIListContentConfiguration` doesn't cover your layout needs, create a custom content configuration. The pattern uses two types: a **configuration struct** (data + factory) and a **content view class** (UIView rendering).

### The UIContentConfiguration protocol

```swift
public protocol UIContentConfiguration {
    func makeContentView() -> UIView & UIContentView
    func updated(for state: UIConfigurationState) -> Self
}
```

### The UIContentView protocol

```swift
public protocol UIContentView: AnyObject {
    var configuration: UIContentConfiguration { get set }
}
```

### Complete working example — a rating cell

✅ **Correct — full custom content configuration:**

```swift
// MARK: - Configuration (value type)
struct RatingContentConfiguration: UIContentConfiguration, Hashable {
    var title: String = ""
    var rating: Int = 0       // 0–5 stars
    var subtitle: String = ""
    var isHighlighted: Bool = false
    
    func makeContentView() -> UIView & UIContentView {
        RatingContentView(self)
    }
    
    func updated(for state: UIConfigurationState) -> RatingContentConfiguration {
        guard let state = state as? UICellConfigurationState else { return self }
        var updated = self
        updated.isHighlighted = state.isHighlighted || state.isSelected
        return updated
    }
}

// MARK: - Content View
class RatingContentView: UIView, UIContentView {
    
    var configuration: UIContentConfiguration {
        didSet { apply(configuration) }
    }
    
    private let titleLabel = UILabel()
    private let subtitleLabel = UILabel()
    private let starsLabel = UILabel()
    
    init(_ configuration: UIContentConfiguration) {
        self.configuration = configuration
        super.init(frame: .zero)
        setupViews()
        apply(configuration)
    }
    
    required init?(coder: NSCoder) { fatalError("init(coder:) has not been implemented") }
    
    private func setupViews() {
        let textStack = UIStackView(arrangedSubviews: [titleLabel, subtitleLabel])
        textStack.axis = .vertical
        textStack.spacing = 4
        
        let mainStack = UIStackView(arrangedSubviews: [textStack, starsLabel])
        mainStack.axis = .horizontal
        mainStack.alignment = .center
        mainStack.spacing = 12
        mainStack.translatesAutoresizingMaskIntoConstraints = false
        
        addSubview(mainStack)
        NSLayoutConstraint.activate([
            mainStack.topAnchor.constraint(equalTo: layoutMarginsGuide.topAnchor),
            mainStack.bottomAnchor.constraint(equalTo: layoutMarginsGuide.bottomAnchor),
            mainStack.leadingAnchor.constraint(equalTo: layoutMarginsGuide.leadingAnchor),
            mainStack.trailingAnchor.constraint(equalTo: layoutMarginsGuide.trailingAnchor),
        ])
        
        titleLabel.font = .preferredFont(forTextStyle: .body)
        subtitleLabel.font = .preferredFont(forTextStyle: .caption1)
        subtitleLabel.textColor = .secondaryLabel
        starsLabel.font = .preferredFont(forTextStyle: .body)
        starsLabel.setContentHuggingPriority(.required, for: .horizontal)
    }
    
    private func apply(_ config: UIContentConfiguration) {
        guard let config = config as? RatingContentConfiguration else { return }
        titleLabel.text = config.title
        subtitleLabel.text = config.subtitle
        starsLabel.text = String(repeating: "★", count: config.rating)
                        + String(repeating: "☆", count: 5 - config.rating)
        
        let textColor: UIColor = config.isHighlighted ? .white : .label
        titleLabel.textColor = textColor
        starsLabel.textColor = config.isHighlighted ? .white : .systemYellow
    }
}

// MARK: - Usage in a cell registration
let registration = UICollectionView.CellRegistration<UICollectionViewCell, Restaurant> {
    cell, indexPath, restaurant in
    
    var config = RatingContentConfiguration()
    config.title = restaurant.name
    config.subtitle = restaurant.cuisine
    config.rating = restaurant.starRating
    cell.contentConfiguration = config
}
```

When UIKit assigns a configuration to a cell that already holds a `RatingContentView`, it sets the `configuration` property directly (triggering `didSet` → `apply`) rather than calling `makeContentView()` again. This makes reconfiguration efficient — **subview setup in `init` runs once**; only data application runs on reuse.

❌ **Incorrect — using a class instead of a struct for the configuration:**

```swift
// ❌ Reference type loses value semantics; mutations propagate unexpectedly
class BadConfiguration: UIContentConfiguration {
    var title = ""
    // Mutations to a shared reference corrupt other cells
    func makeContentView() -> UIView & UIContentView { /* ... */ }
    func updated(for state: UIConfigurationState) -> Self { return self }
}
```

---

## 6. Reactive cell updates with @Observable models on iOS 18+

Starting with the iOS 18 runtime, UIKit can **automatically track reads of `@Observable` properties** inside `configurationUpdateHandler` and re-invoke the handler when those properties change. This eliminates the need for manual snapshot `reconfigureItems` calls for in-place content updates.

### Enabling observation tracking

Add to your **Info.plist**:

```xml
<key>UIObservationTrackingEnabled</key>
<true/>
```

On **iOS 26+** (2025), this key is unnecessary — observation tracking is enabled by default. On iOS 18 through 25, the key is required to opt in. The feature was shipped in the iOS 18 runtime but wasn't publicly documented until WWDC 2025.

### How it works

UIKit wraps `configurationUpdateHandler` execution in an observation tracking context. Every `@Observable` property read during the closure is recorded. When any tracked property mutates, UIKit calls `setNeedsUpdateConfiguration()` on the cell automatically, which re-invokes the handler on the next update cycle.

✅ **Correct — automatic cell updates via observation (iOS 18+, with plist key):**

```swift
@Observable class TaskModel {
    var title: String
    var isComplete: Bool
    var priority: Int
    
    init(title: String, isComplete: Bool = false, priority: Int = 0) {
        self.title = title
        self.isComplete = isComplete
        self.priority = priority
    }
}

// In your data source setup — set up the handler once
let cellRegistration = UICollectionView.CellRegistration<UICollectionViewListCell, TaskModel> {
    cell, indexPath, task in
    
    cell.configurationUpdateHandler = { cell, state in
        var content = UIListContentConfiguration.cell().updated(for: state)
        content.text = task.title           // ← tracked automatically
        content.image = UIImage(systemName: task.isComplete ? "checkmark.circle.fill" : "circle")
        
        content.textProperties.color = task.isComplete ? .secondaryLabel : .label
        content.imageProperties.tintColor = task.isComplete ? .systemGreen : .systemGray
        
        cell.contentConfiguration = content
    }
}

// Later, from anywhere:
task.title = "Updated Title"    // Cell updates automatically — no snapshot needed
task.isComplete = true          // Cell updates automatically
```

❌ **Incorrect — manually reconfiguring snapshots for every property change (pre-observation pattern):**

```swift
// ❌ Verbose, error-prone, and now unnecessary for property changes
func taskDidChange(_ task: TaskModel) {
    var snapshot = dataSource.snapshot()
    snapshot.reconfigureItems([task.id])       // ❌ Manual invalidation
    dataSource.apply(snapshot, animatingDifferences: true) // ❌ Snapshot churn
}
```

### What observation tracking replaces — and what it doesn't

Observation tracking replaces `reconfigureItems` / `reloadItems` for **property-level content updates** to visible cells. You still need diffable data source snapshots for **structural changes** — inserting, deleting, or reordering items. Think of it as: snapshots manage *which* items exist; observation tracking manages *what* those items display.

The tracking is **lazy and conditional**. Only properties accessed during the handler's execution create dependencies. If the handler hits an `if/else` branch, only properties from the executed branch are tracked. Dependencies re-evaluate each time the handler runs, so tracking adapts automatically to changing code paths.

### Supported update methods beyond cells

Observation tracking (with the plist key on iOS 18, or by default on iOS 26) works in these methods:

- **UIView**: `layoutSubviews()`, `updateConstraints()`, `draw(_:)`
- **UIViewController**: `viewWillLayoutSubviews()`, `updateViewConstraints()`
- **Cells**: `configurationUpdateHandler`, `updateConfiguration(using:)`
- **iOS 26 only**: `updateProperties()` on UIView and UIViewController — a new update phase that runs before layout and is the recommended place for observation-driven property changes

The feature works with both `UICollectionView` and `UITableView`, and with custom `UIContentConfiguration` types — not just `UIListContentConfiguration`.

---

## 7. Migrating from deprecated cell properties to content configurations

The `textLabel`, `detailTextLabel`, and `imageView` properties on `UITableViewCell` were introduced in iOS 3, deprecated starting with iOS 14's modern configuration APIs, and produce compiler warnings in current Xcode. They are **mutually exclusive** with `contentConfiguration` — setting one nullifies the other.

### Before and after

❌ **Before — deprecated:**

```swift
override func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
    let cell = tableView.dequeueReusableCell(withIdentifier: "cell", for: indexPath)
    let item = items[indexPath.row]
    
    cell.textLabel?.text = item.title                          // ❌ Deprecated
    cell.textLabel?.font = .preferredFont(forTextStyle: .body) // ❌ Deprecated
    cell.textLabel?.textColor = .label                         // ❌ Deprecated
    cell.detailTextLabel?.text = item.subtitle                 // ❌ Deprecated
    cell.imageView?.image = UIImage(systemName: item.icon)     // ❌ Deprecated
    cell.imageView?.tintColor = .systemBlue                    // ❌ Deprecated
    
    return cell
}
```

✅ **After — modern content configuration:**

```swift
override func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
    let cell = tableView.dequeueReusableCell(withIdentifier: "cell", for: indexPath)
    let item = items[indexPath.row]
    
    var content = cell.defaultContentConfiguration()
    content.text = item.title
    content.textProperties.font = .preferredFont(forTextStyle: .body)
    content.textProperties.color = .label
    content.secondaryText = item.subtitle
    content.image = UIImage(systemName: item.icon)
    content.imageProperties.tintColor = .systemBlue
    
    cell.contentConfiguration = content
    return cell
}
```

### Property mapping reference

| Deprecated API | Modern Replacement |
|---|---|
| `cell.textLabel?.text` | `config.text` |
| `cell.textLabel?.attributedText` | `config.attributedText` |
| `cell.textLabel?.font` | `config.textProperties.font` |
| `cell.textLabel?.textColor` | `config.textProperties.color` |
| `cell.textLabel?.numberOfLines` | `config.textProperties.numberOfLines` |
| `cell.detailTextLabel?.text` | `config.secondaryText` |
| `cell.imageView?.image` | `config.image` |
| `cell.imageView?.tintColor` | `config.imageProperties.tintColor` |
| (no equivalent) | `config.imageProperties.preferredSymbolConfiguration` |
| (no equivalent) | `config.imageToTextPadding` |
| (no equivalent) | `config.imageProperties.reservedLayoutSize` |

### Critical migration pitfalls

**Never mix APIs.** Setting `contentConfiguration` resets `textLabel`/`detailTextLabel`/`imageView` to `nil`, and vice versa. Pick one approach per cell. Similarly, setting `backgroundConfiguration` nullifies `backgroundColor` and `backgroundView`.

**Always start from a fresh configuration.** Call `cell.defaultContentConfiguration()` (or a static factory like `.subtitleCell()`) each time — don't try to read back and mutate the existing `cell.contentConfiguration`. Configurations are value types designed for write-once-per-cycle semantics.

**`defaultContentConfiguration()` is style-aware.** On `UITableViewCell`, it returns a configuration matching the cell's init style (`.default`, `.subtitle`, `.value1`). On `UICollectionViewListCell`, it matches the list layout's appearance. A plain `UICollectionViewCell` does **not** vend a default content configuration — use the static factory methods instead.

**Don't access underlying subviews directly.** With content configurations, there is no public `UILabel` or `UIImageView` to grab. All customization goes through the configuration's `textProperties`, `secondaryTextProperties`, and `imageProperties` structs.

---

## Conclusion

UIKit's configuration system has matured into a **complete, composable, state-driven architecture** for cell styling. The iOS 18 additions — style-agnostic factory methods via `UIListEnvironment`, and automatic `@Observable` tracking (opt-in via plist key, default in iOS 26) — eliminate two of the biggest remaining pain points: hardcoding list styles and manually triggering cell refreshes.

The mental model is straightforward. Create a configuration struct (system-provided or custom), populate it with data, and assign it. For state-dependent styling, use `configurationUpdateHandler` with `updated(for:)` to get system defaults, then layer on custom overrides. For reactive models, adopt `@Observable` and let UIKit handle invalidation. The deprecated `textLabel`/`imageView` path has no remaining advantages — migrating is both safer and more powerful.
---

## Summary Checklist

- [ ] Using `UIContentConfiguration` (`UIListContentConfiguration`) — not deprecated `textLabel` / `detailTextLabel` / `imageView`
- [ ] `cell.defaultContentConfiguration()` or factory methods (`.cell()`, `.subtitleCell()`) used correctly
- [ ] `UIBackgroundConfiguration` used for cell backgrounds — not direct `backgroundColor` manipulation
- [ ] `configurationUpdateHandler` used for state-dependent styling (selection, highlight) — not subclass overrides
- [ ] `updated(for: state)` called on configuration inside `configurationUpdateHandler`
- [ ] Custom configurations conform to `UIContentConfiguration` with `makeContentView()` and `updated(for:)`
- [ ] Custom content views conform to `UIContentView` with `configuration` property
- [ ] iOS 18+: `UIObservationTrackingEnabled` considered for reactive cell updates with `@Observable`
