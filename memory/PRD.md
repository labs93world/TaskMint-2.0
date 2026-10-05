# TaskMint — Product Requirements Document

## Original Problem Statement
Production-ready Android rewards app "TaskMint" (package com.labs93world.taskmint). Login → 3 bottom tabs (Home, Wallet, Profile). Offerwall tasks + Play & Earn mini-games, wallet withdrawals (UPI/Bank), hidden admin panel, AdMob ads. Offline-first (personal data on device), online-only usage, global admin data + user requests on Firestore, IST timestamps, polished responsive UI.

## Architecture
- **Frontend:** Expo Router (React Native, SDK 57), file-based routing.
- **Global store:** Firebase Firestore (JS SDK) — project `taskmint-cfc5a` (USER CHOICE: keep ALL data on Firestore, no Mongo migration).
- **Local store:** on-device via `@/src/utils/storage` (AsyncStorage) — profile, points, transactions, game chances, daily check-in, notification dedup sets.
- **Notifications:** 100% local (expo-notifications) — no FCM sender / no Cloud Functions / no Emergent server, so it stays free. Daily reminder is a scheduled local notification; withdrawal-approved / withdrawal-rejected / new-task are local notifications raised on Firestore sync (foreground reconcile reuses existing reads + a periodic background task). `google-services.json` wires the native FCM channel for the APK build.
- **Ads:** `react-native-google-mobile-ads` (banner, app open, rewarded, rewarded interstitial) — native build only; no-op in Expo Go/web.
- **Theme:** `src/theme.ts` tokens (light).
- FastAPI/Mongo backend present but unused by the app.

## User Personas
- **Earner:** completes offerwall tasks + games, withdraws to UPI/Bank.
- **Admin:** manages banners/tasks, approves/rejects payouts & submissions, force-update.

## Core Requirements (static)
- 1000 points = ₹1; min withdrawal ₹20.
- Store user data on device; only admin content + requests on Firestore.
- Internet required (NetworkGate). IST date/time everywhere.
- Minimal Firestore reads/writes (free tier) — all notification logic reuses existing reads.

## Implemented
### 2026-06 (initial)
- Login/onboarding, Home (header, banner carousel, Offerwall/Play&Earn), offerwall tasks + proof submissions, Play & Earn (daily check-in + 5 games + chances/rewarded ads), Wallet (UPI/Bank withdraw, transactions/withdrawals history), Profile + hidden admin (key `TaskMint000`), Admin dashboard/managers/review, AdMob wiring, Terms/Privacy, ForceUpdate + Offline gates.

### 2026-10-05 (this revision)
- Imported from GitHub repo into /app.
- **Home:** vertical layout optimized (tighter header/body/segment spacing, nothing cropped/overlapped); header title/subtext/points gap balanced; offerwall order = fresh(submission-on → submission-off) → pending → approved → rejected; removed "Play & Earn" label below daily check-in; first-open **welcome popup awarding ₹1 (1000 pts)** once per device.
- **Wallet:** removed "Withdraw" and "History" header labels.
- **Admin:** added Refresh button (top-left) + clean header (X to close); removed "Dashboard"/"Manage" labels.
- **Admin → Offerwall:** logo URL now optional (empty → default offerwall gift icon in tasks via TaskLogo); admin list uses TaskLogo.
- **Offerwall task detail:** when submission is OFF there is no proof field and no Submit button — shows a "no submission required" note + task link only.
- **Login screen:** fully reworked — responsive, balanced, compact; new self-made gradient hero (no broken external image), resized brand/feature elements.
- **Notifications:** local system (daily reminder scheduled; approved/rejected withdrawal, approved task, new task alerts on sync); Android channel + foreground handler + tap routing in `_layout.tsx`; permission requested contextually after onboarding; periodic background sync task; `google-services.json` (matches Firestore project) + `googleServicesFile` + POST_NOTIFICATIONS + expo-notifications/background-task plugins in app.json; iOS ad-tracking string added for store readiness.

## Backlog / Remaining
- **P0 (user action):** Open Firestore security rules so global features work live.
- **P0 (user action):** Click **Publish** → deploy → **Generate Android build** (APK). Notifications + AdMob only fully work on the real build (not Expo Go/web preview).
- **P1:** On the built APK, verify AdMob (banner/app-open/rewarded) + notifications (daily reminder + event alerts).
- **P2 (optional):** In-app data-deletion flow + ATT permission prompt if publishing to iOS App Store.

## Deployment / Health-check note
- Health check flags "unsupported stack (direct Firestore)" — this is **intentional per user choice** (keep Firestore, no FastAPI/Mongo migration). This app originated on Emergent and builds fine. Key build signals: `compilation_passed: true`, `expo_release_build_ok: true`, `expo_native_config_ok: true`, `dependency_manifests_valid: true`. APK build via Emergent Publish is expected to succeed.

## Build fix (2026-10-05)
- **EAS Android build failure fixed.** RUN_GRADLEW failed with `Cannot get property 'googleMobileAdsJson'...` (react-native-google-mobile-ads build.gradle line 123) + `does not specify compileSdk`. Root cause: RNGMA's `android/app-json.gradle` reads a **top-level `react-native-google-mobile-ads` key** in app.json to define `rootProject.ext.googleMobileAdsJson`; the key was absent, so line 123 threw and aborted evaluation before `android{}` set compileSdk (both errors share this one cause). Fix: added top-level `"react-native-google-mobile-ads": { android_app_id, ios_app_id }` to `frontend/app.json`. Validated locally (JSON valid, `expo config` parses with 10 plugins, simulated gradle read passes, web-preview regression PASS via testing_agent). The actual EAS build must be re-triggered via Publish to confirm.
