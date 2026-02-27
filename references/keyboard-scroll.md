# UIKit keyboard handling and scroll view management in Swift

**`UIKeyboardLayoutGuide` (iOS 15+) is the definitive modern API for keyboard-aware layouts**, replacing hundreds of lines of notification-based boilerplate with a single Auto Layout constraint. This guide covers seven key areas of UIKit keyboard management — from the modern layout guide through legacy notification fallbacks to the newest iOS 26 strongly typed notification APIs — with production-ready code patterns for each.

The keyboard handling landscape shifted fundamentally at WWDC 2021 with `UIKeyboardLayoutGuide`, matured through iOS 17 with `usesBottomSafeArea` and `keyboardDismissPadding`, and gained developer ergonomics in iOS 26 with strongly typed notification messages. Understanding all layers remains critical: the layout guide for new projects, notifications for backward compatibility, and scroll view inset mechanics for both.

---

## 1. UIKeyboardLayoutGuide eliminates notification boilerplate entirely

`UIKeyboardLayoutGuide` is a subclass of `UITrackingLayoutGuide` available on every `UIView` via the `view.keyboardLayoutGuide` property since **iOS 15.0+**. It represents the space the keyboard occupies and exposes standard Auto Layout anchors — `topAnchor`, `bottomAnchor`, `leadingAnchor`, `trailingAnchor`, `heightAnchor`, `widthAnchor`, and center anchors.

**Three behaviors make it powerful.** First, constraints animate with the keyboard automatically — no `UIView.animate` calls needed. Second, when the keyboard is hidden, the guide's `topAnchor` aligns with `safeAreaLayoutGuide.bottomAnchor`, so content rests at the safe area boundary by default. Third, it tracks all keyboard height changes (emoji keyboard, QuickType bar toggle) without additional code.

### ✅ Correct: pin content to the keyboard layout guide

```swift
override func viewDidLoad() {
    super.viewDidLoad()
    
    // Single constraint replaces all notification-based keyboard handling
    textView.bottomAnchor.constraint(
        equalTo: view.keyboardLayoutGuide.topAnchor
    ).isActive = true
}
```

### ✅ Correct: toolbar pinned above keyboard

```swift
NSLayoutConstraint.activate([
    toolbar.leadingAnchor.constraint(equalTo: view.leadingAnchor),
    toolbar.trailingAnchor.constraint(equalTo: view.trailingAnchor),
    toolbar.bottomAnchor.constraint(equalTo: view.keyboardLayoutGuide.topAnchor)
])
// Toolbar sits at the safe area bottom when keyboard is hidden,
// rises automatically when the keyboard appears — no animation code needed.
```

### ❌ Incorrect: manually animating when using the layout guide

```swift
// DON'T DO THIS — the guide already handles animation
NotificationCenter.default.addObserver(
    forName: UIResponder.keyboardWillShowNotification,
    object: nil, queue: .main
) { _ in
    UIView.animate(withDuration: 0.25) {
        self.view.layoutIfNeeded()  // Unnecessary — guide does this automatically
    }
}
```

### ❌ Incorrect: old notification-based approach on iOS 15+

```swift
// Replaced entirely by one constraint to keyboardLayoutGuide.topAnchor
NotificationCenter.default.addObserver(
    self, selector: #selector(keyboardWillShow),
    name: UIResponder.keyboardWillShowNotification, object: nil
)

@objc func keyboardWillShow(notification: Notification) {
    let info = notification.userInfo
    if let endRect = info?[UIResponder.keyboardFrameEndUserInfoKey] as? CGRect {
        var offset = view.bounds.size.height - endRect.origin.y
        if offset == 0.0 { offset = view.safeAreaInsets.bottom }
        UIView.animate(withDuration: 0.25) {
            self.keyboardHeight.constant = offset
            self.view.layoutIfNeeded()
        }
    }
}
// All of this is unnecessary on iOS 15+ — use keyboardLayoutGuide instead.
```

**Version caveat:** iOS 15.0–15.3 had serious bugs (FB9733654, FB9754794) where the keyboard layout guide frame was not updated correctly during view controller transitions. **Reliable operation requires iOS 15.4+**. For apps supporting iOS 15.0–15.3, fall back to notification-based handling.

---

## 2. iPad floating and split keyboards need adaptive constraints

By default, `UIKeyboardLayoutGuide` ignores undocked, floating, and split keyboards on iPad — when the keyboard undocks, the guide drops to the screen bottom as if the keyboard were dismissed. The `followsUndockedKeyboard` property (available since iOS 15.0, default `false`) changes this behavior.

When set to `true`, the guide tracks the keyboard wherever it moves. However, a single set of fixed constraints cannot handle all positions of a floating keyboard. The `UITrackingLayoutGuide` superclass provides edge-aware constraint activation methods for this.

### ✅ Correct: adaptive constraints for floating keyboard

```swift
view.keyboardLayoutGuide.followsUndockedKeyboard = true

// When keyboard is NOT near the top: pin toolbar above keyboard
let awayFromTop = toolbar.bottomAnchor.constraint(
    equalTo: view.keyboardLayoutGuide.topAnchor
)
view.keyboardLayoutGuide.setConstraints(
    [awayFromTop], activeWhenAwayFrom: .top
)

// When keyboard IS near the top: drop toolbar to safe area bottom
let nearTop = toolbar.bottomAnchor.constraint(
    equalTo: view.safeAreaLayoutGuide.bottomAnchor
)
view.keyboardLayoutGuide.setConstraints(
    [nearTop], activeWhenNearEdge: .top
)

// Center toolbar horizontally when keyboard is away from edges
let centered = toolbar.centerXAnchor.constraint(
    equalTo: view.keyboardLayoutGuide.centerXAnchor
)
view.keyboardLayoutGuide.setConstraints(
    [centered], activeWhenAwayFrom: [.leading, .trailing]
)
```

### ❌ Incorrect: fixed constraints with floating keyboard enabled

```swift
view.keyboardLayoutGuide.followsUndockedKeyboard = true

// These fixed constraints break when the floating keyboard is at the top
// (toolbar goes offscreen) or near an edge (misaligned)
toolbar.bottomAnchor.constraint(
    equalTo: view.keyboardLayoutGuide.topAnchor
).isActive = true
toolbar.centerXAnchor.constraint(
    equalTo: view.keyboardLayoutGuide.centerXAnchor
).isActive = true
// ❌ Will cause layout issues in many keyboard positions!
```

### iOS 17 additions: `usesBottomSafeArea` and `keyboardDismissPadding`

Two properties added in **iOS 17.0** refine the guide's behavior:

- **`usesBottomSafeArea`** (default `true`): when `false`, the guide extends to the screen's physical bottom edge when the keyboard is hidden, instead of stopping at the safe area. Useful for edge-to-edge content that should fill the home indicator region.
- **`keyboardDismissPadding`** (default `0`): adds padding above the keyboard for `UIScrollView.keyboardDismissMode = .interactive`, extending the interactive dismiss gesture zone upward.

```swift
// Extend guide to physical bottom (ignore safe area when keyboard hidden)
view.keyboardLayoutGuide.usesBottomSafeArea = false

// Add 60pt of interactive dismiss zone above the keyboard
view.keyboardLayoutGuide.keyboardDismissPadding = 60
```

| Property | iOS Version | Default |
|---|---|---|
| `keyboardLayoutGuide` | 15.0+ | N/A |
| `followsUndockedKeyboard` | 15.0+ | `false` |
| `usesBottomSafeArea` | 17.0+ | `true` |
| `keyboardDismissPadding` | 17.0+ | `0` |

---

## 3. Scroll views and the keyboard layout guide work through frame compression

When you constrain a `UIScrollView`'s bottom to `view.keyboardLayoutGuide.topAnchor`, **the scroll view's frame shrinks** as the keyboard appears. This is fundamentally different from the old notification approach of adjusting `contentInset` — the constraint compresses the frame via Auto Layout, the system's `contentInsetAdjustmentBehavior` continues handling safe area insets normally, and animations happen automatically.

### ✅ Correct: scroll view pinned to keyboard layout guide

```swift
override func viewDidLoad() {
    super.viewDidLoad()
    
    scrollView.translatesAutoresizingMaskIntoConstraints = false
    NSLayoutConstraint.activate([
        scrollView.topAnchor.constraint(equalTo: view.topAnchor),
        scrollView.leadingAnchor.constraint(equalTo: view.leadingAnchor),
        scrollView.trailingAnchor.constraint(equalTo: view.trailingAnchor),
        scrollView.bottomAnchor.constraint(
            equalTo: view.keyboardLayoutGuide.topAnchor
        )
    ])
    
    scrollView.keyboardDismissMode = .interactive
    // That's it. No notifications, no contentInset manipulation.
}
```

### ✅ Correct: input bar between scroll view and keyboard

```swift
NSLayoutConstraint.activate([
    scrollView.topAnchor.constraint(equalTo: view.safeAreaLayoutGuide.topAnchor),
    scrollView.leadingAnchor.constraint(equalTo: view.leadingAnchor),
    scrollView.trailingAnchor.constraint(equalTo: view.trailingAnchor),
    scrollView.bottomAnchor.constraint(equalTo: inputBar.topAnchor),
    
    inputBar.leadingAnchor.constraint(equalTo: view.leadingAnchor),
    inputBar.trailingAnchor.constraint(equalTo: view.trailingAnchor),
    inputBar.bottomAnchor.constraint(
        equalTo: view.keyboardLayoutGuide.topAnchor
    )
])
```

### ❌ Incorrect: double-insetting by mixing approaches

```swift
// Constraint shrinks the frame...
scrollView.bottomAnchor.constraint(
    equalTo: view.keyboardLayoutGuide.topAnchor
).isActive = true

// ...AND notification handler also adjusts contentInset — DOUBLE INSET
NotificationCenter.default.addObserver(
    forName: UIResponder.keyboardWillShowNotification,
    object: nil, queue: .main
) { notification in
    let keyboardHeight = /* ... */
    self.scrollView.contentInset.bottom = keyboardHeight  // ❌ Content shifts up TWICE
    self.scrollView.verticalScrollIndicatorInsets.bottom = keyboardHeight
}
```

### ❌ Incorrect: combining with third-party keyboard libraries

```swift
// ❌ Using IQKeyboardManager or similar alongside keyboardLayoutGuide
// They will fight each other — disable the library when using the guide
scrollView.bottomAnchor.constraint(
    equalTo: view.keyboardLayoutGuide.topAnchor
).isActive = true
// IQKeyboardManager is still active → conflicts and visual glitches
```

**Important limitation:** the keyboard layout guide does **not** automatically scroll to make a focused text field visible. You still need `scrollRectToVisible(_:animated:)` or equivalent logic triggered by `UITextFieldDelegate`/`UITextViewDelegate` methods.

**Stage Manager note (iOS 16+):** Keyboard notifications can be unreliable with Stage Manager and the out-of-process keyboard architecture. The keyboard layout guide is Apple's **recommended approach** as it works reliably across all multitasking configurations.

---

## 4. Pre-iOS 15 notification handling requires precise coordinate math

For apps targeting iOS 14 and earlier, or as a fallback for iOS 15.0–15.3 bugs, notification-based handling remains necessary. Three details determine whether the implementation works correctly: **coordinate conversion**, **safe area subtraction**, and **animation curve matching**.

The keyboard frame from `keyboardFrameEndUserInfoKey` is in **screen coordinate space**. You must convert it to your view's coordinates — failure to do so breaks behavior in Split View, Slide Over, Stage Manager, and landscape orientation. On iOS 16.1+, the `notification.object` is the `UIScreen`, enabling screen-based conversion. On earlier versions, use `view.convert(_:from: view.window)`.

The converted keyboard height **includes the home indicator region** (~34pt on Face ID devices). Since `UIScrollView` with default `contentInsetAdjustmentBehavior` (`.automatic`) already adds `safeAreaInsets.bottom` to `adjustedContentInset`, you must subtract `view.safeAreaInsets.bottom` to avoid double-insetting.

The keyboard uses an **undocumented animation curve value of 7**, which doesn't match any public `UIView.AnimationCurve` case (0–3). The `UIView.AnimationOptions` raw value encodes the curve in bits 16–19, so **bit-shifting left by 16** converts the raw integer to the correct option.

### ✅ Correct: complete notification-based implementation

```swift
final class KeyboardAwareViewController: UIViewController {
    
    @IBOutlet private weak var scrollView: UIScrollView!
    
    override func viewWillAppear(_ animated: Bool) {
        super.viewWillAppear(animated)
        NotificationCenter.default.addObserver(
            self, selector: #selector(keyboardWillShow),
            name: UIResponder.keyboardWillShowNotification, object: nil
        )
        NotificationCenter.default.addObserver(
            self, selector: #selector(keyboardWillHide),
            name: UIResponder.keyboardWillHideNotification, object: nil
        )
    }
    
    override func viewWillDisappear(_ animated: Bool) {
        super.viewWillDisappear(animated)
        NotificationCenter.default.removeObserver(
            self, name: UIResponder.keyboardWillShowNotification, object: nil
        )
        NotificationCenter.default.removeObserver(
            self, name: UIResponder.keyboardWillHideNotification, object: nil
        )
    }
    
    @objc private func keyboardWillShow(_ notification: Notification) {
        guard let userInfo = notification.userInfo,
              let frameValue = userInfo[UIResponder.keyboardFrameEndUserInfoKey] as? NSValue,
              let duration = userInfo[UIResponder.keyboardAnimationDurationUserInfoKey] as? Double,
              let curveRaw = userInfo[UIResponder.keyboardAnimationCurveUserInfoKey] as? UInt
        else { return }
        
        // 1. Convert from screen coordinates to view coordinates
        let keyboardViewFrame = view.convert(frameValue.cgRectValue, from: view.window)
        
        // 2. Subtract safe area bottom to avoid double-insetting
        let bottomInset = keyboardViewFrame.height - view.safeAreaInsets.bottom
        
        // 3. Bit-shift curve value by 16 for AnimationOptions
        let options = UIView.AnimationOptions(rawValue: curveRaw << 16)
        
        UIView.animate(withDuration: duration, delay: 0, options: options) {
            self.scrollView.contentInset.bottom = bottomInset
            self.scrollView.verticalScrollIndicatorInsets.bottom = bottomInset
        }
    }
    
    @objc private func keyboardWillHide(_ notification: Notification) {
        guard let userInfo = notification.userInfo,
              let duration = userInfo[UIResponder.keyboardAnimationDurationUserInfoKey] as? Double,
              let curveRaw = userInfo[UIResponder.keyboardAnimationCurveUserInfoKey] as? UInt
        else { return }
        
        let options = UIView.AnimationOptions(rawValue: curveRaw << 16)
        
        UIView.animate(withDuration: duration, delay: 0, options: options) {
            self.scrollView.contentInset.bottom = 0
            self.scrollView.verticalScrollIndicatorInsets.bottom = 0
        }
    }
}
```

### ✅ Correct: block-based observers with proper token cleanup

```swift
final class KeyboardObservingVC: UIViewController {
    private var showToken: NSObjectProtocol?
    private var hideToken: NSObjectProtocol?
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        showToken = NotificationCenter.default.addObserver(
            forName: UIResponder.keyboardWillShowNotification,
            object: nil, queue: .main
        ) { [weak self] notification in
            self?.handleKeyboardShow(notification)
        }
        
        hideToken = NotificationCenter.default.addObserver(
            forName: UIResponder.keyboardWillHideNotification,
            object: nil, queue: .main
        ) { [weak self] notification in
            self?.handleKeyboardHide(notification)
        }
    }
    
    deinit {
        // Block-based observers MUST be explicitly removed
        if let token = showToken { NotificationCenter.default.removeObserver(token) }
        if let token = hideToken { NotificationCenter.default.removeObserver(token) }
    }
}
```

### ❌ Incorrect: using screen coordinates directly

```swift
@objc func keyboardWillShow(_ notification: Notification) {
    let frame = (notification.userInfo![UIResponder.keyboardFrameEndUserInfoKey] as! NSValue).cgRectValue
    // ❌ Screen coordinates used directly — breaks in Split View, landscape, Stage Manager
    scrollView.contentInset.bottom = frame.height
}
```

### ❌ Incorrect: not subtracting safe area insets

```swift
let converted = view.convert(kbFrame, from: view.window)
// ❌ Full keyboard height includes ~34pt home indicator already in adjustedContentInset
scrollView.contentInset.bottom = converted.height  // Double-insets on Face ID devices!
```

### ❌ Incorrect: hardcoded animation parameters

```swift
// ❌ Hardcoded 0.25s and easeInOut does NOT match the keyboard's actual curve
UIView.animate(withDuration: 0.25, delay: 0, options: .curveEaseInOut) {
    self.scrollView.contentInset.bottom = keyboardHeight
}
```

### ❌ Incorrect: leaking block-based observer tokens

```swift
// ❌ Token is discarded — observer fires forever and captures self strongly
NotificationCenter.default.addObserver(
    forName: UIResponder.keyboardWillShowNotification,
    object: nil, queue: .main
) { notification in
    self.handleKeyboard(notification)  // ❌ Strong capture of self
}
```

---

## 5. `adjustedContentInset` vs `contentInset` is the critical distinction

Since iOS 11, Apple split scroll view insets into two properties. **`contentInset`** (read-write) is what you set manually. **`adjustedContentInset`** (read-only) is the effective total, computed as `contentInset` plus the system's safe area contribution based on `contentInsetAdjustmentBehavior`.

| Behavior | Effect |
|---|---|
| `.automatic` (default) | Adds safe area insets along scrollable axes; special handling inside navigation controllers |
| `.scrollableAxes` | Adds safe area only where content is scrollable or `alwaysBounces` is `true` |
| `.always` | Adds safe area on all edges unconditionally |
| `.never` | No automatic adjustment — `adjustedContentInset == contentInset` |

### ✅ Correct: read `adjustedContentInset` for effective insets

```swift
// contentInset only reflects what YOU set — not what the system added
func visibleContentHeight() -> CGFloat {
    let adjusted = scrollView.adjustedContentInset
    return scrollView.bounds.height - adjusted.top - adjusted.bottom
}
```

### ✅ Correct: respond to adjusted inset changes

```swift
class CustomScrollView: UIScrollView {
    override func adjustedContentInsetDidChange() {
        super.adjustedContentInsetDidChange()
        // React to effective inset changes (safe area rotation, etc.)
        invalidateLayout()
    }
}
```

### ✅ Correct: `.never` for full-screen edge-to-edge content

```swift
// Image viewers, maps, or full-bleed media where YOU handle safe areas
imageScrollView.contentInsetAdjustmentBehavior = .never
// Now adjustedContentInset == contentInset at all times
// WARNING: safe area insets propagate to subviews instead of being consumed
```

### ❌ Incorrect: reading `contentInset` when you mean `adjustedContentInset`

```swift
func calculateVisibleHeight() -> CGFloat {
    let inset = scrollView.contentInset  // ❌ Only your manual inset, not the effective total
    return scrollView.bounds.height - inset.top - inset.bottom
    // On notched iPhone, this ignores the 34pt home indicator safe area
}
```

### ❌ Incorrect: setting `.never` without understanding side effects

```swift
// ❌ Using .never as a quick fix without handling consequences
scrollView.contentInsetAdjustmentBehavior = .never
// Content now goes under navigation bars and home indicator
// Safe area insets propagate to subviews — unexpected child layout changes
```

### ❌ Incorrect: resetting all insets on keyboard hide

```swift
@objc func keyboardWillHide(_ notification: Notification) {
    scrollView.contentInset = .zero  // ❌ Wipes out ALL insets, including custom top/left/right
}

// ✅ Fix: only reset the bottom
@objc func keyboardWillHideCorrectly(_ notification: Notification) {
    scrollView.contentInset.bottom = 0
}
```

The interaction with keyboard handling is direct: when you set `contentInset.bottom = keyboardHeight` and `contentInsetAdjustmentBehavior` is `.automatic`, the system adds `safeAreaInsets.bottom` on top — producing the double-inset bug. Either subtract `safeAreaInsets.bottom` from the keyboard height, or use `UIKeyboardLayoutGuide` which sidesteps the issue entirely through frame compression rather than inset manipulation.

---

## 6. Input accessory views attach toolbars directly to the keyboard

`inputAccessoryView` is a property on `UIResponder` (since iOS 3.2) that UIKit reads when a responder becomes first responder. **`UITextField` and `UITextView` redeclare it as read-write**, so you assign directly. `UIViewController` exposes it as read-only, requiring an override plus `canBecomeFirstResponder` returning `true`.

### ✅ Correct: UIToolbar as inputAccessoryView on a text field

```swift
private func createToolbar() -> UIToolbar {
    let toolbar = UIToolbar(
        frame: CGRect(x: 0, y: 0, width: UIScreen.main.bounds.width, height: 44)
    )
    toolbar.sizeToFit()
    
    let spacer = UIBarButtonItem(
        barButtonSystemItem: .flexibleSpace, target: nil, action: nil
    )
    let done = UIBarButtonItem(
        barButtonSystemItem: .done, target: self, action: #selector(dismissKeyboard)
    )
    toolbar.items = [spacer, done]
    return toolbar
}

override func viewDidLoad() {
    super.viewDidLoad()
    amountTextField.inputAccessoryView = createToolbar()
}
```

### ✅ Correct: Messages-style compose bar on UIViewController

```swift
class ChatViewController: UIViewController {
    
    private lazy var composeBar: ComposeBarView = {
        let bar = ComposeBarView()
        bar.autoresizingMask = .flexibleHeight  // Required for dynamic height
        return bar
    }()
    
    override var inputAccessoryView: UIView? { composeBar }
    override var canBecomeFirstResponder: Bool { true }
    
    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)
        becomeFirstResponder()
    }
}
```

### ✅ Correct: dynamic-height accessory with intrinsicContentSize

```swift
class GrowingInputBar: UIView {
    
    let textView = UITextView()
    
    override init(frame: CGRect) {
        super.init(frame: frame)
        autoresizingMask = .flexibleHeight  // CRITICAL for dynamic sizing
        
        textView.translatesAutoresizingMaskIntoConstraints = false
        textView.isScrollEnabled = false  // Allows growth
        textView.font = .systemFont(ofSize: 16)
        addSubview(textView)
        
        NSLayoutConstraint.activate([
            textView.leadingAnchor.constraint(equalTo: leadingAnchor, constant: 8),
            textView.trailingAnchor.constraint(equalTo: trailingAnchor, constant: -8),
            textView.topAnchor.constraint(equalTo: topAnchor, constant: 8),
            textView.bottomAnchor.constraint(equalTo: bottomAnchor, constant: -8),
        ])
    }
    
    required init?(coder: NSCoder) { fatalError() }
    
    // CRITICAL: must override for dynamic sizing
    override var intrinsicContentSize: CGSize {
        let textSize = textView.sizeThatFits(
            CGSize(width: textView.bounds.width, height: .greatestFiniteMagnitude)
        )
        let height = min(textSize.height + 16, 120)
        return CGSize(width: UIView.noIntrinsicMetric, height: height)
    }
}

// Trigger resize when text changes:
func textViewDidChange(_ textView: UITextView) {
    inputBar.invalidateIntrinsicContentSize()
}
```

### ❌ Incorrect: no frame on the toolbar

```swift
let toolbar = UIToolbar()  // ❌ Frame is .zero — toolbar will be invisible
toolbar.items = [doneButton]
textField.inputAccessoryView = toolbar
```

### ❌ Incorrect: creating a new instance every access

```swift
override var inputAccessoryView: UIView? {
    return UIToolbar(frame: ...)  // ❌ Creates a NEW toolbar on every property access
}

// ✅ Fix: use a lazy var
private lazy var toolbar: UIToolbar = { /* configure once */ }()
override var inputAccessoryView: UIView? { toolbar }
```

### ❌ Incorrect: forgetting canBecomeFirstResponder on UIViewController

```swift
class BrokenVC: UIViewController {
    override var inputAccessoryView: UIView? { someToolbar }
    // ❌ Missing canBecomeFirstResponder — accessory will never appear
}
```

### ❌ Incorrect: memory leak from the well-known retain cycle

```swift
class LeakyVC: UIViewController {
    private lazy var bar: UIView = { UIView(frame: .init(x: 0, y: 0, width: 320, height: 44)) }()
    override var inputAccessoryView: UIView? { bar }
    override var canBecomeFirstResponder: Bool { true }
    // ❌ VC retains bar, system retains VC via responder chain → deinit NEVER called
}

// ✅ Workaround: nil out in viewDidDisappear and resign first responder
override func viewDidDisappear(_ animated: Bool) {
    super.viewDidDisappear(animated)
    resignFirstResponder()
}
```

**`inputAccessoryViewController`** (iOS 8+) provides a view-controller-based alternative. It takes precedence over `inputAccessoryView` if both are set, provides full lifecycle methods (`viewDidLoad`, etc.), and its `UIInputView` is automatically styled to match the keyboard. Use it for complex accessory bars that benefit from containment, or as a workaround for the `inputAccessoryView` retain cycle.

**Design choice:** use `inputAccessoryView` for persistent bars that stay visible when the keyboard hides (Messages-style), and `keyboardLayoutGuide` for adjusting existing layout elements. Avoid combining both on the same view controller.

---

## 7. iOS 26 brings strongly typed keyboard notifications

WWDC 2025 (June 9, 2025) introduced iOS 26 with one major keyboard API improvement: **strongly typed notification messages** that eliminate the fragile `userInfo` dictionary casting pattern.

### The new `NotificationCenter.Message` API

UIKit in iOS 26 represents each keyboard notification as a dedicated `NotificationCenter.Message` type. Properties like `animationDuration` and `endFrame` are available directly on the message object.

### ✅ Correct: iOS 26 strongly typed keyboard notifications

```swift
// iOS 26+: Type-safe keyboard notification handling
NotificationCenter.default.addObserver(
    forMessage: UIResponder.keyboardWillShowMessage
) { message in
    let duration = message.animationDuration   // Double, no casting
    let frame = message.endFrame               // CGRect, no casting
    
    UIView.animate(withDuration: duration) {
        self.bottomConstraint.constant = frame.height
        self.view.layoutIfNeeded()
    }
}
```

### ❌ Incorrect: old userInfo dictionary parsing (still works but now legacy)

```swift
// Pre-iOS 26: fragile dictionary lookups with manual casting
NotificationCenter.default.addObserver(
    forName: UIResponder.keyboardWillShowNotification,
    object: nil, queue: .main
) { notification in
    guard let userInfo = notification.userInfo,
          let frame = userInfo[UIResponder.keyboardFrameEndUserInfoKey] as? CGRect,
          let duration = userInfo[UIResponder.keyboardAnimationDurationUserInfoKey] as? Double
    else { return }  // ❌ Verbose, error-prone, fails silently on type mismatch
}
```

### What did NOT change in iOS 26

**`UIKeyboardLayoutGuide` received no new APIs** in iOS 26. The properties introduced through iOS 17 (`usesBottomSafeArea`, `keyboardDismissPadding`, `followsUndockedKeyboard`) remain the latest additions. The `inputAccessoryView` API is also unchanged, including the long-standing memory leak when used on UIViewController.

Two other iOS 26 features are tangentially relevant to keyboard handling. **`updateProperties()`** is a new `UIView`/`UIViewController` method that runs before `layoutSubviews()` and automatically tracks Observable objects — useful for keyboard-responsive layout updates. The **`flushUpdates`** animation option simplifies animating constraint changes, which can clean up keyboard animation code in notification handlers.

The **Liquid Glass** visual redesign in iOS 26 gives the keyboard translucent, glass-like key caps. This is automatic for apps compiled against the iOS 26 SDK and requires no code changes, but custom `inputAccessoryView` styling may need visual updates to match the new aesthetic.

---

## Conclusion

The decision tree for keyboard handling in 2024–2026 is straightforward. **For iOS 15.4+ targets, use `UIKeyboardLayoutGuide`** — pin your content or scroll view bottom to `view.keyboardLayoutGuide.topAnchor` and let Auto Layout handle animation, safe area tracking, and keyboard height changes. For iPad floating keyboards, enable `followsUndockedKeyboard` with edge-aware adaptive constraints. For scroll views, constrain the frame to the guide and avoid touching `contentInset` for keyboard purposes.

When supporting older versions, the notification-based approach requires three non-negotiable steps: convert the keyboard frame from screen to view coordinates, subtract `view.safeAreaInsets.bottom` from the height, and bit-shift the animation curve by 16 to create `UIView.AnimationOptions`. Always read `adjustedContentInset` (not `contentInset`) when calculating visible content area. For input bars, set `autoresizingMask = .flexibleHeight`, override `intrinsicContentSize`, and clean up in `viewDidDisappear` to avoid the `inputAccessoryView` retain cycle.

On iOS 26, the new strongly typed `NotificationCenter.Message` APIs eliminate the most error-prone part of the legacy approach — though if you're targeting iOS 26, you should already be using `UIKeyboardLayoutGuide` instead of notifications entirely.