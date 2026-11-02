---
name: ngp-flutter
description: NGP TechWorld standard for Flutter mobile apps built on GetX (state, DI and navigation) with the core / data / features layering of the ngp_flutter_clean_architectuer template. Use when creating, editing or reviewing any NGP Flutter/Dart project - pages, controllers, bindings, routes, repositories, DTOs, entities, services, widgets, translations.
---

# NGP Flutter Standard (GetX)

> v0.2.1 (approved architecture). Also follow `ngp-general`.
> **Starter repo:** https://github.com/ngp-techworld/ngp_flutter_template (built from
> `NGPTechWorld/ngp_flutter_clean_architectuer`). Every new app starts from it ("Use this
> template"). When this file and the template disagree, the template's `ARCHITECTURE.md` wins and
> this file gets fixed.

## 1. Stack

| Area | Choice |
|---|---|
| SDK | Flutter pinned with **fvm** (`.fvmrc`, currently 3.44.2). Always run `fvm flutter ...` / `fvm dart ...`. |
| State + DI + navigation | **GetX** (`get`): `GetxController`, `Rx`/`Obx`, `Bindings`, `GetPage`, `Get.toNamed` |
| HTTP | `dio` wrapped in `ApiService` (+ `pretty_dio_logger` in debug only) |
| Key/value cache | `CacheService` contract; default implementation `SecureTokenCache`: tokens in `flutter_secure_storage`, everything else in `get_storage`. |
| Local database | `hive_ce` (+ `hive_ce_generator`), only when the app needs offline data |
| Connectivity | `connectivity_plus` through `NetworkInfo` |
| Notifications | `flutter_local_notifications` through `NotificationService` |
| Push (optional) | `firebase_messaging` through `FcmService` + `NotificationRouter`; skipped until the Firebase config files are added |
| Realtime (optional) | `socket_io_client` through `SocketService`, connects on first use |
| Translations | GetX translations loaded from `assets/lang/ar.json`, `en.json` |
| Assets | generated typed access (`assets.gen.dart`, `fonts.gen.dart`); SVG via `flutter_svg` |
| Lint | `flutter_lints` + rules in §9 |
| Codegen | `build_runner` (`fvm dart run build_runner build`) |

Don't add another state-management, DI or routing package (no Bloc, Provider, Riverpod, get_it,
go_router) in GetX apps. One way to do each thing.

## 2. Layers and the dependency rule

| Layer | Responsibility | May depend on |
|---|---|---|
| `core/` | Shared infrastructure: network, cache, errors, theme, translations, routes, UI kit | nothing |
| `data/` | What data looks like and where it comes from: DTO / Entity / Models / Repositories | `core` |
| `features/` | Screens: one Page + one Controller per feature | `core` + `data` |

Dependencies point one way: **`features → data → core`**.
The only exception is `core/config` and `core/routes`, the composition root, which import
features to register them.

So: `core` never imports `data` or `features`; `data` never imports `features`; a feature never
imports another feature's controller (share through a repository or a `GetxService`).

## 3. Folder structure

```text
lib/
├── main.dart                 entry point: AppBinding().dependencies() → AppBinding.init() → runApp
├── hive_registrar.g.dart     (generated) Hive adapters
├── core/
│   ├── app/                  MyApp: GetMaterialApp (theme, locale, routes)
│   ├── config/               AppBinding: global DI + one-time init
│   ├── routes/               AppRoutes (names) + NavigationManager (GetPage list)
│   ├── services/
│   │   ├── api/              ApiService (Dio) + EndPoints
│   │   ├── cache/            CacheService (contract) + implementations + cache_keys.dart
│   │   ├── connection/       NetworkInfo
│   │   ├── errors/           ErrorHandler, NetworkFailureModel, Failure / Exception
│   │   ├── notifications/    NotificationService (local) + FcmService + NotificationRouter
│   │   ├── params/           request params / bodies
│   │   ├── settings/         AppSettingsService (font scale)
│   │   └── socket/           SocketService (Socket.IO)
│   ├── themes/               ThemeManager
│   ├── translations/         AppTranslations (files in assets/lang)
│   └── utils/
│       ├── color_manager.dart, values_manager.dart (AppSize, AppPadding, FontSize), string_manager.dart
│       ├── *.gen.dart        (generated) typed assets and fonts
│       ├── directional_icons.dart  back/forward icons that flip in RTL
│       ├── helper/           extensions, url_helper (absoluteUrl), jwt, date_helper
│       └── widgets/          app-wide UI kit: buttons, text fields, toasts, app bars...
├── data/
│   ├── dto/                  shape of API data (fromMap / toMap)
│   ├── entity/               app-internal objects used by the UI
│   ├── enums/                e.g. LoadingState
│   ├── models/               AppResponse<T>, PaginatedModel<T>, Hive models
│   └── repos/                repositories: abstract class + Imp class
└── features/
    ├── splash/               splash_page.dart + splash_controller.dart
    ├── home/                 home_page.dart + home_controller.dart
    └── widgets/              widgets shared between screens (loading, empty, dialogs)
```

A feature with several screens gets sub-folders by screen, each with its page + controller
(`features/orders/order_list/…`, `features/orders/order_details/…`). Widgets used only by one
feature go in `features/<name>/widgets/`.

## 4. Boot sequence

```text
main()
 ├─ AppBinding().dependencies()   Get.put: CacheService, AppSettingsService, ApiService, SocketService
 ├─ AppBinding.init()             cache, settings, translations, Hive, notifications, FCM
 └─ runApp(MyApp)                 GetMaterialApp → /splash → /home
```

Each block in `AppBinding.init()` is independent: delete the ones the app doesn't need and remove
their packages from `pubspec.yaml`.

## 5. Data flow

```text
Page ──(Obx)──▶ Controller ──▶ Repository ──▶ ApiService (Dio) / CacheService / Hive
  ▲                                  │
  └────── Rx state ◀── AppResponse<T> ◀┘
```

| Part | Does | Never does |
|---|---|---|
| **Page** (`StatelessWidget`) | Shows state with `Obx`, forwards taps to the controller. | Logic, API calls, `if` business rules. |
| **Controller** (`GetxController`) | Holds reactive state (`Rx`), screen logic, calls repositories, navigates, shows toasts. | Calls `ApiService` or Dio directly; parses JSON. |
| **Repository** | The **only** code that talks to `ApiService` / cache / Hive. Maps DTOs, **always returns `AppResponse<T>`** instead of throwing. | Touches widgets or Rx UI state. |
| **ErrorHandler** | `ErrorHandler.handle(e).failure` turns any error into a `NetworkFailureModel` inside `AppResponse` (see §12). | |

### Repository (abstract + Imp)

```dart
abstract class OrdersRepository {
  Future<AppResponse<PaginatedModel<OrderDto>>> getOrders({required int page});
}

class ImpOrdersRepository extends GetxService implements OrdersRepository {
  final ApiService _api = Get.find<ApiService>();

  @override
  Future<AppResponse<PaginatedModel<OrderDto>>> getOrders({required int page}) async {
    final appResponse = AppResponse<PaginatedModel<OrderDto>>(success: false);
    try {
      final response = await _api.request(
        url: EndPoints.orders,
        method: Method.get,
        requiredToken: true,
        queryParameters: {'page': page},
      );
      appResponse.success = true;
      appResponse.data = PaginatedModel.fromJson(response.data, OrderDto.fromMap);
    } catch (e) {
      appResponse.networkFailure = ErrorHandler.handle(e).failure;
    }
    return appResponse;
  }
}
```

### Controller

```dart
class OrdersController extends GetxController {
  final OrdersRepository _repo = Get.find<OrdersRepository>();

  final Rx<LoadingState> state = LoadingState.idle.obs;
  final RxList<OrderDto> orders = <OrderDto>[].obs;

  @override
  void onInit() {
    super.onInit();
    loadOrders();
  }

  Future<void> loadOrders() async {
    state.value = LoadingState.loading;
    final result = await _repo.getOrders(page: 1);
    if (!result.success) {
      state.value = LoadingState.hasError;
      CustomToasts(message: result.getErrorMessage(), type: CustomToastType.error).show();
      return;
    }
    orders.assignAll(result.data!.data);
    state.value = orders.isEmpty ? LoadingState.doneWithNoData : LoadingState.doneWithData;
  }
}
```

### Page

```dart
class OrdersPage extends GetView<OrdersController> {
  const OrdersPage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: NormalAppBar(title: StringManager.orders.tr),
      body: Obx(() => switch (controller.state.value) {
            LoadingState.loading || LoadingState.idle => const LoadingCard(),
            LoadingState.doneWithNoData => const EmptyCard(),
            LoadingState.hasError => ErrorCard(onRetry: controller.loadOrders), // add to features/widgets
            LoadingState.doneWithData => _OrdersList(orders: controller.orders),
          }),
    );
  }
}
```

Use `LoadingState` for every screen that loads data, and handle all of its values
(loading, empty, error with retry, data).

## 6. Writing rules

- **One file per responsibility:** `<feature>_page.dart` for the view, `<feature>_controller.dart`
  for the logic.
- **Routes:** the name is a constant in `AppRoutes` (`static const ordersRoute = '/orders'`);
  the page and its binding are registered in `NavigationManager`. Navigate with
  `Get.toNamed(AppRoutes.ordersRoute)`, never with a raw string.
- **DI:** global services in `AppBinding`; controllers through `BindingsBuilder` on their route
  (lazy). Repositories are registered once (`Get.lazyPut<OrdersRepository>(() => ImpOrdersRepository(), fenix: true)`)
  in `AppBinding` or in the route binding of the feature that owns them.
- **Program to the contract:** `Get.find<CacheService>()`, `Get.find<OrdersRepository>()`, never
  the concrete class. Swapping an implementation is then a one-line change.
- **Repository = abstract + Imp:** controllers depend on the abstract class, which makes testing
  and swapping easy.
- **No hard-coded values in the UI:** colors from `ColorManager`, sizes from `AppSize` /
  `AppPadding` / `FontSize`, text from `StringManager` keys + `.tr`, images/icons from the generated
  assets class.
- **Repeated widget?** App-wide → `core/utils/widgets`; shared between screens → `features/widgets`.
- **Generated files (`*.g.dart`, `*.gen.dart`) are never edited by hand.** Regenerate with
  `fvm dart run build_runner build`.
- Controllers clean up in `onClose()` (dispose `AnimationController`, `TextEditingController`,
  cancel `Worker`s and stream subscriptions).

## 7. Adding a new feature (checklist)

1. Create `lib/features/<name>/` with `<name>_page.dart` + `<name>_controller.dart`.
2. If it needs data: DTO in `data/dto`, Repository (abstract + Imp) in `data/repos`, the URL in
   `EndPoints`.
3. Add the route name in `AppRoutes`, then a `GetPage` with its binding in `NavigationManager`.
4. Navigate with `Get.toNamed(AppRoutes.<name>Route)`.
5. Add any new text keys to **both** `assets/lang/ar.json` and `en.json`.

## 8. Naming

- Files `snake_case.dart`, one main public class per file, named after it.
- Suffixes: `XxxPage`, `XxxController`, `XxxRepository` + `ImpXxxRepository`, `XxxDto`,
  `XxxEntity`, `XxxModel` (Hive / app models), `XxxService`.
- DTOs: `fromMap` / `toMap` (+ `fromJson` / `toJson` string helpers), field names in camelCase,
  mapping the API's snake_case keys explicitly.
- Rx fields named for what they hold (`orders`, `isSubmitting`), not `data`, `list`, `flag`.
- Imports use the project-absolute form `'/core/...'`, as in the template.

## 9. Lints & formatting

Keep `flutter_lints` and turn on these rules in `analysis_options.yaml`:

```yaml
linter:
  rules:
    - prefer_single_quotes
    - prefer_const_constructors
    - prefer_final_locals
    - require_trailing_commas
    - avoid_print
    - always_declare_return_types
    - use_build_context_synchronously
```

`fvm dart format .` before every commit; CI fails if formatting changes anything.

## 10. RTL and translations

- Arabic is the default locale. Every new string is a key in `ar.json` and `en.json`.
- Use directional layout: `EdgeInsetsDirectional`, `AlignmentDirectional`,
  `PositionedDirectional`. Never `EdgeInsets.only(left: …)` / `right:`.

## 11. Testing

- Unit-test DTO parsing (`fromMap` with a real API sample) and every repository (mock
  `ApiService`, check `AppResponse.success` and the failure path).
- Unit-test controllers by registering a fake repository with `Get.put<OrdersRepository>(FakeOrdersRepository())`
  and calling `Get.reset()` in `tearDown`.
- `test/` mirrors `lib/`. Test files import app code as `package:<app_name>/...`.

## 12. Auth, errors and pagination (built into the template)

- `ApiService.request` adds `Authorization: Bearer <token>` on top of the default headers, so
  `Accept: application/json` is always sent.
- A 401 triggers **one** shared token refresh (`EndPoints.refresh`: `{refreshToken}` in,
  `{accessToken, refreshToken?}` out) and replays the request. With no refresh token (plain Sanctum
  tokens) or a failed refresh, the session is cleared in one place and the app goes back to the
  start route. Point `_onSessionExpired` at the app's login route.
- Tokens live in secure storage through `SecureTokenCache` (`kUserToken`, `kRefreshToken`).
  Never write a credential to GetStorage.
- Errors follow the API shape `{message, code, errors}` (`ngp-laravel` §5). On
  `NetworkFailureModel`: `code` is the HTTP status, `errorCode` is the API `code` to switch on,
  `errors` holds the 422 field errors, `message` is already translated: show it as is.
- `PaginatedModel.fromJson` reads Laravel's `data` + `meta` (`current_page`, `last_page`,
  `per_page`, `total`). Send `?page=` and read `totalPages` to know when to stop.

## 13. Before opening a PR

- [ ] `fvm flutter analyze`, `fvm dart format --set-exit-if-changed .`, `fvm flutter test` pass
- [ ] Pages contain no logic; controllers never call `ApiService` directly
- [ ] Repositories return `AppResponse<T>` and never throw
- [ ] No hard-coded colors, sizes or strings; new keys in `ar.json` and `en.json`
- [ ] Loading / empty / error states handled; checked in RTL on a small phone
