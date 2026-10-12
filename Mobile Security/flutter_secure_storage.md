---
share_link: https://share.note.sx/cdhh1shr#FTEaiRNgdl5RLCwnXpDIBsJ318iWLxoEXNFVI59F3/k
share_updated: 2026-10-12T07:11:12+03:00
---

# flutter_secure_storage

> [!note] What it is A plugin that stores small key-value strings in the platform's secure store: **Keychain** on iOS / macOS, **Android Keystore-backed encryption** on Android. Use it for **runtime secrets obtained after login** (access / refresh tokens, session IDs, a database encryption key), not for secrets baked into the app. See [[Flutter Security - Hard Coded Secrets]].

---

## 1. Where data actually lives

| Platform    | Backing store                                         | Notes                                                                                                                          |
| ----------- | ----------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| iOS / macOS | Keychain                                              | Survives app uninstall / reinstall on iOS (see gotchas)                                                                        |
| Android     | Keystore-protected key encrypting data in app storage | Older versions used `EncryptedSharedPreferences`; newer versions moved to custom ciphers. Check the changelog for your version |

> [!warning] Not bulletproof On a rooted / jailbroken device or with a compromised OS, an attacker can still reach the data or hook the app at runtime (Frida). Secure storage protects against **casual file access, backups, and other apps**, not a determined attacker with full device control.

---

## 2. Setup

```yaml
# pubspec.yaml
dependencies:
  flutter_secure_storage: ^9.0.0   # check pub.dev for the latest version
```

Android: check the plugin README for the `minSdkVersion` your version requires (it has changed between major versions).

```groovy
// android/app/build.gradle
defaultConfig {
    minSdkVersion 23   // verify against the plugin's current requirement
}
```

---

## 3. Basic usage

```dart
const storage = FlutterSecureStorage();

// Write
await storage.write(key: 'access_token', value: token);

// Read (null if missing)
final token = await storage.read(key: 'access_token');

// Check / delete
final exists = await storage.containsKey(key: 'access_token');
await storage.delete(key: 'access_token');

// Read everything / wipe everything
final all = await storage.readAll();
await storage.deleteAll();
```

All values are **strings**. Encode other types yourself:

```dart
await storage.write(key: 'user', value: jsonEncode(user.toJson()));
final user = User.fromJson(jsonDecode((await storage.read(key: 'user'))!));
```

---

## 4. Platform options

```dart
const storage = FlutterSecureStorage(
  aOptions: AndroidOptions(
    encryptedSharedPreferences: true, // behavior depends on package version
  ),
  iOptions: IOSOptions(
    accessibility: KeychainAccessibility.first_unlock_this_device,
  ),
);
```

### iOS Keychain accessibility

|Option|Readable when|Synced / migrated to new device|
|---|---|---|
|`unlocked`|Device unlocked|Yes (backup)|
|`unlocked_this_device`|Device unlocked|No|
|`first_unlock`|After first unlock since boot (works in background)|Yes (backup)|
|`first_unlock_this_device`|After first unlock since boot|No|
|`passcode`|Only if a device passcode is set|No|

**Recommended default for tokens:** `first_unlock_this_device`. Background tasks still work and the item does not leave the device. Use `unlocked_this_device` for stricter foreground-only access.

---

## 5. Wrapper class (recommended)

Keep key names in one place and avoid typos.

```dart
class SecureStore {
  SecureStore([FlutterSecureStorage? storage])
      : _s = storage ?? const FlutterSecureStorage(
          iOptions: IOSOptions(
            accessibility: KeychainAccessibility.first_unlock_this_device,
          ),
        );

  final FlutterSecureStorage _s;

  static const _kAccess = 'access_token';
  static const _kRefresh = 'refresh_token';

  Future<void> saveTokens({required String access, required String refresh}) =>
      Future.wait([
        _s.write(key: _kAccess, value: access),
        _s.write(key: _kRefresh, value: refresh),
      ]);

  Future<String?> get accessToken => _s.read(key: _kAccess);
  Future<String?> get refreshToken => _s.read(key: _kRefresh);

  Future<void> clear() => _s.deleteAll();
}
```

---

## 6. Dio interceptor with token refresh

```dart
class AuthInterceptor extends QueuedInterceptor {
  AuthInterceptor(this._store, this._dio);
  final SecureStore _store;
  final Dio _dio;

  @override
  Future<void> onRequest(RequestOptions o, RequestInterceptorHandler h) async {
    final token = await _store.accessToken;
    if (token != null) o.headers['Authorization'] = 'Bearer $token';
    h.next(o);
  }

  @override
  Future<void> onError(DioException e, ErrorInterceptorHandler h) async {
    if (e.response?.statusCode == 401) {
      final refresh = await _store.refreshToken;
      if (refresh != null) {
        try {
          // use a clean Dio instance to avoid interceptor loops
          final res = await Dio().post('$baseUrl/auth/refresh',
              data: {'refresh_token': refresh});
          await _store.saveTokens(
            access: res.data['access_token'],
            refresh: res.data['refresh_token'],
          );
          final req = e.requestOptions
            ..headers['Authorization'] = 'Bearer ${res.data['access_token']}';
          return h.resolve(await _dio.fetch(req));
        } catch (_) {
          await _store.clear();   // refresh failed: force re-login
        }
      }
    }
    h.next(e);
  }
}
```

---

## 7. Gotchas

> [!bug] iOS Keychain survives uninstall After deleting and reinstalling the app, old Keychain items can still be there, so a "logged out" user may appear logged in. Fix: on first launch, store a flag in `SharedPreferences` (which **is** wiped on uninstall). If the flag is missing but the Keychain has data, call `deleteAll()`.

```dart
Future<void> clearKeychainOnFreshInstall() async {
  final prefs = await SharedPreferences.getInstance();
  if (!(prefs.getBool('installed_before') ?? false)) {
    await const FlutterSecureStorage().deleteAll();
    await prefs.setBool('installed_before', true);
  }
}
```

> [!bug] Android backup / restore breaks decryption If Auto Backup restores encrypted data to a new device, the Keystore key is **not** restored, so reads throw errors. Disable backup for this data:
> 
> ```xml
> <application
>     android:allowBackup="false"
>     ... >
> ```
> 
> Or use `android:fullBackupContent` / `dataExtractionRules` to exclude the secure-storage files.

> [!bug] Decryption exceptions Reads can throw `PlatformException` (e.g. `BadPaddingException`, key invalidated after a lock-screen change or OS update). Handle it:
> 
> ```dart
> Future<String?> safeRead(String key) async {
>   try {
>     return await storage.read(key: key);
>   } on PlatformException {
>     await storage.deleteAll();   // corrupted: reset and re-authenticate
>     return null;
>   }
> }
> ```

> [!bug] Other common issues
> 
> - **Slow first read** on some Android devices (Keystore init). Don't read it during a hot path like every frame or every list item.
> - **Reading before `runApp`** may need `WidgetsFlutterBinding.ensureInitialized()`.
> - **Background isolates** (e.g. WorkManager) need `first_unlock` accessibility on iOS.
> - **Web:** keys end up in `localStorage`; do not rely on it for sensitive data.

---

## 8. What to store (and not)

|Store here|Do **not** store here|
|---|---|
|Access / refresh tokens|API secret keys baked in at build time (see [[envied vs flutter_dotenv]])|
|Session ID|Large data (images, JSON lists, cached responses)|
|DB encryption key (e.g. for Hive / SQLCipher)|Frequently changing UI state|
|Biometric-protected credentials|Plain user preferences (use `SharedPreferences`)|
|Device-bound pairing secrets|Anything you can fetch from the server when needed|

### Pattern: encrypted Hive box

```dart
const storage = FlutterSecureStorage();
var keyB64 = await storage.read(key: 'hive_key');
if (keyB64 == null) {
  final key = Hive.generateSecureKey();
  keyB64 = base64UrlEncode(key);
  await storage.write(key: 'hive_key', value: keyB64);
}
final box = await Hive.openBox(
  'private',
  encryptionCipher: HiveAesCipher(base64Url.decode(keyB64)),
);
```

Large data lives in Hive; only the small key lives in secure storage.

---

## 9. Add biometrics (optional)

`flutter_secure_storage` does not prompt for biometrics by default. Combine it with `local_auth` to gate access:

```dart
Future<String?> readWithBiometrics(String key) async {
  final ok = await LocalAuthentication().authenticate(
    localizedReason: 'Authenticate to continue',
    options: const AuthenticationOptions(biometricOnly: true),
  );
  return ok ? storage.read(key: key) : null;
}
```

This is an app-level gate (it can be bypassed on a hooked device). Some versions also expose OS-enforced access-control flags on iOS; check the package docs if you need that.

---

## 10. Testing

```dart
setUp(() {
  FlutterSecureStorage.setMockInitialValues({'access_token': 'abc'});
});

test('reads token', () async {
  final store = SecureStore();
  expect(await store.accessToken, 'abc');
});
```

Or inject a mock `FlutterSecureStorage` (via `mocktail`) into the wrapper class.

---

## 11. Auth flow with Cubit

```dart
class AuthCubit extends Cubit<AuthState> {
  AuthCubit(this._store, this._repo) : super(const AuthState.unknown());

  Future<void> checkSession() async {
    final token = await _store.accessToken;
    emit(token == null
        ? const AuthState.unauthenticated()
        : const AuthState.authenticated());
  }

  Future<void> logout() async {
    await _repo.logout();     // best-effort server revoke
    await _store.clear();
    emit(const AuthState.unauthenticated());
  }
}
```

See also [[StateBox Pattern (flutter_bloc)]].

---

## 12. Alternatives

|Package|Use when|
|---|---|
|`shared_preferences`|Non-sensitive settings (plain text on device)|
|`hive` + encryption|Larger local data, key kept in secure storage|
|`sqflite_sqlcipher` / Drift + SQLCipher|Encrypted relational DB|
|`encrypt` / `cryptography`|Custom crypto (you still need somewhere safe for the key)|
|`flutter_keychain`|Older / less maintained; prefer `flutter_secure_storage`|

---

## 13. Checklist

- [ ] Only runtime secrets stored (tokens, session, DB key), no build-time secrets
- [ ] Short-lived access token + refresh token rotation on the backend
- [ ] iOS accessibility set deliberately (`first_unlock_this_device` for most apps)
- [ ] Android `allowBackup="false"` (or backup exclusion rules)
- [ ] `deleteAll()` on logout, and on fresh install (iOS Keychain persistence)
- [ ] `try/catch` around reads, with reset-and-reauth fallback
- [ ] No tokens in logs, `SharedPreferences`, or crash reports
- [ ] Wrapper class with centralized key names
- [ ] Verified the package's current `minSdk` and API on pub.dev (it changes between major versions)
- [ ] Biometric gate added where the data is highly sensitive

> [!tip] Rule of thumb Secure storage protects data **at rest on the device**. It does not protect a secret that was shipped inside the app, and it does not protect against a fully compromised device. Pair it with short-lived tokens and server-side revocation.
