# CLAUDE.md

Guidance for Claude Code (and other AI assistants) working in this repository.

## Project overview

CodeAssist is an Android application (`com.tyron.code`) that is itself an IDE: it lets
users write, build, and run Android apps (Java/Kotlin) directly on an Android device,
using on-device `javac`/`kotlinc` compilation, R8/ProGuard, dependency resolution, and a
layout preview — no desktop machine required.

- Distributed on Google Play and via APK releases (see `changelogs/`).
- License: GPL-3.0 (`LICENSE`). Contributions are accepted under the same license
  (`CONTRIBUTING.md`).
- Community: Discord and Telegram links are in `README.md`.

## Build system

This is a multi-module **Gradle** project (Groovy DSL, not Kotlin DSL) targeting Android.

- Root config: `build.gradle`, `settings.gradle`, `gradle.properties`.
- `compileSdkVersion=32`, `minSdkVersion=26`, `targetSdkVersion=32`, Kotlin `1.5.21`/`1.6.21`,
  Android Gradle Plugin `7.0.3`. Java source/target compatibility is 11 in `app/build.gradle`.
- Shared version numbers/ext properties live in the root `build.gradle` under `ext { ... }`
  (`applicationId`, `versionCode`, `versionName`, SDK levels) — change them there, not in
  individual modules.
- Common dependency versions are centralized in `gradle/dependencies.gradle`, applied to
  `allprojects` from the root `build.gradle`.
- Build with the wrapper, not a system Gradle install: `./gradlew assembleDebug`,
  `./gradlew test`, `./gradlew :app:assembleRelease`, etc.
- CI workflows live in `.github/workflows/` (`android.yml`, `build-apk.yml`) — check these
  for the exact commands/JDK version CI uses before assuming a different toolchain.

## Module layout

The app is split into many Gradle modules (see `settings.gradle` for the authoritative list).
Key ones:

| Module | Purpose |
|---|---|
| `app` | Main Android application module — UI, activities, app-level wiring. |
| `code-editor` | The text/code editor component (built on Rosemoe's CodeEditor, see README credits). |
| `editor-api`, `actions-api`, `completion-api`, `fileeditor-api`, `language-api` | API surfaces that decouple the editor UI from language-specific features. |
| `java-completion`, `kotlin-completion`, `xml-completion` | Per-language code completion engines. |
| `java-stubs`, `android-stubs` | Stub class sources used for completion/compilation without a full JDK/Android SDK on-device. |
| `dependency-resolver` | Resolves Maven/Gradle dependencies for user projects on-device. |
| `layout-preview` (+ `appcompat-widgets`, `cardview`, `constraintlayout`, `proteus-core`, `vector-parser`) | Renders an Android XML layout preview without a full emulator. |
| `build-tools/*` | A large set of vendored/adapted Gradle internals (`builder-*` modules), `javac`, `kotlinc`, `lint`, `manifmerger`, etc., used to perform on-device builds. Treat these as largely vendored — avoid casual edits unless you're specifically working on the on-device build pipeline. |
| `eclipse-formatter`, `google-java-format` | Code formatting integrations. |
| `treeview`, `terminalview`, `event-manager`, `common`, `javapoet` | Shared UI widgets/utilities used across modules. |

When adding a new module, register it in `settings.gradle` and prefer reusing an existing
API module (`*-api`) rather than creating new cross-module coupling.

## Conventions

- Mixed Java/Kotlin codebase; new code in `app` typically uses Kotlin for Android-facing
  code, but large parts of the editor/build-tools layers are Java — match the existing
  language used in the file/module you're editing.
- `app/proguard-rules.pro` controls release-build shrinking; if you add reflection-based
  APIs or libraries, update this file.
- Per `CONTRIBUTING.md`: base feature branches off `master`-equivalent (the default branch
  here is `main`), add tests for new code where practical (Robolectric is configured for
  unit tests targeting SDK 26 — see `testOptions` in `app/build.gradle`), and keep PRs
  focused with a clear title/description.
- Release notes go in `changelogs/<version>/` — add an entry when bumping `versionName`/
  `versionCode` in the root `build.gradle`.

## Things to be careful about

- `build-tools/` mirrors a large chunk of Gradle's own internals (module names like
  `builder-core`, `builder-tooling-api`, etc.) repackaged so builds can run inside the
  Android app's JVM. This is fragile, version-sensitive code — do not "clean up" or
  refactor it without a concrete reason tied to a bug or feature.
- `local_history.patch` and the `.idea/` directory are IDE/editor artifacts checked into
  the repo; don't rely on them as source of truth and avoid regenerating large diffs in
  them.
- There is no Kotlin DSL (`build.gradle.kts`) anywhere — keep new build files in Groovy for
  consistency.
