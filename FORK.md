# Mangaku

Mangaku is a rebranded fork of [Mihon](https://github.com/mihonapp/mihon), licensed under Apache 2.0 like upstream.
It differs from upstream in four ways:

| Change | Where | Why |
|---|---|---|
| Package ID `app.mangaku`, name "Mangaku" | `app/build.gradle.kts`, `i18n/.../base/strings.xml` | Installs next to Mihon instead of clashing with it. Mihon asks forks to rebrand. |
| Telemetry never built in unless `-Pinclude-telemetry=true` | `gradle/build-logic/.../BuildConfig.kt` | Upstream turns Firebase Analytics and Crashlytics on by default for `ci` and `github` builds. |
| In-app updater reads this repo's releases | `app/.../updater/AppUpdateChecker.kt` | Upstream APKs carry a different signature and can't be installed over Mangaku. |
| CI and releases run on this repo, signed with your key | `.github/workflows/` | `update_website.yml` was removed because it only pings Mihon's website. |

## About the "Virus app detected" warning

The OEM security scanner (ColorOS/realme/Xiaomi-style "Phone Manager") showed one generic message for a whole category: *"send unauthorised SMS, use adult content to induce payments, install apps without permission, or steal private information"*.
The source code audit (upstream commit `4823ba69c`) found:

| Claim | Finding |
|---|---|
| Sends SMS | **No.** No `SEND_SMS`/`READ_SMS` permission and no `SmsManager` call anywhere. |
| Installs apps without permission | **Partly true, by design.** It requests `REQUEST_INSTALL_PACKAGES`, `REQUEST_DELETE_PACKAGES`, `UPDATE_PACKAGES_WITHOUT_USER_ACTION` and `QUERY_ALL_PACKAGES` to install, update and list extension APKs. A silent install only happens if you set up Shizuku yourself. Otherwise Android asks you every time. |
| Adult content | Many sources and extensions are tagged NSFW (`tachiyomi.extension.nsfw`), and the library holds adult manhwa. |
| Steals private info | **No sign of it in the code.** Upstream release builds include Firebase Analytics and Crashlytics, which Mangaku leaves out. |

The heuristics most likely to trip are the install permissions plus loading code from other APKs (`DelegateLastClassLoaderCompat` loads each extension's dex into the app process), on a sideloaded app that isn't from a store.

**The real risk is extensions, not the app.** An extension is arbitrary code that runs inside the app with its permissions. Only add repos you trust.

### Checking that your installed Mihon is genuine

The code above only covers the source. An APK from a mirror site could be repackaged. To check yours:

1. Download the APK only from <https://github.com/mihonapp/mihon/releases>.
2. Install it **over** your current Mihon, without uninstalling first.
3. If Android accepts it as an update, your copy has the same signature as the official one. If you get "App not installed" or a signature conflict, your copy came from someone else: back it up, uninstall it, and install the official APK.

## Signing releases

To keep each release installable as an update, every release needs the same key. Generate the key **on your own machine** and never commit it:

```sh
keytool -genkeypair -v -keystore mangaku.keystore -alias mangaku \
  -keyalg RSA -keysize 4096 -validity 10000
base64 -w0 mangaku.keystore > mangaku.keystore.b64   # macOS: base64 -i mangaku.keystore
```

Then add these four secrets under **Settings → Secrets and variables → Actions**:

| Secret | Value |
|---|---|
| `SIGNING_KEY` | Contents of `mangaku.keystore.b64` |
| `KEY_STORE_PASSWORD` | Keystore password |
| `ALIAS` | `mangaku` |
| `KEY_PASSWORD` | Key password |

Keep a backup of `mangaku.keystore`. If you lose it, users can't update without uninstalling.

To release, push a tag such as `git tag v0.21.3 && git push origin v0.21.3`. `release.yml` builds and signs the APKs, then creates a **draft** release. Publish the draft to make it visible to the in-app updater.

`build.yml` runs on `main`, on `ccr-*` branches, and on demand. It produces an APK as a workflow artifact. Each run signs that APK with a throwaway debug key, so use it for testing only.

## Keeping up with upstream

```sh
git remote add upstream https://github.com/mihonapp/mihon.git
git fetch upstream
git merge upstream/main     # conflicts, if any, will be in the files listed above
```

## Known leftovers

- Tracker OAuth still uses the `app.mihon://` redirect scheme, because the tracker sites only accept the URI Mihon registered. With both apps installed, Android may ask which app should handle the login.
- The launcher icon is still Mihon's (`app/src/main/res/mipmap-*`, `icons/`).
- Help links in settings still point to mihon.app.
