# SPECS.md — KiScreen (receptor de duplicación de pantalla para Android TV)

## 1. Objetivo

App Android TV nativa (Kotlin + Jetpack Compose + C vía NDK) que convierte el televisor
Xiaomi de 50" (Android TV) en un **receptor de duplicación de pantalla para macOS**.
Desde el MacBook M1, en *Centro de control → Duplicar pantalla*, aparece la tele y al
elegirla se ve la pantalla del Mac **con sonido** en el televisor.

Caso de uso principal: ver cursos online desde el sofá con el portátil, mostrando la
imagen y el sonido en la tele.

**Uso exclusivamente personal.** No se publica en la Play Store. El proyecto incorpora
código GPL-3.0 (UxPlay), lo cual es aceptable para uso privado.

## 2. Alcance

### Incluido (MVP)
- Anunciarse en la red local para que el Mac vea la tele como destino de duplicación.
- Emparejamiento y cifrado del protocolo de duplicación (reutilizando la librería de UxPlay).
- Recepción y decodificación del vídeo H.264 de la duplicación de pantalla.
- Recepción y reproducción del audio (AAC-ELD) que acompaña a la duplicación.
- Control de volumen enviado desde el Mac.
- Pantalla de espera ("Esperando conexión… Nombre: KiScreen") y vuelta a ella al desconectar.

### Excluido (por ahora)
- Envío de solo vídeo/URL (modo "AirPlay vídeo"), audio-solo (ALAC), fotos.
- Google Cast, Miracast, DLNA.
- Vídeo H.265/HEVC.
- Código PIN / contraseña de conexión.
- Arranque automático al encender la tele (ver fase 6, opcional).

## 3. Entorno

| Elemento | Valor |
|---|---|
| Emisor | MacBook M1 con macOS reciente |
| Receptor | Xiaomi TV 50" con Android TV (misma red Wi-Fi que el Mac) |
| Desarrollo | Android Studio en Mac; Claude Code compila e instala por ADB en red |
| Lenguajes | Kotlin (app), C (librería de protocolo), CMake (build nativo) |
| UI | Jetpack Compose (sin Leanback/XML) |

**A comprobar en la fase 0** (no suponer):
- `adb shell getprop ro.product.cpu.abilist` → ABI(s) del televisor. Muchas teles
  Xiaomi/MediaTek tienen *userspace* de 32 bits (`armeabi-v7a`) aunque la CPU sea de 64.
- `adb shell getprop ro.build.version.sdk` → versión de Android, para fijar `minSdk`
  (propuesta: `minSdk 26` si la tele lo permite).

## 4. Arquitectura

```
┌─────────────────────────── App Android TV ───────────────────────────┐
│                                                                      │
│  UI (Compose)          Kotlin                        Nativo (C/NDK)  │
│  ┌──────────────┐   ┌───────────────────────┐   ┌──────────────────┐ │
│  │ WaitingScreen│◄──│ ReceiverController    │◄─►│ JNI bridge       │ │
│  │ MirrorScreen │   │  (estado: Idle /      │   │ (receiver_jni.c) │ │
│  │ (SurfaceView)│   │   Connected / Error)  │   └────────┬─────────┘ │
│  └──────▲───────┘   ├───────────────────────┤            │           │
│         │           │ DiscoveryAdvertiser   │   ┌────────▼─────────┐ │
│         │           │  (mDNS: _airplay,     │   │ UxPlay lib/      │ │
│         │           │   _raop)              │   │ (raop, pairing,  │ │
│         │           ├───────────────────────┤   │  fairplay, rtp,  │ │
│         └───────────│ VideoDecoder          │◄──│  mirror, ntp)    │ │
│                     │  (MediaCodec H.264)   │   ├──────────────────┤ │
│                     ├───────────────────────┤   │ OpenSSL (crypto) │ │
│                     │ AudioPlayer           │◄──│ libplist         │ │
│                     │  (MediaCodec AAC-ELD  │   └──────────────────┘ │
│                     │   + AudioTrack)       │                        │
│                     └───────────────────────┘                        │
└──────────────────────────────────────────────────────────────────────┘
```

### 4.1 Componentes

**DiscoveryAdvertiser (Kotlin)**
- Publica dos servicios mDNS: `_airplay._tcp` y `_raop._tcp` (este último con nombre
  `<DEVICEID_HEX>@<Nombre>`), con los registros TXT que espera macOS.
- Los **valores TXT** (features, model, srcvers, pk, deviceid, flags, etc.) se copian del
  comportamiento de `lib/dnssd.c` de UxPlay. La clave pública `pk` la proporciona la
  parte nativa (se expone por JNI).
- Implementación: `NsdManager` + `NsdServiceInfo.setAttribute`. Si los TXT no llegan bien
  al Mac (problema conocido en algunos Android), sustituir por la librería **JmDNS**.
- Mantener un `WifiManager.MulticastLock` mientras la app está activa.
- `deviceid`: Android no da la MAC real → generar 6 bytes aleatorios **una vez** y
  guardarlos (SharedPreferences/DataStore) para que el Mac reconozca siempre la misma tele.

**Librería nativa (C)**
- Se reutiliza la carpeta `lib/` del proyecto **UxPlay** (https://github.com/FDH2/UxPlay,
  GPL-3.0), copiada en `app/src/main/cpp/third_party/uxplay/` con su licencia y el
  commit de origen anotado en `THIRD_PARTY.md`.
- **No** se usa la parte de UxPlay que depende de GStreamer (renderers) ni `dnssd.c`
  para el anuncio (lo hace Kotlin). Si `raop` necesita datos de `dnssd`, crear un *shim*
  mínimo en C en vez de modificar el código original.
- `uxplay.cpp` del repositorio original sirve como **referencia** de cómo se inicializa
  la librería y qué callbacks se registran.
- Dependencias:
  - **OpenSSL**: preferente el paquete Prefab `com.android.ndk.thirdparty:openssl`
    (verificar disponibilidad/versión); alternativa: compilar OpenSSL desde fuente para
    las ABI necesarias con un script en `scripts/`.
  - **libplist** (https://github.com/libimobiledevice/libplist): compilar desde fuente
    con un `CMakeLists.txt` propio que liste sus fuentes.

**JNI bridge (`receiver_jni.c` + `NativeReceiver.kt`)**
- `start(name, deviceId): Boolean`, `stop()`, `getPublicKey(): ByteArray`, `getPorts()`.
- Callbacks nativos → Kotlin: `onConnected()`, `onDisconnected()`,
  `onVideoData(ByteArray, ptsUs)`, `onAudioData(ByteArray, ptsUs)`, `onVolume(Float)`,
  `onLog(level, msg)`.
- Los callbacks llegan desde hilos nativos: usar `AttachCurrentThread`, cachear
  `jmethodID` en `JNI_OnLoad` y no bloquear los hilos de red (copiar y encolar).

**VideoDecoder (Kotlin)**
- `MediaCodec` tipo `video/avc` configurado sobre el `Surface` de un `SurfaceView`.
- Los primeros datos traen SPS/PPS: usarlos para configurar (`csd-0`/`csd-1`) o
  enviarlos con `BUFFER_FLAG_CODEC_CONFIG`.
- Si llega un SPS nuevo (el Mac cambia de resolución) → reiniciar el codec.
- Mantener la relación de aspecto (bandas negras si hace falta).

**AudioPlayer (Kotlin)**
- `MediaCodec` tipo `audio/mp4a-latm`, perfil AAC-ELD (`AACObjectELD`), 44 100 Hz,
  estéreo. `csd-0` de referencia: `F8 E8 50 00` (el mismo *codec_data* que usa UxPlay
  para AAC-ELD; verificar).
- Salida por `AudioTrack` (PCM 16 bits). MVP: reproducir según llega con un búfer
  pequeño; sincronización labial fina queda para más adelante.
- Aplicar el volumen recibido del Mac (escala de AirPlay: -30 a 0 dB, -144 = silencio).

**ReceiverController + UI (Compose)**
- Estados: `Idle` (pantalla de espera), `Connected` (vídeo a pantalla completa),
  `Error(mensaje)`.
- Pantalla de espera: nombre del receptor, IP local y el texto "En tu Mac: Centro de
  control → Duplicar pantalla → KiScreen".
- `FLAG_KEEP_SCREEN_ON` mientras hay conexión.
- Botón Atrás del mando en `Connected` → cortar la sesión y volver a `Idle`.
- El servidor vive mientras la Activity está en primer plano (MVP).

### 4.2 Manifest
- Permisos: `INTERNET`, `ACCESS_NETWORK_STATE`, `ACCESS_WIFI_STATE`,
  `CHANGE_WIFI_MULTICAST_STATE`, `WAKE_LOCK`.
- TV: categoría `LEANBACK_LAUNCHER`, `android:banner`,
  `uses-feature android.hardware.touchscreen required=false`,
  `uses-feature android.software.leanback required=false`.
- Orientación horizontal.

## 5. Plan por fases

Cada fase termina con una **prueba real en la tele** antes de seguir.

| Fase | Qué se hace | Criterio de "hecho" |
|---|---|---|
| 0. Entorno | Comprobar ADB conectado a la tele, NDK y CMake instalados, ABI y SDK de la tele. App vacía instalada. | La app vacía se abre en la tele. |
| 1. Descubrimiento | `DiscoveryAdvertiser` con TXT completos (pk provisional si hace falta). | "KiScreen" aparece en *Duplicar pantalla* del Mac (la conexión puede fallar). |
| 2. Nativo + protocolo | Integrar UxPlay `lib/`, OpenSSL, libplist, JNI. Arrancar servidor. | Al elegir la tele en el Mac, logcat muestra el handshake (pair-setup, pair-verify, fp-setup, SETUP) sin errores. |
| 3. Vídeo | `VideoDecoder` + `MirrorScreen`. | Se ve la pantalla del Mac en la tele, fluida a 1080p. |
| 4. Audio | `AudioPlayer` + volumen. | Se oye el sonido del Mac en la tele; el volumen del Mac lo cambia. |
| 5. Pulido | Estados, reconexión, cambio de resolución, botón Atrás, pantalla siempre encendida, mensajes de error. | 3 conexiones/desconexiones seguidas sin reiniciar la app; un curso de 30 min sin cortes. |
| 6. Opcional | Servicio en primer plano, arranque al encender, nombre configurable. | — |

## 6. Requisitos no funcionales
- Latencia objetivo ≤ 300 ms (aceptable para ver cursos).
- Sin cierres inesperados durante 1 h de uso.
- Logs con etiqueta `KiScreen` (Kotlin) y `KiScreenNative` (C).

## 7. Riesgos conocidos
- **macOS puede cambiar el protocolo** en actualizaciones; si deja de funcionar, revisar
  si UxPlay tiene una versión más nueva y actualizar `third_party/uxplay`.
- **TXT de mDNS** mal publicados con `NsdManager` → usar JmDNS.
- **Decodificador AAC-ELD** no disponible en el hardware de la tele → plan B: compilar
  un decodificador software (p. ej. fdk-aac) por NDK.
- **ABI de 32 bits** en la tele → compilar `armeabi-v7a` además de `arm64-v8a`.
