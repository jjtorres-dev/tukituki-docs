# decisiones

Repositorio:
tukituki-passenger-app

Branch analizada:
main

Última actualización:
2026-08-27

Fuente de verdad:
Este documento es contexto auxiliar. Si contradice al código actual,
el código y los tests tienen prioridad.

---

## Actualización de estado de ride/pago por polling HTTP, no WebSocket

Estado:
ACTIVA

Qué se decidió:
El estado de un ride en curso se refresca con `Timer.periodic` cada 3
segundos (`ride_searching_screen.dart:75`), y el estado de un pago
cada 2 segundos (`ride_receipt_screen.dart:149`). El polling se
detiene explícitamente al llegar a un estado terminal
(`COMPLETED`/`CANCELLED`/`EXPIRED` para el ride;
`PAID`/`FAILED`/`EXPIRED`/`DISPUTED`/`VOIDED` para el pago).

Por qué:
[PENDIENTE: no hay comentario ni commit que explique por qué se eligió
polling en vez de WebSocket/SSE; solo se puede confirmar el mecanismo
elegido]

Alternativas descartadas:
[PENDIENTE: sin evidencia de que se haya evaluado otra alternativa en
este repo]

Evidencia:
`lib/features/ride/presentation/ride_searching_screen.dart:75-180`,
`lib/features/ride/presentation/ride_receipt_screen.dart:126-157`.

---

## Verificación de teléfono por OTP se volvió opcional en el registro

Estado:
ACTIVA (flujo OTP desconectado); código de OTP LEGACY (huérfano)

Qué se decidió:
Desde el commit `34e62eb` ("make passenger phone verification
optional"), `RegisterScreen` ya no navega a `/otp` tras registrar:
llama directo a `login()` y va a `/splash`. `OtpScreen`,
`OtpArguments` y la ruta `/otp` en `app_router.dart` siguen presentes
en el código pero **ningún flujo actual navega a esa ruta**
(confirmado por búsqueda: la única referencia a `/otp` fuera del
propio archivo del screen es su registro en `app_router.dart`).

Por qué:
[PENDIENTE: sin comentario o commit message que documente la razón de
negocio; solo se observa el cambio de comportamiento]

Alternativas descartadas:
Flujo anterior con verificación obligatoria por SMS/OTP antes de poder
iniciar sesión (`auth/otp/request`, `auth/otp/verify` — endpoints que
siguen existiendo en `AuthRepository` pero ya no se invocan desde
ninguna pantalla activa).

Evidencia:
Commit `34e62eb`; `lib/features/auth/presentation/register_screen.dart`;
`lib/core/router/app_router.dart:31-43`;
`lib/features/auth/presentation/otp_screen.dart` (huérfano).

---

## Prefijo telefónico fijo a Perú (+51)

Estado:
ACTIVA

Qué se decidió:
El input de teléfono solo pide 9 dígitos (`^[0-9]{9}$`) y la app
concatena `'+51$phone'` de forma hardcodeada tanto en login como en
registro. No hay selector de país ni configuración de prefijo.

Por qué:
[PENDIENTE: sin evidencia explícita, consistente con un producto
enfocado en un solo mercado (Perú), pero no hay comentario que lo
confirme]

Alternativas descartadas:
[PENDIENTE: sin evidencia]

Evidencia:
`lib/features/auth/presentation/login_screen.dart:53`,
`lib/features/auth/presentation/register_screen.dart:62`.

---

## Refresh de sesión single-flight con un solo reintento

Estado:
ACTIVA

Qué se decidió:
`AuthInterceptor` solo intenta refrescar el token una vez por request
fallida con 401 (`extra[_authRetryKey]`) y comparte una única
`Future<bool>?` de refresh (`_refreshFuture`) para que múltiples 401
concurrentes no disparen múltiples llamadas a `auth/refresh`. Si el
refresh falla, se limpia la sesión local.

Por qué:
[PENDIENTE: no hay comentario que documente el motivo explícito, pero
el propio código de `_refreshSession()` implica evitar refrescos
concurrentes redundantes]

Alternativas descartadas:
[PENDIENTE: sin evidencia]

Evidencia:
`lib/core/network/api_client.dart:71-235`.

---

## Montos de tarifa como strings decimales, comparados en centavos enteros

Estado:
ACTIVA

Qué se decidió:
Los montos de tarifa (`estimatedFare`, `passengerOfferFare`,
`proposedFare`, etc.) se transportan como `String` y se comparan
mediante `fareAmountInCents`, nunca como `double`.
`normalizePassengerOfferFare` valida el formato antes de enviar una
oferta al backend.

Por qué:
Evitar errores de precisión de punto flotante al comparar/negociar
montos de dinero (inferible directamente de la implementación —
`fare_amount.dart` normaliza a enteros de centavos antes de comparar).

Alternativas descartadas:
Comparar como `double`/`num` directo (el propio código evita
explícitamente ese camino).

Evidencia:
`lib/features/ride/domain/fare_amount.dart`;
`test/features/ride/domain/fare_negotiation_test.dart`.

---

## Perfil de pasajero como paso separado tras autenticarse

Estado:
ACTIVA

Qué se decidió:
Existe una entidad/paso independiente ("completar perfil": nombre y
apellido) después de login/registro y antes de poder operar en
`/home`, con su propio repositorio (`PassengerProfileRepository`) y
endpoint (`passengers/me`), distinto del `PublicUser` de auth.

Por qué:
[PENDIENTE: sin comentario explícito; se infiere de la separación de
endpoints (`auth/me` vs `passengers/me`) que la cuenta y el perfil de
pasajero son conceptos distintos en el backend]

Alternativas descartadas:
[PENDIENTE: sin evidencia]

Evidencia:
`lib/features/passenger/data/passenger_profile_repository.dart`;
`lib/core/router/app_router.dart:45-49` (`/complete-profile`);
`lib/features/auth/presentation/splash_screen.dart:126`.

**Corrección histórica (checkpoint `R4.1A`, 2026-08-18):** al momento en
que se escribió la entrada de arriba, este "paso obligatorio" **no lo
era realmente** — `SplashScreen._checkSession()` llamaba a
`getMyProfile()` pero descartaba el resultado sin usarlo (nunca
asignado a variable) y siempre continuaba a `/home`, sin importar si el
perfil existía. `CompleteProfileScreen`/`/complete-profile` compilaban
y funcionaban si se navegaba a ellos manualmente, pero ningún flujo
real (registro, login, reapertura de la app) los alcanzaba jamás. Esta
entrada quedó desactualizada respecto al código desde su redacción —
ver `R4.2` abajo para la corrección.

---

## Gate de identidad obligatorio antes de Home — corrige el flujo muerto de "Sobre ti" (R4.2)

Estado:
**FINAL-CLOSED-ON-MAIN. Integrado a
`main`@`91357d17cdb8e154124021b1ab9dc33a3bdd9ae6` mediante fast-forward
puro (`CROSS-APP-R4.2D`, 2026-08-18) y confirmado por
`MAIN-PHYSICAL-SMOKE-PASS` de JuanJo (`CROSS-APP-R4.2E`, 2026-08-18):**
cuenta con `PassengerProfile` → Home, persiste tras cerrar/reabrir;
cuenta sin `PassengerProfile` → "Completa tu perfil", persiste tras
cerrar/reabrir sin completar. `flutter analyze` limpio, 181/181 tests.
`test/r4-passenger-identity` eliminada (local y remota) tras confirmar
contención total en `main`.

Qué se decidió:

Regla oficial de producto (JuanJo, `R4.2B`/`R4.2C`, 2026-08-18),
aplicada por igual a cuentas nuevas y antiguas — **sin distinguir
"legacy" de "nueva" mediante ningún flag, timestamp, período de gracia
ni booleano local**:

1. Passenger sin `PassengerProfile` y sin ride activo → debe completar
   "Sobre ti" antes de continuar (`/complete-profile`).
2. Passenger sin `PassengerProfile` pero **con** ride activo → el ride
   activo tiene prioridad; nunca se interrumpe un viaje en curso por el
   formulario de perfil.
3. Cuando ese ride deje de estar activo, el gate vuelve a aplicar en la
   siguiente resolución (no hay excepción permanente: se recalcula
   fresco en cada entrada a `Splash`).
4. Passenger con `PassengerProfile` → routing normal (Home o ride
   activo, según corresponda).

`PassengerProfile` (no `User`) es la fuente canónica de identidad —
`User` no tiene `firstName`/`lastName` en Backend; `PassengerProfile`
ya los exige como no-nulos desde antes de este checkpoint. No fue
necesario ningún cambio de Backend.

Por qué:

`SplashScreen` es el único punto de entrada real a la app (registro →
login → `/splash`; login → `/splash`; reapertura → `/splash`), así que
corregirlo ahí cierra los tres caminos a la vez sin tocar
`RegisterScreen`/`LoginScreen`. La prioridad del ride activo sobre el
gate evita que una cuenta ya en medio de un viaje (típicamente una
cuenta creada antes de este fix) quede bloqueada por un formulario para
poder ver su propio viaje.

Alternativas descartadas:

- Distinguir cuentas "legacy" de "nuevas" mediante un flag persistido,
  `createdAt` de la cuenta, o un período de gracia — descartado
  explícitamente por decisión de producto: el gate se deriva siempre en
  vivo de `GET passengers/me` + `GET rides/active`, nunca de un dato
  local ni de la antigüedad de la cuenta.
- Asumir éxito local tras guardar el perfil y navegar directo a
  `/home` — descartado: `CompleteProfileScreen` vuelve a `/splash`
  tras guardar (o tras un 409 de "perfil ya existe") para que el mismo
  resolver reconsulte Backend, en vez de confiar en el resultado local
  de la escritura.

Consecuencia en código (`tukituki-passenger-app`, ya en
`main`@`91357d17`; rama `test/r4-passenger-identity` eliminada tras
`MAIN-PHYSICAL-SMOKE-PASS`, `CROSS-APP-R4.2E`):

- Nuevo `lib/features/auth/domain/passenger_session_state.dart`:
  `resolvePassengerSessionState()` (función pura, testeable sin
  `BuildContext`) + `routeForPassengerSessionState()`. Mismo patrón de
  "resolver separado del widget" que ya usa `tukituki-driver-app`
  (`driver_session_state.dart`), pero con solo 2 estados
  (`identityRequired`/`ready`) — el modelo Passenger no necesita el
  enum multi-estado de Driver porque `PassengerProfile` no tiene un
  ciclo de revisión administrativa.
- `SplashScreen._checkSession()` ahora usa ese resolver en vez de
  descartar el resultado de `getMyProfile()`.
- `CompleteProfileScreen._save()`: éxito y 409 ambos navegan a
  `/splash` (antes: `/home` directo, asumiendo éxito local).
- Nuevo `lib/core/display_name.dart`: `displayCompactName(firstName,
  lastName)` → "Juan Pérez" → "Juan P.", "Juan" (sin apellido) →
  "Juan", nunca "null" ni un "." suelto, nunca persiste el resultado.
  **Sin consumidor todavía** — es infraestructura preparada para
  `R4.3` (ficha del Driver mostrando el nombre del Passenger); no se
  tocó el getter equivalente ya existente del lado Driver
  (`PassengerRideOffer.driverDisplayName`) por estar fuera de alcance.
- Tests: 155 → 181 en la rama (26 nuevos: resolver puro, helper de
  nombre, `CompleteProfileScreen` completo, más cobertura de
  integración de las 4 combinaciones `hasProfile`×`activeRide` en
  `SplashScreen`). `flutter analyze` limpio.

Explícitamente NO tocado por este checkpoint:

- Backend (`tukituki-backend`, `test/otp-demo-staging` preservada sin
  tocar), `tukituki-driver-app`, Firebase/FCM, mapas/animación de
  marcador, DNI/CE/foto/correo/dirección/contacto de emergencia del
  Passenger (quedan para `R4.5`), Railway, producción.

Validación física (JuanJo, APK
`TukiTuki-Pasajero-R4.2-Identity-Test.apk` contra Backend STAGING,
2026-08-18): cuenta sin perfil → "Completa tu perfil" (no Home);
cerrar/reabrir antes de completar → vuelve a "Completa tu perfil";
completar y guardar → Home; cerrar/reabrir después de completar →
Home directo; logout/login con perfil completo → Home directo.
**PHYSICAL-REVIEW-PASS.**

Evidencia:
`lib/features/auth/domain/passenger_session_state.dart`;
`lib/features/auth/presentation/splash_screen.dart`;
`lib/features/passenger/presentation/complete_profile_screen.dart`;
`lib/core/display_name.dart`; `test/features/auth/domain/passenger_session_state_test.dart`;
`test/core/display_name_test.dart`;
`test/features/passenger/presentation/complete_profile_screen_test.dart`;
`test/features/auth/presentation/splash_screen_test.dart`.
Backend verificado por lectura directa (`src/modules/users/entities/user.entity.ts`,
`src/modules/passengers/entities/passenger-profile.entity.ts`,
`src/modules/passengers/passengers.controller.ts`), sin modificar
ningún archivo del repo Backend.

---

## Método de pago fijo a CASH al crear un ride

Estado:
ACTIVA

Qué se decidió:
`RideRepository.createRide` envía `'paymentMethod': 'CASH'` de forma
fija; no hay UI para elegir método de pago al solicitar un viaje,
aunque el modelo de recibo (`RideReceiptPayment`) sí modela otros
`status` de pago (`PENDING`, `PAID`, `FAILED`, `EXPIRED`, `DISPUTED`,
`VOIDED`) y otros campos como `cashReceived`/`changeGiven`.

Por qué:
[PENDIENTE: sin evidencia de justificación; solo se confirma el
comportamiento actual del código]

Alternativas descartadas:
[PENDIENTE: sin evidencia — el modelo de datos sugiere que el backend
soporta más de un método/estado de pago, pero el cliente no expone esa
elección todavía]

Evidencia:
`lib/features/ride/data/ride_repository.dart:20-40`;
`lib/features/ride/domain/ride_receipt.dart:81-133`.

---

## La app no distingue cancelación normal del conductor de un "no-show"

Estado:
PENDIENTE (limitación de backend, documentada explícitamente en código)

Qué se decidió:
`PassengerRide.cancelledBy` se mapea crudo desde el backend
(`PASSENGER`/`DRIVER`/`ADMIN`/`SYSTEM`) y la UI trata toda cancelación
con `cancelledBy == 'DRIVER'` igual, sin distinguir un no-show
confirmado. El código documenta explícitamente que no se debe inventar
esa distinción en el cliente.

Por qué:
El backend no expone todavía un campo tipo `cancellationType` a
Passenger (comentario explícito en el código).

Alternativas descartadas:
Inferir/heurísticamente distinguir no-show vs cancelación en el
cliente — descartado explícitamente por comentario ("No inventar esa
distinción aquí").

Evidencia:
`lib/features/ride/domain/passenger_ride.dart:72-83`;
`lib/features/ride/presentation/ride_searching_screen.dart:2515-2520`.

---

## Firma de build Android `release` reutiliza el `signingConfig` de `debug`

Estado:
ACTIVA (estado actual del repo, no se puede confirmar si es temporal)

Qué se decidió:
`android/app/build.gradle.kts` define
`buildTypes { release { signingConfig = signingConfigs.getByName("debug") } }`,
sin un `signingConfig` de release propio ni referencia a un
`key.properties`/keystore de producción en el repo (aunque
`android/.gitignore` sí ignora `key.properties` y `*.keystore`/`*.jks`,
lo que sugiere que se previó tenerlos en algún momento).

Por qué:
[PENDIENTE: sin evidencia; no hay comentario ni commit que lo explique]

Alternativas descartadas:
[PENDIENTE: sin evidencia]

Evidencia:
`android/app/build.gradle.kts:60-67`; `android/.gitignore`.

---

## `CROSS-APP-R4.3` — Identidad real del Driver, foto estable en memoria y visor ampliado (2026-08-18)

Estado:
**FINAL-CLOSED-ON-MAIN.** Integrado a `main`@`5c4f0f9136f2e49a4b746755963e01fa29aa3d49` mediante fast-forward puro (`CROSS-APP-R4.3I`, 2026-08-18) y confirmado por el smoke físico final de JuanJo sobre un APK construido específicamente desde `main` contra Backend STAGING desde `main` (7/7 PASS, incluye `DRIVER_ARRIVING`/`DRIVER_ARRIVED` con foto estable y visor). `flutter analyze` limpio, 196/196 tests. `test/r4-ride-identities` eliminada (local y remota) tras confirmar contención total en `main`.

Qué se decidió:
Primer consumidor real de `displayCompactName()`/`displayCompactNameFromInitial()` (preparado sin consumidor en `R4.2`): la tarjeta del Driver asignado usa `AssignedDriver.lastNameInitial` (nuevo campo, ya expuesto por Backend en `CROSS-APP-R4.3` Backend) para mostrar "Nombre + inicial" en vez de solo el nombre. Copy final: `DRIVER_ARRIVING` → "Tu conductor está en camino"; `DRIVER_ARRIVED` → "Tu conductor ya llegó" / "Identifica a tu conductor antes de subir." — `"Por llegar"` retirado de todo el código.

**Foto del Driver estable en memoria (bug físico encontrado y corregido en el mismo checkpoint)**: con un Driver real con foto, la imagen aparecía/desaparecía cada ciclo de `Timer.periodic` (3s). Causa raíz: el capability token firmado de Storage rota en cada respuesta aunque sea la misma foto; `Image.network` trataba cada URL nueva como un recurso distinto y soltaba el frame anterior mientras cargaba el nuevo. Fix (`_stableDriverPhotoUri` en `RideSearchingScreen`, solo memoria RAM del `State`, nunca disco/`SharedPreferences`/secure storage): alcance por `rideId` + `driverProfileId` (cambia cualquiera de los dos → reset total de la foto guardada); una capability nueva y válida del mismo Driver reemplaza a la anterior; un `photoUrl` null/inválido transitorio conserva la última foto válida en vez de tumbarla; `gaplessPlayback: true` en el `Image.network` para no mostrar blanco entre frames mientras carga la URL rotada.

**Visor de foto ampliada**: tap sobre el avatar solo cuando `ride.status == DRIVER_ARRIVED` y existe una foto real válida (nunca sobre el placeholder de iniciales, nunca en `DRIVER_ARRIVING`). `showDialog` liviano — sin ruta nueva, sin dependencia nueva —, `BoxFit.contain`, cierre por X/back Android/tap fuera del diálogo, siempre vuelve exactamente al mismo ride en `DRIVER_ARRIVED`. El avatar de la tarjeta y el visor leen la **misma** `_stableDriverPhotoUri` — nunca una lectura separada de `driver.photoUrl` crudo — para que "foto visible → tap → visor de esa misma foto" se mantenga cierto incluso durante la rotación del capability token.

Por qué:
`CROSS-APP-R4.1A` había identificado que el ride asignado ya exponía placa/marca/modelo/color/foto del Driver sin PII sensible, pero sin consumidor de nombre compacto ni manejo de estabilidad de imagen bajo polling — este checkpoint cierra ambos.

Alternativas descartadas:
Persistir la última foto válida en disco/`SharedPreferences` para sobrevivir un restart de la app — descartado explícitamente por política de privacidad (el capability token no debe sobrevivir el proceso).

Evidencia:
`lib/core/display_name.dart` (`displayCompactNameFromInitial()`), `lib/features/ride/domain/assigned_driver.dart` (`lastNameInitial`), `lib/features/ride/presentation/ride_searching_screen.dart` (`_stableDriverPhotoUri`, `_updateStableDriverPhoto()`, `_showDriverPhotoViewer()`, `_buildAssignedDriverCard()`). `flutter analyze` limpio, 196/196 tests (181→196). **`PHYSICAL-REVIEW-PASS`** confirmado por JuanJo sobre APK `TukiTuki-Pasajero-R4.3F-Foto-Estable-Viewer-Retest.apk`: identidad real en solicitud/propuesta/Driver seleccionado, copy final sin "Por llegar", foto real estable durante múltiples ciclos de polling en `DRIVER_ARRIVING` y `DRIVER_ARRIVED`, visor abre/cierra (X, back Android, reapertura) y la foto sigue estable después de usar el visor — los 7 tests físicos de `R4.3F` en PASS.

---

## `R4.4B` — Interpolación animada del marcador del Driver en el mapa (2026-08-20)

Estado:
**FINAL-CLOSED-ON-MAIN.** Integrado a `main`@`7d717ff4b908dd1c0d5f932fca30c280b03b5f8f` mediante fast-forward puro (2026-08-20), tras `PHYSICAL-REVIEW-PASS` de JuanJo (dispositivo real, campus universitario, terreno abierto — ver `estado-proyecto.md` checkpoint `R4.4B` para el detalle completo). `flutter analyze` limpio, **205/205 tests en verde** (196→205, +9 del grupo `R4.4B: interpolación visual del marker del Driver`). `test/r4-smooth-driver-marker` eliminada local y remotamente tras confirmar contención total. **Pendiente sin cerrar**: el umbral de salto grande (300m, `_driverMarkerLargeJumpMetersThreshold`) no se validó en zona de señal GPS difícil — la prueba física cubrió solo el caso base (terreno abierto, buena señal).

Qué se decidió:
`_RideRouteMapState` (dentro de `ride_searching_screen.dart`) pasa a ser un `State` con `SingleTickerProviderStateMixin` y un `AnimationController` propio (`_driverMarkerAnimationController`, duración 2800ms) que interpola linealmente la posición *visual* del marcador del Driver (`_displayDriverPosition`) entre la posición mostrada actualmente y la nueva posición recibida en cada poll (cada 3s), en vez de saltar directo a la coordenada nueva. La lógica vive en `_handleDriverLocationUpdate()`, con estas reglas, en este orden:
- Si cambia `rideId` o `driverProfileId` respecto al poll anterior → reset total sin animar, la nueva posición se muestra directa. Nunca se interpola entre dos conductores o dos rides distintos.
- Si `driverLocation` llega `null` de forma transitoria (mismo ride+Driver) → se conserva el último marcador válido, nunca se mueve a `origin`/`(0,0)`/desaparece.
- Primera posición válida para el ride+Driver actual → directa, sin animar.
- Diferencia de coordenada por debajo de `0.0000005` grados respecto al target actual → se ignora, no reinicia la animación (evita parpadeo/ticker innecesario por ruido de precisión numérica).
- Salto mayor a 300 metros respecto a la posición visual actual (distancia calculada con la fórmula de Haversine, función pura `_driverMarkerDistanceMeters`, sin agregar `geolocator` solo para esto) → snap directo sin animar, como hardening ante un salto de GPS evidente (teleport/reconexión).
- Cualquier otro caso → anima linealmente desde la posición **visual actual** (nunca desde el último target, para no "saltar hacia atrás" si llega una tercera posición C mientras se seguía animando de A hacia B) hacia la nueva posición.

Por qué:
`R4.4` (movimiento suave del marcador del Driver) quedó como el checkpoint `NEXT` tras el cierre de `CROSS-APP-R4.3` (`estado-proyecto.md`, sección 16). El polling de 3s (ya usado por `RideSearchingScreen` para refrescar el estado del ride) hacía que el marcador del Driver saltara de forma discreta y poco natural entre posiciones — el objetivo de este checkpoint es una animación visual suave sin cambiar el mecanismo de transporte (sigue siendo HTTP polling, nunca WebSocket) ni inventar datos de posición que Backend no envía (nunca ETA, nunca ruta simulada).

Alternativas descartadas:
- Interpolar siempre desde el último "target" (última posición recibida) en vez de la posición visual actual — descartado porque produce un salto visible hacia atrás si una nueva posición llega mientras la animación anterior todavía está en curso.
- No tener guard de salto grande y animar siempre — descartado porque un salto de GPS real (túnel, reconexión, teleport de emulador) produciría una animación larga y visualmente incorrecta cruzando zonas por las que el Driver nunca pasó; se prefirió un snap directo en ese caso.
- Añadir una dependencia (`geolocator` u otra) solo para el cálculo de distancia — descartado, se implementó Haversine como función pura sin dependencias nuevas.

Evidencia:
`lib/features/ride/presentation/ride_searching_screen.dart` (`_RideRouteMapState`, `_handleDriverLocationUpdate()`, `_onDriverMarkerAnimationTick()`, `_driverMarkerDistanceMeters()`); `test/features/ride/presentation/ride_searching_screen_test.dart`, grupo `'R4.4B: interpolación visual del marker del Driver'` (9 casos: primera posición directa, animación A→B con punto intermedio verificado, interrupción por una posición C sin salto hacia atrás, coordenada igual sin reiniciar animación, `driverLocation` null transitorio, cambio de conductor, cambio de ride, salto >300m con snap directo, dispose sin dejar Timer/Ticker pendiente). Commit `7d717ff4b908dd1c0d5f932fca30c280b03b5f8f`, rama `origin/test/r4-smooth-driver-marker`.

**`PHYSICAL-REVIEW-PASS`** confirmado por JuanJo (2026-08-20, dispositivo real, campus universitario, terreno abierto y buena señal GPS): movimiento del marcador fluido y sin tirones en el caso base. Con esa aprobación, `test/r4-smooth-driver-marker` se fusionó a `main` por fast-forward puro — ver checkpoint `R4.4B` en `estado-proyecto.md` sección 16 para el detalle completo de la integración. **Pendiente explícito, sin cerrar**: el umbral de salto grande (300m, `_driverMarkerLargeJumpMetersThreshold`) no se probó en zona de señal GPS difícil (interiores, zonas urbanas densas, túneles) — la prueba física cubrió solo el caso base. Ver también `errores-conocidos.md` sobre una referencia a un `AGENTS.md` que no existe en este repositorio.

---

## `DESIGN-SYSTEM-R1` — Infraestructura de tema, sin migrar pantallas (2026-08-20)

Estado:
EN CURSO, rama `test/design-system-r1`, sin commit todavía — JuanJo
va a probar en dispositivo físico antes de commitear. No confundir con
un checkpoint cerrado.

Qué se decidió (dos decisiones de diseño técnico, ninguna del sistema
de diseño en sí — ese es `sistema-de-diseno.md`, fuente de verdad para
colores/tipografía/medidas):

**1. Organización del tema en cuatro archivos bajo `lib/core/theme/`**,
uno por tipo de token (`passenger_colors.dart`, `passenger_typography.dart`,
`passenger_spacing.dart`) más uno que ensambla el `ThemeData`
(`passenger_theme.dart`). Cada archivo de tokens es una clase con
constructor privado (`const PassengerColors._()`) y solo miembros
`static const`, mismo patrón que ya usa `DriverPalette` en
`tukituki-driver-app` (`lib/core/theme/driver_palette.dart`) — se copió
esa convención en vez de inventar una nueva, y se extendió a
tipografía y medidas porque Driver todavía no las tiene tokenizadas
(`sistema-de-diseno.md` sección 8, "Aplicación al Driver" sigue
pendiente).

**2. El `ThemeData` global NO fija `fontFamily: 'Manrope'` a nivel de
tema**, aunque la fuente ya está declarada en `pubspec.yaml` (Paso 2 de
este checkpoint) y `PassengerTypography` ya expone los 9 estilos de la
sección 3 con `fontFamily: 'Manrope'` explícito por estilo.

Por qué:

Sobre la decisión 2, la que alguien podría querer "corregir" sin
contexto: si `ThemeData(fontFamily: 'Manrope')` se fija a nivel global,
Flutter lo propaga a través de `textTheme` a **todo** `Text` que no
especifique su propio `fontFamily` — es decir, cambiaría de Roboto a
Manrope el texto de las seis pantallas que hoy tienen sus estilos
escritos a mano (`_darkGreen`, `_ctaYellow`, etc., ver auditoría de
`DESIGN-SYSTEM-R1` en el reporte del checkpoint) sin que ninguna de
ellas lo pidiera explícitamente. El objetivo explícito de este
checkpoint era "infraestructura de tema, cero cambio visual salvo lo
que se herede del tema" (cursor, selección, ripple, barra de estado) —
fijar `fontFamily` global habría roto esa condición para toda la app de
golpe, no solo para los cuatro puntos ya documentados en
`errores-conocidos.md`. La migración real de cada pantalla a
`PassengerTypography` (y ahí sí, a Manrope) es la tarea siguiente,
pantalla por pantalla, no un interruptor global.

Alternativas descartadas:

- Fijar `fontFamily: 'Manrope'` en el `ThemeData` ahora, ya que la
  fuente estaba lista — descartada explícitamente por la razón de
  arriba: hace exactamente lo que este checkpoint tenía prohibido
  hacer (alterar el aspecto de pantallas no migradas).
- Un solo archivo `passenger_theme.dart` con todos los tokens
  inline — descartada para mantener el mismo patrón de un archivo por
  responsabilidad que ya usa `DriverPalette`, y porque mezclar colores,
  tipografía y medidas en un solo archivo dificulta encontrar un token
  cuando la migración de pantallas empiece.

Evidencia:
`lib/core/theme/passenger_colors.dart`, `passenger_typography.dart`,
`passenger_spacing.dart`, `passenger_theme.dart`; `lib/app.dart`
(`theme: PassengerTheme.light`); `tukituki-driver-app/lib/core/theme/driver_palette.dart`
(patrón de referencia); `errores-conocidos.md`, sección "Pantallas que
dependían del `ThemeData` por defecto" (los cuatro puntos activos que sí
cambian de aspecto por herencia del `ColorScheme`, no de la fuente).

---

## Campo único de nombre en "Completa tu perfil" (en vez de nombre y apellido separados) — propuesta sin decidir

Estado:
PENDIENTE DE DECIDIR. Ningún código tocado; es una observación de
producto surgida de revisar el registro de InDriver (2026-08-24),
compartida con la propuesta de WhatsApp de abajo.

Qué se propone:
Reemplazar los dos campos actuales de "Completa tu perfil"
(`firstName`/`lastName`) por un solo campo de nombre libre. El
razonamiento: el Driver solo necesita saber a quién recoge, no un
nombre legal completo separado en nombre y apellido.

Por qué:
Surge de comparar contra el registro de InDriver, que no separa
nombre/apellido en ningún punto de su flujo.

Qué falta para decidir:
- Qué espera Backend hoy: `PassengerProfile` exige `firstName` y
  `lastName` como no-nulos (ver entrada "Gate de identidad
  obligatorio antes de Home" arriba,
  `passenger-profile.entity.ts`) — pasar a un solo campo implica
  cambiar Backend, no solo la app.
- Qué muestra la app del Driver hoy con esos dos campos:
  `displayCompactName(firstName, lastName)` en
  `lib/core/display_name.dart` construye "Juan P." a partir de
  ambos (consumido desde `CROSS-APP-R4.3`, ver entrada arriba); un
  solo campo de nombre libre rompe ese formato o exige repensarlo.
- Sin fecha ni owner asignado.

Evidencia:
Ninguna en código todavía. Referencias a revisar cuando se retome:
`lib/features/passenger/presentation/complete_profile_screen.dart`;
`lib/core/display_name.dart`; Backend
`src/modules/passengers/entities/passenger-profile.entity.ts`.

---

## Código de verificación por WhatsApp como alternativa al SMS — propuesta sin evaluar

Estado:
PENDIENTE DE EVALUAR. Ningún código tocado; observación de producto
surgida de la misma revisión del registro de InDriver (2026-08-24)
que el punto anterior.

Qué se propone:
Ofrecer WhatsApp como canal alternativo (o reemplazo) del SMS para
el código de verificación del registro.

Por qué:
En Tarapoto el SMS puede demorar o no llegar; WhatsApp es de uso
prácticamente universal ahí.

Qué falta para evaluar:
Es un proyecto, no un ajuste — requiere la API de WhatsApp Business,
aprobación de Meta, y plantillas de mensaje registradas y aprobadas.
Sin evaluar costo ni tiempo todavía.

Evidencia:
Backend ya tiene un mecanismo de OTP construido para SMS
(`auth/otp/request`, `auth/otp/verify` — ver entrada "Verificación
de teléfono por OTP se volvió opcional en el registro" arriba:
actualmente desconectado del flujo activo de la app, pero presente
en código). Ninguna implementación de WhatsApp existe todavía.

---

## Contexto compartido: InDriver no usa contraseña — vale la pena revisar la arquitectura de auth de TukiTuki

Estado:
Nota de contexto para las dos propuestas de arriba, no una decisión
de producto en sí misma. PENDIENTE DE EVALUAR.

Qué se propone:
El registro de InDriver es número de teléfono → código de
verificación → nombre, sin contraseña en ningún punto. TukiTuki sí
pide contraseña al registrarse (`RegisterScreen`,
`RegisterPassengerDto.password` en Backend). Dado que Backend ya
tiene el mecanismo de OTP construido (ver entrada de arriba, aunque
hoy desconectado del flujo activo), vale la pena revisar más
adelante si la contraseña sigue siendo necesaria o si TukiTuki
podría migrar a un flujo solo-OTP como el de InDriver.

Por qué:
Surge de la misma revisión del registro de InDriver que originó las
dos propuestas de arriba (campo de nombre único, verificación por
WhatsApp) — se anota aquí como contexto compartido de ambas, no
como una propuesta separada.

Qué falta para evaluar:
Sin evaluar. Implica repensar la arquitectura de autenticación
completa (Backend + esta app), no solo la pantalla de registro.

Evidencia:
`lib/features/auth/presentation/register_screen.dart` (campo de
contraseña); Backend
`src/modules/auth/dto/register-passenger.dto.ts` (campo `password`
obligatorio); Backend `auth/otp/request`/`auth/otp/verify` ya
existen (ver entrada "Verificación de teléfono por OTP se volvió
opcional en el registro" arriba).

---

## `headerGrowthBudget` de `GradientHeaderSheet` se autolimita

Estado:
ACTIVA — comportamiento confirmado del código actual, verificado
empíricamente migrando `complete_profile_screen.dart` (2026-08-24).

Qué se decidió/observó:
Subir `headerGrowthBudget` no hace crecer el header indefinidamente.
`GradientHeaderSheet` fuerza `sheetMinHeight = totalHeight -
headerMaxHeight` como piso de altura de la hoja; mientras el
contenido real de `sheetChildren` sea más bajo que ese piso, el
`Column` centra el sobrante como padding y el header queda pegado a
`headerMaxHeight`. Pero en cuanto `headerMaxHeight` crece lo
suficiente para que ese piso caiga por debajo de la altura natural
del contenido, la hoja pasa a depender solo de su contenido — el
`ConstrainedBox` deja de ser la restricción activa — y el header deja
de crecer más allá de ese punto, sin importar cuánto se siga subiendo
el budget. Es decir: por encima de cierto valor, seguir subiendo
`headerGrowthBudget` es inofensivo (no hay riesgo de que el header
termine tragándose toda la pantalla), pero tampoco sigue teniendo
efecto.

Confirmado migrando `complete_profile_screen.dart` (dos campos, poco
contenido): con `headerGrowthBudget: 300` el residuo de la hoja bajó
a ~7px lógicos (prácticamente cero) con un header de ~47% de la
pantalla — evidencia de que ya se estaba cerca del punto de
saturación, no de que siguiera creciendo proporcionalmente al budget.

Por qué:
Se infiere directamente de la implementación de `GradientHeaderSheet`
(`sheetMinHeight`/`sheetMaxHeight` calculados a partir de
`headerMinHeight`/`headerMaxHeight`, `Column` con
`mainAxisAlignment.center` dentro de un `ConstrainedBox` con solo
`minHeight`) — no hay comentario explícito en el widget que lo
documente como comportamiento intencional, pero el mecanismo se
verificó leyendo el código y confirmando el resultado en pantalla.

Alternativas descartadas:
Ninguna — es una observación sobre el comportamiento existente, no
una decisión de diseño tomada en este checkpoint.

Evidencia:
`lib/core/widgets/gradient_header_sheet.dart` (`sheetMinHeight`,
`sheetMaxHeight`, `Column(mainAxisAlignment: MainAxisAlignment.center)`
dentro del `ConstrainedBox`); mediciones sobre
`complete_profile_screen.dart` en `headerGrowthBudget` 24/150/180/300
(2026-08-24).

---

## `G4B-CONTRACT-R1`: la app manda `destination.isManualSelection` explícito, ya no se infiere por texto (2026-08-24)

Estado:
**FINAL-CLOSED-ON-MAIN.** Commit `f31704a8863f323134eddcc25322d94e7d3eb2d5` ("feat: send explicit manual-selection flag for destination"), integrado a `main`@`f31704a` mediante fast-forward puro (2026-08-24). `flutter analyze` limpio, 211/211 tests. `test/g4b-contract-r1` eliminada (local y remota) tras confirmar contención total en `main`. Verificado en el emulador contra Backend STAGING: destino elegido en el mapa se resuelve a dirección real sin placeholder visible; destino elegido por búsqueda conserva nombre y dirección tal cual.

Qué se decidió:
`HomeScreen` agrega un estado local `_destinationIsManualSelection` (`bool`, `false` por defecto) que se pone en `true` únicamente cuando el destino se elige tocando el mapa (nunca por autocomplete/búsqueda), y en `false` en cualquier otro camino (autocomplete, reset). Esa bandera se manda a Backend como `destination.isManualSelection` en `FareRepository.getFareEstimate` (`fares/estimate`). La misma bandera reemplaza también la lectura local: antes, `HomeScreen` decidía si reemplazar la tarjeta por la dirección real resuelta por Backend comparando `_selectedDestinationAddress == _manualDestinationPlaceholder`; ahora usa directamente `_destinationIsManualSelection`. `_manualDestinationPlaceholder` (el literal `'Destino seleccionado en el mapa'`) se sigue mostrando en pantalla y enviando como `address` (Backend lo exige no vacío), pero ya no se compara contra nada como señal — ni en esta app ni en Backend, salvo en el fallback legado documentado del lado Backend.

Por qué:
Corresponde a la misma decisión de contrato tomada del lado Backend (ver `docs/contexto/Backend/decisiones.md`, entrada `G4B-CONTRACT-R1`): la señal de "destino elegido en el mapa" pasa de inferirse por comparación de texto contra un literal de copy de UI a ser un campo explícito del contrato, para no depender de que ambos repos mantengan ese literal sincronizado en silencio.

Condición de retiro del fallback legado (vive del lado Backend, no de esta app):
El Backend conserva temporalmente la comparación de texto como fallback solo para clientes que omitan `isManualSelection` — es decir, la única instalación pre-contrato (un APK de prueba en el celular físico de JuanJo, sin distribución pública). Se retira en cuanto esa instalación se reinstale con una build de esta app que ya incluya este commit (`f31704a` en adelante). No aplica ninguna acción futura de este lado más allá de asegurar que esa reinstalación ocurra.

Evidencia:
`lib/features/fare/data/fare_repository.dart` (`destinationIsManualSelection`), `lib/features/home/home_screen.dart` (`_destinationIsManualSelection`), `test/features/home/home_screen_test.dart`. Commit `f31704a`.

---

## `ORIGIN-ADDRESS-R1`: dirección real del origen visible apenas hay GPS, con cache por distancia y guard de concurrencia (2026-08-24)

Estado:
**FINAL-CLOSED-ON-MAIN.** Dos commits, integrados a `main`@`afaadb4` mediante fast-forward puro (2026-08-24): `9b9322b` ("feat: show the real origin address as soon as GPS is available") y `afaadb4` ("fix: guard fares/origin-address against duplicate in-flight requests", agregado tras un ANR reportado en el emulador durante la validación — ver `errores-conocidos.md`). `flutter analyze` limpio, 217/217 tests. `test/origin-address-r1` eliminada (local y remota) tras confirmar contención total en `main`. Verificado en el emulador: la dirección real del origen aparece apenas hay GPS, sin esperar destino, y taps repetidos en "centrar en mi ubicación" ya no producen el freeze reportado inicialmente.

Qué se decidió:
`FareRepository.getOriginAddress()` llama a `GET fares/origin-address` (nuevo endpoint del Backend, ver `docs/contexto/Backend/decisiones.md`). `HomeScreen._loadCurrentLocation()` lo dispara en paralelo (`unawaited`) apenas resuelve un fix de GPS — independiente de la cotización, no espera a que el pasajero elija destino. La prioridad de qué texto se muestra como "Origen" queda: `quote.originAddress` (si ya hay cotización) → `_resolvedOriginAddress` (resuelto preemptivamente) → `'Esperando GPS...'`/`'Tu ubicación actual'` (interino).

**Cache por distancia — `_originAddressCacheDistanceMeters = 50`** (constante nombrada y documentada, no un número suelto): si una nueva posición GPS cae dentro de 50 metros de la última posición para la que ya se resolvió una dirección, no se vuelve a llamar al backend — se reutiliza la dirección ya conocida. **Sin validar en calle todavía** — mismo estado que `_driverMarkerLargeJumpMetersThreshold` (300m, umbral de salto grande del marcador del Driver, `R4.4B`), que tampoco se probó en zona de señal GPS difícil. Ajustar si la prueba física muestra que 50m es muy chico (parpadea con jitter normal de GPS) o muy grande (no actualiza al moverse una cuadra corta).

**Guard de concurrencia (agregado tras el ANR del emulador, ver `errores-conocidos.md`)**: el guard `_locating` ya existente solo serializa la parte de `_loadCurrentLocation` (fetch de GPS + `animateCamera`) — no cubre `_resolveOriginAddress`, que se lanza con `unawaited` y por lo tanto puede seguir en vuelo después de que `_locating` ya volvió a `false`. Sin un guard propio, taps repetidos antes de que la primera respuesta llegara acumulaban varias llamadas concurrentes a `fares/origin-address` para prácticamente el mismo punto (el cache por distancia no alcanzaba a filtrarlas, porque solo se actualizaba al recibir una respuesta exitosa). Fix: `_originAddressInFlightPosition` + `_originAddressInFlightRequestId`, marcados **al iniciar** la llamada (no al recibir la respuesta) y limpiados al terminar (éxito o fallo). Deliberadamente en un campo separado de `_resolvedOriginAddressPosition` (que solo se mueve en éxito): mezclarlos habría dejado el cache de "ya resuelto" apuntando a un punto cuya dirección nunca se obtuvo si la llamada fallaba, bloqueando reintentos futuros para ese mismo lugar.

Por qué:
Hoy la dirección real del origen solo aparecía después de elegir destino (llegaba dentro de la cotización); antes de eso el pasajero veía "Tu ubicación actual", que no confirmaba si el GPS había acertado. Objetivo de producto: mostrar la dirección real desde el primer momento. El guard de concurrencia se agregó después de que JuanJo reportó un ANR en el emulador al tocar repetidamente "centrar en mi ubicación" — la investigación (ver `errores-conocidos.md`) no confirmó que la acumulación de llamadas fuera la causa directa del ANR (las llamadas HTTP async no bloquean el hilo principal por sí solas), pero sí confirmó que era un bug real y desperdiciaba cupo del límite por usuario del Backend (`ORIGIN_ADDRESS_RATE_LIMIT_MAX`) — se corrigió independientemente de si terminaba siendo la causa del freeze.

Alternativas descartadas:
Extraer la fórmula de Haversine (`_originAddressDistanceMeters`) a un util compartido con `_driverMarkerDistanceMeters` (`ride_searching_screen.dart`, R4.4B) — descartado para no tocar ese código ya aprobado físicamente en un checkpoint cerrado; se duplicó deliberadamente en su lugar.

Evidencia:
`lib/features/fare/data/fare_repository.dart` (`getOriginAddress`), `lib/features/home/home_screen.dart` (`_resolveOriginAddress`, `_originAddressCacheDistanceMeters`, `_originAddressInFlightPosition`, `_originAddressInFlightRequestId`, `_originAddressDistanceMeters`), `test/features/home/home_screen_test.dart` (grupo `ORIGIN-ADDRESS-R1`, incluye el caso "varios taps rápidos sin moverse producen una sola llamada al repositorio"). Commits `9b9322b`, `afaadb4`.

---

## `SUGGESTED-DESTINATIONS-R1`: destinos sugeridos en Home a partir del historial de viajes (2026-08-24)

Estado:
**FINAL-CLOSED-ON-MAIN.** Commit `777ca05` ("feat: show suggested destinations in Home from ride history"), integrado a `main`@`777ca05` mediante fast-forward puro (2026-08-24). `flutter analyze` limpio, 238/238 tests. `test/suggested-destinations-r1` eliminada (local y remota) tras confirmar contención total en `main`. Verificado en el emulador **solo el caso de historial vacío**: no aparecen sugerencias y la pantalla se comporta normal — ver `errores-conocidos.md` para lo que sigue pendiente con historial real.

Qué se decidió:
`RideRepository.getHistory({status: 'COMPLETED', limit: 50})` consume `GET passenger/rides/history` (antes nunca invocado desde esta app). `HomeScreen` lo pide una sola vez en `initState`, en paralelo, sin depender del GPS y sin persistir nada — se recalcula fresco cada vez que se monta una `HomeScreen` nueva.

`resolveSuggestedDestinations()` (`lib/features/home/domain/suggested_destinations.dart`, función pura sin `BuildContext`) calcula hasta `suggestedDestinationsCount` (constante nombrada, valor `2`, igual que InDriver) destinos por **frecuencia dentro de la ventana recibida — no recencia pura**: un lugar visitado varias veces le gana a uno visitado una sola vez más recientemente, con empate resuelto por el más reciente de los dos. No confía en que `history` venga ordenado — compara `requestedAt` explícitamente. Deduplica por `destinationAddress` normalizada (`trim` + minúsculas) y filtra los literales de fallback del Backend (`'Destino seleccionado'`, `'Destino seleccionado en el mapa'` — no son direcciones reales, sugerirlas no tiene sentido).

Al tocar una sugerencia (`_selectSuggestedDestination`): mismo camino que `_selectPlacePrediction` a partir de fijar el destino (mover cámara, `_maybeAutoEstimateFare()`), pero usando `destinationLatitude`/`destinationLongitude` del historial directamente — **sin `places/autocomplete` ni `getDetails`**, cero llamadas nuevas a Google, sin riesgo de resolver a un lugar distinto del tocado. La pastilla queda deshabilitada (sin `onTap`, atenuada) mientras `_currentPosition == null` — tocarla antes fijaría el destino igual, pero la cotización no dispara sin origen, así que se evita mostrar un control activo que no hace nada visible todavía.

Caso vacío (sin historial o sin destinos que pasen los filtros): la sección de sugerencias no se muestra — ni mensaje ni espacio reservado, mismo criterio que el resto de los enriquecimientos best-effort de esta pantalla. Un fallo de red al pedir el historial se trata igual (oculta, `debugPrint`, no bloquea nada).

**Limitación conocida, documentada en el propio código (`suggested_destinations.dart`), sin solución sin cambiar Backend**: el dedup por texto puede tratar como distintos dos formatos de la misma dirección (p. ej. con/sin ciudad) — no hay `placeId` ni coordenadas de origen suficientemente estables persistidas en el historial para deduplicar de forma más robusta.

Por qué:
El alcance original de esta tarea era solo la app (el endpoint de historial ya existía, pero sin coordenadas de destino). Se decidió tocar también Backend (ver `docs/contexto/Backend/decisiones.md`, misma entrada) porque resolver el texto vía Places al tocar una sugerencia costaba dos llamadas a Google por toque y, más grave, no garantizaba llegar exactamente al mismo lugar que generó ese texto — el pasajero tocaría "UPEU" y podría terminar fijando un punto distinto. Exponer las coordenadas ya calculadas en PostGIS elimina ambos problemas de raíz.

Alternativas descartadas:
- Recencia pura (los N destinos distintos más recientes) en vez de frecuencia — descartada: un viaje aislado de ayer no debería opacar un destino recurrente como el campus universitario, que es el caso de uso real que motivó esta función.
- Resolver coordenadas al tocar vía `places/autocomplete`+`getDetails` (el plan original, antes de decidir tocar Backend) — descartada por el costo y el riesgo de inexactitud ya explicados.

Evidencia:
`lib/features/ride/domain/ride_history_item.dart`, `lib/features/ride/data/ride_repository.dart` (`getHistory`), `lib/features/home/domain/suggested_destinations.dart`, `lib/features/home/home_screen.dart` (`_loadSuggestedDestinations`, `_selectSuggestedDestination`, `_buildSuggestedDestinationChip`). Tests: `ride_history_item_test.dart`, `suggested_destinations_test.dart`, `ride_repository_test.dart`, `home_screen_test.dart` (grupo `SUGGESTED-DESTINATIONS-R1`). Commit `777ca05`.

---

## "Hoja ceñida al contenido" y "header de un tercio de pantalla" son objetivos incompatibles con poco contenido

Estado:
ACTIVA — decisión de diseño tomada para
`complete_profile_screen.dart` (2026-08-24), documentada aquí porque
aplica a cualquier pantalla futura que use `GradientHeaderSheet` con
contenido corto.

Qué se decidió:
En una pantalla con `sheetChildren` cortos (p. ej. "Completa tu
perfil", solo dos campos), no existe un valor de
`headerGrowthBudget` que simultáneamente (a) deje el residuo de la
hoja cerca de cero y (b) mantenga el header en aproximadamente un
tercio de la pantalla — porque el punto en el que el residuo llega a
cero está determinado únicamente por `altura total de pantalla −
altura natural del contenido`, sin relación con qué tan "corto"
debería verse el header. En `complete-profile`, ese punto de residuo
cero cae con el header en ~47% de la pantalla (medido con
`headerGrowthBudget: 300`) — muy por encima de un tercio.

Se priorizó la proporción del header sobre cerrar el residuo:
`headerGrowthBudget: 180` deja el header en ~34% de la pantalla con
un residuo de ~113px lógicos en la hoja. Se descartó explícitamente
llevar el residuo a cero, porque un header al 47% con el logo en su
tamaño normal (92/150, igual que Login) se veía desbalanceado — el
verde dominando la pantalla con el logo "perdido" en el centro.

Por qué:
Revisión visual física (JuanJo) tras migrar `complete_profile_screen.dart`:
con `headerGrowthBudget: 24` (heredado de Register) la hoja tenía un
hueco grande antes del indicador de pasos; con `headerGrowthBudget: 300`
el hueco desapareció pero el header pasó a ocupar casi la mitad de
la pantalla. `180` fue el mejor punto encontrado probando 120/150/180
dentro del rango pedido.

Alternativas descartadas:
- Llevar el residuo a exactamente cero (`headerGrowthBudget` ~294,
  header ~47%) — descartado por verse desbalanceado.
- Achicar el logo en vez de agrandar el header — no ataca la causa
  (el residuo viene de que el *contenido* de la hoja es corto, no de
  que el logo sea grande) y esta pantalla, a diferencia de Register,
  no compite por espacio con un tercer campo, así que no había razón
  para mantener el logo chico.

Evidencia:
`lib/features/passenger/presentation/complete_profile_screen.dart`
(`headerGrowthBudget: 180`, comentario junto a `GradientHeaderSheet`);
mediciones en `headerGrowthBudget` 24/150/180/300 sobre APK de
desarrollo sobre emulador (2026-08-24).

---

## `HOME-LAYOUT-R1` — layout definitivo de Home y geometría del mapa (2026-08-26)

Estado:
**PUBLICADO EN RAMA, PENDIENTE DE FUSIÓN A `main`.** Implementado en
`test/home-layout-r1`@`a79abba22e33e0fa9af48ab82bfe90204a2736e7`,
verificado y aprobado por JuanJo en emulador. No queda ningún pendiente
funcional dentro del checkpoint; su integración a `main` requiere una
autorización posterior.

### Altura del marcador como constante única

La altura lógica del marcador de origen (`64`) vive en un solo lugar:
`_originMarkerLogicalSize.height`. Ese mismo valor alimenta tanto
`BitmapDescriptor.asset` como el desplazamiento de la etiqueta, al que
se suman `6` px lógicos de separación visual.

Motivo: el bug original fue tener ese número duplicado y desincronizado
del asset real mediante `_originMarkerIconHeight = 44`, valor heredado
del pin por defecto de Google. Si alguien vuelve a escribir la altura a
mano en dos sitios, el bug regresa.

### Reencuadre sincronizado con la medición real

El ajuste de cámara se dispara cuando la hoja reporta su nueva altura
medida y se programa post-frame, agrupando cambios consecutivos de
geometría. Ya no depende de un retraso fijo.

Antes había `120 ms` fijos, que eran una carrera contra el layout, no
una solución: si el padding todavía correspondía a la hoja pequeña, la
cámara encuadraba con geometría vieja y el origen quedaba oculto.

### El reencuadre se dispara por cotización, no por sesión

Una cotización nueva siempre reencuadra, aunque el usuario haya movido
el mapa antes. Un cambio de altura sin cotización nueva —por ejemplo,
el teclado— no reencuadra si el usuario ya movió el mapa manualmente
después del último encuadre automático.

La operación está protegida por generación de destino/cotización para
que una respuesta tardía no mueva la cámara.

### Margen de encuadre: 48 px

El margen de `newLatLngBounds` queda fijado en `48` px lógicos. Se
eligió deliberadamente por debajo de valores mayores porque los viajes
típicos son urbanos y cortos —menos de 1 km—: un margen amplio aleja
tanto la cámara que se pierde el detalle de calles que el pasajero
necesita para ubicarse. Validado en rutas de 0.7 km y 5.9 km.

### Franja de barra de estado fija en verde de marca

La franja reservada detrás de la barra de estado usa
`PassengerColors.verdeMarca`; no sigue directamente el tema del
sistema. La app tiene un solo tema hoy y una franja adaptativa dejaría
negro sobre crema para usuarios con el celular en modo oscuro, siendo
el único elemento oscuro de la pantalla.

El modo oscuro está planificado como checkpoint propio. Cuando llegue,
la franja deberá seguir al **tema de la app**, no directamente al
sistema, y el cambio se hará desde el token.

**PREGUNTA ABIERTA:** si el modo oscuro seguirá al sistema o será un
ajuste dentro de la app. Depende del menú lateral, que aún no existe.

Evidencia:
`tukituki-passenger-app`, rama `test/home-layout-r1`, commit
`a79abba22e33e0fa9af48ab82bfe90204a2736e7`;
`lib/features/home/home_screen.dart`;
`test/features/home/home_screen_test.dart`. `flutter analyze` limpio,
242/242 tests en verde y validación de emulador aprobada por JuanJo.

---

## `DESIGN-SYSTEM-R2` — Tokens nuevos para migrar Home, sin migrar todavía (2026-08-26)

Estado:
**FINAL-CLOSED-ON-MAIN.** Rama `test/design-system-r2`, creada desde
`main`@`a79abba`, integrada por fast-forward puro (`main` avanzó a
`3dc1a219b8134237be1e7d5e8a786545ac60dcc2`, push a `origin/main`
confirmado). `flutter analyze` limpio, 242/242 tests en verde (sin
cambio de conteo — el checkpoint solo agrega declaraciones `static
const`, ningún comportamiento nuevo). Rama eliminada local y
remotamente tras confirmar contención total. **Alcance deliberadamente
acotado a tokens: `home_screen.dart` no se tocó en ningún momento.**

Qué se decidió:

**1. El color `aviso` (`#B8641E`, nuevo) no reutiliza `alerta`
(`#E8951A`).** `alerta` es idénticamente el mismo hex que el acento de
la app del Conductor (`sistema-de-diseno.md` sección 2, "Acento").
Usarlo de forma prominente en Home — la pantalla de mapa a pantalla
completa que el pasajero ve durante todo el viaje, no una pantalla de
formulario ocasional como Login/Registro — erosiona la señal de
reconocimiento entre las dos apps que describe la sección 1 del
sistema de diseño: "el pasajero, al subirse de noche, reconoce de un
vistazo que la pantalla que le muestra el conductor es realmente la
app del conductor". Un ocre distinto (`aviso`) mantiene la familia
cálida de la paleta sin arriesgar esa señal.

**2. Tampoco reutiliza `destino` (`#D8542C`).** Antes de este
checkpoint, `home_screen.dart` usaba `destino` también para el aviso
de "sin ubicación" — mezclaba "esto es el pin de destino en el mapa"
con "esto necesita tu atención", dos significados que no deberían
compartir un mismo color.

**`aviso` queda PROVISIONAL, aprobado explícitamente como tal por
JuanJo (2026-08-26)**: no se pudo validar en pantalla en este
checkpoint porque los tokens no se usan en ninguna vista todavía. Se
valida recién en la migración real de `home_screen.dart`, con atención
especial al caso "cotización vencida" — ese texto va sobre
`verdeMarca` (verde oscuro), no sobre `crema`; un ocre oscuro como
`aviso` sobre un fondo oscuro puede quedar con poco contraste, a
diferencia de su uso sobre `fondoAviso`/`crema` (claro). Si falla, el
valor se corrige en un solo lugar (`passenger_colors.dart`).

**3. `TukiSearchBar` como componente separado, no como parámetro de
`TukiTextField`.** La barra de búsqueda de destino de Home cambia la
**estructura** respecto al patrón de "Campo de texto"
(`TukiTextField`), no solo valores puntuales: alto flexible en vez de
fijo (56), un ícono de lupa en posición fija en vez de un slot
`prefix`/`suffix` genérico, y dos estados mutuamente excluyentes a la
derecha (spinner de carga / botón de limpiar) en vez de un `suffix`
libre. Meterlo como flag booleano en `TukiTextField` habría obligado a
ese widget a ramificar buena parte de su `build()` según el flag —en
la práctica, mantener dos componentes dentro de un mismo archivo con
un `if` en el medio. Un widget separado que reutiliza los mismos
tokens de color (`bordeCampo`, `verdeMarca`, `radioCampoBoton`) es más
simple de leer y de testear por separado, y deja margen si la barra
termina necesitando comportamiento propio (debounce, lista de
predicciones acoplada). **No se implementó en este checkpoint** —
queda documentado como recomendación en `sistema-de-diseno.md`
("Barra de búsqueda") para el checkpoint que migre Home.

**4. El radio `15` del campo de búsqueda actual NO se tokenizó.** No
es una decisión de diseño — es un literal cercano al `14`
(`radioCampoBoton`) del sistema, sin intención detrás. Se fuerza a
`radioCampoBoton` en la migración de `home_screen.dart`, no se agrega
un token nuevo solo para preservar un valor que nunca fue elegido a
propósito.

Además, en el mismo checkpoint: dos tokens nuevos para texto sobre
fondo oscuro (`textoTenueSobreOscuro`/`textoSecundarioSobreOscuro`,
nombrados por rol, no por color — primer caso del sistema fuera del
header degradado), y dos radios nuevos sin decisión pendiente
(`radioPildora` = 30 para chips/píldoras, `radioEtiquetaMarcador` = 11
para la etiqueta del marcador de origen) — el sistema solo contemplaba
14 y 26 porque se definió sobre pantallas sin mapa ni chips.

Por qué:

Preparar la migración de `home_screen.dart` al sistema de diseño sin
mezclar la decisión de qué token usar con la implementación del
cambio visual — permite a JuanJo revisar y aprobar cada token (en
particular el color `aviso`, marcado explícitamente provisional) antes
de que se toque código de pantalla.

Alternativas descartadas:

- Reutilizar `alerta` (`#E8951A`) para el nuevo color de aviso —
  descartada por el riesgo de confusión de marca con el Conductor (ver
  punto 1).
- Seguir reutilizando `destino` (`#D8542C`) para el aviso de "sin
  ubicación" — descartada por mezclar dos significados distintos (ver
  punto 2).
- Parámetro `isSearchBar`/similar en `TukiTextField` en vez de un
  componente separado — descartada por acoplar dos formas visuales
  distintas a un mismo widget y hacer más frágil cualquier cambio
  futuro a "Campo de texto" (ver punto 3).
- Tokenizar el radio `15` actual del campo de búsqueda tal cual, para
  no "perder" el valor — descartada porque ese valor nunca fue una
  decisión de diseño, solo un literal cercano al token ya existente.

Evidencia:
`tukituki-passenger-app`, rama `test/design-system-r2`, commit
`3dc1a219b8134237be1e7d5e8a786545ac60dcc2`;
`lib/core/theme/passenger_colors.dart`,
`lib/core/theme/passenger_spacing.dart`,
`lib/core/theme/passenger_typography.dart`. `docs/contexto/sistema-de-diseno.md`
(secciones 2, 3, 4 y 5, marcado `[R2]`). `flutter analyze` limpio,
242/242 tests en verde antes y después de la fusión a `main`.

---

## `HOME-DESIGN-R1-PARCIAL` — migración de colores de Home + `TukiSearchBar`, sin tipografía ni medidas (2026-08-26)

Estado:
**FINAL-CLOSED-ON-MAIN.** Rama `test/home-design-r1`, tres commits
(`eea548b`, `6b2b743`, `ce3680a`), integrada a `main`@`ce3680a91055dcaf913b4d9704a6b74877694cfe`
mediante fast-forward puro. `flutter analyze` limpio, 249/249 tests en
verde (242 baseline + 7 de `TukiSearchBar`). Rama eliminada local y
remotamente tras confirmar contención total.

Qué se decidió:

**Eliminar los íconos decorativos de los chips de métricas y del
reloj de vigencia de cotización.** Dos razones convergentes, no una
sola: (1) fallaban el mínimo de contraste de WCAG sobre `verdeMarca`
— el ícono de cada chip (`acento` sobre el overlay translúcido del
chip) daba 2.33:1, un umbral de 3:1 exigido para elementos gráficos;
el ícono de reloj (`aviso` sobre `verdeMarca`) daba 2.90:1; (2) no
aportaban información que el texto que los acompaña no diera ya
("3.2 km" ya dice que es distancia sin un ícono de ruta al lado). La
referencia de InDriver, revisada como parte de esta decisión, tampoco
usa íconos en ese lugar. Los tres textos se conservaron sin cambio de
posición ni tamaño.

**El chevron de la etiqueta del marcador SÍ se conservó, recoloreado
a `blanco` (12.5:1, no `acento`).** No es decoración — es el
afordance que indica que la etiqueta es tocable para ajustar el punto
de recogida; sin él nadie descubre esa función. Es la única excepción
a la regla de "eliminar lo que falla contraste": acá el elemento sí
aporta información (affordance de interacción), así que se corrige el
color en vez de quitarlo.

**`acento` (`#1F7A3E`) no debe usarse sobre `verdeMarca`.** Confirmado
por cálculo de contraste WCAG (luminancia relativa, no impresión
visual): 2.33:1 contra `verdeMarca` puro, y **peor todavía, 1.61:1**,
contra el fondo real compuesto de los chips (`verdeMarca` con un
overlay blanco al 12% de opacidad encima) — el overlay aclara el
fondo hacia un tono más cercano a la propia luminancia de `acento`,
así que en vez de ayudar, empeora el contraste real. Para texto o
íconos sobre fondo oscuro, el sistema ya tiene `textoTenueSobreOscuro`
(`#8FA891`, 4.87:1) y `textoSecundarioSobreOscuro` (`#B9C8BC`, 7.17:1)
— ninguno de los dos es `acento`.

**Consecuencia sobre `aviso`**: con el ícono de reloj eliminado y el
texto de "cotización vencida" movido a `textoSecundarioSobreOscuro`,
`aviso` quedó con un solo uso en toda la app — el ícono de la caja
"sin ubicación", sobre `crema` (4.10:1, pasa cómodo). Se descartó
partirlo en dos tokens (uno para fondo claro, otro para oscuro) porque
ya no hace falta: no queda ningún uso sobre fondo oscuro que resolver.
Se le quitó la marca de "provisional" en `passenger_colors.dart` y en
`sistema-de-diseno.md`.

**Encontrado durante la Etapa 1, no en la auditoría original**:
`_secondaryGreen` (el color de la línea de ruta sobre el mapa) **no
era código muerto** — una auditoría previa lo había dado por no
renderizado; se verificó línea por línea que sigue coloreando la
polyline real (`GoogleMap(polylines: _polylines)`) cada vez que hay
una cotización con ruta. Se le dio token propio, `lineaRuta`
(`#5C8A17`, sin cambio de valor), en una categoría nueva del sistema
("Superposición sobre el mapa") en vez de forzarlo a `acento` — ver
`sistema-de-diseno.md`.

**Corrección de rol no pedida explícitamente**: la caja de "sin
ubicación" usaba `destino` (el color del pin de destino) antes de
este checkpoint. Se migró a `aviso`/`fondoAviso` en vez de a `destino`
1:1 — es exactamente el caso que motivó crear `aviso` en
`DESIGN-SYSTEM-R2` (evitar mezclar "esto es el pin de destino" con
"esto necesita tu atención"), así que perpetuarlo con `destino` habría
vaciado de sentido al token nuevo.

Alcance deliberadamente parcial:

**Tipografía y medidas de `home_screen.dart` NO se migraron.**
Decisión explícita de JuanJo: Home va a reestructurarse en el
rediseño de flujo que sigue a este checkpoint, y migrar tamaño y
tipografía de una pantalla que va a cambiar de forma habría sido
trabajo duplicado. `home_screen.dart` sigue sin importar
`PassengerTypography` — todo su texto sigue en la fuente por defecto,
igual que antes de este checkpoint.

Por qué:

Plan de 5 etapas aprobado por JuanJo antes de escribir código (colores
→ tipografía → medidas → limpieza de código muerto), pensado para
poder aislar una regresión física sin deshacer trabajo si algo se
rompía en el mapa (mismo mecanismo que causó el bug que corrigió
`HOME-LAYOUT-R1`: la altura medida de la hoja alimenta el padding del
mapa y el reencuadre de cámara). Solo se autorizaron y ejecutaron las
etapas 0 y 1; las etapas 2-4 (tipografía, medidas, limpieza) quedaron
sin empezar cuando JuanJo decidió priorizar el rediseño de flujo antes
de seguir.

Alternativas descartadas:

- Forzar el color de los tres íconos decorativos en vez de quitarlos
  (p. ej. a `blanco`) — descartada porque, a diferencia del chevron,
  no aportaban información nueva; quitarlos es más simple que
  recolorearlos sin necesidad.
- Partir `aviso` en dos tokens (claro/oscuro) — descartada porque, tras
  quitar el único uso sobre fondo oscuro, ya no había un segundo
  contexto que resolver.
- Migrar `_secondaryGreen` a `acento` — descartada explícitamente por
  JuanJo: es un color validado contra el mapa real, no contra la
  paleta de interfaz; forzarlo a `acento` lo habría oscurecido sin
  probar ese cambio en calle.
- Migrar la caja de "sin ubicación" a `destino` 1:1, siguiendo la
  regla general de "los que tienen token exacto migran 1:1" — descartada
  por la razón de rol explicada arriba.

Evidencia:
`tukituki-passenger-app`, rama `test/home-design-r1`, commits
`eea548b` (TukiSearchBar), `6b2b743` (colores), `ce3680a` (corrección
de contraste); `lib/core/theme/passenger_colors.dart`,
`lib/core/widgets/tuki_search_bar.dart`,
`test/core/widgets/tuki_search_bar_test.dart`,
`lib/features/home/home_screen.dart`.
`docs/contexto/sistema-de-diseno.md` (sección 2, nota de `aviso`
actualizada). `flutter analyze` limpio, 249/249 tests en verde.

---

## `HOME-FLOW-R1` — rediseño del flujo de Home: búsqueda aparte, tarjeta flotante, recentrado al limpiar (2026-08-27)

Estado:
**FINAL-CLOSED-ON-MAIN.** Rama `test/home-flow-r1`, cuatro commits
(`b974dcc`, `609d384`, `41d910a`, `45dcff2`), creada desde
`main`@`ce3680a`, integrada por fast-forward puro
(`main`@`45dcff24379e09b42662b5c2bbcbdf90ddf955d1`). `flutter analyze`
limpio, 258/258 tests en verde. Verificado y aprobado por JuanJo en
emulador. Rama eliminada local y remotamente tras confirmar contención
total. Ver `historial-checkpoints.md` para el detalle etapa por etapa.

Qué se decidió:

**1. La búsqueda es pantalla aparte (`SearchDestinationScreen`); el
estado con destino ya elegido NO lo es — es una rama de la propia
Home.** Antes de este checkpoint, buscar y tener un destino elegido
convivían dentro de la misma `HomeScreen` (campo inline + hoja
inferior). Ahora buscar abre una pantalla dedicada que solo existe
mientras se escribe; en cuanto se elige un resultado, su
`SearchDestinationResult` vuelve a `HomeScreen` y esa pantalla se
descarta — el estado "con destino" sigue viviendo en `HomeScreen`
(tarjeta flotante + hoja con cotización/oferta), nunca en la pantalla
de búsqueda.

Por qué:
Con el teclado abierto, el mapa detrás de un campo inline es
prácticamente inservible (tapado en más de la mitad de la pantalla) —
una pantalla dedicada libera esa restricción mientras se escribe. Además,
la lista de predicciones de autocomplete crece con cada tecla; hospedarla
dentro de la misma hoja que ya tiene que mostrar cotización, oferta y
CTA la habría obligado a competir por espacio con contenido que no
tiene relación con el acto de escribir una búsqueda.

**2. El botón atrás desde Home-con-destino limpia el destino y vuelve
a Home vacío, no a la pantalla de búsqueda.** Un `PopScope` (`canPop:
destination == null`) intercepta el pop mientras hay destino elegido y
llama a `_clearDestination()` — el mismo método que ya usaba "Quitar
destino" en la tarjeta. No hay ningún camino, ni back físico ni el
botón de la tarjeta, que devuelva a `SearchDestinationScreen` con el
destino ya elegido.

Por qué:
Volver a una búsqueda ya cumplida no tiene sentido para el usuario — el
destino ya está fijado y confirmado; retroceder a la pantalla que sirvió
para elegirlo no es una acción que el pasajero esperaría poder deshacer
por partes. Volver a Home vacío es la única acción de "atrás" coherente
en este punto del flujo.

**3. Al limpiar el destino, la cámara recentra en el origen con el
mismo zoom de entrada (16).** `_clearDestination()` termina con
`unawaited(_moveCameraToCurrentLocation())` — reutiliza exactamente la
misma función que ya usa el botón de recentrar (a través de
`_loadCurrentLocation()`) y el fallback de `onMapCreated`; mismo target
(posición GPS actual) y mismo zoom (16). No se escribió ningún camino de
cámara nuevo.

Por qué:
El usuario vuelve a Home vacío específicamente para elegir otro
destino. Sin este fix, la cámara se quedaba en el último encuadre de
ruta (zoom alejado, a nivel región) — a ese nivel no se distinguen
calles ni es posible tocar el mapa con precisión para elegir un punto
nuevo. Verificado en emulador que era el comportamiento real antes del
fix (reportado explícitamente por JuanJo tras probar "atrás"/"Quitar
destino").

La protección de gesto manual existente (`_cameraMovedByUserSinceRouteFit`,
`HOME-LAYOUT-R1`) no entró en conflicto con este recentrado: al momento
de mover la cámara, `_advanceDestinationGeneration()` (invocado al
principio de `_clearDestination()`) ya invalidó cualquier reencuadre de
ruta pendiente y dejó `_activeRouteQuoteGeneration` en `null` —
`_markCameraMovedByUser()` es un no-op en ese estado, así que no había
ningún ajuste automático activo con el que este recentrado pudiera
pelear. No hizo falta resolver ningún conflicto real entre ambas reglas.

**4. Se eliminó el footer de CTA en Home vacío.** No se trata de haber
quitado un botón dentro de un footer que sigue existiendo — el estado
sin destino directamente no tiene footer.

Por qué:
Sin destino elegido no hay ninguna acción de CTA que ofrecer todavía
(ni cotizar ni pedir viaje) — un footer vacío o con un botón
deshabilitado habría sido espacio de pantalla sin propósito, además de
competir con el mapa recién liberado del punto 1.

**5. Los destinos sugeridos se quedan en Home, no en la pantalla de
búsqueda, y pasaron de chips horizontales a lista vertical.**
Preservan su máximo de `suggestedDestinationsCount` = 2, sin cambio
(`SUGGESTED-DESTINATIONS-R1`).

Por qué:
En formato chip la dirección truncaba (verificado en emulador durante
`HOME-FLOW-R1`, etapa 3); en lista vertical se lee completa. Además, se
verificó que en la práctica los chips terminaban apilándose
verticalmente de todas formas — el formato de chip no aportaba nada
sobre una lista real. Quedan en Home (no en `SearchDestinationScreen`)
porque son un atajo para no tener que buscar — moverlas a la pantalla
de búsqueda habría sido contradictorio con su propio propósito.

**6. El overlay superior (menú + tarjeta) se mide con el mismo
mecanismo `_MeasureSize` que ya usaba el overlay inferior, y ambos
delegan en un único `_handleOverlaySizeChanged`.** `GoogleMap.padding`
pasa a usar `top` y `bottom` simultáneamente (antes solo `bottom`).

Por qué:
Mismo patrón que ya resolvió `HOME-LAYOUT-R1` para el overlay
inferior — el reencuadre de cámara y el centrado del marcador propio
dependen de conocer el área de mapa realmente libre; con la tarjeta
ahora flotando arriba en vez de vivir dentro de la hoja, esa área libre
depende también de la altura del overlay superior (incluida la llegada
asíncrona de la dirección de origen, que puede cambiar su alto). Un
mecanismo único evitaba mantener dos copias de la misma lógica de
"programar reencuadre cuando cambia la altura medida".

**7. El origen en la pantalla de búsqueda es de solo lectura.**
`SearchDestinationScreen` recibe `originAddress`/`originCoordinates`
como datos ya resueltos por `HomeScreen`; no hay ningún control para
editarlos desde ahí.

Por qué:
Editar el origen es, en efecto, el ajuste del punto de recogida — una
funcionalidad de producto distinta y más amplia que "buscar destino",
que además choca contra el límite real de 30 geocodificaciones por hora
del endpoint (`fares/origin-address`, ver `ORIGIN-ADDRESS-R1`). Se
decidió dejarla fuera de este checkpoint y tratarla como un checkpoint
aparte cuando se aborde, en vez de agregarla de paso aquí.

Alternativas descartadas:

- Mantener el campo de búsqueda inline dentro de la hoja de Home (el
  diseño anterior) — descartada por el problema de espacio con el
  teclado abierto y la lista de predicciones creciente (ver punto 1).
- Que el botón atrás devuelva a `SearchDestinationScreen` en vez de a
  Home vacío — descartada explícitamente: no tiene sentido reabrir una
  búsqueda ya resuelta (ver punto 2).
- Reutilizar el flujo completo de `_recenterOnCurrentLocation()`
  (incluida la re-adquisición de GPS vía `_loadCurrentLocation()`) para
  el recentrado al limpiar destino — descartada: el pedido era
  específicamente de cámara, y disparar una relectura de GPS completa
  (con sus efectos colaterales: `_locating`, mensajes de error,
  re-resolución de dirección de origen) para una acción que no lo
  necesita habría sido un efecto secundario no pedido. Se reutilizó en
  cambio `_moveCameraToCurrentLocation()`, la pieza de cámara pura que
  ya comparten tanto el botón de recentrar como el fallback de
  `onMapCreated`.
- Dejar un footer vacío o con CTA deshabilitado en Home vacío en vez de
  quitarlo del todo — descartada, sin ninguna acción real que ofrecer
  todavía (ver punto 4).
- Mover los destinos sugeridos a `SearchDestinationScreen` — descartada
  por ser contradictoria con su propósito de atajo (ver punto 5).
- Permitir editar el origen desde `SearchDestinationScreen` en este
  mismo checkpoint — descartada por alcance y por el límite de
  geocodificaciones por hora; queda como checkpoint aparte (ver punto
  7).

Evidencia:
`tukituki-passenger-app`, rama `test/home-flow-r1`, commits `b974dcc`,
`609d384`, `41d910a`, `45dcff2`; `lib/features/home/home_screen.dart`
(`_clearDestination`, `_moveCameraToCurrentLocation`,
`_handleOverlaySizeChanged`, `PopScope`/`onPopInvokedWithResult`),
`lib/features/home/search_destination_screen.dart`,
`lib/features/home/domain/search_destination_result.dart`,
`test/features/home/home_screen_test.dart`,
`test/features/home/search_destination_screen_test.dart`. `flutter
analyze` limpio, 258/258 tests en verde. Verificación en emulador por
JuanJo del recentrado al limpiar destino (punto 3) y del pin
transitoriamente tapado por la tarjeta flotante durante el cálculo de
tarifa (ver `App-passenger/errores-conocidos.md`, aceptado sin
corregir).
