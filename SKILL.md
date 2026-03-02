---
name: law-of-demeter
description: Use when accessing nested object properties. Use when chaining method calls. Use when reaching through objects to get data.
---

# Law of Demeter (Don't Talk to Strangers)

## Overview

**Only talk to your immediate friends, not strangers.**

A method should only call methods on: itself, its parameters, objects it creates, or its direct components. Never reach through an object to access another object's internals.

## When to Use

- Accessing nested properties: `obj.a.b.c`
- Chaining method calls: `obj.first.second.third`
- Reaching through objects for data
- Long dot chains in your code

## The Iron Rule

```
NEVER chain through objects. Ask, don't reach.
```

**No exceptions:**
- Not for "it's simpler"
- Not for "it's just one chain"
- Not for "the data is there"
- Not for "fewer lines of code"

## Detection: The Chain Smell

If you see multiple dots, you're violating LoD:

```swift
// ❌ VIOLATION: Reaching through objects
func employeeCity(company: Company, employeeId: String) -> String? {
    company.employees
        .first(where: { $0.id == employeeId })?
        .address.city  // Reaching into employee, then into address
}

// More violations:
let zipCode = user.profile.address.zipCode
let last4 = order.customer.paymentMethod.last4Digits
let theme = user.profile.settings.theme
```

## The Correct Pattern: Ask, Don't Reach

Let objects expose what's needed:

```swift
// ✅ CORRECT: Ask the object directly
class Employee {
    let id: String
    let name: String
    private let address: Address

    init(id: String, name: String, address: Address) {
        self.id = id
        self.name = name
        self.address = address
    }

    var city: String {
        address.city  // Employee asks its own address
    }
}

class Company {
    private let employees: [Employee]

    var employeeCities: [String: String] {
        Dictionary(uniqueKeysWithValues: employees.map { ($0.id, $0.city) })
    }

    func employeeCity(id: String) -> String? {
        employees.first(where: { $0.id == id })?.city
    }
}

// Usage: Ask company, don't reach through it
let city = company.employeeCity(id: employeeId)
```

## Why Chains Are Bad

| Problem | Impact |
|---------|--------|
| **Tight coupling** | Caller knows internal structure |
| **Fragile code** | Structure changes break all callers |
| **Hidden dependencies** | Not obvious what's needed |
| **Hard to test** | Must mock entire chain |
| **Null danger** | Each `.` is a potential nil |

## Allowed Method Calls

A method `m` of class `C` should only call methods on:

1. **`self`** - C's own methods
2. **Parameters** - Objects passed to `m`
3. **Created objects** - Objects `m` creates
4. **Components** - C's direct instance variables
5. **Globals** - Accessible global objects (sparingly)

```swift
class OrderProcessor {
    private let logger: Logger  // Component

    init(logger: Logger) {
        self.logger = logger
    }

    func process(_ order: Order) -> Receipt {  // Parameter
        validate(order)                         // self
        let receipt = Receipt(order: order)     // Created
        logger.log("Processed order")           // Component
        return receipt
    }

    // ❌ NOT ALLOWED: order.customer.address.city
    // ✅ ALLOWED: order.shippingCity
}
```

## Swift-Specific Considerations

### Properties Over Getters

Swift uses properties, not getter methods. Follow Swift API Design Guidelines:

```swift
// ❌ AVOID: Java/TypeScript-style getters
class User {
    func getProfile() -> Profile { ... }
    func getAddress() -> Address { ... }
    func getZipCode() -> String { ... }
}

// ✅ CORRECT: Swift properties
class User {
    var profile: Profile { ... }
    var address: Address { ... }
    var zipCode: String { ... }
}
```

### Access Control

Use Swift's access control to enforce encapsulation:

```swift
class Wallet {
    private(set) var balance: Double = 1000.0  // Read-only from outside

    func canAfford(amount: Double) -> Bool {
        balance >= amount
    }

    func deduct(amount: Double) {
        balance -= amount
    }
}

class Person {
    private let wallet: Wallet  // Hide the wallet itself

    func canAffordPurchase(price: Double) -> Bool {
        wallet.canAfford(amount: price)
    }
}

class Store {
    func processPurchase(person: Person, price: Double) -> Bool {
        // ❌ Cannot do: person.wallet.balance
        // ✅ Must do: person.canAffordPurchase
        person.canAffordPurchase(price: price)
    }
}
```

### Protocol-Oriented Design

Use protocols to define clear interfaces:

```swift
protocol Purchaser {
    func canAfford(amount: Double) -> Bool
    func completePurchase(amount: Double) -> Bool
}

class Person: Purchaser {
    private let wallet: Wallet

    func canAfford(amount: Double) -> Bool {
        wallet.canAfford(amount: amount)
    }

    func completePurchase(amount: Double) -> Bool {
        guard canAfford(amount: amount) else { return false }
        wallet.deduct(amount: amount)
        return true
    }
}

class Store {
    func processPurchase(purchaser: Purchaser, price: Double) {
        // Store only knows about Purchaser protocol, not Person or Wallet
        if purchaser.canAfford(amount: price) {
            _ = purchaser.completePurchase(amount: price)
        }
    }
}
```

### Optional Chaining

Optional chaining (`?.`) still violates LoD even though it's safe:

```swift
// ❌ VIOLATION: Safe but still coupled
let zipCode = user?.profile?.address?.zipCode

// ✅ CORRECT: Single level
let zipCode = user?.zipCode
```

## Pressure Resistance Protocol

### 1. "It's Simpler"
**Pressure:** "One line with dots is simpler than adding properties"

**Response:** Simple to write ≠ simple to maintain. Chains create fragile code.

**Action:** Add properties that expose needed data.

### 2. "It's Just One Chain"
**Pressure:** "It's only two dots, not a big deal"

**Response:** Two dots = two objects you're coupled to. Both can change and break you.

**Action:** Even short chains should be eliminated.

### 3. "The Data Is Right There"
**Pressure:** "The structure has the data, why wrap it?"

**Response:** Structure changes. Wrapping isolates you from changes.

**Action:** Ask the owner for the data.

### 4. "It's Read-Only"
**Pressure:** "I'm just reading, not modifying"

**Response:** Reading through chains still couples you to structure.

**Action:** Ask for what you need.

## Red Flags - STOP and Reconsider

If you notice ANY of these, refactor:

- Multiple dots: `a.b.c.d`
- Chained property access: `obj.first.second.third`
- Optional chains: `a?.b?.c?.d`
- Nil checks for nested access
- Structure knowledge in calling code
- Mocking chains in tests

**All of these mean: Add a property or method to ask directly.**

## Refactoring Chains

```swift
// ❌ BEFORE: Chain
let zip = user.profile.address.zipCode

// ✅ AFTER: Ask
// In User class:
var zipCode: String {
    profile.zipCode
}

// In Profile class:
var zipCode: String {
    address.zipCode
}

// Usage:
let zip = user.zipCode
```

## Real-World Swift Examples

### Example 1: Navigation (UIKit)

```swift
// ❌ VIOLATION
class ViewController: UIViewController {
    func updateUI() {
        navigationController?.navigationBar.tintColor = .blue
        navigationController?.navigationBar.prefersLargeTitles = true
    }
}

// ✅ CORRECT
class ViewController: UIViewController {
    private var navigationBarTintColor: UIColor? {
        get { navigationController?.navigationBar.tintColor }
        set { navigationController?.navigationBar.tintColor = newValue }
    }

    private var prefersLargeNavigationTitles: Bool {
        get { navigationController?.navigationBar.prefersLargeTitles ?? false }
        set { navigationController?.navigationBar.prefersLargeTitles = newValue }
    }

    func updateUI() {
        navigationBarTintColor = .blue
        prefersLargeNavigationTitles = true
    }
}
```

### Example 2: Data Models

```swift
// ❌ VIOLATION
struct OrderView {
    let order: Order

    var displayText: String {
        "Order from \(order.customer.address.city), \(order.customer.address.state)"
    }
}

// ✅ CORRECT
struct Address {
    let city: String
    let state: String

    var cityState: String {
        "\(city), \(state)"
    }
}

struct Customer {
    private let address: Address

    var location: String {
        address.cityState
    }
}

struct Order {
    private let customer: Customer

    var customerLocation: String {
        customer.location
    }
}

struct OrderView {
    let order: Order

    var displayText: String {
        "Order from \(order.customerLocation)"
    }
}
```

### Example 3: View Configuration (UIKit)

```swift
// ❌ VIOLATION
class ProfileViewController: UIViewController {
    func configure(with user: User) {
        nameLabel.text = user.profile.displayName
        emailLabel.text = user.profile.contact.email
        phoneLabel.text = user.profile.contact.phone
    }
}

// ✅ CORRECT
struct User {
    private let profile: Profile

    var displayName: String {
        profile.displayName
    }

    var email: String {
        profile.email
    }

    var phone: String {
        profile.phone
    }
}

class ProfileViewController: UIViewController {
    func configure(with user: User) {
        nameLabel.text = user.displayName
        emailLabel.text = user.email
        phoneLabel.text = user.phone
    }
}
```

### Example 4: SwiftUI Navigation

```swift
// ❌ VIOLATION
struct ContentView: View {
    let app: App

    var body: some View {
        NavigationStack {
            Text(app.currentUser.profile.displayName)
                .navigationTitle(app.settings.theme.name)
                .foregroundColor(Color(app.settings.theme.colors.primary))
        }
    }
}

// ✅ CORRECT
struct App {
    private let currentUser: User
    private let settings: Settings

    var userDisplayName: String {
        currentUser.displayName
    }

    var navigationTitle: String {
        settings.themeName
    }

    var primaryColor: Color {
        settings.primaryColor
    }
}

struct ContentView: View {
    let app: App

    var body: some View {
        NavigationStack {
            Text(app.userDisplayName)
                .navigationTitle(app.navigationTitle)
                .foregroundColor(app.primaryColor)
        }
    }
}
```

### Example 5: SwiftUI State Management

```swift
// ❌ VIOLATION
@Observable
class ViewModel {
    let orderManager: OrderManager

    var displayText: String {
        guard let order = orderManager.orders.first else { return "" }
        return order.customer.address.formattedAddress
    }
}

struct OrderView: View {
    @State private var viewModel: ViewModel

    var body: some View {
        Text(viewModel.displayText)
    }
}

// ✅ CORRECT
struct Order {
    let id: String
    private let customer: Customer

    var shippingAddress: String {
        customer.formattedShippingAddress
    }
}

@Observable
class ViewModel {
    private let orderManager: OrderManager

    var currentOrderAddress: String {
        orderManager.currentOrderAddress
    }
}

struct OrderManager {
    var orders: [Order]

    var currentOrderAddress: String {
        orders.first?.shippingAddress ?? ""
    }
}

struct OrderView: View {
    @State private var viewModel: ViewModel

    var body: some View {
        Text(viewModel.currentOrderAddress)
    }
}
```

### Example 6: SwiftUI Environment and Preferences

```swift
// ❌ VIOLATION
struct SettingsView: View {
    @Environment(\.colorScheme) var colorScheme
    let appState: AppState

    var body: some View {
        VStack {
            Text("Theme: \(appState.userPreferences.theme.displayName)")
            Text("Font Size: \(appState.userPreferences.accessibility.fontSize)")
            Toggle("Dark Mode",
                   isOn: .constant(appState.userPreferences.theme.isDark))
        }
    }
}

// ✅ CORRECT
struct AppState {
    private let userPreferences: UserPreferences

    var themeName: String {
        userPreferences.themeName
    }

    var fontSize: Double {
        userPreferences.fontSize
    }

    var isDarkMode: Bool {
        userPreferences.isDarkMode
    }
}

struct SettingsView: View {
    @Environment(\.colorScheme) var colorScheme
    let appState: AppState

    var body: some View {
        VStack {
            Text("Theme: \(appState.themeName)")
            Text("Font Size: \(appState.fontSize)")
            Toggle("Dark Mode", isOn: .constant(appState.isDarkMode))
        }
    }
}
```

## Quick Reference

| Chain (Bad) | Ask (Good) |
|-------------|------------|
| `company.employees[0].address.city` | `company.employeeCity(id: id)` |
| `order.customer.paymentMethod.last4` | `order.paymentLast4Digits` |
| `user.profile.settings.theme` | `user.preferredTheme` |
| `car.engine.fuel.level` | `car.fuelLevel` |
| `view.layer.shadowColor` | `view.shadowColor` (via extension) |

## Common Rationalizations (All Invalid)

| Excuse | Reality |
|--------|---------|
| "It's simpler" | Chains are simpler to write, harder to maintain. |
| "Just one chain" | One chain = multiple couplings. |
| "Data is right there" | Expose it properly through properties. |
| "It's read-only" | Reading chains still couples you. |
| "Fewer lines" | Lines don't matter. Maintainability does. |
| "It's obvious what it does" | Obvious coupling is still coupling. |
| "Swift has optional chaining" | Optional chaining makes it safe, not good design. |

## The Bottom Line

**Ask objects for what you need. Don't reach through them.**

When you need data from nested objects: add a property on the owner that returns it. Never chain through multiple objects. Each dot is a dependency you're taking on.

## Swift 6+ Considerations

### Strict Concurrency

With Swift 6's strict concurrency checking, chains interact with actor isolation and `Sendable` in subtle ways:

```swift
// ❌ VIOLATION: Leaks deep object graph details and couples to internals
actor OrderManager {
    var orders: [Order]

    func getCustomerEmail(orderId: String) async -> String? {
        orders.first(where: { $0.id == orderId })?.customer.email
        // Harder to keep a narrow actor boundary; easy to start exposing non-Sendable internals
    }
}

// ✅ CORRECT: Keeps a narrow, well-defined actor boundary
actor OrderManager {
    var orders: [Order]

    func customerEmail(orderId: String) async -> String? {
        orders.first(where: { $0.id == orderId })?.customerEmail
    }
}

struct Order: Sendable {
    let id: String
    private let customer: Customer

    var customerEmail: String {
        customer.email
    }
}
```

### Value Semantics

Swift's value types (structs) make copying cheap but don't excuse chains:

```swift
// ❌ VIOLATION: Still coupled even with structs
struct ViewState {
    let user: User
    let settings: Settings
}

let theme = viewState.user.preferences.theme
let fontSize = viewState.settings.display.fontSize

// ✅ CORRECT: Expose what's needed
struct ViewState {
    private let user: User
    private let settings: Settings

    var theme: Theme {
        user.theme
    }

    var fontSize: CGFloat {
        settings.fontSize
    }
}

let theme = viewState.theme
let fontSize = viewState.fontSize
```

## Testing Benefits

Following LoD makes testing dramatically easier:

```swift
// ❌ VIOLATION: Hard to test
class OrderProcessor {
    func process(_ order: Order) {
        let email = order.customer.contact.email
        sendConfirmation(to: email)
    }
}

// Must mock: Order -> Customer -> Contact -> email

// ✅ CORRECT: Easy to test
protocol EmailProvider {
    var email: String { get }
}

class OrderProcessor {
    func process(_ order: EmailProvider) {
        sendConfirmation(to: order.email)
    }
}

// Mock just one thing: EmailProvider
```
