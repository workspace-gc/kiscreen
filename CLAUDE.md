# CLAUDE.md — KiScreen

Instrucciones para Claude Code en este proyecto. La especificación completa está en
**SPECS.md**: léela antes de empezar cualquier tarea.

## Sobre el usuario
- Se llama Mario, es desarrollador principiante en Android nativo (viene de Flutter/Dart).
- **Responde siempre en español.**
- Explica cada cambio importante de forma sencilla y con **un ejemplo concreto**
  ("este callback se ejecuta cuando el Mac envía un trozo de vídeo, por ejemplo…").
- Si algo es ambiguo, **pregunta antes de decidir**, y haz **una sola pregunta cada vez**.

## Forma de trabajar
- Sigue las **fases de SPECS.md §5 en orden**. No empieces una fase hasta que la
  anterior cumpla su criterio de "hecho" **probado en la tele**.
- Al terminar cada fase: compila, instala, abre la app, revisa logcat y dile a Mario
  exactamente qué tiene que comprobar en el Mac/tele. Espera su confirmación.
- Haz un commit por cada paso que funcione (mensajes en español, formato
  `fase N: descripción`).
- Cambios pequeños e incrementales; no reescribas archivos enteros sin motivo.

## Proyecto
- Creado por Mario en Android Studio (plantilla *Empty Activity*, Compose). **Lee el
  package name y el `applicationId` del propio proyecto**; no los cambies.
- Kotlin + Jetpack Compose (sin Leanback/XML). C con NDK y CMake
  (`externalNativeBuild`).
- Estructura objetivo:
  ```
  app/src/main/
    java/<package>/
      MainActivity.kt
      ui/            WaitingScreen.kt, MirrorScreen.kt
      receiver/      ReceiverController.kt, NativeReceiver.kt
      discovery/     DiscoveryAdvertiser.kt
      media/         VideoDecoder.kt, AudioPlayer.kt
    cpp/
      CMakeLists.txt
      receiver_jni.c
      third_party/uxplay/   (lib/ de UxPlay, sin modificar)
      third_party/libplist/
  scripts/            (scripts de build de dependencias, si hacen falta)
  THIRD_PARTY.md      (origen, commit y licencia de cada dependencia)
  ```

## Código de terceros
- `third_party/` **no se edita a mano**. Si hace falta un cambio, crea un parche en
  `patches/` y documéntalo en `THIRD_PARTY.md`, o resuélvelo con un *shim* propio.
- Conserva los archivos de licencia (UxPlay es GPL-3.0). Proyecto de uso personal.
- Usa `uxplay.cpp` del repositorio original solo como referencia de uso de la librería.

## Comandos

El SDK de Android está en `~/Library/Android/sdk`. Si `adb` no está en el PATH, usa
`~/Library/Android/sdk/platform-tools/adb`.

```bash
# Conectar con la tele (Mario te dará la IP si no la sabes; pregúntala)
adb connect <IP_TELE>:5555
adb devices                         # debe aparecer la tele como "device"

# Info del dispositivo (fase 0)
adb shell getprop ro.product.cpu.abilist
adb shell getprop ro.build.version.sdk

# Compilar e instalar
./gradlew assembleDebug
./gradlew installDebug

# Abrir la app
adb shell am start -n <applicationId>/.MainActivity

# Logs del proyecto
adb logcat -c && adb logcat -s KiScreen:V KiScreenNative:V AndroidRuntime:E
```

Antes de instalar, comprueba siempre con `adb devices` que la tele está conectada. Si
hay varios dispositivos, usa `-s <IP_TELE>:5555`.

## Preparar la tele (si ADB no conecta)
Explícale a Mario estos pasos (el texto de los menús puede variar):
1. Ajustes → Preferencias del dispositivo → Información → pulsar 7 veces en *Compilación*
   para activar las Opciones de desarrollador.
2. Opciones de desarrollador → activar *Depuración USB* / *Depuración por red*.
3. Ver la IP en Ajustes → Red. Al primer `adb connect`, aceptar el aviso en la tele.

## Reglas
- No desinstales ni toques otras apps de la tele. Solo `installDebug` / `uninstall` de
  este `applicationId`.
- No cambies ajustes del sistema de la tele por ADB sin preguntar.
- No subas el proyecto a ningún repositorio remoto sin que Mario lo pida.
- Si algo del protocolo falla, revisa primero logcat y compara con lo que hace UxPlay
  antes de cambiar código a ciegas.
- Etiquetas de log: `KiScreen` (Kotlin) y `KiScreenNative` (C).
