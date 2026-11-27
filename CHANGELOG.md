[Leer en español](./CHANGELOG.es.md)

# Changelog (DeckMix Player)

Chronological record of versions and improvements implemented in DeckMix Player, ordered from newest to oldest.

## [v1.2.1] - Configurable Master EQ, Stereo Balance, Onboarding, Developer Mode, and Lossless `.m4a`

*(Includes all Mix Mode improvements and bug fixes from internal build `v1.1.1`.)*

### Audio Engine and Master Bus Controls
- **Configurable 5 / 10 / 15-band Master Graphic EQ + Q factor (`masterEq.ts`, `AudioEngine.ts`, `SettingsView.tsx`, `usePlayer.tsx`, `backup.ts`):**
  - Added selectable Master EQ resolutions of **5 bands** (`60 Hz`–`14 kHz`), **10 bands** (`32 Hz`–`16 kHz`, ISO octaves), and **15 bands** (`25 Hz`–`16 kHz`, ISO 2/3-octaves), plus an adjustable **Q factor** slider (`0.5`–`4.0`, default `1.0`) applied to peaking bands while preserving stable low-shelf and high-shelf extremes.
  - Added smooth log-frequency curve resampling (`resampleMasterEqBands`) when switching live between 5 / 10 / 15 bands and automatic migration of legacy 6-band profiles from `UserPreferences` or JSON backups without breaking saved configurations.
  - Preserved true audio-graph bypass: `BiquadFilterNode` stages are dynamically rebuilt on band-count changes and fully disconnected from the Master chain whenever the EQ is off or flat (`0 dB`).
- **Left / Right Stereo Balance with true bypass (`AudioEngine.ts`, `SettingsView.tsx`, `db.ts`, `backup.ts`, `usePlayer.tsx`):**
  - Added an **L / R stereo balance** control (`-100%` Left to `+100%` Right, with quick Center button and slider double-click reset) in `Settings -> Audio and playback` alongside Mono mode.
  - Implemented via an on-demand `StereoPannerNode` between `masterGain` and `limiter`, bypassed completely at `0%`, and persisted in `UserPreferences` and JSON backups.

### Onboarding, Public Developer Mode, and Diagnostics
- **First-run Welcome / Onboarding wizard (`OnboardingModal.tsx`, `App.tsx`, `SettingsView.tsx`, `usePlayer.tsx`, `db.ts`):**
  - Added a 4-step dark DeckMix onboarding modal shown only on first launch (`onboardingCompleted` persisted in `UserPreferences`, with a persistent **Skip** button): (1) Language (`Español` / `English`), (2) First music folder linking (native picker in Electron or browser picker on `localhost`), (3) Quick preferences (Auto-Gain LUFS, Autoplay, and Electron desktop toggles), and (4) Mix Mode and essential keyboard shortcuts overview.
  - Automatically marks `onboardingCompleted` for existing profiles upgrading from earlier versions that already have linked folders or library tracks, while exposing a **"View welcome guide again"** button in `Settings -> About DeckMix`.
- **Public Developer Mode with detailed error toasts (`devErrorBus.ts`, `DevErrorToasts.tsx`, `SettingsView.tsx`, `App.tsx`, `usePlayer.tsx`, `AudioEngine.ts`, `lrclibClient.ts`, `midiController.ts`):**
  - Added a **"Developer mode"** toggle in `Settings` (disabled by default for regular users) persisted in `UserPreferences` and JSON backups.
  - When enabled, displays non-blocking bottom-right diagnostic toasts (capped at 4 visible cards with 6-second deduplication and `xN` counter) for real captured failures across audio decoding/streaming, Electron IPC, FFmpeg, LRCLIB, Discord RPC, Web MIDI, folder watcher, Wi-Fi remote, and unhandled runtime errors, complete with collapsible technical details and one-click copy to clipboard.

### Lossless `.m4a` (FLAC / ALAC) Playback in Electron
- **Automatic lossless `.m4a` FLAC / ALAC support via FFmpeg (`electron/main/libraryBridge.ts`, `electron/main/mediaTools.ts`, `electron/main/index.ts`, `electron/preload/index.ts`, `vite-env.d.ts`, `AudioEngine.ts`):**
  - When an `.m4a` track fails in `<audio>` or `decodeAudioData`, Electron detects the embedded codec via `music-metadata` (with an `ffmpeg -i` fallback probe) and losslessly remuxes FLAC (`-c:a copy`) or converts ALAC (`-c:a flac`) using atomic writes into a bounded `512 MB` LRU cache under `userData/cache/m4a-flac/`.
  - Served transparently via `deckmix://local/m4a-cache?key=...` with the same CORS (`Access-Control-Allow-Origin: *`, `Access-Control-Expose-Headers`) and HTTP `Range` (`206 Partial Content`) headers as `deckmix://local/file`, supporting instant `loadStream()` playback as well as Mix Mode `decodeTrackAudio`.
  - Includes single-flight per-file deduplication, automatic FFmpeg child-process cleanup on `before-quit`, and clean localized fallback messages on web `localhost` or when FFmpeg is not installed.

### Internationalization and Settings Search
- **Full 1:1 ES/EN i18n parity and updated Settings search (`src/i18n/locales/en/`, `src/i18n/locales/es/`, `SettingsView.tsx`):** added all Spanish and English translations for Master EQ bands/Q, Stereo Balance, Developer Mode, Onboarding, and lossless `.m4a` notices, plus bilingual search keywords (`balance`, `pan`, `factor q`, `5/10/15`, `developer`, `onboarding`, `welcome`, `m4a`, `alac`, etc.) in `Settings`.

---

## [v1.1.1] - Mix Mode Update

*(Internal development build; included in `v1.2.1`.)*

### New Mix Mode Features
- **Unobstructed waveform canvas and dedicated bottom control bar (`WaveformCanvas.tsx`):** relocated the preview Play/Pause and Restart buttons (along with `RGB`, `bars` selector, Move mix, Pan mode, and Zoom controls) out of the center of the waveforms into a dedicated toolbar directly beneath the canvas.
- **Zoomed waveform panning and interactive minimap scrollbar (`WaveformCanvas.tsx`):** added horizontal viewport navigation at zoom `> 1.0x` without accidentally shifting Track A or Track B, via a draggable horizontal scrollbar beneath the waveforms (highlighting the transition overlap region), a `Hand` pan-mode toggle (`H`), middle-mouse drag or `Alt + drag`, `Shift + wheel` / horizontal trackpad scroll, cursor-anchored `Ctrl/Alt + wheel` zoom, and one-click zoom reset to `1.0x`.
- **Move transition window with curves anchored to the music (`mixMath.ts`, `WaveformCanvas.tsx`, `MixTransitionPanel.tsx`):** new **"Move mix"** mode (`W`, plus `[` / `]` per bar and `Shift + [` / `]` per beat) that shifts Track A and Track B together in beat sync while automatically counter-shifting Volume, 3-band EQ, Filter, and FX curves (`shiftTransitionCurvesForWindowMove`) so every node stays locked to the exact same beat, vocal, or drop without re-editing.
- **Live CUE playback line (`WaveformCanvas.tsx`, `MixTransitionPanel.tsx`):** draws a 60 fps cyan `CUE` playhead directly inside Track A's or Track B's lane while pre-listening on CUE, plus an off-screen directional edge badge (`<< CUE` / `CUE >>`) when zoomed outside the CUE position.
- **Mix Mode keyboard shortcuts and in-editor shortcuts panel (`MixTransitionPanel.tsx`, `useKeyboardShortcuts.ts`, `SettingsView.tsx`, `usePlayer.tsx`):** shortcuts active automatically while **Edit transition** is open:
  - `1`–`6`: select and focus editable curves over the waveforms (`Volume`, `EQ Lows`, `EQ Mids`, `EQ Highs`, `Filter`, `Effects`).
  - `Space` (single tap): play or pause transition preview.
  - **Quick double-tap `Space`** (`<= 340 ms`): restart transition preview from pre-roll, configurable to **`R`**, **`Backspace`**, or **`0`** in both `Settings -> Keyboard shortcuts` and the Mix shortcuts dialog.
  - `W` (Move mix mode), `[` / `]` (shift 1 bar), `Shift + [` / `]` (shift 1 beat), `H` (Pan mode when zoomed), `+` / `-` (Zoom), `Shift + A` / `Shift + B` (CUE Track A / B), and `?` (or the header keyboard icon button) to open the Mix shortcuts panel.
- **Smooth preview fade-in and fade-out (`Deck.ts`, `AudioEngine.ts`):** added a gentle fade-in on Track A at preview pre-roll start and a smooth fade-out on Track B at the end of post-roll (`fadeOutAtCtxTime` / `fadeOutSec` on `Deck.playScheduled`) to remove abrupt cuts at preview boundaries.
- **Smart CUE volume and stacked dual volume control (`cuePlayer.ts`, `usePlayer.tsx`, `PlayerBar.tsx`):** when CUE shares the same audio output device as normal playback, CUE automatically matches and follows the main volume slider and mute toggle; when CUE uses a separate output device and the transition editor is open, a vertically centered **CUE volume slider** appears stacked right above the main volume slider in `PlayerBar`.
- **Automatic CUE / preview mutual exclusion on shared output (`usePlayer.tsx`, `MixTransitionPanel.tsx`):** when CUE and normal music use the same audio output, starting CUE pauses the transition preview (or normal playback) and starting/resuming preview or playback stops CUE so they never overlap, while keeping simultaneous playback when using a dedicated CUE output.

### Mix Mode and Playback Bug Fixes
- **Exact automation curve paths through every node (`WaveformCanvas.tsx`):** replaced fixed 40/44-step uniform sampling with vertex-exact `buildCurvePolylinePoints` for both Canvas 2D drawing and SVG click targets, fixing curve lines cutting corners or detaching from nodes at high zoom (`3.5x`–`4.0x`) and fixing zoomed left/right boundary rendering.
- **Sample-accurate Track B beat sync after seeking Track A (`Deck.ts`, `AudioEngine.ts`, `usePlayer.tsx`):** fixed Track B occasionally entering slightly early or late when manually seeking near the end of Track A during normal playback by pre-decoding armed Auto-Mix tracks into `AudioBuffer`, upgrading cached Track A from streaming to `AudioBuffer` during seeks (eliminating MP3/M4A VBR seek drift), waiting until `HTMLAudioElement` seeks settle (`isSeeking()`), accounting for post-seek `startDelaySec` (`getRemainingWallSecTo`), and analytically solving exact mid-transition progress `u` (`solveProgressFromConsumedSec`) when seeking onto or past `outPointSecA`.
- **Eliminated end-of-transition micro-cut when switching to Track B (`Deck.ts`, `AudioEngine.ts`):** stopped disconnecting live main-chain Web Audio nodes (`toneStage`, `midSide`, `pitchShifterNode`) while Track B is playing at full volume, replaced 1-sample parameter resets with smooth `40–60 ms` ramps, and added a `60 ms` micro-fade when clearing outgoing Deck A.
- **Playlist progression and Autoplay queue fix (`usePlayer.tsx`, `TrackList.tsx`):** fixed an issue where finishing a song inside a playlist could prematurely jump to a random library Autoplay track before the last song of the playlist unless Shuffle was toggled.

---

## [v1.0.1] - Day-One Portable Build Hotfix

*(Note: The `v1.0.0` portable release currently published on GitHub already ships this `v1.0.1` day-one hotfix build; the release tag remained `v1.0.0` by oversight.)*

### Packaged Portable `.exe` Bug Fixes and Visual Polish
- **V8 bytecode module resolution (`electron/main/libraryBridge.ts`, `electron/main/updater.ts`):** fixed runtime `ERR_VM_DYNAMIC_IMPORT_CALLBACK_MISSING` errors inside the compiled `out/main/index.cjsc` bytecode by loading `music-metadata` and `electron-updater` safely via `createRequire`.
- **Native Electron file pickers (`electron/main/index.ts`, `electron/preload/index.ts`, `PlaylistModal.tsx`, `ProfileView.tsx`, `SettingsView.tsx`):** replaced hidden `<input type="file">` pickers blocked in packaged Electron 45 with native `dialog.showOpenDialog` IPC handlers (`pickImageFile`, `pickJsonFile`) for profile avatars, playlist covers, and JSON backup/transition imports.
- **Native Mini Player and Tray icon fixes (`electron/main/miniPlayerWindow.ts`, `electron/main/desktopIntegration.ts`):** fixed blank/delayed rendering when reopening the frameless native mini player (`backgroundThrottling: false`, `revealWindow` with fallback) and fixed system Tray icon loading inside packaged `.asar` builds (`nativeImage.createFromBuffer`).
- **Packaged renderer and Lyrics Preview fixes (`App.tsx`, `LyricsView.tsx`):** guarded web-only `/api/app-session/stream` `EventSource` connections in Electron builds and resolved compact/expanded `Lyrics Preview` scroll and blanking glitches.

---

## [v1.0.0] - Electron Desktop App and Advanced Audio

### Desktop Architecture, Portable Packaging, and Production Protection
- **Electron migration (`electron.vite.config.ts`, `electron/main/index.ts`, `electron/preload/index.ts`, `src/main.tsx`):** `electron-vite` build pipeline for `main`, `preload`, and `renderer` while keeping web `localhost` mode 100% functional (`npm run dev`, `npm run build`, and `/api/local-library/*` endpoints). Every native API is exposed strictly via `contextBridge` on `window.deckmixElectron`, with zero `electron` or Node imports inside `src/`.
- **Native-style main window and custom window controls (`WindowControls.tsx`, `TopBar.tsx`, `NowPlayingStageView.tsx`, `LyricsView.tsx`, `electron/main/index.ts`):** custom dark title bar using `titleBarStyle: 'hidden'` and custom React window controls (`WindowControls` aligned in `TopBar` and overlaid in stage mode, automatically hidden in fullscreen to reclaim top-bar space), native `F11` fullscreen toggle, and window size, position, and maximized state persistence in `window-state.json`, validated against active displays (`screen.getAllDisplays()`).
- **True portable packaging and optional updates (`electron-builder.yml`, `electron/main/index.ts`, `electron/main/updater.ts`, `package.json`):** Windows x64 build support (`dir`, `nsis` installer, and `portable` executable), with **`DeckMix-Player-Portable-1.0.0-x64.exe`** already compiled and verified live. When `PORTABLE_EXECUTABLE_DIR` is set, the visible **`DeckMix-Data`** folder (and **`DeckMix-Data/session`**) is automatically created alongside the `.exe`, redirecting `app.setPath('userData', ...)` and `app.setPath('sessionData', ...)` without writing to `%APPDATA%`. Includes optional manual update checks via `electron-updater` in `Settings` (no auto-downloads, with clean notices in dev mode or when no feed is published).
- **Production code protection (`electron.vite.config.ts`, `package.json`, `electron-builder.yml`, `index.html`, `electron/main/index.ts`, `electron/main/miniPlayerWindow.ts`):** `main` process compiled to V8 bytecode via `electron-vite` (`build.bytecode: true` emitting `out/main/index.cjsc` + `bytecode-loader.cjs` + `index.cjs` stub with `"main": "./out/main/index.cjs"` in `package.json`), active minification with no sourcemaps in `renderer` (`sourcemap: false`, `minify: true`, `0` `.map` files), clean `app.asar` packaging (`asar: true` excluding `src/`, `electron/`, `docs/`, `scripts/`, and `*.map`), relative `./` asset URLs in `index.html` for direct disk loading, and `DevTools` (`devTools: !app.isPackaged`) plus inspection shortcuts (`F12`, `Ctrl/Cmd+Shift+I/J/C`, `Ctrl/Cmd+U`) disabled when `app.isPackaged` is `true` (while remaining enabled in `electron:dev`).

### Instant Local Streaming and 4 GB RAM Optimization
- **Direct local audio streaming (`src/audio/Deck.ts`, `src/audio/AudioEngine.ts`):** new `loadStream` path using `HTMLAudioElement` -> `MediaElementAudioSourceNode` connected to the same Web Audio `Deck` chain, cutting initial load of large FLAC tracks to `~50–150 ms` and `~2–4 MB` of buffer instead of `200+ MB` in RAM. Preserves micro-fades, EQ, Crossfade, Auto-Mix, MIDI, `AudioBuffer`-backed `startMixPreview`, and automatic fallback to `decodeTrackAudio` whenever full buffer decoding is required.
- **`deckmix://` protocol optimization (`electron/main/index.ts`):** in-memory `folderPaths` cache (`cachedBridgeFolderPaths`) to avoid repeated disk config reads, `512 KB` stream read chunks (`fs.createReadStream` with `highWaterMark: 512 * 1024`), and a `2 MB` byte-based `ReadableStream` `highWaterMark` (`2 * 1024 * 1024`).
- **Deferred and cancellable auto DSP analysis (`src/state/usePlayer.tsx`, `src/analysis/analysisQueue.ts`, `electron/main/desktopIntegration.ts`):** `3.5 s` delay before starting automatic DSP analysis for the active track, immediate cancellation when skipping quickly between songs (`cancelAutoTrackAnalysis`), and `ffmpegDspEnabled` enabled by default in Electron.

### Native Library Bridge and `userData` Storage
- **`deckmix://` protocol and IPC bridge (`electron/main/index.ts`, `electron/main/libraryBridge.ts`):** replaces the Vite dev server bridge in desktop mode with secure IPC and a custom `deckmix://` protocol supporting direct disk streaming, `HTTP Range` (`206`), CORS/ETag headers, and safe `fs.createReadStream` teardown with backpressure handling on cancel/seek.
- **Native folder picker, background watcher, and safe missing-track cleanup (`libraryBridge.ts`, `index.ts`, `vite.config.ts`, `localBridge.ts`, `usePlayer.tsx`, `SettingsView.tsx`):** cross-platform folder picker (`dialog.showOpenDialog` with `openDirectory` and `multiSelections`) without relying on `powershell.exe`, debounced background `fs.watch` (ignoring `.deckmix-*` temp files), and accessible-folder verification (`accessibleFolderIndices` / `configuredFolderCount`) to automatically prune individually missing files from connected folders (`pruneMissingTrackIds` / `deleteTracksByIds`) without wiping tracks on temporarily disconnected drives, plus a manual `"Clean up missing tracks"` button in `Settings`.
- **Atomic `userData` JSON storage (`electron/main/userDataStore.ts`, `src/storage/db.ts`):** browser-profile-independent JSON persistence under `app.getPath('userData')/storage` (`preferences.json`, `listening-stats.json`, `playlists.json`, `mix-transitions.json`, `tracks-index.json`, `analysis-cache.json`) with compact `Uint8Array` serialization, automatic one-time migration from IndexedDB, `flushAllSync()` on shutdown, and synced IndexedDB support in web browsers.
- **Native disk file actions:** `"Show in File Explorer"` (`shell.showItemInFolder`) from any track's context menu and optional `"Move duplicates to Trash"` (`shell.trashItem` with confirmation) from the Duplicate Detector.

### Windows Integrations and Native Frameless Mini Player
- **System tray, taskbar, and power management (`electron/main/desktopIntegration.ts`):** `Tray` icon with dynamic context menu and `"Minimize to tray on close"`; 4 reactive Windows thumbnail buttons (`setThumbarButtons`: Like, Previous, Play/Pause, Next); taskbar progress bar (`setProgressBar`) during scans, batch DSP analysis, and exports; native Windows `Jump List` (`app.setUserTasks` for Library, Liked Songs, Search, and Resume); configurable global shortcuts (`globalShortcut`); sleep prevention (`powerSaveBlocker`); optional launch at Windows login (`app.setLoginItemSettings` with `--start-hidden`); and single-instance lock (`requestSingleInstanceLock` + `second-instance`).
- **File associations and Discord Rich Presence (`index.ts`, `electron/main/discordRpc.ts`):** `"Open with DeckMix"` and CLI file-path support for `.flac`, `.mp3`, `.wav`, `.ogg`, `.m4a`, `.opus`, and `.aac` with session-scoped token authorization; dependency-free native IPC client for **Discord Rich Presence** (`\\?\pipe\discord-ipc-*`, `Listening` activity type, sent only on track changes, play/pause, or seek, failing silently when Discord is closed).
- **Frameless native mini player with instant reopen (`electron/main/miniPlayerWindow.ts`, `electron/main/index.ts`, `src/components/NativeMiniPlayer.tsx`, `src/components/MiniPlayerButton.tsx`):** `alwaysOnTop` frameless window (`skipTaskbar`, native rounded corners, bounds persisted in `mini-player-state.json`, isolated `?mini=1` entry without mounting the audio engine) that reuses its `BrowserWindow` via `hide()` / `show()` (`paintWhenInitiallyHidden: true`, cached icon path, and clean `destroy()` on app quit) for instant reopening. Features a 1:1 port of the web mini player design: compact bar mode (`~58–64 px`), horizontally centered drag grip, large cover with dynamic accent tint, hover controls, interactive play queue, built-in settings panel, and volume control (also added to the web PiP player).

### Advanced Audio, Hardware, Disk Tag Editor, and Remote Control
- **MP3/FLAC export and native disk tag editor via FFmpeg (`electron/main/mediaTools.ts`, `EditAnalysisModal.tsx`, `MixTransitionPanel.tsx`, `TrackList.tsx`):** `ffmpeg` detection from `PATH` or custom path in `Settings`; export of single transitions and full mixed playlists to **MP3** (configurable `128–320 kbps` bitrate) and **FLAC** alongside WAV; direct disk tag editing (title, artist, album, cover art, BPM, and key) for ID3/Vorbis/FLAC via atomic remux (`-map 0 -map_metadata 0 -c copy` preserving `attached_pic`), `music-metadata` pre-verification, clear handling of locked files (`EBUSY`/`EPERM`/`EACCES`), and in-place track updates preserving `trackId` and history.
- **Audio output selector and CUE pre-listening (`AudioEngine.ts`, `cuePlayer.ts`, `MixTransitionPanel.tsx`, `SettingsView.tsx`):** primary output device selection via `setSinkId` (speakers, headphones, HDMI, USB audio interface) with automatic fallback to default on disconnect, plus optional secondary device for on-demand **CUE preview** in Mix Mode without adding permanent nodes to the Master Web Audio graph.
- **MIDI DJ controller support (`src/audio/midiController.ts`, `SettingsView.tsx`, `usePlayer.tsx`):** Web MIDI API (`navigator.requestMIDIAccess`) integration with Electron MIDI permissions enabled and **MIDI Learn** mode for mapping physical buttons/faders to `Play/Pause`, `Crossfader`, 3-band `EQ` (`Low/Mid/High`), `Filter`, and `Hot Cues` (`C1–C4`), keeping audio stages cleanly bypassed when neutral.
- **Wi-Fi remote control with PIN and FFmpeg-assisted DSP (`electron/main/remoteControlServer.ts`, `mediaTools.ts`, `analysisQueue.ts`):** optional local-network HTTP + SSE server (off by default, configurable port with `EADDRINUSE` detection, regenerable PIN verified via SHA-256 + `timingSafeEqual`, rate limiting, and zero-dependency dark mobile web UI) and **FFmpeg-assisted DSP decoding** (`22050 Hz` mono `f32le` with duration/timeout caps and child-process cleanup) to reduce RAM spikes on large Hi-Res FLAC or `.m4a` ALAC/FLAC libraries while retaining the Web Worker pool fallback.

### Final Polish, Internal Audit, and Internationalization
- **UI polish, smooth lyrics preview, and "About DeckMix" (`SettingsView.tsx`, `LyricsView.tsx`, `vite.config.ts`, `electron.vite.config.ts`, `src/vite-env.d.ts`, `src/i18n/locales/`):** new `"About DeckMix"` section at the bottom of `Settings` displaying dynamic version (`__APP_VERSION__` + `app.getVersion()` in Electron), runtime (`Electron Desktop` / `Web (localhost)`), and `"Created by Thizi04"` credit; visible rename of `"Daily Mixes"` to **`"Deck Mixes"`**; and smooth vertical `translate3d` line animation (`useLayoutEffect` + width-scoped `ResizeObserver`) with keyed compact/expanded containers in `Lyrics Preview` to eliminate abrupt jumps and ensure scrolling in expanded mode never blanks the compact card when clicking `"Hide"`.
- **Lifecycle, memory, and Web Audio audit (`electron/main/`, `src/audio/`, `src/analysis/`, `src/state/`):** explicit teardown of SSE sockets, `deckmix://` streams, Discord pipes, `fs.watch`, timers, and `ffmpeg` child processes on cancel or `before-quit`/`will-quit`; true tone-stage bypass in `Deck.ts` during normal playback; zero-copy `Float32Array` worker transfers and strict protection of manual edits (`bpmSource`/`keySource`); IPC deduplication and redundant re-render reductions with zero visual or layout changes.
- **Full ES/EN internationalization (`src/i18n/`, `desktopIntegration.ts`, `SettingsView.tsx`, `remoteControlServer.ts`):** active language synced to Electron main (Tray, Thumbar, Jump List, native dialogs, IPC errors, and Wi-Fi remote UI), exact `1333`-key parity across Spanish and English with spelling fixes, and NFD accent-normalized Settings search covering all sections in both languages.

---

## [v0.9.0-beta] - 4 GB RAM Performance, Utilities, Fullscreen, and Bilingual ES/EN

### DSP Performance and `Out of Memory` Protection
- **Parallel and algorithmic DSP analysis optimization (`dsp.ts`, `analysisQueue.ts`, `AudioEngine.ts`, `usePlayer.tsx`):** adaptive 2–4 Web Worker pool with decode semaphore, 2:1 / 4:1 boxcar decimation, cached FFT tables and Hann windows, O(N) prefix sums, 2-stage phase search, single-pass PCM extraction for peaks/LUFS/RGB/vocals/cues, and batched updates to React.
- **RAM optimization for 4 GB PCs and Hi-Res FLAC (`AudioEngine.ts`, `Deck.ts`, `audioSource.ts`, `analysisQueue.ts`, `PlayerBar.tsx`, `MiniPlayerButton.tsx`):** `navigator.deviceMemory`-aware LRU cache and Worker concurrency, real in-flight download cancellation via `AbortController`, lazy preloading ~22 s before transition instead of `t = 0s`, buffer/fade cleanup before `decodeAudioData`, and `seek` fixes for rapid clicks, mouse release outside the slider, and paused states.

### Expanded Stage View, Adaptive Mini Player, and Live Visualizer
- **Unified Expanded Stage View and Fullscreen (`NowPlayingStageView.tsx`, `LyricsView.tsx`, `NowPlayingPanel.tsx`, `PlayerBar.tsx`, `App.tsx`):** Theater Mode and Fullscreen shared across Cover, About the artist, and Lyrics with a shared top bar, related-tracks carousel, details/credits card, and always-pinned bottom `PlayerBar`.
- **Live frequency spectrum visualizer (`StageSpectrumVisualizer.tsx`, `NowPlayingStageView.tsx`):** real-time `AnalyserNode` spectrum visualizer integrated into Theater Mode and Fullscreen with a persistent top-bar toggle and automatic animation pause when paused or tab-hidden.
- **Resizable floating mini player (`MiniPlayerButton.tsx`):** adaptive 3-mode layout based on window size (`Compact bar`, `Large cover` with hover controls including `Shuffle` and `Repeat`, and interactive `Play queue` when enlarged); integrated `Miniplayer settings` panel with persistent `Background color` (full cover tint) and `Play queue` toggles, plus window dimensions remembered in `localStorage`.

### Backup, Duplicate Detector, Desktop Polish, Portability, and i18n
- **Full profile backup and Duplicate Detector (`backup.ts`, `duplicates.ts`, `db.ts`, `usePlayer.tsx`, `SettingsView.tsx`):**
  - Portable `.json` export and import from `Settings` (playlists, folders, Liked Songs, Wrapped stats, Hot Cues, BPM/Key/LUFS analyses, Mix transitions, Master EQ, and user preferences without copying music files).
  - Smart duplicate song detector (title/artist normalization and automatic ranking by availability, lossless format, and bitrate) with non-destructive hiding, optional restore, and automatic merging of playlists/likes/queues to the chosen copy.
- **Zoom level, session restore, and shortcuts (`AudioEngine.ts`, `usePlayer.tsx`, `useKeyboardShortcuts.ts`, `PlayerBar.tsx`, `TrackRow.tsx`, `SearchView.tsx`, `SettingsView.tsx`, `db.ts`, `backup.ts`):**
  - Visual **Zoom level** selector (`70 %`–`130 %`) in `Settings` with layout previews (`Compact`, `Default`, `Spacious`), local persistence + JSON backup inclusion, and `Ctrl + +` / `Ctrl + -` / `Ctrl + 0` shortcuts.
  - Lightweight restore of the last paused song and timestamp on reopen without decoding audio until Play is pressed, mouse-wheel volume control + percentage tooltip, dynamic `document.title`, unified right-click context menu in `TrackRow` and `SearchView`, and expanded shortcuts (`N`/`P`, `J`/`K`, `0`–`9`, `S`, `R`, `L`, `F`, `G`, `Q`).
- **Cross-platform portability and safety (`vite.config.ts`, `localBridge.ts`, `usePlayer.tsx`, `HomeView.tsx`, `TrackList.tsx`, `SettingsView.tsx`, `LinkFolderModal.tsx`, `README.md`):** `process.platform !== 'win32'` guard on `/api/local-library/pick` and `openWindowsFolderPicker` with `nativePickerAvailable` flag for clean fallback to browser folder picker or manual path entry on macOS/Linux; portable launch scripts with auto-build of `dist/` and clear documentation for cross-platform web mode and optional Windows desktop launcher.
- **Bilingual interface and GitHub documentation (`src/i18n/`, `usePlayer.tsx`, `SettingsView.tsx`, `README.md`, `README.es.md`, `CHANGELOG.md`, `CHANGELOG.es.md`):** strictly typed modular EN/ES internationalization system with browser-language auto-detection on first launch (English fallback), live switching from `Settings -> Language` persisted across `localStorage`, IndexedDB, and JSON backups, dynamic locale formatting, and full EN/ES GitHub documentation parity.

---

## [v0.8.0-beta] - Synced Lyrics, Local Profile, and Web Desktop Experience

### Synced Lyrics and Local Metadata
- **Synced lyrics with strict duration validation (`src/types/lyrics.ts`, `lrcParser.ts`, `lrclibClient.ts`, `useTrackLyrics.ts`, `LyricsView.tsx`, `coverColor.ts`):** priority local `.lrc` sidecar loading, LRCLIB search with `±4.0s` tolerance, manual offset (`-0.5s / +0.5s`), `Versions` selector, IndexedDB v4 cache (`lyrics`), cover-tinted `Lyrics Preview` card, and Theater / Fullscreen modes.
- **Official app icon, fast cover/metadata extraction, and multiple pinned folders (`public/`, `vite.config.ts`, `localBridge.ts`, `libraryScanner.ts`, `SettingsView.tsx`, `LinkFolderModal.tsx`, `camelotColors.ts`, `TrackRow.tsx`):**
  - SVG/PNG favicon and `manifest.webmanifest`; `/api/local-library/metadata` and `/api/local-library/cover` endpoints to read tags and covers directly in Node; automatic background repair and manual `Repair covers`.
  - Support for multiple persistent pinned folders on disk (`folderPaths`) and Spotify-style `BPM · Key` columns visible only when `Mix` is active.

### Profile, Recommendations, and Auto Playlists
- **Editable profile, centralized settings, and local intelligence (`ProfileView.tsx`, `SettingsView.tsx`, `UserAvatar.tsx`, `recommendations.ts`, `HomeView.tsx`, `Sidebar.tsx`, `NowPlayingPanel.tsx`):**
  - Contextual avatar with `Profile` view, photo/initials/color editing, real listening Wrapped stats (`listening-stats`), and automatic Daily Mixes (`Made for you`).
  - Collapsible playlist folders in `Your Library`, custom playlist covers and descriptions (`PlaylistModal.tsx`), dynamic-color view headers, `About the artist` / `File details` cards, smart Autoplay, Song Radio, and recommended suggestions at the bottom of playlists.
- **Auto-creation and updating of playlists from plays (`AutoPlaylistModal.tsx`, `recommendations.ts`, `Sidebar.tsx`, `SettingsView.tsx`, `TrackList.tsx`):** `top-played`, `monthly-top`, `recent-vibe`, and `recently-played` rules with live preview, optional continuous auto-update mode, and per-playlist controls (`Update playlist`, `Auto: ON/OFF`).

### Web Desktop Audio and Chrome `--app` Launcher
- **Desktop-style playback (`AudioEngine.ts`, `masterEq.ts`, `MiniPlayerButton.tsx`, `PlayerBar.tsx`):** Media Session API, floating Mini Player (Document Picture-in-Picture), configurable simple crossfade (`0–12 s`, 4 s default), 6-band Master Graphic EQ, Mono mode, 3 Auto-Gain LUFS levels, and Sleep Timer with smooth fade-out.
- **Fast transport micro-fades (`Deck.ts`, `AudioEngine.ts`, `usePlayer.tsx`):** `sourceGain` stage with `fadingSources` queue and automatic ducking to smooth pause (`0.22 s`), resume (`0.18 s`), track skips (`0.22 s` / `0.16 s`), and `seek` (`0.14 s` / `0.12 s`) without UI freezes.
- **Chrome `--app` launcher with clean auto-shutdown (`scripts/DeckMix.bat`, `scripts/launch-deckmix.vbs`, `scripts/run-deckmix.ps1`, `vite.config.ts`, `App.tsx`):** silent Chrome app-mode launch and automatic server shutdown via SSE (`/api/app-session/stream`) when closing the window.

---

## [v0.7.0-beta] - Pro Audio Engine and Algorithmic Transitions

- **Real-time Key Lock / Master Tempo and harmonic Key Shift (`public/worklets/pitchShifterProcessor.js`, `musicalKey.ts`, `mixMath.ts`, `Deck.ts`, `AudioEngine.ts`, `MixTransitionPanel.tsx`):** `AudioWorkletProcessor` (`deckmix-pitch-shifter`) with Hann-windowed circular buffer to compensate tempo changes up to `±16%` without altering pitch, plus per-deck `Key Shift` selector (`-4` to `+4` semitones).
- **Offline `.wav` audio export (`offlineMixRenderer.ts`, `MixTransitionPanel.tsx`, `TrackList.tsx`):** `OfflineAudioContext` renderer to download 16-bit stereo PCM for both single transitions (`Export WAV`) and full mixed playlists (`Export Mix WAV`).
- **Mid/Side vocal / instrumental isolation (`mix.ts`, `Deck.ts`, `offlineMixRenderer.ts`, `MixTransitionPanel.tsx`):** clickless parallel Web Audio stage per deck with `Normal`, `Instrumental Only`, and `Vocals Only` modes.
- **Per-list `Mix` toggle and Web Audio re-engineering (`Deck.ts`, `AudioEngine.ts`, `TrackList.tsx`, `usePlayer.tsx`):** decoded `AudioBuffer` LRU cache (up to 12 tracks / 320 MB), on-demand connection of heavy subgraphs, precomputed Hann table in `pitchShifterProcessor.js`, visual duration trimming at `outPointSecA`, and per-list `Mix` state (`mixedScopeIds`).
- **Algorithmic transition integration (`dsp.ts`, `mixMath.ts`, `presets.ts`, `MixTransitionPanel.tsx`):** `Lambda(i, j)` cost matrix (`findOptimalMixPoints`) with 128-bin vocal profile, `Wave`, `Melt`, and `Slam` presets, `12` bars, per-track sub-presets, and smart ordering with LUFS and spectral energy continuity.

---

## [v0.6.0-beta] - FX, Musical Intelligence, and Real-Time Precision

*(Previously Phases 2.8 and 2.9)*

- **Direct node editing on top of both waveforms (`WaveformCanvas.tsx`, `MixTransitionPanel.tsx`):** active when selecting `Volume`, `Equalizer`, `Filter`, or `Effects` (click on curve to add a node, drag point to edit, double-click to remove).
- **Parallel FX chain (`Deck.ts`, `AudioEngine.ts`, `presets.ts`, `TrackList.tsx`):** post-EQ/Filter stage independent of `deckGain` to preserve tails when dry drops to `0%`: BPM-synced `DelayNode` (`1/8`, `1/4`, `3/8`, `1/2` beat) with feedback `highpass` for **Echo Out**, and synthetic stereo `ConvolverNode` for **Reverb Wash**, plus the `Effects` card and dedicated presets.
- **Integrated Auto-Gain (-14 LUFS) (`dsp.ts`, `Deck.ts`, `AudioEngine.ts`, `usePlayer.tsx`):** gated BS.1770 LUFS computation in Web Worker (`CURRENT_ANALYSIS_VERSION = 2`), automatic `setInputTrimDb` trim on `inputGain` (`±8 dB` range), and persistent `autoGainEnabled` preference.
- **Phrase detection (`IN`, `DROP`, `OUT`) and Smart Auto-Mix (`dsp.ts`, `mixMath.ts`, `usePlayer.tsx`):** structural detection via low/mid energy in Worker, Outro/Intro-anchored `outPointSecA`/`inPointSecB` calculation, and automatic preset selection (`selectSmartAutoPreset`).
- **RGB frequency waveforms + pinpoint zoom (`WaveformCanvas.tsx`):** `[low, mid, high]` band extraction in Worker (`waveformRgbBands`), spectral rendering with `RGB` toggle, and `1.0x`–`4.0x` zoom (`Ctrl+Wheel` or `- / +` buttons).
- **Hot Cues (`C1`–`C4`) and Master/Limiter VU meter (`db.ts`, `MasterMeter.tsx`, `MixTransitionPanel.tsx`):** IndexedDB persistence (`saveTrackHotCues`), visual markers over waveforms, and real-time RMS/Peak meter + Master limiter reduction (`DynamicsCompressorNode.reduction`).

---

## [v0.5.0-beta] - Web-Player Feature Parity

*(Previously Phase 2.7)*

- **Real interactive queue (`manualQueueIds`, `upNextTrackIds`):** integrated into `usePlayer.tsx` with `Queue` / `Now Playing` tabs in `NowPlayingPanel.tsx` and a dedicated button in `PlayerBar.tsx`.
- **Persistent favorites, pins, and history (`db.ts`):** `Liked Songs` (`likedTrackIds`), pinned playlists (`pinnedPlaylistIds`), and real `Recently Played` history (`recentlyPlayedTrackIds`) stored in IndexedDB `UserPreferences`.
- **Unified navigation and search (`SearchView.tsx`, `HomeView.tsx`, `Sidebar.tsx`):** unified `SearchView.tsx` (`Ctrl+K` / `/`) with local categories by BPM/Camelot/genre, `Top Result`, and entity carousels; `HomeView.tsx` with arrow-scrollable horizontal shelves; and `Sidebar.tsx` collapsible to mini mode (`76px`) with pins and `List | Compact` density.
- **Detailed library views (`TrackList.tsx`, `TrackRow.tsx`):** sticky topbar on scroll, Album view without repeated covers + cover lightbox + `"More by <Artist>"`, Artist view (`Popular`, `Discography`, `Appears On`), sortable column headers, collapsible magnifying-glass filter, `List | Compact` density, unified right-click / `...` context menu, animated `.eq-bars` equalizer, and harmonic `Smart Reorder` by BPM + Camelot.
- **Playback and mix-editor polish (`PlayerBar.tsx`, `MixTransitionPanel.tsx`):** total/remaining duration toggle in `PlayerBar.tsx`, visible harmonic/BPM delta chip, `< Previous | Next >` navigation, and dirty-state confirmation.

---

## [v0.4.0-beta] - Desktop UI Redesign, Local Bridge, and Mix Mode Interface

*(Previously Phases 2.5 and 2.6)*

- **Dark desktop shell (`TopBar.tsx`, `Sidebar.tsx`, `HomeView.tsx`, `TrackList.tsx`, `CatalogGrid.tsx`, `NowPlayingPanel.tsx`, `PlayerBar.tsx`, `App.tsx`):** Spotify-Desktop-inspired dark shell with drag-resizable left and right side panels (persisted in `localStorage`) and responsive cards with `line-clamp-2` to prevent title clipping.
- **Persistent Vite local bridge (`vite.config.ts`, `src/storage/localBridge.ts`, `src/utils/audioSource.ts`, `src/components/LinkFolderModal.tsx`):** loopback-restricted (`127.0.0.1` / `::1`) `/api/local-library/*` endpoints and configuration in `.deckmix-library.json` (Git-ignored) to link folders without copying files and automatically reconnect across page reloads, paired with a native folder picker (`POST /api/local-library/pick`) and `<input webkitdirectory>` / drag & drop fallbacks.
- **Spotify-style Mix Mode panel (`MixTransitionPanel.tsx`, `TrackList.tsx`, `WaveformCanvas.tsx`):** enabling `Mix` and clicking any `Auto` / `Custom` pill between tracks opens the `Edit transition [Beta]` side panel alongside the playlist (with an option to expand to center), featuring a stylized `spotify` dual waveform with overlaid Volume (cyan), EQ (yellow), and Filter (magenta) curves, centered circular preview Play/Pause button, interactive bottom `N bars` pill (`2 / 4 / 8 / 12 / 16`), preset carousel (`Custom`, `Auto`, `Fade`, `Rise`, `Blend`, `Bass Swap`, `Filter Fade`), and color-coded accordion cards for `Volume`, `Equalizer` (Low/Mid/High), `Filter` (LPF/HPF + `Q`), and `Tempo & timing`.

---

## [v0.3.0-beta] - Manual Mix Mode

*(Previously Phase 2)*

- **Visual dual-waveform Canvas 2D editor (`WaveformCanvas.tsx`):** beat/bar grid (2, 4, 8, or 16 bars), horizontal dragging with magnetic snap (`Bar`, `Beat`, `Free`, or `Shift`), fine nudge, and direct seek on the top ruler.
- **Interactive curve editor:** Volume (equal-power), 3-band EQ (`250 Hz`, `1 kHz`, `3.8 kHz`), and logarithmic LPF/HPF filters with configurable `Q` resonance; classic presets (`Fade`, `Blend`, `Bass Swap`, `Rise`, `Filter Fade`) and custom mode.
- **Lookahead engine and Auto-Mix (`AudioEngine.ts`, `Deck.ts`):** soft Master `DynamicsCompressorNode` limiter, sample-accurate lookahead scheduler (`30 ms` / `160 ms` window), coupled-ramp beatmatching (`±6%`, `0.5x/2x` support), exact accumulated position integrator under variable `playbackRate`, analytical curve evaluation for pause/seek mid-transition, and playlist Auto-Mix mode.
- **Transition persistence (`db.ts`, `mixStorage.ts`):** IndexedDB v3 persistence (`mixTransitions`) and portable JSON export/import.

---

## [v0.2.0-alpha] - Music Analysis DSP and Library Navigation

*(Previously Phase 1.5)*

- **Artists and Albums navigation (`CatalogGrid.tsx`):** dedicated views, sorting by title, artist, album, track number, duration, BPM, and Camelot key, plus expanded support for FLAC, M4A, MP4, AAC, OPUS, and WEBM.
- **`analysis` store in IndexedDB v2 (`DB_VERSION = 2`):** stores `waveformPeaks` as `Uint8Array`, `analysisVersion`, `bpmSource` / `keySource`, and strict protection against overwriting manual edits or valid tags.
- **Pure TypeScript DSP engine inside Web Worker (`src/analysis/dsp.ts`, `analysisWorker.ts`, `analysisQueue.ts`):** safe mono copy for Transferables without detaching the active `AudioBuffer`, full-duration `waveformPeaks`, center-window slicing for long tracks, anti-aliasing low-pass filter + downsampling, BPM + `firstBeatOffsetSec` detection, and 36->12 HPCP with logarithmic compression + Krumhansl-Schmuckler and Camelot wheel mapping.
- **Priority queue and manual editing (`analysisQueue.ts`, `EditAnalysisModal.tsx`):** priority FIFO queue favoring the playing/loading track and protected manual editing.

---

## [v0.1.0-alpha] - Initial Local Player

*(Previously Phase 1)*

- **Local file loading:** MP3, WAV, FLAC, and OGG support via File System Access API (`showDirectoryPicker`), `<input webkitdirectory>`, or drag & drop.
- **Indexing and playlists in IndexedDB:** tag and album cover reading with `music-metadata` (`parseBlob`, `selectCover`), incremental scanning in IndexedDB (`idb`) using content fingerprints (`fingerprint.ts`), and local playlist management with drag & drop reordering (`@dnd-kit/sortable`) plus move-up/move-down controls.
- **Dual-Deck engine (`AudioEngine.ts`, `Deck.ts`):** React-decoupled engine with two `Deck` instances (`A` and `B`), on-demand decoding, and background preloading of the next track.
- **Initial dark interface:** Sidebar, central track list with quick search, and a fixed bottom playback bar with keyboard shortcuts.
