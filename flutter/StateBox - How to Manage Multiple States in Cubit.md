---
share_link: https://share.note.sx/rbwcg3qa#Nqf6qcMHOpN33Xz+72BSjk0H5TwR6hg+exCDRICH3Ho
share_updated: 2026-10-12T05:07:07+03:00
---
## 1. The StateBox wrapper

```dart
enum BoxStatus { initial, loading, success, failure }

class StateBox<T> extends Equatable {
  final BoxStatus status;
  final T? data;
  final String? error;

  const StateBox({
    this.status = BoxStatus.initial,
    this.data,
    this.error,
  });

  const StateBox.loading({T? previous})
      : status = BoxStatus.loading, data = previous, error = null;

  const StateBox.success(T this.data)
      : status = BoxStatus.success, error = null;

  const StateBox.failure(String this.error, {T? previous})
      : status = BoxStatus.failure, data = previous;

  bool get isLoading => status == BoxStatus.loading;
  bool get isSuccess => status == BoxStatus.success;
  bool get isFailure => status == BoxStatus.failure;

  @override
  List<Object?> get props => [status, data, error];
}
```

## 2. One state class holding multiple boxes

```dart
class HomeState extends Equatable {
  final StateBox<List<Service>> services;
  final StateBox<List<Order>> orders;
  final StateBox<User> profile;

  const HomeState({
    this.services = const StateBox(),
    this.orders = const StateBox(),
    this.profile = const StateBox(),
  });

  HomeState copyWith({
    StateBox<List<Service>>? services,
    StateBox<List<Order>>? orders,
    StateBox<User>? profile,
  }) => HomeState(
        services: services ?? this.services,
        orders: orders ?? this.orders,
        profile: profile ?? this.profile,
      );

  @override
  List<Object?> get props => [services, orders, profile];
}
```

## 3. The Cubit

```dart
class HomeCubit extends Cubit<HomeState> {
  HomeCubit(this._repo) : super(const HomeState());
  final HomeRepo _repo;

  Future<void> loadServices() async {
    emit(state.copyWith(
      services: StateBox.loading(previous: state.services.data),
    ));
    try {
      final data = await _repo.getServices();
      emit(state.copyWith(services: StateBox.success(data)));
    } catch (e) {
      emit(state.copyWith(
        services: StateBox.failure(e.toString(), previous: state.services.data),
      ));
    }
  }

  // Run independent loads in parallel
  Future<void> loadAll() => Future.wait([loadServices(), loadOrders(), loadProfile()]);
}
```

## 4. UI: rebuild only what changed

Use `BlocSelector` (or `buildWhen`) so each section rebuilds only when its own box changes:

```dart
BlocSelector<HomeCubit, HomeState, StateBox<List<Service>>>(
  selector: (state) => state.services,
  builder: (context, box) {
    if (box.isLoading && box.data == null) return const CircularProgressIndicator();
    if (box.isFailure && box.data == null) return Text(box.error!);
    return ServicesList(box.data ?? const []);
  },
)
```

## When to use which approach

| Approach                                            | Best for                                                                             |
| --------------------------------------------------- | ------------------------------------------------------------------------------------ |
| **StateBox + `copyWith`** (above)                   | Screens with several independent data sources that load, fail, or refresh separately |
| **Sealed classes** (`Loading` / `Loaded` / `Error`) | A single flow with mutually exclusive states, e.g. a login screen                    |
| **One state + status enum + nullable fields**       | Simple screens; gets messy as the number of fields grows                             |

## Tips

- [ ] Keep `data` when emitting `loading` or `failure`, so the UI doesn't flash empty during a refresh.
- [ ] Make every state class `Equatable`, or Cubit won't dedupe emissions correctly.
- [ ] If one Cubit grows too large, split it into one Cubit per feature and combine them with `MultiBlocProvider`.
## More cases

### Case 1: Reusable `when` helper for the UI

Avoids repeating the same `if` chain in every widget.

```dart
extension StateBoxX<T> on StateBox<T> {
  R when<R>({
    required R Function() loading,
    required R Function(String error) failure,
    required R Function(T data) success,
    R Function()? initial,
  }) {
    if (isLoading && data == null) return loading();
    if (isFailure && data == null) return failure(error ?? 'Unknown error');
    if (data != null) return success(data as T);
    return (initial ?? loading)();
  }
}
```

```dart
BlocSelector<HomeCubit, HomeState, StateBox<List<Order>>>(
  selector: (s) => s.orders,
  builder: (context, box) => box.when(
    loading: () => const Center(child: CircularProgressIndicator()),
    failure: (e) => ErrorView(message: e, onRetry: context.read<HomeCubit>().loadOrders),
    success: (orders) => OrdersList(orders),
  ),
)
```

### Case 2: Pull-to-refresh (keep old data visible)

Because `loading` keeps `previous`, the list stays on screen while refreshing.

```dart
RefreshIndicator(
  onRefresh: context.read<HomeCubit>().loadOrders,
  child: BlocBuilder<HomeCubit, HomeState>(
    buildWhen: (p, c) => p.orders != c.orders,
    builder: (context, state) => OrdersList(state.orders.data ?? const []),
  ),
)
```

### Case 3: Pagination

Add paging info next to the box, or extend the data type.

```dart
class PagedList<T> extends Equatable {
  final List<T> items;
  final int page;
  final bool hasMore;
  const PagedList({this.items = const [], this.page = 0, this.hasMore = true});

  @override
  List<Object?> get props => [items, page, hasMore];
}

// State field
final StateBox<PagedList<Order>> orders;

// Cubit
Future<void> loadMoreOrders() async {
  final current = state.orders.data ?? const PagedList<Order>();
  if (state.orders.isLoading || !current.hasMore) return;

  emit(state.copyWith(orders: StateBox.loading(previous: current)));
  try {
    final next = await _repo.getOrders(page: current.page + 1);
    emit(state.copyWith(
      orders: StateBox.success(PagedList(
        items: [...current.items, ...next.items],
        page: current.page + 1,
        hasMore: next.hasMore,
      )),
    ));
  } catch (e) {
    emit(state.copyWith(
      orders: StateBox.failure(e.toString(), previous: current),
    ));
  }
}
```

### Case 4: Search with debounce

```dart
Timer? _debounce;

void onSearchChanged(String query) {
  _debounce?.cancel();
  if (query.trim().isEmpty) {
    emit(state.copyWith(searchResults: const StateBox()));
    return;
  }
  _debounce = Timer(const Duration(milliseconds: 400), () => _search(query));
}

Future<void> _search(String query) async {
  emit(state.copyWith(searchResults: const StateBox.loading()));
  try {
    final results = await _repo.search(query);
    emit(state.copyWith(searchResults: StateBox.success(results)));
  } catch (e) {
    emit(state.copyWith(searchResults: StateBox.failure(e.toString())));
  }
}

@override
Future<void> close() {
  _debounce?.cancel();
  return super.close();
}
```