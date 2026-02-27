# UIKit animation best practices in Swift (2024–2026)

**Every UIKit animation API ultimately feeds into Core Animation's render server**, but choosing the right wrapper determines how much control you get over interactivity, interruptibility, and layer-level properties. This guide covers the full modern stack—from fire-and-forget `UIView.animate` through gesture-driven `UIViewPropertyAnimator` to layer-only `CABasicAnimation`—with correct and incorrect patterns, the iOS 17 spring redesign, the iOS 18 SwiftUI-UIKit animation bridge, and the iOS 26 `.flushUpdates` option that finally eliminates manual `layoutIfNeeded()` calls.

---

## 1. UIView.animate: the workhorse API

### The `.allowUserInteraction` option and model-layer hit testing

When `UIView.animate` runs, UIKit immediately sets every animatable property to its **final value on the model layer**, then creates a Core Animation interpolation on the **presentation layer** so the user sees a smooth transition. This split creates a critical hit-testing quirk: **touch hit testing evaluates against the model layer, not the presentation layer**. A button animating from y = 0 to y = 400 is tappable at y = 400 the instant the animation begins, even though it visually appears near the top of the screen.

By default, UIKit disables user interaction entirely during animations. Adding `.allowUserInteraction` re-enables touches—but hit testing still targets the model layer. To match the visual position, override `hitTest(_:with:)` and query `layer.presentation()`:

```swift
// ✅ Correct: enable interaction during animation
UIView.animate(withDuration: 0.6, delay: 0, options: [.allowUserInteraction], animations: {
    self.myButton.center.y += 300
}, completion: nil)
```

```swift
// ❌ Incorrect: forgetting .allowUserInteraction — button is untappable during flight
UIView.animate(withDuration: 0.6) {
    self.myButton.center.y += 300
}
```

To hit-test against the visual position, implement presentation-layer checking:

```swift
// ✅ Correct: presentation-layer hit testing for animating views
override func hitTest(_ point: CGPoint, with event: UIEvent?) -> UIView? {
    guard let presentationLayer = layer.presentation() else {
        return super.hitTest(point, with: event)
    }
    let presentationPoint = layer.convert(point, from: presentationLayer)
    return super.hitTest(presentationPoint, with: event)
}
```

### The completion handler's `finished` parameter

The `Bool` passed to the completion closure indicates whether the animation ran to completion. **`finished` is `false` when the animation is interrupted**—for example, if the view is removed from the hierarchy or `layer.removeAllAnimations()` is called. Since iOS 8's additive animation system, adding a new animation on the same property no longer interrupts the first, so both completions fire with `finished == true`.

```swift
// ✅ Correct: check finished before destructive cleanup
UIView.animate(withDuration: 0.5, animations: {
    self.cardView.alpha = 0
}, completion: { finished in
    if finished {
        self.cardView.removeFromSuperview()
    }
})
```

```swift
// ❌ Incorrect: ignoring finished — may remove the view prematurely if interrupted
UIView.animate(withDuration: 0.5, animations: {
    self.cardView.alpha = 0
}, completion: { _ in
    self.cardView.removeFromSuperview() // Dangerous if animation was cancelled
})
```

### Additive animations since iOS 8

Before iOS 8, starting a new `UIView.animate` call on the same property **removed** the existing `CAAnimation` and replaced it, causing a visual jump. Since iOS 8, all `UIView.animate` calls produce **additive `CAAnimation` objects** by default. The old animation continues running while the new one layers on top, blending velocities smoothly.

Additive composition works for **`center`, `frame`, `bounds`, and `transform`**. Properties like `alpha` and `backgroundColor` fall back to non-additive replacement behavior. The older `.beginFromCurrentState` option still stops the current animation and starts fresh from the presentation layer's value—better than pre-iOS 8 jumping, but inferior to the seamless blending of additive animations.

```swift
// ✅ Additive blending: second animation composes smoothly over the first
UIView.animate(withDuration: 1.0) {
    self.circle.center.x = 300
}
DispatchQueue.main.asyncAfter(deadline: .now() + 0.5) {
    UIView.animate(withDuration: 1.0) {
        self.circle.center.y = 400  // Blends with the in-flight x animation
    }
}
```

To truly cancel an additive animation, capture the presentation layer position first:

```swift
// ✅ Correct: cancel by snapping model to presentation, then removing animations
let current = circle.layer.presentation()?.position ?? circle.layer.position
circle.center = CGPoint(x: current.x, y: current.y)
circle.layer.removeAllAnimations()
```

---

## 2. UIViewPropertyAnimator: the interactive animation engine

`UIViewPropertyAnimator` (iOS 10+) wraps Core Animation in a **state machine** that enables pausing, scrubbing, reversing, and interrupting animations—capabilities that `UIView.animate` simply cannot provide.

### Full state machine: inactive → active → stopped

The animator has three states defined by `UIViewAnimatingState`. Understanding valid transitions prevents the runtime crashes that plague this API:

```
                         startAnimation()
              ┌──────────────────────────────────────────┐
              │          pauseAnimation()                 │
              │    fractionComplete = x                   │
              ▼                                           │
        ┌──────────┐                                ┌────┴─────┐
 init──▶│ INACTIVE │                                │  ACTIVE  │
        │          │◀──── animation completes ──────│(running / │
        │          │      (pausesOnCompletion=false) │  paused)  │
        │          │                                │          │
        │          │◀──── stopAnimation(true) ──────│          │
        └──────────┘                                └────┬─────┘
              ▲                                          │
              │                            stopAnimation(false)
              │                                          │
              │         finishAnimation(at:)              ▼
              │◀─────────────────────────────────  ┌──────────┐
              │                                    │ STOPPED  │
                                                   └──────────┘
```

**Key transitions and rules:**

| Method | From state | To state | `isRunning` | Notes |
|---|---|---|---|---|
| `startAnimation()` | inactive / active (paused) | active | `true` | Begins or resumes playback |
| `pauseAnimation()` | inactive / active (running) | active (paused) | `false` | From inactive, enters active paused—useful for scrubbing setup |
| `stopAnimation(true)` | active | **inactive** | `false` | Animations removed; completion blocks **not** called |
| `stopAnimation(false)` | active | **stopped** | `false` | Animations frozen; **must** call `finishAnimation(at:)` next |
| `finishAnimation(at:)` | stopped | **inactive** | `false` | Snaps to `.start`, `.end`, or `.current`; calls completion |
| `continueAnimation(...)` | active (paused) | active (running) | `true` | Resumes with optional new timing and duration factor |

**Crashes you must avoid:**

```swift
// ❌ CRASH: finishAnimation when not in stopped state
animator.startAnimation()
animator.finishAnimation(at: .end) // 💥 Runtime crash

// ❌ CRASH: releasing a stopped animator without finishing
animator.startAnimation()
animator.stopAnimation(false) // State: .stopped
animator = nil               // 💥 "error to release a stopped property animator"

// ❌ CRASH: calling startAnimation after stopAnimation(false) without finishing
animator.stopAnimation(false) // State: .stopped
animator.startAnimation()     // 💥 Must call finishAnimation(at:) first
```

### `stopAnimation(true)` vs `stopAnimation(false)` + `finishAnimation(at:)`

**`stopAnimation(true)`** is a hard teardown: animations are removed, the state goes to `.inactive`, and completion blocks never fire. The view freezes wherever it currently is. Use this when you intend to **discard the animator and create a new one** (the interrupt-and-retarget pattern).

**`stopAnimation(false)`** followed by **`finishAnimation(at:)`** is a graceful two-step. The `.stopped` state freezes the animation, then `finishAnimation(at:)` lets you choose the final position:

```swift
// ✅ Graceful stop: snap to end and trigger completion
animator.stopAnimation(false)
animator.finishAnimation(at: .end) // View jumps to target; completion fires with .end

// ✅ Graceful stop: keep current position
animator.stopAnimation(false)
animator.finishAnimation(at: .current) // View stays where it is; completion fires with .current
```

The critical difference: **`stopAnimation(true)` never calls completion blocks**, while the two-step always does.

### Gesture-driven scrubbing with `fractionComplete`

The core interactive pattern: create an animator, immediately pause it, then drive progress from a gesture recognizer. Always **save `animationProgress` on `.began`** so that interrupting a running animation doesn't cause a jarring offset:

```swift
// ✅ Complete gesture-driven card animation
private var animator: UIViewPropertyAnimator!
private var animationProgress: CGFloat = 0

@objc func handlePan(_ recognizer: UIPanGestureRecognizer) {
    switch recognizer.state {
    case .began:
        animator = UIViewPropertyAnimator(duration: 0.5, dampingRatio: 1) {
            self.cardView.frame.origin.y = self.expandedY
        }
        animator.pauseAnimation()
        animationProgress = animator.fractionComplete   // Save current progress

    case .changed:
        let translation = recognizer.translation(in: view)
        let fraction = translation.y / totalDistance
        animator.fractionComplete = fraction + animationProgress  // Add saved offset

    case .ended:
        let velocity = recognizer.velocity(in: view)
        if velocity.y > 0 {
            animator.isReversed = true
        }
        animator.continueAnimation(withTimingParameters: nil, durationFactor: 0)

    default: break
    }
}
```

```swift
// ❌ Incorrect: forgetting to save progress — causes a jump when interrupting mid-flight
case .began:
    animator.pauseAnimation()
    // Missing: animationProgress = animator.fractionComplete

case .changed:
    animator.fractionComplete = fraction  // Jumps to 0 on interruption!
```

When `scrubsLinearly` is `true` (the default), paused scrubbing ignores the timing curve and maps linearly. This prevents confusing dead zones at the start and end of ease-in-out curves.

### Velocity handoff with `continueAnimation`

`UISpringTimingParameters` expects a **relative velocity** via `CGVector`—not raw points per second. A magnitude of **1.0** means the initial velocity would cover the total remaining animation distance in one second. You must normalize the gesture velocity:

```swift
// ✅ Correct: normalize gesture velocity for spring handoff
func relativeVelocity(for velocity: CGFloat, from current: CGFloat, to target: CGFloat) -> CGFloat {
    guard target - current != 0 else { return 0 }
    return velocity / (target - current)
}

case .ended:
    let velocity = recognizer.velocity(in: view)
    let relVelocity = CGVector(
        dx: relativeVelocity(for: velocity.x, from: card.center.x, to: target.x),
        dy: relativeVelocity(for: velocity.y, from: card.center.y, to: target.y)
    )
    let spring = UISpringTimingParameters(dampingRatio: 0.8, initialVelocity: relVelocity)
    animator.continueAnimation(withTimingParameters: spring, durationFactor: 0)
```

```swift
// ❌ Incorrect: passing raw points/sec as velocity — causes extreme overshoot
let rawVelocity = recognizer.velocity(in: view) // e.g., 800 pts/sec
let spring = UISpringTimingParameters(
    dampingRatio: 0.8,
    initialVelocity: CGVector(dx: rawVelocity.x, dy: rawVelocity.y) // Way too high!
)
```

### Interrupting and re-targeting running animations

The cleanest interrupt-and-retarget pattern uses `stopAnimation(true)` followed by a fresh animator:

```swift
// ✅ Interrupt and retarget to a new position
func animateTo(newTarget: CGPoint) {
    if animator.state == .active {
        animator.stopAnimation(true)  // Hard stop, go to inactive
    }
    animator.addAnimations {
        self.myView.center = newTarget
    }
    animator.startAnimation()
}
```

Apple's iOS calculator exemplifies the pattern—instant highlight on touch-down, animated fade-out on touch-up, fully interruptible on rapid tapping:

```swift
// ✅ Calculator button pattern: instant highlight, animated unhighlight
@objc private func touchDown() {
    animator.stopAnimation(true)
    backgroundColor = highlightedColor            // Instant, no animation
}

@objc private func touchUp() {
    animator = UIViewPropertyAnimator(duration: 0.5, curve: .easeOut) {
        self.backgroundColor = self.normalColor
    }
    animator.startAnimation()
}
```

---

## 3. Spring animations across three eras

### Classic API: `usingSpringWithDamping` (iOS 7+)

The original spring API parameterizes spring behavior with a **damping ratio** (0–1) and a normalized **initial velocity**:

```swift
UIView.animate(withDuration: 0.6, delay: 0,
               usingSpringWithDamping: 0.7,        // 1.0 = critically damped, <1.0 = bouncy
               initialSpringVelocity: 0.0,
               options: [], animations: {
    self.card.center = targetPosition
}, completion: nil)
```

Common damping values: **1.0** for smooth deceleration with no bounce, **0.7** for a subtle professional bounce, **0.5** for noticeable playfulness, and **0.3** for exaggerated emphasis. The `initialSpringVelocity` is normalized so that **1.0 means traversing the total animation distance in one second**.

### iOS 17: `animate(springDuration:bounce:)`

iOS 17 introduced a redesigned spring API that replaces the confusing damping-ratio model with an intuitive **duration + bounce** parameterization, aligned directly with SwiftUI:

```swift
// ✅ iOS 17+: new spring API
UIView.animate(springDuration: 0.5, bounce: 0.0) {   // Smooth, no bounce
    self.myView.center = target
}

UIView.animate(springDuration: 0.5, bounce: 0.3) {   // Bouncy
    self.myView.transform = CGAffineTransform(scaleX: 1.2, y: 1.2)
}
```

The `bounce` parameter ranges from **-1.0 to 1.0**: zero means critically damped, positive values produce overshoot, and negative values create an overdamped sluggish feel. The `springDuration` is a **perceptual duration**—the perceived length of the animation—defaulting to 0.5 seconds.

**Direct SwiftUI alignment (iOS 17+):**

| UIKit | SwiftUI equivalent |
|---|---|
| `springDuration: 0.5, bounce: 0.0` | `.spring(duration: 0.5, bounce: 0.0)` a.k.a. `.smooth` |
| `springDuration: 0.5, bounce: 0.15` | `.spring(duration: 0.5, bounce: 0.15)` a.k.a. `.snappy` |
| `springDuration: 0.5, bounce: 0.3` | `.spring(duration: 0.5, bounce: 0.3)` a.k.a. `.bouncy` |

SwiftUI's `Spring` struct can convert between `(duration, bounce)` and `(mass, stiffness, damping)` for interop with Core Animation's `CASpringAnimation`.

### iOS 18: SwiftUI Animation types in UIKit

iOS 18 bridged the gap entirely. You can now pass a **`SwiftUI.Animation`** value directly to UIKit:

```swift
import SwiftUI

// ✅ iOS 18+: use SwiftUI Animation types with automatic velocity preservation
UIView.animate(.spring(duration: 0.8)) {
    myView.center = CGPoint(x: 200, y: 400)
}

// ✅ Gesture-driven with built-in retargeting — no manual velocity math
switch gesture.state {
case .changed:
    UIView.animate(.interactiveSpring) { bead.center = gesture.location(in: view) }
case .ended:
    UIView.animate(.spring) { bead.center = snapPoint }
}
```

**Critical caveat:** These SwiftUI-bridged animations do **not** create a backing `CAAnimation`. They animate the presentation layer directly. They are also **not compatible** with `UIViewPropertyAnimator` or UIView keyframe animations.

---

## 4. CABasicAnimation for layer-only properties

### Properties that require Core Animation

`UIView.animate` only handles view-level properties (`frame`, `bounds`, `center`, `alpha`, `transform`, `backgroundColor`). The following **CALayer properties are invisible to UIView.animate** and require `CABasicAnimation` or `CATransaction`:

- **`cornerRadius`**, **`borderWidth`**, **`borderColor`**
- **`shadowOpacity`**, `shadowRadius`, `shadowOffset`, `shadowColor`
- **CATransform3D** (3D perspective transforms)
- `anchorPoint`, `sublayerTransform`, `contents`

Placing `myView.layer.cornerRadius = 20` inside a `UIView.animate` block causes the value to **jump instantly** with no interpolation.

### The model layer vs. presentation layer concept

Core Animation maintains two parallel layer trees. The **model layer** (`myView.layer`) stores the "truth" values your code sets directly. The **presentation layer** (`myView.layer.presentation()`) holds the interpolated values currently displayed on screen during animation. When an animation finishes and is removed, the presentation layer **falls back to whatever the model layer says**.

This creates the notorious **"snap-back" bug**: if you add a `CABasicAnimation` without updating the model layer, the view animates beautifully and then **instantly reverts** to its pre-animation state when the animation object is removed.

### The correct pattern: always set the model value

```swift
// ✅ Correct: animate cornerRadius with CABasicAnimation
let animation = CABasicAnimation(keyPath: "cornerRadius")
animation.fromValue = myView.layer.cornerRadius
animation.toValue = 20.0
animation.duration = 0.3
animation.timingFunction = CAMediaTimingFunction(name: .easeInEaseOut)

myView.layer.cornerRadius = 20.0     // ← Update the model layer!
myView.layer.add(animation, forKey: "cornerRadius")
```

```swift
// ❌ Incorrect: forgetting to update the model — snaps back after animation
let animation = CABasicAnimation(keyPath: "cornerRadius")
animation.fromValue = 0.0
animation.toValue = 20.0
animation.duration = 0.3
myView.layer.add(animation, forKey: "cornerRadius")
// ⚠️ cornerRadius SNAPS BACK to 0 when animation completes!
```

```swift
// ❌ Incorrect: the fillMode hack — model and presentation are now permanently out of sync
animation.fillMode = .forwards
animation.isRemovedOnCompletion = false
// Model still says 0! Hit testing, future animations, and memory are all affected.
```

### Animating shadows and borders together

Use `CAAnimationGroup` for simultaneous layer property changes, and always update the model values:

```swift
// ✅ Correct: grouped layer animation
let borderWidth = CABasicAnimation(keyPath: "borderWidth")
borderWidth.fromValue = myView.layer.borderWidth
borderWidth.toValue = 3.0

let borderColor = CABasicAnimation(keyPath: "borderColor")
borderColor.fromValue = myView.layer.borderColor
borderColor.toValue = UIColor.systemGreen.cgColor

let group = CAAnimationGroup()
group.animations = [borderWidth, borderColor]
group.duration = 0.5

myView.layer.borderWidth = 3.0                          // Model update
myView.layer.borderColor = UIColor.systemGreen.cgColor   // Model update
myView.layer.add(group, forKey: "borderChange")
```

### 3D transforms with perspective

`CATransform3D` unlocks perspective rotations impossible with `CGAffineTransform`. Set `m34` for perspective:

```swift
// ✅ Correct: 3D perspective rotation
var perspective = CATransform3DIdentity
perspective.m34 = -1.0 / 500.0   // Perspective depth

let rotated = CATransform3DRotate(perspective, .pi / 6, 0, 1, 0) // Y-axis rotation

let animation = CABasicAnimation(keyPath: "transform")
animation.fromValue = CATransform3DIdentity
animation.toValue = rotated
animation.duration = 0.8
animation.timingFunction = CAMediaTimingFunction(name: .easeInEaseOut)

myView.layer.transform = rotated   // Model update
myView.layer.add(animation, forKey: "3dRotation")
```

To start a new animation from the current visual state during interruption, read `layer.presentation()`:

```swift
// ✅ Correct: interrupt and reanimate from current visual position
let currentRadius = myView.layer.presentation()?.cornerRadius ?? myView.layer.cornerRadius
let animation = CABasicAnimation(keyPath: "cornerRadius")
animation.fromValue = currentRadius   // Start from where it visually IS
animation.toValue = 0.0
animation.duration = 0.3
myView.layer.cornerRadius = 0.0
myView.layer.add(animation, forKey: "cornerRadius")
```

---

## 5. Constraint animation and the iOS 26 revolution

### The classic three-step pattern

Animating Auto Layout constraints requires a specific ritual: **flush pending layout → update constraints → animate `layoutIfNeeded`**:

```swift
// ✅ Correct: the canonical constraint animation pattern
view.layoutIfNeeded()                    // Step 1: flush pending changes
heightConstraint.constant = 300          // Step 2: update constraints OUTSIDE the block

UIView.animate(withDuration: 0.4, delay: 0,
               usingSpringWithDamping: 0.8,
               initialSpringVelocity: 0,
               options: [], animations: {
    self.view.layoutIfNeeded()           // Step 3: force layout INSIDE the block
}, completion: nil)
```

**Why the pre-flush?** Without the initial `layoutIfNeeded()`, any pending constraint changes from earlier code get swept into the animation block and animate unintentionally. You want a clean baseline.

**Why `layoutIfNeeded` inside the block?** `UIView.animate` captures property changes made inside its closure. `layoutIfNeeded()` forces the Auto Layout engine to recalculate frames immediately. Those frame changes—being standard UIView animatable properties—are captured and interpolated by the animation context.

Common mistakes with this pattern:

```swift
// ❌ Incorrect: calling layoutIfNeeded on the child instead of the parent
UIView.animate(withDuration: 0.3) {
    self.childView.layoutIfNeeded()   // Should be self.view.layoutIfNeeded()
}

// ❌ Incorrect: using setNeedsLayout instead of layoutIfNeeded
UIView.animate(withDuration: 0.3) {
    self.view.setNeedsLayout()   // Only marks for update — doesn't force it NOW
}

// ❌ Incorrect: putting layoutIfNeeded OUTSIDE the animation block
heightConstraint.constant = 300
self.view.layoutIfNeeded()   // Layout resolves immediately, nothing left to animate
UIView.animate(withDuration: 0.3) {
    // Nothing to animate here
}
```

### iOS 26: `.flushUpdates` eliminates the ceremony

iOS 26 (announced at WWDC 2025) introduces **`.flushUpdates`**, a `UIView.AnimationOptions` value that automatically applies pending trait, property, and layout updates before entering and after exiting the animation context. This collapses the three-step pattern into a single, clean block:

```swift
// ✅ iOS 26: no layoutIfNeeded needed anywhere
UIView.animate(withDuration: 0.3, options: .flushUpdates) {
    heightConstraint.constant = 300
    leadingConstraint.isActive = false
    trailingConstraint.isActive = true
}
```

The option also works with `UIViewPropertyAnimator` via a new `flushUpdates` property:

```swift
// ✅ iOS 26: flushUpdates on UIViewPropertyAnimator
let animator = UIViewPropertyAnimator(duration: 1, curve: .easeInOut)
animator.flushUpdates = true
animator.addAnimations {
    heightConstraint.constant = 300
}
animator.startAnimation()
```

Combined with iOS 26's **automatic `@Observable` tracking**, UIKit views that depend on observed model properties invalidate and re-layout automatically. This means you can animate model changes without even referencing views:

```swift
// ✅ iOS 26: animate Observable model changes
UIView.animate(withDuration: 0.3, options: .flushUpdates) {
    model.isExpanded = true
    // UIKit automatically knows which views depend on this property
}
```

| Step | Classic pattern (≤ iOS 18) | iOS 26 with `.flushUpdates` |
|---|---|---|
| Pre-flush | `view.layoutIfNeeded()` | Automatic |
| Update constraints | Outside animation block | Inside animation closure |
| Force layout | `view.layoutIfNeeded()` inside block | Automatic |

The `@Observable` tracking can be back-deployed to iOS 18 via the `UIObservationTrackingEnabled` Info.plist key, but `.flushUpdates` itself requires iOS 26.

---

## 6. UIView.transition for content swaps

`UIView.transition` handles **discrete state changes** that can't be interpolated: swapping images, changing text, adding or removing subviews. Regular `UIView.animate` can only interpolate continuous properties.

### `transition(with:)` — animate within a container

```swift
// ✅ Cross-dissolve an image change
UIView.transition(with: imageView, duration: 0.25,
                  options: .transitionCrossDissolve, animations: {
    self.imageView.image = UIImage(named: "newPhoto")
}, completion: nil)

// ✅ Cross-dissolve a label text change
UIView.transition(with: label, duration: 0.25,
                  options: .transitionCrossDissolve, animations: {
    self.label.text = "Updated text"
}, completion: nil)

// ✅ Swap subviews within a container with cross-dissolve
UIView.transition(with: containerView, duration: 0.3,
                  options: .transitionCrossDissolve, animations: {
    oldView.removeFromSuperview()
    containerView.addSubview(newView)
}, completion: nil)
```

### `transition(from:to:)` — replace one view with another

By default, `fromView` is removed from the hierarchy and `toView` is added. To simply toggle visibility instead, include `.showHideTransitionViews`:

```swift
// ✅ Card flip with show/hide — safe to repeat
UIView.transition(from: frontCard, to: backCard, duration: 0.5,
                  options: [.transitionFlipFromRight, .showHideTransitionViews],
                  completion: { _ in self.showingFront.toggle() })
```

```swift
// ❌ Incorrect: without .showHideTransitionViews, fromView is removed from hierarchy
UIView.transition(from: frontCard, to: backCard, duration: 0.5,
                  options: [.transitionFlipFromRight], completion: nil)
// Second call crashes — frontCard is no longer in the view hierarchy!
```

### All seven built-in transition styles

| Option | Effect |
|---|---|
| `.transitionCrossDissolve` | Smooth cross-fade |
| `.transitionFlipFromLeft` / `Right` / `Top` / `Bottom` | 3D flip around the corresponding axis |
| `.transitionCurlUp` / `CurlDown` | Page curl effect |

**Rule of thumb:** use `UIView.animate` to interpolate property values; use `UIView.transition` to apply visual effects around discrete state changes.

---

## 7. When to use each API: a decision guide

Choosing the right animation API depends on three factors: **what** you're animating, **how interactive** it needs to be, and **which iOS version** you target.

### Decision flowchart

```
What are you animating?
│
├── Content changes (image, text, subview swap)?
│   └─▶ UIView.transition
│
├── Layer-only property (cornerRadius, shadow, border, 3D transform)?
│   └─▶ CABasicAnimation / CAAnimationGroup
│
├── Standard view properties (position, size, alpha, transform, color)?
│   │
│   ├── Need gesture-driven scrubbing, pause, reverse?
│   │   └─▶ UIViewPropertyAnimator
│   │
│   ├── Need velocity preservation across retargets? (iOS 18+)
│   │   └─▶ UIView.animate(.swiftUIAnimation)
│   │
│   └── Simple fire-and-forget?
│       └─▶ UIView.animate(withDuration:) or UIView.animate(springDuration:bounce:)
│
└── Complex multi-property with different timing?
    └─▶ CAAnimationGroup or UIView.animateKeyframes
```

### Comparison summary

| Criterion | `UIView.animate` | `UIViewPropertyAnimator` | `CABasicAnimation` | `UIView.animate(.swiftUI)` (iOS 18+) |
|---|---|---|---|---|
| Ease of use | ★★★★★ | ★★★☆☆ | ★★☆☆☆ | ★★★★★ |
| Pause / scrub / reverse | ✗ | ✓ | ✗ | ✗ |
| Gesture velocity handoff | Manual | Manual + `continueAnimation` | ✗ | Automatic |
| Layer-only properties | ✗ | ✗ | ✓ | ✗ |
| 3D transforms | ✗ | ✗ | ✓ | ✗ |
| Autoreverse / repeat | ✓ | ✗ | ✓ | ✓ |
| Creates `CAAnimation` | ✓ | ✓ | ✓ | ✗ |
| Additive by default | ✓ (iOS 8+) | Separate animators | Manual | ✓ |

### Key recommendations for modern codebases

- **Default choice for iOS 17+:** `UIView.animate(springDuration:bounce:)` for fire-and-forget springs with intuitive parameters.
- **Default choice for iOS 18+:** `UIView.animate(.spring)` when you want automatic velocity preservation across gesture-to-animation transitions without manual normalization math.
- **Interactive animations:** `UIViewPropertyAnimator` remains the only option for `fractionComplete` scrubbing, `isReversed` toggling, and the full state machine control needed for custom transitions.
- **Layer properties:** `CABasicAnimation` is non-negotiable for `cornerRadius`, shadows, borders, and 3D transforms. Always update the model layer.
- **Constraint animations on iOS 26+:** use `.flushUpdates` to eliminate boilerplate. On earlier versions, stick to the flush → update → animate `layoutIfNeeded` pattern.
- **Performance:** all UIKit animations are Core Animation under the hood. The render server does the work off the main thread. Performance differences between API levels are **negligible**—choose based on capability, not speed.

---

## Conclusion

UIKit's animation stack has matured into a layered system where each API serves a distinct purpose. **`UIView.animate` handles the 80% case** of simple property transitions; iOS 17's spring reparameterization and iOS 18's SwiftUI bridge make it even more capable with minimal code. **`UIViewPropertyAnimator`'s state machine** unlocks the remaining 20%—interactive gestures, scrubbing, reversal, and interruptible animations—but demands careful state management to avoid runtime crashes. **`CABasicAnimation` remains essential** for the layer properties UIKit cannot reach, provided you respect the model-layer-first rule. And iOS 26's `.flushUpdates` option finally resolves one of UIKit's most error-prone patterns: the constraint animation dance. Across all these APIs, the unifying principle is that **Core Animation splits reality into a model layer (truth) and a presentation layer (appearance)**—understanding that split is the foundation of every correct animation pattern in UIKit.
---

## Summary Checklist

- [ ] Correct API chosen: `UIView.animate` for one-shot, `UIViewPropertyAnimator` for interactive, `CABasicAnimation` for layer properties
- [ ] `UIView.animate` completion checks `finished` parameter before destructive cleanup
- [ ] `.allowUserInteraction` added when interaction needed during animation (with model-layer hit-testing awareness)
- [ ] `UIViewPropertyAnimator` state machine respected: no `finishAnimation` from non-stopped state
- [ ] `stopAnimation(false)` always followed by `finishAnimation(at:)` before releasing animator
- [ ] `CABasicAnimation` sets model value BEFORE adding animation to layer
- [ ] Spring animations use iOS 17+ `UIView.animate(springDuration:bounce:)` when available
- [ ] Constraint animation pattern: flush layout → update constant → animate `layoutIfNeeded()` on superview
- [ ] iOS 26+: `.flushUpdates` used for constraint and Observable-driven animation
- [ ] `UIView.transition` used for discrete changes (images, text) that can't be interpolated
