# Agent Guidelines for UIKit Expert Skill

This document provides guidance for AI agents working with this skill to ensure consistency and avoid common pitfalls.

## Core Principles

### 1. UIKit Focus Only
**This is a UIKit skill.** Do not include:
- SwiftUI-only patterns (except when bridging is necessary)
- General Swift language features unrelated to UIKit
- Backend or server-side Swift patterns
- Objective-C patterns (use Swift equivalents)

### 2. No Code Formatting or Linting
**Do not include formatting/linting rules.** Avoid:
- Property ordering requirements
- Code organization mandates
- Whitespace or indentation rules
- Naming convention enforcement
- File structure requirements

**Exception**: Mention organization patterns as *optional suggestions* for readability, never as requirements.

### 3. No Architectural Opinions
**Stick to facts, not architectures.** Avoid:
- Enforcing MVVM, MVC, VIPER, or any specific architecture
- Mandating view model patterns
- Requiring specific folder structures
- Dictating dependency injection patterns
- Prescribing coordinator/router patterns

**Exception**: Suggest separating business logic for testability without enforcing how.

### 4. No Tool-Specific Instructions
**Agents cannot use external tools.** Do not include:
- Xcode Instruments profiling step-by-step instructions
- Debugging tool walkthroughs
- IDE-specific feature instructions
- Command-line tool usage beyond basic git

**Exception**: Mention that users can profile with Instruments or use Memory Graph Debugger if issues arise, but don't provide detailed instructions.

## Content Guidelines

### Suggestions vs Requirements

**Use "suggest" or "consider" for optional optimizations:**
- ✅ "Consider downsampling images when loading from disk"
- ✅ "Consider using `UIViewPropertyAnimator` for gesture-driven animations"
- ❌ "Always downsample images"
- ❌ "Always use UIViewPropertyAnimator"

**Use "always" or "never" only for correctness issues:**
- ✅ "Always set `translatesAutoresizingMaskIntoConstraints = false` on programmatic views"
- ✅ "Never change constraint priority from/to `.required` (1000) at runtime"
- ✅ "Always call `super` in lifecycle overrides"
- ✅ "Always use `[weak self]` in escaping closures by default"

### Performance Optimizations

**Present performance optimizations as optional improvements:**
- Image downsampling: Suggest when full-resolution loading is detected
- Prefetching: Suggest for media-heavy lists
- Constraint optimization: Suggest when churn is detected

**Do not automatically apply optimizations.** Let developers decide based on their performance needs.

### iOS Version Targeting

**Be version-aware without being prescriptive:**
- Always note the minimum iOS version for APIs (e.g., "iOS 15+", "iOS 17+")
- Provide fallback patterns for older deployment targets when relevant
- Use `#available` guards in code examples for newer APIs
- Do not assume the project targets the latest iOS version

## What to Include

### ✅ Include These Topics:
- View controller lifecycle correctness (especially `viewIsAppearing`)
- Auto Layout best practices (batch activation, churn avoidance)
- Modern collection view APIs (diffable, compositional, CellRegistration)
- Cell configuration with `UIContentConfiguration`
- Navigation bar appearance and concurrent transition safety
- Animation API selection and correct usage patterns
- Memory management (retain cycles, [weak self], delegate ownership)
- Concurrency and main thread safety (@MainActor, Task lifecycle)
- UIKit–SwiftUI interoperability
- Image loading and downsampling
- Keyboard handling with layout guides
- Trait handling, Dynamic Type, dark mode, accessibility
- Modern API adoption (Observation, updateProperties, .flushUpdates)
- Common anti-patterns AI generators produce

### ❌ Exclude These Topics:
- Swift concurrency deep dives (actors, structured concurrency theory)
- Code formatting and style rules
- Architectural patterns and mandates
- Tool usage instructions (Instruments, debuggers)
- File organization requirements
- Testing frameworks and patterns
- Build system configuration
- Project structure mandates
- Storyboard/Interface Builder workflows (focus on programmatic UIKit)

## Language and Tone

### Use Clear, Direct Language:
- "Consider X when Y" (for optimizations)
- "Avoid X because Y" (for anti-patterns)
- "X is preferred over Y" (for best practices)
- "Use X instead of deprecated Y" (for API migration)

### Avoid Prescriptive Language:
- ❌ "You must organize your view controllers like this"
- ❌ "Always use MVVM with coordinators"
- ❌ "Profile with Instruments following these steps"
- ❌ "Structure your project like this"

## Common AI Generator Mistakes to Correct

When reviewing or generating UIKit code, watch for these frequent AI mistakes:
- Geometry work in `viewDidLoad` instead of `viewIsAppearing`
- Missing `translatesAutoresizingMaskIntoConstraints = false`
- Individual `.isActive = true` instead of `NSLayoutConstraint.activate([])`
- Using deprecated `textLabel`/`detailTextLabel` instead of `UIContentConfiguration`
- Missing `[weak self]` in escaping closures
- No Task cancellation in `viewDidDisappear`
- Setting navigation bar appearance on `navigationBar` instead of `navigationItem`
- Missing `scrollEdgeAppearance` configuration (transparent bar bug)
- No `prepareForReuse` cleanup for async image loading
- Using `traitCollectionDidChange` instead of `registerForTraitChanges`
- Using keyboard notifications instead of `UIKeyboardLayoutGuide`

## Updating the Skill

When adding new content:
1. Ask: "Is this UIKit-specific?"
2. Ask: "Is this a fact or an opinion?"
3. Ask: "Can agents actually use this?"
4. Ask: "Is this about correctness or style?"

If unsure, err on the side of excluding content. It's better to have a focused, factual skill than a comprehensive but opinionated one.

## Summary

**Focus**: UIKit APIs, patterns, and correctness
**Avoid**: Formatting, architecture, tools, Swift language features
**Tone**: Factual, helpful, non-prescriptive
**Goal**: Make agents UIKit experts without enforcing opinions
