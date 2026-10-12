---
share_link: https://share.note.sx/jfeksdto#/0p1DyhQfYONJWv+yyoRPKmfdKfE2bzupcGJbILvuVY
share_updated: 2026-10-10T01:19:44+03:00
---
> **Observer defines a one-to-many dependency between objects, so when one object (the Subject) changes state, all its dependents (the Observers) are notified and updated automatically.**
   
   Category: **Behavioral Pattern**
## The Idea

One object holds some state that changes. Other objects care about that state and want to react when it changes. Instead of those objects constantly _asking_ ("did it change yet?"), they **subscribe** once, and the subject **notifies** them whenever something happens.

---
## The Problem

A weather station has new measurements (temperature, humidity, pressure), and several displays must update: current conditions, statistics, forecast.

```dart
class WeatherData {
  final CurrentConditionsDisplay current;
  final StatisticsDisplay stats;
  final ForecastDisplay forecast;

  WeatherData(this.current, this.stats, this.forecast);

  void measurementsChanged(double temp, double humidity, double pressure) {
    // Hard-wired to concrete displays
    current.update(temp, humidity, pressure);
    stats.update(temp, humidity, pressure);
    forecast.update(temp, humidity, pressure);
    // New display? Edit this class again.
  }
}
```

**Drawbacks:**
- `WeatherData` is coupled to every concrete display.
- Adding or removing a display means modifying `WeatherData`.
- Displays can't be added or removed at runtime.
- You're coding against implementations instead of an interface.

## The Solution: Observer Pattern

### Components

|Component|Role|
|---|---|
|**Subject** (interface)|Lets observers register, unregister, and be notified|
|**Concrete Subject**|Holds the state; calls `notifyObservers()` when it changes|
|**Observer** (interface)|Defines the `update()` method the subject calls|
|**Concrete Observers**|React to the update in their own way|

![[Observer Pattern.png]]
### Implementation in Dart

```dart
// Observer
abstract class Observer {
  void update(double temp, double humidity, double pressure);
}

// Subject
abstract class Subject {
  void registerObserver(Observer o);
  void removeObserver(Observer o);
  void notifyObservers();
}

// Concrete Subject
class WeatherData implements Subject {
  final List<Observer> _observers = [];
  double _temp = 0, _humidity = 0, _pressure = 0;

  @override
  void registerObserver(Observer o) => _observers.add(o);

  @override
  void removeObserver(Observer o) => _observers.remove(o);

  @override
  void notifyObservers() {
    for (final o in _observers) {
      o.update(_temp, _humidity, _pressure);
    }
  }

  void setMeasurements(double temp, double humidity, double pressure) {
    _temp = temp;
    _humidity = humidity;
    _pressure = pressure;
    notifyObservers();
  }
}

// Concrete Observer
class CurrentConditionsDisplay implements Observer {
  CurrentConditionsDisplay(Subject weatherData) {
    weatherData.registerObserver(this);
  }

  @override
  void update(double temp, double humidity, double pressure) {
    print('Current: ${temp}°C, $humidity% humidity');
  }
}
```

Usage:
```dart
void main() {
  final weather = WeatherData();
  final display = CurrentConditionsDisplay(weather);

  weather.setMeasurements(27, 65, 1013);  // Current: 27.0°C, 65.0% humidity

  weather.removeObserver(display);        // no longer notified
}
```

> [!Notes]
> Adding a new display = one new class that registers itself. `WeatherData` never changes.

---
## Push vs Pull

|Style|How it works|Trade-off|
|---|---|---|
|**Push**|Subject sends the data in `update(...)`|Simple, but every observer gets all the data, even what it doesn't need; changing the data set breaks every observer's signature|
|**Pull**|Subject sends a simple "something changed" signal; observers read what they need from the subject's getters|More flexible; observers take only what they need|

---
## Observer in Dart and Flutter

Dart and Flutter have the pattern built in:

- **`Stream` / `StreamController`**: subscribers `listen()` to a stream of events.
- **`ChangeNotifier` + `addListener()`**: the classic Subject/Observer pair. `notifyListeners()` is the notify call.
- **`ValueNotifier` / `ValueListenableBuilder`**: a notifier holding one value, with a widget that rebuilds on change.
- **`Provider`, `Riverpod`, `Bloc`**: state management libraries built on observing state changes.
- **`setState`-driven UIs and `AnimationController` listeners**: same idea.

![[Observer Pattern Book.png]]

- Bolc
![[Observer Pattern Bloc.png]]

- Cubit
![[Observer Pattern Cubit.png]]
---
## Design Principle: Loose Coupling

> **Strive for loosely coupled designs between objects that interact.**

Subject and observers can interact, but they know very little about each other:

- The subject only knows that observers implement the `Observer` interface.
- Observers can be added or removed at any time, even at runtime.
- You never modify the subject to add a new kind of observer.
- Subject and observers can be reused and changed independently.
---
## When to Use Observer

- A change in one object requires updating others, and you don't know how many or which ones.
- You want objects to be notified without tightly coupling them.
- Parts of a UI must react to data/model changes (event handling, reactive UIs).
- You need dynamic subscribe/unsubscribe at runtime.

## Pros and Cons

**Pros**

- **Loose coupling:** The subject only knows about the `Observer` interface, not the concrete classes behind it.
- **Open/Closed (OCP):** You can add or remove observers without changing the subject's code.
- **Runtime flexibility:** Relationships between subject and observers can be set up and changed while the program runs.
- **Broadcast communication:** One state change automatically reaches every interested observer.

**Cons**

- **No guaranteed order:** Observers may be notified in an unpredictable order, so they shouldn't depend on each other.
- **Memory leaks:** Observers that forget to unsubscribe stay referenced by the subject and can't be garbage collected.
- **Hard to debug:** Cascading updates (one observer triggering another change) make the flow difficult to trace.
- **Performance cost:** Frequent notifications to many observers can slow the system down, especially if updates are expensive.
---
## Relation to SOLID

- **OCP (Open/Closed):** you can add a new observer without modifying the subject. In Flutter, you can add another widget that listens to a `ChangeNotifier` without touching the notifier.
- **DIP (Dependency Inversion):** the subject depends on the `Observer` interface, not on concrete displays. `ChangeNotifier` only knows about `VoidCallback` listeners, never about your widgets.
- **SRP (Single Responsibility):** the subject manages state, and each observer handles its own reaction. A cubit holds the logic and `BlocBuilder` only draws the UI.
- **ISP (Interface Segregation):** `Observer` is a small interface with a single method, `update()`. Observers aren't forced to implement anything they don't use.
---
## Observer vs Strategy

**Observer**

- **Purpose:** Notify many objects automatically when a subject's state changes.
- **Direction:** The subject pushes updates out to many observers.
- **Relationship:** One-to-many. A single subject can have any number of observers at once.
- **Key idea:** "Tell me when something changes."

**Strategy**

- **Purpose:** Swap one algorithm or behavior for another at runtime.
- **Direction:** The context delegates work to a single strategy.
- **Relationship:** One-to-one. The context uses only one strategy at a time, though it can switch between them.
- **Key idea:** "Do this task in whichever way I choose."

**Main difference:** Observer is about _communication_ (reacting to changes), while Strategy is about _behavior_ (choosing how something gets done)


> [!TIP]
>**OO Principles (so far):**
>1. **Encapsulate what varies:** Isolate the parts that change from the parts that stay the same.
>2. **Favor composition over inheritance:** Prefer "has-a" over "is-a" for more flexibility, including runtime changes.
>3. **Program to interfaces, not implementations:** Depend on abstractions, not concrete classes.
>4. **Strive for loosely coupled designs between objects that interact:** _(newest)_ Objects should know as little about each other as possible.

