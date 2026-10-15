Best practices for writing clean, maintainable, and scalable code with `flutter_bloc`.

---
## 1. Cubit vs Bloc

**Cubit** exposes functions that emit states. **Bloc** receives events and maps them to states.

|                                                              | Cubit        | Bloc            |
| ------------------------------------------------------------ | ------------ | --------------- |
| Input                                                        | Method calls | Events          |
| Boilerplate                                                  | Low          | Higher          |
| Traceability                                                 | Only states  | Events + states |
| Event transformers <br>(debounce, throttle, <br>restartable) | No           | Yes             |

**Rule of thumb**

- Start with **Cubit** for most features (forms, CRUD, simple screens).
- Use **Bloc** when you need event traceability or event transformers (search with debounce, rapid taps with throttle, streams).

```dart
// Cubit
class CounterCubit extends Cubit<int> {
  CounterCubit() : super(0);
  void increment() => emit(state + 1);
}

// Bloc with debounce
class SearchBloc extends Bloc<SearchEvent, SearchState> {
  SearchBloc(this._repo) : super(const SearchState()) {
    on<QueryChanged>(
      _onQueryChanged,
      transformer: (events, mapper) =>
          events.debounceTime(const Duration(milliseconds: 300)).switchMap(mapper),
    );
  }
  final SearchRepository _repo;
  // ...
}
```

---

## 2. Enum vs Class (State Modeling)

Two common ways to model state. Pick based on the screen.

### Option A: One class + status enum

Best when data must **persist across statuses** (e.g., keep the list visible while refreshing or showing an error).

```dart
enum ProductsStatus { initial, loading, success, failure }

class ProductsState extends Equatable {
  const ProductsState({
    this.status = ProductsStatus.initial,
    this.products = const [],
    this.errorMessage,
  });

  final ProductsStatus status;
  final List<Product> products;
  final String? errorMessage;

  ProductsState copyWith({
    ProductsStatus? status,
    List<Product>? products,
    String? errorMessage,
  }) =>
      ProductsState(
        status: status ?? this.status,
        products: products ?? this.products,
        errorMessage: errorMessage ?? this.errorMessage,
      );

  @override
  List<Object?> get props => [status, products, errorMessage];
}
```

### Option B: Sealed classes

Best when states are **mutually exclusive** and each carries different data. The compiler forces you to handle every case.

```dart
sealed class ProfileState {}
class ProfileInitial extends ProfileState {}
class ProfileLoading extends ProfileState {}
class ProfileLoaded extends ProfileState {
  ProfileLoaded(this.user);
  final User user;
}
class ProfileError extends ProfileState {
  ProfileError(this.message);
  final String message;
}

// Exhaustive switch
Widget build(BuildContext context) {
  final state = context.watch<ProfileCubit>().state;
  return switch (state) {
    ProfileInitial() => const SizedBox.shrink(),
    ProfileLoading() => const CircularProgressIndicator(),
    ProfileLoaded(:final user) => ProfileView(user: user),
    ProfileError(:final message) => ErrorView(message: message),
  };
}
```

**Rule of thumb:** lists, forms, pagination → _status enum + copyWith_. Simple load/loaded/error screens → _sealed classes_.

---

## 3. Don't Pass Callbacks to the Cubit; Use `BlocListener`

Passing `onSuccess` / `onError` callbacks couples the Cubit to the UI and breaks the unidirectional flow.

```dart
// ❌ Bad
cubit.login(
  email, password,
  onSuccess: () => Navigator.pushReplacementNamed(context, '/home'),
  onError: (e) => ScaffoldMessenger.of(context).showSnackBar(...),
);
```

```dart
// ✅ Good: the Cubit only emits states
BlocListener<LoginCubit, LoginState>(
  listenWhen: (prev, curr) => prev.status != curr.status,
  listener: (context, state) {
    if (state.status == LoginStatus.success) {
      Navigator.pushReplacementNamed(context, '/home');
    } else if (state.status == LoginStatus.failure) {
      ScaffoldMessenger.of(context)
          .showSnackBar(SnackBar(content: Text(state.errorMessage ?? 'Error')));
    }
  },
  child: const LoginForm(),
);
```

Use `BlocListener` for **one-time side effects** (navigation, snackbars, dialogs). Use `BlocBuilder` for **building UI**. Use `BlocConsumer` only when you need both in the same place.

---

## 4. Don't Store Data Inside the Cubit

The state is the single source of truth. Mutable fields inside the Cubit are invisible to the UI and cause bugs.

```dart
// ❌ Bad
class CartCubit extends Cubit<CartState> {
  CartCubit() : super(CartInitial());
  final List<Item> _items = []; // hidden mutable data

  void add(Item item) {
    _items.add(item);
    emit(CartUpdated()); // Equatable can't tell anything changed
  }
}
```

```dart
// ✅ Good: data lives in an immutable state
class CartCubit extends Cubit<CartState> {
  CartCubit() : super(const CartState());

  void add(Item item) =>
      emit(state.copyWith(items: [...state.items, item]));
}
```

Allowed in a Cubit: injected dependencies (repositories, use cases), `StreamSubscription`s, and timers. Not allowed: business or UI data.

---

## 5. Avoid Logic Inside Your State

States should be **simple, immutable data holders**.

```dart
// ❌ Bad
class CartState {
  final List<Item> items;
  double calculateDiscountedTotal(Coupon c) { /* business logic */ }
  Future<void> save() async { /* side effect */ }
}
```

```dart
// ✅ Good: logic goes to Cubit / use case / domain layer
class CartCubit extends Cubit<CartState> {
  CartCubit(this._applyCoupon) : super(const CartState());
  final ApplyCouponUseCase _applyCoupon;

  void applyCoupon(Coupon coupon) {
    final total = _applyCoupon(state.items, coupon);
    emit(state.copyWith(total: total));
  }
}
```

Tiny derived getters are fine (`bool get isEmpty => items.isEmpty;`). Anything with branching business rules or side effects is not.

---

## 6. Bloc = MVI + Provider

Bloc is essentially two familiar ideas combined:

**MVI (Model-View-Intent)**

- **Intent** → Events (Bloc) or method calls (Cubit)
- **Model** → one immutable State
- **View** → rebuilds from the State
- Flow is **unidirectional**: `UI → Event → Bloc → State → UI`

**Provider**

- `BlocProvider` is `InheritedWidget`-based dependency injection.
- It creates the Bloc, exposes it down the tree, and **closes it automatically**.

```
 ┌──────┐  event   ┌──────┐  state   ┌──────┐
 │  UI  │ ───────▶ │ Bloc │ ───────▶ │  UI  │
 └──────┘          └──────┘          └──────┘
        (Provider delivers the Bloc to the tree)
```

Keeping this mental model prevents anti-patterns like two-way bindings or mutating state from the widget.

---

## 7. Where to Create and Inject Your Cubit

Provide a Cubit **as low as possible, but as high as needed**.

|Scope|Where|Example|
|---|---|---|
|App-wide|Above `MaterialApp`|`AuthCubit`, `ThemeCubit`, `LocaleCubit`|
|Feature / screen|At the route level|`ProductsCubit`, `CheckoutCubit`|
|Local widget|Around the single subtree|Small isolated UI|

```dart
// Route-level provider
GoRoute(
  path: '/products',
  builder: (context, state) => BlocProvider(
    create: (_) => ProductsCubit(getIt<ProductsRepository>())..load(),
    child: const ProductsPage(),
  ),
);
```

Tips:

- Never create a Cubit inside `build()` without `BlocProvider`; it will be recreated and leak.
- Use `BlocProvider.value` **only** to pass an _existing_ instance to a new route/dialog (it will not close it for you).
- Inject repositories with `RepositoryProvider` or `get_it`. Cubits receive them in the constructor so they stay testable.
- Use `MultiBlocProvider` to avoid deeply nested providers.

---

## 8. Don't Create a Cubit for Each Action

One Cubit per **feature or screen**, not per button.

```
// ❌ Bad
LoginCubit, LogoutCubit, RefreshTokenCubit, ValidateEmailCubit ...

// ✅ Good
AuthCubit   → login(), logout(), refreshToken()
```

Guidelines:

- Group by **responsibility** (a cohesive set of states and transitions).
- If methods share the same state, they belong together.
- Split a Cubit only when it grows into two clearly independent concerns (e.g., `ProductsCubit` and `CartCubit`).
- Cubits should not call each other directly. Coordinate with `BlocListener` in the UI, or with a shared repository stream.

---

## 9. Don't Pass `BuildContext` Outside the Cubit

A Cubit with a `BuildContext` is hard to test, risks using a stale context after `await`, and leaks UI into business logic.

```dart
// ❌ Bad
void submit(BuildContext context) async {
  await repo.save();
  Navigator.of(context).pop();
}
```

```dart
// ✅ Good
void submit() async {
  emit(state.copyWith(status: Status.loading));
  try {
    await repo.save();
    emit(state.copyWith(status: Status.success));
  } catch (e) {
    emit(state.copyWith(status: Status.failure, errorMessage: e.toString()));
  }
}

// In the UI
BlocListener<FormCubit, FormState>(
  listener: (context, state) {
    if (state.status == Status.success) Navigator.of(context).pop();
  },
  child: ...,
);
```

If you need localized strings, emit a **message key / error type** and resolve the text in the UI layer.

---

## 10. Don't Access Cubits on Every Small Widget

Reading the Cubit in every tiny widget creates hidden coupling and unnecessary rebuilds.

```dart
// ❌ Bad: every leaf widget knows about the Cubit
class PriceText extends StatelessWidget {
  Widget build(BuildContext context) =>
      Text('${context.watch<CartCubit>().state.total}');
}
```

```dart
// ✅ Good: parent reads state, passes plain data down
BlocBuilder<CartCubit, CartState>(
  builder: (context, state) => CartSummary(total: state.total),
);

// Or limit rebuilds to what matters
BlocSelector<CartCubit, CartState, double>(
  selector: (state) => state.total,
  builder: (context, total) => Text('$total'),
);
```

Tips:

- Smart (container) widgets talk to the Cubit; dumb (presentational) widgets receive data and callbacks.
- Use `buildWhen` / `listenWhen` / `BlocSelector` to narrow rebuilds.
- Use `context.read<T>()` in event handlers (`onPressed`); use `context.watch<T>()` or `BlocBuilder` only inside `build`.
- Reusable widgets should not depend on a specific Cubit.

---

## Quick Checklist

- [ ] Cubit by default, Bloc when you need events or transformers
- [ ] State modeled intentionally (status enum + `copyWith`, or sealed classes)
- [ ] Side effects handled with `BlocListener`, not callbacks
- [ ] All data lives in immutable state
- [ ] No business logic in State classes
- [ ] Unidirectional flow: UI → Event → Bloc → State → UI
- [ ] Cubit provided at the right scope
- [ ] One Cubit per feature, not per action
- [ ] No `BuildContext` inside Cubits
- [ ] Leaf widgets stay Bloc-agnostic; rebuilds are scoped