# OPSS Driver App (opss-driver)

Android driver-side app for the OPSS ride service, package **`ir.opss.driver`**.
This repository was rebranded from the vendor app (Alaadcin/ecardo); the code
package, application id, backend hosts and CI all target OPSS now.

## Backend configuration

A single source of truth in `app/build.gradle`:

| BuildConfig field       | Default                 | Purpose                              |
|-------------------------|-------------------------|--------------------------------------|
| `SERVER_BASE_URL`       | `https://opss.ir/`      | Primary backend (serves `webservice_shark.php`) |
| `SERVER_FALLBACK_URL`   | `http://46.100.58.20/`  | Used only while the primary host is unreachable |

> **2026-10-11:** the origin server moved to a new IP, so the fallback default
> above is stale until the new IP is wired in. The primary domain is fronted by
> Cloudflare, so the app only needs the fallback while DNS is unavailable.

Override per build without touching the repo:

```bash
./gradlew assembleProdDebug -PSERVER_BASE_URL=https://your-host/ -PSERVER_FALLBACK_URL=http://backup-ip/
```

The URLs flow into `com.utils.CommonUtilities.SERVER` and
`com.general.files.MyApp` at compile time — there are no hardcoded hosts left
in the sources.

## Build

```bash
./gradlew assembleProdDebug     # what CI builds and releases
```

Requirements: JDK 11 for Gradle, Android SDK with build-tools 31 and the NDK
version pinned in `app/build.gradle` (`22.0.7026061`).

### Release signing

`app/build.gradle` signs release builds **only if** the vendor keystore
`foodtakila.jks` exists at one of:

- `C:/Users/sitw2/Documents/foodtakila.jks` (Windows build machine)
- `/opt/opss/foodtakila.jks` (Linux/CI)

Without it the build produces a debug-signed APK, which cannot install over a
vendor-signed copy and must not go to Cafe Bazaar. **The keystore is not in
this repository by design — do not commit it.**

## CI (`.github/workflows`)

- **`build.yml`** — runs on every push to `main` (and `fix/**` branches):
  1. `bump` (main pushes only) bumps the patch version + versionCode and
     commits, so the build that follows really contains the released version.
  2. `build` produces the debug APK (`opss-<slug>-<version>.apk`).
  3. `smoke-test` and `runtime-logs` install it on an API 30 emulator, launch
     it and scan logcat for crashes.
  4. `release` publishes the APK as a GitHub release tagged `v<version>` —
     tag, asset name and the APK's internal versionName always match.
- **`verify-apk.yml`** — manual dispatch; downloads the latest release and
  gates on: zip integrity, manifest zip-layout rules (targetSdk 29), signature
  scheme, ABI list, presence of the two configured hosts in the dex, absence
  of the old vendor host, and a live probe of both backend endpoints
  (a hosting-panel page answering 200 is treated as *not deployed*).

## Known open items

1. **The PHP API is not deployed yet.** `https://opss.ir/webservice_shark.php`
   (and the fallback IP) currently serve the FastPanel default page, so no
   backend request can succeed until the webservice is deployed on the server.
2. **Release keystore missing** — see above; needed for a store-signed build.
3. **Firebase config is provisional.** `app/google-services.json` still points
   at the old Firebase project (`alaadcin-c3cd4`). Registering
   `ir.opss.driver` in a Firebase console and dropping in a fresh
   `google-services.json` is required for FCM push and Crashlytics to work;
   until then the app builds and runs, but push notifications will not.
4. **Rotate the leaked credentials** from the earlier migration session (the
   GitHub PAT used there must be revoked).
