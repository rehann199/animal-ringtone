# Technical Case Study: Animal Ringtone

An engineering write-up of a Flutter sound-library and ringtone app with a serverless backend. It is based on read-only inspection of the production codebase. No production source is reproduced; descriptions are deliberately high level.

## 1. Product context

Animal Ringtone lets people browse animal and bird sounds, download them, and use them as ringtones, alarms, notification tones or per-contact tones. It also supports recording and posting short sounds. It is free to use with ads, with a premium tier that removes ads and unlocks every sound.

What makes it more than a content list is the amount of platform integration around a small piece of media: system sound writing, media sessions, offline storage, background playback and a moderated community feed.

## 2. Requirements that shaped the design

| Area | Requirement |
|---|---|
| Platform | Android and iOS from one codebase; accept that iOS cannot set ringtones |
| Content | Catalog that can change without an app release, but must always work offline |
| Playback | One consistent player state across several screens; background and lock-screen control |
| Storage | User downloads, an automatic play cache, and an optional public copy in Music |
| Monetisation | Free, rewarded and premium access per sound; ads that can be switched off remotely |
| Community | User recordings with reporting, blocking and moderation |
| Reach | 15 languages, including right-to-left languages |
| Safety | Never crash because a non-core service (analytics, notifications, billing, sync) failed |

## 3. Architecture

The app is layered: screens → Riverpod providers → repositories and services → platform and network. See [ARCHITECTURE.md](ARCHITECTURE.md) for diagrams.

- **Navigation.** `go_router` with a stateful tab shell (five tabs) so each tab keeps its scroll and state. Notification taps, deep links and paywall redirects go through the same router; a guard avoids sending already-premium users to the paywall from a stale push.
- **State.** Riverpod providers expose repositories, services and derived state (catalog, access, premium, connectivity, downloads, recents). Download and playback state is broadcast as a combined snapshot so widgets watch one reactive value.
- **Data access.** Repository interfaces with local, HTTP and Firestore implementations. The production sound repository sources the catalog through a fetch → cache → bundled-fallback chain, so the layer above it never sees a failure.
- **Services as single entry points.** Billing, ads, notifications, permissions, analytics and privacy each have one facade. Screens do not import the underlying SDKs.
- **Startup.** Errors are routed to crash reporting from the first frame, Firebase and background audio are initialised defensively, and a native splash overlay is inflated on top of the Flutter view until the first Flutter frame renders, so there is no blank gap.

## 4. Media and files

### Playback
A single shared `just_audio` player backs every play control. On Android a native media-session controller owns the notification and lock-screen controls; on iOS Now Playing is fed through the background-audio package. A coordinator keeps the two in sync and reads the selected language without a build context for localised notification text.

### Source resolution
A resolver chooses a downloaded file, then a play-cached file, then a stream. Streaming also writes to a cache in the background so the next play is local.

### Downloads and public export
A download manager owns the in-flight map so a sound cannot be downloaded twice concurrently, broadcasts progress, and repairs records with a zero file size by re-statting the real file. On Android it can export a download into public `Music/<app>` through MediaStore so it shows in the Files app, and remove that copy when the download is deleted.

### Recording
The recorder caps clip length, stops itself at the cap and hands the finished session to the preview step. Recordings are managed locally and can be renamed, shared or posted.

### Sharing
Sharing copies the file to a temporary name equal to the sound's title before invoking the OS share sheet, because the receiving app may otherwise show the internal catalog filename.

## 5. Platform integration (Android)

| Capability | Mechanism |
|---|---|
| Set ringtone / alarm / notification | Kotlin bridge: permission and OS checks, staged temp file, MediaStore reuse, `RingtoneManager` |
| Contact tone | Contact picker and per-contact custom ringtone, behind a runtime permission |
| Media session | Foreground media service, session controller, notification actions routed back to Dart |
| Public export | MediaStore insert with a pending flag |
| Updates | Play in-app update, flexible preferred, immediate as fallback |
| Install attribution | Play install referrer feeds deep-link handling on first launch |
| Splash | Single activity with a static overlay removed on first Flutter frame |

Permissions requested are the ones the features need: notifications, microphone (recording), contacts (contact tone), write-settings (system sounds, granted through a system screen), and media-playback foreground service.

## 6. Backend

A Cloudflare Workers service in TypeScript (Hono routing) with object storage for audio. It serves a health check, the catalog, audio with HTTP range support and cache headers, and a posts API. Identity tokens are verified against Google's public keys with `jose`, and Firestore and Messaging are reached over REST because the Node Admin SDK does not run on Workers. Layers follow route → service → repository, matching the Flutter side.

## 7. Authentication, sync and account lifecycle

Google and Apple sign-in. Signed-in users get a profile document, a push token and per-section sync documents. Sign-out removes the push token best-effort; reset and delete clear on-device state through one service. Guests can use the library without an account.

## 8. Monetisation and access

- **Access tiers** are computed per category from catalog order and flags: free, rewarded, premium.
- **Rewarded unlocks** last one hour per sound.
- **Premium** is purchased through a centralised billing service that handles availability, product queries, the purchase stream, restore, and verification before updating premium state. Billing initialisation never throws.
- **Ads** go through a mediation layer with several networks, loaded on demand, with remote placement flags, a global kill switch, fail-closed defaults, and per-zone throttling.
- **iOS** requests tracking transparency at most once per install; ad consent is collected by the mediation SDK.

## 9. Privacy and permissions

- Permission requests are preceded by an explanatory sheet where the system dialog cannot be repeated (notifications) or where the reason is non-obvious (microphone).
- Notification state is read for analytics without prompting; actual gating always goes through the request path.
- Account deletion and reset remove on-device data; cloud deletion is handled before the local wipe.
- Analytics have a debug-only validator that flags blank names, reserved prefixes and missing required parameters before they reach production.

## 10. Performance and resilience

- Artwork for media notifications loads off the main thread.
- Connectivity is verified with a lightweight request in addition to interface state, and rapid flaps are debounced so the offline dialog does not flicker.
- Network actions the user triggers directly re-check connectivity at tap time rather than trusting a cached status.
- Ads and sync degrade silently when offline; rewarded flows explain why nothing happened.
- Release builds use R8 minification and resource shrinking.
- Firebase Performance traces the startup path.

## 11. Testing

Unit tests cover ad error normalisation, ad lifecycle logging, billing error normalisation, the analytics validator and the analytics route observer (about thirty cases), plus a default widget test. The audio pipeline, set-as bridge and sync logic are verified manually; there are no automated tests around them.

## 12. Trade-offs and maintainability

- **Strengths:** clear service facades, interface-based repositories, consistent defensive handling of non-core failures, extensive doc comments explaining why.
- **Weaknesses:** the app is large (about 79k lines) and some screens are big; the sync service is a single 700-line coordinator; several comments still refer to internal planning phases; test coverage is concentrated on ads, billing and analytics.
- **Platform gap by design:** iOS cannot set ringtones, so the same product has different capabilities per OS.

## 13. Possible next steps

- Automated tests around source resolution, download manager and sync merging.
- Split the sync coordinator into per-section units.
- Typed enums for category and access instead of strings in some paths.
- Move more of the access policy to the backend so it can change without a release.
