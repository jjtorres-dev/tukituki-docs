# arquitectura

Repositorio:
tukituki-passenger-app (nota: la plantilla de esta tarea decía
"tukituki-backend"; el repositorio analizado es en realidad la app
Flutter del pasajero — se documenta el repo real, no el de la plantilla)

Branch analizada:
main

Commit analizado:
a5d2411e791c391d4dca72a79e21cf6e73395335

Última actualización:
2026-08-15

Fuente de verdad:
Este documento es contexto auxiliar. Si contradice al código actual,
el código y los tests tienen prioridad.

---

## Stack tecnológico

- **Framework**: Flutter 3.44.6 (channel stable), Dart SDK `^3.12.2`
  (`pubspec.yaml`, verificado con `flutter --version`).
- **State management**: `flutter_riverpod ^2.6.1` — solo `Provider` /
  `FutureProvider` para exponer repositorios; el estado mutable de
  pantalla vive en `StatefulWidget` + `setState` (no hay
  `StateNotifier`/`AsyncNotifier`).
- **Navegación**: `go_router ^17.4.0`, ruteo declarativo centralizado en
  `lib/core/router/app_router.dart`.
- **HTTP**: `dio ^5.11.0`.
- **Almacenamiento seguro**: `flutter_secure_storage 10.3.1` (pinned,
  sin `^`).
- **Mapas**: `google_maps_flutter ^2.18.0`.
- **Geolocalización**: `geolocator ^14.0.3`.
- **Lints**: `flutter_lints ^6.0.0` vía `analysis_options.yaml`.
- Plataformas presentes: solo `android/`. No existe carpeta `ios/`.

`flutter analyze` no reporta issues sobre este commit (ver
`errores-conocidos.md`).

## Estructura y responsabilidad de carpetas

```
lib/
  app.dart              MaterialApp.router raíz, tema (Material 3, seed amber)
  main.dart             bootstrap, valida API_BASE_URL, ProviderScope
  core/
    config/app_config.dart   API_BASE_URL vía --dart-define (obligatorio)
    network/api_client.dart  Dio + AuthInterceptor (bearer + refresh)
    router/app_router.dart   rutas GoRouter
    storage/secure_storage.dart  FlutterSecureStorage + StorageKeys
  features/
    auth/       data/ (AuthRepository) domain/ (PublicUser) presentation/ (splash, login, register, otp)
    fare/       data/ (FareRepository) domain/ (FareEstimate)
    health/     health_repository.dart + health_screen.dart (sin subcarpetas)
    home/       home_screen.dart, search_destination_screen.dart, offer_fare_screen.dart (sin presentation/ propia, archivos sueltos); domain/ (SearchDestinationResult, OfferFareResult, ShortAddressLabel, SuggestedDestinations) — sin data/ propio, reutiliza fare/places/ride
    passenger/  data/ (PassengerProfileRepository) presentation/ (complete_profile_screen)
    places/     data/ (PlacesRepository) domain/ (PlacePrediction, PlaceDetails)
    ride/       data/ (RideRepository, PaymentPreferenceRepository) domain/ (PassengerRide y modelos asociados, PaymentMethod) presentation/ (ride_searching_screen, ride_receipt_screen, PaymentMethodPickerSheet)
```

La convención `data/` (repos) + `domain/` (modelos JSON) + `presentation/`
(widgets) se cumple en `auth`, `fare`, `passenger`, `places`, `ride`.
`health` y `home` son excepciones — no la siguen del todo (ver
`convenciones.md`): `home` sigue sin `data/` propio, pero desde
`HOME-FLOW-R1`/`FARE-PANEL-R1` sí tiene su propia carpeta `domain/`
(tipos de resultado de pantallas y utilidades puras) y varios archivos
de pantalla sueltos además de `home_screen.dart` — dejó de ser "un solo
archivo sin subcarpetas".

## Módulos principales

- **auth**: registro, login, recuperación/validación de sesión
  (`splash_screen.dart`), OTP (código presente pero sin flujo que lo
  invoque — ver decisión "verificación de teléfono opcional").
- **passenger**: alta de perfil de pasajero (`nombre`/`apellido`) tras
  login/registro. En `main`@`a5d2411e` este paso NO se hacía cumplir
  realmente (`SplashScreen` descartaba el resultado de
  `getMyProfile()` y siempre continuaba a `/home` — ver
  `decisiones.md`); corregido en `test/r4-passenger-identity` (`R4.2`,
  `PHYSICAL-REVIEW-PASS`) mediante un resolver puro
  (`lib/features/auth/domain/passenger_session_state.dart`) que hace
  que un `Passenger` sin `PassengerProfile` y sin ride activo sea
  enviado a `/complete-profile` antes de `/home` — con un ride activo
  siempre teniendo prioridad sobre ese gate. **Integrado a
  `main`@`91357d17` (`CROSS-APP-R4.2D`, fast-forward, 2026-08-18) y
  confirmado por `MAIN-PHYSICAL-SMOKE-PASS` (`CROSS-APP-R4.2E`,
  2026-08-18) — `FINAL-CLOSED`.**
- **fare**: cotización de tarifa (`fares/estimate`) antes de crear un
  ride.
- **places**: autocomplete y detalle de direcciones, proxied por el
  backend (`places/autocomplete`, `places/:placeId`) — el cliente no
  llama directamente a la API de Google Places.
- **ride**: ciclo de vida completo del viaje del pasajero — creación,
  polling de estado, ofertas de conductores (negociación), selección
  de oferta, cancelación, código de inicio (PIN), recibo, pago y
  calificación. **Desde `FARE-PANEL-R1` (2026-08-31)** también aloja
  la preferencia de método de pago del pasajero (`PaymentPreferenceRepository`,
  `flutter_secure_storage`, dato por dispositivo no por cuenta) y su
  selector compartido (`PaymentMethodPickerSheet`,
  `lib/features/ride/presentation/payment_method_picker_sheet.dart`,
  extraído de `home_screen.dart` porque lo reutilizan tanto Home como
  `OfferFareScreen`).
- **home**: pantalla mapa + selección de destino + negociación de
  tarifa antes de crear el ride (`lib/features/home/home_screen.dart`,
  la pantalla más grande del repo). **Desde `FARE-PANEL-R1` (2026-08-31)**
  ya no es un único archivo: `offer_fare_screen.dart` es la pantalla de
  confirmación de tarifa ("Ofrece tu tarifa"), que devuelve su
  resultado por `Navigator.pop` en vez de crear el ride directamente
  (mismo patrón que ya usaba `search_destination_screen.dart` desde
  `HOME-FLOW-R1`); `domain/short_address_label.dart` es la utilidad
  pura de acortado de direcciones (mismo patrón que
  `ride/domain/fare_amount.dart`) reutilizada por las tarjetas
  origen/destino de ambas pantallas.

## Entidades principales

`PublicUser`, `FareEstimate`, `PassengerRide`, `PassengerRideOffer`,
`AssignedDriver` (+ `AssignedDriverVehicle`), `DriverLocation`,
`PassengerRideStartCode`, `RideReceipt` (+ `RideReceiptPayment`,
`RideReceiptFare`), `PlacePrediction`, `PlaceDetails`. Detalle de
campos y estados en `glosario.md`.

## Flujo de datos

1. `main.dart` exige `API_BASE_URL` (dart-define) antes de arrancar; si
   falta, lanza `StateError` inmediatamente (falla rápido, sin fallback
   a URL hardcodeada).
2. Todas las requests pasan por el `Dio` único de
   `core/network/api_client.dart`, con `AuthInterceptor`:
   - agrega `Authorization: Bearer <accessToken>` salvo en
     `auth/login`, `auth/refresh`, `auth/register`, `auth/otp/*`;
   - ante un 401 (fuera de esas rutas) intenta un único refresh con
     `auth/refresh` (single-flight vía `_refreshFuture` compartido, evita
     refrescos concurrentes), reintenta la request original una sola vez
     y si falla limpia la sesión local.
3. Los repositorios (`*Repository`) llaman a `Dio` y devuelven modelos de
   dominio parseados con `fromJson` defensivo (casi todos los campos
   String/num usan `?.toString() ?? fallback`, no asumen tipos exactos
   del backend).
4. No hay WebSockets ni Server-Sent Events: el estado de un ride en
   curso y el estado de un pago se actualizan por **polling HTTP**
   (`Timer.periodic`):
   - `ride_searching_screen.dart:75` — cada 3s mientras el ride no está
     en estado terminal.
   - `ride_receipt_screen.dart:149` — cada 2s mientras el pago no está
     en estado terminal (`PAID`, `FAILED`, `EXPIRED`, `DISPUTED`,
     `VOIDED`).
5. Sesión persistida en `flutter_secure_storage` (`access_token`,
   `refresh_token`, `session_id`). No hay caché local de datos de
   dominio (rides, perfil, etc.) — todo se re-consulta al backend.

## Servicios externos

- **Backend HTTP propio de TukiTuki** (base URL inyectada por
  `API_BASE_URL`; un comentario en `auth_repository.dart:175` menciona
  "Railway" como posible entorno de hosting del backend, no confirmable
  desde este repo).
- **Google Maps SDK for Android**, clave inyectada vía
  `android/local.properties` → `MAPS_API_KEY` → manifest placeholder
  (`android/app/build.gradle.kts`, `AndroidManifest.xml`). El archivo
  `local.properties` está en `.gitignore` (`android/.gitignore:6`), no
  se versiona ninguna clave.
- **Google Places**: consumido indirectamente, siempre a través del
  backend propio (no hay SDK de Places ni API key de Places en el
  cliente).
- **Geolocalización del dispositivo** vía `geolocator` (permisos
  `ACCESS_FINE_LOCATION` / `ACCESS_COARSE_LOCATION` en el manifest).

## Persistencia

Únicamente `flutter_secure_storage` para credenciales de sesión
(`access_token`, `refresh_token`, `session_id`). No hay base de datos
local (SQLite/Hive/Isar), no hay caché offline, no hay `shared_preferences`
en `pubspec.yaml`.

## Autenticación / autorización

- Login por `phoneE164` + password (`auth/login`), teléfono construido
  como `'+51$phone'` con un input de 9 dígitos validado por regex — el
  prefijo `+51` (Perú) está **hardcodeado** en `login_screen.dart` y
  `register_screen.dart`, no es configurable.
- Tokens JWT-like (`accessToken`/`refreshToken`/`sessionId`) opacos
  para el cliente: no se decodifican, solo se guardan y reenvían.
- Autorización de rol: `splash_screen.dart:117` valida
  `user.roles.contains('PASSENGER')` y `user.status == 'ACTIVE'` antes
  de considerar la sesión válida; si no cumple, limpia sesión y va a
  `/login`.
- No hay biometría, no hay OAuth/social login, no hay MFA obligatorio
  (el flujo de OTP existe en código pero no está conectado — ver
  `decisiones.md`).

## Deploy

- No se encontró Dockerfile, docker-compose, ni carpeta `.github/`
  (sin CI/CD configurado en el repo).
- Solo existe target Android (`android/`); no hay `ios/`.
- `android/app/build.gradle.kts`: el build type `release` reutiliza el
  `signingConfig` de `debug` (`signingConfig = signingConfigs.getByName("debug")`)
  — no hay configuración de firma de producción en el repo.
- `applicationId` / namespace: `pe.tukituki.passenger`.
- No hay scripts de build/release (`fastlane`, `codemagic.yaml`,
  Makefile, etc.).

## Cosas importantes que actualmente NO existen

Verificado por búsqueda en `pubspec.yaml`, estructura de carpetas y
código — no solo por ausencia superficial:

- Sin CI/CD (no `.github/workflows`, no otro pipeline detectado).
- Sin Docker/deploy scripts.
- Sin plataforma iOS.
- Sin tiempo real (WebSocket/SSE): todo el estado en vivo (ride,
  pago) se actualiza por polling HTTP.
- Sin base de datos local ni caché offline.
- Sin `shared_preferences` ni almacenamiento no seguro de datos de
  usuario.
- Sin analítica ni crash reporting (no Firebase, Sentry, Crashlytics
  en `pubspec.yaml`).
- Sin internacionalización (`flutter_localizations`/`intl`/`.arb`):
  todos los textos están hardcodeados en español dentro de los widgets.
- Sin gestión de entornos múltiples (staging/prod) más allá del único
  `--dart-define=API_BASE_URL` pasado manualmente al build/run.
- Sin notificaciones push.
- Sin tests de integración (`integration_test/`); solo unit e widget
  tests bajo `test/`.
