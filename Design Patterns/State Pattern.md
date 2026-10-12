---
share_link: https://share.note.sx/cb2xn5fv#0b6VV/zPCirLXSkrp9nF8KAYI4YomDJhqt6Dob8AqD8
share_updated: 2026-10-12T07:12:12+03:00
---


>  In one line **State lets an object change its behavior when its internal state changes. The object will appear to change its class.**

**Category:** Behavioral pattern 
**Key idea:** Strategy and State were "twins separated at birth": same structure, different intent.

---
## The Idea

Instead of using `if / else` or `switch` blocks everywhere to check the current state, move each state's behavior into its own class. The main class (the **Context**) holds a reference to the current state object and **delegates** all state-dependent work to it. To change state, the context simply switches to another state object.

---

## The Problem

A media player with three states: stopped, playing, paused.

```dart
class MediaPlayer {
  String state = 'stopped';

  void play() {
    if (state == 'stopped') {
      print('Start playing');
      state = 'playing';
    } else if (state == 'paused') {
      print('Resume playing');
      state = 'playing';
    } else if (state == 'playing') {
      print('Already playing');
    }
  }

  void pause() {
    if (state == 'playing') {
      print('Paused');
      state = 'paused';
    } else {
      print('Cannot pause');
    }
  }

  // stop() has the same kind of conditions...
}
```

**Drawbacks:**

- **Conditionals everywhere:** Every method checks the state with `if/else`.
- **Breaks OCP:** Adding a new state (e.g., `buffering`) means editing every method.
- **Hard to follow:** Transition rules are scattered across the class.
- **Error-prone:** State is a plain string, so typos fail silently at runtime.

---

## The Solution

Make each state a class that implements a common interface. Each state knows what to do for every action and which state comes next.

### Components

|Component|Role|In this example|
|---|---|---|
|**State** (interface)|Declares the actions that depend on state|`PlayerState`|
|**Concrete States**|Implement behavior for one state and trigger transitions|`StoppedState`, `PlayingState`, `PausedState`|
|**Context**|Holds the current state and delegates actions to it|`MediaPlayer`|

### State diagram

![[State Pattern.png]]
Simplified rules:

- **Stopped:** `play()` → Playing
- **Playing:** `pause()` → Paused, `stop()` → Stopped
- **Paused:** `play()` → Playing, `stop()` → Stopped

### Implementation in Dart

```dart
// 1. The State (interface)
abstract class PlayerState {
  void play(MediaPlayer player);
  void pause(MediaPlayer player);
  void stop(MediaPlayer player);
}

// 2. Concrete states
class StoppedState implements PlayerState {
  @override
  void play(MediaPlayer player) {
    print('Start playing');
    player.setState(PlayingState());
  }

  @override
  void pause(MediaPlayer player) => print('Cannot pause: player is stopped');

  @override
  void stop(MediaPlayer player) => print('Already stopped');
}

class PlayingState implements PlayerState {
  @override
  void play(MediaPlayer player) => print('Already playing');

  @override
  void pause(MediaPlayer player) {
    print('Paused');
    player.setState(PausedState());
  }

  @override
  void stop(MediaPlayer player) {
    print('Stopped');
    player.setState(StoppedState());
  }
}

class PausedState implements PlayerState {
  @override
  void play(MediaPlayer player) {
    print('Resume playing');
    player.setState(PlayingState());
  }

  @override
  void pause(MediaPlayer player) => print('Already paused');

  @override
  void stop(MediaPlayer player) {
    print('Stopped');
    player.setState(StoppedState());
  }
}

// 3. The Context
class MediaPlayer {
  PlayerState _state = StoppedState(); // initial state

  void setState(PlayerState state) => _state = state;

  // Delegate everything to the current state
  void play() => _state.play(this);
  void pause() => _state.pause(this);
  void stop() => _state.stop(this);
}
```

### Usage

```dart
void main() {
  final player = MediaPlayer();

  player.play();  // Start playing
  player.pause(); // Paused
  player.play();  // Resume playing
  player.stop();  // Stopped
  player.pause(); // Cannot pause: player is stopped
}
```

### Adding a new state

Adding **Buffering** = one new class plus the transitions that lead to it. The context's methods stay the same, and no `if/else` chains need editing.

---

## State vs Strategy

Their class diagrams look almost identical (a context that delegates to an interface), but their **intent** is different.

|Aspect|State|Strategy|
|---|---|---|
|**Purpose**|Behavior changes as internal state changes|Swap one algorithm for another|
|**Who changes it?**|The states/context change it **automatically**|The **client** picks it from outside|
|**Awareness**|States often know about each other (transitions)|Strategies are independent of each other|
|**Lifecycle**|Context moves through states over time|Context usually keeps one strategy until changed|
|**Key idea**|"I behave differently depending on my state."|"Do this task in whichever way I choose."|

> [!tip] Quick test Does the object switch behavior **by itself** as it runs? → **State**. Does the outside code **choose** how a task is done? → **Strategy**.

---

## Dart Tip: Enums and Sealed Classes

For simple cases, Dart 3 can model states without the full pattern.

```dart
sealed class LoadState {}
class Loading extends LoadState {}
class Success extends LoadState { final String data; Success(this.data); }
class Failure extends LoadState { final String message; Failure(this.message); }

Widget build(LoadState state) => switch (state) {
  Loading()    => const CircularProgressIndicator(),
  Success(:final data)    => Text(data),
  Failure(:final message) => Text('Error: $message'),
};
```

This is a related idea (a value that is one of several states, with exhaustive checks), but the behavior stays outside the state in the `switch`. The **State pattern** puts the behavior **inside** each state class, which pays off when each state has many actions and complex transitions.

---

## Real-World Examples

- **Media player:** stopped, playing, paused, buffering.
- **Order lifecycle:** pending, paid, shipped, delivered, cancelled.
- **Document workflow:** draft, in review, published.
- **Vending / gumball machine:** no coin, has coin, sold, sold out.
- **Network connection:** connecting, connected, disconnected, retrying.
- **Traffic light:** red, green, yellow.

---

## OO Principles and SOLID

- **Encapsulate what varies:** State-specific behavior is pulled into separate classes.
- **Favor composition over inheritance:** The context _has a_ state object and delegates to it.
- **Program to interfaces:** The context depends on `PlayerState`, not on concrete states.
- **OCP:** Add new states without rewriting the context's methods.
- **SRP:** Each state class handles the behavior of one state only.
- **DIP:** The context depends on the state abstraction.

---

## When to Use State

- An object's behavior **depends on its state** and changes at runtime.
- You have large `if/else` or `switch` blocks that check the current state in many methods.
- The object has **many states** with clear transition rules.
- You want to add new states without modifying existing ones much.

## When NOT to Use

- There are only two or three states with little behavior. An enum and a `switch` is simpler.
- The states rarely change and transitions are trivial.

---

## Pros and Cons

**Pros**

- **No big conditionals:** Polymorphism replaces `if/else` and `switch` on state.
- **Single Responsibility:** Each state's logic lives in one place.
- **Open/Closed:** New states can be added with minimal changes.
- **Clear transitions:** Rules for moving between states are explicit.

**Cons**

- **More classes:** Every state is its own class.
- **Coupled states:** Concrete states often know about the next state, so they depend on each other.
- **Overkill for simple cases:** Not worth it for a few trivial states.
- **Duplicated behavior:** Similar actions (like "Already playing") repeat across states unless you add a base class.

---

## Quick Recap

- Each state is a class implementing the same interface.
- The context **has a** current state and **delegates** to it.
- States trigger transitions by calling `context.setState(...)`.
- Behavior changes automatically as state changes, with no `if/else` chains.
- **State** = changes itself over time. **Strategy** = chosen by the client.