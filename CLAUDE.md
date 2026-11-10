# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

**See also [AGENTS.md](AGENTS.md)** for coding conventions, naming patterns, and step-by-step checklists for adding screens, database tables, and components.

## Project Overview

Rivo is a native Android dialer and contacts app built with Kotlin and Jetpack Compose. Package: `com.grinch.rivo4`. License: GPLv3.

## Build Commands

```bash
# Play Store build (with ads & billing)
./gradlew assemblePlayRelease

# Play Store bundle
./gradlew bundlePlayRelease

# F-Droid / de-googled build
./gradlew assembleFossRelease -PdisableDependencyInfo

# Debug build
./gradlew assemblePlayDebug

# Run unit tests
./gradlew test

# Run a single test class
./gradlew testPlayDebugUnitTest --tests "com.grinch.rivo4.ContactUtilsTest"

# Lint
./gradlew lint
```

Release signing requires env vars: `KEYSTORE_PATH`, `KEYSTORE_PASSWORD`, `KEY_ALIAS`, `KEY_PASSWORD`.

## Build Flavors

- **play** — Google Play version with ads (`play-services-ads`) and billing (`billing-ktx`). Default.
- **foss** — De-googled version for F-Droid. minSdk=29. Output: `RivoPhone_degoogled.apk`.

## Architecture

MVVM pattern with Koin dependency injection.

```
app/src/main/java/com/grinch/rivo4/
├── view/                    # UI layer (Jetpack Compose)
│   ├── screen/              # Full screens (Dialpad, Recents, Contact, CallScreen, settings/, onboarding/)
│   ├── components/          # Reusable Compose components
│   └── theme/               # Material 3 / Material You theming
├── modal/                   # Data layer
│   ├── data/                # Data classes (Contact, CallLog, etc.)
│   ├── db/                  # Room database, entities, DAOs
│   ├── interface/           # Repository interfaces (IContactsRepository, ICallLogRepository)
│   └── repository/          # Repository implementations
├── controller/              # Business logic & services
│   ├── *ViewModel.kt        # ViewModels (Contacts, CallLog, Backup, CallAnalytics)
│   ├── CallService.kt       # InCallService — main call handling
│   ├── CallActivity.kt      # In-call screen activity
│   ├── util/                # Utilities (PreferenceManager, ContactUtils, etc.)
│   ├── shizuku/             # Shizuku ADB integration for call recording
│   ├── fakecall/            # Fake call feature
│   ├── lock/                # Biometric lock
│   ├── floating/            # Floating call window service
│   └── reminder/            # Callback reminders
├── ads/                     # Ad integration (play flavor)
├── billing/                 # In-app billing (play flavor)
├── RivoApp.kt               # Application class (Koin init, Coil image loader)
├── AppModule.kt             # Koin DI module definitions
└── MainActivity.kt          # Entry point, Compose host
```

## Key Technical Details

- **Kotlin 2.1.0**, JVM target 17, AGP 8.13.2
- **Compose BOM 2025.12.01**, Material 3 1.5.0-alpha18
- **Navigation**: compose-destinations (KSP code-gen, type-safe routes)
- **Database**: Room v2.6.1 — version 6 with migrations in `RivoDatabase.ALL_MIGRATIONS`
- **Image loading**: Coil 2.7.0
- **Serialization**: kotlinx-serialization-json
- **Permissions**: Accompanist Permissions 0.37.3
- **Call recording**: Shizuku 13.1.5 (IPC via `IShellService.aidl`)
- **ProGuard** enabled for release builds; rules in `app/proguard-rules.pro`
- **Gradle build dir** defaults to `~/.gradle-builds/Rivo4/` (external to project); CI uses `app/build`

## Feature: Caller ID Override (#31# CID)

Two independent settings in **Call Accounts** screen allow per-call CLIR override via `#31#` prefix:

- **Show Caller ID for Contacts** (`KEY_FORCE_CID_CONTACTS`) — prepends `#31#` when calling a saved phonebook contact (`contactId != null`)
- **Show Caller ID for Callbacks** (`KEY_FORCE_CID_INCOMING_CALLBACK`) — prepends `#31#` when returning an incoming/missed call from call history, MissedCallScreen, or PostCallScreen

Guards: emergency numbers and voicemail are always excluded.

### Call flow integration

- Core logic lives in `makeCall()` (`controller/util/utils.kt`) — evaluates `contactId` and `isIncomingCallback` parameters against preference state
- `isIncomingCallback` is determined at the call site: `CallLog.Calls.INCOMING_TYPE` / `MISSED_TYPE` in Recents/CallLogs, always `true` in MissedCallScreen, threaded via `wasIncoming` Intent extra through PostCallActivity
- `CallLauncher` threads `isIncomingCallback` through async flows (permission requests, SIM picker, number picker) via `pendingIsIncomingCallback` state
- URI construction uses `Uri.fromParts("tel", dialString, null)` to safely encode `#` as `%23` — `TelecomManager.placeCall()` reads `getSchemeSpecificPart()` which decodes correctly

### Carrier dependency

`#31#` is a GSM supplementary service code (3GPP TS 24.082). Behavior is carrier-dependent and best-effort. CDMA networks use `*82` instead (not implemented). Settings wording uses "Try to send" to reflect this.

## Testing

JUnit 4 tests in `app/src/test/`. `unitTests.returnDefaultValues = true` in Gradle config so Android framework classes return defaults in unit tests.

## Localization

English strings in `app/src/main/res/values/strings.xml`. Translations managed via Crowdin with auto-sync GitHub Actions workflow. Locale config auto-generated.

## CI/CD

GitHub Actions (`.github/workflows/build.yaml`):
- Triggers on push to `main`, version tags (`v*`), PRs to `main`
- Builds both play and foss flavors
- Preview releases published to `preview` tag on main pushes
- Production releases on version tags
