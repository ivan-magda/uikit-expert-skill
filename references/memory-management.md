# UIKit memory management: the definitive Swift guide to retain cycles

**Every non-trivial iOS app leaks memory — usually through the same handful of patterns.** The retain cycle remains Swift's most common memory bug in UIKit codebases, and it hides in closures, timers, delegates, and now Swift concurrency's `Task`. This guide covers every major trap with compilable code showing both the broken and fixed versions, incorporating the latest guidance through Swift 6.2, WWDC 2024's "Analyze heap memory" session, and WWDC 2025's concurrency overhaul. The core rule is simple: **understand who owns whom, and break every bidirectional strong reference**.

---

## 1. Default to `[weak self]` — reserve `unowned` for provable lifetimes

The Swift community consensus for 2024–2026 is unambiguous: **use `[weak self]` by default** in any `@escaping` closure where the closure might outlive `self`. The performance difference between `weak` and `unowned` is negligible — a few nanoseconds of optional wrapping that only matters in tight loops creating millions of references. The safety difference is enormous: `unowned` crashes your app instantly if the referenced object has been deallocated, while `weak` gracefully becomes `nil`.

### When `[weak self]` is required

Use it in network callbacks, timers, animation completions, Combine subscriptions, notification observers, stored closures — any `@escaping` closure that another object retains. Since Swift 5.8 (**SE-0365**), you get a major quality-of-life improvement: after `guard let self` or `if let self`, implicit `self` is allowed, eliminating the old `self.` boilerplate:

```swift
// ✅ Modern pattern (Swift 5.8+) — implicit self after unwrap
timer = Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true) { [weak self] _ in
    guard let self else { return }
    fireCount += 1                    // No self. prefix needed
    print("Fired \(fireCount) times") // Clean, readable code
}
```

### When you do NOT need `[weak self]`

**Non-escaping closures cannot create retain cycles** — the closure executes immediately and is never stored. This includes `map`, `filter`, `forEach`, `compactMap`, and `UIView.animate`:

```swift
// ✅ No [weak self] needed — UIView.animate is non-escaping
UIView.animate(withDuration: 0.3) {
    self.view.alpha = 0  // Perfectly safe
}

// ✅ No [weak self] needed — map is non-escaping
let names = items.map { self.format($0) }
```

### When `[unowned self]` is genuinely safe

Reserve `unowned` for two narrow scenarios where lifetime is provably guaranteed:

**Stored `lazy var` closures (not immediately applied):** The closure is a property of the instance. The instance must exist to access it, so `self` is guaranteed alive:

```swift
// ✅ Safe: lazy var stored closure — self always outlives the closure
class HTMLElement {
    let name: String
    let text: String?

    lazy var asHTML: () -> String = { [unowned self] in
        if let text { return "<\(name)>\(text)</\(name)>" }
        return "<\(name) />"
    }

    init(name: String, text: String? = nil) {
        self.name = name
        self.text = text
    }
}
```

**Critical distinction:** immediately-applied lazy closures (`lazy var label: UILabel = { ... }()` with the trailing `()`) are `@noescape` and execute at first access. They create no retain cycle and need neither `[weak self]` nor `[unowned self]`.

**Parent-child relationships with guaranteed lifetimes:** When a child object structurally cannot outlive its parent, `unowned` expresses this correctly:

```swift
// ✅ Safe: unowned is correct when parent structurally outlives child
class Parent {
    var child: Child?
    init() { child = Child(parent: self) }
}

class Child {
    unowned let parent: Parent  // Parent always outlives Child
    init(parent: Parent) { self.parent = parent }
}
```

---

## 2. Four classic retain cycle traps every UIKit developer must know

### Trap A: Timer's target-selector retains you forever

`Timer.scheduledTimer(timeInterval:target:selector:userInfo:repeats:)` **strongly retains its target**. The RunLoop also retains the Timer independently. This means `deinit` will never fire to invalidate it — a deadlock:

```swift
// ❌ RETAIN CYCLE: ViewController → timer → ViewController (target)
class TimerViewController: UIViewController {
    private var timer: Timer?

    override func viewDidLoad() {
        super.viewDidLoad()
        timer = Timer.scheduledTimer(
            timeInterval: 1.0, target: self,
            selector: #selector(timerFired),
            userInfo: nil, repeats: true
        )
    }

    @objc private func timerFired() { print("Tick") }

    deinit {
        timer?.invalidate() // ❌ NEVER CALLED — Timer retains self
        print("Deallocated")  // Never printed
    }
}
```

The fix uses the block-based API (iOS 10+) with `[weak self]` and invalidates in a lifecycle method that is guaranteed to run:

```swift
// ✅ Block-based API + [weak self] + invalidate in viewWillDisappear
class TimerViewController: UIViewController {
    private var timer: Timer?
    private var count = 0

    override func viewDidLoad() {
        super.viewDidLoad()
        timer = Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true) { [weak self] _ in
            guard let self else { return }
            count += 1
            print("Tick \(count)")
        }
    }

    override func viewWillDisappear(_ animated: Bool) {
        super.viewWillDisappear(animated)
        timer?.invalidate()  // ✅ Guaranteed to be called
        timer = nil
    }

    deinit {
        timer?.invalidate()  // Belt-and-suspenders safety net
        print("TimerViewController deallocated ✅")
    }
}
```

### Trap B: NotificationCenter's closure API holds your closure hostage

`NotificationCenter.addObserver(forName:object:queue:using:)` returns a token that the NotificationCenter retains. If the closure captures `self` strongly and the token is stored on `self`, a cycle forms:

```swift
// ❌ Token → closure → self, self → token = CYCLE
class NotificationVC: UIViewController {
    private var observer: NSObjectProtocol?

    override func viewDidLoad() {
        super.viewDidLoad()
        observer = NotificationCenter.default.addObserver(
            forName: UIApplication.didBecomeActiveNotification,
            object: nil, queue: .main
        ) { notification in
            self.handleActivation()  // ❌ Strong capture
        }
    }

    deinit { /* Never called */ }
}
```

```swift
// ✅ [weak self] breaks the closure's hold on self
class NotificationVC: UIViewController {
    private var observer: NSObjectProtocol?

    override func viewDidLoad() {
        super.viewDidLoad()
        observer = NotificationCenter.default.addObserver(
            forName: UIApplication.didBecomeActiveNotification,
            object: nil, queue: .main
        ) { [weak self] _ in
            guard let self else { return }
            handleActivation()
        }
    }

    deinit {
        if let observer { NotificationCenter.default.removeObserver(observer) }
        print("NotificationVC deallocated ✅")
    }
}
```

The modern alternative is **Combine** (iOS 13+), where `AnyCancellable` auto-cancels on deallocation:

```swift
// ✅ Combine alternative — automatic cleanup
import Combine

class NotificationVC: UIViewController {
    private var cancellables = Set<AnyCancellable>()

    override func viewDidLoad() {
        super.viewDidLoad()
        NotificationCenter.default
            .publisher(for: UIApplication.didBecomeActiveNotification)
            .sink { [weak self] _ in self?.handleActivation() }
            .store(in: &cancellables)
    }
}
```

### Trap C: CADisplayLink has no block API — you need a weak proxy

Unlike Timer, **`CADisplayLink` offers no closure-based initializer** even in iOS 18. It always strongly retains its target through `CADisplayLink(target:selector:)`. The only solution is a **weak proxy object** that sits between the display link and your view controller:

```swift
// ❌ RETAIN CYCLE: ViewController → displayLink → ViewController (target)
class AnimationVC: UIViewController {
    private var displayLink: CADisplayLink?

    override func viewDidLoad() {
        super.viewDidLoad()
        displayLink = CADisplayLink(target: self, selector: #selector(handleFrame))
        displayLink?.add(to: .main, forMode: .common)
    }

    deinit { displayLink?.invalidate() } // ❌ Never called
}
```

The fix introduces a `WeakProxy` that holds only a `weak` reference to the real target:

```swift
// ✅ WeakProxy pattern — the only solution for CADisplayLink
final class WeakProxy: NSObject {
    private weak var target: AnyObject?
    private let action: Selector

    init(target: AnyObject, action: Selector) {
        self.target = target
        self.action = action
        super.init()
    }

    @objc func proxyCallback(_ sender: CADisplayLink) {
        if let target {
            _ = target.perform(action, with: sender)
        } else {
            sender.invalidate()  // Auto-cleanup when target is gone
        }
    }
}

class AnimationVC: UIViewController {
    private var displayLink: CADisplayLink?

    override func viewDidLoad() {
        super.viewDidLoad()
        let proxy = WeakProxy(target: self, action: #selector(handleFrame(_:)))
        displayLink = CADisplayLink(
            target: proxy,
            selector: #selector(WeakProxy.proxyCallback(_:))
        )
        displayLink?.add(to: .main, forMode: .common)
    }

    @objc private func handleFrame(_ link: CADisplayLink) {
        print("Frame at \(link.timestamp)")
    }

    override func viewWillDisappear(_ animated: Bool) {
        super.viewWillDisappear(animated)
        displayLink?.invalidate()
        displayLink = nil
    }

    deinit {
        displayLink?.invalidate()
        print("AnimationVC deallocated ✅")
    }
}
```

The memory graph becomes: `RunLoop → CADisplayLink → WeakProxy --weak→ ViewController`. When the VC deallocates, the proxy's target becomes `nil` and it auto-invalidates the display link. An even more generic alternative uses `forwardingTarget(for:)` to transparently forward any selector — this is the pattern Facebook's React Native uses in production.

### Trap D: Nested closures silently re-capture strong self

This is the subtlest trap. An outer closure correctly uses `[weak self]` and `guard let self`. But `guard let self` creates a **strong local reference**. Any inner stored closure captures this strong `self` by default:

```swift
// ❌ Inner stored closure captures the strong self from guard let
class NestedVC: UIViewController {
    var onSave: (() -> Void)?

    func startObserving() {
        someService.observe { [weak self] data in
            guard let self else { return }

            // BUG: This stored closure captures the strong `self`
            self.onSave = {
                self.saveData()  // ← Strong self from guard let = RETAIN CYCLE
            }
        }
    }

    deinit { print("Deallocated") } // ❌ Never called
}
```

```swift
// ✅ Re-capture [weak self] in every inner stored/escaping closure
class NestedVC: UIViewController {
    var onSave: (() -> Void)?

    func startObserving() {
        someService.observe { [weak self] data in
            guard let self else { return }

            self.onSave = { [weak self] in  // ✅ Re-weaken
                guard let self else { return }
                self.saveData()
            }
        }
    }

    deinit { print("NestedVC deallocated ✅") }
}
```

**The rule:** non-escaping inner closures (like `UIView.animate` or `DispatchQueue.main.async`) are safe without re-capturing. But any inner closure **stored as a property on self** must re-capture `[weak self]`. Swift 5.8's SE-0365 actually helps here — the compiler requires explicit `self` in nested closures within a `[weak self]` outer closure, serving as a built-in safety net.

---

## 3. Delegate ownership: `weak var` on an `AnyObject`-constrained protocol

The delegate pattern is UIKit's backbone, and its retain cycle is the most textbook example in iOS development. A parent creates a child, the child's delegate points back to the parent. Without `weak`, neither can deallocate.

The protocol **must** be constrained to `AnyObject` because `weak` only applies to reference types. Without the constraint, the compiler rejects `weak var delegate`:

```
'weak' must not be applied to non-class-bound 'any MyDelegate';
consider adding a protocol conformance that has a class bound
```

```swift
// ❌ WRONG: Strong delegate creates retain cycle
protocol FormDelegate {  // Missing AnyObject constraint
    func didSave(data: [String: Any])
}

class FormView: UIView {
    var delegate: FormDelegate?  // ❌ Strong — can't even add weak without AnyObject
}
```

```swift
// ✅ CORRECT: AnyObject constraint + weak var
protocol FormDelegate: AnyObject {
    func didSave(data: [String: Any])
}

class FormView: UIView {
    weak var delegate: FormDelegate?  // ✅ Weak reference, no cycle

    func userTappedSave() {
        delegate?.didSave(data: ["name": "John"])
    }
}

class ParentVC: UIViewController, FormDelegate {
    let formView = FormView()

    override func viewDidLoad() {
        super.viewDidLoad()
        formView.delegate = self  // Safe: formView holds WEAK ref to self
    }

    func didSave(data: [String: Any]) { print("Saved") }
    deinit { print("ParentVC deallocated ✅") }
}
```

Even when a retain cycle isn't structurally guaranteed, `weak` is correct for **ownership semantics** — a child object should never claim ownership of its delegate.

---

## 4. Closure capture lists: value types snapshot, reference types share

Swift closures capture **a reference to the variable** by default — not a copy, even for value types like `Int` or `String`. This means the closure sees mutations that happen after its creation:

```swift
var counter = 0
let closure = { print(counter) }
counter = 42
closure()  // Prints 42, NOT 0
```

A **capture list** changes this to a snapshot — a constant copy frozen at the moment the closure is created:

```swift
var counter = 0
let closure = { [counter] in print(counter) }
counter = 42
closure()  // Prints 0 — snapshot at creation time
```

For reference types, the capture list copies **the reference pointer**, not the object. Property mutations on the same object are still visible, but reassigning the variable to a different object is not:

| Behavior | Default (no capture list) | With capture list `[x]` |
|----------|--------------------------|------------------------|
| What's captured | Reference to variable | Snapshot copy (constant) |
| Sees later changes | Yes | No (reassignment); Yes (mutation of ref type properties) |
| Mutability | Read/write if `var` | Read-only |
| Evaluated when | At closure execution | At closure creation |

The practical trap in UIKit: capturing a struct value (like a `Message`) in a capture list gives you the empty initial value, not the user's edits. Capture `self` weakly and access the struct through `self` instead to always get the current value.

---

## 5. Five patterns for verifying UIViewController deallocation

### Print in `deinit` — the simplest check

```swift
class ProfileVC: UIViewController {
    deinit {
        #if DEBUG
        print("♻️ \(String(describing: type(of: self))) deinit")
        #endif
    }
}
```

Dismiss the VC and watch the console. No print? You have a leak.

### Dispatch-after-delay assertion — the most powerful runtime check

Pioneered by Arek Holko and used by the `DeallocationChecker` library, this fires a delayed assertion after a VC disappears:

```swift
extension UIViewController {
    func checkDeallocation(afterDelay delay: TimeInterval = 2.0) {
        let type = type(of: self)

        if isMovingFromParent || isBeingDismissed {
            DispatchQueue.main.asyncAfter(deadline: .now() + delay) { [weak self] in
                assert(self == nil,
                    "\(type) not deallocated after being dismissed — possible leak")
            }
        }
    }
}

// Call in every VC:
override func viewDidDisappear(_ animated: Bool) {
    super.viewDidDisappear(animated)
    checkDeallocation()
}
```

### Symbolic breakpoint — zero code changes required

Add a symbolic breakpoint on `-[UIViewController dealloc]` in Xcode's Breakpoint Navigator. Set the action to play a sound and log `--- dealloc @(id)[$arg1 description]@` with "Automatically continue" checked. Every VC deallocation produces an audible pop. Dismiss a VC and hear silence? That's a leak.

### Unit test with `addTeardownBlock`

```swift
func trackForMemoryLeaks(_ instance: AnyObject,
                          file: StaticString = #file, line: UInt = #line) {
    addTeardownBlock { [weak instance] in
        XCTAssertNil(instance,
            "Instance should have been deallocated. Potential memory leak.",
            file: file, line: line)
    }
}
```

### Community libraries worth adopting

**LifetimeTracker** (by Krzysztof Zabłocki) shows a real-time visual overlay counting live instances per type. **LeakedViewControllerDetector** auto-detects when a VC closes but doesn't `deinit`, showing an alert with a screenshot. Both are set-and-forget debug tools.

---

## 6. Xcode Memory Graph Debugger: a step-by-step workflow

**Step 0 — Enable Malloc Stack Logging.** Edit your scheme (⌘<), go to Run → Diagnostics, check "Malloc Stack" with "Live Allocations Only." This gives you allocation backtraces for every object.

**Step 1 — Exercise the suspected flow.** Navigate to the suspect VC, dismiss it, and **repeat 3–5 times**. If you pushed the VC 5 times and all 5 instances still exist, the leak is obvious.

**Step 2 — Capture the graph.** Click the **three-circle icon** in the debug bar (between the view debugger and location simulator buttons), or use Debug → Debug Memory Graph. The app pauses.

**Step 3 — Read the navigator.** The left panel lists all live objects by type with instance counts. `MyViewController (5)` after dismissing 5 times confirms a leak. **Purple "!" indicators** mark objects Xcode has auto-detected as leaked (unreachable from any root).

**Step 4 — Trace the graph.** Click an instance to see a visual graph in the center panel. **Bold arrows are strong references; dashed are weak.** Follow strong arrows to find the cycle — typically a closure context pointing back to your VC, or a Timer, or a delegate without `weak`.

**Step 5 — Check backtraces.** With Malloc Stack Logging enabled, the right inspector shows exactly which line of code created each object, pinpointing where the problematic reference was established.

**Important caveat:** Xcode's auto-detection only flags objects with **no root reference**. Retain cycles still reachable from `UIWindow` won't show purple markers. **Manual investigation is essential** — search for your class name and check instance counts.

WWDC 2024's "Analyze heap memory" session introduced improved closure context visualization: **the Memory Graph Debugger now labels closure captures explicitly**, showing which closures hold strong references to which objects. Set the Reflection Metadata Level to "All" in build settings for accurate weak/unowned reference display. You can export `.memgraph` files with File → Export Memory Graph and analyze them with command-line tools: `leaks --fullContent App.memgraph` returns exit code 1 if leaks are found, enabling CI/CD integration.

---

## 7. Swift concurrency's hidden retention: `Task` keeps your VC alive

Swift's `Task { }` closures don't require `self.` explicitly — the compiler allows implicit `self` because Tasks are `@Sendable` and don't structurally form retain cycles. But they **strongly capture `self`** for the entire duration of the task. For a quick network call, this is harmless. For a long-running `AsyncSequence` loop, it's a genuine memory leak:

```swift
// ❌ PROBLEMATIC: Task retains self until the infinite stream ends (never)
class UserListVC: UIViewController {
    private var observationTask: Task<Void, Never>?

    override func viewWillAppear(_ animated: Bool) {
        super.viewWillAppear(animated)
        observationTask = Task {
            for await users in userList.$users.values {
                updateTableView(withUsers: users)  // self retained forever
            }
        }
    }

    deinit {
        observationTask?.cancel()  // ❌ NEVER CALLED — self never deallocates
    }
}
```

The fix: **cancel tasks in `viewDidDisappear`, not `deinit`**. For long-running streams, unwrap `self` per iteration instead of once at the top:

```swift
// ✅ FIXED: Cancel on disappear + per-iteration weak self for streams
class UserListVC: UIViewController {
    private var loadingTask: Task<Void, Never>?
    private var observationTask: Task<Void, Never>?

    override func viewWillAppear(_ animated: Bool) {
        super.viewWillAppear(animated)

        // One-shot task — completes quickly, strong self is acceptable
        loadingTask = Task {
            do {
                let profile = try await profileService.loadProfile()
                renderProfile(profile)
            } catch {
                if !Task.isCancelled { showError(error) }
            }
            loadingTask = nil
        }

        // Long-running stream — unwrap weak self per iteration
        observationTask = Task { [weak self] in
            guard let stream = self?.userList.$users.values else { return }
            for await users in stream {
                guard let self else { break }  // Strong ref scoped to one iteration
                self.updateTableView(withUsers: users)
            }  // Strong self released at end of each iteration
        }
    }

    override func viewDidDisappear(_ animated: Bool) {
        super.viewDidDisappear(animated)
        loadingTask?.cancel()
        loadingTask = nil
        observationTask?.cancel()  // ✅ Triggers CancellationError → releases self
        observationTask = nil
    }

    deinit {
        loadingTask?.cancel()       // Safety net
        observationTask?.cancel()
        print("UserListVC deallocated ✅")
    }
}
```

A common mistake is using `[weak self]` with an immediate `guard let self` at the top of a Task — this re-establishes a strong reference for the **entire task duration**, defeating the purpose. Only unwrap per-iteration for loops, or use `self?` at each call site.

### Structured concurrency is inherently safer

`TaskGroup`, `async let`, and `withThrowingTaskGroup` do not have the same problem. Child tasks are scoped to their parent — when the scope exits, children are automatically cancelled. **`withDiscardingTaskGroup`** (Swift 5.9) goes further by freeing resources immediately when each child task finishes, improving memory behavior for fan-out patterns.

### `withCheckedContinuation` risks

If a legacy API never calls its completion handler, the continuation is never resumed, and everything it captured stays in memory permanently. Always handle the nil/failure path and consider `[weak self]` in the continuation closure.

---

## What's new from WWDC 2024, WWDC 2025, and Swift 6

**WWDC 2024** focused on heap memory analysis. The "Analyze heap memory" session explicitly called out closure contexts in the Memory Graph Debugger and warned against using methods directly as closure values (which creates hidden retain cycles). Swift 6.0 shipped with **complete concurrency checking** — `@Sendable` enforcement, region-based isolation (SE-0414), and the `sending` keyword (SE-0430). These don't detect retain cycles directly but encourage value types and make isolation boundaries explicit, reducing accidental strong captures.

**WWDC 2025** introduced Swift 6.2's "Approachable Concurrency" overhaul. **SE-0466** defaults new projects to `@MainActor` isolation — all types implicitly run on the main actor unless opted out with `@concurrent`. **SE-0461** changes nonisolated async functions to run on the caller's actor by default rather than hopping to the global concurrent executor, reducing unexpected thread switches. Xcode 26's debugger can now **follow execution into async tasks**, a major improvement for diagnosing Task-related retention. These changes don't eliminate the `Task` retention problem but make the concurrency model more predictable, reducing the surface area for surprises.

**SE-0365** (Swift 5.8) remains the most impactful quality-of-life change for memory management code: implicit `self` after unwrapping `[weak self]`. But its nested closure rule — requiring explicit recapture in inner closures — provides a genuine safety mechanism that prevents Trap D at the compiler level.

## Conclusion

The retain cycle landscape in Swift has matured but not simplified. The four classical traps — Timer, NotificationCenter, CADisplayLink, and nested closures — remain exactly as dangerous as ever in UIKit codebases. Swift concurrency adds a fifth category where `Task` silently holds objects alive without forming a traditional cycle. **The key insight is that `deinit` is unreliable as a cleanup mechanism** for anything involved in the very cycle you're trying to break; lifecycle methods like `viewWillDisappear` are the correct invalidation point.

Three practices prevent the vast majority of leaks: always use `[weak self]` in escaping closures (unless you can prove lifetime), always declare delegates as `weak var` on `AnyObject`-constrained protocols, and always cancel Tasks in view lifecycle methods rather than `deinit`. Combine that with routine Memory Graph Debugger checks — especially watching instance counts after repeated navigation — and `deinit` print statements during development, and most leaks become visible before they ship. The tools keep improving, but the fundamental discipline remains unchanged: **know your ownership graph, and verify it regularly**.
---

## Summary Checklist

- [ ] `[weak self]` used in all escaping closures by default
- [ ] `[unowned self]` reserved only for provable-lifetime cases (lazy var stored closures)
- [ ] Timer uses block-based API with `[weak self]`; invalidated in `viewWillDisappear`
- [ ] CADisplayLink uses weak proxy pattern (no block API available)
- [ ] NotificationCenter closure observer uses `[weak self]`; token removed in `deinit`
- [ ] Nested stored closures re-capture `[weak self]` (not relying on outer `guard let self`)
- [ ] Delegates declared as `weak var delegate: SomeDelegate?` on `AnyObject`-constrained protocol
- [ ] `deinit` print/log implemented for development-time leak verification
- [ ] Tasks cancelled in `viewWillDisappear` or `viewDidDisappear` — not relying on `deinit`
- [ ] Memory Graph Debugger used periodically — checking instance counts after repeated navigation
