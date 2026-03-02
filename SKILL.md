# Law of Demeter - Swift Skill

A comprehensive guide to applying the Law of Demeter (Principle of Least Knowledge) in Swift 6+ development.

## Overview

The Law of Demeter is a design principle that promotes loose coupling by limiting an object's knowledge of other objects. This skill provides Swift-specific guidance on applying this principle effectively.

## What's Inside

**SKILL.md** - A complete guide covering:
- Core principles of the Law of Demeter
- Swift-specific naming conventions (properties over getters)
- Access control patterns using `private`, `private(set)`, etc.
- Protocol-oriented design for clear interfaces
- Real-world iOS/Swift examples
- Swift 6+ concurrency considerations with actors
- Testing benefits and patterns

## Key Principles

**Only talk to your immediate friends:**
- Call methods on `self`
- Call methods on parameters
- Call methods on objects you create
- Call methods on direct components
- Never chain through multiple objects

## Swift-Specific Features

This guide adapts the Law of Demeter specifically for Swift, including:
- Using computed properties instead of getter methods
- Leveraging Swift's access control (`private`, `fileprivate`, `internal`, `public`)
- Protocol-oriented design patterns
- Optional chaining considerations
- Swift 6 strict concurrency and actor isolation

## Examples

```swift
// ❌ VIOLATION: Chaining through objects
let city = user.profile.address.city

// ✅ CORRECT: Ask the object directly
let city = user.city
```

## License

MIT License - See LICENSE file for details
