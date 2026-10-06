# Architecture

All diagrams are original and conceptual. They describe responsibilities, not production class or file names.

## 1. Layers

```mermaid
flowchart TD
    UI[Screens and widgets<br/>go_router tab shell]
    ST[State<br/>Riverpod providers]
    RP[Repositories<br/>interfaces + local / HTTP / Firestore implementations]
    SV[Services<br/>audio, downloads, set-as, share, ads, billing,<br/>auth, sync, notifications, analytics]
    NT[Native bridges<br/>Kotlin media session, ringtone, MediaStore]
    BE[Cloudflare Workers backend]
    FB[Firebase]
    UI --> ST --> RP
    ST --> SV
    SV --> NT
    RP --> BE
    RP --> FB
    SV --> FB
```

Rules the code follows:

- Widgets talk to providers and services, not to SDKs. Billing, ads, notifications and permissions each have one facade.
- Repositories sit behind interfaces, with implementations chosen by providers, so a data source can change without touching screens.
- Platform code is behind platform channels and exposed as small, typed Dart services.

## 2. Where audio comes from

```mermaid
flowchart LR
    TAP[User taps play] --> PL[Shared audio player]
    PL --> RES{Source resolver}
    RES -->|1: file exists| DL[Downloaded copy]
    RES -->|2: cached| PC[Play cache]
    RES -->|3: otherwise| NET[Stream from backend]
    NET -.background.-> PC
    PL --> MS[Native media session<br/>notification + lock screen]
    PL --> UIW[Waveform, mini controls, player screen]
```

The player does not know which branch ran. Missing or corrupted local files are treated as "not downloaded". Playback requests carry a token so overlapping taps cannot interleave on the single player.

## 3. Set a sound as ringtone (Android)

```mermaid
sequenceDiagram
    participant U as User
    participant G as Access gate
    participant D as Dart service
    participant N as Kotlin bridge
    U->>G: Tap Ringtone / Alarm / Notification
    G-->>U: Rewarded ad or paywall if the sound is gated
    G->>D: Proceed
    D->>D: Download stream to a temp file if needed
    D->>N: canWriteSettings?
    N-->>D: false
    D-->>U: Explain, open the write-settings screen
    D->>N: setSystemSound(type, file, title)
    N->>N: Validate OS level and file
    N->>N: Reuse or insert MediaStore entry
    N->>N: RingtoneManager default for type
    N->>N: Delete staged temp file
    N-->>D: success or specific error code
    D-->>U: Localised message
```

Contact tones follow the same shape with a contact picker and a per-contact custom ringtone, behind a runtime contacts permission. iOS skips the native call and shows an explanation with a share alternative.

## 4. Catalog

```mermaid
flowchart LR
    APP[App] --> CS[Catalog service]
    CS -->|1| NET[Backend catalog endpoint]
    NET -->|success| CACHE[(Last-good cache)]
    CS -->|2 on failure| CACHE
    CS -->|3 on failure| BND[Bundled catalog]
```

The chain never fails: its last resort ships inside the app. Sound access (free, rewarded, premium) is computed from catalog order on the device.

## 5. Posting and moderation

```mermaid
flowchart LR
    R[Record] --> P[Preview and name]
    P --> SI{Signed in?}
    SI -->|no| AUTH[Google / Apple sign-in]
    AUTH --> UP
    SI -->|yes| UP[Upload to backend]
    UP --> FEED[Posts feed]
    FEED --> RPT[Report post]
    FEED --> BLK[Block author]
    RPT --> Q[(Reports + moderation queue)]
    Q --> ADM[Admin review screens]
```

Reports are only treated as successful once the server acknowledges the write. The rules allow a user to create their own reports, and restrict queue reads and enforcement updates to moderation admins.

## 6. Ads and access

```mermaid
flowchart TD
    REQ[Feature wants an ad] --> PREM{Premium?}
    PREM -->|yes| SKIP[Skip silently]
    PREM -->|no| CFG{Remote config confirmed this session?}
    CFG -->|no| OFF[Treat every placement as off]
    CFG -->|yes| PLC{Placement flag true and kill switch off?}
    PLC -->|no| OFF
    PLC -->|yes| TH{Zone throttle and cooldown ok?}
    TH -->|no| OFF
    TH -->|yes| SHOW[Load on demand and show]
```

Rewarded unlocks are stored per sound with an expiry time.

## 7. Backend

A Cloudflare Workers service written in TypeScript with Hono routing. It exposes a health check, the catalog, range-aware audio streaming with cache headers, and a posts API. Writes require a verified Firebase ID token. Tokens are verified against Google's public keys with `jose`, and the service reaches Firestore and Messaging over their REST APIs rather than the Node Admin SDK, which does not run in the Workers runtime. Code follows a route → service → repository pattern that mirrors the Flutter app.
