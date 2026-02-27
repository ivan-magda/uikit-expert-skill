# Modern UIKit Collection View APIs: the definitive best-practices guide

The UIKit Collection View stack underwent a radical modernization between iOS 13 and iOS 17. **Diffable data sources, compositional layout, type-safe cell registration, and list configurations** now form a cohesive, declarative toolkit that replaces both the old `UICollectionViewFlowLayout`/delegate-callback pattern *and* `UITableView` itself. This guide distills every production-critical pattern and pitfall across seven topic areas, with concrete ✅ correct and ❌ incorrect Swift examples throughout.

---

## 1. Diffable data source identity: use the model ID, never the full struct

The diffable data source algorithm tracks items across snapshots by **identity**—determined by `Hashable` and `Equatable` conformance. Apple's explicit recommendation (WWDC 2021 session 10252, official sample code) is to populate snapshots with **stable identifiers** (e.g., `UUID`, database primary key), not full model objects.

When a snapshot stores the full struct with synthesized `Hashable`, changing *any* property makes the diff engine interpret the update as a delete-then-insert of a new item. The collection view **loses selection state, running animations, and cell caches**.

### ✅ Correct: snapshot stores `Recipe.ID`, cell provider looks up full model

```swift
struct Recipe: Identifiable {
    let id: UUID
    var title: String
    var isFavorite: Bool
}

// Item identifier is Recipe.ID (UUID), not Recipe
var dataSource: UICollectionViewDiffableDataSource<Section, Recipe.ID>!

dataSource = .init(collectionView: collectionView) {
    collectionView, indexPath, recipeID -> UICollectionViewCell? in
    let recipe = RecipeStore.shared.recipe(for: recipeID)
    return collectionView.dequeueConfiguredReusableCell(
        using: self.cellRegistration, for: indexPath, item: recipe
    )
}

// Build snapshot with IDs only
var snapshot = NSDiffableDataSourceSnapshot<Section, Recipe.ID>()
snapshot.appendSections([.main])
snapshot.appendItems(recipes.map(\.id), toSection: .main)
dataSource.applySnapshotUsingReloadData(snapshot) // initial load
```

### ❌ Incorrect: full struct as item identifier

```swift
// BAD — synthesized Hashable uses ALL stored properties
var dataSource: UICollectionViewDiffableDataSource<Section, Recipe>!

// Toggling recipe.isFavorite now causes a delete + insert animation
// instead of an in-place update. Selection is lost. reconfigureItems
// and reloadItems cannot work because identity is unstable.
```

### Hashable and Equatable must both key on ID only

If you *do* store model structs directly, you **must** override both `hash(into:)` and `==` to use only the stable identifier. The Hashable contract requires that two values that compare equal must produce the same hash. Violating this crashes the diff engine.

```swift
// ✅ Correct custom conformance
struct Item: Hashable {
    let id: UUID
    var title: String
    var count: Int

    func hash(into hasher: inout Hasher) { hasher.combine(id) }
    static func == (lhs: Self, rhs: Self) -> Bool { lhs.id == rhs.id }
}
```

```swift
// ❌ Incorrect — hash includes title, equality uses only id
struct Item: Hashable {
    let id: UUID
    var title: String

    func hash(into hasher: inout Hasher) { hasher.combine(title) } // BAD
    static func == (lhs: Self, rhs: Self) -> Bool { lhs.id == rhs.id }
}
// Two items with same id but different titles will be == yet hash
// differently → violates Hashable contract → crash or undefined behavior
```

### The `BUG_IN_CLIENT` duplicate-identifier crash

The crash message `BUG_IN_CLIENT_OF_DIFFABLE_DATA_SOURCE__DUPLICATE_ITEM_IDENTIFIERS` (or the assertion `_identifiers.count == _items.count`) fires when a snapshot contains two items that resolve to the same identity. Every item identifier in a snapshot **must be unique**.

**Common triggers and fixes:**

- **Duplicate models in source data.** Deduplicate before building the snapshot: `let unique = Array(Dictionary(grouping: items, by: \.id).compactMap(\.value.first))`
- **Enum cases without distinguishing associated values.** `case loading` can appear only once. Fix: `case loading(UUID)` so each placeholder is unique.
- **Incorrect Hashable** causing unrelated items to hash-collide into equality.
- **Race conditions** applying snapshots from both main and background queues simultaneously—the internal state becomes inconsistent. Always apply from **one queue consistently** (prefer main).

```swift
// ❌ Crash: enum case used as placeholder without unique identity
enum CellItem: Hashable {
    case loading
    case content(Model)
}
let items = Array(repeating: CellItem.loading, count: 5) // 💥 duplicates

// ✅ Fix: give each placeholder a unique ID
enum CellItem: Hashable {
    case loading(UUID)
    case content(Model)
}
let items = (0..<5).map { _ in CellItem.loading(UUID()) }
```

---

## 2. Snapshot application: `apply` vs `applySnapshotUsingReloadData`, and reconfigure vs reload

### Choosing the right apply method

| Behavior | `apply(animatingDifferences:)` | `applySnapshotUsingReloadData` |
|---|---|---|
| Diffing | Always computes diff (iOS 15+) | No diff — full reset |
| Animation | Yes if `true`; still diffs if `false` on iOS 15+ | Never |
| Cell reuse | Cells for unchanged items stay in place | All cells discarded and re-dequeued |
| Selection state | Preserved for unchanged items | Lost |
| Best for | Incremental updates | Initial load, wholesale data replacement |

A critical behavioral change landed in **iOS 15**: calling `apply(animatingDifferences: false)` no longer acts as `reloadData`—it still diffs, just without animation. Use `applySnapshotUsingReloadData` when you genuinely want a full reset.

```swift
// ✅ Initial load — no diff needed, no animation desired
func loadInitialData() {
    var snapshot = NSDiffableDataSourceSnapshot<Section, Recipe.ID>()
    snapshot.appendSections([.main])
    snapshot.appendItems(allRecipeIDs)
    dataSource.applySnapshotUsingReloadData(snapshot)
}

// ✅ Subsequent update — diff and animate
func recipesDidChange(_ updatedIDs: [Recipe.ID]) {
    var snapshot = NSDiffableDataSourceSnapshot<Section, Recipe.ID>()
    snapshot.appendSections([.main])
    snapshot.appendItems(updatedIDs)
    dataSource.apply(snapshot, animatingDifferences: true)
}
```

### `reconfigureItems` vs `reloadItems` — the exact difference

Both are mutating methods on `NSDiffableDataSourceSnapshot` (iOS 15+). **`reconfigureItems` reuses the existing cell in place**; `reloadItems` discards the cell and dequeues a fresh one.

| Aspect | `reconfigureItems` | `reloadItems` |
|---|---|---|
| Cell lifecycle | Existing cell returned from dequeue | New cell dequeued; `prepareForReuse()` called |
| Cell type can change | ❌ Must be the same registration | ✅ Can switch to a different cell class |
| Prefetched cells | Preserved | Discarded (wastes prefetch work) |
| Performance | **Better** — avoids dequeue overhead | Worse — full creation cycle |
| Running cell state | Preserved (animations, gestures) | Reset |

Apple's guidance (WWDC 2021): **prefer `reconfigureItems` unless the cell type itself must change**.

```swift
// ✅ Preferred: reconfigure for content-only updates
func recipeContentChanged(_ recipeID: Recipe.ID) {
    var snapshot = dataSource.snapshot()
    snapshot.reconfigureItems([recipeID])
    dataSource.apply(snapshot, animatingDifferences: true)
}

// ✅ Required: reload when cell type changes (e.g., compact → expanded)
func switchCellType(for itemID: Recipe.ID) {
    var snapshot = dataSource.snapshot()
    snapshot.reloadItems([itemID])
    dataSource.apply(snapshot, animatingDifferences: true)
}
```

### Thread safety rule

`apply()` can be called from a background thread, but you **must always use the same queue**. Mixing main-queue and background-queue applies causes internal state corruption and assertion failures. The safest pattern is to always dispatch to `DispatchQueue.main` or use Swift concurrency's `@MainActor`.

---

## 3. Compositional layout: section providers, caching, and orthogonal scrolling

### The section provider pattern

`UICollectionViewCompositionalLayout(sectionProvider:)` accepts a closure called for **each section** whenever layout is needed (including on rotation and trait changes). This is the recommended approach for multi-section layouts.

```swift
// ✅ Section-provider with environment-adaptive columns
func createLayout() -> UICollectionViewLayout {
    UICollectionViewCompositionalLayout { [weak self] sectionIndex, environment in
        guard let self,
              let sectionKind = self.dataSource.sectionIdentifier(for: sectionIndex)
        else { return nil }

        switch sectionKind {
        case .hero:
            return self.createHeroSection()
        case .featured:
            let section = self.createCarouselSection()
            section.orthogonalScrollingBehavior = .groupPaging
            return section
        case .list:
            var config = UICollectionLayoutListConfiguration(appearance: .insetGrouped)
            config.headerMode = .supplementary
            return NSCollectionLayoutSection.list(
                using: config, layoutEnvironment: environment
            )
        }
    }
}
```

### Layout section caching for scroll performance

The section provider closure can fire multiple times per layout pass. For complex sections with many nested groups and supplementary items, **cache `NSCollectionLayoutSection` objects** and invalidate only when data structure changes.

```swift
// ✅ Cache sections to avoid repeated object allocation
private var sectionLayoutCache: [Int: NSCollectionLayoutSection] = [:]

func createLayout() -> UICollectionViewLayout {
    UICollectionViewCompositionalLayout { [weak self] sectionIndex, env in
        if let cached = self?.sectionLayoutCache[sectionIndex] {
            return cached
        }
        let section = self?.buildSection(at: sectionIndex, environment: env)
        self?.sectionLayoutCache[sectionIndex] = section
        return section
    }
}

// Invalidate on data changes that alter section structure
func dataDidChange() {
    sectionLayoutCache.removeAll()
    collectionView.collectionViewLayout.invalidateLayout()
}
```

### Orthogonal scrolling and `visibleItemsInvalidationHandler`

Orthogonal scrolling enables horizontally-scrollable carousels inside a vertically-scrolling collection view—replacing the old "collection-view-inside-table-view-cell" hack with a single line:

```swift
section.orthogonalScrollingBehavior = .groupPaging
// Options: .continuous, .continuousGroupLeadingBoundary,
//          .groupPaging, .groupPagingCentered, .paging
```

Orthogonal sections do **not** trigger `UIScrollViewDelegate`. To react to scroll position (e.g., for parallax or scale effects), use `visibleItemsInvalidationHandler`:

```swift
section.visibleItemsInvalidationHandler = { visibleItems, scrollOffset, environment in
    let centerX = scrollOffset.x + environment.container.contentSize.width / 2
    for item in visibleItems {
        let distance = abs(item.frame.midX - centerX)
        let scale = max(1.0 - distance / environment.container.contentSize.width, 0.75)
        item.transform = CGAffineTransform(scaleX: scale, y: scale)
    }
}
```

### Supplementary items: headers, footers, badges, decorations

Compositional layout provides three supplementary types:

```swift
// ✅ Boundary supplementary (sticky header)
let headerSize = NSCollectionLayoutSize(
    widthDimension: .fractionalWidth(1.0),
    heightDimension: .estimated(44)
)
let header = NSCollectionLayoutBoundarySupplementaryItem(
    layoutSize: headerSize,
    elementKind: "section-header",
    alignment: .top
)
header.pinToVisibleBounds = true   // sticky header
section.boundarySupplementaryItems = [header]

// ✅ Item-level supplementary (badge)
let badge = NSCollectionLayoutSupplementaryItem(
    layoutSize: NSCollectionLayoutSize(
        widthDimension: .absolute(20), heightDimension: .absolute(20)
    ),
    elementKind: "badge",
    containerAnchor: NSCollectionLayoutAnchor(
        edges: [.top, .trailing], fractionalOffset: CGPoint(x: 0.3, y: -0.3)
    )
)
let item = NSCollectionLayoutItem(layoutSize: itemSize, supplementaryItems: [badge])

// ✅ Decoration item (section background) — registered on LAYOUT, not collectionView
let background = NSCollectionLayoutDecorationItem.background(
    elementKind: "section-background"
)
section.decorationItems = [background]
layout.register(SectionBackgroundView.self, forDecorationViewOfKind: "section-background")
```

---

## 4. `CellRegistration` eliminates string-based reuse identifiers

`UICollectionView.CellRegistration<Cell, Item>` (iOS 14+) replaces the old register-then-dequeue dance with a single, type-safe construct. No string identifiers, no forced downcasting, no possibility of runtime mismatches.

```swift
// ✅ Modern: type-safe, no strings, no manual registration
private lazy var cellRegistration = UICollectionView.CellRegistration<PhotoCell, Photo> {
    cell, indexPath, photo in
    cell.imageView.image = photo.thumbnail
    cell.titleLabel.text = photo.title
}

// In cell provider:
dataSource = .init(collectionView: collectionView) { [weak self] cv, indexPath, photo in
    guard let self else { return nil }
    return cv.dequeueConfiguredReusableCell(
        using: self.cellRegistration, for: indexPath, item: photo
    )
}
```

```swift
// ❌ Legacy: string-based, type-unsafe, crash-prone
collectionView.register(PhotoCell.self, forCellWithReuseIdentifier: "PhotoCell")

// In cellForItemAt — string typos crash at runtime, forced cast is unsafe
let cell = collectionView.dequeueReusableCell(
    withReuseIdentifier: "PhotoCell", for: indexPath
) as! PhotoCell
```

### Critical rule: never create registrations inside the cell provider

Apple's documentation states explicitly: creating a registration inside the `CellProvider` closure creates a **new** registration on every call, defeating cell reuse and crashing on iOS 15+.

```swift
// ❌ WRONG — new registration per call → crash on iOS 15+
dataSource = .init(collectionView: collectionView) { cv, indexPath, item in
    let reg = UICollectionView.CellRegistration<MyCell, Item> { cell, _, item in
        cell.configure(with: item)
    }
    return cv.dequeueConfiguredReusableCell(using: reg, for: indexPath, item: item)
}

// ✅ CORRECT — store registration as a lazy property
private lazy var cellReg = UICollectionView.CellRegistration<MyCell, Item> {
    cell, indexPath, item in
    cell.configure(with: item)
}
```

`SupplementaryRegistration` follows the identical pattern for headers and footers:

```swift
private lazy var headerReg = UICollectionView.SupplementaryRegistration<HeaderView>(
    elementKind: "section-header"
) { [weak self] headerView, _, indexPath in
    let section = self?.dataSource.snapshot().sectionIdentifiers[indexPath.section]
    var config = UIListContentConfiguration.groupedHeader()
    config.text = section?.title
    headerView.contentConfiguration = config
}

dataSource.supplementaryViewProvider = { [weak self] cv, kind, indexPath in
    guard let self else { return nil }
    return cv.dequeueConfiguredReusableSupplementary(using: self.headerReg, for: indexPath)
}
```

---

## 5. `UICollectionLayoutListConfiguration` — five appearances and a rich accessories API

### All five list appearances

| Appearance | Visual description | Typical use |
|---|---|---|
| `.plain` | Full-width cells, no rounding. Matches `UITableView.Style.plain`. | Contacts, search results |
| `.grouped` | Rounded-corner groups on grouped background. Matches `.grouped`. | Settings, forms |
| `.insetGrouped` | Inset rounded cards with margin from edges. Matches `.insetGrouped`. | Modern settings, detail forms |
| `.sidebar` | Translucent background, rounded selection highlight. iPadOS sidebar. | `UISplitViewController` primary column |
| `.sidebarPlain` | Like sidebar but without rounded selection highlight. | Secondary navigation panes |

### Swipe actions configuration

Swipe actions are set on the **layout configuration**, not on a delegate method:

```swift
var config = UICollectionLayoutListConfiguration(appearance: .plain)

config.trailingSwipeActionsConfigurationProvider = { indexPath in
    let delete = UIContextualAction(style: .destructive, title: "Delete") {
        _, _, completion in
        self.deleteItem(at: indexPath)
        completion(true)
    }
    delete.image = UIImage(systemName: "trash")
    return UISwipeActionsConfiguration(actions: [delete])
}

config.leadingSwipeActionsConfigurationProvider = { indexPath in
    let pin = UIContextualAction(style: .normal, title: "Pin") {
        _, _, completion in
        self.pinItem(at: indexPath)
        completion(true)
    }
    pin.backgroundColor = .systemOrange
    return UISwipeActionsConfiguration(actions: [pin])
}
```

### Cell accessories

`UICollectionViewListCell.accessories` accepts an array of `UICellAccessory` values. Each accessory has a `displayed:` parameter controlling visibility in editing vs. non-editing modes.

```swift
cell.accessories = [
    .disclosureIndicator(),                          // trailing chevron
    .checkmark(),                                     // trailing checkmark
    .delete(displayed: .whenEditing, actionHandler: { /* reveal swipe */ }),
    .reorder(displayed: .whenEditing),                // drag handle
    .detail(displayed: .whenNotEditing, actionHandler: { /* info tap */ }),
    .multiselect(displayed: .whenEditing),            // circular checkbox
    .outline(displayed: .always,                      // expand/collapse
             options: .init(style: .header)),
    .label(text: "99+")                               // trailing badge label
]
```

### Separators and headers

```swift
var config = UICollectionLayoutListConfiguration(appearance: .insetGrouped)
config.headerMode = .supplementary      // or .firstItemInSection
config.footerMode = .supplementary
config.showsSeparators = true

// Per-item separator control
config.itemSeparatorHandler = { indexPath, sectionConfig in
    var c = sectionConfig
    if indexPath.item == 0 { c.topSeparatorVisibility = .hidden }
    c.bottomSeparatorInsets = .init(top: 0, leading: 60, bottom: 0, trailing: 0)
    return c
}
```

---

## 6. Self-sizing cells require an unbroken constraint chain and estimated dimensions

### Estimated dimensions on both item and group

For self-sizing to engage, **both** the item and its containing group must use `.estimated()` for the self-sizing axis. Using `.fractionalHeight(1.0)` on the item while the group uses `.estimated()` creates a circular dependency that silently breaks self-sizing.

```swift
// ✅ Correct: both item and group use .estimated for height
let itemSize = NSCollectionLayoutSize(
    widthDimension: .fractionalWidth(1.0),
    heightDimension: .estimated(80)
)
let item = NSCollectionLayoutItem(layoutSize: itemSize)

let groupSize = NSCollectionLayoutSize(
    widthDimension: .fractionalWidth(1.0),
    heightDimension: .estimated(80)
)
let group = NSCollectionLayoutGroup.vertical(layoutSize: groupSize, subitems: [item])
```

```swift
// ❌ Incorrect: item says "be 100% of group height" — circular
let itemSize = NSCollectionLayoutSize(
    widthDimension: .fractionalWidth(1.0),
    heightDimension: .fractionalHeight(1.0)  // BAD — circular with group's .estimated
)
let groupSize = NSCollectionLayoutSize(
    widthDimension: .fractionalWidth(1.0),
    heightDimension: .estimated(80)
)
```

### The constraint chain: top to bottom, in contentView

Every custom self-sizing cell needs an unambiguous vertical constraint chain from `contentView.topAnchor` through all content down to `contentView.bottomAnchor`. Missing the bottom anchor is the single most common self-sizing bug.

```swift
// ✅ Correct: complete top-to-bottom chain in contentView
class DynamicCell: UICollectionViewCell {
    private let titleLabel = UILabel()
    private let bodyLabel = UILabel()

    override init(frame: CGRect) {
        super.init(frame: frame)
        bodyLabel.numberOfLines = 0  // essential for multi-line
        [titleLabel, bodyLabel].forEach {
            $0.translatesAutoresizingMaskIntoConstraints = false
            contentView.addSubview($0)  // contentView, not self
        }
        NSLayoutConstraint.activate([
            titleLabel.topAnchor.constraint(equalTo: contentView.topAnchor, constant: 12),
            titleLabel.leadingAnchor.constraint(equalTo: contentView.leadingAnchor, constant: 16),
            titleLabel.trailingAnchor.constraint(equalTo: contentView.trailingAnchor, constant: -16),

            bodyLabel.topAnchor.constraint(equalTo: titleLabel.bottomAnchor, constant: 8),
            bodyLabel.leadingAnchor.constraint(equalTo: contentView.leadingAnchor, constant: 16),
            bodyLabel.trailingAnchor.constraint(equalTo: contentView.trailingAnchor, constant: -16),
            bodyLabel.bottomAnchor.constraint(equalTo: contentView.bottomAnchor, constant: -12)
            // ↑ this bottom constraint completes the chain
        ])
    }
    required init?(coder: NSCoder) { fatalError() }
}
```

```swift
// ❌ Incorrect: missing bottom constraint — cell collapses to zero height
NSLayoutConstraint.activate([
    titleLabel.topAnchor.constraint(equalTo: contentView.topAnchor, constant: 12),
    // ... leading, trailing constraints ...
    bodyLabel.topAnchor.constraint(equalTo: titleLabel.bottomAnchor, constant: 8),
    // ... leading, trailing constraints ...
    // NO bottom constraint — contentView cannot compute its height
])
```

**Other common mistakes** that break self-sizing:

- Adding subviews to `self` instead of `contentView`
- Leaving `numberOfLines` at its default of `1` on labels expected to wrap
- Using `.absolute()` or `.fractionalHeight()` dimensions instead of `.estimated()`
- Setting an explicit height constraint on the cell that conflicts with Auto Layout intrinsic sizing

When using `UICollectionViewListCell` with `UIListContentConfiguration`, self-sizing is **automatic**—the content configuration handles the entire internal layout. No manual constraints are needed.

```swift
// ✅ UIListContentConfiguration — self-sizing is free
let reg = UICollectionView.CellRegistration<UICollectionViewListCell, Item> {
    cell, indexPath, item in
    var content = cell.defaultContentConfiguration()
    content.text = item.title
    content.secondaryText = item.subtitle
    content.image = UIImage(systemName: "star")
    cell.contentConfiguration = content
    // No constraints needed — self-sizing just works
}
```

---

## 7. Migrating from UITableView: a practical mapping

Apple explicitly recommends UICollectionView with list configuration for all new list-based UI (WWDC 2020). UITableView is not deprecated but receives **no new API investment**—every modern list feature (compositional layout, section snapshots, cell registration, rich accessories) is exclusive to UICollectionView.

### Step-by-step migration

**Step 1 — Replace the view and layout:**

```swift
// Before
let tableView = UITableView(frame: .zero, style: .insetGrouped)

// After
var config = UICollectionLayoutListConfiguration(appearance: .insetGrouped)
config.headerMode = .supplementary
let layout = UICollectionViewCompositionalLayout.list(using: config)
let collectionView = UICollectionView(frame: .zero, collectionViewLayout: layout)
```

**Step 2 — Replace cell registration and configuration:**

```swift
// Before (deprecated textLabel API)
tableView.register(UITableViewCell.self, forCellReuseIdentifier: "cell")
// In delegate:
let cell = tableView.dequeueReusableCell(withIdentifier: "cell", for: indexPath)
cell.textLabel?.text = item.title        // deprecated
cell.detailTextLabel?.text = item.detail  // deprecated
cell.imageView?.image = item.icon         // deprecated
cell.accessoryType = .disclosureIndicator

// After (modern content configuration)
let reg = UICollectionView.CellRegistration<UICollectionViewListCell, Item> {
    cell, indexPath, item in
    var content = cell.defaultContentConfiguration()
    content.text = item.title
    content.secondaryText = item.detail
    content.image = item.icon
    cell.contentConfiguration = content
    cell.accessories = [.disclosureIndicator()]
}
```

**Step 3 — Replace data source:**

```swift
// Before
UITableViewDiffableDataSource<Section, Item>(tableView: tableView) { tv, ip, item in ... }

// After
UICollectionViewDiffableDataSource<Section, Item>(collectionView: collectionView) { cv, ip, item in
    cv.dequeueConfiguredReusableCell(using: reg, for: ip, item: item)
}
```

**Step 4 — Replace delegate:**

```swift
// Before: UITableViewDelegate
func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath)

// After: UICollectionViewDelegate
func collectionView(_ collectionView: UICollectionView, didSelectItemAt indexPath: IndexPath)
```

**Step 5 — Replace background customization:**

```swift
// Before
cell.backgroundColor = .systemBackground
cell.selectedBackgroundView = selectedView

// After — use UIBackgroundConfiguration exclusively
var bg = UIBackgroundConfiguration.listGroupedCell()
bg.backgroundColor = .systemBackground
cell.backgroundConfiguration = bg
```

### Complete API equivalence table

| UITableView | UICollectionView equivalent |
|---|---|
| `UITableView(style: .insetGrouped)` | `UICollectionViewCompositionalLayout.list(using: config)` |
| `UITableViewCell` | `UICollectionViewListCell` |
| `cell.textLabel?.text` | `content.text` via `UIListContentConfiguration` |
| `cell.detailTextLabel?.text` | `content.secondaryText` |
| `cell.imageView?.image` | `content.image` |
| `cell.accessoryType` | `cell.accessories = [...]` |
| `tableView(_:trailingSwipeActionsConfigurationForRowAt:)` | `config.trailingSwipeActionsConfigurationProvider` |
| `UITableViewHeaderFooterView` | `SupplementaryRegistration` with list header element kind |
| `tableView(_:heightForRowAt:)` | `.estimated()` dimensions (self-sizing by default) |
| `cell.backgroundColor` | `cell.backgroundConfiguration` |
| Single-style table | Mix list + grid + carousel sections in one view |

### What you gain by migrating

UICollectionView with list layout gives you everything UITableView offers **plus** the ability to mix list sections with grid and carousel sections in the same view, orthogonal scrolling, per-section layout customization, `NSDiffableDataSourceSectionSnapshot` for expandable/collapsible outlines, multi-column iPad sidebar support via `.sidebar` appearance, and a richer cell accessories system with functional accessories like `.outline()` and `.multiselect()`. The **content configuration system** (`UIListContentConfiguration`, `UIBackgroundConfiguration`) works identically in both UITableView and UICollectionView cells, making the migration of cell content straightforward.

## Conclusion

The modern UIKit collection view stack is built on four interlocking pillars. **Diffable data sources** demand stable, ID-based identity—store identifiers in snapshots, never full structs, and always prefer `reconfigureItems` over `reloadItems` for in-place content updates. **Compositional layout** replaces flow layout and table view layout with a declarative, composable system where each section can have its own behavior, including orthogonal scrolling. **Type-safe cell registration** eliminates an entire class of string-based runtime crashes. And **list configuration** makes UICollectionView a strict superset of UITableView, rendering the latter unnecessary for new projects. Together, these APIs form a coherent, production-ready architecture that handles everything from simple settings screens to complex, multi-section feeds with heterogeneous layouts—all without a single `reloadData()` call or string-based reuse identifier.