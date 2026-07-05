# AI WhatsApp Smart Assistant — Android App

This is the **Android client** of AI WhatsApp Smart Assistant. It captures WhatsApp
notifications, classifies them locally and via a backend AI service, and can
draft or auto-send context-aware replies through WhatsApp's own quick-reply
mechanism.

This module is a **working architectural scaffold**, not a store-ready
finished app. It's built so every core flow (notification capture → local
classification → rule evaluation → AI reply → send) is real, wired, and
testable end-to-end, while several "nice to have" screens from the original
spec are intentionally left as clearly-marked TODOs so the codebase stays
honest about what's implemented vs. stubbed. See **"What's fully implemented
vs. stubbed"** below.

---

## ⚠️ Read this before you build

1. **WhatsApp Terms of Service.** WhatsApp's ToS prohibits automated /
   bulk messaging through unofficial clients. This app avoids the highest-risk
   pattern (Accessibility Service tap-simulation or WhatsApp Web/API scraping)
   by only ever sending replies through the **RemoteInput action WhatsApp
   itself attaches to its own notification** — the same channel used by the
   system "quick reply" button. That said, *any* automated reply behavior is
   still automation from WhatsApp's perspective and carries some risk of
   account action. Don't distribute this pointed at accounts you're not
   prepared to lose.
2. **Consent & wiretapping-style laws.** Auto-replying to people without
   disclosure can implicate consent/recording laws in some jurisdictions. A
   disclosure line is included by default in `AiConfig.DEFAULT_SYSTEM_PROMPT`
   — don't remove it without understanding your local law.
3. **Never disable the OTP/scam hard-block.** `EvaluateAutoReplyRulesUseCase`
   refuses to auto-reply to anything flagged as an OTP or suspected scam,
   *before* any user-configurable rule is even consulted. Treat this as a
   non-negotiable safety invariant, not a setting.

---

## Architecture

Clean Architecture + MVVM, single-activity Compose UI:

```
ui/            Compose screens + ViewModels (Dashboard, Messages, Settings, Onboarding)
domain/        Pure Kotlin models + use cases (no Android/Room/Retrofit deps)
data/
  local/       Room entities/DAOs + SQLCipher-encrypted AppDatabase
  remote/      Retrofit ApiService + DTOs (talks to the Spring Boot backend)
  security/    EncryptedSharedPreferences wrapper (API keys, JWT, DB passphrase, PIN)
  repository/  MessageRepository - single source of truth, the only thing ViewModels/Workers touch
service/       WhatsAppNotificationListenerService + FCM push service
worker/        SmartReplyWorker (WorkManager) - the orchestration point per message
di/            Hilt modules (NetworkModule, AppModule)
```

**Data flow for one incoming WhatsApp message:**

```
WhatsApp posts notification
        │
        ▼
WhatsAppNotificationListenerService.onNotificationPosted()
  - filters to com.whatsapp / com.whatsapp.w4b only
  - extracts sender, text, group flag, media flag
  - ignores empty/"deleted message" placeholder updates
  - runs LocalMessageHeuristics (OTP/spam/scam/urgency) fully offline
  - stashes the notification's RemoteInput reply Action in
    PendingReplyActionRegistry (in-memory bridge — Actions/PendingIntents
    aren't WorkManager-Data-serializable)
  - enqueues SmartReplyWorker with everything else as WorkManager Data
        │
        ▼
SmartReplyWorker.doWork()
  1. Persist to encrypted Room DB immediately
  2. Hard-block OTP / suspected scam -> status = SUPPRESSED, stop
  3. EvaluateAutoReplyRulesUseCase checks contact scope, work hours,
     driving/vacation/sleep mode, system DND
  4. If allowed: call backend /api/v1/ai/smart-reply for a styled,
     language-matched reply
  5. If a fresh RemoteInput action is available, send it exactly the way
     the system "quick reply" button would; otherwise store as
     AI_SUGGESTED for the user to review/send from the Messages screen
```

## What's fully implemented vs. stubbed

**Fully implemented, real, wired:**
- NotificationListenerService capture (sender/text/timestamp/group/media/deleted-message filtering)
- Local, offline-first OTP/spam/scam/greeting/urgency heuristics + unit tests
- Local script-based language detection for en/hi/kn/ta/te/ml (+ backend does authoritative detection)
- Encrypted Room DB via SQLCipher, phone numbers stored only as salted hashes
- EncryptedSharedPreferences for API keys / JWT / PIN hash / DB passphrase
- Auto-reply rule engine with OTP/scam hard-block + unit tests (MockK/Turbine-ready)
- WorkManager orchestration with Hilt injection (`@HiltWorker`)
- RemoteInput-based reply sending bridge (`PendingReplyActionRegistry`)
- Retrofit/Hilt networking layer with Bearer-token auth interceptor
- Compose UI: Onboarding (explains exactly what the app reads), Dashboard,
  Message list w/ search + filter chips, Settings (AI config, API key entry,
  PIN, biometric toggle, theme)

**Stubbed / TODO (clearly marked in code, straightforward to fill in):**
- Driving mode / vacation mode / sleep mode detection (`SmartReplyWorker` passes `false` — wire to Android Auto state, a scheduled window, or a manual toggle)
- Per-contact reply-style preference and conversation history lookback (currently always `ReplyStyle.FRIENDLY`, empty history)
- Learning engine persistence (`LearnedReplyEntity`/`LearnedReplyDao` exist; nothing writes to them yet from the UI "edit before send" flow)
- Biometric prompt UI (the `biometric` dependency and preference toggle exist; the actual `BiometricPrompt` invocation is not wired to app-launch)
- FCM token registration call to backend
- Backup/restore, export chat, import settings screens
- Contact-picker UI for `RuleScope.SELECTED_CONTACTS`
- App launcher icon is a placeholder vector — replace via Android Studio's Image Asset tool before shipping
- `gradle-wrapper.jar` binary is not included (see below)

## Building an APK via GitHub Actions (no computer needed)

This repo includes `.github/workflows/build-apk.yml`, which builds a debug
APK automatically on every push to `main` using GitHub's own build servers —
useful if you don't have Android Studio installed anywhere and just want an
installable APK from your phone's browser.

1. Push/upload this project to a GitHub repo (via `git push` or the web
   "Upload files" UI).
2. Go to the repo's **Actions** tab → you should see "Build Debug APK" running
   (or trigger it manually with the "Run workflow" button).
3. When it finishes, open the run → scroll to **Artifacts** → download
   `app-debug-apk` (a zip containing `app-debug.apk`).
4. Unzip it (most phone file managers can do this), then open the `.apk` file
   directly to install. Android will ask you to allow "install from this
   source" the first time — approve it just for this file.

**Important caveats about this specific build path:**
- It builds via `gradle assembleDebug` directly (not `./gradlew`) because the
  binary `gradle-wrapper.jar` isn't committed to this repo — see "Building"
  above for why. Functionally identical output either way.
- Firebase (push notifications, Crashlytics, Analytics) is skipped in this
  build since no real `google-services.json` is present — the plugin only
  applies if that file exists (see `app/build.gradle.kts`). The core
  notification-capture → classify → auto-reply flow works fine without it;
  only FCM push and crash reporting are inactive until you add your own
  Firebase project config.
- This produces a **debug** build — fine for personal use/testing, but debug
  builds are unsigned for release and shouldn't be distributed publicly.

**Gradle wrapper jar:** this environment has no network access to
`services.gradle.org`, so `gradle/wrapper/gradle-wrapper.jar` could not be
downloaded here. `gradle-wrapper.properties` already points at Gradle 8.7.
Before building, run once (with your own machine's internet access):

```bash
gradle wrapper --gradle-version 8.7
```

or simply open the project in a recent Android Studio (Koala/2024.1+), which
will offer to generate the wrapper automatically.

**Then:**

```bash
./gradlew assembleDebug      # debug APK -> app/build/outputs/apk/debug/
./gradlew assembleRelease    # release APK (needs a signing config - see below)
./gradlew testDebugUnitTest  # run the unit tests included in this scaffold
```

**Signing a release build:** add a `signingConfigs` block in
`app/build.gradle.kts` pointing at your keystore, or configure it via
`~/.gradle/gradle.properties` (`RELEASE_STORE_FILE`, `RELEASE_KEY_ALIAS`, etc.)
— never commit keystore passwords to source control.

**Backend URL:** set in `app/build.gradle.kts` via `buildConfigField("String", "API_BASE_URL", ...)`
per build type. Point this at your deployed Spring Boot backend (see
`/backend/README.md`).

## Required manual setup after cloning

1. Add your own `google-services.json` (Firebase Console → Project settings →
   your Android app) into `app/` for FCM/Crashlytics to work.
2. Generate the Gradle wrapper jar as noted above.
3. Replace the placeholder launcher icon.
4. Configure `API_BASE_URL` for your deployed backend.
