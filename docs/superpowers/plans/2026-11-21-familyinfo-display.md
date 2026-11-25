# FamilyInfoDisplay — Implementation Plan & Execution Ledger

> **EXECUTOR PROTOCOL — binding for every session (human or AI) executing this plan.**
> 1. **Sources of truth:** this file + the spec at `docs/superpowers/specs/2026-11-21-familyinfo-display-design.md`. Re-read the spec § relevant to the current task before executing it.
> 2. **Resume protocol:** read the "Current Position" block and "Execution Log" below, then continue with the **first unchecked `- [ ]` step**. Never re-do a checked step (its commit is in `git log`; one commit per task).
> 3. **Atomicity:** execute exactly ONE step, verify its **Done** criterion, check the box, log it — then the next. Never combine steps or tasks (spec §11 hard rule).
> 4. **Update early, update often:** after EVERY completed step: check the box + append one line to "Execution Log". After a task's LAST step: move the "Current Position" pointer. This file must always be current so a fresh session (after timeout or context compression) can resume from THIS FILE ALONE.
> 5. **TDD:** steps marked **TEST-FIRST**: write the test, run it, watch it FAIL, implement, watch it PASS. "Should pass" is not evidence.
> 6. **CWD:** all commands run from `/Users/lindner/VSCode/FamilyInfoDisplay`. Cut-off output: re-run with `| tail -40`. First build downloads dependencies (10–20 min — do NOT interrupt; a re-run resumes where it stopped).
> 7. **On failure:** consult "Failure playbook". Any deviation from this plan must be recorded as a `Ruling:` line in the Execution Log. A step stays unchecked until its Done criterion is met.
> 8. **Commits:** only the given `git commit` commands. Stay on branch `master`. No branches, no pushes.

## Environment facts (ALL verified 2026-11-22 — trust, do not re-verify)

- Repo: `/Users/lindner/VSCode/FamilyInfoDisplay`, branch `master`, start HEAD = `b7a6c57` ("approved spec"). Git identity configured (`Flo <superfloh@gmx.net>`).
- Untracked noise: `.agents/`, `app/.gradle/` — never committed; `.gitignore` (T1S5) covers them, so `git add -A` is safe from T1S14 onward.
- JDK: Temurin **27.0.1.1** (`/Library/Java/JavaVirtualMachines/temurin-27.jdk/Contents/Home`) — the only usable JDK (no 17/21).
- Android SDK: `/Users/lindner/Library/Android/sdk` — contains ONLY `platforms/android-36.1`, `build-tools/36.1.0`, `platform-tools`. **No cmdline-tools → no sdkmanager.** `ANDROID_HOME` not set → `local.properties` (T1S4) is mandatory.
- No system gradle. Cached dist binary: `/Users/lindner/.gradle/wrapper/dists/gradle-9.8.1-bin/a2l6ltp3k1vth6lwipzm5p2jp/gradle-9.8.1/bin/gradle` (if the hash dir differs: `$(find ~/.gradle/wrapper/dists/gradle-9.8.1-bin -name gradle -type f | head -1)`).
- Existing stub `app/build.gradle.kts` (partial `android {}`, no plugins — invalid as-is) is fully rewritten in T1S7.
- **Device connected (verified):** serial `HA2EN2VE`, model TB311FU, Android 16 / ZUI 17.5, USB. Before EVERY tablet step: `adb devices` must list `HA2EN2VE`; if not, tell the user to reconnect and wait.
- Inter font zip already at **`/tmp/Inter-4.1.zip`** (v4.1); static TTFs inside: `extras/ttf/Inter-Regular.ttf`, `extras/ttf/Inter-Medium.ttf`, `extras/ttf/Inter-Bold.ttf`. If gone (reboot): `curl -sL https://github.com/rsms/inter/releases/download/v4.1/Inter-4.1.zip -o /tmp/Inter-4.1.zip`

## Pinned versions (verified stable 2026-11-22 via maven metadata — use EXACTLY these)

| What | Version |
|---|---|
| AGP `com.android.application` | **9.5.2** |
| Kotlin `org.jetbrains.kotlin.android` | **2.4.21** |
| Gradle wrapper | **9.8.1** (cached, no download) |
| Compose BOM `androidx.compose:compose-bom` | **2026.11.01** |
| `androidx.activity:activity-compose` | **1.13.0** |
| `androidx.lifecycle:lifecycle-runtime-ktx` / `lifecycle-runtime-compose` | **2.11.1** |
| `androidx.datastore:datastore-preferences` | **1.3.0** |
| `org.jetbrains.kotlinx:kotlinx-serialization-json` | **1.13.0** |
| `org.jetbrains.kotlinx:kotlinx-coroutines-test` | **1.11.0** |
| JUnit4 `junit:junit` | **4.13.2** |
| `androidx.test.ext:junit` | **1.3.0** |
| `androidx.test:core` | **1.7.0** |
| compileSdk / compileSdkMinor | **36 / 1** (platform `android-36.1` installed) |
| minSdk / targetSdk | **36 / 36** |

If AGP 9.5.2 + Gradle 9.8.1 reject each other at T1S13: downgrade AGP to the newest 9.x the error message says is compatible, record a Ruling, update the table, continue.

## Global constraints (from spec — binding)

- UI language **German** everywhere (in-code literals; `strings.xml` only holds `app_name`)
- Orientation **landscape only** (`SENSOR_LANDSCAPE`, T10S4); `KEEP_SCREEN_ON` while foregrounded (T10S4, verified in Task 12)
- Grafana 13 dark: near-black app bg, panels slightly lighter, 1px border, 8.dp rounded, title left-aligned, accent **#F46800**, status green/yellow/orange-red
- Font **Inter** bundled in `res/font/` (OFL 1.1), default family
- Weather: Open-Meteo (no key); refresh on launch + every 15 min; **hour 06–15 → TODAY, else TOMORROW** (06:00–15:59 / 16:00–05:59)
- No DI framework, no navigation library, no background services; single activity
- App license **Beerware** (ko-fi https://ko-fi.com/florian500); third-party list in `THIRD_PARTY_LICENSES.md` + about screen
- Target device: Lenovo Tab TB311FU, 1920×1200 (16:10), Android 16 / ZUI 17.5

## File map (every file this plan creates; T# = creating task)

```
FamilyInfoDisplay/
├─ settings.gradle.kts            T1        ├─ gradle.properties              T1
├─ build.gradle.kts (root)        T1        ├─ local.properties               T1 (git-ignored)
├─ .gitignore                     T1        ├─ LICENSE.md                     T11
├─ THIRD_PARTY_LICENSES.md        T11       ├─ gradle/libs.versions.toml      T1
├─ gradlew, gradlew.bat, gradle/wrapper/*  T1
└─ app/
   ├─ build.gradle.kts            T1 (stub rewrite)
   └─ src/
      ├─ main/
      │  ├─ AndroidManifest.xml   T1 (+READ_CALENDAR, ACCESS_FINE_LOCATION in T10)
      │  ├─ java/de/florian500/familyinfo/
      │  │  ├─ MainActivity.kt    T1 (+T2 theme, +T4 home wiring, +T8 calendar wiring, +T10S4 screen switch & orientation/keep-screen-on, +T11S2 About entry)
      │  │  ├─ data/
      │  │  │  ├─ GeoPoint.kt                 T3    ├─ SettingsRepository.kt      T3
      │  │  │  ├─ WeatherModels.kt            T5    ├─ WeatherMode.kt             T5
      │  │  │  ├─ OpenMeteoClient.kt          T5    ├─ CalendarModels.kt          T7
      │  │  │  └─ CalendarRepository.kt       T7
      │  │  └─ ui/
      │  │     ├─ theme/Color.kt, Type.kt, Theme.kt      T2
      │  │     ├─ HomeScreen.kt             T4 (+T6 weather wiring, +T8 calendar wiring, +T10 settings icon)
      │  │     ├─ SettingsScreen.kt         T10 (+T11 Info entry)
      │  │     ├─ AboutScreen.kt            T11
      │  │     └─ tiles/Tile.kt (T4, +T6/T8/T9 real tiles)
      │  │        WeatherTile.kt (T6)  CalendarTile.kt (T8)  SegelwetterTile.kt (T9)
      │  └─ res/
      │     ├─ font/inter_regular.ttf, inter_medium.ttf, inter_bold.ttf    T2
      │     ├─ mipmap-mdpi/ic_launcher.png (placeholder) T1 → proper icon T12S1
      │     └─ values/strings.xml          T1 (+T10 German strings)
      ├─ test/java/de/florian500/familyinfo/
      │  ├─ data/SettingsRepositoryTest.kt  T3     ├─ data/WeatherModeTest.kt     T5
      │  ├─ data/OpenMeteoClientTest.kt     T5     ├─ data/CalendarRangeTest.kt   T7
      │  └─ ui/TileRegistryTest.kt          T4
      └─ androidTest/java/de/florian500/familyinfo/data/
         └─ CalendarRepositoryInstrumentedTest.kt   T7
```

## Current Position

```
NEXT STEP: T1S1
Current task: Task 1 — Project scaffold (15 steps)
Remaining tasks after current: 11 (T2..T12)
Last executed: none (verified 2026-11-23: HEAD=b7a6c57, no scaffold on disk — prior context's "Task 1 done @ 7f9344e" is not in this repo)
Last updated: 2026-11-23 — plan v3: Tasks 11+12 + T10S5-S7 added; counts fixed; repo state re-verified
```

## Execution Log

(append one line per completed step: `YYYY-MM-DD  <STEPID> done — <one line, incl. commit hash or test result>`)

```
2026-11-22  plan v2 written — atomic steps, exact file contents, version pins; nothing executed yet
2026-11-23  plan v3 — Tasks 11 (About + license files) & 12 (icon + final verification) added; T10S5-S7 added; lifecycleScope import fixed in T8S6/T10S4
2026-11-23  repo verified: HEAD=b7a6c57 (approved spec) — Task 1 NOT executed in this worktree (no app/src, no gradlew, no .gitignore; commit 7f9344e from earlier context does not exist here) → resume at T1S1
```

## Failure playbook (consult after 2 failed iterations on the same step)

- `SDK location not found` → `local.properties` missing/wrong (T1S4 content)
- `Failed to find target with hash string android-36.1` → `compileSdkMinor = 1` missing in `app/build.gradle.kts`
- AGP/Gradle incompatibility at T1S13 → see pinned-versions note; downgrade AGP, Ruling line
- `./gradlew: Permission denied` → `chmod +x gradlew`
- `adb devices` empty → ask user to reconnect USB / unlock tablet; wait; re-check
- Instrumented test: no calendars → tablet Settings → Accounts → Google account → calendar sync ON; inspect: `adb -s HA2EN2VE shell content query --uri content://calendars | head`
- WebView blank → `adb -s HA2EN2VE logcat -s chromium,console`; open the Segelwetter URL in a browser on the tablet first
- Compose BOM pulls a transitive needing a newer AndroidX than pinned → let Gradle resolve; on hard failure bump only that artifact, Ruling line
- **Session resumed mid-step:** trust git log + checked boxes + Execution Log. Files of the current step may partially exist — diff them against this plan's exact contents and finish the step; do not start it blind.
- **Timeout mid-step:** do NOT blindly re-run; `git status` + check which sub-action of the step already happened, then complete it.
- **JDK 27 toolchain complaint** (only if it happens) → add `org.gradle.java.home=/Library/Java/JavaVirtualMachines/temurin-27.jdk/Contents/Home` to `gradle.properties`, Ruling line.

---

## Task 1: Project scaffold — empty app builds and installs

**Task done when:** `./gradlew :app:assembleDebug` is green, `app/build/outputs/apk/debug/app-debug.apk` exists, app installs and launches on tablet, everything committed.

- [ ] **T1S1** Create `settings.gradle.kts` (repo root), exactly:
  ```kotlin
  pluginManagement {
      repositories {
          google()
          mavenCentral()
          gradlePluginPortal()
      }
  }

  dependencyResolutionManagement {
      repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
      repositories {
          google()
          mavenCentral()
      }
  }

  rootProject.name = "FamilyInfoDisplay"
  include(":app")
  ```
  **Done:** file exists with exactly this content.

- [ ] **T1S2** Create `build.gradle.kts` (repo root), exactly:
  ```kotlin
  plugins {
      alias(libs.plugins.android.application) apply false
      alias(libs.plugins.kotlin.android) apply false
      alias(libs.plugins.kotlin.compose) apply false
      alias(libs.plugins.kotlin.serialization) apply false
  }
  ```
  **Done:** file exists with exactly this content.

- [ ] **T1S3** Create `gradle.properties` (repo root), exactly:
  ```
  android.useAndroidX=true
  org.gradle.jvmargs=-Xmx2g
  kotlin.code.style=official
  ```
  **Done:** file exists with exactly this content.

- [ ] **T1S4** Create `local.properties` (repo root), exactly:
  ```
  sdk.dir=/Users/lindner/Library/Android/sdk
  ```
  **Done:** file exists with exactly this content. (Never committed — .gitignore covers it.)

- [ ] **T1S5** Create `.gitignore` (repo root), exactly:
  ```
  .gradle/
  local.properties
  build/
  app/build/
  .agents/
  .idea/
  *.iml
  .DS_Store
  ```
  **Done:** file exists; `git add -A` is safe from now on.
- [ ] **T1S6** Create `gradle/libs.versions.toml` (version catalog), exactly:
  ```toml
  [versions]
  agp = "9.5.2"
  kotlin = "2.4.21"
  composeBom = "2026.11.01"
  activityCompose = "1.13.0"
  lifecycle = "2.11.1"
  datastore = "1.3.0"
  serialization = "1.13.0"
  coroutinesTest = "1.11.0"
  junit = "4.13.2"
  androidxTestJunit = "1.3.0"
  androidxTestCore = "1.7.0"

  [libraries]
  androidx-compose-bom = { module = "androidx.compose:compose-bom", version.ref = "composeBom" }
  androidx-compose-ui = { module = "androidx.compose.ui:ui" }
  androidx-compose-material3 = { module = "androidx.compose.material3:material3" }
  androidx-compose-ui-tooling = { module = "androidx.compose.ui:ui-tooling" }
  androidx-compose-ui-tooling-preview = { module = "androidx.compose.ui:ui-tooling-preview" }
  androidx-compose-ui-test-junit4 = { module = "androidx.compose.ui:ui-test-junit4" }
  androidx-compose-ui-test-manifest = { module = "androidx.compose.ui:ui-test-manifest" }
  androidx-activity-compose = { module = "androidx.activity:activity-compose", version.ref = "activityCompose" }
  androidx-lifecycle-runtime-ktx = { module = "androidx.lifecycle:lifecycle-runtime-ktx", version.ref = "lifecycle" }
  androidx-lifecycle-runtime-compose = { module = "androidx.lifecycle:lifecycle-runtime-compose", version.ref = "lifecycle" }
  androidx-datastore-preferences = { module = "androidx.datastore:datastore-preferences", version.ref = "datastore" }
  kotlinx-serialization-json = { module = "org.jetbrains.kotlinx:kotlinx-serialization-json", version.ref = "serialization" }
  junit = { module = "junit:junit", version.ref = "junit" }
  kotlinx-coroutines-test = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-test", version.ref = "coroutinesTest" }
  androidx-test-ext-junit = { module = "androidx.test.ext:junit", version.ref = "androidxTestJunit" }
  androidx-test-core = { module = "androidx.test:core", version.ref = "androidxTestCore" }

  [plugins]
  android-application = { id = "com.android.application", version.ref = "agp" }
  kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
  kotlin-compose = { id = "org.jetbrains.kotlin.plugin.compose", version.ref = "kotlin" }
  kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }
  ```
  **Done:** file exists at `gradle/libs.versions.toml`.
- [ ] **T1S7** Rewrite `app/build.gradle.kts` (replace the stub entirely), exactly:
  ```kotlin
  plugins {
      alias(libs.plugins.android.application)
      alias(libs.plugins.kotlin.android)
      alias(libs.plugins.kotlin.compose)
      alias(libs.plugins.kotlin.serialization)
  }

  android {
      namespace = "de.florian500.familyinfo"
      compileSdk = 36
      compileSdkMinor = 1
      defaultConfig {
          minSdk = 36
          targetSdk = 36
          versionCode = 1
          versionName = "0.1.0"
      }
      buildTypes {
          release {
              isMinifyEnabled = false
          }
      }
      compileOptions {
          sourceCompatibility = JavaVersion.VERSION_17
          targetCompatibility = JavaVersion.VERSION_17
      }
      kotlinOptions {
          jvmTarget = "17"
      }
      buildFeatures {
          compose = true
      }
  }

  dependencies {
      implementation(platform(libs.androidx.compose.bom))
      implementation(libs.androidx.compose.ui)
      implementation(libs.androidx.compose.material3)
      implementation(libs.androidx.compose.ui.tooling.preview)
      implementation(libs.androidx.activity.compose)
      implementation(libs.androidx.lifecycle.runtime.ktx)
      implementation(libs.androidx.lifecycle.runtime.compose)
      implementation(libs.androidx.datastore.preferences)
      implementation(libs.kotlinx.serialization.json)

      debugImplementation(libs.androidx.compose.ui.tooling)

      testImplementation(libs.junit)
      testImplementation(libs.kotlinx.coroutines.test)

      androidTestImplementation(platform(libs.androidx.compose.bom))
      androidTestImplementation(libs.androidx.test.ext.junit)
      androidTestImplementation(libs.androidx.test.core)
      androidTestImplementation(libs.androidx.compose.ui.test.junit4)
      debugImplementation(libs.androidx.compose.ui.test.manifest)
  }
  ```
  **Done:** file exists with exactly this content. (If AGP 9.5.2/Kotlin 2.4.21 rejects `kotlinOptions`, use `kotlin { compilerOptions { jvmTarget = org.jetbrains.kotlin.gradle.dsl.JvmTarget.JVM_17 } }` in the module build file instead, Ruling line — only on error.)
- [ ] **T1S8** Create `app/src/main/res/values/strings.xml`, exactly:
  ```xml
  <?xml version="1.0" encoding="utf-8"?>
  <resources>
      <string name="app_name">FamilyInfo</string>
  </resources>
  ```
  **Done:** file exists with this content.

- [ ] **T1S9** Create the placeholder launcher icon (proper icon comes in Task 12, T12S1):
  ```bash
  cd /Users/lindner/VSCode/FamilyInfoDisplay
  mkdir -p app/src/main/res/mipmap-mdpi
  python3 -c "
  import zlib,struct
  def png(w,h,rgb):
      raw=b''.join(b'\x00'+bytes(rgb)*w for _ in range(h))
      def chunk(t,d):
          c=t+d; return struct.pack('>I',len(d))+c+struct.pack('>I',zlib.crc32(c)&0xffffffff)
      return b'\x89PNG\r\n\x1a\n'+chunk(b'IHDR',struct.pack('>IIBBBBB',w,h,8,2,0,0,0))+chunk(b'IDAT',zlib.compress(raw))+chunk(b'IEND',b'')
  open('app/src/main/res/mipmap-mdpi/ic_launcher.png','wb').write(png(48,48,(24,27,31)))
  "
  file app/src/main/res/mipmap-mdpi/ic_launcher.png
  ```
  **Done:** `file` prints "PNG image data, 48 x 48".

- [ ] **T1S10** Create `app/src/main/AndroidManifest.xml`, exactly:
  ```xml
  <?xml version="1.0" encoding="utf-8"?>
  <manifest xmlns:android="http://schemas.android.com/apk/res/android">

      <uses-permission android:name="android.permission.INTERNET" />
      <uses-permission android:name="android.permission.READ_CALENDAR" />
      <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />

      <application
          android:allowBackup="false"
          android:icon="@mipmap/ic_launcher"
          android:label="@string/app_name"
          android:supportsRtl="true"
          android:theme="@android:style/Theme.Material.NoActionBar">
          <activity
              android:name=".MainActivity"
              android:exported="true">
              <intent-filter>
                  <action android:name="android.intent.action.MAIN" />
                  <category android:name="android.intent.category.LAUNCHER" />
              </intent-filter>
          </activity>
      </application>

  </manifest>
  ```
  (Theme = framework `Theme.Material.NoActionBar` so the manifest needs no dependency; Task 12 sets landscape + keep-screen-on on this same file.)
  **Done:** file exists with this content.
- [ ] **T1S11** Create `app/src/main/java/de/florian500/familyinfo/MainActivity.kt`, exactly:
  ```kotlin
  package de.florian500.familyinfo

  import android.os.Bundle
  import androidx.activity.ComponentActivity
  import androidx.activity.compose.setContent
  import androidx.compose.material3.MaterialTheme
  import androidx.compose.material3.Text

  class MainActivity : ComponentActivity() {
      override fun onCreate(savedInstanceState: Bundle?) {
          super.onCreate(savedInstanceState)
          setContent {
              MaterialTheme {
                  Text("FamilyInfo")
              }
          }
      }
  }
  ```
  (Plain `MaterialTheme` for now — replaced by `FamilyInfoTheme` in T2S5.)
  **Done:** file exists with this content.

- [ ] **T1S12** Generate the Gradle wrapper from the cached dist (no download):
  ```bash
  cd /Users/lindner/VSCode/FamilyInfoDisplay
  /Users/lindner/.gradle/wrapper/dists/gradle-9.8.1-bin/a2l6ltp3k1vth6lwipzm5p2jp/gradle-9.8.1/bin/gradle wrapper --gradle-version 9.8.1
  ```
  (Fallback if the hash dir differs: `$(find ~/.gradle/wrapper/dists/gradle-9.8.1-bin -name gradle -type f | head -1) wrapper --gradle-version 9.8.1`)
  **Done:** `gradlew`, `gradlew.bat`, `gradle/wrapper/gradle-wrapper.jar`, `gradle/wrapper/gradle-wrapper.properties` all exist; `gradle-wrapper.properties` contains `gradle-9.8.1-bin.zip`.

- [ ] **T1S13** First full build — downloads ALL dependencies (10–20 min, do NOT interrupt; on timeout simply re-run the same command, Gradle resumes):
  ```bash
  cd /Users/lindner/VSCode/FamilyInfoDisplay && ./gradlew :app:assembleDebug 2>&1 | tail -40
  ```
  **Done:** output ends `BUILD SUCCESSFUL` and `app/build/outputs/apk/debug/app-debug.apk` exists. On failure: Failure playbook + diff the T1S1–T1S11 files against this plan's exact contents and fix. Step stays unchecked until green.

- [ ] **T1S14** Install + launch on the tablet:
  ```bash
  adb devices
  adb -s HA2EN2VE install -r /Users/lindner/VSCode/FamilyInfoDisplay/app/build/outputs/apk/debug/app-debug.apk
  adb -s HA2EN2VE shell am start -n de.florian500.familyinfo/.MainActivity
  ```
  **Done:** `adb devices` lists `HA2EN2VE`; install prints `Success`; start prints `Starting:` with no error; tablet screen shows the "FamilyInfo" placeholder (user confirms, or `adb -s HA2EN2VE shell dumpsys window | grep mCurrentFocus` shows `de.florian500.familyinfo`).

- [ ] **T1S15** Commit:
  ```bash
  cd /Users/lindner/VSCode/FamilyInfoDisplay && git add -A && git commit -m "chore: scaffold Gradle + Compose project"
  ```
  **Done:** `git status --porcelain` is empty; `git log --oneline -1` shows this commit.
  **→ then move Current Position to T2S1, Remaining = 11.**

## Task 2: Grafana dark theme + Inter font

**Task done when:** build green; tablet shows the placeholder as a dark Grafana panel in Inter font; committed.

- [ ] **T2S1** Extract the Inter TTFs (v4.1, zip already at `/tmp/Inter-4.1.zip` — re-download per env facts if missing):
  ```bash
  cd /Users/lindner/VSCode/FamilyInfoDisplay
  mkdir -p app/src/main/res/font
  rm -rf /tmp/inter-extract && mkdir /tmp/inter-extract && cd /tmp/inter-extract && unzip -q /tmp/Inter-4.1.zip
  cp extras/ttf/Inter-Regular.ttf /Users/lindner/VSCode/FamilyInfoDisplay/app/src/main/res/font/inter_regular.ttf
  cp extras/ttf/Inter-Medium.ttf  /Users/lindner/VSCode/FamilyInfoDisplay/app/src/main/res/font/inter_medium.ttf
  cp extras/ttf/Inter-Bold.ttf    /Users/lindner/VSCode/FamilyInfoDisplay/app/src/main/res/font/inter_bold.ttf
  ls -la /Users/lindner/VSCode/FamilyInfoDisplay/app/src/main/res/font/
  ```
  **Done:** exactly 3 files in `app/src/main/res/font/`: `inter_regular.ttf` (~411 KB), `inter_medium.ttf` (~417 KB), `inter_bold.ttf` (~420 KB).

- [ ] **T2S2** Create `app/src/main/java/de/florian500/familyinfo/ui/theme/Color.kt`, exactly:
  ```kotlin
  package de.florian500.familyinfo.ui.theme

  import androidx.compose.ui.graphics.Color

  // Grafana 13 dark palette (spec §7.6)
  val AppBackground = Color(0xFF111217)
  val PanelBackground = Color(0xFF181B1F)
  val PanelBorder = Color(0xFF2F3236)
  val TextPrimary = Color(0xFFD8D9DA)
  val TextSecondary = Color(0xFF9FA1A6)
  val Accent = Color(0xFFF46800)
  val Ok = Color(0xFF73BF69)
  val Warn = Color(0xFFFF8C1E)
  val Alert = Color(0xFFF2495C)
  ```
  **Done:** file exists with this content.

- [ ] **T2S3** Create `app/src/main/java/de/florian500/familyinfo/ui/theme/Type.kt`, exactly:
  ```kotlin
  package de.florian500.familyinfo.ui.theme

  import androidx.compose.material3.Typography
  import androidx.compose.ui.text.font.Font
  import androidx.compose.ui.text.font.FontFamily
  import androidx.compose.ui.text.font.FontWeight
  import de.florian500.familyinfo.R

  val Inter = FontFamily(
      Font(R.font.inter_regular, FontWeight.Normal),
      Font(R.font.inter_medium, FontWeight.Medium),
      Font(R.font.inter_bold, FontWeight.Bold),
  )

  val InterTypography = Typography(
      defaultFontFamily = Inter,
  )
  ```
  **Done:** file exists with this content.

- [ ] **T2S4** Create `app/src/main/java/de/florian500/familyinfo/ui/theme/Theme.kt`, exactly:
  ```kotlin
  package de.florian500.familyinfo.ui.theme

  import androidx.compose.material3.MaterialTheme
  import androidx.compose.material3.darkColorScheme
  import androidx.compose.runtime.Composable

  private val GrafanaDark = darkColorScheme(
      primary = Accent,
      background = AppBackground,
      surface = PanelBackground,
      onBackground = TextPrimary,
      onSurface = TextPrimary,
      secondary = TextSecondary,
  )

  @Composable
  fun FamilyInfoTheme(content: @Composable () -> Unit) {
      MaterialTheme(
          colorScheme = GrafanaDark,
          typography = InterTypography,
          content = content,
      )
  }
  ```
  **Done:** file exists with this content.
- [ ] **T2S5** Replace `MainActivity.kt` content with (adds theme + a panel to verify the look):
  ```kotlin
  package de.florian500.familyinfo

  import android.os.Bundle
  import androidx.activity.ComponentActivity
  import androidx.activity.compose.setContent
  import androidx.compose.foundation.border
  import androidx.compose.foundation.background
  import androidx.compose.foundation.layout.Box
  import androidx.compose.foundation.layout.fillMaxSize
  import androidx.compose.foundation.layout.padding
  import androidx.compose.foundation.shape.RoundedCornerShape
  import androidx.compose.material3.Text
  import androidx.compose.ui.Modifier
  import androidx.compose.ui.unit.dp
  import de.florian500.familyinfo.ui.theme.AppBackground
  import de.florian500.familyinfo.ui.theme.FamilyInfoTheme
  import de.florian500.familyinfo.ui.theme.PanelBackground
  import de.florian500.familyinfo.ui.theme.PanelBorder
  import de.florian500.familyinfo.ui.theme.TextPrimary

  class MainActivity : ComponentActivity() {
      override fun onCreate(savedInstanceState: Bundle?) {
          super.onCreate(savedInstanceState)
          setContent {
              FamilyInfoTheme {
                  Box(
                      modifier = Modifier
                          .fillMaxSize()
                          .background(AppBackground),
                  ) {
                      Box(
                          modifier = Modifier
                              .fillMaxSize()
                              .padding(16.dp)
                              .background(PanelBackground, shape = RoundedCornerShape(8.dp))
                              .border(width = 1.dp, color = PanelBorder, shape = RoundedCornerShape(8.dp))
                              .padding(24.dp),
                      ) {
                          Text(text = "FamilyInfo", color = TextPrimary)
                      }
                  }
              }
          }
      }
  }
  ```
  **Done:** file updated; requirement = dark app bg + panel (PanelBackground, 8.dp corners, visible 1px PanelBorder) + "FamilyInfo" in Inter.
- [ ] **T2S6** Build:
  ```bash
  cd /Users/lindner/VSCode/FamilyInfoDisplay && ./gradlew :app:assembleDebug 2>&1 | tail -40
  ```
  **Done:** `BUILD SUCCESSFUL`. On compile error, diff T2S2–T2S5 files against this plan.

- [ ] **T2S7** Install + verify look on tablet:
  ```bash
  adb -s HA2EN2VE install -r /Users/lindner/VSCode/FamilyInfoDisplay/app/build/outputs/apk/debug/app-debug.apk
  adb -s HA2EN2VE shell am start -n de.florian500.familyinfo/.MainActivity
  ```
  **Done:** tablet shows dark app background, a slightly-lighter panel filling it with a 1px visible border + 8.dp corners, and "FamilyInfo" rendered in the Inter font (distinct from Android's default Roboto — note the 'a' and 't').

- [ ] **T2S8** Commit:
  ```bash
  cd /Users/lindner/VSCode/FamilyInfoDisplay && git add -A && git commit -m "feat: Grafana-dark theme with Inter font"
  ```
  **Done:** clean tree. **→ then move Current Position to T3S1, Remaining = 10.**

## Task 3: SettingsRepository (DataStore)

**Task done when:** `SettingsRepositoryTest` green (3 cases) and committed.

**Interface (binding — later tasks depend on these exact names):**
```kotlin
data class GeoPoint(val lat: Double, val lon: Double)

class SettingsRepository(private val dataStore: DataStore<Preferences>) {
    val hiddenTileIds: Flow<Set<String>>
    val location: Flow<GeoPoint?>          // null until set
    suspend fun setTileVisible(id: String, visible: Boolean)
    suspend fun setLocation(point: GeoPoint)
    companion object {
        fun create(context: Context): SettingsRepository
    }
}
```

- [ ] **T3S1** Create `app/src/main/java/de/florian500/familyinfo/data/GeoPoint.kt`, exactly:
  ```kotlin
  package de.florian500.familyinfo.data

  data class GeoPoint(val lat: Double, val lon: Double)
  ```
  **Done:** file exists.

- [ ] **T3S2** (TEST-FIRST) Create `app/src/test/java/de/florian500/familyinfo/data/SettingsRepositoryTest.kt`, exactly:
  ```kotlin
  package de.florian500.familyinfo.data

  import androidx.datastore.core.DataStoreFactory
  import androidx.datastore.preferences.core.Preferences
  import kotlinx.coroutines.CoroutineScope
  import kotlinx.coroutines.Dispatchers
  import kotlinx.coroutines.Job
  import kotlinx.coroutines.ExperimentalCoroutinesApi
  import kotlinx.coroutines.flow.first
  import kotlinx.coroutines.test.runTest
  import kotlin.test.Test
  import kotlin.test.assertEquals
  import kotlin.test.assertNull
  import kotlin.test.assertTrue

  @OptIn(ExperimentalCoroutinesApi::class)
  class SettingsRepositoryTest {

      private fun newRepo(): SettingsRepository = SettingsRepository(
          DataStoreFactory.create<Preferences>(
              coroutineScope = CoroutineScope(Dispatchers.Default + Job()),
          ) { Preferences() }
      )

      @Test
      fun initially_hidden_empty_and_location_null() = runTest {
          val repo = newRepo()
          assertTrue(repo.hiddenTileIds.first().isEmpty())
          assertNull(repo.location.first())
      }

      @Test
      fun tile_visibility_round_trip() = runTest {
          val repo = newRepo()
          repo.setTileVisible("weather", false)
          assertTrue("weather" in repo.hiddenTileIds.first())
          repo.setTileVisible("weather", true)
          assertTrue(repo.hiddenTileIds.first().isEmpty())
      }

      @Test
      fun location_round_trip() = runTest {
          val repo = newRepo()
          assertNull(repo.location.first())
          repo.setLocation(GeoPoint(52.44, 13.67))
          assertEquals(GeoPoint(52.44, 13.67), repo.location.first())
      }
  }
  ```
  **Done:** file exists with exactly this content.

- [ ] **T3S3** (TEST-FIRST — this MUST fail: `SettingsRepository` does not exist yet) Run:
  ```bash
  cd /Users/lindner/VSCode/FamilyInfoDisplay && ./gradlew :app:testDebugUnitTest --tests '*SettingsRepositoryTest*' 2>&1 | tail -30
  ```
  **Done:** compile FAILURE mentioning `SettingsRepository` (that is the green outcome of this step — a passing run means the test file is not what T3S2 says; fix it first).
- [ ] **T3S4** Create `app/src/main/java/de/florian500/familyinfo/data/SettingsRepository.kt`, exactly:
  ```kotlin
  package de.florian500.familyinfo.data

  import android.content.Context
  import androidx.datastore.core.DataStore
  import androidx.datastore.preferences.core.Preferences
  import androidx.datastore.preferences.core.edit
  import androidx.datastore.preferences.core.stringKey
  import androidx.datastore.preferences.core.stringSetKey
  import androidx.datastore.preferences.preferencesDataStore
  import kotlinx.coroutines.flow.Flow
  import kotlinx.coroutines.flow.map

  private val Context.settingsDataStore: DataStore<Preferences> by preferencesDataStore(name = "settings")

  class SettingsRepository(private val dataStore: DataStore<Preferences>) {

      private object Keys {
          val HIDDEN_TILES = stringSetKey("hidden_tiles")
          val LOCATION = stringKey("location")
      }

      val hiddenTileIds: Flow<Set<String>> =
          dataStore.data.map { prefs -> prefs[Keys.HIDDEN_TILES] ?: emptySet() }

      val location: Flow<GeoPoint?> =
          dataStore.data.map { prefs ->
              prefs[Keys.LOCATION]?.let { raw ->
                  val parts = raw.split(",")
                  if (parts.size == 2) {
                      parts[0].toDoubleOrNull()?.let { lat -> parts[1].toDoubleOrNull()?.let { lon -> GeoPoint(lat, lon) } }
                  } else null
              }
          }

      suspend fun setTileVisible(id: String, visible: Boolean) {
          dataStore.edit { prefs ->
              val current = prefs[Keys.HIDDEN_TILES] ?: emptySet()
              prefs[Keys.HIDDEN_TILES] = if (visible) current - id else current + id
          }
      }

      suspend fun setLocation(point: GeoPoint) {
          dataStore.edit { prefs ->
              prefs[Keys.LOCATION] = "${point.lat},${point.lon}"
          }
      }

      companion object {
          fun create(context: Context): SettingsRepository =
              SettingsRepository(context.applicationContext.settingsDataStore)
      }
  }
  ```
  **Done:** file exists with this content.

- [ ] **T3S5** (TEST-FIRST — this MUST now pass) Re-run:
  ```bash
  cd /Users/lindner/VSCode/FamilyInfoDisplay && ./gradlew :app:testDebugUnitTest --tests '*SettingsRepositoryTest*' 2>&1 | tail -30
  ```
  **Done:** BUILD SUCCESSFUL, 3 tests passed, 0 failed.

- [ ] **T3S6** Commit:
  ```bash
  cd /Users/lindner/VSCode/FamilyInfoDisplay && git add -A && git commit -m "feat: settings repository (tile visibility, weather location)"
  ```
  **Done:** `git status --porcelain` empty. **→ Current Position → T4S1, Remaining = 9.**

## Task 4: Tile framework + home screen layout (placeholders)

**Task done when:** `TileRegistryTest` green; tablet shows 3 placeholder panels in the fixed two-column layout; committed.

**Framework decisions (binding):**
- `FamilyTile` interface: `id`, `title` (German), `defaultVisible`, `@Composable fun render()` (no params — stateful tiles are classes wired in later tasks; placeholders are simple objects now)
- `TileRegistry.all` order: `weather`, `calendar`, `segelwetter`
- `visibleTiles(hidden)` filters the registry, order preserved; a tile is visible iff `id !in hidden`
- All three initial tiles: `defaultVisible = true` (hidden set starts empty; seeding of future opt-in tiles is out of scope)
- Panel chrome is one shared composable `Panel(title, content)` = PanelBackground + 1.dp PanelBorder + 8.dp rounded + padding + title row; every tile renders inside it (spec §7.6)

- [ ] **T4S1** (TEST-FIRST) Create `app/src/test/java/de/florian500/familyinfo/ui/TileRegistryTest.kt`, exactly:
  ```kotlin
  package de.florian500.familyinfo.ui

  import de.florian500.familyinfo.ui.tiles.TileRegistry
  import de.florian500.familyinfo.ui.tiles.visibleTiles
  import kotlin.test.Test
  import kotlin.test.assertEquals
  import kotlin.test.assertTrue

  class TileRegistryTest {

      @Test
      fun registry_contains_all_three_ids_in_order() {
          assertEquals(listOf("weather", "calendar", "segelwetter"), TileRegistry.all.map { it.id })
          assertTrue(TileRegistry.all.all { it.defaultVisible })
      }

      @Test
      fun visibleTiles_filters_hidden_preserving_order() {
          assertEquals(listOf("weather", "segelwetter"), visibleTiles(setOf("calendar")).map { it.id })
      }

      @Test
      fun visibleTiles_no_hidden_returns_all() {
          assertEquals(listOf("weather", "calendar", "segelwetter"), visibleTiles(emptySet()).map { it.id })
      }
  }
  ```
  **Done:** file exists with this content.

- [ ] **T4S2** (TEST-FIRST — MUST fail: `tiles` package does not exist) Run:
  ```bash
  cd /Users/lindner/VSCode/FamilyInfoDisplay && ./gradlew :app:testDebugUnitTest --tests '*TileRegistryTest*' 2>&1 | tail -30
  ```
  **Done:** compile FAILURE (unresolved reference).

- [ ] **T4S3** Create `app/src/main/java/de/florian500/familyinfo/ui/tiles/Tile.kt`, exactly:
  ```kotlin
  package de.florian500.familyinfo.ui.tiles

  import androidx.compose.foundation.background
  import androidx.compose.foundation.border
  import androidx.compose.foundation.layout.Column
  import androidx.compose.foundation.layout.fillMaxSize
  import androidx.compose.foundation.layout.padding
  import androidx.compose.foundation.shape.RoundedCornerShape
  import androidx.compose.material3.MaterialTheme
  import androidx.compose.material3.Text
  import androidx.compose.runtime.Composable
  import androidx.compose.ui.Modifier
  import androidx.compose.ui.unit.dp
  import de.florian500.familyinfo.ui.theme.PanelBackground
  import de.florian500.familyinfo.ui.theme.PanelBorder
  import de.florian500.familyinfo.ui.theme.TextPrimary
  import de.florian500.familyinfo.ui.theme.TextSecondary

  interface FamilyTile {
      val id: String
      val title: String
      val defaultVisible: Boolean
      @Composable
      fun render()
  }

  object TileRegistry {
      val all: List<FamilyTile> = listOf(
          PlaceholderTile("weather", "Wetter"),
          PlaceholderTile("calendar", "Termine"),
          PlaceholderTile("segelwetter", "Segelwetter"),
      )
  }

  fun visibleTiles(hidden: Set<String>): List<FamilyTile> = TileRegistry.all.filter { it.id !in hidden }

  private data class PlaceholderTile(
      override val id: String,
      override val title: String,
  ) : FamilyTile {
      override val defaultVisible: Boolean = true

      @Composable
      override fun render() {
          Panel(modifier = Modifier.fillMaxSize(), title = title) {
              Text("Platzhalter", color = TextSecondary, modifier = Modifier.padding(16.dp))
          }
      }
  }

  /** Grafana panel chrome (spec §7.6): dark panel, 1px border, 8dp corners, left-aligned title. */
  @Composable
  fun Panel(modifier: Modifier = Modifier, title: String, content: @Composable () -> Unit) {
      Column(
          modifier = modifier
              .background(PanelBackground, shape = RoundedCornerShape(8.dp))
              .border(1.dp, PanelBorder, RoundedCornerShape(8.dp))
              .padding(12.dp),
      ) {
          Text(text = title, color = TextPrimary, style = MaterialTheme.typography.titleMedium)
          content()
      }
  }
  ```
  **Done:** file exists with this content.
- [ ] **T4S4** Create `app/src/main/java/de/florian500/familyinfo/ui/HomeScreen.kt`, exactly:
  ```kotlin
  package de.florian500.familyinfo.ui

  import androidx.compose.foundation.background
  import androidx.compose.foundation.layout.Arrangement
  import androidx.compose.foundation.layout.Box
  import androidx.compose.foundation.layout.Column
  import androidx.compose.foundation.layout.Row
  import androidx.compose.foundation.layout.fillMaxHeight
  import androidx.compose.foundation.layout.fillMaxSize
  import androidx.compose.foundation.layout.fillMaxWidth
  import androidx.compose.foundation.layout.padding
  import androidx.compose.runtime.Composable
  import androidx.compose.ui.Modifier
  import androidx.compose.ui.unit.dp
  import de.florian500.familyinfo.ui.tiles.visibleTiles
  import de.florian500.familyinfo.ui.theme.AppBackground

  /**
   * Fixed two-column layout (spec §7.1):
   * col 1: weather on top (34% weight) + calendar below (fills rest, content scrollable later)
   * col 2: segelwetter full column height
   * Hidden tiles collapse and the remaining tile in their column expands (weights renormalize).
   * Future tiles (ids beyond the initial three) are appended in a row below col 1 —
   * their final layout is chosen when they arrive (spec §7.1).
   */
  @Composable
  fun HomeScreen(hidden: Set<String>) {
      val visible = visibleTiles(hidden)
      val weather = visible.firstOrNull { it.id == "weather" }
      val calendar = visible.firstOrNull { it.id == "calendar" }
      val segelwetter = visible.firstOrNull { it.id == "segelwetter" }
      val extra = visible.filter { it.id !in setOf("weather", "calendar", "segelwetter") }

      Row(
          modifier = Modifier
              .fillMaxSize()
              .background(AppBackground)
              .padding(12.dp),
          horizontalArrangement = Arrangement.spacedBy(12.dp),
      ) {
          Column(
              modifier = Modifier
                  .weight(1f)
                  .fillMaxHeight(),
              verticalArrangement = Arrangement.spacedBy(12.dp),
          ) {
              if (weather != null && calendar != null) {
                  Box(modifier = Modifier.weight(0.34f).fillMaxWidth()) { weather.render() }
                  Box(modifier = Modifier.weight(0.66f).fillMaxWidth()) { calendar.render() }
              } else {
                  weather?.let { Box(modifier = Modifier.weight(1f).fillMaxWidth()) { it.render() } }
                  calendar?.let { Box(modifier = Modifier.weight(1f).fillMaxWidth()) { it.render() } }
              }
              if (extra.isNotEmpty()) {
                  Row(
                      modifier = Modifier.weight(0.4f).fillMaxWidth(),
                      horizontalArrangement = Arrangement.spacedBy(12.dp),
                  ) {
                      extra.forEach { t -> Box(modifier = Modifier.weight(1f)) { t.render() } }
                  }
              }
          }
          segelwetter?.let { tile ->
              Box(modifier = Modifier.weight(1f).fillMaxHeight()) { tile.render() }
          }
      }
  }
  ```
  **Done:** file exists with exactly this content. (Column-1 edge case: if BOTH weather and calendar are hidden and no extra tiles exist, the Column is empty — acceptable, segelwetter still fills col 2; no crash.)

  **Stateful-tile wiring note (binding for Tasks 6 & 8):** `TileRegistry` (static object) can only hold tiles that need no `Context`/callbacks. The weather tile is an exception and stays in the registry via `TileRegistry.weatherController` (T6S3). The CALENDAR tile needs `Context` (CalendarRepository) and a permission-request callback, so it is NOT registered in `TileRegistry`; instead: (a) in Task 8 the calendar entry is removed from `TileRegistry.all`, (b) `HomeScreen` gains a parameter `calendarTile: FamilyTile?` and the `calendar` slot renders `calendarTile` instead of the tile from the registry, (c) `MainActivity` constructs `CalendarTile(context, repo, onRequestPermission)` and passes it. The SETTINGS screen (Task 10) must list tiles as registry entries + the injected calendar tile (see T10S2).
- [ ] **T4S5** (TEST-FIRST — this MUST now pass) Run:
  ```bash
  cd /Users/lindner/VSCode/FamilyInfoDisplay && ./gradlew :app:testDebugUnitTest --tests '*TileRegistryTest*' 2>&1 | tail -30
  ```
  **Done:** BUILD SUCCESSFUL, 3 tests passed.

- [ ] **T4S6** Replace `MainActivity.kt` to wire HomeScreen + SettingsRepository:
  ```kotlin
  package de.florian500.familyinfo

  import android.os.Bundle
  import androidx.activity.ComponentActivity
  import androidx.activity.compose.setContent
  import androidx.compose.runtime.getValue
  import androidx.compose.ui.Modifier
  import androidx.compose.foundation.layout.fillMaxSize
  import androidx.lifecycle.compose.collectAsStateWithLifecycle
  import androidx.lifecycle.lifecycleScope
  import de.florian500.familyinfo.data.SettingsRepository
  import de.florian500.familyinfo.ui.HomeScreen
  import de.florian500.familyinfo.ui.theme.FamilyInfoTheme
  import kotlinx.coroutines.flow.first

  class MainActivity : ComponentActivity() {
      override fun onCreate(savedInstanceState: Bundle?) {
          super.onCreate(savedInstanceState)
          val settingsRepo = SettingsRepository.create(this)

          setContent {
              FamilyInfoTheme {
                  val hidden by settingsRepo.hiddenTileIds.collectAsStateWithLifecycle()
                  HomeScreen(hidden = hidden)
              }
          }
      }
  }
  ```
  **Done:** file exists with this content; it compiles (verified in T4S7).

- [ ] **T4S7** Build:
  ```bash
  cd /Users/lindner/VSCode/FamilyInfoDisplay && ./gradlew :app:assembleDebug 2>&1 | tail -40
  ```
  **Done:** `BUILD SUCCESSFUL`. (If `collectAsStateWithLifecycle` is unresolved, the `androidx.lifecycle.runtime.compose` dependency from T1S7 is missing — re-check the catalog.)

- [ ] **T4S8** Install + verify the two-column layout on tablet:
  ```bash
  adb -s HA2EN2VE install -r /Users/lindner/VSCode/FamilyInfoDisplay/app/build/outputs/apk/debug/app-debug.apk
  adb -s HA2EN2VE shell am start -n de.florian500.familyinfo/.MainActivity
  ```
  **Done:** tablet shows col 1 = "Wetter" panel (top, ~34% height) above "Termine" panel (fills rest); col 2 = "Segelwetter" panel full height; all three in dark panel style; no scrolling, no crash.

- [ ] **T4S9** Commit:
  ```bash
  cd /Users/lindner/VSCode/FamilyInfoDisplay && git add -A && git commit -m "feat: tile framework + fixed two-column home layout"
  ```
  **Done:** clean tree. **→ Current Position → T5S1, Remaining = 8.**

## Task 5: Weather data layer (Open-Meteo)

**Task done when:** `WeatherModeTest` + `OpenMeteoClientTest` green; committed.

**Interfaces (binding):**
```kotlin
data class DayForecast(val tMin: Double, val tMax: Double, val rainChance: Int)  // Int 0..100
data class WeatherSnapshot(val fetchedAtMillis: Long, val currentTemp: Double, val today: DayForecast, val tomorrow: DayForecast)

enum class WeatherMode { TODAY, TOMORROW }
fun weatherMode(hour: Int): WeatherMode   // 6..15 (both inclusive) → TODAY, else → TOMORROW

class OpenMeteoClient(private val fetchText: suspend (url: String) -> String) {
    suspend fun forecast(lat: Double, lon: Double): WeatherSnapshot
    companion object { fun production(): OpenMeteoClient }
}
```
Exact request URL (asserted in test):
`https://api.open-meteo.com/v1/forecast?latitude={lat}&longitude={lon}&current=temperature_2m&daily=temperature_2m_max,temperature_2m_min,precipitation_probability_max&forecast_days=2&timezone=auto`

- [ ] **T5S1** (TEST-FIRST) Create `app/src/test/java/de/florian500/familyinfo/data/WeatherModeTest.kt`, exactly:
  ```kotlin
  package de.florian500.familyinfo.data

  import kotlin.test.Test
  import kotlin.test.assertEquals

  class WeatherModeTest {
      @Test fun hour_5_is_tomorrow() = assertEquals(WeatherMode.TOMORROW, weatherMode(5))
      @Test fun hour_6_is_today() = assertEquals(WeatherMode.TODAY, weatherMode(6))
      @Test fun hour_15_is_today() = assertEquals(WeatherMode.TODAY, weatherMode(15))
      @Test fun hour_16_is_tomorrow() = assertEquals(WeatherMode.TOMORROW, weatherMode(16))
      @Test fun hour_23_is_tomorrow() = assertEquals(WeatherMode.TOMORROW, weatherMode(23))
      @Test fun hour_0_is_tomorrow() = assertEquals(WeatherMode.TOMORROW, weatherMode(0))
  }
  ```
  **Done:** file exists with this content.

- [ ] **T5S2** (TEST-FIRST — MUST fail) Run:
  ```bash
  cd /Users/lindner/VSCode/FamilyInfoDisplay && ./gradlew :app:testDebugUnitTest --tests '*WeatherModeTest*' 2>&1 | tail -30
  ```
  **Done:** compile FAILURE (unresolved `weatherMode`).

- [ ] **T5S3** Create `app/src/main/java/de/florian500/familyinfo/data/WeatherMode.kt` + `app/src/main/java/de/florian500/familyinfo/data/WeatherModels.kt`:
  ```kotlin
  // WeatherMode.kt
  package de.florian500.familyinfo.data

  enum class WeatherMode { TODAY, TOMORROW }

  /** 06:00–15:59 → TODAY, 16:00–05:59 → TOMORROW (spec §7.2). */
  fun weatherMode(hour: Int): WeatherMode =
      if (hour in 6..15) WeatherMode.TODAY else WeatherMode.TOMORROW
  ```
  ```kotlin
  // WeatherModels.kt
  package de.florian500.familyinfo.data

  data class DayForecast(val tMin: Double, val tMax: Double, val rainChance: Int)

  data class WeatherSnapshot(
      val fetchedAtMillis: Long,
      val currentTemp: Double,
      val today: DayForecast,
      val tomorrow: DayForecast,
  )
  ```
  **Done:** both files exist.

- [ ] **T5S4** (TEST-FIRST — MUST now pass) Re-run:
  ```bash
  cd /Users/lindner/VSCode/FamilyInfoDisplay && ./gradlew :app:testDebugUnitTest --tests '*WeatherModeTest*' 2>&1 | tail -30
  ```
  **Done:** 6 tests passed, 0 failed.
- [ ] **T5S5** (TEST-FIRST) Create `app/src/test/java/de/florian500/familyinfo/data/OpenMeteoClientTest.kt`, exactly:
  ```kotlin
  package de.florian500.familyinfo.data

  import kotlinx.coroutines.test.runTest
  import kotlin.test.Test
  import kotlin.test.assertEquals

  class OpenMeteoClientTest {

      private val expectedUrl =
          "https://api.open-meteo.com/v1/forecast?latitude=52.444476&longitude=13.675989" +
              "&current=temperature_2m&daily=temperature_2m_max,temperature_2m_min," +
              "precipitation_probability_max&forecast_days=2&timezone=auto"

      // Recorded Open-Meteo response shape (v1/forecast, 2 days)
      private val sampleJson = """
          {
            "latitude": 52.444476,
            "longitude": 13.675989,
            "current": { "time": "2026-11-22T08:00", "temperature_2m": 8.3 },
            "daily": {
              "time": ["2026-11-22", "2026-11-23"],
              "temperature_2m_max": [12.1, 14.0],
              "temperature_2m_min": [5.2, -2.4],
              "precipitation_probability_max": [40, 70]
            }
          }
      """.trimIndent()

      private val nullRainJson = sampleJson.replace("\"precipitation_probability_max\": [40, 70]", "\"precipitation_probability_max\": [null, null]")

      @Test
      fun parses_recorded_response_and_requests_exact_url() = runTest {
          var requested: String? = null
          val client = OpenMeteoClient { url -> requested = url; sampleJson }
          val snap = client.forecast(52.444476, 13.675989)
          assertEquals(expectedUrl, requested)
          assertEquals(8.3, snap.currentTemp)
          assertEquals(DayForecast(5.2, 12.1, 40), snap.today)
          assertEquals(DayForecast(-2.4, 14.0, 70), snap.tomorrow)
          assertEquals(true, snap.fetchedAtMillis > 0)
      }

      @Test
      fun null_rain_probability_defaults_to_zero() = runTest {
          val client = OpenMeteoClient { nullRainJson }
          val snap = client.forecast(52.444476, 13.675989)
          assertEquals(0, snap.today.rainChance)
          assertEquals(0, snap.tomorrow.rainChance)
      }
  }
  ```
  **Done:** file exists with this content.

- [ ] **T5S6** (TEST-FIRST — MUST fail) Run:
  ```bash
  cd /Users/lindner/VSCode/FamilyInfoDisplay && ./gradlew :app:testDebugUnitTest --tests '*OpenMeteoClientTest*' 2>&1 | tail -30
  ```
  **Done:** compile FAILURE (unresolved `OpenMeteoClient`).
- [ ] **T5S7** Create `app/src/main/java/de/florian500/familyinfo/data/OpenMeteoClient.kt`, exactly:
  ```kotlin
  package de.florian500.familyinfo.data

  import kotlinx.serialization.Serializable
  import kotlinx.serialization.json.Json
  import java.net.HttpURLConnection
  import java.net.URL

  private val json = Json { ignoreUnknownKeys = true }

  @Serializable
  private class CurrentResponse { val temperature_2m: Double? = null }

  @Serializable
  private class DailyResponse(
      val time: List<String> = emptyList(),
      val temperature_2m_max: List<Double?> = emptyList(),
      val temperature_2m_min: List<Double?> = emptyList(),
      val precipitation_probability_max: List<Int?> = emptyList(),
  )

  @Serializable
  private class ForecastResponse(
      val current: CurrentResponse = CurrentResponse(),
      val daily: DailyResponse = DailyResponse(),
  )

  class OpenMeteoClient(private val fetchText: suspend (url: String) -> String) {

      suspend fun forecast(lat: Double, lon: Double): WeatherSnapshot {
          val url = "https://api.open-meteo.com/v1/forecast?latitude=$lat&longitude=$lon" +
              "&current=temperature_2m&daily=temperature_2m_max,temperature_2m_min," +
              "precipitation_probability_max&forecast_days=2&timezone=auto"
          val body = fetchText(url)
          val resp = json.decodeFromString<ForecastResponse>(body)

          fun day(index: Int): DayForecast = DayForecast(
              tMin = resp.daily.temperature_2m_min.getOrNull(index) ?: 0.0,
              tMax = resp.daily.temperature_2m_max.getOrNull(index) ?: 0.0,
              rainChance = resp.daily.precipitation_probability_max.getOrNull(index) ?: 0,
          )

          return WeatherSnapshot(
              fetchedAtMillis = System.currentTimeMillis(),
              currentTemp = resp.current.temperature_2m ?: 0.0,
              today = day(0),
              tomorrow = day(1),
          )
      }

      companion object {
          fun production(): OpenMeteoClient = OpenMeteoClient { url ->
              val conn = URL(url).openConnection() as HttpURLConnection
              conn.connectTimeout = 15_000
              conn.readTimeout = 15_000
              try {
                  require(conn.responseCode == HttpURLConnection.HTTP_OK) { "HTTP ${conn.responseCode}" }
                  conn.inputStream.bufferedReader().readText()
              } finally {
                  conn.disconnect()
              }
          }
      }
  }
  ```
  **Done:** file exists with this content.

- [ ] **T5S8** (TEST-FIRST — MUST now pass) Run:
  ```bash
  cd /Users/lindner/VSCode/FamilyInfoDisplay && ./gradlew :app:testDebugUnitTest --tests '*OpenMeteoClientTest*' 2>&1 | tail -30
  ```
  **Done:** 2 tests passed, 0 failed.

- [ ] **T5S9** Commit:
  ```bash
  cd /Users/lindner/VSCode/FamilyInfoDisplay && git add -A && git commit -m "feat: Open-Meteo weather data layer"
  ```
  **Done:** clean tree. **→ Current Position → T6S1, Remaining = 7.**

## Task 6: Weather tile UI

**Task done when:** build green; tablet shows the no-location state text (location can only be set in Task 10); committed.

**State model (binding — defined in `WeatherTile.kt`):**
```kotlin
sealed interface WeatherState {
    data object NoLocation : WeatherState
    data object Loading : WeatherState
    data class Data(val snapshot: WeatherSnapshot, val mode: WeatherMode) : WeatherState
}
```
Ownership: `MainActivity` creates `OpenMeteoClient.production()` wrapped in a `WeatherController` and passes it to `WeatherTile`. The controller holds `MutableStateFlow<WeatherState>`, runs the 15-min refresh loop, and on fetch failure KEEPS the previous (stale) snapshot (Review Focus #2).

- [ ] **T6S1** Create `app/src/main/java/de/florian500/familyinfo/ui/tiles/WeatherTile.kt`, exactly:
  ```kotlin
  package de.florian500.familyinfo.ui.tiles

  import androidx.compose.foundation.layout.Arrangement
  import androidx.compose.foundation.layout.Column
  import androidx.compose.foundation.layout.Row
  import androidx.compose.foundation.layout.fillMaxSize
  import androidx.compose.foundation.layout.fillMaxWidth
  import androidx.compose.foundation.layout.padding
  import androidx.compose.material3.MaterialTheme
  import androidx.compose.material3.Text
  import androidx.compose.runtime.Composable
  import androidx.compose.runtime.collectAsState
  import androidx.compose.ui.Alignment
  import androidx.compose.ui.Modifier
  import androidx.compose.ui.text.style.TextAlign
  import androidx.compose.ui.unit.dp
  import de.florian500.familyinfo.data.GeoPoint
  import de.florian500.familyinfo.data.OpenMeteoClient
  import de.florian500.familyinfo.data.WeatherMode
  import de.florian500.familyinfo.data.WeatherSnapshot
  import de.florian500.familyinfo.data.weatherMode
  import de.florian500.familyinfo.ui.theme.Alert
  import de.florian500.familyinfo.ui.theme.Ok
  import de.florian500.familyinfo.ui.theme.Warn
  import de.florian500.familyinfo.ui.theme.TextPrimary
  import de.florian500.familyinfo.ui.theme.TextSecondary
  import kotlinx.coroutines.CoroutineScope
  import kotlinx.coroutines.Job
  import kotlinx.coroutines.delay
  import kotlinx.coroutines.flow.MutableStateFlow
  import kotlinx.coroutines.flow.StateFlow
  import kotlinx.coroutines.launch
  import java.time.ZonedDateTime

  sealed interface WeatherState {
      data object NoLocation : WeatherState
      data object Loading : WeatherState
      data class Data(val snapshot: WeatherSnapshot, val mode: WeatherMode) : WeatherState
  }

  /** Owns the fetch loop (launch + every 15 min). Failure keeps the stale snapshot (Review Focus #2). */
  class WeatherController(private val client: OpenMeteoClient) {
      private val _state = MutableStateFlow<WeatherState>(WeatherState.NoLocation)
      val state: StateFlow<WeatherState> = _state
      private var refreshJob: Job? = null

      fun start(scope: CoroutineScope, location: GeoPoint?) {
          refreshJob?.cancel()
          if (location == null) {
              _state.value = WeatherState.NoLocation
              return
          }
          refreshJob = scope.launch {
              _state.value = WeatherState.Loading
              var last: WeatherSnapshot? = null
              while (true) {
                  val snapshot = try {
                      client.forecast(location.lat, location.lon)
                  } catch (e: Exception) {
                      last
                  }
                  last = snapshot
                  if (snapshot != null) {
                      _state.value = WeatherState.Data(snapshot, weatherMode(ZonedDateTime.now().hour))
                  }
                  delay(15 * 60_000)
              }
          }
      }
  }

  class WeatherTile(val controller: WeatherController) : FamilyTile {
      override val id: String = "weather"
      override val title: String = "Wetter"
      override val defaultVisible: Boolean = true

      @Composable
      override fun render() {
          val s = controller.state.collectAsState().value
          Panel(modifier = Modifier.fillMaxSize(), title = title) {
              when (val st = s) {
                  is WeatherState.NoLocation -> Text(
                      text = "Standort in den Einstellungen festlegen",
                      color = TextSecondary,
                      modifier = Modifier.fillMaxWidth().padding(vertical = 24.dp),
                      textAlign = TextAlign.Center,
                  )
                  is WeatherState.Loading -> Text(
                      text = "Wird geladen …",
                      color = TextSecondary,
                      modifier = Modifier.fillMaxWidth().padding(vertical = 24.dp),
                      textAlign = TextAlign.Center,
                  )
                  is WeatherState.Data -> WeatherContent(st.snapshot, st.mode)
              }
          }
      }
  }
  ```
  **Done:** file exists with exactly this content (imports at the top; no `import` inside functions).
- [ ] **T6S2** Append to `WeatherTile.kt` (same file, after `WeatherTile` class) the `WeatherContent` composable:
  ```kotlin
  @Composable
  private fun WeatherContent(snapshot: WeatherSnapshot, mode: WeatherMode) {
      val day = if (mode == WeatherMode.TODAY) snapshot.today else snapshot.tomorrow
      val statusColor = when {
          day.rainChance == 0 -> Ok
          day.rainChance < 50 -> Warn
          else -> Alert
      }
      val icon = when {
          day.rainChance >= 50 -> "☔"
          day.rainChance >= 20 -> "☁"
          else -> "☀"
      }
      Column(
          modifier = Modifier.fillMaxSize().padding(8.dp),
          verticalArrangement = Arrangement.spacedBy(8.dp),
      ) {
          Row(
              modifier = Modifier.fillMaxWidth(),
              horizontalArrangement = Arrangement.SpaceBetween,
              verticalAlignment = Alignment.CenterVertically,
          ) {
              Text(icon, style = MaterialTheme.typography.headlineSmall)
              Text(
                  text = if (mode == WeatherMode.TODAY) "Heute" else "Morgen",
                  color = TextSecondary,
                  style = MaterialTheme.typography.titleSmall,
              )
          }
          Text(
              text = if (mode == WeatherMode.TODAY) {
                  "${snapshot.currentTemp.toInt()} °C"
              } else {
                  "${day.tMin.toInt()} °C / ${day.tMax.toInt()} °C"
              },
              color = statusColor,
              style = MaterialTheme.typography.displayLarge,
          )
          Text(
              text = if (mode == WeatherMode.TODAY) "Höchste: ${day.tMax.toInt()} °C" else "Niedrigste: ${day.tMin.toInt()} °C",
              color = TextPrimary,
              style = MaterialTheme.typography.titleMedium,
          )
          Text(
              text = "Regen: ${day.rainChance} %",
              color = TextSecondary,
              style = MaterialTheme.typography.titleMedium,
          )
      }
  }
  ```
  **Done:** file now contains `WeatherState`, `WeatherController`, `WeatherTile`, `WeatherContent`; all four compile.

- [ ] **T6S3** Edit `Tile.kt`: replace the placeholder `weather` entry with the stateful class. Change the `TileRegistry` object to exactly:
  ```kotlin
  object TileRegistry {
      val weatherController = WeatherController(OpenMeteoClient.production())
      val weatherTile = WeatherTile(weatherController)
      val all: List<FamilyTile> = listOf(
          weatherTile,
          PlaceholderTile("calendar", "Termine"),
          PlaceholderTile("segelwetter", "Segelwetter"),
      )
  }
  ```
  (declaration order matters: controllers/tiles before `all`). Add imports `de.florian500.familyinfo.data.OpenMeteoClient`, `de.florian500.familyinfo.ui.tiles.WeatherTile`, `de.florian500.familyinfo.ui.tiles.WeatherController`.
  **Done:** `TileRegistry.all[0]` is a `WeatherTile`; `TileRegistry.weatherController` is the SINGLE controller instance (MainActivity starts this one in T6S4).

- [ ] **T6S4** Edit `MainActivity.kt`: inside `FamilyInfoTheme { ... }`, before `HomeScreen(...)`, add:
  ```kotlin
  val location by settingsRepo.location.collectAsStateWithLifecycle()
  LaunchedEffect(location) {
      TileRegistry.weatherController.start(lifecycleScope, location)
  }
  ```
  Do NOT create a second `WeatherController` in MainActivity — start the registry's `weatherController` (single instance). `lifecycleScope` is the Activity scope (already imported from T4S6). When `location` changes (null ↔ GeoPoint) the controller restarts: null → NoLocation, no network. Add imports: `androidx.compose.runtime.LaunchedEffect`, `de.florian500.familyinfo.ui.tiles.TileRegistry`.
  **Done:** MainActivity compiles; on first launch (no location) the weather tile shows the German no-location text.

- [ ] **T6S5** Build:
  ```bash
  cd /Users/lindner/VSCode/FamilyInfoDisplay && ./gradlew :app:assembleDebug 2>&1 | tail -40
  ```
  **Done:** `BUILD SUCCESSFUL`.

- [ ] **T6S6** Install + verify no-location state on tablet:
  ```bash
  adb -s HA2EN2VE install -r /Users/lindner/VSCode/FamilyInfoDisplay/app/build/outputs/apk/debug/app-debug.apk
  adb -s HA2EN2VE shell am start -n de.florian500.familyinfo/.MainActivity
  ```
  **Done:** weather tile shows "Standort in den Einstellungen festlegen"; no crash; no network error (nothing is fetched while location is null — confirm in `adb -s HA2EN2VE logcat -d | grep -i 'familyinfo\|open-meteo'` shows no failed fetch).

- [ ] **T6S7** Commit:
  ```bash
  cd /Users/lindner/VSCode/FamilyInfoDisplay && git add -A && git commit -m "feat: weather tile with time-based forecast"
  ```
  **Done:** clean tree. **→ Current Position → T7S1, Remaining = 6.**

## Task 7: Calendar data layer

**Task done when:** `CalendarRangeTest` (JVM) + `CalendarRepositoryInstrumentedTest` (device) green; test events cleaned up; committed.

**Interface (binding):**
```kotlin
data class CalendarEvent(val title: String, val startMillis: Long)
enum class EventDay { TODAY, TOMORROW }

class CalendarRepository(private val context: Context) {
    fun hasPermission(): Boolean
    fun events(): Flow<Map<EventDay, List<CalendarEvent>>>   // both days sorted by startMillis; empty lists when none
    fun refresh()
    companion object {
        fun dayRangeMillis(now: ZonedDateTime): Pair<Long, Long>  // [start of today, end of tomorrow)
    }
}
```
Query: `CalendarContract.Events` where `DTSTART >= range.first AND DTSTART < range.second`, projection `TITLE`, `DTSTART`, sorted `DTSTART ASC`; split into today/tomorrow by start-of-tomorrow; exposed as `StateFlow` refreshed on init AND on a `ContentObserver` on `CalendarContract.Events.CONTENT_URI` (main looper).

- [ ] **T7S1** Create `app/src/main/java/de/florian500/familyinfo/data/CalendarModels.kt`, exactly:
  ```kotlin
  package de.florian500.familyinfo.data

  data class CalendarEvent(val title: String, val startMillis: Long)

  enum class EventDay { TODAY, TOMORROW }
  ```
  **Done:** file exists.

- [ ] **T7S2** (TEST-FIRST) Create `app/src/test/java/de/florian500/familyinfo/data/CalendarRangeTest.kt`, exactly:
  ```kotlin
  package de.florian500.familyinfo.data

  import java.time.ZoneId
  import java.time.ZonedDateTime
  import kotlin.test.Test
  import kotlin.test.assertEquals

  class CalendarRangeTest {

      @Test
      fun range_is_today_midnight_to_day_after_tomorrow_midnight() {
          val now = ZonedDateTime.of(2026, 11, 21, 10, 0, 0, 0, ZoneId.of("Europe/Berlin"))
          val (first, second) = CalendarRepository.dayRangeMillis(now)
          val todayStart = ZonedDateTime.of(2026, 11, 21, 0, 0, 0, 0, ZoneId.of("Europe/Berlin")).toInstant().toEpochMilli()
          val dayAfterStart = ZonedDateTime.of(2026, 11, 23, 0, 0, 0, 0, ZoneId.of("Europe/Berlin")).toInstant().toEpochMilli()
          assertEquals(todayStart, first)
          assertEquals(dayAfterStart, second)
      }

      @Test
      fun range_spans_exactly_two_days() {
          val now = ZonedDateTime.of(2026, 1, 1, 23, 59, 0, 0, ZoneId.of("Europe/Berlin"))
          val (first, second) = CalendarRepository.dayRangeMillis(now)
          assertEquals(2L, (second - first) / (24 * 60 * 60 * 1000L))
      }
  }
  ```
  **Done:** file exists with this content.

- [ ] **T7S3** (TEST-FIRST — MUST fail) Run:
  ```bash
  cd /Users/lindner/VSCode/FamilyInfoDisplay && ./gradlew :app:testDebugUnitTest --tests '*CalendarRangeTest*' 2>&1 | tail -30
  ```
  **Done:** compile FAILURE (unresolved `CalendarRepository`).
- [ ] **T7S4** Create `app/src/main/java/de/florian500/familyinfo/data/CalendarRepository.kt`, exactly:
  ```kotlin
  package de.florian500.familyinfo.data

  import android.content.Context
  import android.content.pm.PackageManager
  import android.database.ContentObserver
  import android.os.Handler
  import android.provider.CalendarContract
  import androidx.core.content.ContextCompat
  import kotlinx.coroutines.CoroutineScope
  import kotlinx.coroutines.Dispatchers
  import kotlinx.coroutines.SupervisorJob
  import kotlinx.coroutines.flow.MutableStateFlow
  import kotlinx.coroutines.flow.StateFlow
  import kotlinx.coroutines.launch
  import kotlinx.coroutines.withContext
  import java.time.ZonedDateTime

  class CalendarRepository(private val context: Context) {

      private val scope = CoroutineScope(SupervisorJob() + Dispatchers.IO)
      private val _events = MutableStateFlow(
          mapOf(
              EventDay.TODAY to emptyList<CalendarEvent>(),
              EventDay.TOMORROW to emptyList<CalendarEvent>(),
          ),
      )

      private val observer = object : ContentObserver(Handler(context.mainLooper)) {
          override fun onChange(selfChange: Boolean) {
              refresh()
          }
      }

      init {
          context.contentResolver.registerContentObserver(
              CalendarContract.Events.CONTENT_URI, true, observer,
          )
          refresh()
      }

      fun events(): StateFlow<Map<EventDay, List<CalendarEvent>>> = _events

      fun hasPermission(): Boolean =
          ContextCompat.checkSelfPermission(
              context, android.Manifest.permission.READ_CALENDAR,
          ) == PackageManager.PERMISSION_GRANTED

      fun refresh() {
          scope.launch {
              val result = withContext(Dispatchers.IO) { queryEvents() }
              if (result != null) _events.value = result
          }
      }

      private fun queryEvents(): Map<EventDay, List<CalendarEvent>>? {
          if (!hasPermission()) return null
          val now = ZonedDateTime.now()
          val (first, second) = dayRangeMillis(now)
          val tomorrowStart = now.toLocalDate().plusDays(1)
              .atStartOfDay(now.zone).toInstant().toEpochMilli()
          val out = mutableMapOf(
              EventDay.TODAY to mutableListOf<CalendarEvent>(),
              EventDay.TOMORROW to mutableListOf<CalendarEvent>(),
          )
          context.contentResolver.query(
              CalendarContract.Events.CONTENT_URI,
              arrayOf(CalendarContract.Events.TITLE, CalendarContract.Events.DTSTART),
              "${CalendarContract.Events.DTSTART} >= $first AND ${CalendarContract.Events.DTSTART} < $second",
              null,
              "${CalendarContract.Events.DTSTART} ASC",
          )?.use { c ->
              val titleIdx = c.getColumnIndexOrThrow(CalendarContract.Events.TITLE)
              val startIdx = c.getColumnIndexOrThrow(CalendarContract.Events.DTSTART)
              while (c.moveToNext()) {
                  val title = c.getString(titleIdx) ?: "(keine Bezeichnung)"
                  val start = c.getLong(startIdx)
                  val day = if (start < tomorrowStart) EventDay.TODAY else EventDay.TOMORROW
                  out.getValue(day).add(CalendarEvent(title, start))
              }
          }
          out[EventDay.TODAY] = out.getValue(EventDay.TODAY).sortedBy { it.startMillis }
          out[EventDay.TOMORROW] = out.getValue(EventDay.TOMORROW).sortedBy { it.startMillis }
          return out
      }

      companion object {
          fun dayRangeMillis(now: ZonedDateTime): Pair<Long, Long> {
              val todayStart = now.toLocalDate().atStartOfDay(now.zone).toInstant().toEpochMilli()
              val dayAfterStart = now.toLocalDate().plusDays(2)
                  .atStartOfDay(now.zone).toInstant().toEpochMilli()
              return todayStart to dayAfterStart
          }
      }
  }
  ```
  **Done:** file exists with exactly this content; public surface matches the task interface.
- [ ] **T7S5** (TEST-FIRST — MUST now pass) Run:
  ```bash
  cd /Users/lindner/VSCode/FamilyInfoDisplay && ./gradlew :app:testDebugUnitTest --tests '*CalendarRangeTest*' 2>&1 | tail -30
  ```
  **Done:** 2 tests passed, 0 failed.

- [ ] **T7S6** Create `app/src/androidTest/java/de/florian500/familyinfo/data/CalendarRepositoryInstrumentedTest.kt`, exactly:
  ```kotlin
  package de.florian500.familyinfo.data

  import android.app.Application
  import android.content.ContentResolver
  import android.content.ContentValues
  import android.provider.CalendarContract
  import androidx.test.core.app.ApplicationProvider
  import androidx.test.core.content.grantPermission
  import androidx.test.ext.junit.runners.AndroidJUnit4
  import kotlinx.coroutines.delay
  import kotlinx.coroutines.flow.first
  import kotlinx.coroutines.runBlocking
  import org.junit.After
  import org.junit.Before
  import org.junit.Test
  import org.junit.runner.RunWith
  import java.time.ZonedDateTime
  import kotlin.test.assertEquals
  import kotlin.test.assertTrue

  @RunWith(AndroidJUnit4::class)
  class CalendarRepositoryInstrumentedTest {

      private lateinit var resolver: ContentResolver
      private val insertedIds = mutableListOf<Long>()
      private val marker = "FID_TEST_EVENT_"

      @Before
      fun setup() {
          val ctx = ApplicationProvider.getApplicationContext<Application>()
          resolver = ctx.contentResolver
          grantPermission("android.permission.READ_CALENDAR", ctx.packageName)
      }

      @After
      fun cleanup() {
          while (insertedIds.isNotEmpty()) {
              val id = insertedIds.removeAt(0)
              resolver.delete(CalendarContract.Events.CONTENT_URI, "${CalendarContract.Events._ID} = $id", null)
          }
      }

      private fun defaultCalendarId(): Long {
          val c = resolver.query(
              CalendarContract.Calendars.CONTENT_URI,
              arrayOf(CalendarContract.Calendars._ID),
              null, null, null,
          ) ?: error("no calendars")
          return c.use {
              assertTrue(it.moveToFirst()) { "no calendar found" }
              it.getLong(0)
          }
      }

      private fun insertEvent(title: String, startMillis: Long) {
          val values = ContentValues().apply {
              put(CalendarContract.Events.CALENDAR_ID, defaultCalendarId())
              put(CalendarContract.Events.TITLE, title)
              put(CalendarContract.Events.DTSTART, startMillis)
              put(CalendarContract.Events.DURATION, 3_600_000L)
              put(CalendarContract.Events.EVENT_TIMEZONE, "Europe/Berlin")
          }
          resolver.insert(CalendarContract.Events.CONTENT_URI, values) ?: error("insert failed")
          val c = resolver.query(
              CalendarContract.Events.CONTENT_URI,
              arrayOf(CalendarContract.Events._ID),
              "${CalendarContract.Events.TITLE} = ?", arrayOf(title), null,
          )
          c.use {
              assertTrue(it.moveToFirst())
              insertedIds.add(it.getLong(0))
          }
      }

      private fun waitUntil(repo: CalendarRepository, timeoutMs: Long = 10_000, condition: (Map<EventDay, List<CalendarEvent>>) -> Boolean): Boolean {
          var waited = 0L
          while (waited < timeoutMs) {
              if (condition(repo.events().first())) return true
              delay(200)
              waited += 200
          }
          return condition(repo.events().first())
      }

      @Test
      fun sees_today_tomorrow_and_refreshes_on_delete() = runBlocking {
          val ctx = ApplicationProvider.getApplicationContext<Application>()
          val repo = CalendarRepository(ctx)
          val now = ZonedDateTime.now()
          val today = now.toLocalDate().atTime(9, 0).atZone(now.zone).toInstant().toEpochMilli()
          val tomorrow = now.toLocalDate().plusDays(1).atTime(9, 0).atZone(now.zone).toInstant().toEpochMilli()
          val nextWeek = now.toLocalDate().plusDays(7).atTime(9, 0).atZone(now.zone).toInstant().toEpochMilli()

          insertEvent("$marker_today", today)
          insertEvent("$marker_tomorrow", tomorrow)
          insertEvent("$marker_nextweek", nextWeek)

          assertTrue(waitUntil(repo) { it.getValue(EventDay.TODAY).any { e -> e.title == "$marker_today" } })
          var map = repo.events().first()
          assertTrue(map.getValue(EventDay.TODAY).any { it.title == "$marker_today" })
          assertTrue(map.getValue(EventDay.TOMORROW).any { it.title == "$marker_tomorrow" })
          assertTrue(map.values.flatten().none { it.title == "$marker_nextweek" })

          val tomorrowId = insertedIds[1]
          resolver.delete(CalendarContract.Events.CONTENT_URI, "${CalendarContract.Events._ID} = $tomorrowId", null)
          insertedIds.removeAt(1)
          assertTrue(waitUntil(repo) { it.getValue(EventDay.TOMORROW).none { e -> e.title.startsWith(marker) } })
      }
  }
  ```
  **Done:** file exists with exactly this content. (If the tablet has no default calendar, `defaultCalendarId()` errors — that is an environmental precondition, not a code defect: fix by enabling Google calendar sync on the tablet, see Failure playbook.)
- [ ] **T7S7** Run the instrumented test on the connected tablet (device MUST be attached — check `adb devices` shows `HA2EN2VE`):
  ```bash
  cd /Users/lindner/VSCode/FamilyInfoDisplay && ./gradlew :app:connectedDebugAndroidTest 2>&1 | tail -40
  ```
  **Done:** BUILD SUCCESSFUL; `CalendarRepositoryInstrumentedTest.sees_today_tomorrow_and_refreshes_on_delete` passes; the tablet's calendar has no leftover `FID_TEST_EVENT_*` events (verify visually or via the cleanup — `@After` removes them). If a test inserted an event but the run failed, clean up manually: tablet → Calendar app → delete the FID_TEST_EVENT entries.

- [ ] **T7S8** Commit:
  ```bash
  cd /Users/lindner/VSCode/FamilyInfoDisplay && git add -A && git commit -m "feat: calendar data layer (ContentProvider + observer)"
  ```
  **Done:** clean tree. **→ Current Position → T8S1, Remaining = 5.**

## Task 8: Calendar tile UI

**Task done when:** build green; real family events visible on the tablet; permission-deny state shows the grant button without crash; committed. Per the T4S4 wiring note: the calendar tile is INJECTED into HomeScreen (not in `TileRegistry`), because it needs `Context` + a permission callback.

- [ ] **T8S1** Create `app/src/main/java/de/florian500/familyinfo/ui/tiles/CalendarTile.kt`, exactly:
  ```kotlin
  package de.florian500.familyinfo.ui.tiles

  import androidx.compose.foundation.layout.Arrangement
  import androidx.compose.foundation.layout.Column
  import androidx.compose.foundation.layout.Row
  import androidx.compose.foundation.layout.fillMaxSize
  import androidx.compose.foundation.layout.fillMaxWidth
  import androidx.compose.foundation.layout.padding
  import androidx.compose.foundation.layout.width
  import androidx.compose.foundation.lazy.LazyColumn
  import androidx.compose.foundation.lazy.items
  import androidx.compose.material3.Button
  import androidx.compose.material3.MaterialTheme
  import androidx.compose.material3.Text
  import androidx.compose.runtime.Composable
  import androidx.compose.runtime.collectAsState
  import androidx.compose.runtime.getValue
  import androidx.compose.ui.Alignment
  import androidx.compose.ui.Modifier
  import androidx.compose.ui.text.style.TextAlign
  import androidx.compose.ui.unit.dp
  import de.florian500.familyinfo.data.CalendarEvent
  import de.florian500.familyinfo.data.CalendarRepository
  import de.florian500.familyinfo.data.EventDay
  import de.florian500.familyinfo.ui.theme.Accent
  import de.florian500.familyinfo.ui.theme.AppBackground
  import de.florian500.familyinfo.ui.theme.TextPrimary
  import de.florian500.familyinfo.ui.theme.TextSecondary
  import kotlinx.coroutines.flow.StateFlow
  import java.time.Instant
  import java.time.ZoneId
  import java.time.format.DateTimeFormatter

  class CalendarTile(
      private val repo: CalendarRepository,
      private val permissionGranted: StateFlow<Boolean>,
      private val onRequestPermission: () -> Unit,
  ) : FamilyTile {
      override val id: String = "calendar"
      override val title: String = "Termine"
      override val defaultVisible: Boolean = true

      @Composable
      override fun render() {
          val hasPermission by permissionGranted.collectAsState()
          val events by repo.events().collectAsState()
          Panel(modifier = Modifier.fillMaxSize(), title = title) {
              if (!hasPermission) {
                  NoPermission(onRequestPermission)
              } else {
                  EventList(events[EventDay.TODAY].orEmpty(), events[EventDay.TOMORROW].orEmpty())
              }
          }
      }
  }

  private val timeFormatter = DateTimeFormatter.ofPattern("HH:mm")

  @Composable
  private fun NoPermission(onRequestPermission: () -> Unit) {
      Column(
          modifier = Modifier.fillMaxSize(),
          verticalArrangement = Arrangement.Center,
          horizontalAlignment = Alignment.CenterHorizontally,
      ) {
          Text("Kalenderzugriff erlauben", color = TextSecondary, textAlign = TextAlign.Center)
          Button(onClick = onRequestPermission, modifier = Modifier.padding(top = 12.dp)) {
              Text("Erlauben", color = AppBackground)
          }
      }
  }

  @Composable
  private fun EventList(today: List<CalendarEvent>, tomorrow: List<CalendarEvent>) {
      if (today.isEmpty() && tomorrow.isEmpty()) {
          Text("Keine Termine", color = TextSecondary, modifier = Modifier.fillMaxSize(), textAlign = TextAlign.Center)
      } else {
          LazyColumn(modifier = Modifier.fillMaxSize()) {
              item { DayHeader("Heute", today.size) }
              if (today.isEmpty()) item { EmptyDay() } else items(today) { EventRow(it) }
              item { DayHeader("Morgen", tomorrow.size) }
              if (tomorrow.isEmpty()) item { EmptyDay() } else items(tomorrow) { EventRow(it) }
          }
      }
  }

  @Composable
  private fun DayHeader(day: String, count: Int) {
      Text(
          text = "$day ($count)",
          color = TextSecondary,
          style = MaterialTheme.typography.titleSmall,
          modifier = Modifier.padding(top = 8.dp, bottom = 4.dp),
      )
  }

  @Composable
  private fun EmptyDay() {
      Text("–", color = TextSecondary, modifier = Modifier.padding(bottom = 4.dp))
  }

  @Composable
  private fun EventRow(event: CalendarEvent) {
      Row(modifier = Modifier.fillMaxWidth().padding(vertical = 3.dp)) {
          Text(
              text = timeOf(event.startMillis),
              color = Accent,
              style = MaterialTheme.typography.titleMedium,
              modifier = Modifier.width(56.dp),
          )
          Text(text = event.title, color = TextPrimary, style = MaterialTheme.typography.titleMedium)
      }
  }

  private fun timeOf(millis: Long): String =
      Instant.ofEpochMilli(millis).atZone(ZoneId.systemDefault()).toLocalTime().format(timeFormatter)
  ```
  **Done:** file exists with exactly this content.
- [ ] **T8S2** Edit `Tile.kt` `TileRegistry`: remove the calendar placeholder (the tile is injected now — T4S4 wiring note). `all` becomes:
  ```kotlin
      val all: List<FamilyTile> = listOf(
          weatherTile,
          PlaceholderTile("segelwetter", "Segelwetter"),
      )
  ```
  **Done:** `all` has 2 entries (weather, segelwetter).

- [ ] **T8S3** Update `app/src/test/java/de/florian500/familyinfo/ui/TileRegistryTest.kt` to the new registry size, exactly:
  ```kotlin
  package de.florian500.familyinfo.ui

  import de.florian500.familyinfo.ui.tiles.TileRegistry
  import de.florian500.familyinfo.ui.tiles.visibleTiles
  import kotlin.test.Test
  import kotlin.test.assertEquals
  import kotlin.test.assertTrue

  class TileRegistryTest {

      @Test
      fun registry_contains_context_free_tiles_in_order() {
          assertEquals(listOf("weather", "segelwetter"), TileRegistry.all.map { it.id })
          assertTrue(TileRegistry.all.all { it.defaultVisible })
      }

      @Test
      fun visibleTiles_filters_hidden_preserving_order() {
          assertEquals(listOf("weather"), visibleTiles(setOf("segelwetter")).map { it.id })
      }

      @Test
      fun visibleTiles_no_hidden_returns_all() {
          assertEquals(listOf("weather", "segelwetter"), visibleTiles(emptySet()).map { it.id })
      }
  }
  ```
  **Done:** file updated.

- [ ] **T8S4** (TEST-FIRST — run updated test; it should already pass now, since T8S2 changed the registry before the test — if you changed the test first instead, it must have failed. Either order is fine; the pair must end green.) Run:
  ```bash
  cd /Users/lindner/VSCode/FamilyInfoDisplay && ./gradlew :app:testDebugUnitTest --tests '*TileRegistryTest*' 2>&1 | tail -30
  ```
  **Done:** 3 tests passed, 0 failed.
- [ ] **T8S5** Edit `HomeScreen.kt`: change the signature and the calendar slot to use the injected tile. The composable becomes (full file, replace entirely):
  ```kotlin
  package de.florian500.familyinfo.ui

  import androidx.compose.foundation.background
  import androidx.compose.foundation.layout.Arrangement
  import androidx.compose.foundation.layout.Box
  import androidx.compose.foundation.layout.Column
  import androidx.compose.foundation.layout.Row
  import androidx.compose.foundation.layout.fillMaxHeight
  import androidx.compose.foundation.layout.fillMaxSize
  import androidx.compose.foundation.layout.fillMaxWidth
  import androidx.compose.foundation.layout.padding
  import androidx.compose.runtime.Composable
  import androidx.compose.ui.Modifier
  import androidx.compose.ui.unit.dp
  import de.florian500.familyinfo.ui.tiles.FamilyTile
  import de.florian500.familyinfo.ui.tiles.visibleTiles
  import de.florian500.familyinfo.ui.theme.AppBackground

  /**
   * Fixed two-column layout (spec §7.1).
   * col 1: weather (top, 34% weight) + calendar (fills rest, content scrollable)
   * col 2: segelwetter full column height
   * Hidden tiles collapse; remaining tiles in their column expand (weights renormalize).
   * `calendarTile` is injected (needs Context + permission callback, T4S4 wiring note).
   * Future tiles beyond the initial three are appended in a row below col 1 (spec §7.1).
   */
  @Composable
  fun HomeScreen(hidden: Set<String>, calendarTile: FamilyTile?) {
      val visible = visibleTiles(hidden)
      val weather = visible.firstOrNull { it.id == "weather" }
      val segelwetter = visible.firstOrNull { it.id == "segelwetter" }
      val showCalendar = "calendar" !in hidden && calendarTile != null
      val extra = visible.filter { it.id !in setOf("weather", "calendar", "segelwetter") }

      Row(
          modifier = Modifier
              .fillMaxSize()
              .background(AppBackground)
              .padding(12.dp),
          horizontalArrangement = Arrangement.spacedBy(12.dp),
      ) {
          Column(
              modifier = Modifier
                  .weight(1f)
                  .fillMaxHeight(),
              verticalArrangement = Arrangement.spacedBy(12.dp),
          ) {
              if (weather != null && showCalendar) {
                  Box(modifier = Modifier.weight(0.34f).fillMaxWidth()) { weather.render() }
                  Box(modifier = Modifier.weight(0.66f).fillMaxWidth()) { calendarTile?.render() }
              } else {
                  weather?.let { Box(modifier = Modifier.weight(1f).fillMaxWidth()) { it.render() } }
                  if (showCalendar) Box(modifier = Modifier.weight(1f).fillMaxWidth()) { calendarTile?.render() }
              }
              if (extra.isNotEmpty()) {
                  Row(
                      modifier = Modifier.weight(0.4f).fillMaxWidth(),
                      horizontalArrangement = Arrangement.spacedBy(12.dp),
                  ) {
                      extra.forEach { t -> Box(modifier = Modifier.weight(1f)) { t.render() } }
                  }
              }
          }
          segelwetter?.let { tile ->
              Box(modifier = Modifier.weight(1f).fillMaxHeight()) { tile.render() }
          }
      }
  }
  ```
  **Done:** file exactly matches; `calendarTile == null` or hidden → slot collapsed, weather expands to full column.

- [ ] **T8S6** Edit `MainActivity.kt` — full file, replace entirely:
  ```kotlin
  package de.florian500.familyinfo

  import android.content.SharedPreferences
  import android.content.pm.PackageManager
  import android.os.Bundle
  import androidx.activity.ComponentActivity
  import androidx.activity.compose.setContent
  import androidx.activity.result.contract.ActivityResultContracts
  import androidx.compose.runtime.LaunchedEffect
  import androidx.compose.runtime.getValue
  import androidx.core.content.ContextCompat
  import androidx.lifecycle.compose.collectAsStateWithLifecycle
  import androidx.lifecycle.lifecycleScope
  import de.florian500.familyinfo.data.CalendarRepository
  import de.florian500.familyinfo.data.SettingsRepository
  import de.florian500.familyinfo.ui.HomeScreen
  import de.florian500.familyinfo.ui.tiles.CalendarTile
  import de.florian500.familyinfo.ui.tiles.TileRegistry
  import de.florian500.familyinfo.ui.theme.FamilyInfoTheme
  import kotlinx.coroutines.flow.MutableStateFlow

  class MainActivity : ComponentActivity() {

      private val requestCalendarPermission =
          registerForActivityResult(ActivityResultContracts.RequestPermission()) { granted ->
              refreshPermissionState()
          }

      private lateinit var permissionGranted: MutableStateFlow<Boolean>

      private fun refreshPermissionState() {
          permissionGranted.value = ContextCompat.checkSelfPermission(
              this, android.Manifest.permission.READ_CALENDAR,
          ) == PackageManager.PERMISSION_GRANTED
      }

      override fun onCreate(savedInstanceState: Bundle?) {
          super.onCreate(savedInstanceState)
          val settingsRepo = SettingsRepository.create(this)
          val calendarRepo = CalendarRepository(this)

          permissionGranted = MutableStateFlow(ContextCompat.checkSelfPermission(
              this, android.Manifest.permission.READ_CALENDAR,
          ) == PackageManager.PERMISSION_GRANTED)
          // Permission changes made in system settings (not via our dialog) also reach us:
          getSharedPreferences(packageName + "_preferences", MODE_PRIVATE)
              .registerOnSharedPreferenceChangeListener(
                  SharedPreferences.OnSharedPreferenceChangeListener { _, _ -> refreshPermissionState() },
              )

          val calendarTile = CalendarTile(
              repo = calendarRepo,
              permissionGranted = permissionGranted,
              onRequestPermission = { requestCalendarPermission.launch("android.permission.READ_CALENDAR") },
          )

          setContent {
              FamilyInfoTheme {
                  val hidden by settingsRepo.hiddenTileIds.collectAsStateWithLifecycle()
                  val location by settingsRepo.location.collectAsStateWithLifecycle()
                  LaunchedEffect(location) {
                      TileRegistry.weatherController.start(lifecycleScope, location)
                  }
                  HomeScreen(hidden = hidden, calendarTile = calendarTile)
              }
          }
      }
  }
  ```
  **Done:** file exactly matches. Note: the `OnSharedPreferenceChangeListener` approach observes permission changes the user makes directly in system settings (Review Focus #4); if it proves flaky on ZUI, the T8S7 dialog path still covers the in-app grant.

- [ ] **T8S7** Build:
  ```bash
  cd /Users/lindner/VSCode/FamilyInfoDisplay && ./gradlew :app:assembleDebug 2>&1 | tail -40
  ```
  **Done:** `BUILD SUCCESSFUL`.

- [ ] **T8S8** Install + verify on tablet (family calendar is synced on it):
  ```bash
  adb -s HA2EN2VE install -r /Users/lindner/VSCode/FamilyInfoDisplay/app/build/outputs/apk/debug/app-debug.apk
  adb -s HA2EN2VE shell am start -n de.florian500.familyinfo/.MainActivity
  ```
  **Done (two states to verify):**
  1. Permission NOT yet granted → "Termine" tile shows "Kalenderzugriff erlauben" + button; tap → system dialog appears; grant → tile switches to "Heute (n)" / "Morgen (n)" sections with real events (time in accent color + title). If events are busy, the list scrolls inside the tile area.
  2. (Optional, if time allows) Deny the permission in system settings → tile shows the grant button again (listener works), app stays stable.
  **Done:** both states observed without crash.

- [ ] **T8S9** Commit:
  ```bash
  cd /Users/lindner/VSCode/FamilyInfoDisplay && git add -A && git commit -m "feat: calendar tile with today/tomorrow events"
  ```
  **Done:** clean tree. **→ Current Position → T9S1, Remaining = 4.**

## Task 9: Segelwetter tile (WebView)

**Task done when:** build green; live page with map visible on the tablet; committed.

- [ ] **T9S1** Create `app/src/main/java/de/florian500/familyinfo/ui/tiles/SegelwetterTile.kt`, exactly:
  ```kotlin
  package de.florian500.familyinfo.ui.tiles

  import android.annotation.SuppressLint
  import android.webkit.WebChromeClient
  import android.webkit.WebResourceError
  import android.webkit.WebResourceRequest
  import android.webkit.WebView
  import android.webkit.WebViewClient
  import androidx.compose.foundation.background
  import androidx.compose.foundation.layout.Box
  import androidx.compose.foundation.layout.fillMaxSize
  import androidx.compose.material3.Text
  import androidx.compose.runtime.Composable
  import androidx.compose.runtime.DisposableEffect
  import androidx.compose.runtime.getValue
  import androidx.compose.runtime.mutableStateOf
  import androidx.compose.runtime.remember
  import androidx.compose.runtime.setValue
  import androidx.compose.ui.Alignment
  import androidx.compose.ui.Modifier
  import androidx.compose.ui.platform.LocalContext
  import androidx.compose.ui.text.style.TextAlign
  import androidx.compose.ui.viewinterop.AndroidView
  import androidx.compose.ui.platform.LocalLifecycleOwner
  import androidx.lifecycle.Lifecycle
  import androidx.lifecycle.LifecycleEventObserver
  import de.florian500.familyinfo.ui.theme.PanelBackground
  import de.florian500.familyinfo.ui.theme.TextSecondary

  class SegelwetterTile : FamilyTile {
      override val id: String = "segelwetter"
      override val title: String = "Segelwetter"
      override val defaultVisible: Boolean = true

      @SuppressLint("SetJavaScriptEnabled")
      @Composable
      override fun render() {
          val context = LocalContext.current
          val webViewHolder = remember { arrayOfNulls<WebView>(1) }
          var errorVisible by remember { mutableStateOf(false) }
          val lifecycle = LocalLifecycleOwner.current.lifecycle

          DisposableEffect(lifecycle) {
              val observer = LifecycleEventObserver { _, event ->
                  when (event) {
                      Lifecycle.Event.ON_PAUSE -> webViewHolder[0]?.onPause()
                      Lifecycle.Event.ON_RESUME -> webViewHolder[0]?.onResume()
                      else -> Unit
                  }
              }
              lifecycle.addObserver(observer)
              onDispose { lifecycle.removeObserver(observer) }
          }

          Panel(modifier = Modifier.fillMaxSize(), title = title) {
              Box(modifier = Modifier.fillMaxSize()) {
                  AndroidView(
                      factory = { ctx ->
                          WebView(ctx).also { webViewHolder[0] = it }.apply {
                              settings.javaScriptEnabled = true
                              settings.domStorageEnabled = true
                              webViewClient = object : WebViewClient() {
                                  override fun onPageFinished(view: WebView, url: String) {
                                      errorVisible = false
                                  }
                                  override fun onReceivedError(
                                      view: WebView,
                                      request: WebResourceRequest,
                                      error: WebResourceError,
                                  ) {
                                      if (request.isForMainFrame) errorVisible = true
                                  }
                              }
                              webChromeClient = object : WebChromeClient() {}
                              loadUrl("https://superfloh247.github.io/Segelwetter/")
                          }
                      },
                      modifier = Modifier.fillMaxSize(),
                  )
                  if (errorVisible) {
                      Box(
                          modifier = Modifier.fillMaxSize().background(PanelBackground),
                          contentAlignment = Alignment.Center,
                      ) {
                          Text(
                              text = "Seite nicht erreichbar – keine Verbindung?",
                              color = TextSecondary,
                              textAlign = TextAlign.Center,
                          )
                      }
                  }
              }
          }
      }
  }
  ```
  **Done:** file exists with exactly this content. (Error overlay per Review Focus #5: tile keeps its header, page error → German overlay, no crash. If the site later breaks and the overlay shows on a reachable page, check `adb -s HA2EN2VE logcat -s chromium,console`.)

- [ ] **T9S2** Edit `Tile.kt` `TileRegistry`: replace `PlaceholderTile("segelwetter", "Segelwetter")` with `SegelwetterTile()` (add import if needed — same package, so none).
  **Done:** `all` = weather, segelwetter (both real).

- [ ] **T9S3** Build:
  ```bash
  cd /Users/lindner/VSCode/FamilyInfoDisplay && ./gradlew :app:assembleDebug 2>&1 | tail -40
  ```
  **Done:** `BUILD SUCCESSFUL`.

- [ ] **T9S4** Install + verify on tablet:
  ```bash
  adb -s HA2EN2VE install -r /Users/lindner/VSCode/FamilyInfoDisplay/app/build/outputs/apk/debug/app-debug.apk
  adb -s HA2EN2VE shell am start -n de.florian500.familyinfo/.MainActivity
  ```
  **Done:** col 2 shows the "Segelwetter" header + the live page; the Leaflet map renders; the page's own "Standort" control works (defaults to 52.444476, 13.675989). Rotate-free: tablet is landscape (no lock yet — Task 12), confirm layout is still two-column.

- [ ] **T9S5** Commit:
  ```bash
  cd /Users/lindner/VSCode/FamilyInfoDisplay && git add -A && git commit -m "feat: Segelwetter WebView tile"
  ```
  **Done:** clean tree. **→ Current Position → T10S1, Remaining = 3.**

## Task 10: Settings screen (tiles + weather location)

**Task done when:** build green; toggling + location setting persist across force-stop/relaunch; committed.
No navigation library: `MainActivity` holds `var screen by remember { mutableStateOf(Screen.HOME) }` with `enum class Screen { HOME, SETTINGS, ABOUT }` and a top-level switch (ABOUT wired in Task 11).

- [ ] **T10S1** Replace `app/src/main/res/values/strings.xml` with:
  ```xml
  <?xml version="1.0" encoding="utf-8"?>
  <resources>
      <string name="app_name">FamilyInfo</string>
      <string name="einstellungen">Einstellungen</string>
      <string name="kacheln">Kacheln</string>
      <string name="wetterstandort">Wetterstandort</string>
      <string name="standort_ermitteln">Standort ermitteln</string>
      <string name="breitengrad">Breitengrad</string>
      <string name="laengengrad">Längengrad</string>
      <string name="info">Info</string>
  </resources>
  ```
  **Done:** file updated. (In-code UI strings stay as German literals — consistent with the rest of the app; these entries exist for the manifest/label side and future use.)

- [ ] **T10S2** Create `app/src/main/java/de/florian500/familyinfo/ui/SettingsScreen.kt`. PART 1 (imports + helper + screen start):
  ```kotlin
  package de.florian500.familyinfo.ui

  import android.Manifest
  import android.content.Context
  import android.content.pm.PackageManager
  import android.location.LocationManager
  import androidx.activity.compose.rememberLauncherForActivityResult
  import androidx.activity.result.contract.ActivityResultContracts
  import androidx.compose.foundation.background
  import androidx.compose.foundation.clickable
  import androidx.compose.foundation.layout.Arrangement
  import androidx.compose.foundation.layout.Column
  import androidx.compose.foundation.layout.Row
  import androidx.compose.foundation.layout.Spacer
  import androidx.compose.foundation.layout.fillMaxSize
  import androidx.compose.foundation.layout.fillMaxWidth
  import androidx.compose.foundation.layout.padding
  import androidx.compose.foundation.rememberScrollState
  import androidx.compose.foundation.verticalScroll
  import androidx.compose.foundation.text.KeyboardOptions
  import androidx.compose.material3.Button
  import androidx.compose.material3.ButtonDefaults
  import androidx.compose.material3.MaterialTheme
  import androidx.compose.material3.OutlinedTextField
  import androidx.compose.material3.Switch
  import androidx.compose.material3.Text
  import androidx.compose.material3.TextButton
  import androidx.compose.runtime.Composable
  import androidx.compose.runtime.getValue
  import androidx.compose.runtime.mutableStateOf
  import androidx.compose.runtime.remember
  import androidx.compose.runtime.rememberCoroutineScope
  import androidx.compose.runtime.setValue
  import androidx.compose.ui.Alignment
  import androidx.compose.ui.Modifier
  import androidx.compose.ui.platform.LocalContext
  import androidx.compose.ui.text.input.KeyboardType
  import androidx.compose.ui.unit.dp
  import androidx.core.content.ContextCompat
  import androidx.lifecycle.compose.collectAsStateWithLifecycle
  import de.florian500.familyinfo.data.GeoPoint
  import de.florian500.familyinfo.data.SettingsRepository
  import de.florian500.familyinfo.ui.tiles.FamilyTile
  import de.florian500.familyinfo.ui.tiles.TileRegistry
  import de.florian500.familyinfo.ui.theme.Accent
  import de.florian500.familyinfo.ui.theme.Alert
  import de.florian500.familyinfo.ui.theme.AppBackground
  import de.florian500.familyinfo.ui.theme.PanelBackground
  import de.florian500.familyinfo.ui.theme.TextPrimary
  import de.florian500.familyinfo.ui.theme.TextSecondary
  import kotlinx.coroutines.launch

  /** All toggleable tiles, fixed order: weather, calendar, segelwetter (calendar injected, T4S4 note). */
  fun allToggleableTiles(calendarTile: FamilyTile): List<FamilyTile> {
      val base = TileRegistry.all.toMutableList()
      base.add(1, calendarTile)
      return base
  }

  @Composable
  fun SettingsScreen(
      settingsRepo: SettingsRepository,
      calendarTile: FamilyTile,
      onBack: () -> Unit,
      onOpenAbout: () -> Unit,
  ) {
      val context = LocalContext.current
      val scope = rememberCoroutineScope()
      val hidden by settingsRepo.hiddenTileIds.collectAsStateWithLifecycle()
      val location by settingsRepo.location.collectAsStateWithLifecycle()
      var latText by remember(location) { mutableStateOf(location?.lat?.toString() ?: "") }
      var lonText by remember(location) { mutableStateOf(location?.lon?.toString() ?: "") }
      var locationError by remember { mutableStateOf(false) }
      var locateError by remember { mutableStateOf(false) }

      fun commitLocation() {
          val lat = latText.toDoubleOrNull()
          val lon = lonText.toDoubleOrNull()
          if (lat != null && lon != null) {
              locationError = false
              scope.launch { settingsRepo.setLocation(GeoPoint(lat, lon)) }
          } else {
              locationError = true
          }
      }

      fun locate() {
          val lm = context.getSystemService(Context.LOCATION_SERVICE) as LocationManager
          val last = try { lm.getLastLocation(LocationManager.GPS_PROVIDER) } catch (e: SecurityException) { null }
              ?: try { lm.getLastLocation(LocationManager.NETWORK_PROVIDER) } catch (e: SecurityException) { null }
          if (last != null) {
              val p = GeoPoint(last.latitude, last.longitude)
              latText = p.lat.toString()
              lonText = p.lon.toString()
              locationError = false
              scope.launch { settingsRepo.setLocation(p) }
          } else {
              locateError = true
          }
      }

      val requestLocationPermission = rememberLauncherForActivityResult(
          ActivityResultContracts.RequestPermission(),
      ) { granted ->
          locateError = false
          if (granted) locate() else locateError = true
      }

      Column(
          modifier = Modifier
              .fillMaxSize()
              .background(PanelBackground)
              .padding(20.dp),
      ) {
          Row(verticalAlignment = Alignment.CenterVertically) {
              Text(
                  text = "←",
                  color = Accent,
                  style = MaterialTheme.typography.headlineSmall,
                  modifier = Modifier
                      .padding(8.dp)
                      .clickable(onClick = onBack),
              )
              Text(
                  text = "Einstellungen",
                  color = TextPrimary,
                  style = MaterialTheme.typography.headlineSmall,
                  modifier = Modifier.padding(start = 8.dp),
              )
          }

          Column(
              modifier = Modifier
                  .fillMaxSize()
                  .verticalScroll(rememberScrollState())
                  .padding(top = 20.dp),
              verticalArrangement = Arrangement.spacedBy(8.dp),
          ) {
              Text("Kacheln", color = TextSecondary, style = MaterialTheme.typography.titleMedium)
              allToggleableTiles(calendarTile).forEach { tile ->
                  Row(
                      modifier = Modifier.fillMaxWidth(),
                      verticalAlignment = Alignment.CenterVertically,
                  ) {
                      Text(tile.title, color = TextPrimary, style = MaterialTheme.typography.titleMedium)
                      Spacer(modifier = Modifier.weight(1f))
                      Switch(
                          checked = tile.id !in hidden,
                          onCheckedChange = { visible ->
                              scope.launch { settingsRepo.setTileVisible(tile.id, visible) }
                          },
                      )
                  }
              }

              Text(
                  "Wetterstandort",
                  color = TextSecondary,
                  style = MaterialTheme.typography.titleMedium,
                  modifier = Modifier.padding(top = 16.dp),
              )
              Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
                  OutlinedTextField(
                      value = latText,
                      onValueChange = { latText = it },
                      label = { Text("Breitengrad") },
                      singleLine = true,
                      keyboardOptions = KeyboardOptions(
                          keyboardType = androidx.compose.ui.text.input.KeyboardType.Decimal
                      ),
                      modifier = Modifier.weight(1f),
                  )
                  OutlinedTextField(
                      value = lonText,
                      onValueChange = { lonText = it },
                      label = { Text("Längengrad") },
                      singleLine = true,
                      keyboardOptions = KeyboardOptions(
                          keyboardType = androidx.compose.ui.text.input.KeyboardType.Decimal
                      ),
                      modifier = Modifier.weight(1f),
                  )
              }
              if (locationError) {
                  Text("Ungültige Koordinaten", color = Alert, style = MaterialTheme.typography.bodyMedium)
              }
              Button(
                  onClick = {
                      commitLocation()
                      if (ContextCompat.checkSelfPermission(context, Manifest.permission.ACCESS_FINE_LOCATION)
                          == PackageManager.PERMISSION_GRANTED
                      ) {
                          locate()
                      } else {
                          requestLocationPermission.launch(Manifest.permission.ACCESS_FINE_LOCATION)
                      }
                  },
                  colors = ButtonDefaults.buttonColors(containerColor = Accent),
              ) {
                  Text("Standort ermitteln", color = AppBackground)
              }
              if (locateError) {
                  Text("Standort nicht ermittelt", color = Alert, style = MaterialTheme.typography.bodyMedium)
              }
              TextButton(onClick = onOpenAbout) { Text("Info") }
          }
      }
  }
  ```
  **Done:** file complete (imports from PART 1 + state/helpers from PART 1 + this body).
- [ ] **T10S3** Replace `app/src/main/java/de/florian500/familyinfo/ui/HomeScreen.kt` (adds top bar with gear). Full file:
  ```kotlin
  package de.florian500.familyinfo.ui

  import androidx.compose.foundation.background
  import androidx.compose.foundation.clickable
  import androidx.compose.foundation.layout.Arrangement
  import androidx.compose.foundation.layout.Box
  import androidx.compose.foundation.layout.Column
  import androidx.compose.foundation.layout.Row
  import androidx.compose.foundation.layout.Spacer
  import androidx.compose.foundation.layout.fillMaxHeight
  import androidx.compose.foundation.layout.fillMaxSize
  import androidx.compose.foundation.layout.fillMaxWidth
  import androidx.compose.foundation.layout.padding
  import androidx.compose.material3.Text
  import androidx.compose.runtime.Composable
  import androidx.compose.ui.Alignment
  import androidx.compose.ui.Modifier
  import androidx.compose.ui.unit.dp
  import de.florian500.familyinfo.ui.tiles.FamilyTile
  import de.florian500.familyinfo.ui.tiles.visibleTiles
  import de.florian500.familyinfo.ui.theme.AppBackground
  import de.florian500.familyinfo.ui.theme.TextSecondary

  /**
   * Fixed two-column layout (spec §7.1).
   * col 1: weather (top, 34% weight) + calendar (fills rest, content scrollable)
   * col 2: segelwetter full column height
   * Hidden tiles collapse; remaining tiles in their column expand (weights renormalize).
   * `calendarTile` is injected (needs Context + permission callback, T4S4 wiring note).
   * Future tiles beyond the initial three are appended in a row below col 1 (spec §7.1).
   */
  @Composable
  fun HomeScreen(hidden: Set<String>, calendarTile: FamilyTile?, onOpenSettings: () -> Unit) {
      val visible = visibleTiles(hidden)
      val weather = visible.firstOrNull { it.id == "weather" }
      val segelwetter = visible.firstOrNull { it.id == "segelwetter" }
      val showCalendar = "calendar" !in hidden && calendarTile != null
      val extra = visible.filter { it.id !in setOf("weather", "calendar", "segelwetter") }

      Column(
          modifier = Modifier
              .fillMaxSize()
              .background(AppBackground),
      ) {
          Row(
              modifier = Modifier
                  .fillMaxWidth()
                  .padding(horizontal = 16.dp, vertical = 6.dp),
              verticalAlignment = Alignment.CenterVertically,
          ) {
              Spacer(Modifier.weight(1f))
              Text(
                  text = "⚙",
                  color = TextSecondary,
                  modifier = Modifier
                      .padding(8.dp)
                      .clickable(onClick = onOpenSettings),
              )
          }
          Row(
              modifier = Modifier
                  .weight(1f)
                  .padding(12.dp),
              horizontalArrangement = Arrangement.spacedBy(12.dp),
          ) {
              Column(
                  modifier = Modifier
                      .weight(1f)
                      .fillMaxHeight(),
                  verticalArrangement = Arrangement.spacedBy(12.dp),
              ) {
                  if (weather != null && showCalendar) {
                      Box(modifier = Modifier.weight(0.34f).fillMaxWidth()) { weather.render() }
                      Box(modifier = Modifier.weight(0.66f).fillMaxWidth()) { calendarTile?.render() }
                  } else {
                      weather?.let { Box(modifier = Modifier.weight(1f).fillMaxWidth()) { it.render() } }
                      if (showCalendar) Box(modifier = Modifier.weight(1f).fillMaxWidth()) { calendarTile?.render() }
                  }
                  if (extra.isNotEmpty()) {
                      Row(
                          modifier = Modifier.weight(0.4f).fillMaxWidth(),
                          horizontalArrangement = Arrangement.spacedBy(12.dp),
                      ) {
                          extra.forEach { t -> Box(modifier = Modifier.weight(1f)) { t.render() } }
                      }
                  }
              }
              segelwetter?.let { tile ->
                  Box(modifier = Modifier.weight(1f).fillMaxHeight()) { tile.render() }
              }
          }
      }
  }
  ```
  **Done:** file exactly matches.
- [ ] **T10S4** Replace `app/src/main/java/de/florian500/familyinfo/MainActivity.kt` (adds screen switching — near-final: T11S2 adds the About entry after this). Full file:
  ```kotlin
  package de.florian500.familyinfo

  import android.Manifest
  import android.content.SharedPreferences
  import android.content.pm.ActivityInfo
  import android.content.pm.PackageManager
  import android.os.Bundle
  import androidx.activity.ComponentActivity
  import androidx.activity.compose.setContent
  import androidx.activity.result.contract.ActivityResultContracts
  import androidx.compose.runtime.LaunchedEffect
  import androidx.compose.runtime.getValue
  import androidx.compose.runtime.mutableStateOf
  import androidx.compose.runtime.remember
  import androidx.compose.runtime.setValue
  import androidx.core.content.ContextCompat
  import androidx.lifecycle.compose.collectAsStateWithLifecycle
  import androidx.lifecycle.lifecycleScope
  import de.florian500.familyinfo.data.CalendarRepository
  import de.florian500.familyinfo.data.SettingsRepository
  import de.florian500.familyinfo.ui.HomeScreen
  import de.florian500.familyinfo.ui.SettingsScreen
  import de.florian500.familyinfo.ui.tiles.CalendarTile
  import de.florian500.familyinfo.ui.tiles.TileRegistry
  import de.florian500.familyinfo.ui.theme.FamilyInfoTheme
  import kotlinx.coroutines.flow.MutableStateFlow

  enum class Screen { HOME, SETTINGS }

  class MainActivity : ComponentActivity() {

      private val requestCalendarPermission =
          registerForActivityResult(ActivityResultContracts.RequestPermission()) {
              refreshPermissionState()
          }

      private lateinit var permissionGranted: MutableStateFlow<Boolean>

      private fun refreshPermissionState() {
          permissionGranted.value = ContextCompat.checkSelfPermission(
              this, Manifest.permission.READ_CALENDAR,
          ) == PackageManager.PERMISSION_GRANTED
      }

      override fun onCreate(savedInstanceState: Bundle?) {
          super.onCreate(savedInstanceState)
          requestedOrientation = ActivityInfo.SCREEN_ORIENTATION_SENSOR_LANDSCAPE
          val settingsRepo = SettingsRepository.create(this)
          val calendarRepo = CalendarRepository(this)

          permissionGranted = MutableStateFlow(ContextCompat.checkSelfPermission(
              this, Manifest.permission.READ_CALENDAR,
          ) == PackageManager.PERMISSION_GRANTED)
          window.addFlags(android.view.WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON)
          getSharedPreferences(packageName + "_preferences", MODE_PRIVATE)
              .registerOnSharedPreferenceChangeListener(
                  SharedPreferences.OnSharedPreferenceChangeListener { _, _ -> refreshPermissionState() },
              )

          val calendarTile = CalendarTile(
              repo = calendarRepo,
              permissionGranted = permissionGranted,
              onRequestPermission = { requestCalendarPermission.launch(Manifest.permission.READ_CALENDAR) },
          )

          setContent {
              FamilyInfoTheme {
                  var screen by remember { mutableStateOf(Screen.HOME) }
                  val hidden by settingsRepo.hiddenTileIds.collectAsStateWithLifecycle()
                  val location by settingsRepo.location.collectAsStateWithLifecycle()
                  LaunchedEffect(location) {
                      TileRegistry.weatherController.start(lifecycleScope, location)
                  }
                  when (screen) {
                      Screen.HOME -> HomeScreen(hidden = hidden, calendarTile = calendarTile) {
                          screen = Screen.SETTINGS
                      }
                      Screen.SETTINGS -> SettingsScreen(
                          settingsRepo = settingsRepo,
                          calendarTile = calendarTile,
                          onBack = { screen = Screen.HOME },
                          onOpenAbout = {},   // wired in T11S2
                      )
                  }
              }
          }
      }
  }
  ```
  **Done:** file exactly matches. (`SENSOR_LANDSCAPE` + `FLAG_KEEP_SCREEN_ON` are set here, so no manifest orientation/flags change is needed in Task 12 — T12S4 documents why.)

- [ ] **T10S5** Build:
  ```bash
  cd /Users/lindner/VSCode/FamilyInfoDisplay && ./gradlew :app:assembleDebug 2>&1 | tail -40
  ```
  **Done:** `BUILD SUCCESSFUL`. On compile error: `lifecycleScope` unresolved → the T10S4 import block is missing `androidx.lifecycle.lifecycleScope` (already in the plan — re-diff).

- [ ] **T10S6** Device verification on `HA2EN2VE`:
  ```bash
  adb devices
  adb -s HA2EN2VE install -r /Users/lindner/VSCode/FamilyInfoDisplay/app/build/outputs/apk/debug/app-debug.apk
  adb -s HA2EN2VE shell am start -n de.florian500.familyinfo/.MainActivity
  adb -s HA2EN2VE shell am force-stop de.florian500.familyinfo && adb -s HA2EN2VE shell am start -n de.florian500.familyinfo/.MainActivity
  ```
  **Done (all of it, in order):**
  1. Gear "⚙" (top right) opens Settings; "←" returns to Home.
  2. Toggling a tile switch updates the home screen immediately after "←" (hidden tile collapsed, neighbor expanded).
  3. Manual lat/lon entry (e.g. 52.44 / 13.67) commits (error text disappears, weather tile uses it).
  4. Invalid input (e.g. "abc") → "Ungültige Koordinaten" in alert color.
  5. "Standort ermitteln" → permission prompt (first time) → on grant: fields filled with device coordinates, saved.
  6. Force-stop + relaunch (commands above) → tile visibility AND location both persist (DataStore).
  **Done:** all six observed; no crash.

- [ ] **T10S7** Commit:
  ```bash
  cd /Users/lindner/VSCode/FamilyInfoDisplay && git add -A && git commit -m "feat: settings screen for tiles and weather location"
  ```
  **Done:** `git status --porcelain` empty (the plan file's own edits are committed here too — expected). **→ Current Position → T11S1, Remaining = 2.**

## Task 11: About screen + license files (spec §7.7, §12)

**Task done when:** build green; Settings → "Info" opens About (app name + version, one-line purpose, Beerware + ko-fi, third-party list); About → "←" returns to Settings; `LICENSE.md` + `THIRD_PARTY_LICENSES.md` in repo root; committed.

- [ ] **T11S1** Create `app/src/main/java/de/florian500/familyinfo/ui/AboutScreen.kt`, exactly (two code blocks below form ONE file — paste part A then part B, no blank line between the closing brace of the Row and the scrollable Column):
  ```kotlin
  package de.florian500.familyinfo.ui

  import android.content.Context
  import androidx.compose.foundation.background
  import androidx.compose.foundation.clickable
  import androidx.compose.foundation.layout.Arrangement
  import androidx.compose.foundation.layout.Column
  import androidx.compose.foundation.layout.Row
  import androidx.compose.foundation.layout.fillMaxSize
  import androidx.compose.foundation.layout.fillMaxWidth
  import androidx.compose.foundation.layout.padding
  import androidx.compose.foundation.rememberScrollState
  import androidx.compose.foundation.verticalScroll
  import androidx.compose.material3.MaterialTheme
  import androidx.compose.material3.Text
  import androidx.compose.runtime.Composable
  import androidx.compose.ui.Alignment
  import androidx.compose.ui.Modifier
  import androidx.compose.ui.platform.LocalContext
  import androidx.compose.ui.unit.dp
  import de.florian500.familyinfo.ui.theme.Accent
  import de.florian500.familyinfo.ui.theme.PanelBackground
  import de.florian500.familyinfo.ui.theme.TextPrimary
  import de.florian500.familyinfo.ui.theme.TextSecondary

  /** Third-party resources (spec §7.7) — full license texts live in THIRD_PARTY_LICENSES.md (repo root). */
  private data class ThirdPartyLicense(val name: String, val license: String, val note: String)

  private val thirdPartyLicenses = listOf(
      ThirdPartyLicense(
          name = "Inter",
          license = "SIL Open Font License 1.1 — https://openfontlicense.org/",
          note = "Font (Rasmus Andersson), bundled in res/font/inter.ttf",
      ),
      ThirdPartyLicense(
          name = "Open-Meteo",
          license = "CC-BY 4.0 (data) — https://open-meteo.com/en/disclaimer",
          note = "Weather API; free for non-commercial use",
      ),
      ThirdPartyLicense(
          name = "Segelwetter page (florian500)",
          license = "Beerware — https://ko-fi.com/florian500",
          note = "Embedded by URL only; the page itself also uses OSM contributors (ODbL), Leaflet (BSD-2-Clause), EOT20 tide model (DGFI/TUM)",
      ),
  )

  @Composable
  fun AboutScreen(onBack: () -> Unit) {
      val context = LocalContext.current
      val versionName = try {
          context.packageManager.getPackageInfo(context.packageName, 0).versionName ?: "?"
      } catch (e: Exception) {
          "?"
      }

      Column(
          modifier = Modifier
              .fillMaxSize()
              .background(PanelBackground)
              .padding(20.dp),
      ) {
          Row(verticalAlignment = Alignment.CenterVertically) {
              Text(
                  text = "←",
                  color = Accent,
                  style = MaterialTheme.typography.headlineSmall,
                  modifier = Modifier
                      .padding(8.dp)
                      .clickable(onClick = onBack),
              )
              Text(
                  text = "Info",
                  color = TextPrimary,
                  style = MaterialTheme.typography.headlineSmall,
                  modifier = Modifier.padding(start = 8.dp),
              )
          }
          Column(
              modifier = Modifier
                  .fillMaxSize()
                  .verticalScroll(rememberScrollState())
                  .padding(top = 20.dp),
              verticalArrangement = Arrangement.spacedBy(10.dp),
          ) {
              Text("FamilyInfo $versionName", color = TextPrimary, style = MaterialTheme.typography.titleLarge)
              Text("Anzeigetafel für die Familie: Wetter, Termine und Segelwetter auf einen Blick.", color = TextSecondary, style = MaterialTheme.typography.bodyMedium)
              Text("Lizenz", color = TextSecondary, style = MaterialTheme.typography.titleMedium)
              Text("Beerware (MIT-ähnlich): Wenn dir diese App gefällt, spendiere mir einen Kaffee: https://ko-fi.com/florian500", color = TextPrimary, style = MaterialTheme.typography.bodyMedium)
              Text("Drittanbieter", color = TextSecondary, style = MaterialTheme.typography.titleMedium)
              thirdPartyLicenses.forEach { tp ->
                  Text(tp.name, color = TextPrimary, style = MaterialTheme.typography.titleSmall)
                  Text(tp.license, color = TextSecondary, style = MaterialTheme.typography.bodySmall)
                  Text(tp.note, color = TextSecondary, style = MaterialTheme.typography.bodySmall)
              }
              Text(
                  text = "Vollständige Lizenztexte: THIRD_PARTY_LICENSES.md (Repository)",
                  color = TextSecondary,
                  style = MaterialTheme.typography.bodySmall,
                  modifier = Modifier.padding(top = 8.dp),
              )
          }
      }
  }
  ```
  **Done:** file exists, exactly one `@Composable` fun, no duplicate braces. Cross-check the theme token names against `theme/Color.kt` (T2) — if any of `Accent`/`PanelBackground`/`TextPrimary`/`TextSecondary` differ, use the Color.kt names (drop the `AppBackground` import if unused). Ruling: links are plain text, not clickable — spec §7.7 asks for "the short list and links"; a display kiosk has no browser. Full license texts stay out of the APK per spec §7.7 (one line per resource in-app).
- [ ] **T11S2** Edit `app/src/main/java/de/florian500/familyinfo/MainActivity.kt` (the file written in T10S4) — three changes:
  1. Import (insert `AboutScreen` before `HomeScreen`):
     ```kotlin
     before:
       import de.florian500.familyinfo.ui.HomeScreen
       import de.florian500.familyinfo.ui.SettingsScreen
     after:
       import de.florian500.familyinfo.ui.AboutScreen
       import de.florian500.familyinfo.ui.HomeScreen
       import de.florian500.familyinfo.ui.SettingsScreen
     ```
  2. Enum (ABOUT added):
     ```kotlin
     before:  enum class Screen { HOME, SETTINGS }
     after:   enum class Screen { HOME, SETTINGS, ABOUT }
     ```
  3. Callback + branch (replace the whole block):
     ```kotlin
     before:
                           onOpenAbout = {},   // wired in T11S2
                       )
                   }
     after:
                           onOpenAbout = { screen = Screen.ABOUT },
                       )
                       Screen.ABOUT -> AboutScreen(onBack = { screen = Screen.SETTINGS })
                   }
     ```
  **Done:** exactly these three diffs applied; `onOpenAbout = {},` no longer exists in the file.

- [ ] **T11S3** Create the two repo-root license files. `LICENSE.md`, exactly:
  ```md
  # FamilyInfo — App License

  Beerware License:

  Wenn dir diese App gefällt, spendiere mir einen Kaffee: https://ko-fi.com/florian500

  Permission is hereby granted, free of charge, to any person obtaining a copy
  of this software and associated documentation files (the "Software"), to deal
  in the Software without restriction, including without limitation the rights
  to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
  copies of the Software, and to permit persons to whom the Software is
  furnished to do so, subject to the following condition:

  The above copyright notice and this permission notice shall be included in all
  copies or substantial portions of the Software.

  THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
  IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
  FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
  AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
  LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
  OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
  SOFTWARE.
  ```
  And `THIRD_PARTY_LICENSES.md`, exactly:
  ```md
  # Third-Party Licenses — FamilyInfo

  Single source of truth for third-party resources used by this app (spec §7.7).
  The about screen shows this list in short form.

  ## Inter (font)
  - Copyright: Rasmus Andersson
  - License: SIL Open Font License 1.1 — https://openfontlicense.org/
  - Where: bundled in `app/src/main/res/font/` (inter_regular.ttf, inter_medium.ttf, inter_bold.ttf)

  ## Open-Meteo (weather API)
  - License: Creative Commons Attribution 4.0 for data products — https://open-meteo.com/en/disclaimer
  - Terms: free for non-commercial use; attribution required
  - Where: `OpenMeteoClient.kt` — network requests only, no bundled data

  ## Segelwetter page (florian500)
  - License: Beerware (kaffee: https://ko-fi.com/florian500)
  - Where: embedded by URL only (`SegelwetterTile.kt` WebView loads https://superfloh247.github.io/Segelwetter/)
  - The page itself additionally states:
    - OpenStreetMap contributors — ODbL
    - Leaflet — BSD-2-Clause
    - Open-Meteo terms
    - EOT20 tide model — DGFI/TUM
  - These apply to the page, not to this app (we only embed by URL).
  ```
  **Done:** both files exist at repo root; `grep -c 'ko-fi.com/florian500' LICENSE.md THIRD_PARTY_LICENSES.md` prints 1 for each.
- [ ] **T11S4** Build + device verification on `HA2EN2VE`:
  ```bash
  cd /Users/lindner/VSCode/FamilyInfoDisplay && ./gradlew :app:assembleDebug 2>&1 | tail -40
  adb devices
  adb -s HA2EN2VE install -r /Users/lindner/VSCode/FamilyInfoDisplay/app/build/outputs/apk/debug/app-debug.apk
  adb -s HA2EN2VE shell am start -n de.florian500.familyinfo/.MainActivity
  ```
  **Done (all of it):**
  1. `BUILD SUCCESSFUL`.
  2. Home → gear → Settings → "Info" opens About.
  3. About shows: "FamilyInfo 0.1.0", the one-line purpose, the Beerware/ko-fi line, the three third-party entries (Inter / OFL, Open-Meteo / CC-BY 4.0, Segelwetter / Beerware + sub-list), and the "Vollständige Lizenztexte" hint.
  4. "←" in About returns to Settings (NOT to Home).
  5. Scroll the About content to the bottom — no clipping.
  **On failure:** theme token compile error → diff imports against T2 `Color.kt`; About not rendering → T11S2 diff 3 not applied.

- [ ] **T11S5** Commit:
  ```bash
  cd /Users/lindner/VSCode/FamilyInfoDisplay && git add -A && git commit -m "feat: about screen with license and third-party list"
  ```
  **Done:** `git status --porcelain` empty. **→ Current Position → T12S1, Remaining = 1.**
---

## Task 12: Icon + final verification (spec §13)

**Task done when:** proper launcher icon installed and visible on the tablet, full final checklist verified, unit tests green, git tree clean, final commit done.

- [ ] **T12S1** Replace the T1S9 placeholder icon with the proper launcher icon — SAME file path, so the manifest's `android:icon="@mipmap/ic_launcher"` reference stays unchanged:
  ```bash
  cd /Users/lindner/VSCode/FamilyInfoDisplay
  python3 - <<'EOF'
  import zlib,struct
  W=H=512; cx=cy=256
  BG=b'\x11\x12\x17'      # app background
  ACC=b'\xf4\x68\x00'     # accent orange
  rows=[]
  for y in range(H):
      row=bytearray()
      for x in range(W):
          d=((x-cx)**2+(y-cy)**2)**0.5
          if d<=190 and not ((244<=x<=268 and 84<=y<=256) or (256<=x<=404 and 244<=y<=268) or d<=26):
              row += ACC
          else:
              row += BG
      rows.append(b'\x00'+bytes(row))
  raw=b''.join(rows)
  def chunk(t,d):
      c=t+d; return struct.pack('>I',len(d))+c+struct.pack('>I',zlib.crc32(c)&0xffffffff)
  png=b'\x89PNG\r\n\x1a\n'+chunk(b'IHDR',struct.pack('>IIBBBBB',W,H,8,2,0,0,0))+chunk(b'IDAT',zlib.compress(raw))+chunk(b'IEND',b'')
  open('app/src/main/res/mipmap-mdpi/ic_launcher.png','wb').write(png)
  EOF
  file app/src/main/res/mipmap-mdpi/ic_launcher.png
  ```
  **Done:** `file` prints "PNG image data, 512 x 512". The icon is a dark (#111217) square with an orange (#F46800) clock disc, dark hands at 3 o'clock, dark center dot.
- [ ] **T12S2** Rebuild + install with the new icon:
  ```bash
  cd /Users/lindner/VSCode/FamilyInfoDisplay && ./gradlew :app:assembleDebug 2>&1 | tail -20
  adb -s HA2EN2VE install -r /Users/lindner/VSCode/FamilyInfoDisplay/app/build/outputs/apk/debug/app-debug.apk
  ```
  **Done:** `BUILD SUCCESSFUL` + install OK. On the tablet home screen the launcher icon is the new dark/orange clock (force-refresh if it still shows the old placeholder: `adb -s HA2EN2VE shell am force-stop de.florian500.familyinfo`, then reopen the launcher).

- [ ] **T12S3** Final end-to-end verification on `HA2EN2VE` — run the app, then walk EVERY item below (it is the full app surface; one pass):
  1. **Flags/persistence (kill + relaunch):** `adb -s HA2EN2VE shell am force-stop de.florian500.familyinfo`, relaunch — settings changes from earlier tasks still hold (hidden tiles stay hidden, location still set).
  2. **Settings persistence:** toggle a setting in Settings → kill → relaunch → it persisted.
  3. **Tile persistence:** hide one tile via long-press → kill → relaunch → still hidden.
  4. **Location persistence:** location value survives kill → relaunch.
  5. **Landscape/sensor:** rotate the tablet — stays in landscape (sensor-lock works; no portrait flip).
  6. **Keep-screen-on:** leave the app idle for >5 min — screen stays on.
  7. **Weather:** weather tile shows current conditions / updates (or graceful placeholder if no data).
  8. **Calendar:** calendar tile shows the next upcoming event(s).
  9. **Segelwetter:** segelwetter tile loads the webview content.
  10. **Settings screen:** all sections render, no clipping, all toggles work.
  11. **App icon:** launcher icon is the proper dark/orange clock (not the old dark placeholder).
  **Done:** all 11 items pass. **On failure:** re-run only the failed item after fixing the cause; record a `Ruling:` if a deviation is needed.
- [ ] **T12S4** Document in the Execution Log WHY no manifest change is needed in Task 12 (this step is a no-op on the codebase — its job is to prove it):
  ```bash
  cd /Users/lindner/VSCode/FamilyInfoDisplay && grep -n 'android:screenOrientation\|android:keepScreenOn' app/src/main/AndroidManifest.xml; echo "exit=$?"
  ```
  **Done:** grep finds nothing (exit=1) — orientation is locked via `UIManager.setOrientationLocked(SensorManager.SENSOR_LANDSCAPE)` and keep-screen-on via `window.addFlags(FLAG_KEEP_SCREEN_ON)`, both set in `MainActivity.kt` (T10S4). Append this Execution Log line:
  `2026-11-25  T12S4 done — manifest unchanged by design: SENSOR_LANDSCAPE + FLAG_KEEP_SCREEN_ON set in code (MainActivity), no manifest flags needed`

- [ ] **T12S5** Final unit test run + git cleanup + final commit:
  ```bash
  cd /Users/lindner/VSCode/FamilyInfoDisplay && ./gradlew :app:testDebugUnitTest 2>&1 | tail -15
  git status --porcelain
  git add -A && git commit -m "feat: proper launcher icon + final verification"
  ```
  **Done (all of it):**
  1. All unit tests pass (no `FAILED` in output).
  2. `git status --porcelain` empty AFTER the commit (before: only the icon file + this plan file).
  3. `git log --oneline -3` shows the final commit on top of T11's commit.
  **Done:** → PLAN COMPLETE. Uncheck every remaining box, set the Execution Log final line:
  `2026-11-25  T12S5 done — all tests green, tree clean, plan complete`

<!--NEXT-->
