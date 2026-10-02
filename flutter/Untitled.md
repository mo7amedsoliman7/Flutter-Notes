# Flutter Interview Topics

---

# The Three Trees: Widget → Element → RenderObject

> [!NOTE]
> Flutter maintains **three parallel trees** at all times. Confusing them is one of the most common Flutter misconceptions in interviews.

Every piece of UI in Flutter is described by three separate, synchronized trees — each with a different job.

---

## 🌳 1. Widget Tree — *"What should exist?"*

Widgets are **immutable blueprints**. Every `build()` call produces a fresh widget tree.

- Cheap to create and throw away
- Describe configuration only (color, text, padding…)
- Hold **no state** and do **no layout**

> [!IMPORTANT]
> Widgets are not the actual UI. They are descriptions of the UI — like a recipe, not the meal.

---

## 🌳 2. Element Tree — *"What actually exists?"*

Elements are the **live instances** that persist across rebuilds. Flutter diffs the new widget tree against the existing element tree to decide what to reuse, update, or replace.

| Element job | Detail |
|---|---|
| Lifecycle management | Owns the `State` object for `StatefulWidget` |
| Tree reconciliation | Diffs old vs new widget to decide: reuse or replace |
| BuildContext | Every `BuildContext` **is** an `Element` |

---

## 🌳 3. RenderObject Tree — *"How does it look and where does it go?"*

RenderObjects do the actual **layout and painting** work.

- Expensive to create — Flutter reuses them across rebuilds whenever possible
- Measure sizes, compute positions, produce paint commands
- Only updated when layout or paint properties actually change

---

## The Reconciliation Loop

```
build() produces a new Widget tree
         ↓
Element tree diffs it (cheap — compares runtimeType + Key)
         ↓
Only changed RenderObjects are updated (expensive layout avoided)
```

---

## 🎯 Key Interview Points

1. Widgets are **immutable** and cheap — recreated every `build()`.
2. Elements are **mutable** and persistent — they own `State` and `BuildContext`.
3. RenderObjects do **layout and paint** — expensive, reused aggressively.
4. `BuildContext` is not a separate object — it *is* the `Element`.
5. Flutter diffs at the Element layer, not the Widget layer.

---

# StatefulWidget Lifecycle

> [!NOTE]
> A `StatefulWidget` goes through a well-defined sequence of method calls from creation to disposal. Knowing the order and purpose of each is a core interview question.

---

## 🔁 The Lifecycle in Order

### 1. `createState()` — *"Attach a State object"*

Called once when the widget is first inserted into the tree. Returns the mutable `State` instance.

```dart
@override
State<MyWidget> createState() => _MyWidgetState();
```

### 2. `initState()` — *"One-time setup"*

Called once, immediately after the `State` is created. Safe to call `super.initState()` and initialize subscriptions, controllers, or data fetches here.

```dart
@override
void initState() {
  super.initState(); // always call first
  _controller = AnimationController(vsync: this);
  _fetchData();
}
```

> [!IMPORTANT]
> Never call `setState()` inside `initState()` — the widget hasn't mounted yet.

### 3. `didChangeDependencies()` — *"An InheritedWidget changed"*

Called after `initState()` and whenever an `InheritedWidget` this widget depends on changes. Safe to perform expensive work that depends on `context`.

### 4. `build()` — *"Describe the UI"*

Called every time state changes. Must be **pure** — no side effects, no heavy computation.

```dart
@override
Widget build(BuildContext context) {
  return Text(_counter.toString()); // just describe, don't compute
}
```

### 5. `didUpdateWidget()` — *"Parent rebuilt with new config"*

Called when the parent rebuilds and passes a new widget instance with different properties. Old widget is passed as parameter for comparison.

```dart
@override
void didUpdateWidget(MyWidget oldWidget) {
  super.didUpdateWidget(oldWidget);
  if (oldWidget.color != widget.color) _animate();
}
```

### 6. `deactivate()` — *"Temporarily removed from tree"*

Called when the State is removed from the tree (e.g., navigating away). May be reinserted before `dispose()` is called.

### 7. `dispose()` — *"Final cleanup"*

Called when the State is permanently removed. Cancel subscriptions, dispose controllers, remove listeners here.

```dart
@override
void dispose() {
  _controller.dispose(); // ✅ always clean up
  _subscription.cancel();
  super.dispose(); // always call last
}
```

---

## Lifecycle Summary Table

| Method | When called | Calls `setState()`? |
|---|---|---|
| `createState()` | Once, on insertion | ❌ |
| `initState()` | Once, after creation | ❌ |
| `didChangeDependencies()` | After init + on InheritedWidget change | ✅ safe |
| `build()` | Every rebuild | ❌ |
| `didUpdateWidget()` | Parent passes new config | ✅ safe |
| `deactivate()` | Removed from tree temporarily | ❌ |
| `dispose()` | Permanently removed | ❌ |

---

## 🎯 Key Interview Points

1. `initState()` → `didChangeDependencies()` → `build()` is the startup sequence.
2. `dispose()` is your only chance to clean up — memory leaks come from skipping it.
3. `didUpdateWidget()` fires when the parent rebuilds, **not** when your own state changes.
4. `build()` must be pure — side effects and heavy work belong in `initState()` or async methods.
5. `deactivate()` ≠ `dispose()` — the widget may come back after `deactivate()`.

---
---

# Keys in Flutter

> [!NOTE]
> Keys help Flutter **preserve State** across widget tree changes. Without them, Flutter may attach the wrong State to the wrong widget.

---

## 🔑 Why Keys Exist

Flutter identifies elements by their **position in the tree** and their **runtimeType**. When you reorder, add, or remove widgets of the same type at the same level, Flutter can match them to the wrong element.

```dart
// Swapping these without keys — Flutter keeps State in-place by position ❌
Column(children: [WidgetA(), WidgetB()])  →  Column(children: [WidgetB(), WidgetA()])
```

A Key tells Flutter: *"match this widget to the element with the same key, not the same position."*

---

## 🔑 Types of Keys

| Key type | Use case |
|---|---|
| `ValueKey(value)` | Stable, unique primitive (ID, string, int) |
| `ObjectKey(object)` | Unique object identity |
| `UniqueKey()` | Force a new element every rebuild (never reuse State) |
| `GlobalKey` | Access State or RenderObject from anywhere in the tree |
| `PageStorageKey` | Preserve scroll position across navigations |

---

## When You *Need* a Key

Only needed when **siblings of the same type** are reordered, inserted, or removed:

```dart
// ✅ Keys ensure correct State is preserved after reorder
ListView(children: [
  TodoItem(key: ValueKey(todo.id), todo: todo)
  for (todo in todos)
])
```

```dart
// ❌ UniqueKey() recreates State on every rebuild — avoid unless intentional
UniqueKey() // forces new element each time
```

---

## GlobalKey Caution

```dart
final key = GlobalKey<FormState>();
// Anywhere in the app:
key.currentState?.validate(); // ✅ access State from outside the tree
```

> [!IMPORTANT]
> `GlobalKey` is expensive — it registers in a global map. Use sparingly. Prefer passing callbacks or using state management instead.

---

## 🎯 Key Interview Points

1. Flutter matches elements by **position + runtimeType** by default — Keys override that.
2. Keys only matter for **same-type siblings** being reordered/added/removed.
3. `ValueKey` / `ObjectKey` → preserve State. `UniqueKey` → always destroy and recreate State.
4. `GlobalKey` lets you access State from anywhere — but it's expensive and should be a last resort.
5. Stateless widgets rarely need Keys; Stateful widgets with dynamic lists almost always do.

---
---

# Streams vs Futures

> [!NOTE]
> `Future` = **one** async value. `Stream` = **zero or more** async values over time. Choosing the wrong one is a common source of bugs.

---

## ⚡ 1. Future — *"Give me the result, once"*

```dart
Future<User> fetchUser(int id) async {
  final response = await http.get('/users/$id');
  return User.fromJson(response.body);
}
```

- Completes with a single value **or** an error
- Can't emit multiple values
- Use with `async/await` or `.then()/.catchError()`

---

## 🌊 2. Stream — *"Give me values as they arrive"*

```dart
Stream<int> countdown(int from) async* {
  for (int i = from; i >= 0; i--) {
    await Future.delayed(Duration(seconds: 1));
    yield i; // emits a value each second
  }
}
```

- Can emit **many values** over time
- Can be listened to with `await for`, `StreamBuilder`, or `.listen()`
- **Single-subscription** (default) vs **broadcast** streams

---

## Comparison Table

| | `Future<T>` | `Stream<T>` |
|---|---|---|
| Values emitted | Exactly one | Zero or more |
| Awaitable | ✅ `await` | ✅ `await for` |
| Flutter widget | `FutureBuilder` | `StreamBuilder` |
| Common use | HTTP request, file read | WebSocket, Firestore, events |
| Error handling | `.catchError()` / `try-catch` | `onError` / `try-catch` in `await for` |

---

## Single-Subscription vs Broadcast Streams

```dart
// Single-subscription — only one listener allowed
final stream = Stream.fromIterable([1, 2, 3]);
stream.listen(print); // ✅
stream.listen(print); // ❌ StateError: stream already listened to

// Broadcast — multiple listeners allowed
final broadcast = StreamController<int>.broadcast();
broadcast.stream.listen(print); // ✅
broadcast.stream.listen(print); // ✅
```

> [!IMPORTANT]
> Always cancel `StreamSubscription` in `dispose()` to avoid memory leaks.

```dart
late StreamSubscription _sub;

@override
void initState() {
  super.initState();
  _sub = myStream.listen(_handleData);
}

@override
void dispose() {
  _sub.cancel(); // ✅
  super.dispose();
}
```

---

## 🎯 Key Interview Points

1. `Future` = one value. `Stream` = sequence of values over time.
2. Default streams are **single-subscription** — only one `.listen()` allowed.
3. **Broadcast streams** allow multiple listeners — but late subscribers miss past events.
4. Always **cancel subscriptions** in `dispose()` — a common memory leak source.
5. `StreamBuilder` rebuilds the widget every time a new event arrives — don't put expensive logic in `builder`.

---