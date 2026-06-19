# Contacts List View Controller

## Problem Description

The mobile team at a mid-sized company is building a new iOS app to replace their aging address book integration. The app needs a contacts screen that loads a list of people (name + email) and lets support staff quickly update a contact's display name without refreshing the whole list — a common action when someone changes their name after a company merger.

The previous implementation used a table view with string-based cell identifiers and full model reloads on every update. This caused visible flicker when a name was edited and made the app feel slow. The engineering lead has asked you to rebuild the contacts screen as a `ContactsViewController` using a collection view, keeping the code programmatic (no storyboards or nibs).

The view controller should be self-contained and ready to drop into a `UINavigationController`. Hard-code an initial list of 20 contacts so the screen is fully usable without a real backend. Each contact should have a unique identifier, a full name, and an email address.

The screen must also support:
- Efficiently updating a single contact's display name without touching any other cells or reloading the whole list
- Cells that visually reflect selection state (e.g., a tinted or highlighted background when a cell is selected)

The layout does not need to be elaborate, but it should be built with extensibility in mind — the list may gain multiple sections or supplementary views in the future.

## Output Specification

Produce a single Swift source file named `ContactsViewController.swift` containing the complete `ContactsViewController` implementation. The file should include:

- The `Contact` model type (with an `id` field, `name`, and `email`)
- The `ContactsViewController` class with all collection view setup done programmatically
- A public (or internal) `updateContact(_ contact: Contact)` method that updates the named contact efficiently
- Mock data for 20 contacts hardcoded inside the view controller

No external dependencies, third-party libraries, or additional files are required. The file should compile against iOS 15+ with Xcode's standard UIKit framework.
