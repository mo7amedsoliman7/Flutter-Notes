---
share_link: https://share.note.sx/h3lthb3y#GUyRjbwip7CJ8lmykvLiz1N9bXe45+hx8OfRFPktm3Y
share_updated: 2026-10-12T07:12:49+03:00
---
# Design Patterns: #3 Decorator Pattern

> **Decorator attaches additional responsibilities to an object dynamically. It gives you a flexible alternative to subclassing for extending functionality.**

Category: **Structural Pattern**

## The Idea

You have an object, and you want to add behavior to it (extra features, extra cost, extra formatting) **without changing its class** and **without creating a subclass for every combination**. So you **wrap** the object inside another object that has the same interface, adds its own behavior, and delegates the rest to the wrapped object. You can stack as many wrappers as you like.

Real-life analogy: ordering a coffee. You start with an espresso, then add milk, then mocha, then whip. Each addition wraps the drink and adds to the description and the price. The result is still "a drink".

## The Problem

A coffee shop sells beverages with optional add-ons (milk, mocha, soy, whip). Using inheritance:

```dart
abstract class Beverage {
  String description = 'Unknown beverage';
  double cost();
}

class Espresso extends Beverage { /* ... */ }
class EspressoWithMilk extends Beverage { /* ... */ }
class EspressoWithMilkAndMocha extends Beverage { /* ... */ }
class DarkRoastWithMochaAndWhip extends Beverage { /* ... */ }
// ...and so on for every combination
```

**Drawbacks:**
- **Class explosion**: every combination of base drink and add-ons needs its own class.
- A price change for one add-on means editing many classes.
- New add-on = new classes for all combinations.
- Customers wanting "double mocha" breaks the design.
- Behavior is fixed at compile time.

Putting boolean flags (`hasMilk`, `hasMocha`, ...) in the superclass is also messy: subclasses inherit things that don't apply to them, and every new add-on changes the superclass.

## Design Principle: Open-Closed

> **Classes should be open for extension, but closed for modification.**

Decorator lets you extend behavior by adding new classes, without touching the code that already works and has been tested.

## The Solution: Decorator Pattern

### Components

| Component | Role |
|---|---|
| **Component** (interface/abstract) | The common type for both plain objects and decorators |
| **Concrete Component** | The basic object you want to add behavior to |
| **Decorator** (abstract) | Implements the same Component type **and holds a reference** to a Component |
| **Concrete Decorators** | Add their own behavior before/after delegating to the wrapped component |
![[Decorator Pattern.png]]
Key point: the decorator **is-a** Beverage (same type, so it can be used anywhere a Beverage is expected) and **has-a** Beverage (the thing it wraps).

### Implementation in Dart

```dart
// Component
abstract class Beverage {
  String getDescription();
  double cost();
}

// Concrete Components
class Espresso implements Beverage {
  @override
  String getDescription() => 'Espresso';
  @override
  double cost() => 1.99;
}

class HouseBlend implements Beverage {
  @override
  String getDescription() => 'House Blend Coffee';
  @override
  double cost() => 0.89;
}

// Decorator (abstract)
abstract class CondimentDecorator implements Beverage {
  final Beverage beverage; // the wrapped object
  CondimentDecorator(this.beverage);
}

// Concrete Decorators
class Milk extends CondimentDecorator {
  Milk(super.beverage);

  @override
  String getDescription() => '${beverage.getDescription()}, Milk';
  @override
  double cost() => beverage.cost() + 0.10;
}

class Mocha extends CondimentDecorator {
  Mocha(super.beverage);

  @override
  String getDescription() => '${beverage.getDescription()}, Mocha';
  @override
  double cost() => beverage.cost() + 0.20;
}

class Whip extends CondimentDecorator {
  Whip(super.beverage);

  @override
  String getDescription() => '${beverage.getDescription()}, Whip';
  @override
  double cost() => beverage.cost() + 0.15;
}
```

Usage:

```dart
void main() {
  Beverage drink = Espresso();
  print('${drink.getDescription()} \$${drink.cost()}');
  // Espresso $1.99

  drink = Mocha(drink);
  drink = Mocha(drink);   // double mocha, no new class needed
  drink = Whip(drink);
  print('${drink.getDescription()} \$${drink.cost()}');
  // Espresso, Mocha, Mocha, Whip $2.54
}
```

Each wrapper calls the wrapped object and adds its own part. New add-on? One new class; nothing existing changes.

## Decorator in Dart and Flutter

- **Widgets are decorators in spirit**: `Padding(child: ...)`, `Opacity(child: ...)`, `Align(child: ...)`, `DecoratedBox(child: ...)`. Each wraps a child widget and adds a behavior or visual effect, and the result is still a `Widget`.
- **Streams and IO**: wrapping a byte stream with a transformer or codec (e.g. `utf8.decoder`, `gzip`) adds behavior around the original stream.
- **Repository / service wrappers**: wrap a repository with a `CachedUserRepository` or `LoggingUserRepository` that implements the same interface and delegates to the real one.

```dart
abstract class UserRepository {
  Future<User> getUser(String id);
}

class ApiUserRepository implements UserRepository {
  @override
  Future<User> getUser(String id) async => /* HTTP call */ User(id);
}

// Decorator: adds logging
class LoggingUserRepository implements UserRepository {
  final UserRepository inner;
  LoggingUserRepository(this.inner);

  @override
  Future<User> getUser(String id) async {
    print('Fetching user $id');
    final user = await inner.getUser(id);
    print('Fetched user $id');
    return user;
  }
}

// Decorator: adds caching
class CachedUserRepository implements UserRepository {
  final UserRepository inner;
  final Map<String, User> _cache = {};
  CachedUserRepository(this.inner);

  @override
  Future<User> getUser(String id) async =>
      _cache[id] ??= await inner.getUser(id);
}

// Stack them
final repo = LoggingUserRepository(CachedUserRepository(ApiUserRepository()));
```

## When to Use Decorator

- You want to add responsibilities to objects **dynamically** and **transparently**, without affecting other objects.
- Extension by subclassing would cause a **class explosion** or isn't possible.
- Behaviors should be **combinable in any order** at runtime.
- You want to follow the **Open-Closed Principle**.

## Pros and Cons

| Pros | Cons |
|---|---|
| ✅ Extend behavior without modifying existing classes (**OCP**) | ❌ Many small, similar classes |
| ✅ Combine behaviors flexibly at runtime | ❌ Code that depends on a specific concrete type may break (the wrapper hides it) |
| ✅ Avoids a class explosion from subclassing | ❌ Creating a fully decorated object takes several lines of wrapping (factories or builders help) |
| ✅ Each decorator has a single responsibility (**SRP**) | ❌ The order of decorators can change results and be hard to debug |

## Relation to SOLID

| Principle | Connection |
|---|---|
| **OCP** | Add behavior by adding classes, not editing old ones |
| **SRP** | Each decorator does one small job |
| **LSP** | A decorator must be substitutable for the component it wraps |
| **DIP** | Clients depend on the Component interface, not concrete classes |

## Decorator vs Inheritance vs Strategy

| | Decorator | Inheritance | Strategy |
|---|---|---|---|
| **When extended** | Runtime | Compile time | Runtime |
| **Mechanism** | Wrap the object | Subclass it | Swap an algorithm |
| **Combine multiple?** | Yes, stack them | Class explosion | Usually one at a time |
| **Changes** | The object's *skin* (adds behavior around it) | The object's *class* | The object's *guts* (how it does something) |

## Summary

> Instead of subclassing for every combination, wrap the object. Decorators share the same type as what they wrap, so you can stack as many as you want and the rest of the code never knows the difference.