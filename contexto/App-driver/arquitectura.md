# arquitectura

Repositorio:
tukituki-driver-app

Branch analizada:
main

Commit analizado:
2857ff227086442f1589a24f31626dc1b801d426

Última actualización:
2026-08-18

Fuente de verdad:
Este documento es contexto auxiliar. Si contradice al código actual,
el código y los tests tienen prioridad.

---

## Stack tecnológico

- **Flutter/Dart**, app móvil nativa. El único directorio de plataforma presente en el repo es `android/`: no existen `ios/`, `web/`, `linux/`, `macos/` ni `windows/` (verificado, ninguno está en el filesystem ni trackeado por git). `build/` sí contiene artefactos de una build local de Windows (`build/app`, `.cxx`), pero son generados y no forman parte del proyecto versionado.
- Dart SDK `^3.12.2` (`pubspec.yaml:8`). SDK instalado localmente resuelto: Flutter 3.44.6 / Dart 3.12.2 (channel stable), verificado con `flutter --version`.
- Gestión de estado/DI: `flutter_riverpod: 2.6.1` (versión clavada, no rango) — usado **solo como inyección de dependencias** (`Provider<T>` para `Dio`, `FlutterSecureStorage`, repositorios). No hay `StateNotifier`, `Notifier` ni `AsyncNotifier` en `lib/` (verificado por búsqueda). El estado de pantalla vive en `ConsumerStatefulWidget`/`StatefulWidget` con `setState` manual.
- Navegación: `go_router: ^17.4.0`, rutas declaradas en un único archivo (`lib/core/router/app_router.dart`).
- HTTP: `dio: ^5.11.0`, cliente único (`dioProvider`) con interceptores propios.
- Almacenamiento seguro: `flutter_secure_storage: 10.3.1` (versión clavada) para tokens de sesión.
- Geolocalización: `geolocator: ^14.0.3`.
- Mapas: `google_maps_flutter: ^2.18.0`.
- Selección de archivos (onboarding de Driver, ver más abajo): `image_picker: ^1.2.3` (cámara/galería), `file_picker: ^12.0.0` (PDF).
- Locale forzado a español de Perú para widgets oficiales de Material (p.ej. `showDatePicker`): `flutter_localizations` (SDK) + `locale: Locale('es', 'PE')` en `app.dart`. No es internacionalización de strings de la app (sin `.arb`, ver más abajo) — solo cambia el idioma de componentes nativos de Flutter/Material.
- Lint: `flutter_lints: ^6.0.0` vía `analysis_options.yaml` (`include: package:flutter_lints/flutter.yaml`, sin reglas propias añadidas).
- Sin paquete de mocking (no `mockito`/`mocktail` en `pubspec.yaml`); los tests de repositorios usan un `HttpClientAdapter` fijo (fake handmade) en vez de mocks. Excepción puntual: `flutter_secure_storage_platform_interface` como `dev_dependency` para simular `flutter_secure_storage` en memoria (`TestFlutterSecureStoragePlatform`, el propio helper de test que expone ese paquete) — necesario para testear el marcador de contexto de reenvío del onboarding sin un canal de plataforma real.

## Estructura y responsabilidad de carpetas

```
lib/
  app.dart                 MaterialApp.router raíz, tema global (Material 3, seed ámbar), locale es_PE
  main.dart                bootstrap: WidgetsFlutterBinding + ProviderScope
  core/
    config/app_config.dart     lee API_BASE_URL desde --dart-define
    network/api_client.dart    dioProvider + interceptores (logging, auth/refresh)
    router/app_router.dart     rutas go_router (ride/pago + onboarding)
    router/driver_onboarding_routes.dart  paths + routing de sesión del onboarding (ver módulo abajo)
    storage/secure_storage.dart  claves de FlutterSecureStorage
    theme/driver_palette.dart  paleta de colores compartida
  features/
    auth/
      data/auth_repository.dart          login/logout/isDriver/hasSession + resolveSessionState() (routing del onboarding) + marcador de contexto de reenvío
      domain/authenticated_user.dart, driver_session_state.dart  usuario autenticado + DriverSessionKind/resolveDriverApplicationState (función pura de routing)
      presentation/create_account_screen.dart, driver_splash_screen.dart, login_screen.dart
    driver/
      data/   9 repositorios: 4 de ride/pago (offers, operations, payments, rides) + 5 de onboarding (document, photo_uploader, profile, storage, vehicle) — 1 por área funcional del backend
      domain/ modelos inmutables con fromJson defensivo, sin lógica de red — incluye los del onboarding (driver_application, driver_document, driver_vehicle)
      presentation/ pantallas de ride/pago (algunas muy grandes, ver convenciones.md) + presentation/onboarding/ (10 pantallas: Sobre ti, Tu mototaxi, Tus documentos, Revisar y enviar, Correcciones requeridas, Solicitud en revisión, Suspendida, Error de estado, progreso, scaffold compartido)
test/                        espejo de lib/features/**, más 3 tests sueltos en la raíz de test/
android/                     único target de plataforma con configuración de producto real
build/, .dart_tool/, .gradle/  artefactos generados, no fuente
```

No existe carpeta `lib/features/driver/state/` ni un patrón de "controllers" separados: la orquestación vive directamente en las clases `State` de cada pantalla — incluido el onboarding, que no introdujo ninguna capa nueva de arquitectura, solo más repositorios/modelos/pantallas siguiendo el mismo patrón.

## Módulos principales

- **auth**: login por teléfono E.164 + password, verificación de rol `DRIVER` vía `GET auth/me`, ciclo de refresh de tokens.
- **driver/data**: 4 repositorios, cada uno mapea 1:1 a un grupo de endpoints bajo `drivers/me/*`:
  - `DriverOffersRepository` — ofertas de viaje (`ride-offers`) y contraofertas.
  - `DriverOperationsRepository` — estado operativo (online/offline/heartbeat), ubicación GPS, estadísticas diarias.
  - `DriverRidesRepository` — ciclo de vida del viaje activo (arrive, start, complete, cancel, waiting/no-show).
  - `DriverPaymentsRepository` — consulta y confirmación de pago en efectivo.
- **driver/presentation**: pantallas de Home, viaje activo, cobro en efectivo, pago completado, más helpers de UI (`driver_cancel_ride_flow.dart`, `driver_counter_offer_dialog.dart`, `driver_post_ride_presence.dart`, `driver_home_map.dart`).

## Módulo: Driver Onboarding

Flujo completo de alta de un conductor nuevo hasta quedar habilitado para operar, más el ciclo de corrección tras un rechazo del admin. No es un módulo aparte a nivel de carpetas — vive repartido en `features/auth` (sesión/routing) y `features/driver` (datos/pantallas del onboarding en sí), siguiendo la misma estructura `data`/`domain`/`presentation` del resto del repo.

**Flujo normal** (una sola pasada, sin rechazo):

```
Tu cuenta (registro + login)
  → Sobre ti           (crea DriverProfile DRAFT + foto)
  → Tu mototaxi         (crea DriverVehicle DRAFT)
  → Tus documentos       (licencia + SOAT + tarjeta de propiedad)
  → Revisar y enviar     (resumen + POST drivers/me/submit)
  → Solicitud en revisión (PENDING_REVIEW, solo lectura, esperando al admin)
```

**Flujo de corrección** (cuando el admin rechaza algo):

```
Admin rechaza (perfil y/o vehículo y/o algún documento)
  → Correcciones requeridas  (5 secciones siempre visibles: Sobre ti,
                               Tu mototaxi, Licencia, SOAT, TIV — solo
                               las observadas muestran el motivo exacto
                               del admin + botón "Corregir")
  → corregir cada observación (reutiliza las pantallas de Sobre ti/Tu
                               mototaxi/Tus documentos en modo edición,
                               el documento se abre enfocado en el tipo
                               observado)
  → Revisar y reenviar        (misma pantalla del flujo normal, en modo
                               "resubmission": sin botones Editar
                               generales, CTA "Reenviar solicitud")
  → Solicitud en revisión     (mismo POST drivers/me/submit reutilizado)
```

**Session routing** (`resolveSessionState()` en `AuthRepository` + la función pura `resolveDriverApplicationState()` en `driver_session_state.dart`): en cada arranque/tras cada acción, se consulta `GET auth/me` + `GET drivers/me` (+ `GET drivers/me/vehicle`/`GET drivers/me/documents` cuando el estado lo requiere) y se deriva un único `DriverSessionKind` — nunca hay una bandera local de "en qué paso quedé", todo se deriva de datos reales de Backend en cada resolución. Valores: `noProfile`, `draftNoVehicle`, `draftDocumentsIncomplete`, `draftDocumentsComplete`, `correctionsRequired`, `pendingReview`, `approved`, `approvedRoleMismatch`, `suspended`, `unknownApplicationStatus`. El mapeo a rutas concretas vive en `driver_onboarding_routes.dart` (`routeForDriverSessionKind`/`goToDriverSessionRoute`), separado de la función pura de resolución para poder testear el cálculo del estado sin `BuildContext`/`go_router`.

`correctionsRequired` **nunca** se deriva de `application.status == REJECTED` en solitario (Backend marca ese status en cualquier rechazo, incluso si solo observó el vehículo o un documento) — usa la función pura `hasPendingDriverCorrections()`: `application.rejectionReason` no vacío, `vehicle.status == REJECTED`, o algún documento de los 3 objetivo con `status == REJECTED`.

**Repositorios de datos** (`features/driver/data/`): `DriverProfileRepository` (perfil), `DriverVehicleRepository` (vehículo), `DriverDocumentRepository` (metadata de documentos), `DriverStorageRepository` (presign/complete de subidas), `DriverPhotoUploader` (el `PUT` aislado al bucket, ver "Seguridad de subida de archivos" abajo).

**Contexto de reenvío (resubmission marker)**: Backend no conserva historial de rechazos — un `DriverProfile` corregido vuelve a `DRAFT` sin ningún rastro de que viniera de un rechazo. Para no perder ese contexto entre reinicios de la app o cierres de sesión de la misma cuenta, `AuthRepository` guarda un marcador local no sensible (booleano, nunca la razón de rechazo ni ningún dato del expediente) en `flutter_secure_storage`, **scoped por cuenta** (`AuthenticatedUser.id`, nunca un booleano global ni un dato de identidad como DNI/email/teléfono). Se marca al detectar `REJECTED`, se limpia al confirmarse un reenvío exitoso o cualquier estado terminal (`PENDING_REVIEW`/`APPROVED`/`SUSPENDED`), y deliberadamente **no** se borra en el logout (cerrar sesión termina la sesión, no el ciclo administrativo de la solicitud).

**Seguridad de subida de archivos**: tanto la foto de perfil (Paso 2) como los documentos (Paso 4) siguen el mismo patrón presign→PUT externo→complete: `DriverStorageRepository` (autenticado, vía `dioProvider`) obtiene una URL presignada de Railway Storage y confirma la subida; el `PUT` real a esa URL usa un `Dio()` efímero y aislado (`DriverPhotoUploader`, sin `AuthInterceptor`) que solo envía el header `Content-Type` — nunca el `Authorization: Bearer` de sesión de TukiTuki, que filtraría la credencial a un dominio de bucket externo.

## Entidades principales (domain/)

De ride/pago: `DriverActiveRide`, `AssignedPassenger`, `DriverRideOffer`, `DriverPendingProposal`, `DriverOperationalState` (+ enum `DriverOperationalStatus`), `DriverDailyStats`, `DriverPendingPayment`, `DriverRideCompletion`, `DriverRidePayment`, `DriverRideWaiting`, `DriverCancellationReason` (enum).

De onboarding: `AuthenticatedUser` (usuario autenticado, `GET auth/me`), `DriverSessionState`/`DriverSessionKind` (resultado del routing, no un recurso de Backend), `DriverApplication` (perfil del conductor, `DriverApplicationStatus`), `DriverVehicle` (`VehicleStatus`/`VehicleOwnership`), `DriverDocument` (`DriverDocumentType`/`DriverDocumentStatus`).

Ver `glosario.md` para el detalle de cada una.

Patrón común: todas son clases inmutables (`const` constructor) con `factory .fromJson`, parsing defensivo (`?? valor por defecto`, `tryParse`) y **sin** lógica de red ni Riverpod dentro del modelo.

## Flujo de datos

1. UI (`ConsumerState`) llama a un repositorio vía `ref.read(xxxRepositoryProvider)`.
2. El repositorio llama a `Dio` (single instance, `dioProvider`) contra `AppConfig.normalizedApiBaseUrl` + un path relativo (p.ej. `drivers/me/rides/active`).
3. `AuthInterceptor` (`lib/core/network/api_client.dart`) añade `Authorization: Bearer <accessToken>` salvo en rutas de `auth/login|refresh|register|otp/*`; en un 401 intenta un único refresh (con coalescing vía `_refreshFuture` para no disparar refresh concurrentes) y reintenta la request original una vez.
4. El repositorio parsea el JSON con el `factory fromJson` del modelo correspondiente y devuelve un objeto de dominio (nunca `Map` crudo) a la UI.
5. **No hay WebSockets ni push notifications**: toda actualización de estado (ofertas activas, estado del viaje, espera de no-show) se obtiene por **polling** desde la pantalla, con `Timer.periodic` en distintos intervalos observados: ofertas cada 3s, heartbeat de presencia cada 10s (Home) / cada 30s (post-venta, `driver_post_ride_presence.dart`), estado del viaje activo cada 3s, conteo de espera (waiting) cada 1s con re-sync al backend cada 3s.
6. Backend es tratado explícitamente como única fuente de verdad temporal y de negocio (comentarios recurrentes: "Backend es la única autoridad de tiempo", "Backend es la única fuente autoritativa") — el cliente nunca deriva localmente montos, tiempos de espera o disponibilidad.

## Servicios externos

- **Backend TukiTuki** (HTTP/REST, JSON) vía la URL inyectada en `API_BASE_URL`. No hay contrato/OpenAPI ni SDK generado en este repo: los paths y payloads están hardcodeados en cada repositorio.
- **Google Maps Platform** (`google_maps_flutter`), con API key inyectada en Android vía Secrets Gradle Plugin (`android/app/build.gradle.kts`): clave real en `android/secrets.properties` (gitignored, no versionado, confirmado no trackeado por git), placeholder versionado en `android/local.defaults.properties`.
- **Geolocalización del dispositivo** vía `geolocator` (GPS nativo), no un servicio externo de terceros aparte del propio SO.
- No hay integración de pasarela de pago electrónico en este repo: el único método de pago implementado del lado Driver es **efectivo** (`DriverPaymentsRepository.confirmCashPayment`); YAPE/PLIN/CARD se mencionan solo como valores posibles de `payment.method` que este repositorio filtra pero no procesa (`driver_pending_payment.dart`).

## Persistencia

- **No hay base de datos local** (sin `sqflite`, `hive`, `isar`, etc. en `pubspec.yaml`).
- Persistencia local limitada a `flutter_secure_storage`: `access_token`, `refresh_token`, `session_id` (`lib/core/storage/secure_storage.dart`) + `driver_resubmission_context:<userId>` (marcador no sensible del onboarding, scoped por cuenta, ver módulo Driver Onboarding arriba).
- Todo lo demás (viaje activo, ofertas, estadísticas, estado del onboarding) se reconsulta a Backend en cada arranque/ciclo de polling — el patrón "RESTORE" visible en logs de test (`DRIVER RESTORE - verificando viaje activo...`) confirma que el estado se reconstruye desde el servidor, no desde caché local. El onboarding sigue el mismo principio: no existe un `onboardingStep` local ni remoto, el paso actual se deriva siempre de `resolveSessionState()`.

## Autenticación / autorización

- Login por `phoneE164` + `password` contra `POST auth/login`; la respuesta debe incluir `accessToken`, `refreshToken` y `sessionId` (los 3, si falta uno se trata como respuesta inválida — `auth_repository.dart:48-57`).
- Registro de cuenta nueva (Paso 1, "Tu cuenta"): `POST auth/register/passenger`, reutilizado internamente para Driver App (no existe `POST auth/register/driver` — decisión de producto). Esa respuesta nunca trae sesión/tokens; el registro se completa con un `login` real por separado.
- Verificación de rol de conductor: `GET auth/me`, exige que `roles` contenga el string `'DRIVER'` para entrar a Home — mientras el conductor está en onboarding (`DriverProfile` sin `APPROVED`), el routing lo mantiene dentro del flujo de onboarding en vez de bloquear el acceso a la app.
- **OTP diferido (decisión de producto MVP)**: `isPhoneVerified == false` **no** bloquea ningún paso del onboarding — el interceptor ya excluye `auth/otp/*` de la cabecera Authorization (preparado), pero ninguna pantalla lo invoca todavía; Backend sigue exigiendo `phoneE164` único.
- Refresh automático transparente ante 401 vía `AuthInterceptor`, con una sola reintentona por request (`_authRetryKey` en `extra`) para evitar loops.
- Cierre de sesión: best-effort en backend (`POST auth/logout`, ignora `DioException`) + limpieza local garantizada en `finally` — borra `access_token`/`refresh_token`/`session_id`, deliberadamente **no** el marcador de contexto de reenvío del onboarding (ver arriba).
- No hay biometría ni control de permisos granular más allá de "es o no DRIVER".

## Deploy

- **No existe pipeline de CI/CD en este repositorio** (no hay carpeta `.github/workflows`, ni `.gitlab-ci.yml`, ni `Dockerfile`, ni `fastlane`, verificado con búsqueda de archivos). El único target de build configurado con datos reales de producto es Android (`android/app/build.gradle.kts`, `applicationId = "pe.tukituki.driver"`).
- El build de release Android firma con las claves de **debug** por defecto (`signingConfig = signingConfigs.getByName("debug")`, con un comentario `// TODO: Add your own signing config` — build.gradle.kts:41-43): no hay signing de producción configurado en el repo.
- `API_BASE_URL` se inyecta obligatoriamente por `--dart-define` en build/run (`AppConfig.normalizedApiBaseUrl` lanza `StateError` si está vacío); no hay archivo de entorno (`.env`) ni variantes `dev/staging/prod` declaradas en el código.
- `MAPS_API_KEY` se inyecta vía Secrets Gradle Plugin, con fallback a un placeholder para builds sin la clave real (CI/otros desarrolladores).

## Cosas importantes que actualmente NO existen

(Verificado por búsqueda de archivos/dependencias, no supuesto.)

- No hay pipeline de CI/CD (sin `.github/`, sin `Dockerfile`).
- No hay tests de integración/E2E (`integration_test/`); solo `flutter_test` (unit + widget tests).
- No hay WebSockets ni ningún mecanismo de push/real-time: todo es polling HTTP.
- No hay gestor de estado global tipo `StateNotifier`/BLoC: Riverpod solo se usa como DI.
- No hay persistencia local estructurada (DB/cache) más allá de 3 claves en secure storage.
- No hay internacionalización de strings de la app (`intl`/`.arb`): todos los textos de la UI están hardcodeados en español. `flutter_localizations` sí está presente, pero solo fuerza el locale `es_PE` de widgets oficiales de Material (`showDatePicker`, etc.) — no traduce nada propio del app ni existe soporte para otro idioma.
- No hay manejo de crash reporting/analytics (sin Firebase, Sentry, Crashlytics en `pubspec.yaml`).
- No hay soporte iOS versionado en el repo (no existe carpeta `ios/`); tampoco `web/`, `linux/`, `macos/`, `windows/`.
- README.md es el boilerplate por defecto de `flutter create`, sin documentación real del proyecto.
