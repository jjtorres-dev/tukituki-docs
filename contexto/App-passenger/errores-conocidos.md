# errores-conocidos

Repositorio:
tukituki-passenger-app

Branch analizada:
main

Commit analizado:
5c4f0f9136f2e49a4b746755963e01fa29aa3d49

Última actualización:
2026-08-27 (agregados pendientes conocidos de `HOME-FLOW-R1`)

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

- **Inconsistencia encontrada y corregida en la fuente (`FARE-PANEL-R1`, etapa 5, 2026-08-31) — referencia rota a `App-passenger/decisiones.md`**: el doc-comment de la heurística de acortado de direcciones (`_shortAddressLabel`, agregada en `HOME-LAYOUT-R1`) decía "ver `App-passenger/decisiones.md` para el detalle del costo estimado" de agregar un campo de nombre corto en Backend — esa entrada nunca existió en `decisiones.md` (búsqueda exhaustiva, sin resultados), mismo patrón que la referencia rota a `AGENTS.md` de arriba. Al extraer la función a `lib/features/home/domain/short_address_label.dart` en esta etapa, la referencia rota se reemplazó por una nota honesta ("agregar ese campo requeriría tocar `google-geocoding.service.ts`, fuera de alcance de esta app") en vez de arrastrarla. **Pendiente real, no bloqueante**: si en algún momento se decide evaluar agregar ese campo corto en Backend, esa evaluación de costo todavía no está escrita en ningún lado — habría que escribirla desde cero, no recuperarla de una entrada perdida.

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

## ANR al tocar repetidamente "centrar en mi ubicación" en el emulador (2026-08-24, sin reproducir en físico todavía)

Estado:
PENDIENTE DE REPRODUCIR EN DISPOSITIVO FÍSICO — no confirmado como bug real de la app ni descartado como límite del emulador.

Qué se observó:
JuanJo reportó que tocar repetidamente el botón "Centrar en mi ubicación" de `home_screen.dart` en el emulador Android congeló la app ("TukiTuki Pasajero isn't responding"), durante la validación de `ORIGIN-ADDRESS-R1`.

Causa más probable (por lectura de código):
`_loadCurrentLocation()` llama a `_moveCameraToCurrentLocation()` → `GoogleMapController.animateCamera(...)`, un round-trip por canal de plataforma hacia la vista nativa de Google Maps. El guard `_locating` (`home_screen.dart`) sí serializa correctamente las llamadas a `_loadCurrentLocation` — no hay overlap ahí (confirmado por lectura: `_locating = true` se asigna de forma síncrona antes del primer `await`, y Dart no puede entrelazar dos invocaciones entre ese chequeo y esa asignación) — pero SÍ permite ciclos consecutivos rápidos uno tras otro si cada uno termina rápido, cada uno disparando su propio `animateCamera` nativo. Llamadas repetidas de `animateCamera` en sucesión rápida es una causa conocida de jank/ANR de `google_maps_flutter` en emuladores sin aceleración GPU real (Google Maps renderizado por software). Esta llamada y este flujo **ya existían antes de `ORIGIN-ADDRESS-R1`** (el checkpoint que agregó `GET fares/origin-address`) — no fueron introducidos por ese checkpoint.

Se descartó como causa directa del ANR (aunque sí era un bug real aparte, ya corregido en la misma investigación): la acumulación de llamadas a `GET fares/origin-address` sin guard de "ya hay una en curso" — taps repetidos podían disparar varias llamadas concurrentes a ese endpoint nuevo antes de que la primera respondiera. Las llamadas HTTP de Dio son async/no bloquean el hilo principal de Android por sí solas, así que no explican un ANR (bloqueo del hilo principal) por sí mismas, aunque sí eran un desperdicio de cupo del rate limit del backend (`ORIGIN_ADDRESS_RATE_LIMIT_MAX`). Ese guard ya se agregó (`_originAddressInFlightPosition`/`_originAddressInFlightRequestId` en `home_screen.dart`), independientemente de si resulta ser o no la causa del ANR.

Qué falta para confirmar:
Reproducir el mismo tap repetido en un dispositivo físico o en un emulador con aceleración GPU confirmada. Si el freeze desaparece ahí, confirma que el mecanismo raíz es el emulador (probablemente sin aceleración GPU) y no un bug de la app. Si persiste en físico, es un bug real preexistente en el flujo de recentrado (`_moveCameraToCurrentLocation`), no introducido por `ORIGIN-ADDRESS-R1`, y ameritaría su propio checkpoint (p. ej. debounce del botón o límite de frecuencia de `animateCamera`).

Evidencia:
`lib/features/home/home_screen.dart` (`_loadCurrentLocation`, `_moveCameraToCurrentLocation`, `_resolveOriginAddress`).

**Actualización (`ORIGIN-ADDRESS-R1`, 2026-08-24): el guard de concurrencia de `fares/origin-address` ya está en `main`** (commit `afaadb4`, fast-forward de `test/origin-address-r1`). JuanJo reconfirmó en el emulador, con el guard puesto, que taps repetidos en "centrar en mi ubicación" ya no producen el freeze — la dirección se mantiene estable. Esto es consistente con la hipótesis de arriba (el ANR no dependía de la acumulación de llamadas a `fares/origin-address`, que era un bug real pero aparte): con ese bug corregido y el freeze ya sin reproducirse en el mismo emulador, la explicación más probable que queda en pie sigue siendo `animateCamera` contra `GoogleMap` en un emulador sin aceleración GPU. **Sigue sin probarse en un dispositivo físico** — no se cierra esta entrada hasta esa validación.

## Destinos sugeridos (`SUGGESTED-DESTINATIONS-R1`) — solo verificado el caso vacío, falta verificación visual con historial real

Estado:
PENDIENTE DE VERIFICACIÓN VISUAL. El checkpoint está `FINAL-CLOSED-ON-MAIN` (629→777ca05, ver `decisiones.md`) con `flutter analyze` limpio y 238/238 tests en verde, pero eso cubre la lógica (widget tests con historial simulado vía fakes) — no reemplaza ver la funcionalidad real contra una cuenta con historial real en STAGING.

Qué se validó:
JuanJo confirmó en el emulador el **caso de historial vacío**: una cuenta sin viajes `COMPLETED` no muestra ninguna sugerencia y el resto de la pantalla Home se comporta normal (sin espacio reservado, sin mensaje, sin romper nada). Al momento de este checkpoint, la cuenta de prueba usada no tenía viajes `COMPLETED` en STAGING — generarlos requiere completar el ciclo de vida completo de un ride (con un conductor, real o vía llamadas directas a la API) y no se hizo antes de fusionar.

Qué falta verificar cuando exista una cuenta con historial real:
- Aspecto visual de las pastillas de sugerencia (`_buildSuggestedDestinationChip`) — truncado de direcciones largas, espaciado, cómo se ven dos chips juntos.
- Que tocar una sugerencia efectivamente fije el destino, dibuje la ruta y dispare la cotización — la lógica está cubierta por test (`home_screen_test.dart`, grupo `SUGGESTED-DESTINATIONS-R1`), pero no se probó contra el backend real de STAGING.
- Que el orden por frecuencia se vea correcto con datos reales (un destino visitado varias veces apareciendo antes que uno visitado una sola vez).

Evidencia:
`lib/features/home/home_screen.dart` (`_buildSuggestedDestinationChip`, `_selectSuggestedDestination`); `docs/contexto/App-passenger/decisiones.md`, entrada `SUGGESTED-DESTINATIONS-R1`.

## Pin del destino tapado por la tarjeta flotante mientras se calcula la tarifa (`HOME-FLOW-R1`, 2026-08-27)

Estado:
ACEPTADO SIN CORREGIR — comportamiento transitorio, verificado en
emulador.

Qué se observó:
Al elegir un destino tocando el mapa en la zona alta de la pantalla, el
pin del destino queda tapado por la tarjeta origen/destino flotante
(overlay superior, ver `App-passenger/decisiones.md`, `HOME-FLOW-R1`)
mientras la tarifa todavía se está calculando.

Por qué se aceptó sin corregir:
Es transitorio: en cuanto llega la cotización, `_fitCameraToRoute`
reencuadra la cámara a la ruta completa (origen + destino + polyline) y
el pin queda visible. La ventana en la que el pin permanece tapado es
solo la duración de la llamada a `fares/estimate`, no un estado
persistente. Verificado en emulador por JuanJo — observación menor, sin
impacto funcional (el pin sigue existiendo y es tocable normalmente
apenas se reencuadra).

Evidencia:
`lib/features/home/home_screen.dart` (`_buildOriginDestinationCard`
como overlay superior, `_fitCameraToRoute`).

## Resultados de Google Places no vienen ordenados por cercanía (`HOME-FLOW-R1`, 2026-08-27)

Estado:
PENDIENTE. Sin evidencia de una causa dentro de esta app — comportamiento
observado del propio servicio de Google Places.

Qué se observó:
Al buscar "upeu" en `SearchDestinationScreen`, un resultado a ~4.x km de
distancia del origen aparece antes en la lista que otro resultado a
~1.4 km. `SearchDestinationScreen` muestra las predicciones en el orden
que entrega `places/autocomplete` — no aplica ningún ordenamiento propio
por distancia.

Por qué no se corrigió en este checkpoint:
Es el orden que devuelve Google Places, no una decisión ni un bug de
lógica de esta app. Corregirlo requeriría calcular distancia por
predicción (que Places no siempre expone sin una llamada adicional por
resultado a `getDetails`) y reordenar del lado cliente, o depender de un
parámetro de sesgo geográfico distinto en la llamada a Places —
ninguno de los dos se evaluó ni se implementó en este checkpoint.

Evidencia:
`lib/features/home/search_destination_screen.dart` (consumo de
`places/autocomplete`); observado en emulador por JuanJo con la
búsqueda "upeu".

## Notas de alcance de esta verificación

- No se ejecutó `flutter build apk`/`appbundle` completo (fuera de
  alcance de esta documentación; no se instalaron toolchains
  adicionales).
- No se ejecutó la app en un emulador/dispositivo real; los hallazgos
  de esta sección provienen de lectura de código, `flutter analyze` y
  `flutter test` únicamente.

## Campo de precio acepta texto libre

Estado:
**RESUELTO (`FARE-PANEL-R1`, etapa 2, 2026-08-27).** Preexistente;
no fue introducido por `HOME-LAYOUT-R1`.

El campo de texto libre del panel de precio de Home fue reemplazado
por un stepper no editable (`[ − S/ X.XX + ]`, rango S/3.00–S/50.00,
pasos de S/0.50) — ya no hay ningún `TextField` en el que se pueda
escribir una letra. El campo de monto que sí sigue siendo editable a
mano (`OfferFareScreen`, "Ofrece tu tarifa", etapa 4) tiene su propio
filtro de entrada (solo dígitos y un separador decimal,
`TextInputFormatter`) y clamp de rango al perder el foco — el hallazgo
original ("el input debería filtrar a dígitos") quedó cubierto ahí.

## Alineación del ancla del marcador sin validar en zoom máximo

Estado:
PENDIENTE DE CONFIRMACIÓN VISUAL EN DISPOSITIVO FÍSICO.

A ese nivel de zoom el mapa no muestra calles ni referencias contra las
cuales medir. El ancla está correcta por construcción
(`Offset(0.5, 1.0)` y la punta del PNG toca el borde inferior del
lienzo), pero falta confirmación visual en un dispositivo físico.

## Reformateo transversal en `home_screen.dart`

Estado:
LIMITACIÓN DEL HISTORIAL; sin cambio de comportamiento conocido.

Se ejecutó `dart format` sobre el archivo completo, tocando métodos de
otros checkpoints. No cambia el comportamiento, pero aproximadamente
780 registros del diff son solo whitespace y dificultan revisar el
historial de ese archivo.

## Estado "cotización vencida" nunca validado visualmente

Estado:
**RESUELTO (`FARE-PANEL-R1`, etapa 2, 2026-08-27) — por eliminación del
texto, no por validación visual del color.** Encontrado durante
`HOME-DESIGN-R1-PARCIAL` (2026-08-26) al intentar confirmar en
emulador el color corregido de ese texto (`textoSecundarioSobreOscuro`,
ver `decisiones.md`). El panel de precio rediseñado en `FARE-PANEL-R1`
dejó de mostrar el texto de vigencia/vencimiento de la cotización por
completo (compactación visual, etapa 2) — el mecanismo de fondo
(`_scheduleQuoteExpiryTimer`, auto-renovación silenciosa sin botón
manual) sigue funcionando exactamente igual, descrito abajo, solo que
ya no hay ningún texto en pantalla cuyo color haga falta validar. La
pregunta original ("¿se ve bien 'Cotización vencida' con su color
corregido?") queda sin objeto.

Qué se observó:
`_scheduleQuoteExpiryTimer` (`home_screen.dart`) programa un `Timer`
con la duración exacta hasta que la cotización vence; al disparar,
llama directo a `_estimateFare()` y renueva la cotización sola — el
Passenger nunca ve un botón manual para renovarla. En la práctica,
esto significa que el texto/estado "Cotización vencida" está diseñado
para **autocorregirse en el mismo instante en que aparecería**, con
red funcionando. Cortar la red (modo avión) no ayuda a verlo: el mismo
`Timer` dispara igual, `_estimateFare()` falla por falta de conexión,
y la tarjeta se reemplaza por "No se pudo conectar" en vez de mostrar
el estado vencido.

Qué falta para confirmar:
No se ha visto en pantalla real el texto "Cotización vencida" con su
color corregido. Dos caminos identificados, ninguno ejecutado todavía:
- Un test de widget (posiblemente con captura vía `matchesGoldenFile`)
  que fuerce ese estado con un `FareRepository` fake cuyo
  `estimateRide()` nunca resuelva, para que el `Timer` de renovación
  quede "colgado" a mitad de camino sin reemplazar la tarjeta.
- Adelantar el reloj del sistema del emulador más allá de la hora de
  vencimiento (el `Timer` de Dart corre sobre el reloj monotónico, no
  sobre la hora de pared, así que no debería dispararse antes de
  tiempo solo por adelantar la fecha) y disparar un `setState`
  cualquiera (p. ej. escribir en el campo de precio) para que la UI
  recalcule el estado con la hora adelantada.

Evidencia:
`lib/features/home/home_screen.dart` (`_scheduleQuoteExpiryTimer`,
`_isQuoteExpired`); `docs/contexto/App-passenger/decisiones.md`,
entrada `HOME-DESIGN-R1-PARCIAL`.
