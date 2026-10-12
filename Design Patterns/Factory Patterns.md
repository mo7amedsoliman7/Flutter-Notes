---
share_link: https://share.note.sx/mwbfollj#esrVslin5nQz3p47xsnaP8oB8tLOqODsT0JI1gSN4nk
share_updated: 2026-10-12T07:12:34+03:00
---
# Design Patterns: Factory Patterns

> **Factory Method** defines an interface for creating an object, but lets subclasses decide which class to instantiate. It lets a class defer instantiation to subclasses.
> 
> **Abstract Factory** provides an interface for creating families of related or dependent objects without specifying their concrete classes.

Category: **Creational Patterns**

"Factory" is really a small family: **Simple Factory** (an idiom), **Factory Method**, and **Abstract Factory**. All three move the `new` out of the code that uses the objects.

## The Problem: `new` Means Concrete

Every time you write `SomeConcreteClass()`, your code is tied to that concrete class.

```dart
class PizzaStore {
  Pizza orderPizza(String type) {
    Pizza pizza;

    // This part changes whenever the menu changes
    if (type == 'cheese') {
      pizza = CheesePizza();
    } else if (type == 'pepperoni') {
      pizza = PepperoniPizza();
    } else if (type == 'veggie') {
      pizza = VeggiePizza();
    } else {
      throw ArgumentError('Unknown pizza: $type');
    }

    // This part stays the same
    pizza.prepare();
    pizza.bake();
    pizza.cut();
    pizza.box();
    return pizza;
  }
}
```

**Drawbacks:**

- The creation `if/else` changes whenever the menu changes, but the prepare/bake/cut/box part never does. They're mixed together.
- Violates the **Open-Closed Principle**: adding a pizza means editing `orderPizza`.
- The store is coupled to every concrete pizza class.
- Every other class that needs pizzas repeats the same `if/else`.

**Principle at play:** identify what varies (which pizza gets created) and encapsulate it.

## Level 1: Simple Factory (Idiom)

Move the creation code into its own class. Not an official GoF pattern, but a useful first step.

```dart
class SimplePizzaFactory {
  Pizza createPizza(String type) {
    switch (type) {
      case 'cheese':
        return CheesePizza();
      case 'pepperoni':
        return PepperoniPizza();
      case 'veggie':
        return VeggiePizza();
      default:
        throw ArgumentError('Unknown pizza: $type');
    }
  }
}

class PizzaStore {
  final SimplePizzaFactory factory;   // has a
  PizzaStore(this.factory);

  Pizza orderPizza(String type) {
    final pizza = factory.createPizza(type); // creation delegated
    pizza.prepare();
    pizza.bake();
    pizza.cut();
    pizza.box();
    return pizza;
  }
}
```

`PizzaStore` no longer knows concrete pizzas, and many clients can reuse the factory. But the factory itself still has the `switch`, and it's a single fixed way of creating.

## Level 2: Factory Method Pattern

Now the store franchises: New York style, Chicago style, each with different pizzas. Make `createPizza` an **abstract method** that subclasses implement.

```dart
abstract class PizzaStore {
  // The Factory Method: subclasses decide what to create
  Pizza createPizza(String type);

  // The rest of the process is shared
  Pizza orderPizza(String type) {
    final pizza = createPizza(type);
    pizza.prepare();
    pizza.bake();
    pizza.cut();
    pizza.box();
    return pizza;
  }
}

class NYPizzaStore extends PizzaStore {
  @override
  Pizza createPizza(String type) {
    switch (type) {
      case 'cheese':
        return NYStyleCheesePizza();
      case 'veggie':
        return NYStyleVeggiePizza();
      default:
        throw ArgumentError('Unknown pizza: $type');
    }
  }
}

class ChicagoPizzaStore extends PizzaStore {
  @override
  Pizza createPizza(String type) {
    switch (type) {
      case 'cheese':
        return ChicagoStyleCheesePizza();
      case 'veggie':
        return ChicagoStyleVeggiePizza();
      default:
        throw ArgumentError('Unknown pizza: $type');
    }
  }
}
```

Usage:

```dart
final PizzaStore store = NYPizzaStore();
final pizza = store.orderPizza('cheese'); // NY-style cheese pizza
```

`orderPizza` (the **Creator**) never knows which concrete pizza (the **Product**) it's working with. The subclass decides.

### Components

|Component|Role|
|---|---|
|**Product** (`Pizza`)|The interface/abstract type of what is created|
|**Concrete Products**|`NYStyleCheesePizza`, `ChicagoStyleVeggiePizza`, ...|
|**Creator** (`PizzaStore`)|Declares the factory method; contains logic that uses the product|
|**Concrete Creators**|Override the factory method to return a specific product|

![[Factory Patterns.png]]
Two parallel hierarchies: creators and products. Each concrete creator knows its matching concrete products.

## Design Principle: Dependency Inversion

> **Depend upon abstractions. Do not depend upon concrete classes.**

Before: `PizzaStore` (high level) depended on every concrete pizza (low level). After: both depend on the `Pizza` abstraction. The concrete classes are now "inverted" to depend on the abstraction too.

Handy guidelines for staying on track:

- No variable should hold a reference to a concrete class (`new` means concrete, so route it through a factory).
- No class should derive from a concrete class.
- No method should override an implemented method of its base class.

They're guidelines, not laws; you'll break them sometimes, but know when you do.

## Level 3: Abstract Factory Pattern

Pizzas in different regions use different **families of ingredients**: dough, sauce, cheese. A New York pizza must use thin crust dough with marinara and reggiano; a Chicago pizza uses thick crust with plum tomato sauce and mozzarella. We need to create _families_ of related objects, without mixing them.

```dart
// Abstract products
abstract class Dough {}
abstract class Sauce {}
abstract class Cheese {}

// Concrete products
class ThinCrustDough implements Dough {}
class ThickCrustDough implements Dough {}
class MarinaraSauce implements Sauce {}
class PlumTomatoSauce implements Sauce {}
class ReggianoCheese implements Cheese {}
class MozzarellaCheese implements Cheese {}

// Abstract Factory: one method per product in the family
abstract class PizzaIngredientFactory {
  Dough createDough();
  Sauce createSauce();
  Cheese createCheese();
}

// Concrete factories: each produces a consistent family
class NYIngredientFactory implements PizzaIngredientFactory {
  @override
  Dough createDough() => ThinCrustDough();
  @override
  Sauce createSauce() => MarinaraSauce();
  @override
  Cheese createCheese() => ReggianoCheese();
}

class ChicagoIngredientFactory implements PizzaIngredientFactory {
  @override
  Dough createDough() => ThickCrustDough();
  @override
  Sauce createSauce() => PlumTomatoSauce();
  @override
  Cheese createCheese() => MozzarellaCheese();
}
```

The pizza now receives the factory and asks it for ingredients:

```dart
abstract class Pizza {
  late Dough dough;
  late Sauce sauce;
  late Cheese cheese;

  void prepare(); // collects ingredients from the factory
  void bake() => print('Bake for 25 minutes');
}

class CheesePizza extends Pizza {
  final PizzaIngredientFactory ingredients;
  CheesePizza(this.ingredients);

  @override
  void prepare() {
    dough = ingredients.createDough();
    sauce = ingredients.createSauce();
    cheese = ingredients.createCheese();
  }
}

// The store (a Factory Method) picks the ingredient factory
class NYPizzaStore extends PizzaStore {
  @override
  Pizza createPizza(String type) {
    final ingredients = NYIngredientFactory();
    return CheesePizza(ingredients); // same pizza class, regional ingredients
  }
}
```

Notice that Abstract Factory often uses Factory Methods inside (`createDough()` etc. are factory methods).

### Components

|Component|Role|
|---|---|
|**Abstract Factory**|Interface with a creation method for each product in the family|
|**Concrete Factories**|Create a consistent set of related products|
|**Abstract Products**|Interfaces for each kind of product|
|**Client**|Uses only the abstract factory and abstract products|

## Factory Method vs Abstract Factory

||Factory Method|Abstract Factory|
|---|---|---|
|**Creates**|**One** product|A **family** of related products|
|**Mechanism**|Inheritance: a subclass overrides the method|Composition: the client receives a factory object|
|**Extending**|Add a new Creator subclass|Add a new Concrete Factory; but adding a new _product type_ means changing the interface and all factories|
|**Use when**|A class can't anticipate which class to create|Products must be used together and be consistent|

## Factory in Dart and Flutter

Dart has **language support** for factories:

**`factory` constructors:** return an existing instance, a subtype, or cache objects.

```dart
abstract class Shape {
  factory Shape(String type) {
    switch (type) {
      case 'circle':
        return Circle();
      case 'square':
        return Square();
      default:
        throw ArgumentError('Unknown shape: $type');
    }
  }
  double area();
}

final s = Shape('circle'); // looks like constructing Shape, returns a Circle
```

**Parsing JSON into the right subtype:**

```dart
abstract class Message {
  factory Message.fromJson(Map<String, dynamic> json) {
    switch (json['type']) {
      case 'text':
        return TextMessage(json['body']);
      case 'image':
        return ImageMessage(json['url']);
      default:
        throw FormatException('Unknown type: ${json['type']}');
    }
  }
}
```

**Platform-adaptive widgets** (Abstract Factory idea):

```dart
abstract class WidgetFactory {
  Widget createButton(String label, VoidCallback onTap);
  Widget createSwitch(bool value, ValueChanged<bool> onChanged);
}

class MaterialWidgetFactory implements WidgetFactory {
  @override
  Widget createButton(String label, VoidCallback onTap) =>
      ElevatedButton(onPressed: onTap, child: Text(label));
  @override
  Widget createSwitch(bool value, ValueChanged<bool> onChanged) =>
      Switch(value: value, onChanged: onChanged);
}

class CupertinoWidgetFactory implements WidgetFactory {
  @override
  Widget createButton(String label, VoidCallback onTap) =>
      CupertinoButton(onPressed: onTap, child: Text(label));
  @override
  Widget createSwitch(bool value, ValueChanged<bool> onChanged) =>
      CupertinoSwitch(value: value, onChanged: onChanged);
}

// Pick one family per platform
final WidgetFactory ui =
    Platform.isIOS ? CupertinoWidgetFactory() : MaterialWidgetFactory();
```

**Environment-based services (Factory Method / Simple Factory):**

```dart
ApiClient createApiClient(Environment env) {
  switch (env) {
    case Environment.dev:
      return ApiClient(baseUrl: 'https://dev.api.example.com');
    case Environment.prod:
      return ApiClient(baseUrl: 'https://api.example.com');
  }
}
```

Flutter itself: `Theme.of(context)`, `TextStyle.lerp`, and `Image.network` vs `Image.asset` named constructors are all factory-style creation.

## When to Use Which

- **Simple Factory / factory constructor:** one place decides which concrete class to build; no need for extension via subclassing.
- **Factory Method:** a base class has a process that needs an object, but subclasses should decide which one.
- **Abstract Factory:** you must create **families** of related objects that must stay consistent (themes, platforms, regions, environments).
- Consider using a plain constructor if there's only one concrete class and no reason it will change.

## Pros and Cons

|Pros|Cons|
|---|---|
|✅ Loose coupling: client depends on abstractions (**DIP**)|❌ More classes and indirection|
|✅ Add new products without changing client code (**OCP**)|❌ Abstract Factory: adding a new product type touches every factory|
|✅ Creation code lives in one place (**SRP**)|❌ Can be overkill for simple creation|
|✅ Abstract Factory guarantees products from one family are used together|❌ Factory Method can force subclassing just to change what's created|

## Relation to SOLID

|Principle|Connection|
|---|---|
|**DIP**|The core principle of this chapter: depend on abstractions, not concrete classes|
|**OCP**|New products/factories without editing existing client code|
|**SRP**|Creation logic is separated from usage logic|
|**LSP**|Every concrete product must be substitutable for its abstract type|

## Factory vs Strategy vs Decorator

| |Factory|Strategy|Decorator|
|---|---|---|---|
|**Type**|Creational|Behavioral|Structural|
|**Question it answers**|_What_ do I create?|_How_ do I do this?|_What_ else should it do?|
|**Typical result**|A new object|A swappable algorithm|A wrapped object with added behavior|

They combine well: a factory often decides _which strategy_ or _which decorator chain_ to build.

## Summary

> Don't scatter `new ConcreteClass()` through your code. Put creation behind a factory so the code that uses objects depends only on abstractions, and the code that creates them is easy to find and change.