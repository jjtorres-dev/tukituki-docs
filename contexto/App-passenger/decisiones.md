# decisiones

Repositorio:
tukituki-passenger-app

Branch analizada:
main

Última actualización:
2026-08-24

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
