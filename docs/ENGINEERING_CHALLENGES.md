# Engineering Challenges

Seven problems that stand out in the production code. Outcomes describe observable behaviour in the code; no metrics are claimed.

---

## 1. Setting system sounds on Android without leaving debris

**Problem**
Writing the default ringtone, alarm or notification sound requires a special permission, behaves differently across OS levels, and a naive implementation inserts a new media row and file on every tap.

**Context**
The feature is the product's headline. It also covers per-contact ringtones, which need the contacts permission and a contact picker.

**Engineering approach**
- A Kotlin bridge validates OS version, write-settings permission and the source file before doing anything.
- Remote audio is first downloaded to a temporary local file, because the platform API needs a real path.
- The bridge reuses an existing MediaStore entry for the same title instead of inserting a duplicate, then sets the default through `RingtoneManager`.
- The staged temp file is always deleted, on success and on failure, and the app's own private download copy is never deleted by this path.
- Every failure maps to a specific error code that the Dart side turns into a localised message.
- A separate export path copies a download into public `Music/<app>` using a pending-write flag so a half-written file is not visible.

**Trade-offs**
Reusing entries by title means two different sounds with the same title would collide, so titles are the identity here. Needing the write-settings screen adds a step the user must complete outside the app.

**Outcome**
Repeating "Set as" does not accumulate files, and a permission or file problem is explained rather than silently failing.

---

## 2. Making iOS a first-class citizen where the platform forbids the feature

**Problem**
iOS offers no API to set ringtones or contact tones, but the product is built around that action.

**Engineering approach**
A platform-specific service reports capability flags. On iOS the flags are false, so the UI shows a complete explanatory flow and a real alternative (sharing the file to another app) instead of a dead-end button.

**Trade-offs**
The iOS experience is less magical than Android's, but it is honest and does not risk review rejection for a private-API workaround.

**Outcome**
One UI flow serves both platforms and branches on capabilities, not on `if (iOS)` scattered through screens.

---

## 3. One player, one source of truth, three possible sources

**Problem**
A sound may be downloaded, previously streamed, or only available over the network. The UI must behave identically, and rapid taps must not leave the player stuck in a loading state.

**Engineering approach**
- A resolver is the only code that decides the source: downloaded file, then play cache, then stream.
- Streams are saved to a best-effort cache in the background; failures are swallowed because playback must never depend on the cache.
- A monotonic token invalidates earlier play requests when a newer one starts.
- Missing or corrupted files on disk are treated as "not downloaded".
- A failure stream lets the UI show a message instead of a stuck spinner.

**Trade-offs**
The play cache is separate from user downloads, which means two kinds of local audio to explain; Settings → Storage shows both so nothing becomes invisible orphan storage.

**Outcome**
The player, waveform and mini-controls stay in sync, and replays do not re-stream.

---

## 4. Lock-screen controls that cannot freeze playback

**Problem**
Android media notifications and lock-screen controls need artwork as software bitmaps. Fetching it on the calling thread caused ANRs when playback started.

**Engineering approach**
A native media-session controller sets a placeholder synchronously, loads real artwork on a background thread, and ignores late results for a track that has since changed. just_audio does playback; the native side only mirrors state and routes remote-control actions back.

**Trade-offs**
Two layers (Dart player and native session) must be kept in sync, with a coordinator translating state both ways.

**Outcome**
Playback start does not wait on network or disk I/O for artwork.

---

## 5. Remote-controlled ads that default to off

**Problem**
Placements are toggled remotely, with several ad formats and networks behind a mediation layer, and rewarded flows must not trap the user.

**Engineering approach**
- One facade owns every full-screen ad request and loads strictly on demand.
- Config is fail-closed: until a real remote read succeeds, every placement is off; afterwards a placement must be explicitly `true`, and a global kill switch overrides all.
- Premium users are skipped in every gate, even if a caller forgets to check.
- A rewarded gate that cannot load an ad in time grants the reward rather than punishing the user; offline users get an explanatory dialog for rewarded flows but silent skipping for interstitials.
- Per-zone request throttling avoids bursts that can suppress fills.
- Unit tests cover ad error normalisation and lifecycle logging.

**Trade-offs**
Fail-closed protects policy compliance but gives up revenue when the config read fails. On-demand loading adds latency at show time compared with pre-warming.

**Outcome**
Ad behaviour is explainable and testable, and screens never call an ad SDK directly.

---

## 6. Local-first sync across devices

**Problem**
Favourites, recents, search history and settings should follow the account, work offline, and update live on a second device, without write loops.

**Engineering approach**
A single sync service reconciles local and cloud state once on start, then opens per-section live listeners. Apply-remote writes are flagged so they do not trigger a push that re-fires the listener. Account deletion and app reset clear local state in one place.

**Trade-offs**
Per-section documents keep writes small but need merge logic per section.

**Outcome**
Changes made on another signed-in device apply without waiting for the next launch.

---

## 7. Community content on a mobile app

**Problem**
Allowing user-recorded posts requires abuse controls and store-policy-friendly flows.

**Engineering approach**
- Posting and reporting require sign-in; the server verifies the identity token.
- Users can report a post and block an author; blocked authors and reported posts are filtered from the reporter's feed.
- Reports are only treated as sent after the server acknowledges the write, so offline cache echoes or quota failures cannot fake success.
- A moderation queue and review screens exist for admins, with Firestore rules distinguishing owners from moderation admins.
- Recordings have a short hard duration cap, matching the product's short-clip content and limiting upload size.

**Trade-offs**
Moderation depends on admin review capacity; the app provides the tooling, not automated detection.

**Outcome**
A reporting and blocking loop exists end to end, from the card menu to an admin decision.
