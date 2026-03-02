# Law of Demeter for Swift (AgentSkills skill)

Strict, human-readable guidance for spotting and fixing Law of Demeter (LoD) violations in Swift code. This repo contains the skill definition used by AgentSkills.

## Why this exists

Deep reach-through chains leak internal structure, make refactors risky, and increase coupling across modules. This skill encourages Swift-idiomatic APIs that:

- express intent
- hide structure
- preserve Swift naming conventions
- reduce refactor blast radius

## When to use

Use this skill when reviewing or refactoring Swift code in domain/business logic. It’s strict by default and assumes a likely LoD violation when:

- the code is in domain/app/business logic
- the caller traverses 2+ domain hops to get a value or trigger behavior
- the traversal exposes another type’s internal structure or storage shape
- the caller could reasonably ask an owning type/service for the same result

## What counts as a “stranger”

Inside a method/computed property body, safe collaborators are generally:

- `self`
- parameters
- locals created in the method
- direct stored properties / direct collaborators

A likely LoD violation happens when code reaches *through* a collaborator to a nested collaborator that the caller does not own.

## Detection heuristics (strict)

Flag or strongly scrutinize patterns like:

- Deep property traversal: `a.b.c`, `a.b.c.d`, `a?.b?.c`
- Domain call + traversal: `service.fetchX().y.z`, `repo.load().nested.value`
- Async call + traversal: `await actor.snapshot().nested.value`
- Temporary drilling variables: `let profile = user.profile` then `return profile.address.postalCode`
- Repeated chain access across files (same chain appearing in multiple places)

### Severity levels

- **High:** async/actor traversal after `await`, 2+ domain hops in business logic, public APIs exposing nested structure, repeated chain smells
- **Medium:** 2-hop domain traversal in internal code, UI/view models reaching into nested domain types, tests needing deep stubs
- **Low / note:** borderline chain in boundary mapping code, one-off reads in localized adapters

## False-positive guardrails

Even in strict mode, don’t auto-flag unless the code also leaks domain internals:

- Standard library pipelines: `orders.filter { $0.isOpen }.map(\.id).sorted()`
- Common string/value transformations
- Intentional fluent APIs / builders / DSLs
- DTO/adapter mapping code at boundaries (keep it localized)

## Swift-specific guidance

- Preserve Swift naming at call sites
  - cheap read-only data → property: `employee.city`
  - action/async/throws/work → method: `company.city(for:)`, `accountService.marketingEmailsEnabled()`
- Don’t suggest Java-style names like `getCity()` / `getMarketingEmailsEnabled()`
- Dot count is a trigger, not proof: long chains are signals, classify them
- Concurrency doesn’t exempt LoD: don’t traverse internals after `await`

## Refactoring strategy (minimal, Swift-idiomatic)

When you flag a violation, propose the smallest safe refactor:

1. Add a forwarding property (cheap, stable data)
2. Add an owner-level method (query/action/async/throws)
3. Move behavior to owner
4. Add a façade/protocol where helpful
5. Collapse async traversal behind an actor/service API

### Minimal change path

- Add a property or method on the owner
- Replace the call site chain
- Keep behavior unchanged
- Keep names Swift-idiomatic

## Examples

### ❌ Flag: caller traverses nested structure
```swift
let city = company.employee(for: employeeID)?.address.city
```

### ✅ Prefer: owner-level query
```swift
let city = company.city(for: employeeID)
```

### ❌ High severity: async traversal chain
```swift
let marketingEmailsEnabled = try await accountService.currentAccount().owner.notificationSettings.marketingEmailsEnabled
```

### ✅ Prefer: intent-focused service API
```swift
let marketingEmailsEnabled = try await accountService.marketingEmailsEnabled()
```

## License

MIT
