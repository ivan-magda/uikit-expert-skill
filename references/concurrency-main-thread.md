# UIKit Swift concurrency: the definitive guide to main thread safety

**Swift's concurrency model has fundamentally changed how iOS developers write thread-safe UIKit code.** Since Apple annotated `UIViewController` with `@MainActor` in Swift 5.5 and then enforced strict data-race safety in Swift 6, every UIKit project must grapple with actor isolation, `Sendable` requirements, and Task lifecycle management. Swift 6.2 (Xcode 26, WWDC 2025) pushes the envelope further with **MainActor-by-default isolation** via SE-0466, eliminating most boilerplate while demanding developers understand when and how to escape to background threads. This guide covers eight critical topics with concrete ✅ correct and ❌ incorrect code examples drawn from Swift Evolution proposals, WWDC sessions, and community best practices through early 2026.

---

## 1. UIViewController subclasses inherit @MainActor automatically

Apple annotated `UIViewController` (along with `UIView`, `UILabel`, and most UIKit classes) with `@MainActor` starting with the **iOS 15 SDK / Swift 5.5** in 2021. In Swift 5.x language modes, the compiler only partially enforced this. **Full enforcement arrived with Swift 6 strict concurrency mode in Xcode 16 (2024).**

The key rule: when a class is `@MainActor`, every subclass, every method, and every stored property inherits that isolation automatically. You never need to re-annotate `viewDidLoad`, `@IBAction` handlers, or any other overrides.

```swift
// ✅ CORRECT — no annotations needed on methods
class ProfileViewController: UIViewController {
    var username = ""  // MainActor-isolated property (inherited)

    override func viewDidLoad() {
        super.viewDidLoad()          // Already on MainActor
        username = "Alice"           // Safe — same isolation domain
    }

    @IBAction func refreshTapped(_ sender: UIButton) {
        loadData()                   // Safe — MainActor-isolated call
    }

    func loadData() {
        Task {
            let user = try await api.fetchUser()
            username = user.name     // ✅ Task {} inherits MainActor context
        }
    }
}
```

```swift
// ❌ INCORRECT — redundant annotations (a code smell, not an error)
class ProfileViewController: UIViewController {
    @MainActor var username = ""              // Redundant

    @MainActor override func viewDidLoad() { // Redundant
        super.viewDidLoad()
    }

    @MainActor func loadData() {             // Redundant
        // ...
    }
}
```

Three common mistakes trip developers up. First, **`deinit` is always `nonisolated`** even in `@MainActor` types — accessing isolated properties inside `deinit` is a compiler error in Swift 6. Second, **initializers inherited from non-`@MainActor` superclasses** (like `NSObject.init()`) can produce surprising isolation mismatches. Third, `Task.detached` inside a view controller does **not** inherit MainActor context, which leads directly to data races (covered in section 3).

---

## 2. Task lifecycle: store, cancel, and check before updating UI

Unlike SwiftUI's `.task` modifier, which automatically cancels work when a view disappears, **UIKit provides no built-in Task lifecycle management**. Developers must store `Task` references, cancel them at the right moment, and check cancellation before touching the UI.

### The complete pattern

```swift
// ✅ CORRECT — full Task lifecycle management
class SearchViewController: UIViewController {
    private var searchTask: Task<Void, Never>?
    private var monitorTask: Task<Void, Never>?

    override func viewDidLoad() {
        super.viewDidLoad()
        monitorTask = Task { [weak self] in
            for await notification in NotificationCenter.default
                .notifications(named: UIApplication.didEnterBackgroundNotification) {
                guard let self else { return }
                handleBackground(notification)
            }
        }
    }

    func search(query: String) {
        searchTask?.cancel()                    // Cancel previous search
        searchTask = Task {
            defer { searchTask = nil }          // Clean up reference
            let results = try? await api.search(query)
            guard !Task.isCancelled else { return }  // Check before UI update
            tableView.reloadData()
        }
    }

    override func viewDidDisappear(_ animated: Bool) {
        super.viewDidDisappear(animated)
        searchTask?.cancel()
        monitorTask?.cancel()
    }
}
```

### Why deinit alone is not enough

A critical gotcha: **if a `Task` captures `self` strongly (which it does by default), the Task keeps the view controller alive**, preventing `deinit` from ever firing. This creates a retain cycle that only breaks when the Task completes.

```swift
// ❌ INCORRECT — retain cycle prevents deinit from running
class DetailViewController: UIViewController {
    private var streamTask: Task<Void, Never>?

    deinit {
        streamTask?.cancel()  // ⚠️ May never execute!
    }

    func startMonitoring() {
        streamTask = Task {
            for await event in eventStream {  // Long-lived — implicit self capture
                handleEvent(event)            // self is retained indefinitely
            }
        }
    }
}
```

```swift
// ✅ CORRECT — cancel in viewDidDisappear + weak self for long-lived tasks
class DetailViewController: UIViewController {
    private var streamTask: Task<Void, Never>?

    func startMonitoring() {
        let stream = eventStream              // Capture stream, not self
        streamTask = Task { [weak self] in
            for await event in stream {
                guard let self else { return }
                handleEvent(event)
            }
        }
    }

    override func viewDidDisappear(_ animated: Bool) {
        super.viewDidDisappear(animated)
        streamTask?.cancel()
        streamTask = nil
    }
}
```

Swift uses **cooperative cancellation** — calling `cancel()` merely sets a flag. The task body must check `Task.isCancelled` or call `try Task.checkCancellation()` (which throws `CancellationError`) at appropriate points. Always check cancellation after any `await` and before performing UI updates to avoid showing stale data on a screen the user has already left.

---

## 3. Task.detached drops actor isolation and invites data races

The difference between `Task {}` and `Task.detached {}` is a frequent source of bugs. **`Task {}` inherits the enclosing actor's isolation**, priority, and task-local values. **`Task.detached {}` inherits nothing** — it runs on the global concurrent executor with no actor affiliation.

```swift
// ❌ INCORRECT — data race: updating UI from a detached task
class FeedViewController: UIViewController {
    func refresh() {
        Task.detached {
            let posts = try await self.api.fetchPosts()
            self.posts = posts            // ❌ NOT on MainActor!
            self.tableView.reloadData()   // ❌ UIKit call from background thread
        }
    }
}
```

In Swift 5 this compiles silently and crashes at runtime with unpredictable UI corruption. In Swift 6 strict mode, the compiler catches it. The fixes:

```swift
// ✅ FIX 1 — Use MainActor.run to hop back
class FeedViewController: UIViewController {
    func refresh() {
        Task.detached {
            let posts = try await self.api.fetchPosts()
            await MainActor.run {
                self.posts = posts
                self.tableView.reloadData()
            }
        }
    }
}

// ✅ FIX 2 (preferred) — Use Task {} instead, which inherits MainActor
class FeedViewController: UIViewController {
    func refresh() {
        Task {
            let posts = try await api.fetchPosts()
            self.posts = posts            // ✅ Still on MainActor
            tableView.reloadData()        // ✅ Safe
        }
    }
}
```

The Swift team's official guidance is clear: **avoid `Task.detached` in almost all cases.** David Smith of Apple's Swift team stated in July 2025: "I'm always pretty skeptical of using `detached` just to disable inheritance; I would generally prefer moving the code to a `@concurrent async` function." When you need background execution, the modern approach (Swift 6.2+) is to call a `@concurrent` function from a regular `Task`:

```swift
// ✅ BEST (Swift 6.2+) — @concurrent for explicit background work
class FeedViewController: UIViewController {
    func refresh() {
        Task {
            let posts = await fetchPosts()  // Hops to background
            self.posts = posts              // Back on MainActor automatically
            tableView.reloadData()
        }
    }

    @concurrent
    func fetchPosts() async -> [Post] {
        // Guaranteed to run on background thread
        try await api.fetchPosts()
    }
}
```

---

## 4. MainActor.run versus MainActor.assumeIsolated serve different worlds

These two APIs solve different problems and operate in fundamentally different contexts.

**`MainActor.run {}`** is an `async` function that **switches execution to the main actor** from any async context. It requires `await`, may introduce a suspension point, and the closure body is synchronous (you cannot `await` inside it).

**`MainActor.assumeIsolated {}`** is a **synchronous** function that asserts at runtime you're already on the main actor. It has zero overhead — no task scheduling, no context switch. If called from a background thread, it crashes with a fatal error. Introduced in **Swift 5.9 via SE-0392**.

```swift
// ✅ MainActor.run — hop TO the main actor from a background context
func processData() async {
    let result = await heavyComputation()
    await MainActor.run {
        label.text = result          // Guaranteed on main actor
    }
}

// ✅ MainActor.assumeIsolated — assert you're ALREADY on main actor
// Ideal for bridging legacy synchronous callbacks
class AttachmentProvider: NSTextAttachmentViewProvider {
    override func loadView() {
        // loadView() is synchronous and not annotated @MainActor,
        // but UIKit always calls it on the main thread
        MainActor.assumeIsolated {
            let hostingView = UIHostingController(rootView: MyView())
            self.view = hostingView.view
        }
    }
}
```

```swift
// ❌ INCORRECT — using assumeIsolated from a background thread → crash
Task.detached {
    MainActor.assumeIsolated {
        label.text = "Updated"  // 💥 Fatal error at runtime
    }
}

// ❌ INCORRECT — trying to await inside MainActor.run
Task {
    await MainActor.run {
        await viewModel.fetchData()   // Compiler error: body is synchronous
    }
}

// ✅ FIX — use Task { @MainActor in } when you need async + MainActor
Task { @MainActor in
    statusLabel.text = "Loading..."
    await viewModel.fetchData()       // Can await here
    statusLabel.text = "Done"
}
```

| Feature | `MainActor.run {}` | `MainActor.assumeIsolated {}` |
|---|---|---|
| **Context** | Async (requires `await`) | Synchronous only |
| **Behavior** | Switches to main actor | Asserts already on main actor |
| **Suspension** | Yes (potential hop) | None (inline execution) |
| **Wrong-thread behavior** | Safely hops to correct thread | Crashes at runtime |
| **Introduced** | Swift 5.5 | Swift 5.9 (SE-0392) |
| **Primary use case** | Background → main thread | Legacy sync callbacks known to be on main |

---

## 5. Swift 6 strict concurrency creates a cascade of @MainActor obligations

Enabling Swift 6 language mode (`-swift-version 6`) transforms concurrency warnings into hard errors. The most disruptive effect for UIKit developers is the **cascading `@MainActor` requirement**: since `UIViewController` is `@MainActor`, any protocol it conforms to must also be compatible with MainActor isolation.

### Protocol conformance conflicts

UIKit's own protocols like `UITableViewDataSource` and `UITableViewDelegate` use **whole-conformance `@MainActor` isolation**, so conforming from a `@MainActor` view controller works seamlessly. The problem arises with your own protocols or third-party protocols that are *not* `@MainActor`:

```swift
// ❌ INCORRECT — non-isolated protocol vs @MainActor class
protocol DataProvider {
    func fetchItems() -> [Item]
}

class ItemViewController: UIViewController, DataProvider {
    func fetchItems() -> [Item] {
        // ❌ Error: Main actor-isolated instance method 'fetchItems()'
        // cannot satisfy nonisolated protocol requirement
        return items
    }
}
```

Swift 6 provides three solutions, each appropriate in different situations:

```swift
// ✅ FIX 1 — @preconcurrency conformance (SE-0423, recommended for migration)
class ItemViewController: UIViewController, @preconcurrency DataProvider {
    func fetchItems() -> [Item] {
        return items  // ✅ Compiler adds runtime isolation check
    }
}

// ✅ FIX 2 — nonisolated + assumeIsolated (manual bridge)
class ItemViewController: UIViewController, DataProvider {
    nonisolated func fetchItems() -> [Item] {
        MainActor.assumeIsolated {
            return items  // ✅ Runtime assertion that we're on main
        }
    }
}

// ✅ FIX 3 — Isolated conformance (SE-0470, Swift 6.2+)
class ItemViewController: UIViewController, @MainActor DataProvider {
    func fetchItems() -> [Item] {
        return items  // ✅ Conformance explicitly scoped to MainActor
    }
}
```

### Using nonisolated to escape MainActor

Not every method in a view controller needs main-thread access. Use `nonisolated` to opt specific methods out of MainActor isolation, enabling them to run on any thread:

```swift
class AnalyticsViewController: UIViewController {
    let analyticsID = UUID()  // Immutable — safe from any thread

    // ✅ Pure computation, no UI access needed
    nonisolated func computeHash(for data: Data) -> String {
        data.base64EncodedString()
    }

    // ❌ INCORRECT — accessing isolated state from nonisolated method
    nonisolated func badMethod() -> String {
        return title ?? ""  // ❌ Error: cannot access MainActor property 'title'
    }
}
```

### Sendable requirements at actor boundaries

Any value that crosses an actor boundary must conform to `Sendable`. Swift 6 enforces this at compile time:

```swift
// ❌ INCORRECT — non-Sendable type crossing actor boundary
class MutableConfig {          // Not Sendable (class with var)
    var retryCount = 3
}

@MainActor
class SettingsVC: UIViewController {
    func apply(config: MutableConfig) {
        Task.detached {
            print(config.retryCount)  // ❌ Sending non-Sendable 'config'
        }                             //    across actor boundary
    }
}

// ✅ FIX — make the type Sendable
struct ImmutableConfig: Sendable {
    let retryCount: Int
}

// ✅ OR — use Mutex for mutable thread-safe state (Swift 6+)
import Synchronization

final class SafeConfig: Sendable {
    private let state = Mutex<Int>(3)
    var retryCount: Int {
        state.withLock { $0 }
    }
}
```

Key Swift Evolution proposals driving these changes: **SE-0401** removed property-wrapper-based isolation inference, **SE-0414** introduced region-based isolation to eliminate false-positive Sendable warnings, and **SE-0423** added `@preconcurrency` conformance and dynamic actor isolation checks.

---

## 6. iOS 26 makes MainActor the default — and changes everything

**SE-0466 (Control Default Actor Isolation Inference)**, shipped with Swift 6.2 and Xcode 26 at WWDC 2025, introduces the most significant change to Swift's concurrency model since actors were introduced. When enabled, **all unannotated code in a module is implicitly `@MainActor`** — functions, classes, structs, global variables, and properties all run on the main actor unless explicitly opted out.

New Xcode 26 projects enable this by default. Existing projects can opt in through build settings or Swift Package Manager:

```swift
// Package.swift (Swift 6.2+)
.target(
    name: "MyApp",
    swiftSettings: [
        .defaultIsolation(MainActor.self)
    ]
)
```

### What changes in practice

Code that previously required explicit `@MainActor` now just works:

```swift
// BEFORE (nonisolated default) — Swift 6.0
var appState = AppState()  // ❌ Error: mutable global requires isolation

@MainActor                 // Required boilerplate
class HomeViewModel {
    var items: [Item] = []
    func reload() async { /* ... */ }
}

// AFTER (MainActor default) — Swift 6.2 with defaultIsolation
var appState = AppState()  // ✅ Implicitly @MainActor

class HomeViewModel {       // ✅ Implicitly @MainActor
    var items: [Item] = []
    func reload() async { /* ... */ }
}
```

### Opting out for background work

The flip side: you must now explicitly mark code that should run off the main actor. Swift 6.2 provides two mechanisms:

```swift
// Use 'nonisolated' to remove MainActor isolation
nonisolated func parseJSON(_ data: Data) -> [Item] {
    // Runs wherever the caller runs (inherits caller isolation)
    try JSONDecoder().decode([Item].self, from: data)
}

// Use '@concurrent' to guarantee background execution (new in Swift 6.2)
@concurrent
func compressImage(_ image: UIImage) async -> Data {
    // Guaranteed to run on a background thread
    image.jpegData(compressionQuality: 0.7)!
}

// Use 'nonisolated' on types that must cross isolation boundaries
nonisolated class NetworkResponse {
    let data: Data
    let statusCode: Int
}
```

### Community reception is mixed

Proponents (including Holly Borla from the Swift team and Antoine van der Lee of SwiftLee) argue it dramatically simplifies adoption — most app code is UI-bound, and making MainActor the default eliminates the biggest source of Swift 6 migration pain. Critics like Matt Massicotte warn that it creates a "language dialect" where code behavior depends on an invisible build setting, and that when you inevitably need concurrency, the interactions become harder to reason about. **The practical impact is clear: for typical UIKit/SwiftUI apps, SE-0466 reduces concurrency annotations by 60-80% while maintaining full data-race safety.**

---

## 7. Migrating from GCD to structured concurrency, pattern by pattern

### DispatchQueue.main.async → Task or @MainActor

```swift
// ❌ BEFORE (GCD)
func handleResponse(_ data: Data) {
    DispatchQueue.global().async {
        let parsed = self.parse(data)
        DispatchQueue.main.async {
            self.label.text = parsed.title
            self.tableView.reloadData()
        }
    }
}

// ✅ AFTER (Swift Concurrency)
func handleResponse(_ data: Data) {
    Task {
        let parsed = await parse(data)   // Background work via nonisolated func
        label.text = parsed.title        // Back on MainActor (inherited by Task)
        tableView.reloadData()
    }
}

nonisolated func parse(_ data: Data) async -> Article {
    try! JSONDecoder().decode(Article.self, from: data)
}
```

### DispatchGroup → TaskGroup

```swift
// ❌ BEFORE (GCD)
let group = DispatchGroup()
var profile: Profile?
var posts: [Post] = []
group.enter()
fetchProfile { result in profile = result; group.leave() }
group.enter()
fetchPosts { result in posts = result; group.leave() }
group.notify(queue: .main) {
    self.display(profile: profile!, posts: posts)
}

// ✅ AFTER (Swift Concurrency)
func loadDashboard() async throws {
    async let profile = api.fetchProfile()
    async let posts = api.fetchPosts()
    let (p, ps) = try await (profile, posts)  // Parallel execution
    display(profile: p, posts: ps)            // On MainActor
}
```

### Serial DispatchQueue → Actor

```swift
// ❌ BEFORE (GCD — reader-writer queue)
final class ImageCache {
    private let queue = DispatchQueue(label: "cache", attributes: .concurrent)
    private var storage: [URL: Data] = [:]
    func get(_ url: URL) -> Data? {
        queue.sync { storage[url] }
    }
    func set(_ data: Data, for url: URL) {
        queue.async(flags: .barrier) { self.storage[url] = data }
    }
}

// ✅ AFTER (Actor — compiler-enforced isolation)
actor ImageCache {
    private var storage: [URL: Data] = [:]
    func get(_ url: URL) -> Data? { storage[url] }
    func set(_ data: Data, for url: URL) { storage[url] = data }
}
```

### Completion handlers → async/await via continuations

```swift
// Bridge legacy callback API to async/await
func fetchUser() async throws -> User {
    try await withCheckedThrowingContinuation { continuation in
        legacyFetchUser { result in
            continuation.resume(with: result)  // Must be called EXACTLY once
        }
    }
}
```

**Critical migration pitfall**: never use `DispatchSemaphore.wait()` or `os_unfair_lock` across an `await` suspension point. The cooperative thread pool assumes threads are never blocked — violating this can deadlock your entire app.

---

## 8. Actors protect shared state but reentrancy demands vigilance

Actors provide **compile-time enforced mutual exclusion** — the compiler rejects direct access to an actor's state from outside, requiring `await` for cross-isolation calls. Unlike locks, you cannot forget synchronization.

```swift
actor AnalyticsEngine {
    private var events: [Event] = []

    func track(_ event: Event) {
        events.append(event)      // No await needed — inside the actor
    }

    func flush() async throws {
        let batch = events
        events.removeAll()
        try await network.upload(batch)
    }
}

// External usage always requires await
let engine = AnalyticsEngine()
await engine.track(.screenView("home"))
```

### Actor reentrancy: the hidden trap

When an actor method hits an `await`, the actor **does not block** — it processes other messages. This means the actor's state can change between suspension points, violating assumptions made before the `await`.

```swift
// ❌ INCORRECT — reentrancy bug: double withdrawal
actor BankAccount {
    var balance = 1000

    func withdraw(_ amount: Int) async -> Bool {
        guard amount <= balance else { return false }  // Check
        await authorizeTransaction()                    // ⚠️ Suspension!
        // Another withdraw() call can execute here and see stale balance
        balance -= amount                               // ❌ May go negative
        return true
    }
}
// Two concurrent withdraw(1000) calls both pass the guard, both subtract
```

```swift
// ✅ FIX — perform all checks AFTER suspension points
actor BankAccount {
    var balance = 1000

    func withdraw(_ amount: Int) async -> Bool {
        await authorizeTransaction()         // Suspend first
        guard amount <= balance else { return false }  // Check AFTER
        balance -= amount                    // ✅ State is fresh
        return true
    }
}
```

### Deduplicating in-flight work with Task caching

The most robust reentrancy pattern — used in Apple's own WWDC examples — stores in-progress `Task` references so concurrent callers share a single operation:

```swift
// ✅ CORRECT — reentrancy-safe image downloader
actor ImageDownloader {
    private enum CacheEntry {
        case inProgress(Task<UIImage, Error>)
        case ready(UIImage)
    }
    private var cache: [URL: CacheEntry] = [:]

    func image(from url: URL) async throws -> UIImage {
        if let entry = cache[url] {
            switch entry {
            case .ready(let image):
                return image
            case .inProgress(let task):
                return try await task.value  // Reuse existing download
            }
        }

        let task = Task { try await downloadImage(from: url) }
        cache[url] = .inProgress(task)       // Store before suspension
        do {
            let image = try await task.value
            cache[url] = .ready(image)
            return image
        } catch {
            cache[url] = nil                 // Clean up on failure
            throw error
        }
    }
}
```

**The golden rule for actors**: if all your actor methods are synchronous (no `await`), reentrancy is not a concern and the actor behaves identically to a serial dispatch queue. **Reentrancy only arises when actor methods contain suspension points.** Treat every `await` inside an actor as a point where all assumptions about state must be re-verified.

---

## Conclusion

Swift concurrency in UIKit has matured from an opt-in experiment to the default programming model. The trajectory is clear: **Swift 6.0 made data races compile-time errors, and Swift 6.2 made MainActor isolation the default** — together eliminating entire categories of threading bugs that plagued GCD-era code. The practical patterns that matter most are storing and cancelling `Task` references in view controller lifecycle methods, preferring `Task {}` over `Task.detached {}` to preserve actor isolation, using `nonisolated` and `@concurrent` to explicitly opt into background execution, and treating every `await` inside an actor as a reentrancy boundary. For teams migrating existing UIKit apps, the recommended path is enabling strict concurrency module-by-module, using `@preconcurrency` conformances as a bridge, and adopting `defaultIsolation(MainActor.self)` once on Swift 6.2 to let the compiler handle what developers previously enforced through discipline alone.