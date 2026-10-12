---
share_link: https://share.note.sx/1hecxcwl#Vb6BQjnY2XvzuJeLJELrI218SDUDuiVsZb2ywFjAXjg
share_updated: 2026-10-07T05:31:32+03:00
---
> **Strategy defines a family of algorithms, puts each one in its own class, and makes them interchangeable at runtime.**
   
   Category: **Behavioral Pattern**

## The Idea

Instead of packing every way of doing a task into one class full of `if / else` or `switch` blocks, you move each way into its own class (a _strategy_). The main class (the **Context**) receives the strategy it should use from the outside.

---
## The Problem

Imagine a checkout screen with several payment methods:

```dart
class PaymentService {
  void pay(String method, double amount) {
    if (method == 'credit_card') {
      print('Paying $amount with credit card');
    } else if (method == 'paypal') {
      print('Paying $amount with PayPal');
    } else if (method == 'wallet') {
      print('Paying $amount with wallet');
    }
    // Every new method means editing this class!
  }
}
```

**Drawbacks:**
- Every new payment method means modifying existing code (violates the **Open/Closed Principle**).
- The class keeps growing and getting more complex.
- Hard to test each method in isolation.
- Different logic is tangled together in one place.
## The Solution: Strategy Pattern

### Components

|Component|Role|
|---|---|
|**Strategy** (interface)|Common interface for all algorithms|
|**Concrete Strategies**|The actual implementations, each done differently|
|**Context**|The class that uses a strategy and delegates work to it|

![[Strategy Pattern.png]]
### Implementation in Dart

```dart
// 1. The Strategy (interface)
abstract class PaymentStrategy {
  void pay(double amount);
}

// 2. Concrete strategies
class CreditCardPayment implements PaymentStrategy {
  @override
  void pay(double amount) => print('Paid $amount using Credit Card');
}

class PaypalPayment implements PaymentStrategy {
  @override
  void pay(double amount) => print('Paid $amount using PayPal');
}

class WalletPayment implements PaymentStrategy {
  @override
  void pay(double amount) => print('Paid $amount using Wallet');
}

// 3. The Context
class CheckoutService {
  PaymentStrategy _strategy; 

  CheckoutService(this._strategy);

  // The strategy can be swapped at runtime
  void setStrategy(PaymentStrategy strategy) => _strategy = strategy;

  void checkout(double amount) => _strategy.pay(amount);
}
```

Usage:

```dart
void main() {
  final checkout = CheckoutService(CreditCardPayment());
  checkout.checkout(500); // Paid 500.0 using Credit Card

  checkout.setStrategy(WalletPayment());
  checkout.checkout(200); // Paid 200.0 using Wallet
}
```

Now if you want to add **Apple Pay**, you just create a new class, without touching `CheckoutService`.
### Adding a new method

Adding **Apple Pay** = one new class. `CheckoutService` is never touched.

```dart
class ApplePayPayment implements PaymentStrategy {
  @override
  void pay(double amount) => print('Paid $amount using Apple Pay');
}
```
### Testing becomes easy

Each strategy is tested alone, and the context can be tested with a fake strategy.

```dart
class FakePayment implements PaymentStrategy {
  double? paid;

  @override
  void pay(double amount) => paid = amount;
}

void main() {
  final fake = FakePayment();
  CheckoutService(fake).checkout(100);
  assert(fake.paid == 100);
}
```

---
## Pros and Cons

**Pros**

- **Open/Closed:** Add new strategies without touching existing code.
- **Runtime flexibility:** Swap behavior while the app runs.
- **No big conditionals:** Polymorphism replaces `if/else` and `switch`.
- **Testable:** Every algorithm can be tested in isolation.
- **Reusable:** The same strategy can be used by different contexts.

**Cons**

- **More classes:** Every algorithm needs its own class (or function).
- **Client must know the options:** Something has to choose the right strategy.
- **Overkill for simple cases:** Not worth it for a couple of rarely changing branches.

---
## Strategy vs Other Patterns

| Pattern      | Focus                                          | Difference                                                                |
| ------------ | ---------------------------------------------- | ------------------------------------------------------------------------- |
| **Observer** | Notify many objects about a state change       | One-to-many communication; Strategy is one-to-one behavior swapping       |
| **State**    | Behavior changes as the object's state changes | State changes itself automatically; with Strategy the **client** picks it |
