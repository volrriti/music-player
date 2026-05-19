# music-player

## Overview

A native Android music player app built in Kotlin. Clean, feature-rich, no ads, no streaming — plays local files only.

- **Package:** `com.example.musicplayer`
- **Min SDK:** Android 8 — **Target:** Android 14+
- **Architecture:** MVVM
- **Language:** Kotlin

---

## Dependencies

**libs.versions.toml:**

```toml
[libraries]
androidx-media3-exoplayer = { module = "androidx.media3:media3-exoplayer", version.ref = "media3Exoplayer" }
androidx-media3-ui = { module = "androidx.media3:media3-ui", version.ref = "media3Exoplayer" }
androidx-media3-session = { module = "androidx.media3:media3-session", version.ref = "media3Exoplayer" }
androidx-documentfile = { module = "androidx.documentfile:documentfile", version = "1.0.1" }
jaudiotagger = { group = "org.jaudiotagger", name = "jaudiotagger", version = "2.2.5" }
gson = { group = "com.google.code.gson", name = "gson", version = "2.10.1" }
```

**build.gradle (app):**

```kotlin
implementation(libs.androidx.media3.exoplayer)
implementation(libs.androidx.media3.ui)
implementation(libs.androidx.media3.session)
implementation(libs.androidx.recyclerview)
implementation(libs.androidx.lifecycle.viewmodel.ktx)
implementation(libs.androidx.viewpager2)
implementation(libs.material)
implementation(libs.androidx.fragment.ktx)
implementation(libs.androidx.documentfile)
implementation(libs.jaudiotagger)
implementation(libs.gson)
```

---

## File Structure

```
com.example.musicplayer/
├── MusicPlayerApp.kt
├── MainActivity.kt
├── NowPlayingActivity.kt
├── SettingsActivity.kt
├── ThemeManager.kt
├── PlaybackService.kt
├── PlayerViewModel.kt
├── MusicScanner.kt
├── LyricsParser.kt
├── AlbumArtLoader.kt
├── Song.kt
├── LyricLine.kt
├── SongAdapter.kt
├── LyricsAdapter.kt
├── GroupAdapter.kt
├── SongsFragment.kt
├── AlbumsFragment.kt
├── ArtistsFragment.kt
├── GroupSongsFragment.kt
├── ViewPagerAdapter.kt

res/layout/
├── activity_main.xml
├── activity_now_playing.xml
├── activity_settings.xml
├── fragment_albums.xml
├── fragment_artists.xml
├── fragment_songs.xml
├── song_item.xml
├── lyric_item.xml
├── queue_item.xml

res/drawable/
├── bg_circle_control.xml         ← circle background for prev/next buttons
├── bg_rounded_square_control.xml ← rounded square background for play/pause
├── bg_playing_dot.xml            ← dot indicator for active queue song
├── bg_queue_highlight.xml        ← highlight background for active queue song
├── bg_play_pause_circle.xml
├── seekbar_progress.xml
├── seekbar_thumb.xml
├── ic_launcher_background.xml
├── ic_launcher_foreground.xml
├── ic_play.xml
├── ic_pause.xml
├── ic_skip_next.xml
├── ic_skip_previous.xml
├── lyrics_fade_top.xml
├── lyrics_fade_bottom.xml

res/font/
├── geist.xml
├── geist_thin.ttf
├── geist_extralight.ttf
├── geist_light.ttf
├── geist_regular.ttf
├── geist_medium.ttf
├── geist_semibold.ttf
├── geist_bold.ttf
├── geist_extrabold.ttf
├── geist_black.ttf

res/values/
├── colors.xml
├── themes.xml

res/values-night/
├── colors.xml
├── themes.xml
```

---

## Architecture & Key Design Decisions

### `MusicPlayerApp.kt`

Custom `Application` class. Holds a reference to the current `MainActivity` instance (`currentMainActivity`). This is how `NowPlayingActivity` accesses the shared `PlayerViewModel` without being directly connected to `MainActivity`.

### `ThemeManager.kt`

Standalone object. Handles light/dark/system theme switching via `AppCompatDelegate.setDefaultNightMode()`. Persists selection to SharedPreferences under key `"theme_mode"` in `"settings"` prefs. Called on app launch from `MusicPlayerApp.onCreate()` to restore theme before any activity starts.

### `PlayerViewModel.kt`

Shared across all fragments via `activityViewModels()`. Holds:

- `songList` — full library
- `currentSong`
- `lyrics`
- `albums` — `Map<String, List<Song>>`
- `artists` — `Map<String, List<Song>>`
- `queue` — active playback queue (may differ from full library when playing from album/artist)

### `MainActivity.kt`

- Uses `DrawerLayout` with a hamburger menu (☰) — no TabLayout or ViewPager2
- Drawer contains: Songs / Albums / Artists / Pick Folder / Settings / Light / Dark / Follow System
- One fragment shown at a time in `fragmentContainer`, last selected screen persisted via `SharedPreferences`
- Player bar at bottom: shows current song title, artist, Play/Pause, album art thumbnail
- Tapping player bar opens `NowPlayingActivity`
- Builds its own `MediaController` connected to `PlaybackService`
- On launch: loads cached song list from `SharedPreferences` (Gson JSON) instantly — only rescans if no cache exists
- On folder pick: scans, saves to cache, updates ViewModel
- Theme switcher wired in drawer: calls `ThemeManager.set(prefs, mode)`

### `NowPlayingActivity.kt`

- Separate activity, builds its own `MediaController` directly connected to `PlaybackService`
- Top swipe area: three horizontal views — music info / lyrics / queue — swipe left/right to switch between them
- Music info view: album art (large) + title + artist
- Lyrics view: RecyclerView with synced, highlighted, auto-scrolling lyrics
- Queue view: reorderable queue via drag handles on the `—` handle; tap handle to remove song with collapse animation
- Controls bar pinned to bottom, outside swipe zone: ⏮ / Play/Pause / ⏭ / repeat
- Swipe down → closes activity (disabled when queue view is active so scrolling works freely)
- `playNext()` / `playPrevious()` use `viewModel.queue` (not `songList`) so album/artist context is respected
- Previous: restarts song if >3 seconds in, wraps to last song at start of list
- Next: wraps to first song at end of list
- Auto-advance: `onPlaybackStateChanged` catches `STATE_ENDED` → calls `playNext()`
- Album art loads via `AlbumArtLoader`, updates on song change
- Lyrics highlight and auto-scroll: active line stays in upper third of screen via `scrollLyricToUpperThird()` using `LinearLayoutManager.scrollToPositionWithOffset()`
- Gets ViewModel via `MusicPlayerApp.currentMainActivity?.viewModel`
- Lyrics RecyclerView has `setPadding(0, 80dp, 0, 80dp)` with `clipToPadding = false`
- `musicInfoView` uses `ConstraintLayout`. Album art constrained to 1:1 ratio. Title/artist/art vertically chained with `packed` chain style.
- `ivNowPlayingArt` rounding applied once in `onCreate` via `ViewOutlineProvider.setRoundRect()`, corner radius `40f`.
- Queue header shows `X/Y` song position on the left and `Clear` button on the right
- Queue status bar overlap fixed via `setOnApplyWindowInsetsListener` on `queueView`
- When song changes via queue tap, lyrics are reloaded via `viewModel.loadLyrics()`
- Active queue song scrolls to upper third when queue view opens, using `scrollQueueToCurrentSong()`
- Swipe-down dismiss is guarded: `currentViewState != ViewState.QUEUE` check prevents dismiss interfering with queue scroll
- `isSeeking` flag prevents seekbar position update loop from fighting user drag
#### Controls layout

Five elements in a horizontal `LinearLayout`:

- Previous (52dp circle, `bg_circle_control`)
- Play/Pause (68dp rounded square, `bg_rounded_square_control`)
- Next (52dp circle, `bg_circle_control`)

#### Queue song removal animation

Tapping the `—` drag handle triggers `onItemDismiss(position, itemView)` which collapses the item height from its measured height to 0 over 250ms using `ValueAnimator`, then calls `notifyItemRemoved`. No slide — pure vertical collapse so surrounding items slide up smoothly.

#### Drag-to-reorder

- `isDragging` flag on `SongAdapter` prevents `onQueueChanged` from firing on every step
- `onSelectedChanged` sets `isDragging = true` when drag starts
- `clearView` sets `isDragging = false` and propagates final order to ViewModel once
- `isLongPressDragEnabled() = false` to avoid conflict with handle touch listener

### `PlaybackService.kt`

Media3 `MediaSessionService` — handles background playback and notification controls.

### `MusicScanner.kt`

- Takes a folder URI (from `DocumentFile`/folder picker)
- Recursively scans all subdirectories
- Filters by `audio/` MIME type
- Extracts metadata (title, artist, album) via `MediaMetadataRetriever`
- Returns `List<Song>` sorted by title

### `AlbumArtLoader.kt`

- Extracts embedded album art from audio file URI using `MediaMetadataRetriever`
- In-memory `LruCache` (1/8 of available memory)
- Uses `imageView.tag` to prevent recycled rows from showing wrong art
- Background thread for extraction, posts result back to main thread

### `LyricsParser.kt`

Gets lyrics two ways:

1. Embedded in audio file via jaudiotagger (`FieldKey.LYRICS`), fallback to `MediaMetadataRetriever` key 28
2. `.lrc` file next to the audio file in the same folder

Parses LRC `[mm:ss.xx]` timestamps into `List<LyricLine>`.

### `LyricsAdapter.kt`

- Left-aligned lyrics, large text (35sp active, 25sp inactive)
- Bold typeface on active line, normal on inactive
- Highlights active lyric line with alpha (1f active, 0.4f inactive)
- Smooth alpha fade animation on highlight change (300ms)
- Uses `DiffUtil` in `updateLines()` for smooth list updates
- Only highlights within the window `start..nextLineStart`
- Tap a lyric line → seeks playback to that timestamp
- Active line scrolled to upper third of screen

### `GroupSongsFragment.kt`

Shows songs within a tapped album or artist. On song tap, builds a queue from that group. Sets this as `viewModel.queue`.

### Song caching

Songs serialized to JSON via Gson, stored in `SharedPreferences` under `"cached_songs"`. Loaded instantly on launch.

---

## Data Models

```kotlin
data class Song(
    val title: String,
    val artist: String?,
    val album: String?,
    val uri: Uri,
    val albumArtist: String? = null,
    val fileType: String? = null,
    val quality: String? = null
)

data class LyricLine(
    val timestampMs: Long,
    val text: String
)
```

---

## Theme System

Uses `AppCompatDelegate` day/night mode. Three options: Light, Dark, Follow System.

**`res/values/themes.xml`:**

```xml
<style name="Theme.MusicPlayer" parent="Theme.MaterialComponents.DayNight.NoActionBar">
    <item name="colorPrimary">@color/black</item>
    <item name="colorPrimaryVariant">@color/black</item>
    <item name="colorOnPrimary">@color/white</item>
    <item name="colorSurface">@color/white</item>
    <item name="colorOnSurface">@color/black</item>
    <item name="android:windowBackground">@color/white</item>
    <item name="android:textColorPrimary">@color/black</item>
    <item name="android:textColorSecondary">#99000000</item>
    <item name="android:fontFamily">@font/geist</item>
    <item name="android:textStyle">bold</item>
    <item name="android:letterSpacing">-0.03</item>
</style>
```

**`res/values-night/themes.xml`:**

```xml
<style name="Theme.MusicPlayer" parent="Theme.MaterialComponents.DayNight.NoActionBar">
    <item name="colorPrimary">@color/white</item>
    <item name="colorPrimaryVariant">@color/white</item>
    <item name="colorOnPrimary">@color/black</item>
    <item name="colorSurface">@color/black</item>
    <item name="colorOnSurface">@color/white</item>
    <item name="android:windowBackground">@color/black</item>
    <item name="android:textColorPrimary">@color/white</item>
    <item name="android:textColorSecondary">#99ffffff</item>
    <item name="android:fontFamily">@font/geist</item>
    <item name="android:textStyle">bold</item>
    <item name="android:letterSpacing">-0.03</item>
</style>
```

**`res/values/colors.xml`:**

```xml
<color name="black">#FF000000</color>
<color name="white">#FFFFFFFF</color>
<color name="song_highlight">#1A7E57C2</color>
<color name="seekbar_color">#FF000000</color>
<color name="seekbar_remaining">#33000000</color>
<color name="icon_tint">#FF000000</color>
```

**`res/values-night/colors.xml`:**

```xml
<color name="seekbar_color">#FFFFFFFF</color>
<color name="seekbar_remaining">#33FFFFFF</color>
<color name="icon_tint">#FFFFFFFF</color>
```

---

## Drawables

### `bg_circle_control.xml` — prev/next button background

```xml
<shape xmlns:android="http://schemas.android.com/apk/res/android" android:shape="oval">
    <solid android:color="?attr/colorSurface" />
    <stroke android:width="1dp" android:color="?attr/colorOnSurface" android:alpha="0.15"/>
</shape>
```

### `bg_rounded_square_control.xml` — play/pause button background

```xml
<shape xmlns:android="http://schemas.android.com/apk/res/android" android:shape="rectangle">
    <solid android:color="?attr/colorOnSurface" />
    <corners android:radius="20dp" />
</shape>
```

### `bg_playing_dot.xml` — dot indicator for active queue song

```xml
<shape xmlns:android="http://schemas.android.com/apk/res/android" android:shape="oval">
    <solid android:color="?attr/colorOnSurface"/>
</shape>
```

### `bg_queue_highlight.xml` — highlight for active queue song

```xml
<shape xmlns:android="http://schemas.android.com/apk/res/android" android:shape="rectangle">
    <solid android:color="#FF000000"/>
    <corners android:radius="12dp"/>
</shape>
```

### `seekbar_progress.xml`

```xml
<layer-list xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:gravity="center_vertical" android:height="4dp">
        <shape android:shape="rectangle">
            <solid android:color="@color/seekbar_remaining" />
            <corners android:radius="99dp" />
        </shape>
    </item>
    <item android:id="@android:id/progress" android:gravity="center_vertical" android:height="4dp">
        <clip>
            <shape android:shape="rectangle">
                <solid android:color="@color/seekbar_color" />
                <corners android:radius="99dp" />
            </shape>
        </clip>
    </item>
</layer-list>
```

---

## Font

Geist by Vercel. Individual weight TTF files in `res/font/`. Applied globally via theme with `android:letterSpacing="-0.03"` for tight tracking across all text sizes (em units = proportional to font size).

---

## Seekbar

No thumb (`android:thumb="@null"`). Height controlled via drawable layer `android:height="4dp"`. View height is `wrap_content` with `paddingTop/Bottom="20dp"` and negative margins to tighten spacing with time labels. Seekbar and time labels use `paddingStart/End="16dp"` to align with controls. `isSeeking` flag in `NowPlayingActivity` prevents the position update loop from overriding a user drag.

---

## queue_item.xml layout notes

- Outer `LinearLayout` has `paddingStart/End="12dp"` and `paddingTop/Bottom="2dp"` to give rounded highlight room
- `queueItemRoot` inner `LinearLayout` has `background="?android:attr/selectableItemBackground"` normally; swapped to `bg_queue_highlight` when selected
- `playingIndicator` dot is `6dp` oval on the left, `visibility="invisible"` (not gone) when inactive so text doesn't shift
- `btnDragHandle` is a `TextView` showing `—`, used for both drag (move finger) and remove (tap)
- Divider line removed — padding between items provides separation

## activity_now_playing.xml layout notes

- `controlsBar` has `paddingBottom="48dp"` to push controls up from the bottom edge
- Artist `TextView` has `android:textStyle="normal"` to override the global bold theme
- Seekbar `paddingStart/End="16dp"` and time labels `RelativeLayout` `paddingStart/End="16dp"` to align with album art `32dp` content area
- `queueView` uses `setOnApplyWindowInsetsListener` in code to apply status bar top padding — avoids overlap with system UI in edge-to-edge mode

---

## AndroidManifest.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE"/>
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK"/>
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS"/>
    <uses-permission android:name="android.permission.READ_MEDIA_AUDIO"/>
    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"
        android:maxSdkVersion="32"/>
    <application
        android:name=".MusicPlayerApp"
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.MusicPlayer">
        <activity android:name=".NowPlayingActivity" android:exported="false"/>
        <activity
            android:name=".MainActivity"
            android:exported="true"
            android:label="@string/app_name"
            android:theme="@style/Theme.MusicPlayer">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
        <activity android:name=".SettingsActivity" android:exported="false"/>
        <service
            android:name=".PlaybackService"
            android:foregroundServiceType="mediaPlayback"
            android:exported="true">
            <intent-filter>
                <action android:name="androidx.media3.session.MediaSessionService"/>
            </intent-filter>
        </service>
    </application>
</manifest>
```

---

## What's Done ✅

- ExoPlayer + Media3 background playback with notification controls
- Folder picker with permission handling
- Recursive subdirectory scanning
- Song list cached to SharedPreferences — instant load on relaunch
- MVVM with shared ViewModel
- DrawerLayout navigation (Songs / Albums / Artists / Pick Folder / Settings)
- Last selected screen persisted across launches
- Albums and Artists grouping — tap to view songs within group
- Queue system — playing from album/artist sets a contextual queue
- Reorderable queue via drag on `—` handle
- Remove from queue via tap on `—` handle with collapse animation
- Active queue song highlighted with black rounded background + dot indicator
- Active queue song scrolls to upper third when queue view opens
- Queue header: current position `X/Y` on left, `Clear` button on right
- Queue view does not trigger swipe-down-to-dismiss
- Synced lyrics (LRC files + embedded metadata)
- Lyrics updated when changing song via queue tap
- Lyrics highlight + auto-scroll to upper third of screen
- Smooth lyrics fade transition via DiffUtil + alpha animation
- Tap lyric line to seek
- Now Playing screen: swipe left/right (music info ↔ lyrics ↔ queue)
- Swipe down to close Now Playing (info and lyrics views only)
- Album art on Now Playing screen (large, rounded corners 40f)
- Album art thumbnails on song list and albums list
- Album art LruCache — no flicker on scroll
- Previous / Next with wrap-around
- Previous restarts if >3 seconds in
- Auto-advance to next song
- Shuffle button (left of controls) — toggles ExoPlayer shuffle mode
- Repeat button (right of controls) — cycles NONE → ALL → ONE
- Controls: shuffle / prev (circle) / play-pause (rounded square) / next (circle) / repeat
- Seekbar with rounded corners, no thumb, theme-aware colors
- Seekbar and time labels inset to align with album art
- Artist name non-bold on Now Playing screen
- Negative letter spacing (`-0.03em`) applied globally via theme
- Controls bar pushed up with `paddingBottom="48dp"`
- Light / Dark / Follow System theme switcher in drawer
- Geist font applied globally
- All backgrounds use `?attr/colorSurface` — no hardcoded colors

---

## What's Next 📋

- Highlight currently playing song row in song list
- Animated blurred background (album art colors bleed into background)
- Sleep timer (Settings screen)
- Tag editing
- Search + sorting
- albumArtist, fileType, quality not showing — MusicScanner needs updating
- Performance pass
- Final polish

---

## Known Quirks

- `NowPlayingActivity` gets the ViewModel indirectly via `MusicPlayerApp.currentMainActivity` — if `MainActivity` is destroyed, this returns null. Always null-check.
- Both `MainActivity` and `NowPlayingActivity` have separate `MediaController` instances pointing to the same `PlaybackService` — they stay in sync because they share the same session.
- Song cache stores URIs as strings — if the user moves their music folder, the cache becomes stale. Fix: re-pick the folder to trigger a fresh scan.
- `GroupAdapter` takes an optional `representativeSongs: Map<String, Song>` — pass the first song of each album/artist to get its art shown. `ArtistsFragment` currently doesn't pass this (artists show no art thumbnail).
- Font files must be all lowercase with underscores in `res/font/`.
- Do not use hardcoded colors in layouts — always use `?attr/colorSurface`, `?attr/colorOnSurface` etc. so theme switching works correctly.
- `bg_queue_highlight` uses hardcoded `#FF000000` black — if you want it to work in both light and dark themes, replace with `?attr/colorOnSurface`.
- Queue position counter (`X/Y`) matches by URI using `indexOfFirst { it.uri == currentUri }` — not by object equality, which is unreliable with data classes containing Uri.
- `isDragging` flag on `SongAdapter` must be reset in `clearView` of the `ItemTouchHelper` callback — if drag is cancelled abnormally, the flag could get stuck. The `clearView` callback always fires on drag end regardless of outcome.
