---
share_link: https://share.note.sx/5r31kc7x#8GgiSqxq0UWxGXsV+q60ceeNHaYdQPEvccnhjZw2iS0
share_updated: 2026-10-12T07:11:22+03:00
---
# envied vs flutter_dotenv

> [!danger] Short answer **Neither package makes a secret safe.** Both ship the value inside the app. `envied` (with `obfuscate: true`) is _better_ than `flutter_dotenv` because it does not bundle a readable `.env` file, but it is still only a speed bump. Real secrets belong on a backend. See [[Flutter Security - Hard Coded Secrets]].

---

## 1. How each one works

| |`flutter_dotenv`|`envied`|
|---|---|---|
|Approach|Loads `.env` **at runtime**|Generates Dart code **at build time** (`build_runner`)|
|Where the `.env` goes|Bundled as an **asset** inside the APK / IPA|**Not bundled**; values are baked into generated `.g.dart`|
|Readable in the APK?|**Yes**, plain text file in `assets/`|Plain text unless `obfuscate: true`; then XOR-encoded in `libapp.so`|
|Access style|`dotenv.env['API_KEY']` (nullable `String?`)|`Env.apiKey` (typed, non-null)|
|Typos in key names|Found at **runtime** (null / exception)|Found at **build time** (generator error)|
|Types|Strings only (you parse manually)|`String`, `int`, `bool`, `double`, enums, etc.|
|Setup effort|Very low|Medium (codegen)|
|Change values without rebuild|Yes (if you swap the asset)|No, regenerate and rebuild|
|Tree-shaking / const|No|Values are compile-time constants / static finals|

---

## 2. `flutter_dotenv` setup

```yaml
# pubspec.yaml
dependencies:
  flutter_dotenv: ^5.0.0

flutter:
  assets:
    - .env          # <- this ships the file inside the app
```

```dart
Future<void> main() async {
  await dotenv.load(fileName: '.env');
  runApp(const MyApp());
}

final baseUrl = dotenv.env['API_BASE_URL'] ?? '';
```

> [!warning] Why it is weak The `.env` is copied to `assets/flutter_assets/.env`. Anyone can run:
> 
> ```bash
> unzip app-release.apk -d out/ && cat out/assets/flutter_assets/.env
> ```

**Good for:** quick prototypes, non-sensitive config (base URLs, feature flags), local development.

---

## 3. `envied` setup

```yaml
# pubspec.yaml
dependencies:
  envied: ^1.0.0

dev_dependencies:
  envied_generator: ^1.0.0
  build_runner: ^2.4.0
```

```bash
# .env (NOT listed under flutter: assets:)
API_BASE_URL=https://api.example.com
MAPS_KEY=AIza...restricted-key
```

```dart
// lib/env/env.dart
import 'package:envied/envied.dart';

part 'env.g.dart';

@Envied(path: '.env', obfuscate: true)
abstract class Env {
  @EnviedField(varName: 'API_BASE_URL')
  static final String apiBaseUrl = _Env.apiBaseUrl;

  @EnviedField(varName: 'MAPS_KEY', obfuscate: true)
  static final String mapsKey = _Env.mapsKey;
}
```

```bash
dart run build_runner build --delete-conflicting-outputs
```

```dart
final key = Env.mapsKey;   // typed, no runtime parsing
```

> [!warning] Gitignore the generated file too `env.g.dart` contains the values (obfuscated or not). Add both to `.gitignore`:
> 
> ```gitignore
> .env
> .env.*
> lib/env/env.g.dart
> ```
> 
> Then generate it in CI before building.

**Good for:** restricted client-side keys (Maps, public analytics IDs) and typed config where you want build-time safety.

---

## 4. What obfuscation really does

With `obfuscate: true`, `envied` stores the value as an array of integers XOR-ed with a random key, and decodes it at runtime.

- Removes the plain string from `strings libapp.so`. Good against casual scraping.
- **Does not stop** a reverse engineer. The cipher key is stored in the same binary, and the decoded value exists in memory at runtime (Frida can read it).
- Time to break it: minutes for someone experienced.

---

## 5. Third option: `--dart-define`

Built into Flutter, no package needed.

```bash
flutter build apk --dart-define-from-file=config/prod.json
```

```dart
const baseUrl = String.fromEnvironment('API_BASE_URL');
```

|                              | `--dart-define`                                              |
| ---------------------------- | ------------------------------------------------------------ |
| Packages needed              | None                                                         |
| `.env` file shipped as asset | No                                                           |
| Visible in binary            | Yes (plain strings in `libapp.so`)                           |
| Best for                     | Environment URLs, flavors, feature flags, CI-injected config |

---

## 6. Side-by-side verdict

|Need|Pick|
|---|---|
|Quick prototype / local dev|`flutter_dotenv`|
|Non-secret config per environment (CI)|`--dart-define-from-file`|
|Typed config + safer than plain assets|`envied`|
|Restricted client key (Maps, etc.)|`envied` with `obfuscate: true` + key restrictions|
|Stripe secret, OpenAI key, DB password, signing keys|**Neither.** Use a backend|
|User login tokens|`flutter_secure_storage`|

### Ranking by security (weakest to strongest)

1. `flutter_dotenv` (plain asset, trivially readable)
2. `--dart-define` (plain string in binary, but no separate file)
3. `envied` with `obfuscate: true` (encoded, still recoverable)
4. **Backend proxy / short-lived tokens** (secret never leaves the server)

---

## 7. Multiple environments (dev / staging / prod)

```dart
@Envied(path: '.env.dev', name: 'DevEnv')
abstract class DevEnv { /* ... */ }

@Envied(path: '.env.prod', name: 'ProdEnv', obfuscate: true)
abstract class ProdEnv { /* ... */ }
```

Or with `flutter_dotenv`:

```dart
await dotenv.load(fileName: const String.fromEnvironment('ENV_FILE', defaultValue: '.env.dev'));
```

For flavors, `--dart-define-from-file` with one JSON per flavor is usually the simplest.

---

## 8. Migrating from `flutter_dotenv` to `envied`

- [ ] Remove `.env` from `flutter: assets:` in `pubspec.yaml`
- [ ] Replace `dotenv.env['X']` calls with `Env.x`
- [ ] Remove `dotenv.load()` from `main()`
- [ ] Add `.env` and `env.g.dart` to `.gitignore`
- [ ] Add a `build_runner` step to CI before `flutter build`
- [ ] **Rotate any key that was previously bundled as an asset**, since old APKs still contain it
- [ ] Verify: build release, run `strings libapp.so | grep -i <key prefix>` and check the APK `assets/` folder

> [!tip] Self-test After switching, unzip your release APK and confirm there is no `.env` under `assets/flutter_assets/` and that the key no longer appears in `strings` output.

---

## Related

- [[Flutter Security - Hard Coded Secrets]]
- [[flutter_secure_storage]]
- [[Firebase App Check]]
- [[Flutter Flavors]]