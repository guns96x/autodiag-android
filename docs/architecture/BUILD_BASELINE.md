# Build Baseline

**Status:** NORMATIVE. Gemini must not change any value in this document. A value can only change through a new ADR.

## 1. Identity

| Item | Value |
|---|---|
| Application ID (`applicationId`) | `com.guns96x.autodiag` |
| Debug application ID suffix | `.debug` |
| Internal application ID suffix | `.internal` |
| Beta application ID suffix | none (beta and release share `com.guns96x.autodiag`) |
| Root Kotlin package | `com.guns96x.autodiag` |
| Android `namespace` per module | `com.guns96x.autodiag.<module package suffix>` (see `MODULE_DEPENDENCY_GRAPH.md` §2) |
| `versionCode` | `major * 10000 + minor * 100 + patch` |
| `versionName` | `major.minor.patch` from `gradle.properties` keys `autodiag.version.major|minor|patch` |
| First version | `0.1.0` |

## 2. Toolchain

| Tool | Pinned version | Verification in authoring environment |
|---|---|---|
| JDK toolchain | 17 (`kotlin { jvmToolchain(17) }` in every module) | n/a |
| Gradle wrapper | 8.14.3 (`distributionType=bin`, `distributionSha256Sum` must be set from `https://services.gradle.org/distributions/gradle-8.14.3-bin.zip.sha256`) | verified (services.gradle.org) |
| Android Gradle Plugin | 8.10.1 | not network-verifiable here (Google Maven blocked); released stable version |
| Kotlin (JVM, Android, Compose compiler, serialization plugins) | 2.2.0 | verified (Maven Central) |
| KSP | 2.2.0-2.0.2 | verified (Maven Central) |
| compileSdk | 36 | fixed |
| targetSdk | 36 | fixed |
| minSdk | 26 | fixed (spec §57) |
| Android build tools | AGP 8.10.1 default (do not set `buildToolsVersion`) | fixed |

Rules:

- No dynamic versions (`+`, `latest.release`, ranges). Enforced by the CI task `verifyNoDynamicVersions` (Task F1 creates it; it fails if `gradle/libs.versions.toml` contains `+`, `[`, `(` or `latest.` in any version).
- Dependency locking is enabled for all configurations: `dependencyLocking { lockAllConfigurations() }` in the root `build.gradle.kts` `allprojects {}` block. Lock files are committed. CI runs with `--locked-dependencies` equivalent (`STRICT` lock mode).
- If a pinned artifact does not resolve, Gemini stops and reports the exact artifact coordinates. Gemini must not substitute another version.

## 3. Version catalog (`gradle/libs.versions.toml`)

This is the exact content Task F1 writes.

```toml
[versions]
agp = "8.10.1"
kotlin = "2.2.0"
ksp = "2.2.0-2.0.2"
coroutines = "1.10.2"
serialization = "1.9.0"
hilt = "2.56.2"
androidx-core = "1.16.0"
androidx-activity = "1.10.1"
androidx-lifecycle = "2.9.1"
androidx-navigation = "2.9.0"
androidx-hilt-navigation-compose = "1.2.0"
compose-bom = "2025.06.00"
room = "2.7.2"
junit4 = "4.13.2"
turbine = "1.2.1"
robolectric = "4.15.1"
androidx-test-core = "1.6.1"
androidx-test-runner = "1.6.2"
androidx-test-ext-junit = "1.2.1"
detekt = "1.23.8"
spotless = "7.2.1"
ktlint = "1.6.0"
clikt = "5.0.3"
json-schema-validator = "1.5.8"

[libraries]
kotlinx-coroutines-core = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-core", version.ref = "coroutines" }
kotlinx-coroutines-android = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-android", version.ref = "coroutines" }
kotlinx-coroutines-test = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-test", version.ref = "coroutines" }
kotlinx-serialization-json = { module = "org.jetbrains.kotlinx:kotlinx-serialization-json", version.ref = "serialization" }
hilt-android = { module = "com.google.dagger:hilt-android", version.ref = "hilt" }
hilt-compiler = { module = "com.google.dagger:hilt-compiler", version.ref = "hilt" }
hilt-android-testing = { module = "com.google.dagger:hilt-android-testing", version.ref = "hilt" }
androidx-core-ktx = { module = "androidx.core:core-ktx", version.ref = "androidx-core" }
androidx-activity-compose = { module = "androidx.activity:activity-compose", version.ref = "androidx-activity" }
androidx-lifecycle-runtime-compose = { module = "androidx.lifecycle:lifecycle-runtime-compose", version.ref = "androidx-lifecycle" }
androidx-lifecycle-viewmodel-compose = { module = "androidx.lifecycle:lifecycle-viewmodel-compose", version.ref = "androidx-lifecycle" }
androidx-lifecycle-viewmodel-savedstate = { module = "androidx.lifecycle:lifecycle-viewmodel-savedstate", version.ref = "androidx-lifecycle" }
androidx-navigation-compose = { module = "androidx.navigation:navigation-compose", version.ref = "androidx-navigation" }
androidx-hilt-navigation-compose = { module = "androidx.hilt:hilt-navigation-compose", version.ref = "androidx-hilt-navigation-compose" }
compose-bom = { module = "androidx.compose:compose-bom", version.ref = "compose-bom" }
compose-ui = { module = "androidx.compose.ui:ui" }
compose-ui-tooling-preview = { module = "androidx.compose.ui:ui-tooling-preview" }
compose-ui-tooling = { module = "androidx.compose.ui:ui-tooling" }
compose-ui-test-junit4 = { module = "androidx.compose.ui:ui-test-junit4" }
compose-ui-test-manifest = { module = "androidx.compose.ui:ui-test-manifest" }
compose-material3 = { module = "androidx.compose.material3:material3" }
room-runtime = { module = "androidx.room:room-runtime", version.ref = "room" }
room-ktx = { module = "androidx.room:room-ktx", version.ref = "room" }
room-compiler = { module = "androidx.room:room-compiler", version.ref = "room" }
room-testing = { module = "androidx.room:room-testing", version.ref = "room" }
junit4 = { module = "junit:junit", version.ref = "junit4" }
kotlin-test-junit = { module = "org.jetbrains.kotlin:kotlin-test-junit", version.ref = "kotlin" }
turbine = { module = "app.cash.turbine:turbine", version.ref = "turbine" }
robolectric = { module = "org.robolectric:robolectric", version.ref = "robolectric" }
androidx-test-core = { module = "androidx.test:core", version.ref = "androidx-test-core" }
androidx-test-runner = { module = "androidx.test:runner", version.ref = "androidx-test-runner" }
androidx-test-ext-junit = { module = "androidx.test.ext:junit", version.ref = "androidx-test-ext-junit" }
clikt = { module = "com.github.ajalt.clikt:clikt", version.ref = "clikt" }
json-schema-validator = { module = "com.networknt:json-schema-validator", version.ref = "json-schema-validator" }

[plugins]
android-application = { id = "com.android.application", version.ref = "agp" }
android-library = { id = "com.android.library", version.ref = "agp" }
kotlin-jvm = { id = "org.jetbrains.kotlin.jvm", version.ref = "kotlin" }
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }
kotlin-compose = { id = "org.jetbrains.kotlin.plugin.compose", version.ref = "kotlin" }
ksp = { id = "com.google.devtools.ksp", version.ref = "ksp" }
hilt = { id = "com.google.dagger.hilt.android", version.ref = "hilt" }
room = { id = "androidx.room", version.ref = "room" }
detekt = { id = "io.gitlab.arturbosch.detekt", version.ref = "detekt" }
spotless = { id = "com.diffplug.spotless", version.ref = "spotless" }
```

## 4. Libraries that are forbidden

Gemini must not add any dependency that is not in §3. In particular:

| Forbidden | Reason / replacement |
|---|---|
| Mockito, MockK, any mocking library | Use hand-written fakes from `:testkit` |
| kotlinx-datetime, java.time in serialized models | Timestamps are `Long` epoch milliseconds (UTC) |
| Gson, Moshi, Jackson in app modules | kotlinx.serialization only (Jackson is allowed transitively inside `:tools:*` via json-schema-validator only) |
| usb-serial-for-android or any JitPack artifact | USB drivers are implemented in `:transport-usb` (see `TRANSPORT_ADAPTER_CONTRACT.md` §4) |
| Timber, SLF4J, Logback in app modules | Logging via `AutoDiagLogger` (`TRACE_LOGGING_CONTRACT.md`) |
| RxJava, LiveData | Coroutines/Flow only |
| JUnit 5 | JUnit 4 everywhere (Robolectric and Compose tests require it) |
| Any analytics/crash SDK | Spec §59 |

## 5. Module build types

| Kind | Plugins | Used by |
|---|---|---|
| JVM library | `kotlin-jvm`, `kotlin-serialization` (only where the module declares serializable types) | all `core-*`, `transport-api`, `adapter-*`, `protocol-*`, `oem-vag`, runtimes, `usecase`, `testkit` |
| Android library | `android-library`, `kotlin-android` | `transport-bluetooth-spp`, `transport-ble`, `transport-usb` |
| Android library + Room | `android-library`, `kotlin-android`, `ksp`, `room`, `kotlin-serialization` | `storage` |
| Android library + Compose | `android-library`, `kotlin-android`, `kotlin-compose` | `ui` |
| Android application | `android-application`, `kotlin-android`, `kotlin-compose`, `ksp`, `hilt` | `app` |
| JVM application | `kotlin-jvm`, `application`, `kotlin-serialization` | `tools:evidence-toolchain`, `tools:trace-toolchain` |

No convention-plugin (`build-logic`) module is used. Every module has its own explicit `build.gradle.kts`. Shared values (`compileSdk`, `minSdk`, `targetSdk`, `jvmToolchain`) are read from `gradle.properties` keys:

```properties
autodiag.compileSdk=36
autodiag.minSdk=26
autodiag.targetSdk=36
autodiag.jvmToolchain=17
autodiag.version.major=0
autodiag.version.minor=1
autodiag.version.patch=0
org.gradle.jvmargs=-Xmx4g -Dfile.encoding=UTF-8
android.useAndroidX=true
kotlin.code.style=official
org.gradle.configuration-cache=true
```

## 6. Kotlin compiler settings (every module)

```kotlin
kotlin {
    jvmToolchain(17)
    compilerOptions {
        allWarningsAsErrors.set(true)
        freeCompilerArgs.add("-Xjsr305=strict")
    }
}
```

Explicit API mode: `kotlin { explicitApi() }` is enabled in every JVM library module (public API must be declared `public`). It is not enabled in `app`, `ui`, `storage`, `testkit` or `tools:*`.

## 7. Android settings

- `compileOptions { sourceCompatibility = JavaVersion.VERSION_17; targetCompatibility = JavaVersion.VERSION_17 }`
- `buildFeatures { compose = true }` only in `ui` and `app`; `buildConfig = true` only in `app`.
- Build types in `app`: `debug`, `internal` (`initWith(getByName("release"))`, `applicationIdSuffix = ".internal"`, debuggable false), `release`. Beta uses the `release` build type with a different `versionName` suffix `-beta.N` set by CI.
- R8: `isMinifyEnabled = true` for `internal` and `release`; `proguard-rules.pro` keeps `@Serializable` classes (`-keep,includedescriptorclasses class com.guns96x.autodiag.**$$serializer { *; }` and the standard kotlinx.serialization rules).
- `BuildConfig` fields in `app`: `RUNTIME_PACK_PUBLIC_KEY` (String, base64 X.509 ECDSA P-256 key; from Gradle property `autodiag.packPublicKey`; debug builds use the test key committed at `testkit/src/main/resources/keys/test-pack-public.b64`), `ENGINEERING_BUILD` (Boolean; `true` for `debug` only).
- Robolectric tests run with `robolectric.properties` containing `sdk=35` in each Android module's `src/test/resources/`.

## 8. Static analysis and formatting

| Check | Configuration | Command |
|---|---|---|
| Formatting | Spotless with ktlint 1.6.0 for `**/*.kt` and `**/*.kts`; `.editorconfig` at root with `ktlint_code_style = ktlint_official`, `max_line_length = 140`, `ktlint_function_naming_ignore_when_annotated_with = Composable` | `./gradlew spotlessCheck` |
| Detekt | `config/detekt/detekt.yml` generated by `./gradlew detektGenerateConfig`, then `build.maxIssues: 0`; baseline files are forbidden | `./gradlew detekt` |
| Android lint | `lint { warningsAsErrors = true; abortOnError = true; checkDependencies = true }` in `app` | `./gradlew :app:lintDebug` |
| Dynamic versions | custom task `verifyNoDynamicVersions` in root build | `./gradlew verifyNoDynamicVersions` |
| Module boundaries | custom task `verifyModuleBoundaries` reading `docs/architecture/module-dependencies.json` (see `MODULE_DEPENDENCY_GRAPH.md` §5) | `./gradlew verifyModuleBoundaries` |

## 9. Canonical commands

| Purpose | Command |
|---|---|
| Full local gate | `./gradlew verifyNoDynamicVersions verifyModuleBoundaries spotlessCheck detekt test :app:lintDebug :app:assembleDebug` |
| One module's unit tests | `./gradlew :<module>:test` |
| One test class | `./gradlew :<module>:test --tests "<fully.qualified.ClassName>"` |
| Android unit tests (Robolectric) | `./gradlew :<module>:testDebugUnitTest` |
| Instrumented tests | `./gradlew :app:connectedDebugAndroidTest` (only on a device/emulator; CI job `instrumented` uses API 34 emulator image `system-images;android-34;google_apis;x86_64`) |
| Room migrations | `./gradlew :storage:testDebugUnitTest --tests "com.guns96x.autodiag.storage.migration.*"` |
| Evidence tool | `./gradlew :tools:evidence-toolchain:run --args="<subcommand> <options>"` |
| Release build | `./gradlew clean check :app:assembleRelease` |

Note: for JVM modules the unit-test task is `test`; for Android library modules it is `testDebugUnitTest`. `./gradlew test` runs both kinds.

## 10. CI (`.github/workflows/ci.yml`)

Runner `ubuntu-24.04`, `actions/setup-java@v4` with `distribution: temurin`, `java-version: 17`, `gradle/actions/setup-gradle@v4`. Jobs, in this order, each depending on the previous:

1. `static`: `./gradlew verifyNoDynamicVersions verifyModuleBoundaries spotlessCheck detekt`
2. `unit`: `./gradlew test`
3. `evidence`: `./gradlew :tools:evidence-toolchain:run --args="validate-all --pack runtime-packs/vag/current"` (skipped with success until Task R1 creates `runtime-packs/vag/current`; the skip condition is `hashFiles('runtime-packs/vag/current/manifest.json') == ''`)
4. `android`: `./gradlew :app:lintDebug :app:assembleDebug`
5. `coverage`: `./gradlew :tools:evidence-toolchain:run --args="coverage-report --pack runtime-packs/vag/current --out build/reports/coverage"`; uploads `build/reports/coverage` as artifact (same skip condition as `evidence`)
6. `instrumented`: runs only on `workflow_dispatch` and on tags `v*`.
7. `release`: runs only on tags `v*`; `./gradlew :app:assembleRelease`; requires jobs 1–5 green.
