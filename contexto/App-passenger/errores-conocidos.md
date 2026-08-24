# errores-conocidos

Repositorio:
tukituki-passenger-app

Branch analizada:
main

Commit analizado:
5c4f0f9136f2e49a4b746755963e01fa29aa3d49

Última actualización:
2026-08-18 (refrescado tras `CROSS-APP-R4.3I`, fast-forward de `test/r4-ride-identities` a `main`)

Fuente de verdad:
Este documento es contexto auxiliar. Si contradice al código actual,
el código y los tests tienen prioridad.

---

## Baseline de tests

`flutter test` en este commit (2026-08-15, Flutter 3.44.6 / Dart
3.12.2): **155 tests ejecutados, 155 pasando, 0 fallando, 0 skips
visibles en la salida**. No hay tests marcados `skip:` encontrados por
lectura de código.

## Baseline de análisis estático

`flutter analyze` en este commit: **"No issues found!"** — sin
warnings ni infos pendientes bajo el set de reglas de
`flutter_lints ^6.0.0` configurado en `analysis_options.yaml`.

`flutter pub get` (disparado por `flutter analyze`) reporta 14
paquetes con versiones más nuevas disponibles pero incompatibles con
los constraints actuales de `pubspec.yaml` (`flutter pub outdated` da
el detalle) — no es un error, es información de paquetes desactualizados
respecto a upstream; no se han verificado breaking changes de subirlos.

## Comentarios FIXME/TODO relevantes

No se encontraron marcadores `TODO`, `FIXME`, `XXX` ni `HACK` en
`lib/` (búsqueda exhaustiva por patrón). Los comentarios "de advertencia"
presentes en el código son explicativos de invariantes, no marcadores
de trabajo pendiente (ver `passenger_ride.dart:72-83`,
`home_screen.dart:50-57` y `:102-120`).

- **Inconsistencia conocida (2026-08-20) — referencia a un `AGENTS.md` que no existe en este repositorio**: el comentario de `_handleDriverLocationUpdate()` en `lib/features/ride/presentation/ride_searching_screen.dart` (línea ~3278, checkpoint `R4.4B`, ver `decisiones.md`) dice textualmente "Reglas (ver AGENTS.md R4.4B)", pero no existe ningún archivo `AGENTS.md` en la raíz de `tukituki-passenger-app` (confirmado por búsqueda directa — tampoco existe en `tukituki-driver-app`, que tiene el mismo patrón de comentario en `driver_active_ride_screen.dart`). No se creó el archivo ni se editó el comentario como parte de esta auditoría documental (son cambios de código, fuera de su alcance) — queda registrado aquí como rastro para quien retome `R4.4B`.

## Configuraciones delicadas

- **`API_BASE_URL` es obligatorio en runtime**: si se olvida
  `--dart-define=API_BASE_URL=...` al compilar/ejecutar, la app falla
  inmediatamente en `main()` con un `StateError` antes de mostrar
  cualquier UI (`lib/core/config/app_config.dart`). No hay mensaje de
  error amigable en pantalla para este caso — es un crash de arranque.
- **`MAPS_API_KEY` vía `android/local.properties`**: si el archivo o la
  propiedad no existen, `build.gradle.kts` usa `""` como default
  silencioso (`localProperties.getProperty("MAPS_API_KEY", "")`) — el
  build no falla, pero el mapa de Google no funcionará en runtime sin
  aviso explícito en build time.
- **Firma de release = firma de debug** (`android/app/build.gradle.kts:60-67`):
  cualquier APK "release" generado tal cual desde este repo está
  firmado con la clave de debug de Flutter, no apto para publicación
  en Play Store sin configurar un `signingConfig` propio primero.

## Incompatibilidades / limitaciones técnicas demostrables

- **Prefijo telefónico fijo a `+51`** (Perú) hardcodeado en
  `login_screen.dart` y `register_screen.dart` — la app no puede
  autenticar números de otro país sin cambiar código.
- **`cancelledBy == 'DRIVER'` no distingue cancelación normal de
  no-show**: documentado explícitamente en el propio código
  (`passenger_ride.dart:72-77`) como limitación actual del backend, no
  del cliente — "Backend no expone `cancellationType` a Passenger
  todavía".
- **Ruta `/otp` y `OtpScreen` son código huérfano**: siguen compilando
  y están registrados en `app_router.dart`, pero ningún flujo activo
  navega a ellos desde que el registro dejó de requerir OTP (commit
  `34e62eb`). Un cambio futuro que reactive el flujo debe revisar que
  `OtpArguments`/`OtpScreen` sigan siendo compatibles con el backend
  actual (no verificado por tests, porque no hay test que ejercite esa
  ruta).
- **Sin reintentos automáticos de red más allá del refresh de
  sesión**: cualquier error de conexión (`connectionError`,
  timeouts) en una request que no sea 401 se propaga tal cual al
  widget, que solo puede mostrar un mensaje y dejar que el usuario
  reintente manualmente (patrón repetido en todas las pantallas, no
  hay política de retry/backoff).
- **Sin caché offline**: si el dispositivo pierde conectividad durante
  el polling de un ride activo, la última información visible puede
  quedar desactualizada hasta que vuelva la conexión; no hay
  indicador explícito de "sin conexión" distinto del mensaje de error
  genérico por pantalla.
- **"¿Olvidaste tu contraseña?" existe en la UI pero no hay recuperación
  de contraseña implementada** (`login_screen.dart`, agregado
  2026-08-20 al migrar la pantalla al mockup): el enlace es visible,
  respeta el estilo del mockup y responde al toque, pero solo muestra
  un `SnackBar` ("Pronto podrás recuperar tu contraseña") — no navega a
  ninguna pantalla ni llama a ningún endpoint. No existe flujo de
  recuperación de contraseña en ningún punto del cliente ni evidencia
  de un endpoint correspondiente en `AuthRepository`. Antes de
  conectarlo a algo real hace falta la pantalla/ruta y el endpoint de
  backend, ninguno de los cuales existe hoy.
- **"Términos" y "Política de privacidad" no existen como documento,
  ruta ni URL en ninguna parte del proyecto — pendiente legal antes de
  salir al público** (`register_screen.dart`, `_TermsFootnote`,
  agregado 2026-08-24 al quitar el checkbox de aceptación de términos
  — ver `decisiones.md`): la nota al pie de "Crear cuenta" dice "Al
  crear tu cuenta aceptas nuestros Términos y nuestra Política de
  privacidad", y ambos enlaces responden al toque con un `SnackBar`
  ("Pronto podrás leer nuestros términos" / "...nuestra política de
  privacidad"), mismo patrón que "¿Olvidaste tu contraseña?" de
  arriba. Búsqueda exhaustiva en el repo: no hay ningún archivo de
  términos/política, ninguna ruta de la app que los muestre, ni
  ninguna URL externa configurada — el texto le hace aceptar al
  usuario documentos que hoy no existen en ningún lado.

## Discrepancia doc/código encontrada y corregida (checkpoint R4.2)

Las secciones "passenger" de `arquitectura.md` y la entrada "Perfil de
pasajero como paso separado" de `decisiones.md` describían la
finalización de perfil como un "paso obligatorio" — no lo era: el
resultado de `getMyProfile()` en `SplashScreen` se descartaba sin usar
y la app siempre continuaba a `/home`. El propio `flutter test` de
`main`@`a5d2411e` no detectaba esto porque el test entonces vigente
(`'Perfil 404 y sin ride continúa a home'`) afirmaba ese mismo
comportamiento como esperado, en vez de señalarlo como un defecto.
Corregido en `test/r4-passenger-identity` (`R4.2`,
`PHYSICAL-REVIEW-PASS`, 2026-08-18) — ver `decisiones.md`. **Integrado a
`main`@`91357d17cdb8e154124021b1ab9dc33a3bdd9ae6`** (`CROSS-APP-R4.2D`,
fast-forward, 2026-08-18) y confirmado por `MAIN-PHYSICAL-SMOKE-PASS`
de JuanJo (`CROSS-APP-R4.2E`, 2026-08-18) — **`FINAL-CLOSED`**: **181
tests, todos en verde** (`flutter analyze` limpio) — 26 más que el
baseline previo de 155 sobre `main`@`a5d2411e`. Rama de test eliminada
tras confirmar contención total.

## Bugs resueltos durante `CROSS-APP-R4.3` (histórico, ya corregido y aprobado físicamente)

- **BUG RESOLVED — foto del Driver parpadeaba durante el polling** (`DRIVER_ARRIVING`/`DRIVER_ARRIVED`, encontrado y corregido en el mismo checkpoint, 2026-08-18): con un Driver real con foto, la imagen aparecía/desaparecía en cada ciclo de `Timer.periodic` (3s). Causa raíz: el capability token firmado de Storage rota en cada respuesta del Backend aunque sea la misma foto; `Image.network` trataba cada URL nueva como un recurso distinto y soltaba el frame anterior mientras cargaba el nuevo. Fix: última foto válida conservada únicamente en memoria RAM de la instancia (`_stableDriverPhotoUri`), con alcance por `rideId` + `driverProfileId` (cambia cualquiera de los dos → reset total), una capability nueva y válida del mismo Driver reemplaza a la anterior, una respuesta transitoria sin foto válida no borra la última foto válida, `gaplessPlayback: true` para evitar el parpadeo a blanco entre frames. Sin persistencia (`SharedPreferences`/secure storage/disco), sin exposición de `objectKey`. **Status: RESOLVED, PHYSICAL RETEST PASS** (`CROSS-APP-R4.3F`, 7/7 tests físicos de estabilidad/visor, y reconfirmado en el smoke físico final `MAIN-PHYSICAL-SMOKE-PASS`). Ver `decisiones.md`.

## Baseline `main`@`5c4f0f9` (CROSS-APP-R4.3, ya fusionado)

`flutter analyze` limpio, **196/196 tests, todos en verde** (181→196 sobre el baseline previo de `main`@`91357d17`) — validado sobre `test/r4-ride-identities` (2026-08-18) y re-confirmado sin cambios de código tras el fast-forward a `main`. Confirmado además por `MAIN-PHYSICAL-SMOKE-PASS` de JuanJo (smoke físico final, 7/7). **Este es ahora el baseline oficial de `main`.** `test/r4-ride-identities` eliminada local y remotamente tras el cierre.

## Problemas visibles en commits recientes

- El commit `e0e3cff` ("fix: harden passenger fare negotiation") y
  `95ed192` ("fix: harden passenger session recovery") indican que
  hubo bugs previos de robustez en negociación de tarifa y
  recuperación de sesión que ya fueron corregidos en este historial —
  no quedan issues abiertos conocidos asociados a ellos en el repo
  (no hay tracker de issues local).

## Pantallas que dependían del `ThemeData` por defecto — cambian de aspecto con `DESIGN-SYSTEM-R1` (2026-08-20)

Contexto: `DESIGN-SYSTEM-R1` (rama `test/design-system-r1`, sin commit
todavía) reemplazó el `ThemeData` de `lib/app.dart` — antes
`colorSchemeSeed: Colors.amber` genérico, ahora
`ColorScheme.fromSeed(seedColor: PassengerColors.acento, error:
PassengerColors.error)` + `scaffoldBackgroundColor:
PassengerColors.crema` (ver `decisiones.md`, entrada
`DESIGN-SYSTEM-R1`). Ninguna pantalla fue migrada a los tokens nuevos
en este checkpoint — los ~103 `Color()` escritos a mano por pantalla
(splash, login, register, home, ride_searching, ride_receipt) siguen
intactos y no cambian de aspecto.

JuanJo aceptó como esperado que el cursor, la selección de texto, el
ripple y la barra de estado cambien de tono en toda la app por heredar
del tema. Además de eso, se identificaron **cinco puntos concretos**
que dependían 100% del tema por defecto (sin ningún color propio) y
por lo tanto también cambian de aspecto — no solo cursor/ripple, sino
el fondo del `Scaffold`/`AppBar` completo:

1. `lib/features/passenger/presentation/complete_profile_screen.dart`
   — `Scaffold`/`AppBar` sin ningún color propio. **Pantalla activa y
   alcanzable** ("Completa tu perfil").
2. `lib/features/ride/presentation/ride_searching_screen.dart:2844` —
   `Scaffold` del estado "cargando" (spinner), sin `backgroundColor`.
   Activo y alcanzable.
3. `lib/features/ride/presentation/ride_searching_screen.dart:2852-2853`
   — `Scaffold`/`AppBar` del estado "no encontramos un viaje activo"
   (error), sin colores propios. Activo y alcanzable.
4. `lib/features/ride/presentation/ride_searching_screen.dart:2885-2888`
   — `AppBar` de fallback para un estado de ride no cubierto por los
   demás builders, sin `backgroundColor`. Activo y alcanzable.
5. `lib/features/auth/presentation/otp_screen.dart` — mismo patrón
   (`Scaffold`/`AppBar` sin colores propios), pero **es código
   huérfano**: ningún flujo activo navega a `/otp` desde que el
   registro dejó de requerir OTP (commit `34e62eb`, ver `decisiones.md`
   "Verificación de teléfono por OTP se volvió opcional"). El cambio de
   aspecto aquí no es observable en la práctica — **no vale la pena
   investigarlo ni "corregirlo"** salvo que se reactive esa ruta.

Pendiente para la tarea de migración de pantallas (fuera de alcance de
`DESIGN-SYSTEM-R1`): decidir si los puntos 1-4 reciben colores propios
de marca explícitos (como ya tienen splash/login/register/ride_receipt)
o si se dejan heredando del tema — pero ya heredando de los tokens de
marca reales, no de un seed genérico.

## Notas de alcance de esta verificación

- No se ejecutó `flutter build apk`/`appbundle` completo (fuera de
  alcance de esta documentación; no se instalaron toolchains
  adicionales).
- No se ejecutó la app en un emulador/dispositivo real; los hallazgos
  de esta sección provienen de lectura de código, `flutter analyze` y
  `flutter test` únicamente.
