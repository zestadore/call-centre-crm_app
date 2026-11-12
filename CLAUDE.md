# Call Recording CRM — Project Context

Device-based call recording CRM (inspired by Salestrail) for a multi-city Indian call-centre operation. Records SIM + WhatsApp calls from agents' Android phones, plus optional VoIP / external call-centre ingestion, into a central CRM with a web dashboard. Full CRM functionality is in scope (see `/docs/` roadmap).

This file holds settled decisions. Treat them as fixed unless the user explicitly reopens one. Do not re-litigate or suggest alternatives to the locked choices below.

## Repositories

- Backend (Laravel): `call-centre-crm_backend` — github.com/zestadore/call-centre-crm_backend
- Frontend (React): `call-centre-crm_frontend` — github.com/zestadore/call-centre-crm_frontend
- Mobile (Kotlin): `call-centre-crm_app` — on hold for now

## Tech stack (LOCKED)

- **Backend:** Laravel 13 (PHP 8.4+), MySQL 8, Redis (queue/cache/sessions), Laravel Horizon, Laravel Sanctum (token auth for mobile, cookie auth for SPA). REST API under `/api/v1`.
  - Changed from Laravel 11 / PHP 8.3 on 2026-11-12: Laravel 11 stopped getting security fixes in March 2026, and the backend was installed as Laravel 13.
- **Frontend:** React + TypeScript + Vite (SPA, internal auth-only tool, no SSR). TanStack Query, TanStack Table, dnd-kit (pipeline), Recharts, shadcn/ui, React Hook Form + Zod.
- **Mobile:** Kotlin + Jetpack Compose (native Android). MVVM + repository, Retrofit/OkHttp, Coroutines/Flow, Hilt, Room, WorkManager, Coil. Two components: the Agent app (UI) and a sideloaded Capture component (recording engine).
- **Storage:** S3 or Cloudflare R2 for recordings — private bucket, presigned URLs.
- **Real-time:** Laravel Reverb + Echo (later phase: live call status, screen-pop).
- **Deploy:** Laravel Forge or Docker; server in an India region; staging + production.

## Explicitly decided / OUT of scope — do NOT suggest otherwise

- Android only. No iOS — Apple platform cannot record calls, for any product.
- Native Kotlin. No Flutter, no React Native. (RN was only ever an option for the agent-facing UI; the capture/recording engine is always native Kotlin.)
- No AI / transcription / summarization yet — explicitly deferred.
- Auth: Sanctum, not Passport (first-party apps; no OAuth2 provider needs).
- Database: MySQL, not Postgres.
- Compression: self-hosted and free — on-device `MediaCodec` (AAC) and/or server-side FFmpeg (Opus/AAC). No external or paid service.

## Core data model

- **Call:** id, user_id, phone_number, direction (inbound/outbound), channel (`sim`/`whatsapp`/`voip`), source (`device`/`telephony`), start_time, end_time, duration_seconds, recording_path, recording_status (`pending`/`uploaded`/`failed`/`unavailable`), external_call_id, telephony_provider, leg_group_id, timestamps.
- **User:** standard fields + role + extension/sip_id (telephony mapping) + device info.
- CRM entities (Contact, Lead, Deal, Task, Campaign) added in later phases.

## Capture mechanisms (Android)

- **SIM:** call state via `TelephonyManager`/`PhoneStateListener`; audio via the phone's built-in recorder (detected through a `ContentObserver` on MediaStore) or, as fallback, the sideloaded `AccessibilityService` recorder (Play Store bars call recording from Android 9+).
- **WhatsApp:** detected via `NotificationListenerService`; recorded via the same accessibility recorder.
- **Resilience:** foreground service for persistence; Room as the offline queue; WorkManager for background upload with retry. No call is lost offline.

## Jobs & queues (Redis + Horizon)

ProcessRecordingUpload, TranscodeRecording, FetchVoipRecording, ReconcileTelephonyCdrs, SendCampaignMessage, RunFollowUpAutomation, GenerateReportAggregates, RetryFailedUpload. Split queues by priority so recording jobs are never starved by bulk campaign jobs.

## VoIP / external call centre

- Phone system records; we ingest. Endpoint: `POST /api/v1/telephony/{provider}/cdr` (HMAC signature + IP allowlist auth — NOT Sanctum).
- CDR reaches us by push (provider webhook), pull (we poll their API), or a connector we run on a self-hosted PBX (Asterisk/FreePBX). Design stance: webhook primary + polling reconciliation.
- Provider adapter → canonical CDR → create Call (`channel=voip`) → FetchVoipRecording → transcode → R2. Dedup on `external_call_id`.
- Optional outbound control (where the provider API supports it): click-to-call, screen-pop (via Reverb), live call status.

## Build order

1. Backend API contract first — endpoints + data model.
2. React dashboard against it.
3. Deploy to staging early (HTTPS, India region).
4. Mobile app against the staging API URL. Capture layer last, or prototyped in parallel on a real device to de-risk early.

## Dev environment

- Backend + React run in the cloud and/or locally (Laragon/Herd/Docker on Windows).
- Mobile builds on the developer's Windows laptop (Android Studio); keep SDK + project on the SSD. Test on REAL phones — India-market OEMs (Redmi/Xiaomi, Vivo, Oppo, Samsung), including at least one with and one without a built-in recorder. Capture features do NOT work on emulators.
- Mobile talks to the deployed staging API over HTTPS (Android blocks cleartext to non-localhost).

## Conventions

- API versioned `/api/v1`, JSON, paginated lists, policy-based authorization per role (agent / team lead / city manager / admin).
- All timestamps stored in UTC, displayed in IST.
- Phone numbers normalized to E.164.
- Recordings always private; access via short-TTL presigned URLs only.

## Coding conventions — Kotlin / Android (MANDATORY)

Follow Google's official Android architecture guidance (Guide to app architecture, Now in Android patterns) and Kotlin coding conventions exactly. Do not invent structure.

- **Architecture:** MVVM + repository, unidirectional data flow. Layers: `ui` → `domain` (use cases, only where logic is shared) → `data` (repositories, remote, local).
- **Packages (by feature):** `feature/<name>/{ui,data,domain}`, plus `core/{network,database,model,designsystem,common}`. The Capture component (recording engine) is a separate module and never depends on the UI module.
- **UI:** Jetpack Compose only, Material 3. Screens are stateless composables that take a `UiState` + event lambdas; a `…Route` composable connects the ViewModel. Previews for every screen.
- **ViewModels:** `@HiltViewModel`, expose `StateFlow<UiState>` (immutable data class / sealed interface), collected with `collectAsStateWithLifecycle()`. No Android `Context` in ViewModels.
- **DI:** Hilt everywhere (`@Module`/`@InstallIn`). No manual singletons.
- **Async:** Coroutines + Flow; inject dispatchers. No `GlobalScope`, no blocking calls on the main thread.
- **Network:** Retrofit + OkHttp + kotlinx.serialization; DTOs mapped to domain models in the data layer. Sanctum bearer token via an OkHttp interceptor; token kept in encrypted storage. Base URL is the HTTPS staging API.
- **Local/offline:** Room is the source of truth and the offline queue; entities mapped to domain models. WorkManager for uploads with retry and constraints. No call is lost offline.
- **Services:** foreground service for capture; `NotificationListenerService` / `AccessibilityService` only in the Capture module.
- **Results/errors:** repositories return `Result`/sealed types; no swallowed exceptions.
- **Style:** official Kotlin style (ktlint), immutable `val` by default, explicit visibility for public API.
- **Tests:** unit tests for ViewModels and repositories (JUnit, coroutines-test, Turbine). Capture features are verified on real devices only.

## Open items to confirm [TBC]

- Telephony provider (or self-hosted PBX); ingestion-only vs full click-to-call.
- Compression codec/bitrate; on-device vs server-side default; keep original or not.
- Recording retention period.
- Search engine: Meilisearch vs MySQL full-text.
- Transfer/conference call handling; audit-trail scope on recording access.

## Reference docs

Full detail in `/docs/`:

- `03_Final_Technical_Design_Document` — architecture, this project's single technical reference
- `02_Full_CRM_Product_Plan_and_Roadmap` — complete CRM feature catalogue + phased delivery
- `01_Call_Recording_CRM_Phase1_Product_Documentation` — client-facing scope of the first release
