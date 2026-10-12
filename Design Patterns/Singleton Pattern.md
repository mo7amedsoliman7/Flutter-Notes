---
share_link: https://share.note.sx/ysmf3w9k#BvH75IEw9mgriDX/GB0+LSMqzbFe3fPZGmr2RiF67Jc
share_updated: 2026-10-12T07:12:18+03:00
---
# Design Patterns: Singleton Pattern

> **Singleton ensures a class has only one instance, and provides a global point of access to it.**

Category: **Creational Pattern**

## The Idea

Some things should exist exactly **once**: a settings manager, a logger, a connection pool, a cache, a database handle. Creating two of them can cause conflicting state, wasted resources, or inconsistent behavior.

Real-life analogy: a country has one president at a time. Everyone who needs "the president" asks the same global question and gets the same person.

## The Problem

```dart
final a = AppSettings();
final b = AppSettings();
// Two different objects! Change a theme in `a`, and `b` doesn't know.
```

A global variable doesn't fully solve it: nothing stops someone from creating another instance, and a global is created up front even if it's never used.

## The Classic Singleton (Recipe)

1. Make the **constructor private**, so nobody outside can call `new`.
2. Keep a **static field** holding the single instance.
3. Expose a **static accessor** that returns that instance.

### Implementation in Dart

Dart makes this short. A private named constructor plus a `static final` field does it all:

```dart
class AppSettings {
  // 1. Private constructor
  AppSettings._internal();

  // 2. The single instance (static fields are lazily initialized in Dart)
  static final AppSettings _instance = AppSettings._internal();

  // 3. Global access point (factory constructor)
  factory AppSettings() => _instance;

  String theme = 'light';
}

void main() {
  final a = AppSettings();
  final b = AppSettings();

  a.theme = 'dark';
  print(b.theme);          // dark
  print(identical(a, b));  // true
}
```

Alternative style with an explicit `instance` getter:

```dart
class Logger {
  Logger._();
  static final Logger instance = Logger._();

  void log(String message) => print('[LOG] $message');
}

Logger.instance.log('App started');
```

### Components

|Part|Role|
|---|---|
|**Private constructor**|Prevents outside code from creating instances|
|**Static instance field**|Holds the one object|
|**Static accessor** (`instance` / factory constructor)|The only way to get it|

```dart
						┌─────────────────────────────┐
						│                      Singleton                       │
						├─────────────────────────────┤
						│ - _instance : Singleton                      │  ← static
						│ - Singleton._()                                    │  ← private constructor
						├─────────────────────────────┤
						│ + instance : Singleton                       │  ← static, returns _instance
						│ + doSomething()                               │
						└─────────────────────────────┘
```

## Lazy vs Eager Creation

|Style|When created|Notes|
|---|---|---|
|**Eager**|At startup/class load|Simple; wasted if never used|
|**Lazy**|First time it's requested|Saves resources if it's never needed|

In Dart, `static final` fields are **lazy by default**: the initializer runs the first time the field is accessed. That gives you lazy creation for free.

If you need initialization that isn't instant (reading from disk, connecting to a database), make it explicit:

```dart
class Database {
  Database._();
  static Database? _instance;

  static Future<Database> getInstance() async {
    if (_instance != null) return _instance!;
    final db = Database._();
    await db._open();       // async setup
    return _instance = db;
  }

  Future<void> _open() async { /* open connection */ }
}
```

Careful: if two callers invoke `getInstance()` before the first one finishes `_open()`, you can end up with two instances. The fix is to cache the **Future** itself:

```dart
class Database {
  Database._();
  static Future<Database>? _future;

  static Future<Database> getInstance() => _future ??= _create();

  static Future<Database> _create() async {
    final db = Database._();
    await db._open();
    return db;
  }

  Future<void> _open() async { /* open connection */ }
}
```

## Threads, Isolates and Dart

The book spends a lot of time on **multithreading**: in Java, two threads can both see "no instance yet" and create two. Fixes there include `synchronized`, eager initialization, and double-checked locking.

Dart is different:

- Code in one **isolate** runs on a single thread, so the synchronous check-then-create above can't be interrupted. No locking needed.
- **Each isolate has its own memory.** A static singleton is **one per isolate**, not one per app. If you spawn a second isolate (via `Isolate.spawn` or `compute`), it gets its **own copy** of the "singleton" with its own state.
- The async case above (awaiting during creation) is the Dart equivalent of a race condition, and caching the `Future` solves it.

## Singleton in Dart and Flutter

Common uses:

- **Logger / analytics wrapper**
- **App configuration / environment**
- **API client / `Dio` instance** shared across the app
- **Local storage handle** (shared preferences wrapper, database)
- **Services registered in a service locator**, e.g. `get_it`:

```dart
final getIt = GetIt.instance;

void setup() {
  getIt.registerSingleton<Logger>(Logger());
  getIt.registerLazySingleton<ApiClient>(() => ApiClient());
}

// Anywhere:
final api = getIt<ApiClient>();
```

Here the **container** manages "only one instance", while your classes stay normal (public constructors). That keeps the benefits and avoids the main drawbacks below.

Flutter itself uses singleton-like objects, such as `WidgetsBinding.instance` and `PaintingBinding.instance.imageCache`.

## The Dark Side: Why Singleton Is Controversial

|Problem|Why it hurts|
|---|---|
|**Hidden dependencies**|A class calling `Logger.instance` doesn't show it needs a logger in its constructor|
|**Hard to test**|State leaks between tests; you can't easily swap in a fake|
|**Global mutable state**|Any code can change it, making bugs hard to trace|
|**Tight coupling**|Code depends on a concrete class (against **DIP**)|
|**Violates SRP**|The class manages its own lifecycle _and_ its real job|
|**Per-isolate surprise**|Not truly "one per app" in Dart (see above)|

Because of these, many developers treat Singleton as something to use carefully, not by default.

## Better Alternatives (Often)

**1. Dependency Injection.** Create one instance at the top and pass it in:

```dart
class OrderService {
  final Logger logger;          // dependency is explicit
  OrderService(this.logger);
}

void main() {
  final logger = Logger();      // one instance, created in one place
  runApp(MyApp(orderService: OrderService(logger)));
}
```

**2. Provider / Riverpod / get_it scopes.** Let a container hold the single instance:

```dart
runApp(
  Provider<Logger>(
    create: (_) => Logger(),
    child: const MyApp(),
  ),
);
```

**3. Make the singleton implement an interface**, so it can still be replaced in tests:

```dart
abstract class Logger {
  void log(String message);
}

class ConsoleLogger implements Logger {
  ConsoleLogger._();
  static final ConsoleLogger instance = ConsoleLogger._();

  @override
  void log(String message) => print('[LOG] $message');
}

// Code depends on Logger, not ConsoleLogger
class OrderService {
  final Logger logger;
  OrderService({Logger? logger}) : logger = logger ?? ConsoleLogger.instance;
}
```

## When to Use Singleton

- Exactly **one instance** is required for correctness (a hardware resource, a single connection pool, a shared cache).
- You need **controlled global access** to that one instance.
- Creating the object is **expensive** and it can be safely shared.

## When NOT to Use It

- You just want convenient global access to some data. Pass it in or use DI instead.
- The class holds mutable state many parts of the app change.
- You need to test the code that uses it.
- You're not sure you'll never need two.

## Pros and Cons

|Pros|Cons|
|---|---|
|✅ Guarantees a single instance|❌ Global state and hidden dependencies|
|✅ Controlled access point|❌ Harder to unit test|
|✅ Lazy creation saves resources|❌ Couples code to a concrete class|
|✅ Simple in Dart|❌ One _per isolate_, not per app|

## Relation to SOLID

|Principle|Connection|
|---|---|
|**SRP**|Singleton mixes "my job" with "control my own instance count", which is a mild violation|
|**DIP**|Direct `Class.instance` calls couple clients to a concrete class; code to an interface or inject it instead|
|**OCP / LSP**|Hard to extend or substitute a singleton unless it implements an interface|

## Singleton vs Factory vs Static Class
|Pattern|Category|Intent (one line)|Key idea|Dart / Flutter example|
|---|---|---|---|---|
|**Strategy**|Behavioral|Swap one algorithm for another at runtime|Encapsulate what varies; favor composition|Payment methods, validators, sort comparators|
|**Observer**|Behavioral|Notify many objects when one object changes|Loose coupling between subject and observers|`ChangeNotifier`, `Stream`, Provider/Bloc|
|**Decorator**|Structural|Add behavior by wrapping an object|Open for extension, closed for modification|Wrapper widgets, `http` clients with logging/retry|
|**Factory** (Simple, Method, Abstract)|Creational|Hide which concrete class gets created|Depend on abstractions, not concrete classes|`factory` constructors, `fromJson`, platform widget factories|
|**Singleton**|Creational|Exactly one instance with a global access point|Controlled access, but watch hidden global state|`get_it` singletons, logger (one per isolate)|

## Summary

> Singleton ensures one instance and a single access point. In Dart it's easy (private constructor + `static final`), but treat it as a deliberate choice: prefer dependency injection or a service locator for sharing objects, and remember each isolate gets its own "singleton".