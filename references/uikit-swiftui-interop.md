# UIKit–SwiftUI interoperability: the definitive technical reference

**The bridge between UIKit and SwiftUI is now mature but filled with subtle traps.** Since iOS 16, Apple has delivered `sizingOptions`, `UIHostingConfiguration`, and `sizeThatFits(_:)` for representables. iOS 17 added `@Observable` and `UITraitBridgedEnvironmentKey` for bidirectional state flow. iOS 18 brought automatic trait tracking and animation interop. And iOS 26 (beta) introduces native `@Observable` tracking in UIKit itself. This reference covers every major interop pattern with correct and incorrect code examples, spanning iOS 13 through iOS 26.

---

## 1. UIHostingController containment must follow three exact steps

UIKit's view controller containment API requires a strict sequence when embedding a `UIHostingController` inside a parent. Skipping any step breaks appearance callbacks, trait propagation, and the SwiftUI environment.

**The required sequence:**

```swift
// ✅ CORRECT: Full containment pattern
class ParentViewController: UIViewController {
    private var hostingController: UIHostingController<MySwiftUIView>!

    override func viewDidLoad() {
        super.viewDidLoad()
        let swiftUIView = MySwiftUIView(viewModel: viewModel)
        hostingController = UIHostingController(rootView: swiftUIView)

        addChild(hostingController)                              // Step 1
        hostingController.view.translatesAutoresizingMaskIntoConstraints = false
        view.addSubview(hostingController.view)                  // Step 2
        hostingController.didMove(toParent: self)                // Step 3

        NSLayoutConstraint.activate([
            hostingController.view.topAnchor.constraint(equalTo: view.topAnchor),
            hostingController.view.leadingAnchor.constraint(equalTo: view.leadingAnchor),
            hostingController.view.trailingAnchor.constraint(equalTo: view.trailingAnchor),
            hostingController.view.bottomAnchor.constraint(equalTo: view.bottomAnchor)
        ])
    }
}
```

**Skipping `addChild(_:)`** means UIKit never forwards `viewWillAppear`, `viewDidAppear`, rotation events, or trait collection changes. For `UIHostingController`, this silently breaks SwiftUI environment values like `colorScheme` and `horizontalSizeClass`. **Skipping `didMove(toParent:)`** leaves the containment transition incomplete, producing `UIViewControllerHierarchyInconsistency` warnings at runtime.

```swift
// ❌ WRONG: Only adding the view without containment
let hosting = UIHostingController(rootView: MySwiftUIView())
view.addSubview(hosting.view)  // Appears, but lifecycle is broken
// Missing: addChild, didMove(toParent:)
// Result: onAppear/onDisappear never fire, traits don't propagate
```

### The hosting controller must be retained as a stored property

`UIHostingController` manages the SwiftUI render tree, state updates, and trait propagation. If it's deallocated while its view remains in the hierarchy, the view becomes an orphan that can never update.

```swift
// ❌ WRONG: Local variable — deallocated immediately
func addSwiftUIView<T: View>(view: T) {
    let hostingController = UIHostingController(rootView: view)
    view.addSubview(hostingController.view)
    // hostingController deallocated at end of scope
    // View remains but: sizingOptions stop, @State freezes, traits break
}
```

**Concrete bugs:** `sizingOptions` never fires because the owning controller is gone. `@State` changes inside the SwiftUI view silently stop triggering re-renders. Dark mode switches are never forwarded.

### sizingOptions (iOS 16+) solve automatic size tracking

Before iOS 16, developers subclassed `UIHostingController` and called `invalidateIntrinsicContentSize()` manually in `viewDidLayoutSubviews`. The `sizingOptions` property eliminates this hack.

**`.intrinsicContentSize`** makes the hosting controller's view automatically invalidate its intrinsic content size whenever the SwiftUI content's ideal size changes. Use this for Auto Layout containers, stack views, scroll views, and table/collection view cells:

```swift
// ✅ CORRECT: intrinsicContentSize for Auto Layout
hostingController.sizingOptions = .intrinsicContentSize
addChild(hostingController)
hostingController.view.translatesAutoresizingMaskIntoConstraints = false
view.addSubview(hostingController.view)
hostingController.didMove(toParent: self)
// Pin edges only — no width/height constraints needed
```

**`.preferredContentSize`** tracks the SwiftUI content's ideal size in the view controller's `preferredContentSize` property. Use this for popovers and custom container VCs that read `preferredContentSize`:

```swift
// ✅ CORRECT: preferredContentSize for popovers
let hc = UIHostingController(rootView: PopoverContent())
hc.sizingOptions = .preferredContentSize
hc.modalPresentationStyle = .popover
hc.popoverPresentationController?.sourceView = sender
present(hc, animated: true)  // Popover auto-sizes to SwiftUI content
```

Apple's documentation notes that `.preferredContentSize` has a performance cost because it queries the ideal size using an unspecified size proposal on every update.

---

## 2. UIViewRepresentable lifecycle: create once, update many times

SwiftUI calls the three protocol methods in a fixed order: **`makeCoordinator()` → `makeUIView(context:)` → `updateUIView(_:context:)`**. The coordinator is created first so it can be assigned as a delegate during view creation. `makeUIView` is called exactly once. `updateUIView` is called immediately after `makeUIView` and again on every SwiftUI state change.

The critical insight: **SwiftUI reuses the underlying UIView instance** even when the `UIViewRepresentable` struct is recreated. The struct is a value type — a description, not a reference. Any configuration that depends on SwiftUI state must go in `updateUIView`, not `makeUIView`.

```swift
// ✅ CORRECT: State-dependent config in updateUIView
struct SearchField: UIViewRepresentable {
    @Binding var text: String

    func makeCoordinator() -> Coordinator { Coordinator(text: $text) }

    func makeUIView(context: Context) -> UITextField {
        let textField = UITextField()
        textField.borderStyle = .roundedRect        // Static — fine here
        textField.delegate = context.coordinator
        textField.addTarget(context.coordinator,
            action: #selector(Coordinator.textChanged(_:)), for: .editingChanged)
        return textField
    }

    func updateUIView(_ uiView: UITextField, context: Context) {
        if uiView.text != text { uiView.text = text }  // Dynamic — must be here
    }

    class Coordinator: NSObject, UITextFieldDelegate {
        var text: Binding<String>
        init(text: Binding<String>) { self.text = text }
        @objc func textChanged(_ textField: UITextField) {
            text.wrappedValue = textField.text ?? ""
        }
    }
}
```

```swift
// ❌ WRONG: Setting text in makeUIView — only runs once
func makeUIView(context: Context) -> UITextField {
    let textField = UITextField()
    textField.text = text  // Never updates when binding changes!
    return textField
}
func updateUIView(_ uiView: UITextField, context: Context) { }  // Empty!
```

### Guarding against infinite update loops

The classic infinite loop: delegate callback → updates `@Binding` → SwiftUI state change → calls `updateUIView` → updates UIView property → triggers delegate → repeats forever. Two proven guard patterns exist.

**Pattern 1 — Value-comparison guard** (simplest and most common):

```swift
// ✅ CORRECT: Only set when value actually differs
func updateUIView(_ uiView: MKMapView, context: Context) {
    if uiView.centerCoordinate != centerCoordinate {
        uiView.centerCoordinate = centerCoordinate  // Won't retrigger delegate
    }
}
```

**Pattern 2 — Internal-edit flag** (for complex delegates):

```swift
// ✅ CORRECT: Flag suppresses re-entrant updates
class Coordinator: NSObject, UITextViewDelegate {
    var text: Binding<String>
    var isInternalUpdate = false

    func textViewDidChange(_ textView: UITextView) {
        isInternalUpdate = true
        text.wrappedValue = textView.text
    }
}

func updateUIView(_ uiView: UITextView, context: Context) {
    guard !context.coordinator.isInternalUpdate else {
        context.coordinator.isInternalUpdate = false
        return
    }
    if uiView.text != text { uiView.text = text }
}
```

A third technique from Chris Eidhof: **update SwiftUI state asynchronously** from delegate callbacks via `DispatchQueue.main.async`, which breaks the synchronous re-entrant cycle.

```swift
// ❌ WRONG: No guards — infinite loop
func updateUIView(_ uiView: UITextView, context: Context) {
    uiView.text = text  // Always sets, triggers delegate, which sets binding...
}
```

---

## 3. UIViewControllerRepresentable follows the same lifecycle with extra nuances

The lifecycle mirrors UIViewRepresentable: `makeCoordinator()` → `makeUIViewController(context:)` → `updateUIViewController(_:context:)`. The wrapped view controller goes through its own UIKit lifecycle (`viewDidLoad`, `viewWillAppear`, etc.) managed by SwiftUI.

```swift
// ✅ CORRECT: PHPicker with full lifecycle handling
struct ImagePicker: UIViewControllerRepresentable {
    @Binding var selectedImage: UIImage?
    @Environment(\.dismiss) var dismiss

    func makeCoordinator() -> Coordinator { Coordinator(self) }

    func makeUIViewController(context: Context) -> PHPickerViewController {
        var config = PHPickerConfiguration()
        config.selectionLimit = 1
        config.filter = .images
        let picker = PHPickerViewController(configuration: config)
        picker.delegate = context.coordinator
        return picker
    }

    func updateUIViewController(_ vc: PHPickerViewController, context: Context) {
        context.coordinator.parent = self  // Refresh binding references
    }

    class Coordinator: PHPickerViewControllerDelegate {
        var parent: ImagePicker
        init(_ parent: ImagePicker) { self.parent = parent }

        func picker(_ picker: PHPickerViewController,
                    didFinishPicking results: [PHPickerResult]) {
            if let provider = results.first?.itemProvider,
               provider.canLoadObject(ofClass: UIImage.self) {
                provider.loadObject(ofClass: UIImage.self) { image, _ in
                    DispatchQueue.main.async {
                        self.parent.selectedImage = image as? UIImage
                        self.parent.dismiss()
                    }
                }
            } else { parent.dismiss() }
        }
    }
}
```

**Key nuance:** Never push view controllers or present modals inside `updateUIViewController` — it runs on every state change. Navigation actions must be guarded or triggered only by explicit user intent via the coordinator.

```swift
// ❌ WRONG: Pushing a VC in updateUIViewController
func updateUIViewController(_ nav: UINavigationController, context: Context) {
    let detail = DetailViewController()
    detail.data = self.data
    nav.pushViewController(detail, animated: true)  // Pushes on EVERY state change!
}
```

**Cleanup** goes in the static `dismantleUIView(_:coordinator:)` or `dismantleUIViewController(_:coordinator:)` methods — called before the view is removed from the hierarchy. Use these to remove `NotificationCenter` observers, invalidate timers, and cancel network requests.

---

## 4. Sizing pitfalls: why representable views fill all available space

`UIViewRepresentable` is a primitive SwiftUI view type (its `Body` is `Never`). SwiftUI cannot introspect its internals, so it relies on UIKit's sizing signals. **Most UIViews report no intrinsic content size** (returning `UIView.noIntrinsicMetric`), which maps to SwiftUI's `0…0…∞` — the view accepts any proposed size.

Even `UILabel`, which does report intrinsic content size, has a default content hugging priority of **251** (`.defaultLow`). Because the threshold for SwiftUI to clamp the max size is **750** (`.defaultHigh`), the label's max is effectively `∞`, so it expands to fill its container.

The mapping table (documented by the RepresentableKit project) is:

- **No intrinsic size** → min=0, ideal=0, max=∞ (fills everything)
- **Intrinsic size `x`, hugging < 750** → min=0, ideal=x, max=∞ (can still expand)
- **Intrinsic size `x`, hugging ≥ 750** → min=0, ideal=x, max=x (won't expand)
- **Intrinsic size `x`, resistance ≥ 750** → min=x, ideal=x, max=∞ (won't shrink)
- **Both ≥ 750** → min=x, ideal=x, max=x (fixed at intrinsic size)

### Four fixes ranked by modernity

**Fix 1 — `sizeThatFits(_:)` (iOS 16+, recommended).** The protocol method receives a `ProposedViewSize` and returns the desired `CGSize`. This integrates directly with SwiftUI's layout negotiation:

```swift
// ✅ CORRECT: sizeThatFits for width-dependent height
func sizeThatFits(_ proposal: ProposedViewSize,
                  uiView: UITextView, context: Context) -> CGSize? {
    let width = proposal.width ?? UIView.layoutFittingCompressedSize.width
    let size = uiView.sizeThatFits(
        CGSize(width: width, height: .greatestFiniteMagnitude))
    return CGSize(width: width, height: ceil(size.height))
}
```

Handle `nil` dimensions (sent when `.fixedSize()` is applied) by falling back to a compressed fitting size — never force-unwrap `proposal.width`.

**Fix 2 — Content hugging and compression resistance priorities.** Set these in `makeUIView` to control the min/max mapping:

```swift
// ✅ CORRECT: Lock vertical size to intrinsic content
func makeUIView(context: Context) -> UILabel {
    let label = UILabel()
    label.numberOfLines = 0
    label.setContentHuggingPriority(.required, for: .vertical)              // max = intrinsic
    label.setContentCompressionResistancePriority(.required, for: .vertical) // min = intrinsic
    return label
}
```

**Fix 3 — Override `intrinsicContentSize`.** For custom UIView subclasses, override the property and call `invalidateIntrinsicContentSize()` when content changes:

```swift
// ✅ CORRECT: Custom view reporting its size
class SelfSizingTagView: UIView {
    var tags: [String] = [] {
        didSet { invalidateIntrinsicContentSize() }  // Critical!
    }
    override var intrinsicContentSize: CGSize {
        // Calculate based on content
        label.sizeThatFits(CGSize(width: bounds.width, height: .greatestFiniteMagnitude))
    }
}
```

**Fix 4 — `.fixedSize()` modifier.** Replaces the proposed size with `nil`, forcing the view to its ideal size. Only works when the UIView actually reports an intrinsic content size — otherwise the view **collapses to zero**:

```swift
// ✅ CORRECT: fixedSize with a view that has intrinsicContentSize
WrappedLabel(text: "Hello").fixedSize(horizontal: false, vertical: true)

// ❌ WRONG: fixedSize on a view with no intrinsicContentSize — invisible!
WrappedCustomView().fixedSize()  // Collapses to 0×0
```

---

## 5. State bridging across the UIKit–SwiftUI boundary

### @Observable (iOS 17+) enables per-property tracking

The `@Observable` macro delivers dramatically more efficient updates than `ObservableObject`. Only properties **actually read** in a view's `body` trigger re-evaluation — not every `@Published` property on the object.

```swift
// ✅ CORRECT: @Observable with UIHostingController
@Observable class FilterState {
    var selection: String?
    var searchText: String = ""  // Changing this won't update views that only read selection
}

struct FilterView: View {
    @Bindable var state: FilterState  // @Bindable for $ bindings
    var body: some View {
        TextField("Search", text: $state.searchText)
    }
}

// UIKit side:
class FiltersVC: UIViewController {
    private let filterState = FilterState()
    private var hostingController: UIHostingController<FilterView>!

    override func viewDidLoad() {
        super.viewDidLoad()
        let filterView = FilterView(state: filterState)
        hostingController = UIHostingController(rootView: filterView)
        addChild(hostingController)
        view.addSubview(hostingController.view)
        hostingController.didMove(toParent: self)
    }
}
```

```swift
// ❌ WRONG: Mixing @Observable with ObservableObject APIs
@Observable class MyModel { var name = "" }
struct MyView: View {
    @ObservedObject var model: MyModel  // ❌ @ObservedObject is for ObservableObject only
}

// ❌ WRONG: Using @Published inside @Observable
@Observable class MyModel {
    @Published var name = ""  // ❌ Does not work — use plain var
}
```

### UITraitBridgedEnvironmentKey (iOS 17+) bridges traits bidirectionally

This protocol connects a custom UIKit trait to a SwiftUI environment key. Values flow in both directions: set a trait override in UIKit and read it as `@Environment` in SwiftUI, or set `.environment()` in SwiftUI and read it from `traitCollection` in UIKit.

The complete four-step setup:

```swift
// Step 1: Define the UIKit trait
struct AppThemeTrait: UITraitDefinition {
    static let defaultValue: AppTheme = .standard
}
extension UITraitCollection {
    var appTheme: AppTheme { self[AppThemeTrait.self] }
}
extension UIMutableTraits {
    var appTheme: AppTheme {
        get { self[AppThemeTrait.self] }
        set { self[AppThemeTrait.self] = newValue }
    }
}

// Step 2: Define the SwiftUI environment key
struct AppThemeKey: EnvironmentKey {
    static let defaultValue: AppTheme = .standard
}
extension EnvironmentValues {
    var appTheme: AppTheme {
        get { self[AppThemeKey.self] }
        set { self[AppThemeKey.self] = newValue }
    }
}

// Step 3: Bridge them
extension AppThemeKey: UITraitBridgedEnvironmentKey {
    static func read(from traitCollection: UITraitCollection) -> AppTheme {
        traitCollection.appTheme
    }
    static func write(to mutableTraits: inout UIMutableTraits, value: AppTheme) {
        mutableTraits.appTheme = value
    }
}

// Step 4: Usage
// UIKit → SwiftUI: set trait override on scene, window, or VC
windowScene.traitOverrides.appTheme = .monochrome
// SwiftUI reads it automatically:
@Environment(\.appTheme) private var theme
```

### ObservableObject for iOS 13–16 deployment targets

For older targets, use `ObservableObject` with `@Published` properties. The key difference: **every** `@Published` change invalidates **all** observing views, regardless of which property they read.

```swift
class UserSettings: ObservableObject {
    @Published var username: String = ""
    @Published var isLoggedIn: Bool = false
}

struct SettingsView: View {
    @ObservedObject var settings: UserSettings
    var body: some View {
        // Re-evaluates when ANY @Published property changes
        TextField("Username", text: $settings.username)
    }
}

// UIKit side:
let settings = UserSettings()
let hc = UIHostingController(
    rootView: SettingsView(settings: settings)
    // OR: SettingsView2().environmentObject(settings)
)
```

**Migration cheat sheet from `ObservableObject` to `@Observable`:**

- `class Foo: ObservableObject` → `@Observable class Foo`
- `@Published var x` → `var x` (automatic tracking)
- `@StateObject var foo` → `@State var foo`
- `@ObservedObject var foo` → `var foo` (or `@Bindable var foo` for bindings)
- `@EnvironmentObject var foo` → `@Environment(Foo.self) var foo`
- `.environmentObject(foo)` → `.environment(foo)`

---

## 6. Performance: hosting controllers are expensive — reuse them

Each `UIHostingController` instantiation creates a new SwiftUI rendering pipeline with its own AttributeGraph, UIView hierarchy, and environment. Creating one per cell in `cellForRowAt` causes frame drops and memory spikes during scrolling.

```swift
// ❌ WRONG: New hosting controller per cell
func tableView(_ tableView: UITableView,
               cellForRowAt indexPath: IndexPath) -> UITableViewCell {
    let cell = tableView.dequeueReusableCell(withIdentifier: "cell", for: indexPath)
    let hc = UIHostingController(rootView: ItemView(item: items[indexPath.row]))
    addChild(hc)
    cell.contentView.addSubview(hc.view)
    hc.didMove(toParent: self)  // New pipeline per scroll — terrible performance
    return cell
}
```

**The correct pattern reuses one hosting controller per cell** and updates its `rootView`:

```swift
// ✅ CORRECT: Reuse hosting controller, update rootView
class SwiftUICell<Content: View>: UITableViewCell {
    private var hostingController: UIHostingController<Content>?

    func configure(with rootView: Content, parent: UIViewController) {
        if let hc = hostingController {
            hc.rootView = rootView  // Cheap — SwiftUI diffs the view tree
        } else {
            let hc = UIHostingController(rootView: rootView)
            hostingController = hc
            hc.view.backgroundColor = .clear
            parent.addChild(hc)
            hc.view.translatesAutoresizingMaskIntoConstraints = false
            contentView.addSubview(hc.view)
            NSLayoutConstraint.activate([
                hc.view.topAnchor.constraint(equalTo: contentView.topAnchor),
                hc.view.bottomAnchor.constraint(equalTo: contentView.bottomAnchor),
                hc.view.leadingAnchor.constraint(equalTo: contentView.leadingAnchor),
                hc.view.trailingAnchor.constraint(equalTo: contentView.trailingAnchor)
            ])
            hc.didMove(toParent: parent)
        }
    }
}
```

**The best pattern for iOS 16+: `UIHostingConfiguration`.** It hosts SwiftUI content without any view controller overhead — no `addChild`, no containment, no retained controller:

```swift
// ✅ BEST (iOS 16+): UIHostingConfiguration — lightweight, no VC overhead
func tableView(_ tableView: UITableView,
               cellForRowAt indexPath: IndexPath) -> UITableViewCell {
    let cell = tableView.dequeueReusableCell(withIdentifier: "cell", for: indexPath)
    cell.contentConfiguration = UIHostingConfiguration {
        ItemView(item: items[indexPath.row])
    }
    return cell
}
```

### Minimizing SwiftUI body re-evaluations

When `hostingController.rootView = newView` is set, SwiftUI compares the new view struct's stored properties field-by-field. If any property differs, dependent AttributeGraph nodes are marked dirty and `body` is re-evaluated. **Closures cannot be compared** — they always appear "different," forcing re-evaluation every time.

- **Use value-type properties** (String, Int, structs) instead of closures or reference types as view inputs
- **Use `@Observable` over `@ObservedObject`** — property-level tracking means only views reading the changed property update
- **Conform to `Equatable`** for views with closures or reference types, and apply `.equatable()`:

```swift
// ✅ CORRECT: Custom Equatable skips body when data unchanged
struct ItemRow: View, Equatable {
    let item: Item
    let onTap: () -> Void  // Closure — not comparable by default

    static func == (lhs: ItemRow, rhs: ItemRow) -> Bool {
        lhs.item == rhs.item  // Only compare the data
    }

    var body: some View {
        Button(item.name) { onTap() }
    }
}
// Apply: ItemRow(item: item, onTap: handleTap).equatable()
```

- **Extract subviews** to isolate state dependencies — a child view's body only re-evaluates when its own inputs change, not when a sibling's inputs change

---

## What changed at each WWDC since 2022

**iOS 16 (WWDC 2022)** delivered `UIHostingController.sizingOptions`, `UIHostingConfiguration` for cells, `sizeThatFits(_:)` on `UIViewRepresentable`, and `safeAreaRegions` (iOS 16.4).

**iOS 17 (WWDC 2023)** introduced the `@Observable` macro with per-property tracking, `UITraitBridgedEnvironmentKey` for bidirectional trait-environment bridging, custom `UITraitDefinition`, the new trait change registration API replacing `traitCollectionDidChange`, and `viewIsAppearing(_:)` back-deployed to iOS 13.

**iOS 18 (WWDC 2024)** added SwiftUI `Animation` types usable with `UIView.animate`, `UIGestureRecognizerRepresentable` for cross-framework gesture coordination, **automatic trait tracking** in layout/draw methods, and `UIUpdateLink` replacing `CADisplayLink`.

**iOS 26 (WWDC 2025, beta)** is the largest interop leap yet. UIKit now **automatically tracks `@Observable` property access** in `viewWillLayoutSubviews()`, `layoutSubviews()`, `drawRect()`, and the new `updateProperties()` — no manual `setNeedsLayout()` required. The `updateProperties()` lifecycle method runs before layout for non-geometry property updates. `.flushUpdates` in `UIView.animate` eliminates manual `layoutIfNeeded()` calls. `UIHostingSceneDelegate` lets UIKit apps host entire SwiftUI scenes. Observable auto-tracking is back-deployable to iOS 18 via the `UIObservationTrackingEnabled` Info.plist key.

## Conclusion

The UIKit–SwiftUI bridge is no longer a set of workarounds — it's a first-class, bidirectional integration layer. The three most impactful patterns to internalize: **always use proper view controller containment and retain `UIHostingController`** (the single most common source of mysterious bugs); **put all state-dependent configuration in `updateUIView` with equality guards** to prevent both stale UI and infinite loops; and **prefer `UIHostingConfiguration` over manual `UIHostingController` management in cells** for both correctness and performance. For sizing, `sizeThatFits(_:)` (iOS 16+) is the cleanest solution, with content hugging priorities as the fallback for older targets. For state, `@Observable` (iOS 17+) combined with `UITraitBridgedEnvironmentKey` provides fine-grained, bidirectional data flow that makes the boundary between frameworks nearly invisible. With iOS 26's automatic observation tracking in UIKit, the two frameworks are converging toward a unified programming model.