
# Flutter Security: Hard-coded Secrets

> [!danger] Core rule **Anything shipped inside the app can be extracted.** Obfuscation, `.env` files, and `--dart-define` only make extraction slower. They do not make a secret safe. A real secret must never live in the client.

Related: OWASP Mobile Top 10 (M1: Improper Credential Usage), OWASP MASVS (MASVS-STORAGE, MASVS-CRYPTO).

---

## 1. What counts as a secret?

| Type                            | Examples                                                                                  | Safe in app?                           |
| ------------------------------- | ----------------------------------------------------------------------------------------- | -------------------------------------- |
| Server credentials              | DB passwords, admin tokens, JWT signing keys                                              | **Never**                              |
| Third-party secret keys         | Stripe `sk_live_...`, SendGrid, Twilio auth token, OpenAI / LLM API keys, AWS secret keys | **Never**                              |
| OAuth client secret             | Client secret for auth code flow                                                          | **Never** (use PKCE, no secret)        |
| Encryption keys                 | Static AES key, hard-coded IV                                                             | **Never**                              |
| Public / restricted identifiers | Firebase config, Google Maps key (restricted), Stripe `pk_...`, Sentry DSN                | Yes, with restrictions                 |
| Runtime user tokens             | Access / refresh tokens                                                                   | Yes, in secure storage (not in source) |

> [!note] Firebase keys Firebase API keys in `google-services.json` and `firebase_options.dart` are **identifiers, not secrets**. Security comes from Firestore / Storage rules, Auth, and App Check, not from hiding the key.

---

## 2. How attackers extract them

Flutter release builds compile Dart to native AOT (`libapp.so`), but strings stay in the binary.

```bash
# 1. Unzip the APK
unzip app-release.apk -d out/

# 2. Strings are readable in the AOT snapshot
strings out/lib/arm64-v8a/libapp.so | grep -iE "api[_-]?key|secret|token|sk_live|AIza"

# 3. Assets are plain files (e.g. a bundled .env)
cat out/assets/flutter_assets/.env
```

Common tools: `apktool`, `jadx`, `strings`, `Frida` (runtime hooking), `reFlutter` / `Blutter` (Dart AOT analysis), and proxies like `mitmproxy` / Burp to read secrets from network traffic.

> [!warning] Even if a secret never appears in the binary, anything the app sends over the network can be seen by someone controlling their own device (proxy + Frida to bypass pinning).

---

## 3. Bad patterns (do not do this)

```dart
// Hard-coded in source
const stripeSecret = 'sk_live_51H...';
const openAiKey = 'sk-proj-...';

// "Hidden" with base64: not encryption
final key = utf8.decode(base64.decode('c2stbGl2ZS0uLi4='));

// .env bundled as an asset (flutter_dotenv): shipped inside the APK
await dotenv.load(fileName: '.env');   // listed under flutter: assets:

// Committed to git
// lib/config/secrets.dart  (and never added to .gitignore)
```

Why each fails:

- Base64, XOR, and string splitting are reversible in minutes.
- `flutter_dotenv` bundles the `.env` file as an asset, so it is readable by anyone who unzips the app.
- Git history keeps secrets forever, even after the file is deleted.

---

## 4. Safer approaches

### 4.1 Move the secret to a backend (best fix)

The app calls **your** API. Your server holds the real key and calls the third party.

```
[Flutter App] --user JWT--> [Your Backend] --secret key--> [Stripe / OpenAI / SMS / Email]
```

```dart
// Client: no third-party secret anywhere
final res = await dio.post('/api/ai/summarize', data: {'text': text});
```

Backend adds: authentication, rate limiting, per-user quotas, and logging. This also prevents someone abusing your paid API quota.

### 4.2 Use short-lived, scoped tokens

- Backend issues a **temporary token** (minutes) for one action (e.g. a signed upload URL for S3 / Cloud Storage, a Stripe ephemeral key).
- Limits damage if the token is stolen.

### 4.3 Restrict keys that must be in the client

|Key|Restriction|
|---|---|
|Google Maps / Places|Restrict by Android package name + SHA-1, iOS bundle ID, and enabled APIs only|
|Firebase|Security rules + Firebase **App Check** (Play Integrity / App Attest)|
|Sentry / analytics|Use the public DSN only, set rate limits|
|Stripe|`pk_` publishable key only, secrets stay on the server|

### 4.4 Inject build-time config (for non-secret config)

Good for environment URLs and feature flags, **not** for true secrets.

```bash
flutter run --dart-define=API_BASE_URL=https://api.example.com
flutter build apk --dart-define-from-file=config/prod.json
```

```dart
const baseUrl = String.fromEnvironment('API_BASE_URL');
```

```json
// config/prod.json (add to .gitignore if it contains anything sensitive)
{
  "API_BASE_URL": "https://api.example.com",
  "MAPS_KEY": "AIza...restricted-key"
}
```

> [!warning] `--dart-define` values are still compiled into the binary and are extractable.

### 4.5 `envied` with obfuscation (speed bump only)

```dart
// env.dart
import 'package:envied/envied.dart';
part 'env.g.dart';

@Envied(path: '.env', obfuscate: true)
abstract class Env {
  @EnviedField(varName: 'MAPS_KEY', obfuscate: true)
  static final String mapsKey = _Env.mapsKey;
}
```

It is better than plain strings (no readable text in `libapp.so`), but a determined attacker can still recover values. Use it for restricted client keys only.

### 4.6 Store runtime secrets securely

For tokens obtained **after** login:

```dart
const storage = FlutterSecureStorage(
  aOptions: AndroidOptions(encryptedSharedPreferences: true),
  iOptions: IOSOptions(accessibility: KeychainAccessibility.first_unlock_this_device),
);

await storage.write(key: 'refresh_token', value: token);
final token = await storage.read(key: 'refresh_token');
```

Avoid `SharedPreferences` for tokens: it is plain text on the device.

### 4.7 OAuth: use PKCE, not a client secret

Mobile apps are **public clients**. Use Authorization Code + PKCE (e.g. `flutter_appauth`) and never embed a client secret.

---

## 5. Hardening (defense in depth)

```bash
# Obfuscate Dart symbols (renames classes/functions; does NOT encrypt strings)
flutter build apk --release --obfuscate --split-debug-info=build/symbols
flutter build appbundle --release --obfuscate --split-debug-info=build/symbols
```

- Keep `build/symbols` private: it is needed to de-obfuscate crash stack traces.
- **Android:** enable R8 / ProGuard (`minifyEnabled true`, `shrinkResources true`) for the native / Java layer.
- **Certificate pinning:** `dio` with a custom `SecurityContext` or a pinning package, to resist traffic interception.
- **Root / jailbreak and tamper detection:** treat as a deterrent, not a guarantee.
- **App attestation:** Play Integrity API (Android) and App Attest / DeviceCheck (iOS) so the backend can confirm requests come from your genuine app. Firebase App Check wraps these.
- Disable debug logging in release (`kReleaseMode`), and never log tokens or keys.

---

## 6. Prevent leaks in the repo

```gitignore
.env
.env.*
config/*.json
*.jks
*.keystore
key.properties
google-services.json   # optional: only if your policy requires it
GoogleService-Info.plist
```

- Pre-commit scanning: `gitleaks`, `trufflehog`, or `detect-secrets`.
- GitHub **Secret scanning** and **Push protection** (repo settings).
- CI/CD: store values in GitHub Actions secrets / GitLab CI variables / Codemagic env vars, and inject them at build time.

```yaml
# .github/workflows/build.yml (snippet)
- name: Build
  run: flutter build apk --dart-define=API_BASE_URL=${{ secrets.API_BASE_URL }}
```

Quick local scan:

```bash
gitleaks detect --source . --verbose
```

---

## 7. If a secret leaked

1. **Revoke / rotate immediately.** Do this first, before cleaning git.
2. Check provider logs for abuse (API usage, billing spikes).
3. Remove from history (`git filter-repo` or BFG), then force-push. Assume it is already compromised, since forks and clones may have it.
4. Ship an app update that no longer needs the old key. Old app versions may still use it, so consider a forced-update mechanism.
5. Add scanning so it does not happen again.

---

## 8. Decision guide

|Question|Answer|
|---|---|
|Does the key grant write or billing access?|Backend only|
|Is it an identifier that is safe by design (Firebase, `pk_`)?|Client, add restrictions and App Check|
|Is it a per-user token from login?|`flutter_secure_storage`|
|Is it an environment URL or feature flag?|`--dart-define-from-file`|
|Do you really need it on-device?|Use a short-lived token from your backend|

---

## 9. Checklist

- [ ] No `sk_`, `secret`, private keys, or passwords in Dart code, assets, or `AndroidManifest.xml` / `Info.plist`
- [ ] `.env` is **not** listed under `flutter: assets:`
- [ ] Third-party secret calls go through my backend
- [ ] Client-side keys (Maps, etc.) restricted by package name, SHA-1, bundle ID, and API scope
- [ ] Tokens stored in `flutter_secure_storage`
- [ ] Release built with `--obfuscate --split-debug-info`
- [ ] Certificate pinning considered for sensitive APIs
- [ ] Firebase rules + App Check enabled
- [ ] `gitleaks` / push protection active
- [ ] Rotation plan documented
- [ ] Decompiled my own release APK to verify nothing sensitive is visible

> [!tip] Self-test Build a release APK, then run `unzip` + `strings libapp.so | grep -i key` and `apktool d app.apk`. If you can find a secret in 5 minutes, so can an attacker.
