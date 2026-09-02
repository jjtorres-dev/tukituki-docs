# errores-conocidos

Repositorio:
tukituki-driver-app

Branch analizada:
main

Commit analizado:
9c2a7f75da3abe375dd16a3336507145aece7455

Última actualización:
2026-09-02 (agregado el patrón "colisión de nombres reservados Android/Flutter con claves de `data` de FCM" tras `DRIVER-PUSH-R1`)

Fuente de verdad:
Este documento es contexto auxiliar. Si contradice al código actual,
el código y los tests tienen prioridad.

---

## Baseline de tests

`flutter test` → **824 tests, todos pasando**, sin failures, sobre el commit analizado. Sin cambio de conteo respecto al baseline anterior (`main`@`2857ff2`, `DRIVER-ONBOARDING-R3`) — `CROSS-APP-R4.3` (identidad compacta del Passenger) solo consume un campo ya expuesto por Backend, sin tests nuevos. Confirmado además por `MAIN-PHYSICAL-SMOKE-PASS` de JuanJo (smoke físico final, 7/7). `test/r4-ride-identities` eliminada local y remotamente tras el cierre.

## Baseline de análisis estático

`flutter analyze` → **"No issues found!"**, sin warnings ni infos pendientes, sobre el commit analizado.

## Comentarios FIXME/TODO relevantes

- `android/app/build.gradle.kts:29` — `// TODO: Specify your own unique Application ID` (el `applicationId` real, `pe.tukituki.driver`, ya está configurado debajo del comentario; el TODO parece residuo del scaffolding de `flutter create` no eliminado, no un problema activo).
- `android/app/build.gradle.kts:41` — `// TODO: Add your own signing config for the release build.` Este sí es un TODO activo: el build de `release` firma con las llaves de **debug** (`signingConfig = signingConfigs.getByName("debug")`), por lo que un `flutter build apk --release` generado desde este repo tal cual **no está firmado para producción**.
- No se encontraron marcadores `FIXME`/`HACK`/`XXX` en `lib/` ni `test/` (búsqueda sin resultados).

## Configuraciones delicadas

- `API_BASE_URL` es obligatorio vía `--dart-define`; si se omite, la app compila pero **falla en runtime** con `StateError` al primer intento de red (`lib/core/config/app_config.dart`). No hay validación en build-time ni mensaje de ayuda visible en UI más allá de la excepción.
- `android/secrets.properties` (clave real de Google Maps) es gitignored vía `android/.gitignore` — confirmado que **no** está trackeado por git en este checkout (`git ls-files` no lo lista). Un clone nuevo sin ese archivo local usa el placeholder `DEFAULT_API_KEY` de `android/local.defaults.properties`, con el cual el mapa de Google no funcionará correctamente (comportamiento esperado, no un bug, pero fácil de confundir con uno si no se conoce este mecanismo).

## Limitaciones técnicas demostrables

- **Sin soporte iOS/web/desktop en el repo**: solo existe configuración de plataforma Android (ver `arquitectura.md`). Cualquier intento de `flutter run -d ios` o similar fallaría por falta de la carpeta `ios/`.
- **Sin WebSockets/push**: todo el estado en vivo depende de polling HTTP con intervalos hardcodeados por pantalla (3s/10s/30s/1min, ver `arquitectura.md`). Esto implica un límite práctico de "frescura" de datos (hasta varios segundos de retraso) y una carga de requests proporcional al número de conductores conectados simultáneamente — no hay backoff visible ante errores repetidos de polling en el código revisado.
- **Sin persistencia local estructurada**: si el dispositivo pierde conectividad al arrancar la app, no hay caché local desde la cual mostrar datos (viaje activo, ofertas) — el flujo `RESTORE` depende enteramente de que Backend responda.
- **Dos archivos de pantalla muy grandes** (`driver_home_screen.dart` ~3472 líneas, `driver_active_ride_screen.dart` ~3311 líneas): no es un bug funcional, pero es una limitación de mantenibilidad demostrable por el tamaño del archivo (ver `convenciones.md`).
- **`DriverActiveRide.agreedFare` nullable "solo por consistencia con el contrato Backend"**: el propio comentario del modelo (`driver_active_ride.dart:31-34`) documenta que en teoría `agreedFare` "siempre debería venir presente" desde `DRIVER_ASSIGNED` en adelante, pero el tipo sigue siendo nullable con fallback a `estimatedFare` — es decir, el cliente asume que puede recibir un contrato inconsistente y se protege, sin que quede registrado si esto ha ocurrido realmente en producción.

## Incompatibilidades / desalineación de versiones

`flutter pub outdated` (ejecutado como parte de `flutter analyze` en este análisis) reportó **14 paquetes con versiones más nuevas disponibles pero incompatibles con las constraints actuales de `pubspec.yaml`**, incluyendo `flutter_riverpod` (2.6.1 usado vs. 3.4.2 disponible — salto de major), `go_router` (17.4.0 vs 17.5.0), `flutter_secure_storage` (10.3.1 vs 11.0.0 — salto de major). Esto no es un error activo (todo compila y los tests pasan), pero es una brecha de actualización demostrable, particularmente relevante para `flutter_riverpod` dado el salto de major version.

## Cobertura de tests no verificada cuantitativamente

`[PENDIENTE: no se ejecutó `flutter test --coverage`]` — no hay evidencia en el repo (sin badge, sin config de coverage) de qué porcentaje del código está cubierto; solo se verificó que los 804 tests existentes pasan.

## Limitaciones aceptadas a propósito

- **La hoja de propuestas pendientes no muestra el método de pago** (`PAYMENT-METHOD-DRIVER-R1`, 2026-08-27): `_buildProposalsSheet` en `driver_home_screen.dart` (la pantalla "Propuesta enviada — Esperando que el pasajero elija a su conductor", tras hacer una contraoferta) NO muestra el chip de método de pago, aunque Backend sí lo envía en `DriverPendingProposalResponseDto`. Causa: el modelo `DriverPendingProposal` (`driver_pending_proposal.dart`) no tiene el campo `paymentMethod` ni lo parsea; el cambio de `PAYMENT-METHOD-DRIVER-R1` solo añadió el campo a `DriverRideOffer` (tarjeta de solicitud entrante) y `DriverActiveRide` (viaje activo). Se aceptó así a propósito: en ese estado el conductor ya decidió involucrarse con el viaje, y el método pesa más *antes* de aceptar/contraofertar que después. **Cómo cerrarlo si se quisiera** (cambio pequeño y autocontenido): agregar `paymentMethod` a `DriverPendingProposal` (campo + `fromJson`), pasarlo por `_proposalFromOffer()` en `driver_home_screen.dart`, y usar `DriverPaymentMethodChip` en `_buildProposalsSheet()` (y opcionalmente en los `Chip` por-propuesta del caso `count > 1`).

## Bugs resueltos durante el desarrollo del onboarding (histórico, ya corregidos y aprobados físicamente)

Registrados aquí solo como referencia histórica — ninguno sigue activo en el commit analizado, verificado por regresión automatizada:

- **Stale "Revisar y enviar" tras editar una sección** (`R3.7.2`): al editar Sobre ti/Tu mototaxi/Tus documentos desde el resumen y volver, la pantalla seguía mostrando el valor anterior aunque Backend ya tenía el dato correcto. Causa: `DriverOnboardingSubmitReviewScreen` solo aplicaba el estado inicial dentro de `initState()`; como las pantallas de edición vuelven con `context.go(...)` a la misma ruta (no `pop()`), `go_router` podía reutilizar el `State` existente sin volver a ejecutar `initState()`. Fix: `didUpdateWidget()` agregado para consumir un `initialState` fresco. **Resuelto y reprobado físicamente.**
- **CTA "Revisar y reenviar" no se habilitaba tras la última corrección** (`R3.8.2`): tras corregir la última observación pendiente en "Correcciones requeridas" y volver, las tarjetas mostraban el estado fresco correctamente pero el botón seguía deshabilitado hasta reiniciar la app. Causa: el flag local `_busy` (anti doble-tap durante la navegación de corrección) solo se reseteaba en la continuación de un `await context.push(...)` que no llegaba a ejecutarse a tiempo cuando el destino volvía vía `context.go()` en vez de un `pop()` imperativo. Fix: `_assignFromState()` (fuente única de `initState`/`didUpdateWidget`/`_loadFresh`) ahora también resetea `_busy`. **Resuelto y reprobado físicamente.**
- **Marcador de contexto de reenvío como booleano global** (`R3.8B`, encontrado antes de la primera prueba física): el marcador local de "sigo en ciclo de reenvío" era una única key compartida por toda la instalación (no por cuenta) y el logout la borraba incondicionalmente — riesgo de compartir contexto entre dos conductores en el mismo dispositivo, y de perder el contexto si el conductor cerraba sesión antes de reenviar. Fix: marcador scoped por `AuthenticatedUser.id`, y `clearSession()` ya no lo toca. **Resuelto antes de cualquier build de prueba física, sin impacto en usuarios reales.**

## Colisión de nombres reservados de Android/Flutter con claves de `data` de FCM (patrón a vigilar)

Estado:
RESUELTO para el caso concreto (clave `route`, `DRIVER-PUSH-R1` fix, `main`@`2fdfd60`) — registrado aquí como **patrón general** a tener en cuenta en cualquier integración nativa futura, no solo como algo ya cerrado.

Qué pasó:
`DRIVER-PUSH-R1` (Etapa 2) leía `data['route']` del `RemoteMessage` para filtrar `RIDE_OFFER_CREATED`, coherente con lo que mandaba el Backend entonces. Con la app **completamente terminada**, tocar cualquier notificación push hacía que Android copiara cada entrada del objeto `data` como extra del intent de arranque de la `MainActivity`; el embedding de Flutter en Android lee el extra llamado literalmente `"route"` (`EXTRA_INITIAL_ROUTE`, `FlutterActivityLaunchConfigs`) y lo usa como `initialRoute`, antes de cualquier lógica de navegación de la app. Como `'ride-offer'` no es una ruta declarada, `go_router` arrancaba con `GoException: no routes for location`, y el `ErrorScreen` por defecto de go_router (botón "Home" → `/`, ruta inexistente) dejaba al usuario atrapado. Nunca se había detectado porque no se había probado "proceso muerto + tap de notificación real" de forma rigurosa hasta la Etapa 3 de `PASSENGER-PUSH-R1`.

Cómo se resolvió:
Backend renombró la clave `route` → `screen` en las 10 ocurrencias donde arma el `data` de una notificación, para todos los tipos de evento (`main`@`48d797bb`); la app lee `data['screen']`. Además se agregó un `errorBuilder` en el `GoRouter` (`RouteNotFoundScreen`, botón → `/splash`) como red de seguridad ante cualquier ruta inválida futura de otra causa. Ver `decisiones.md`, entrada `DRIVER-PUSH-R1` (fix).

Patrón a recordar para el futuro (aplica a cualquier evento push nuevo, y a cualquier integración nativa que reciba datos de un intent):
- **Nunca** poner una clave llamada `route` en el objeto `data` de una notificación FCM para Android. Sospechar también de otros nombres del namespace `io.flutter.*` / `FlutterActivityLaunchConfigs` (p. ej. `initial_route`, `background_isolate_run`).
- Claves ya en uso y confirmadas seguras: `screen`, `eventType`, `rideId`, `offerId`, `status`.
- La única forma fiable de validar el cold start es force-stop del proceso + notificación **real** desde la consola de Firebase Cloud Messaging. Un `adb shell am start ... --es route ...` no reproduce el bug (no pasa por el pipeline interno de Firebase, no dispara `getInitialMessage()`), aunque sí sirve para confirmar que la app al menos arranca sin crashear con esos extras.

Evidencia:
`lib/features/notifications/data/push_message_handler.dart`, `lib/core/router/app_router.dart`, `lib/core/router/route_not_found_screen.dart`. Commits `c11b48f` (`DRIVER-PUSH-R1`) y `2fdfd60` (fix). Backend: `main`@`48d797bb556a63664ee2a79adb10f50839fb1688`.
