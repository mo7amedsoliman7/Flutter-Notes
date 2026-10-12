A small **shopping app** where the three patterns from this series each solve a real problem.

|Feature|Problem|Pattern|
|---|---|---|
|Checkout payment|Many payment methods, new ones added often|**Strategy**|
|Cart and UI updates|Several widgets must react when the cart changes|**Observer**|
|Product data layer|Need logging and caching without touching the API code|**Decorator**|

## Project Structure

```
lib/
├── main.dart
├── models/
│   └── product.dart
├── data/
│   ├── product_repository.dart        // interface
│   ├── api_product_repository.dart    // real implementation
│   ├── caching_product_repository.dart  // Decorator
│   └── logging_product_repository.dart  // Decorator
├── payment/
│   ├── payment_strategy.dart          // Strategy interface
│   └── strategies.dart                // Card, Wallet, CashOnDelivery
├── cart/
│   └── cart_notifier.dart             // Observer (ChangeNotifier)
└── ui/
    ├── product_list_screen.dart
    └── checkout_screen.dart
```

---

## Case 1: Payment Methods → Strategy

**Problem:** checkout has `if (method == 'card') ... else if (method == 'wallet') ...`. Every new method edits the checkout code.

**Solution:** each payment method is a strategy behind one interface.

```dart
// payment/payment_strategy.dart
abstract class PaymentStrategy {
  String get name;
  Future<bool> pay(double amount);
}
```

```dart
// payment/strategies.dart
class CardPayment implements PaymentStrategy {
  @override
  String get name => 'Credit Card';

  @override
  Future<bool> pay(double amount) async {
    // call card gateway SDK
    return true;
  }
}

class WalletPayment implements PaymentStrategy {
  @override
  String get name => 'Wallet';

  @override
  Future<bool> pay(double amount) async {
    // deduct from wallet balance
    return true;
  }
}

class CashOnDelivery implements PaymentStrategy {
  @override
  String get name => 'Cash on Delivery';

  @override
  Future<bool> pay(double amount) async => true; // confirmed at delivery
}
```

```dart
// ui/checkout_screen.dart
class CheckoutScreen extends StatefulWidget {
  final double total;
  const CheckoutScreen({super.key, required this.total});

  @override
  State<CheckoutScreen> createState() => _CheckoutScreenState();
}

class _CheckoutScreenState extends State<CheckoutScreen> {
  final List<PaymentStrategy> _methods = [
    CardPayment(),
    WalletPayment(),
    CashOnDelivery(),
  ];
  late PaymentStrategy _selected = _methods.first; // swap at runtime

  Future<void> _confirm() async {
    final ok = await _selected.pay(widget.total); // UI doesn't know which one
    if (!mounted) return;
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(content: Text(ok ? 'Paid with ${_selected.name}' : 'Payment failed')),
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Checkout')),
      body: Column(
        children: [
          for (final m in _methods)
            RadioListTile<PaymentStrategy>(
              title: Text(m.name),
              value: m,
              groupValue: _selected,
              onChanged: (v) => setState(() => _selected = v!),
            ),
          ElevatedButton(onPressed: _confirm, child: const Text('Pay')),
        ],
      ),
    );
  }
}
```

**Result:** adding Apple Pay or Fawry = one new class + add it to the list. The checkout logic never changes (**OCP**).

---

## Case 2: Cart State → Observer

**Problem:** the cart badge, the product list ("In cart" label), and the checkout total must all update when the cart changes. Manually calling each widget is fragile.

**Solution:** `ChangeNotifier` is the Subject; widgets listening to it are the Observers.

```dart
// cart/cart_notifier.dart
class CartNotifier extends ChangeNotifier {
  final Map<Product, int> _items = {};

  Map<Product, int> get items => Map.unmodifiable(_items);
  int get count => _items.values.fold(0, (a, b) => a + b);
  double get total =>
      _items.entries.fold(0, (sum, e) => sum + e.key.price * e.value);

  void add(Product p) {
    _items.update(p, (q) => q + 1, ifAbsent: () => 1);
    notifyListeners();   // notify all observers
  }

  void remove(Product p) {
    if (!_items.containsKey(p)) return;
    final q = _items[p]!;
    q <= 1 ? _items.remove(p) : _items[p] = q - 1;
    notifyListeners();
  }

  void clear() {
    _items.clear();
    notifyListeners();
  }
}
```

Using it with `provider` (any equivalent, such as Riverpod, works the same way):

```dart
// main.dart
void main() {
  runApp(
    ChangeNotifierProvider(
      create: (_) => CartNotifier(),
      child: const MyApp(),
    ),
  );
}
```

```dart
// Observer #1: badge in the app bar
class CartBadge extends StatelessWidget {
  const CartBadge({super.key});

  @override
  Widget build(BuildContext context) {
    final count = context.watch<CartNotifier>().count; // subscribes
    return Badge(label: Text('$count'), child: const Icon(Icons.shopping_cart));
  }
}

// Observer #2: total in checkout
class TotalText extends StatelessWidget {
  const TotalText({super.key});

  @override
  Widget build(BuildContext context) {
    final total = context.watch<CartNotifier>().total;
    return Text('Total: \$${total.toStringAsFixed(2)}');
  }
}

// Triggering a change (no subscription needed, so use read)
ElevatedButton(
  onPressed: () => context.read<CartNotifier>().add(product),
  child: const Text('Add to cart'),
);
```

**Result:** one `notifyListeners()` updates every interested widget. New UI that depends on the cart = one more `watch`, and `CartNotifier` doesn't change.

---

## Case 3: Data Layer → Decorator

**Problem:** you need caching and logging around API calls, but you don't want to pollute `ApiProductRepository` or duplicate that code across repositories.

**Solution:** decorators implement the same interface and wrap the real repository.

```dart
// data/product_repository.dart
abstract class ProductRepository {
  Future<List<Product>> getProducts();
  Future<Product> getProduct(String id);
}
```

```dart
// data/api_product_repository.dart  (the real component)
class ApiProductRepository implements ProductRepository {
  @override
  Future<List<Product>> getProducts() async {
    // HTTP call (dio / http)
    return [];
  }

  @override
  Future<Product> getProduct(String id) async {
    // HTTP call
    return Product(id: id, name: 'Item', price: 10);
  }
}
```

```dart
// data/caching_product_repository.dart  (Decorator)
class CachingProductRepository implements ProductRepository {
  final ProductRepository inner;
  CachingProductRepository(this.inner);

  List<Product>? _all;
  final Map<String, Product> _byId = {};

  @override
  Future<List<Product>> getProducts() async =>
      _all ??= await inner.getProducts();

  @override
  Future<Product> getProduct(String id) async =>
      _byId[id] ??= await inner.getProduct(id);
}
```

```dart
// data/logging_product_repository.dart  (Decorator)
class LoggingProductRepository implements ProductRepository {
  final ProductRepository inner;
  LoggingProductRepository(this.inner);

  @override
  Future<List<Product>> getProducts() async {
    final sw = Stopwatch()..start();
    final result = await inner.getProducts();
    debugPrint('getProducts: ${result.length} items in ${sw.elapsedMilliseconds}ms');
    return result;
  }

  @override
  Future<Product> getProduct(String id) async {
    debugPrint('getProduct($id)');
    return inner.getProduct(id);
  }
}
```

Compose them in one place (the composition root):

```dart
// main.dart
final ProductRepository productRepository = LoggingProductRepository(
  CachingProductRepository(
    ApiProductRepository(),
  ),
);

// Debug builds: keep logging. Release builds: drop it.
final repo = kDebugMode
    ? LoggingProductRepository(CachingProductRepository(ApiProductRepository()))
    : CachingProductRepository(ApiProductRepository());
```

**Result:** features are toggled by how you wrap, not by editing the API class. Order matters: here logging sits outside caching, so it logs cache hits too. Swap them to log only real network calls.

---

## How the Three Fit Together

```
 UI (widgets)
   │  context.watch<CartNotifier>()        ← Observer
   │  _selected.pay(total)                 ← Strategy
   ▼
 CartNotifier ─────────► ProductRepository (interface)
                            ▲
              LoggingProductRepository      ← Decorator
                 └─ CachingProductRepository  ← Decorator
                      └─ ApiProductRepository   (real work)
```

Notice the **DIP** at work: the UI and notifier depend on interfaces (`PaymentStrategy`, `ProductRepository`), and only `main.dart` knows the concrete classes.

## Testing Becomes Easy

```dart
class FakePayment implements PaymentStrategy {
  @override
  String get name => 'Fake';
  @override
  Future<bool> pay(double amount) async => true;
}

class FakeProductRepository implements ProductRepository {
  @override
  Future<List<Product>> getProducts() async => [Product(id: '1', name: 'Test', price: 5)];
  @override
  Future<Product> getProduct(String id) async => Product(id: id, name: 'Test', price: 5);
}

test('cart total updates and notifies', () {
  final cart = CartNotifier();
  var notified = 0;
  cart.addListener(() => notified++);

  cart.add(Product(id: '1', name: 'Test', price: 5));
  cart.add(Product(id: '1', name: 'Test', price: 5));

  expect(cart.total, 10);
  expect(notified, 2);
});
```

## When NOT to Use Them Here

- **Strategy:** if you only ever have two payment methods that never change, an `if` is fine.
- **Observer:** for one widget that owns its own local state, plain `setState` is simpler than a notifier.
- **Decorator:** if caching is needed in exactly one place, building it into that class may be simpler than a wrapper.

Patterns pay off when change is likely; don't add them just to have them.

## Quick Checklist for Your Own Project

1. Do you have an `if/else` or `switch` choosing between **ways of doing the same thing**? → Strategy
2. Do **multiple widgets/classes** need to react to the same change? → Observer (`ChangeNotifier`, `Stream`, Bloc, Riverpod)
3. Do you want to add **cross-cutting behavior** (logging, caching, retry, auth headers, analytics) around an existing class? → Decorator