[Read in English](./CHANGELOG.md)

# Historial de cambios (DeckMix Player)

Registro cronológico de versiones y mejoras implementadas en DeckMix Player, ordenado de la versión más reciente a la más antigua.

## [v1.0.0] - Aplicación de Escritorio Electron y Audio Avanzado

### Arquitectura Desktop, Empaquetado Portable y Protección de Producción
- **Migración a Electron (`electron.vite.config.ts`, `electron/main/index.ts`, `electron/preload/index.ts`, `src/main.tsx`):** compilación de procesos `main`, `preload` y `renderer` mediante `electron-vite` manteniendo 100 % operativo el modo web en `localhost` (`npm run dev`, `npm run build` y endpoints `/api/local-library/*`). Toda API nativa se expone exclusivamente por `contextBridge` en `window.deckmixElectron`, con cero imports de `electron` o módulos de Node en `src/`.
- **Ventana principal estilo app nativa y controles propios (`WindowControls.tsx`, `TopBar.tsx`, `NowPlayingStageView.tsx`, `LyricsView.tsx`, `electron/main/index.ts`):** barra de título oscura personalizada con `titleBarStyle: 'hidden'` y botones de ventana propios (`WindowControls` alineados en `TopBar` y superpuestos en modo ampliado, ocultos automáticamente en pantalla completa para aprovechar todo el espacio), atajo nativo `F11` y persistencia de tamaño, posición y estado maximizado en `window-state.json` validada contra los monitores activos (`screen.getAllDisplays()`).
- **Empaquetado portable real y actualizaciones opcionales (`electron-builder.yml`, `electron/main/index.ts`, `electron/main/updater.ts`, `package.json`):** soporte para builds Windows x64 (`dir`, instalador `nsis` y ejecutable `portable`), con el ejecutable **`DeckMix-Player-Portable-1.0.0-x64.exe`** ya compilado y verificado en vivo. Cuando `PORTABLE_EXECUTABLE_DIR` está presente, se crea automáticamente la carpeta visible **`DeckMix-Data`** (y **`DeckMix-Data/session`**) junto al `.exe` redirigiendo `app.setPath('userData', ...)` y `app.setPath('sessionData', ...)` sin escribir en `%APPDATA%`. Incluye verificación manual opcional con `electron-updater` desde `Configuración` (sin descargas automáticas y con avisos limpios en modo desarrollo o sin feed publicado).
- **Protección de código en producción (`electron.vite.config.ts`, `package.json`, `electron-builder.yml`, `index.html`, `electron/main/index.ts`, `electron/main/miniPlayerWindow.ts`):** compilación del proceso `main` a bytecode V8 con `electron-vite` (`build.bytecode: true` generando `out/main/index.cjsc` + `bytecode-loader.cjs` + stub `index.cjs` con entrada `"main": "./out/main/index.cjs"`), minificación activa sin sourcemaps en `renderer` (`sourcemap: false`, `minify: true`, `0` archivos `.map`), empaquetado limpio en `app.asar` (`asar: true` excluyendo `src/`, `electron/`, `docs/`, `scripts/` y `*.map`), rutas relativas `./` en `index.html` para carga directa desde disco, y bloqueo de `DevTools` (`devTools: !app.isPackaged`) y atajos de inspección (`F12`, `Ctrl/Cmd+Shift+I/J/C`, `Ctrl/Cmd+U`) cuando `app.isPackaged` es `true` (manteniéndolos habilitados en `electron:dev`).

### Streaming Local Instantáneo y Optimización para 4 GB RAM
- **Reproducción por streaming directo (`src/audio/Deck.ts`, `src/audio/AudioEngine.ts`):** nuevo método `loadStream` basado en `HTMLAudioElement` -> `MediaElementAudioSourceNode` conectado a la misma cadena Web Audio del `Deck`, reduciendo la carga inicial de archivos FLAC pesados a `~50–150 ms` y `~2–4 MB` de buffer en lugar de `200+ MB` en RAM. Mantiene micro-fades, EQ, Crossfade, Auto-Mix, MIDI, `startMixPreview` en `AudioBuffer` y fallback automático a `decodeTrackAudio` si la pista requiere decodificación completa.
- **Optimización del protocolo `deckmix://` (`electron/main/index.ts`):** caché en memoria de `folderPaths` (`cachedBridgeFolderPaths`) para evitar lecturas repetidas de configuración en disco, lectura en bloques de `512 KB` (`fs.createReadStream` con `highWaterMark: 512 * 1024`) y `ReadableStream` con `highWaterMark` por bytes de `2 MB` (`2 * 1024 * 1024`).
- **Análisis DSP diferido y cancelable (`src/state/usePlayer.tsx`, `src/analysis/analysisQueue.ts`, `electron/main/desktopIntegration.ts`):** espera de `3.5 s` antes de lanzar el análisis DSP automático de la pista en curso, cancelación inmediata al saltar rápido entre canciones (`cancelAutoTrackAnalysis`) y `ffmpegDspEnabled` activado por defecto en Electron.

### Biblioteca Nativa y Almacenamiento Persistente en `userData`
- **Protocolo `deckmix://` y puente IPC (`electron/main/index.ts`, `electron/main/libraryBridge.ts`):** reemplazo del servidor Vite en modo escritorio por IPC seguro y protocolo custom `deckmix://` con streaming directo desde disco, soporte `HTTP Range` (`206`), headers CORS/ETag y destrucción segura de `fs.createReadStream` con control de backpressure al cancelar o hacer `seek`.
- **Selector nativo, watcher en tiempo real y limpieza segura de pistas sin archivo (`libraryBridge.ts`, `index.ts`, `vite.config.ts`, `localBridge.ts`, `usePlayer.tsx`, `SettingsView.tsx`):** selector multiplataforma de carpetas (`dialog.showOpenDialog` con `openDirectory` y `multiSelections`) sin depender de `powershell.exe`, `fs.watch` en segundo plano con debounce (ignorando temporales `.deckmix-*`) y detección de carpetas accesibles (`accessibleFolderIndices` / `configuredFolderCount`) para eliminar automáticamente archivos individuales faltantes de carpetas conectadas (`pruneMissingTrackIds` / `deleteTracksByIds`) sin borrar temas de unidades desconectadas, más botón manual `"Limpiar pistas sin archivo"` en `Configuración`.
- **Almacenamiento atómico en `userData` (`electron/main/userDataStore.ts`, `src/storage/db.ts`):** persistencia JSON independiente del perfil del navegador en `app.getPath('userData')/storage` (`preferences.json`, `listening-stats.json`, `playlists.json`, `mix-transitions.json`, `tracks-index.json`, `analysis-cache.json`) con serialización compacta de `Uint8Array`, migración automática inicial desde IndexedDB, `flushAllSync()` al cerrar y respaldo sincronizado en IndexedDB para navegador web.
- **Acciones reales sobre archivos en disco:** acceso directo `"Mostrar en el Explorador de archivos"` (`shell.showItemInFolder`) desde el menú contextual de cualquier canción y opción de mover copias repetidas a la Papelera (`shell.trashItem` con confirmación previa) desde el Detector de duplicados.

### Integraciones de Windows y Mini Reproductor Nativo
- **Bandeja del sistema, barra de tareas y energía (`electron/main/desktopIntegration.ts`):** icono en `Tray` con menú contextual dinámico y opción `"Minimizar a la bandeja al cerrar"`; 4 botones reactivos en la vista previa de Windows (`setThumbarButtons`: Me gusta, Anterior, Reproducir/Pausar, Siguiente); barra de progreso en el icono (`setProgressBar`) durante escaneos, análisis DSP por lotes y exportaciones; `Jump List` nativa (`app.setUserTasks` para Biblioteca, Tus me gusta, Buscar y Reanudar); atajos globales configurables (`globalShortcut`); prevención de suspensión (`powerSaveBlocker`); inicio automático con Windows (`app.setLoginItemSettings` con `--start-hidden`); y lock de instancia única (`requestSingleInstanceLock` + `second-instance`).
- **Asociación de archivos y Discord Rich Presence (`index.ts`, `electron/main/discordRpc.ts`):** soporte para `"Abrir con DeckMix"` y rutas CLI en `.flac`, `.mp3`, `.wav`, `.ogg`, `.m4a`, `.opus` y `.aac` con tokens de acceso por sesión; cliente IPC nativo sin dependencias para **Discord Rich Presence** (`\\?\pipe\discord-ipc-*`, actividad tipo `Listening`, enviado solo al cambiar de pista, play/pause o `seek` y fallo silencioso si Discord está cerrado).
- **Mini reproductor flotante nativo sin marco con apertura rápida (`electron/main/miniPlayerWindow.ts`, `electron/main/index.ts`, `src/components/NativeMiniPlayer.tsx`, `src/components/MiniPlayerButton.tsx`):** ventana `alwaysOnTop` frameless (`skipTaskbar`, esquinas redondeadas nativas, dimensiones persistentes en `mini-player-state.json`, carga aislada por `?mini=1` sin montar el motor de audio) que reutiliza la `BrowserWindow` mediante `hide()` / `show()` (`paintWhenInitiallyHidden: true`, icono cacheado y `destroy()` limpio al cerrar la app) para reabrir de forma instantánea. Incluye el diseño portado 1:1 desde el mini reproductor web: modo barra compacta (`~58–64 px`), grip de arrastre centrado horizontalmente, portada grande con tinte dinámico, controles hover, fila de reproducción interactiva, ajustes integrados y control de volumen (añadido también al PiP web).

### Audio Avanzado, Hardware, Metadatos en Disco y Control Remoto
- **Exportación MP3/FLAC y editor nativo de tags con FFmpeg (`electron/main/mediaTools.ts`, `EditAnalysisModal.tsx`, `MixTransitionPanel.tsx`, `TrackList.tsx`):** detección de `ffmpeg` en `PATH` o ruta personalizada en `Configuración`; exportación de transiciones y playlists mixeadas a **MP3** (bitrate configurable `128–320 kbps`) y **FLAC** además de WAV; edición directa de título, artista, álbum, portada, BPM y tonalidad sobre el archivo real (ID3/Vorbis/FLAC) mediante remux atómico (`-map 0 -map_metadata 0 -c copy` preservando `attached_pic`), verificación previa con `music-metadata`, manejo claro de archivos bloqueados (`EBUSY`/`EPERM`/`EACCES`) y actualización in-place que conserva el `trackId` y el historial.
- **Selector de salida de audio y preescucha CUE (`AudioEngine.ts`, `cuePlayer.ts`, `MixTransitionPanel.tsx`, `SettingsView.tsx`):** selección del dispositivo de salida principal mediante `setSinkId` (parlantes, auriculares, HDMI, interfaz USB) con fallback automático al dispositivo por defecto si se desconecta, y salida secundaria opcional para **preescucha CUE** bajo demanda en Modo Mix sin sumar nodos permanentes al grafo Master.
- **Soporte para controladoras MIDI DJ (`src/audio/midiController.ts`, `SettingsView.tsx`, `usePlayer.tsx`):** integración de Web MIDI API (`navigator.requestMIDIAccess`) con permisos habilitados en Electron y modo **MIDI Learn** para asignar botones/faders físicos a `Play/Pause`, `Crossfader`, `EQ` de 3 bandas (`Low/Mid/High`), `Filtro` y `Hot Cues` (`C1–C4`), manteniendo el grafo en bypass neutro cuando los controles no se usan.
- **Control remoto Wi-Fi con PIN y análisis DSP asistido por FFmpeg (`electron/main/remoteControlServer.ts`, `mediaTools.ts`, `analysisQueue.ts`):** servidor HTTP + SSE opcional en red local (desactivado por defecto, puerto configurable con detección de `EADDRINUSE`, PIN regenerable verificado con SHA-256 + `timingSafeEqual`, rate limit y UI móvil oscura sin dependencias externas) y decodificación **DSP asistida por FFmpeg** (`22050 Hz` mono `f32le` con tope de duración/timeout y cierre de procesos hijos) para reducir picos de RAM en bibliotecas FLAC Hi-Res o `.m4a` ALAC/FLAC manteniendo intacto el pool de Web Workers como fallback.

### Pulido Final, Auditoría Interna e Internacionalización
- **Pulido de interfaz, letras y "Acerca de DeckMix" (`SettingsView.tsx`, `LyricsView.tsx`, `vite.config.ts`, `electron.vite.config.ts`, `src/vite-env.d.ts`, `src/i18n/locales/`):** nueva sección `"Acerca de DeckMix"` al final de `Configuración` con versión dinámica (`__APP_VERSION__` + `app.getVersion()` en Electron), entorno (`Electron Desktop` / `Web (localhost)`) y crédito `"Creado por Thizi04"`; renombre visible de `"Daily Mixes"` a **`"Deck Mixes"`**; y desplazamiento vertical fluido (`translate3d` sincronizado en `useLayoutEffect` con `ResizeObserver` dedicado) más nodos separados por `key` en `"Muestra de la letra"` para evitar saltos bruscos y asegurar que el scroll en modo expandido no vacíe la vista compacta al pulsar `"Ocultar"`.
- **Auditoría de ciclo de vida, memoria y Web Audio (`electron/main/`, `src/audio/`, `src/analysis/`, `src/state/`):** cierre explícito de sockets SSE, streams `deckmix://`, pipes Discord, `fs.watch`, timers y procesos `ffmpeg` al cancelar o en `before-quit`/`will-quit`; bypass real de etapas de tono en `Deck.ts` durante reproducción normal; transferencia zero-copy de `Float32Array` a Workers DSP y protección estricta de ediciones manuales (`bpmSource`/`keySource`); deduplicación IPC y reducción de re-renders redundantes sin alterar diseño ni efectos visuales.
- **Internacionalización completa ES/EN (`src/i18n/`, `desktopIntegration.ts`, `SettingsView.tsx`, `remoteControlServer.ts`):** sincronización del idioma activo hacia Electron main (Tray, Thumbar, Jump List, diálogos nativos, errores IPC y cliente remoto Wi-Fi), paridad exacta de `1333` claves entre español e inglés con corrección ortográfica y buscador interno de `Configuración` con normalización NFD para encontrar todas las secciones en ambos idiomas.

---

## [v0.9.0-beta] - Rendimiento 4 GB RAM, Utilidades, Pantalla Completa y Bilingüe ES/EN

### Rendimiento DSP y Protección contra `Out of Memory`
- **Optimización paralela y algorítmica del análisis DSP (`dsp.ts`, `analysisQueue.ts`, `AudioEngine.ts`, `usePlayer.tsx`):** pool adaptativo de 2 a 4 Web Workers con semáforo de decodificación, decimación boxcar `2:1` / `4:1`, tablas FFT y ventanas Hann cacheadas, suma prefija `O(N)`, búsqueda de fase en 2 etapas, pasado único PCM para picos/LUFS/RGB/voz/cues y batching de resultados hacia React.
- **Gestión de RAM para equipos de 4 GB y FLAC Hi-Res (`AudioEngine.ts`, `Deck.ts`, `audioSource.ts`, `analysisQueue.ts`, `PlayerBar.tsx`, `MiniPlayerButton.tsx`):** caché LRU y concurrencia de Workers adaptadas a `navigator.deviceMemory`, cancelación real de descargas en curso mediante `AbortController`, precarga perezosa ~22 s antes de la transición en lugar de `t = 0s`, liberación previa de buffers/fades antes de `decodeAudioData` y corrección del `seek` ante clics rápidos, suelta fuera del slider y estado pausado.

### Vista Ampliada, Mini Reproductor Adaptativo y Visualizador
- **Vista ampliada y Pantalla Completa unificada (`NowPlayingStageView.tsx`, `LyricsView.tsx`, `NowPlayingPanel.tsx`, `PlayerBar.tsx`, `App.tsx`):** Modo Cine y Pantalla completa compartidos entre Portada, Acerca del artista y Letras con barra superior compartida, carrusel de temas relacionados, tarjeta de detalles/créditos y `PlayerBar` siempre fija abajo.
- **Visualizador de espectro en vivo (`StageSpectrumVisualizer.tsx`, `NowPlayingStageView.tsx`):** visualizador de espectro de frecuencias en tiempo real mediante `AnalyserNode` integrado en Modo Cine y Pantalla completa, con toggle persistente en la barra superior y pausa automática de animación al pausar o cambiar de pestaña.
- **Mini reproductor flotante redimensionable (`MiniPlayerButton.tsx`):** diseño adaptativo en 3 modos según tamaño de ventana (`Barra compacta`, `Portada grande` con controles en hover incluyendo `Aleatorio` y `Repetir`, y `Fila de reproducción` interactiva al agrandar); panel integrado `Configuración del minirreproductor` con toggles persistentes `Color de fondo` (tinte completo por carátula) y `Fila de reproducción`, más memoria de dimensiones en `localStorage`.

### Respaldo, Duplicados, Pulido Desktop, Portabilidad e i18n
- **Respaldo completo del perfil y Detector de duplicados (`backup.ts`, `duplicates.ts`, `db.ts`, `usePlayer.tsx`, `SettingsView.tsx`):**
  - Exportación e importación portátil en `.json` desde `Configuración` (playlists, carpetas, Me gusta, estadísticas Wrapped, Hot Cues, análisis BPM/Tono/LUFS, transiciones Mix, EQ Master y preferencias de usuario sin copiar archivos de audio).
  - Detector inteligente de canciones duplicadas (normalización de título/artista y ranking automático por disponibilidad, formato lossless y bitrate) con ocultamiento no destructivo, restauración opcional y unificación automática de playlists/Me gusta/colas hacia la copia elegida.
- **Nivel de zoom, restauración de sesión y atajos (`AudioEngine.ts`, `usePlayer.tsx`, `useKeyboardShortcuts.ts`, `PlayerBar.tsx`, `TrackRow.tsx`, `SearchView.tsx`, `SettingsView.tsx`, `db.ts`, `backup.ts`):**
  - Selector visual **Nivel de zoom** (`70 %`–`130 %`) en `Configuración` con vistas previas (`Compacto`, `Predeterminado`, `Espaciado`), persistencia local + inclusión en respaldos JSON y atajos `Ctrl + +` / `Ctrl + -` / `Ctrl + 0`.
  - Restauración ligera de la última canción y segundo pausado al reabrir DeckMix sin decodificar audio hasta presionar Play, rueda del mouse + tooltip de porcentaje en el volumen, título dinámico de ventana (`document.title`), clic derecho contextual unificado en `TrackRow` y `SearchView`, y atajos ampliados (`N`/`P`, `J`/`K`, `0`–`9`, `S`, `R`, `L`, `F`, `G`, `Q`).
- **Portabilidad y seguridad multiplataforma (`vite.config.ts`, `localBridge.ts`, `usePlayer.tsx`, `HomeView.tsx`, `TrackList.tsx`, `SettingsView.tsx`, `LinkFolderModal.tsx`, `README.md`):** guard `process.platform !== 'win32'` en `/api/local-library/pick` y `openWindowsFolderPicker` con bandera `nativePickerAvailable` para hacer fallback limpio al selector del navegador o ruta manual en macOS/Linux; scripts de lanzamiento portables con auto-build de `dist/` y documentación clara de modo web multiplataforma y extra opcional para Windows.
- **Interfaz Bilingüe y Documentación para GitHub (`src/i18n/`, `usePlayer.tsx`, `SettingsView.tsx`, `README.md`, `README.es.md`, `CHANGELOG.md`, `CHANGELOG.es.md`):** sistema i18n modular y tipado (`EN` / `ES`) con autodetección por idioma del navegador al iniciar por primera vez (Inglés por defecto), cambio en vivo desde `Configuración -> Idioma` persistido en `localStorage`, IndexedDB y respaldos JSON, formateo por locale y documentación bilingüe enlazada (`README` y `CHANGELOG`).

---

## [v0.8.0-beta] - Letras Sincronizadas, Perfil Local y Experiencia de Escritorio Web

### Letras Sincronizadas y Metadata Local
- **Letras sincronizadas con validación estricta de duración (`src/types/lyrics.ts`, `lrcParser.ts`, `lrclibClient.ts`, `useTrackLyrics.ts`, `LyricsView.tsx`, `coverColor.ts`):** soporte prioritario de sidecars `.lrc` locales, búsqueda en LRCLIB con tolerancia `±4.0s`, offset manual (`-0.5s / +0.5s`), selector de `Versiones`, caché en IndexedDB v4 (`lyrics`), tarjeta `Muestra de la letra` teñida con el color de carátula y modos Cine / Pantalla completa.
- **Ícono oficial, extracción rápida y múltiples carpetas fijas (`public/`, `vite.config.ts`, `localBridge.ts`, `libraryScanner.ts`, `SettingsView.tsx`, `LinkFolderModal.tsx`, `camelotColors.ts`, `TrackRow.tsx`):**
  - Favicon SVG/PNG y `manifest.webmanifest`; endpoints `/api/local-library/metadata` y `/api/local-library/cover` para leer tags y carátulas directamente en Node; reparación automática en segundo plano y manual (`Reparar portadas`).
  - Soporte para múltiples carpetas fijas persistentes en disco (`folderPaths`) y columnas `BPM · Tono` estilo Spotify visibles únicamente cuando `Mixear` está activo.

### Perfil, Recomendaciones y Playlists Automáticas
- **Perfil editable, Configuración centralizada e Inteligencia Local (`ProfileView.tsx`, `SettingsView.tsx`, `UserAvatar.tsx`, `recommendations.ts`, `HomeView.tsx`, `Sidebar.tsx`, `NowPlayingPanel.tsx`):**
  - Avatar contextual con vista `Perfil`, edición de foto/iniciales/colores, estadísticas Wrapped reales de escucha (`listening-stats`) y Daily Mixes automáticos (`Hecho para vos`).
  - Carpetas de playlists desplegables en `Tu biblioteca`, portadas y descripciones personalizadas en playlists (`PlaylistModal.tsx`), header con color dinámico en vistas, tarjetas `Acerca del artista` / `Detalles del archivo`, Autoplay inteligente, Radio de canción y sugerencias recomendadas al final de cada playlist.
- **Auto-creación y actualización de playlists por reproducciones (`AutoPlaylistModal.tsx`, `recommendations.ts`, `Sidebar.tsx`, `SettingsView.tsx`, `TrackList.tsx`):** reglas `top-played`, `monthly-top`, `recent-vibe` y `recently-played` con vista previa en vivo, modo automático continuo opcional y controles por playlist (`Actualizar lista`, `Auto: ON/OFF`).

### Audio Web de Escritorio y Lanzador Chrome `--app`
- **Reproducción estilo escritorio (`AudioEngine.ts`, `masterEq.ts`, `MiniPlayerButton.tsx`, `PlayerBar.tsx`):** Media Session API, Mini reproductor flotante (Document Picture-in-Picture), crossfade simple configurable (`0–12 s`, 4 s por defecto), EQ gráfico Master de 6 bandas, modo Mono, 3 niveles de Auto-Gain LUFS y Sleep Timer con fade-out suave.
- **Micro-fades rápidos de transporte (`Deck.ts`, `AudioEngine.ts`, `usePlayer.tsx`):** etapa `sourceGain` con cola `fadingSources` y ducking automático para suavizar pausa (`0.22 s`), reanudación (`0.18 s`), saltos de canción (`0.22 s` / `0.16 s`) y `seek` (`0.14 s` / `0.12 s`) sin congelar la UI.
- **Lanzador Chrome `--app` con auto-apagado limpio (`scripts/DeckMix.bat`, `scripts/launch-deckmix.vbs`, `scripts/run-deckmix.ps1`, `vite.config.ts`, `App.tsx`):** apertura silenciosa en modo aplicación Chrome y cierre automático del servidor vía SSE (`/api/app-session/stream`) tras cerrar la ventana.

---

## [v0.7.0-beta] - Motor Audio Pro y Transiciones Algoritmicas

- **Key Lock / Master Tempo real y Key Shift armónico (`public/worklets/pitchShifterProcessor.js`, `musicalKey.ts`, `mixMath.ts`, `Deck.ts`, `AudioEngine.ts`, `MixTransitionPanel.tsx`):** procesador `AudioWorkletProcessor` (`deckmix-pitch-shifter`) con buffer circular en ventana Hann para compensar cambios de tempo hasta `±16%` sin alterar el tono, además de selector `Key Shift` por deck (`-4` a `+4` semitonos).
- **Exportación offline a audio `.wav` (`offlineMixRenderer.ts`, `MixTransitionPanel.tsx`, `TrackList.tsx`):** renderizador en `OfflineAudioContext` para descargar en PCM estéreo de 16 bits tanto una transición individual (`Exportar WAV`) como una playlist mixeada completa (`Exportar Mix WAV`).
- **Aislamiento vocal / instrumental Mid/Side (`mix.ts`, `Deck.ts`, `offlineMixRenderer.ts`, `MixTransitionPanel.tsx`):** etapa Web Audio paralela sin clics por deck con modos `Normal`, `Solo base` y `Solo voz`.
- **Botón único `Mixear` independiente por lista y reingeniería Web Audio (`Deck.ts`, `AudioEngine.ts`, `TrackList.tsx`, `usePlayer.tsx`):** caché LRU de `AudioBuffer` decodificados (hasta 12 pistas / 320 MB), conexión bajo demanda de subgrafos pesados, tabla Hann precalculada en `pitchShifterProcessor.js`, recorte visual de duración al `outPointSecA` y estado `Mixear` independiente por lista (`mixedScopeIds`).
- **Integración algorítmica de transiciones (`dsp.ts`, `mixMath.ts`, `presets.ts`, `MixTransitionPanel.tsx`):** matriz de costo `Lambda(i, j)` (`findOptimalMixPoints`) con perfil vocal de 128 bins, presets `Onda`, `Melt` y `Slam`, `12` compases, sub-presets por pista y orden inteligente con continuidad energética LUFS y espectral.

---

## [v0.6.0-beta] - Efectos FX, Inteligencia Musical y Precisión en Tiempo Real

*(Antes Fases 2.8 y 2.9)*

- **Edición directa de nodos encima de ambas ondas (`WaveformCanvas.tsx`, `MixTransitionPanel.tsx`):** al seleccionar `Volumen`, `Ecualizador`, `Filtro` o `Efectos` (clic en línea para agregar nodo, arrastrar punto para editar, doble clic para eliminar).
- **Cadena FX paralela (`Deck.ts`, `AudioEngine.ts`, `presets.ts`, `TrackList.tsx`):** etapa post-EQ/Filter independiente de `deckGain` para preservar colas cuando el dry baja a `0%`: `DelayNode` sincronizado al BPM (`1/8`, `1/4`, `3/8`, `1/2` beat) con `highpass` en feedback para **Echo Out**, y `ConvolverNode` estéreo sintético para **Reverb Wash**, junto con la tarjeta `Efectos` y presets dedicados.
- **Auto-Gain (-14 LUFS) integrado (`dsp.ts`, `Deck.ts`, `AudioEngine.ts`, `usePlayer.tsx`):** cálculo LUFS BS.1770 con gating en Web Worker (`CURRENT_ANALYSIS_VERSION = 2`), trim automático `setInputTrimDb` sobre `inputGain` (rango `±8 dB`) y preferencia persistente `autoGainEnabled`.
- **Detección de Frases (`IN`, `DROP`, `OUT`) y Auto-Mix inteligente (`dsp.ts`, `mixMath.ts`, `usePlayer.tsx`):** detección estructural por energía de graves/medios en Worker, cálculo de puntos `outPointSecA`/`inPointSecB` anclados a Outro/Intro y selección automática de preset (`selectSmartAutoPreset`).
- **Ondas RGB por frecuencias + Zoom milimétrico (`WaveformCanvas.tsx`):** extracción de bandas `[low, mid, high]` en Worker (`waveformRgbBands`), render espectral con toggle `RGB` y zoom `1.0x`–`4.0x` (`Ctrl+Rueda` o botones `- / +`).
- **Hot Cues (`C1`–`C4`) y Vúmetro Master/Limiter (`db.ts`, `MasterMeter.tsx`, `MixTransitionPanel.tsx`):** persistencia en IndexedDB (`saveTrackHotCues`), marcadores visuales sobre las ondas y medidor en tiempo real de nivel RMS/Peak + reducción del limitador Master (`DynamicsCompressorNode.reduction`).

---

## [v0.5.0-beta] - Paridad Funcional estilo Reproductor Web

*(Antes Fase 2.7)*

- **Cola interactiva real (`manualQueueIds`, `upNextTrackIds`):** integrada en `usePlayer.tsx` con pestañas `Cola` / `En reproducción` en `NowPlayingPanel.tsx` y botón dedicado en `PlayerBar.tsx`.
- **Favoritos, pines e historial real (`db.ts`):** `"Tus me gusta"` (`likedTrackIds`), playlists fijadas (`pinnedPlaylistIds`) e historial `"Escuchado recientemente"` (`recentlyPlayedTrackIds`) almacenados en `UserPreferences` de IndexedDB.
- **Navegación unificada y búsqueda (`SearchView.tsx`, `HomeView.tsx`, `Sidebar.tsx`):** búsqueda unificada (`Ctrl+K` / `/`) con categorías locales por BPM/Camelot/género, `Resultado principal` y carruseles por entidad; `HomeView.tsx` con carruseles horizontales con flechas; `Sidebar.tsx` colapsable a modo mini (`76px`), pines y densidad `Lista | Compacta`.
- **Vistas detalladas de Biblioteca (`TrackList.tsx`, `TrackRow.tsx`):** Sticky Topbar al scrollear, vista de Álbum sin carátula repetida + lightbox de portada + `"Más de <Artista>"`, vista de Artista (`Populares`, `Discografía`, `Aparece en`), cabeceras ordenables, buscador colapsable tras lupa, densidad `Lista | Compacta`, menú contextual unificado por clic derecho / `...`, ecualizador animado `.eq-bars`, y `Smart Reorder` armónico por BPM + Camelot.
- **Mejoras en reproducción y edición de mezcla (`PlayerBar.tsx`, `MixTransitionPanel.tsx`):** toggle de duración total/restante en `PlayerBar.tsx`, chip armónico/delta BPM visible, navegación `< Anterior | Siguiente >` y confirmación `dirty`.

---

## [v0.4.0-beta] - Rediseño UI Desktop, Puente Local e Interfaz Modo Mix

*(Antes Fases 2.5 y 2.6)*

- **Shell visual oscuro estilo escritorio (`TopBar.tsx`, `Sidebar.tsx`, `HomeView.tsx`, `TrackList.tsx`, `CatalogGrid.tsx`, `NowPlayingPanel.tsx`, `PlayerBar.tsx`, `App.tsx`):** diseño desktop con paneles laterales izquierdo y derecho redimensionables por arrastre (persistidos en `localStorage`) y tarjetas responsivas con `line-clamp-2` para evitar recortes de títulos.
- **Puente local persistente en Vite (`vite.config.ts`, `src/storage/localBridge.ts`, `src/utils/audioSource.ts`, `src/components/LinkFolderModal.tsx`):** endpoints `/api/local-library/*` restringidos a loopback (`127.0.0.1` / `::1`) con configuración en `.deckmix-library.json` (ignorado por Git) para vincular carpetas sin copiar archivos y reconectar automáticamente tras recargar (`F5`), junto con selector nativo (`POST /api/local-library/pick`) y fallback `<input webkitdirectory>` / drag & drop.
- **Interfaz Modo Mix en panel derecho (`MixTransitionPanel.tsx`, `TrackList.tsx`, `WaveformCanvas.tsx`):** al activar `Mixear` y pulsar cualquier píldora `Auto` / `Personalizada` entre canciones, se abre el panel lateral `Editar transición [Beta]` (con opción de expandir al centro), doble onda variante `spotify` con curvas superpuestas de Volumen (cian), EQ (amarillo) y Filtro (magenta), botón circular central de pre-escucha, píldora inferior `N bars` (`2 / 4 / 8 / 12 / 16`), carrusel de presets (`Personalizada`, `Auto`, `Fade`, `Crescendo`, `Blend`, `Bass Swap`, `Filter Fade`) y tarjetas acordeón para `Volumen`, `Ecualizador` (Bajos/Medios/Agudos), `Filtro` (LPF/HPF + `Q`) y `Tempo y ajuste`.

---

## [v0.3.0-beta] - Modo Mix Manual

*(Antes Fase 2)*

- **Editor visual de doble onda en Canvas 2D (`WaveformCanvas.tsx`):** grilla de beats/compases (2, 4, 8 o 16 compases), arrastre horizontal con snap magnético (`Compás`, `Beat`, `Libre` o `Shift`), nudge fino y seek directo en la regla superior.
- **Editor interactivo de curvas:** Volumen (equal-power), EQ de 3 bandas (`250 Hz`, `1 kHz`, `3.8 kHz`) y filtros LPF/HPF logarítmicos con resonancia `Q` configurable; presets clásicos (`Fade`, `Blend`, `Bass Swap`, `Rise`, `Filter Fade`) y modo personalizado.
- **Motor lookahead y Auto-Mix (`AudioEngine.ts`, `Deck.ts`):** limitador suave `DynamicsCompressorNode` en Master, scheduler lookahead sample-accurate (`30 ms` / ventana `160 ms`), beatmatching con rampa acoplada (`±6%`, soporte `0.5x/2x`), integrador exacto de posición acumulada bajo `playbackRate` variable, evaluación analítica de curvas para pausa/seek en medio de una transición y modo Auto-Mix en playlists.
- **Persistencia de transiciones (`db.ts`, `mixStorage.ts`):** almacenamiento en IndexedDB v3 (`mixTransitions`) y exportación/importación portable en JSON.

---

## [v0.2.0-alpha] - Análisis Musical DSP y Navegación de Biblioteca

*(Antes Fase 1.5)*

- **Navegación por Artistas y Álbumes (`CatalogGrid.tsx`):** vistas dedicadas, ordenamiento por título, artista, álbum, pista, duración, BPM y tonalidad Camelot, y soporte ampliado para FLAC, M4A, MP4, AAC, OPUS y WEBM.
- **Store `analysis` en IndexedDB v2 (`DB_VERSION = 2`):** guardado de `waveformPeaks` como `Uint8Array`, `analysisVersion`, `bpmSource` / `keySource` y protección estricta contra sobrescritura de ediciones manuales y tags válidos.
- **Motor DSP en TypeScript puro sobre Web Worker (`src/analysis/dsp.ts`, `analysisWorker.ts`, `analysisQueue.ts`):** copia mono segura para Transferables sin invalidar el `AudioBuffer` activo, `waveformPeaks` al 100 % de duración, ventana central para tracks largos, pasa-bajos anti-aliasing + downsampling, detección de BPM + `firstBeatOffsetSec`, y HPCP 36->12 con compresión logarítmica + Krumhansl-Schmuckler y rueda Camelot.
- **Cola prioritaria y edición manual (`analysisQueue.ts`, `EditAnalysisModal.tsx`):** cola FIFO con prioridad para el track en reproducción/carga y edición manual protegida.

---

## [v0.1.0-alpha] - Reproductor Local Inicial

*(Antes Fase 1)*

- **Carga de archivos locales:** soporte para MP3, WAV, FLAC y OGG mediante File System Access API (`showDirectoryPicker`), `<input webkitdirectory>` o drag & drop.
- **Indexación y playlists en IndexedDB:** lectura de tags y carátulas con `music-metadata` (`parseBlob`, `selectCover`), escaneo incremental en IndexedDB (`idb`) con huella por contenido (`fingerprint.ts`) y gestión de playlists locales con reordenamiento drag & drop (`@dnd-kit/sortable`) + botones subir/bajar.
- **Motor de doble Deck (`AudioEngine.ts`, `Deck.ts`):** motor desacoplado de React con dos instancias `Deck` (`A` y `B`), decodificación bajo demanda y precarga en background del siguiente track.
- **Interfaz oscura inicial:** Sidebar, lista central con búsqueda rápida y barra de reproducción fija inferior con atajos de teclado.
