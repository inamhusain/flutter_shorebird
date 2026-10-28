# Shorebird session: from release to patch in production

**Length:** 60 min · **Who it's for:** Flutter developers · **Format:** live demo plus hands-on

## What you'll be able to do

1. Ship a store release with Shorebird
2. Fix a bug with a patch, with no store review
3. Review the patch on a staging track before users get it
4. Promote the patch to everyone, or roll it back
5. Show users the fix on the first launch with a manual update gate

## Agenda

| # | Topic | Time |
|---|-------|------|
| 0 | What Shorebird is, how it works, setup | 10 min |
| 1 | Create a release | 10 min |
| 2 | Create a patch | 10 min |
| 3 | Review the patch (staging) | 10 min |
| 4 | Release the patch (promote / rollback) | 5 min |
| 5 | Manual update: the fix on the first launch | 12 min |
| – | Q&A | 3 min |

---

## 0. What Shorebird is and how it works

**What it is:** Shorebird is a code push tool for Flutter. It lets you send Dart code fixes straight to users' phones. You don't need a new store build or app store review.

**Why it matters:**
- A store release takes hours to days because of review and slow user updates. A patch reaches users within minutes.
- Users don't need to update from the store. The app updates itself.
- It follows store rules: only interpreted Dart code changes, never native code.

**How it works:**

```
 shorebird release ──► Shorebird keeps a copy of the release's Dart code
        │
        ▼
 Upload the .aab/.ipa to the Play Store / App Store (normal review, one time)
        │
 Bug found → fix the Dart code
        │
        ▼
 shorebird patch ──► Shorebird compares the new code with the release and makes a small diff (the patch)
        │
        ▼
 Shorebird cloud (console) ──► app launch: the app asks "any patch for my version?"
        │
        ▼
 Patch downloads in the background → applies on the NEXT launch
```

- Shorebird builds the app with its own fork of the Flutter engine, which includes an **updater**.
- **Android:** the patched Dart code runs as AOT machine code, so it's as fast as normal.
- **iOS:** Apple doesn't allow new machine code to be downloaded. Shorebird runs only the *changed* functions in a custom interpreter and everything else at full AOT speed. iOS patches only run on real arm64 devices, not on Simulators.
- Patches are tied to **one exact release version** (`1.0.0+1`). A different version never gets them.
- After a new patch installs and launches successfully, the previous patch is deleted.

### Setup

```bash
curl -fsSL https://shorebird.dev/install.sh | bash
shorebird login
shorebird init        # creates the app_id, writes shorebird.yaml, adds it to pubspec assets (one app_id per flavor)
shorebird doctor      # checks the setup and shows the Flutter version Shorebird uses
```

- **Release** = the build you upload to the Play Store or App Store. Shorebird records its Dart code.
- **Patch** = new Dart code for one exact release version. It downloads over the air.
- The app checks for a patch on launch. It downloads in the background and **applies on the next launch**.

## 1. Create a release

### Version and build number

The release version is `<version>+<build number>`, for example `1.0.0+1`. Set it one of two ways:

**A. In `pubspec.yaml`** (the default):

```yaml
version: 1.0.0+1      # 1.0.0 = version (build-name), 1 = build number
```

**B. On the command line** (overrides pubspec). `--build-name` and `--build-number` are `shorebird release` options:

```bash
shorebird release android --build-name=1.0.0 --build-number=1
shorebird release ios     --build-name=1.0.0 --build-number=1
```

Any other `flutter build` argument goes after a `--` separator, for example `shorebird release android -- --dart-define="foo=bar"`. In PowerShell, quote the separator as `'--'`.

- Bump the build number for every store upload, because the stores reject a duplicate build number.
- Each `version+build` is a **separate release** in Shorebird. Patches only reach that exact pair.

### Flutter version

```bash
shorebird flutter versions list                          # Flutter versions Shorebird supports, ✓ = default
shorebird --version                                      # also shows the current Flutter version
shorebird release android --flutter-version=3.47.7       # pin a Flutter version
shorebird release android --flutter-version=fvm          # use the version from the project's .fvmrc
shorebird release android --flutter-version=system       # use the flutter on your PATH
```

- If you leave out `--flutter-version`, Shorebird uses the Flutter version bundled with Shorebird (the latest stable), marked with ✓ in the list.
- Only `shorebird release` takes `--flutter-version`. Patches are always built with the release's Flutter version, so you never pass it when patching.
- `3.47.7` is the example from the Shorebird docs. Pick a version from `shorebird flutter versions list`.
- Use the same Flutter version for Android and iOS so the two platforms stay in sync.

### Full release commands

```bash
# Android: makes an .aab (default), upload it to Play Console
shorebird release android --flutter-version=3.47.7 --build-name=1.0.0 --build-number=1

# Android APK (for side-loading or testing)
shorebird release android --artifact=apk --flutter-version=3.47.7 --build-name=1.0.0 --build-number=1

# iOS: makes an .ipa, upload it to App Store Connect (Transporter / Xcode)
shorebird release ios --flutter-version=3.47.7 --build-name=1.0.0 --build-number=1

# Both platforms in one command
shorebird release --platforms=android,ios --flutter-version=3.47.7 --build-name=1.0.0 --build-number=1

# With flavors / a different entry point (optional)
shorebird release android --flavor=prod --target=lib/main_prod.dart --build-name=1.0.0 --build-number=1
```

### Check the release

```bash
shorebird releases list                              # all releases
shorebird releases info --release-version 1.0.0+1     # details of one release
```

The release also appears in the [Shorebird console](https://console.shorebird.dev).

- Demo: release a build that has a known bug (for example, tapping Pay crashes).

## 2. Create a patch

Fix the bug in Dart, then run:

```bash
shorebird patch android --release-version=1.0.0+1 --track=staging --dry-run   # build and validate, no upload
shorebird patch android --release-version=1.0.0+1 --track=staging
shorebird patch ios     --release-version=1.0.0+1 --track=staging
shorebird patch --platforms=android,ios --release-version=1.0.0+1 --track=staging   # both at once
```

- `--release-version`: if you leave it out, the CLI uses the release version from the compiled build (pubspec). `--release-version=latest` targets the most recently updated release.
- `--track`: defaults to `stable` (all users). Use `staging` so you can review the patch first.
- With flavors, use the same flag as the release: `shorebird patch android --flavor=prod`.

**Which files a patch can include**

| File / folder | In a patch? | Notes |
|---------------|-------------|-------|
| `lib/**/*.dart` | Yes | Screens, widgets, logic, bug fixes, API calls, text strings in code |
| `pubspec.yaml` and `pubspec.lock` (Dart-only packages) | Yes | Adding, removing or upgrading packages with **no native code** (for example `intl`, `provider`, `equatable`) |
| Generated Dart (`*.g.dart`, `*.freezed.dart`, `l10n` output) | Yes | It's Dart, so it compiles into the patch |
| `assets/` (images, JSON, fonts, files read at runtime) | No | Asset patching isn't supported yet. The CLI warns you or blocks the upload, so this needs a store release |
| `android/` (Kotlin, Java, Gradle, `AndroidManifest.xml`) | No | Native code, needs a store release |
| `ios/` (Swift, Obj-C, `Info.plist`, Podfile) | No | Native code, needs a store release |
| Plugins with native code (`camera`, `firebase_*`, …) | Only if already in the release | Using a plugin that's already in the release binary is fine. Adding a new one, or upgrading one whose native code changes, needs a store release |
| `version:` in `pubspec.yaml` | Don't change it | The patch must target the release version exactly |
| `shorebird.yaml` | No | It's an asset that's bundled into the release build |
| Flutter SDK version | No | Patch uses the same Flutter version as the release |

Rule of thumb: **if it compiles into Dart, it can go in a patch. Anything else needs a store release.**

The CLI checks for asset and native changes before it uploads. `--allow-asset-diffs` and `--allow-native-diffs` publish anyway. The docs mark both as **not recommended**.

## 3. Review the patch (staging)

```bash
shorebird patches list --release-version 1.0.0+1                  # see the patch number and its track
# app_id is in shorebird.yaml (one per flavor)
shorebird preview --app-id <your-app-id> --release-version 1.0.0+1 --track staging --platform android
shorebird preview --app-id <your-app-id> --release-version 1.0.0+1 --track staging --platform ios
```

- Installs the release on a device and points it at the staging track. `--app-id` picks the app (or flavor). Copy it from `shorebird.yaml`.
- The patch downloads in the background, so **kill and relaunch the app twice** to see it.
- iOS patches don't run on Simulators, so use a real iPhone.
- Review checklist:
  - [ ] Bug is fixed
  - [ ] No new crashes on the main flows
  - [ ] It works on both Android and iOS
  - [ ] Patch size and install count look right in the [Shorebird console](https://console.shorebird.dev)

## 4. Release the patch

**Promote staging → stable** (all users):

```bash
shorebird patches set-track --release-version 1.0.0+1 --patch-number 1 --track stable
```

Or in the [console](https://console.shorebird.dev): open the release details page, choose **Change Track** and select **stable**. These flag names need Shorebird CLI 1.6.128 or newer, so run `shorebird upgrade` first if needed.

**Roll back** a bad patch:

```bash
shorebird patches rollback --release-version 1.0.0+1 --patch-number 1
```

Or in the console: open the release, click the three-dot menu on the patch and choose **Rollback**. Devices remove the patch and go back to the previous patch, or to the base release, on their next launch.

- Skip staging when needed: `shorebird patch android` (no `--track`) goes straight to `stable`.

## 5. Manual update: the fix on the first launch

**Problem:** the patch applies only on the *next* launch, so each user sees the bug one more time after we fix it.

**Fix:** an update gate. A loading screen checks for a patch, downloads it and restarts the app before the real app shows.

```
App opens → "Checking for updates" → new patch?
   yes → download → restart (Android) / ask user to reopen (iOS, desktop) → fixed app
   no / no internet / timeout → open the app normally
```

**Step 1. Turn off automatic updates** in `shorebird.yaml`:

```yaml
app_id: your-app-id
auto_update: false
```

**Step 2. Add packages** to `pubspec.yaml`:

```yaml
dependencies:
  shorebird_code_push: ^2.0.7
  restart_app: ^1.10.2
```

**Step 3. Add the gate** in `lib/main.dart` (replace `MyRealApp` with your app widget):

```dart
import 'dart:async';
import 'dart:io';
import 'package:flutter/material.dart';
import 'package:shorebird_code_push/shorebird_code_push.dart';
import 'package:restart_app/restart_app.dart';

void main() => runApp(const UpdateGate());

class UpdateGate extends StatefulWidget {
  const UpdateGate({super.key});
  @override
  State<UpdateGate> createState() => _UpdateGateState();
}

class _UpdateGateState extends State<UpdateGate> {
  final _updater = ShorebirdUpdater();
  String _message = 'Checking for updates...';
  bool _ready = false;

  @override
  void initState() {
    super.initState();
    _checkAndApply();
  }

  Future<void> _checkAndApply() async {
    // Shorebird only works in release builds made with Shorebird
    if (!_updater.isAvailable) {
      setState(() => _ready = true);
      return;
    }
    try {
      final status = await _updater.checkForUpdate().timeout(const Duration(seconds: 5));
      if (status == UpdateStatus.outdated) {
        setState(() => _message = 'Downloading update...');
        await _updater.update().timeout(const Duration(seconds: 30));
        _restart();
        return;
      }
      if (status == UpdateStatus.restartRequired) {
        // Patch was downloaded before. Apply it now.
        _restart();
        return;
      }
    } catch (_) {
      // No internet, timeout or error: do not block the user
    }
    setState(() => _ready = true);
  }

  void _restart() {
    if (Platform.isAndroid) {
      setState(() => _message = 'Applying update...');
      Restart.restartApp();
    } else {
      // iOS and desktop: the app cannot restart itself
      setState(() => _message = 'Update installed. Please close and reopen the app.');
    }
  }

  @override
  Widget build(BuildContext context) {
    if (_ready) return const MyRealApp();
    return MaterialApp(
      debugShowCheckedModeBanner: false,
      home: Scaffold(
        body: Center(
          child: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              const CircularProgressIndicator(),
              const SizedBox(height: 16),
              Text(_message, textAlign: TextAlign.center),
            ],
          ),
        ),
      ),
    );
  }
}
```

**Platform support**

| Platform | Supported | What happens |
|----------|-----------|--------------|
| Android | Yes | Downloads and restarts by itself. Fully automatic. |
| iOS | Yes | Downloads. The user must close and reopen, because Apple doesn't let apps restart themselves. |
| macOS, Windows, Linux | Yes | Same as iOS. |
| Web | No | Shorebird has no web support, and `dart:io` won't build for web. |

**Things to know**

- The gate ships in the store build, so new installs are covered from the first launch.
- A patch only reaches the exact release version it was made for.
- Test in release mode. Shorebird doesn't work in debug builds.
- The gate needs its own **store release**, because `auto_update` lives in `shorebird.yaml`. You can't add the gate with a patch.

**Live demo:** release with the gate → patch the Pay crash → open the app → the user never sees the crash.

---

## Q&A / cheat sheet

```bash
# setup
curl -fsSL https://shorebird.dev/install.sh | bash
shorebird login
shorebird init
shorebird doctor
shorebird flutter versions list

# release
shorebird release android --flutter-version=3.47.7 --build-name=1.0.0 --build-number=1
shorebird release ios     --flutter-version=3.47.7 --build-name=1.0.0 --build-number=1
shorebird releases list

# patch → review → promote / rollback
shorebird patch android --release-version 1.0.0+1 --track staging
shorebird patches list  --release-version 1.0.0+1
shorebird preview       --app-id <your-app-id> --release-version 1.0.0+1 --track staging --platform android
shorebird patches set-track --release-version 1.0.0+1 --patch-number 1 --track stable
shorebird patches rollback  --release-version 1.0.0+1 --patch-number 1
```

Console: https://console.shorebird.dev
Docs: https://docs.shorebird.dev
