# Bluetooth AVRCP media/playlist browsing — failure analysis (Mazda 2 G90)

**Symptom:** Playback and track metadata over Bluetooth work, but browsing the
media library / playlists from the car's head unit (Mazda 2, MZD Connect) shows
an empty list, an error, or never loads. Android Auto browsing works fine.

## How AVRCP browsing reaches Tempus

Unlike Android Auto, the car never talks to Tempus directly. The phone's
Bluetooth stack (`com.android.bluetooth`, AOSP `packages/modules/Bluetooth`,
`audio_util` package) is the AVRCP *target* and proxies every browse request:

1. **At Bluetooth startup** `MediaPlayerList`/`BrowsablePlayerConnector` scans
   every app exposing `android.media.browse.MediaBrowserService`, connects,
   reads the root id, and fetches the root children. An app is only listed as
   a *browsable player* if all of that succeeds within 10 s
   (`CONNECT_TIMEOUT_MS`) **and the root children are non-empty**. The root
   children are prefetched and cached.
2. **Per folder request from the car** (`GetFolderItems`/`ChangePath`),
   `BrowsedPlayerWrapper` opens a *fresh* framework `MediaBrowser` connection,
   calls `getRoot()`, then `subscribe(folderId)`, converts the children, and
   disconnects. On Android 13+ the whole exchange must finish within
   **3 seconds** (`TimeoutHandler.SUBSCRIPTION_TIMEOUT_MS = 3000`), otherwise
   the stack answers the car with `STATUS_LOOKUP_ERROR` → the head unit shows
   an **empty folder**. On Android ≤ 12 there is *no* timeout: an unanswered
   subscription leaves the wrapper connected, and the next request calls
   `MediaBrowser.connect()` on an already-connected browser — browsing stays
   dead until Bluetooth restarts.
3. Only **successful** listings are cached by the stack (5-folder LRU), so a
   folder that failed once is re-fetched from scratch — and fails again the
   same way. From the driver's seat the feature looks permanently broken.

On the app side, each of those throwaway connections goes through media3's
legacy compatibility stub: `onGetRoot` → `onConnect` + `onGetLibraryRoot`
(blocking the binder thread on the app main thread), then `onLoadChildren` →
`MediaLibrarySessionCallback.onGetChildren`.

## Why Tempus failed

### 1. Every browse level below the root tabs was an uncached network call

`MediaBrowserTree.getChildren()` answers the root and tab folders from memory,
but everything below (playlist list, playlist songs, albums, artists, …) is a
live Subsonic API call. The first request also pays service cold-start
(ExoPlayer + Cast + queue restore) because the Bluetooth stack's connection is
what starts the service. DNS + TLS + server latency routinely exceeds the 3 s
budget, and since Tempus had **no cache**, every retry repeated the identical
slow path. Android Auto is unaffected: it holds one long-lived connection and
happily shows a spinner for slow folders.

### 2. Browse items carried artwork URIs the Bluetooth stack resolves synchronously

Song/album/playlist items set `artworkUri` to an
`AlbumArtContentProvider` `content://` URI (which downloads cover art from the
Subsonic server on demand). media3 maps `artworkUri` → the legacy item's
`iconUri`, and on stacks built with `avrcp_target_cover_art_uri_images=true`,
`audio_util.Metadata`/`Image` open **every item's icon URI synchronously**
while converting the children — one network download per item, for up to 500
items, all inside the same 3-second window. Guaranteed timeout.

### 3. Five repository methods could never answer at all

`getPlaylistSongs`, `getIndexes`, `getDirectories`, `getArtistAlbum` and
`search` only resolved their `SettableFuture` on a fully successful response.
A non-2xx status or empty body left the future pending forever → the browse
subscription was never answered → 3 s timeout on modern stacks, a permanent
wedge of the browsed-player wrapper on older ones. (`getRecentlyPlayedSongs`
additionally leaked a Room observer on every call, and the radio-station merge
thread could die without resolving its future.)

## Fixes applied (branch `feat-avrcp`)

1. `fix: always resolve Android Auto/AVRCP browse futures on error responses`
   — every response path now resolves the future (error `LibraryResult` on
   failure); Room observer leak and radio thread fixed.
2. `feat: answer Bluetooth AVRCP browse requests within the stack's 3s budget`
   — in `MediaLibrarySessionCallback`:
   - successful folder listings are cached (10 min TTL) and served to
     `com.android.bluetooth` immediately;
   - the root tabs and the playlist list are prefetched when the Bluetooth
     stack connects (throttled to once per 5 min), so the first folders a car
     opens are already warm;
   - artwork URIs are stripped from items returned to the Bluetooth stack
     (current-track art on the car display is unaffected — it comes from the
     session metadata path, not from browse lists).

Android Auto behaviour is unchanged: it is not in `BLUETOOTH_STACK_PACKAGES`,
so it keeps receiving fresh, artwork-carrying results.

## Remaining caveats

- The *first ever* visit to a deep folder (e.g. songs of one specific
  playlist) still races the 3 s budget if the server is slow; a second attempt
  succeeds from the cache. Pre-warming every playlist's contents was
  deliberately avoided to not hammer the server.
- If the user re-configures the Android Auto tabs, the Bluetooth view can be
  up to 10 minutes stale (cache TTL).
- Servers only reachable via home Wi-Fi/VPN cannot be browsed from the car at
  all — that is a connectivity constraint, not an app bug (playback would be
  equally impossible).

## References

- AOSP `packages/modules/Bluetooth` `audio_util`: `MediaPlayerList.java`,
  `BrowsedPlayerWrapper.java` (3 s subscription timeout, per-request
  connect/disconnect, success-only caching), `helpers/Metadata.java` /
  `helpers/Image.java` (synchronous icon-URI resolution).
- media3 `MediaLibraryServiceLegacyStub` / `MediaSessionServiceLegacyStub`
  (legacy browser bridge used by the Bluetooth stack).
- <https://developer.android.com/reference/android/service/media/MediaBrowserService>
- <https://developer.android.com/reference/android/media/browse/MediaBrowser>
