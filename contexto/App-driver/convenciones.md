# convenciones

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

## PATRONES OBSERVADOS

### Naming

- Prefijo `Driver` en (casi) todos los tipos públicos de `features/driver/` (`DriverActiveRide`, `DriverRideOffer`, `DriverOperationsRepository`, `DriverHomeScreen`...), para distinguirlos de tipos equivalentes que existirían en la app de pasajero (mencionada explícitamente en comentarios: "mismo patrón ya usado en Passenger").
- Excepción documentada: `AssignedPassenger` (en `driver_assigned_passenger.dart`) no lleva el prefijo `Driver` — es un sub-recurso, no una entidad de primer nivel.
- Archivos y providers en `snake_case`/`camelCase` estándar de Dart: `driver_ride_offer.dart` ↔ `DriverRideOffer`, provider `driverOffersRepositoryProvider`.
- Enums con valor "wire" explícito: `DriverCancellationReason` y `DriverOperationalStatus` separan el valor Dart (`passengerRequestedCancel`) del string que espera/devuelve Backend (`'PASSENGER_REQUESTED_CANCEL'`), nunca se manda el `.name` de Dart directamente al backend.
- Widgets privados de una pantalla grande usan prefijo `_` y vive en el mismo archivo que la pantalla (p.ej. `_FareCard`, `_PassengerCard`, `_WaitingSection` dentro de `driver_active_ride_screen.dart`), en vez de un archivo por widget.
- Pantallas de onboarding con prefijo `DriverOnboarding` (`DriverOnboardingAboutYouScreen`, `DriverOnboardingCorrectionsScreen`...), archivos en `features/driver/presentation/onboarding/`. Cuidado documentado: `DriverOnboardingReviewScreen` (pantalla de "Solicitud en revisión", `PENDING_REVIEW`) y `DriverOnboardingSubmitReviewScreen` (Paso 5 "Revisar y enviar") casi comparten nombre a propósito de evitar — son conceptos distintos, el segundo se nombró deliberadamente distinto tras una colisión real detectada durante el desarrollo de `R3.7`.

### Organización de archivos

- Estructura por *feature* (`features/auth`, `features/driver`), cada una dividida en `data/` (repositorios), `domain/` (modelos) y `presentation/` (pantallas/widgets). No hay capa `domain` con "use cases" o "interactors" separados: los repositorios se llaman directamente desde la presentación.
- `core/` agrupa infraestructura transversal (config, network, router, storage, theme) sin sub-dividir por feature.
- Tests son un espejo exacto de `lib/`: `test/features/driver/domain/driver_money_test.dart` ↔ `lib/features/driver/domain/driver_money.dart`. Hay 3 tests en la raíz de `test/` (`driver_counter_offer_dialog_test.dart`, `driver_offers_repository_test.dart`, `driver_ride_offer_test.dart`) que no siguen el espejo — cubren archivos que sí están en subcarpetas de `lib/`, así que la ubicación en `test/` no es 100% consistente con `test/features/**`.

### Convenciones de "DTOs" (modelos domain/)

- Todas las clases de `domain/` son inmutables (`const` constructor, campos `final`) con un `factory X.fromJson(Map<String, dynamic> json)`.
- Parsing defensivo generalizado: cada campo usa `?.toString() ?? 'default'` o `tryParse`, nunca un cast directo que pueda lanzar en producción por un campo faltante — con una excepción deliberada: `DriverRideWaiting.fromJson` sí lanza `FormatException` si faltan campos temporales críticos (comentario explícito: "se rechaza... en vez de fabricar un estado válido a partir de datos ausentes").
- Montos (`fare`, `amountDue`, `grossAmount`, etc.) se modelan como `String`, nunca `double`, para no introducir error de redondeo — ver comentario en `driver_daily_stats.dart` y helpers dedicados en cascada de centavos en `driver_money.dart` (`parseAmountToCents`/`formatCentsAsDecimal`) para los pocos flujos que sí necesitan aritmética (cambio en efectivo).
- Getters derivados calculados en el propio modelo cuando es un dato puro de presentación (`displayFare`, `hasRating`, `hasValidRouteCoordinates`, `hasProposal`, `isCounterOffer`), pero nunca campos calculados que dependan del reloj o de reglas de negocio temporal (esos se dejan a Backend, ver `DriverRideWaiting`).
- Coordenadas (`lat`/`lng`) siempre nullable con validación de rango explícita al parsear (`_tryParseCoordinate` con `minimum`/`maximum`); un valor fuera de rango se descarta en vez de dibujar un marker inválido.

### Repositorios / "services"

- Un repositorio por área funcional de Backend, no uno por entidad (`DriverRidesRepository` cubre `DriverActiveRide`, `DriverRideCompletion`, `DriverPendingPayment`, `DriverRideWaiting` a la vez).
- Cada repositorio se expone como un único `Provider<T>` de Riverpod que envuelve `dioProvider`; no hay abstracción de interfaz/implementación (no `abstract class XRepository` + `XRepositoryImpl`).
- 404 se traduce explícitamente a `null` en los métodos "get single resource that may not exist" (`getActiveRide`, `getRideWaiting`); cualquier otro `DioException` se relanza sin envolver (`rethrow`), nunca se traga en silencio salvo en casos best-effort documentados (`logout()`, heartbeats de presencia).
- Comentarios `///` en los repositorios documentan explícitamente el contrato asumido del backend (orden de la lista, idempotencia, qué status HTTP significa qué) — son la forma principal de "documentación de API" en el repo, ya que no hay OpenAPI/Swagger versionado aquí.

### Pantallas / estado

- No hay "controllers" ni "view models" separados: el estado y la orquestación de red viven en la clase `State` de cada `ConsumerStatefulWidget`/`StatefulWidget`.
- Riverpod (`ref.read`/`ref.watch`) se usa exclusivamente para obtener instancias de repositorios/`Dio`/`SecureStorage`; no hay providers de estado de UI (`StateProvider`, `StreamProvider`, etc.).
- Polling vía `Timer.periodic` directamente en `initState`/métodos de la pantalla, con cancelación explícita en `dispose()`.
- Puntos de inyección para testing declarados como variables globales *override* (p.ej. `driverHomeGpsFetcherOverride` en `driver_home_screen.dart`) en vez de agregar un parámetro de constructor o un provider — comentario explícito: "permite reemplazar la obtención real de GPS sin acoplar Home a Geolocator dentro de los tests ni agregar paquetes nuevos".
- Dos pantallas superan las 3000 líneas en un solo archivo (`driver_home_screen.dart` ~3472, `driver_active_ride_screen.dart` ~3311), con múltiples widgets privados `StatelessWidget` definidos al final del mismo archivo en vez de archivos separados.

### Tests

- Sin librería de mocking: los tests de repositorios usan un `HttpClientAdapter` fijo hecho a mano (`_FixedResponseAdapter`/`_ScriptedAdapter`) que devuelve el JSON exacto que el test define, en vez de mockear `Dio` con `mockito`/`mocktail`. Único auxiliar externo aceptado: `TestFlutterSecureStoragePlatform` (expuesto por el propio `flutter_secure_storage` para pruebas) para simular el storage seguro en memoria sin canal de plataforma real.
- Los tests de widgets grandes incluyen casos "Responsive" explícitos que fijan tamaños de viewport concretos (ej. `390x844`, `412x915`) y verifican ausencia de overflow — patrón repetido en varios `_test.dart` de `presentation/`.
- Nombres de test en español, descriptivos del comportamiento de negocio (p.ej. `'preserva el orden entregado por Backend (cercanía) y NO reordena por expiresAt'`), muchas veces referenciando el checkpoint/bug que motivó el test (`"bug detectado en G4A-AUDIT"`).
- Regresiones se reproducen primero con un test que falla contra el código sin corregir, y solo entonces se aplica el fix (patrón explícito desde el hotfix de `R3.7.2`, repetido en `R3.8.2`) — nunca "parchar a ciegas" un síntoma reportado físicamente sin haber confirmado la causa real en código.
- 804 tests en total ejecutan y pasan sobre el commit analizado (`flutter test`, ver `errores-conocidos.md`).

### Patrones del onboarding de Driver (`R3.3`–`R3.8`)

- **`resolveSessionState()` como única fuente de routing**: ninguna pantalla de onboarding guarda "en qué paso estoy" — tras cualquier acción exitosa (crear, editar, corregir), se vuelve a resolver la sesión completa contra Backend y se navega según el resultado (`goToDriverSessionRoute`), nunca a una ruta hardcodeada. Corrigió un bug real (`about_you`/`vehicle` navegaban con `context.go(...)` hardcodeado tras un `POST`, arrastrado sin corregir desde antes de que ese destino cambiara de significado dos veces).
- **Modo CREATE vs EDIT por presencia de `args`**: las pantallas de Sobre ti/Tu mototaxi/Tus documentos aceptan un objeto `...ScreenArgs?` opcional; su sola presencia decide `POST` (crear) vs `PATCH` (editar), sin un booleano adicional. Reutilizadas tal cual tanto desde "Revisar y enviar" (edición libre) como desde "Correcciones requeridas" (edición restringida al recurso observado).
- **`didUpdateWidget` cuando `go_router` puede reutilizar el `State`**: cuando una pantalla se reabre navegando a la misma ruta (`context.go(ruta, extra: freshState)`) en vez de hacer `pop()`, Flutter/`go_router` puede reutilizar el `State` existente — `initState()` no vuelve a ejecutarse. Toda pantalla de onboarding que puede recibir un `initialState` fresco por ese camino implementa `didUpdateWidget()` para consumirlo (lección de `R3.7.2`, reaplicada en `R3.8`). Cualquier flag local usado para bloquear doble-tap durante una navegación (`_busy`/similar) debe resetearse en el mismo punto donde se consume el estado fresco — no solo en la continuación del `await context.push(...)` original, que puede no llegar a ejecutarse a tiempo si el destino vuelve con `context.go()` (root cause de `R3.8.2`).
- **`context.go()` para transiciones terminales de sesión, `context.push()` para edición con retorno esperado**: `goToDriverSessionRoute` siempre usa `context.go` (reemplaza el stack, consistente con "esto es donde debo estar según el estado real"); las pantallas de edición se abren con `context.push` cuando el llamador espera recuperar control al volver.
- **Subida de archivos con `Dio` aislado**: cualquier `PUT` a una URL presignada de Storage usa una instancia de `Dio` efímera sin `AuthInterceptor` (`DriverPhotoUploader`), nunca el `dioProvider` compartido — evita filtrar el `Authorization: Bearer` de sesión de TukiTuki a un dominio de bucket externo. El `presign`/`complete` sí usa `dioProvider` (son llamadas autenticadas contra el propio Backend).
- **Backend como única fuente de verdad para `status`**: ningún `DriverSessionKind`/regla de negocio se deriva de un booleano local — `hasPendingDriverCorrections()` lee `rejectionReason`/`status` reales de `application`/`vehicle`/`documents` en cada resolución, nunca cachea "ya corregí esto" del lado cliente salvo el marcador de contexto de reenvío (que es metadata de sesión, no de negocio).
- **Sin PII en logs**: los `debugPrint` de captura de errores en pantallas de onboarding interpolan siempre el objeto `error` genérico (nunca un campo específico como teléfono/DNI/placa/`rejectionReason` por nombre); el interceptor de red solo loguea método+path+status, nunca headers/bodies, y solo en `kDebugMode`.

### Error handling

- Excepciones genéricas `Exception('mensaje en español')` para "respuesta vacía del backend" en casi todos los repositorios — no hay una jerarquía de excepciones de dominio propia (no `class ApiException implements Exception`).
- En la capa de presentación, los `catch` diferencian por `statusCode` de `DioException` para mostrar mensajes específicos al usuario (ver `login_screen.dart`), degradando a un mensaje genérico si no hay `response` (error de red).
- Patrón "best-effort" documentado explícitamente con comentario cuando una falla de red no debe romper el flujo (logout remoto, heartbeats de presencia): se captura, se hace `debugPrint`, y se continúa.

### Validación

- Validación de montos de dinero centralizada en funciones puras reutilizables (`validateDriverCounterOfferInput`, `driverCancelDetailValidationError`, `parseAmountToCents`), testeadas por separado de los widgets que las usan.
- Reglas de validación del cliente a veces son **más estrictas** que las de Backend, con la razón documentada en el propio código (ej. `driverCancelDetailMaxLength = 300` en el cliente vs. 500 que acepta Backend, para no depender de un truncamiento silencioso en una columna `varchar(300)` distinta).

### Configuración

- Una única fuente de configuración de entorno: `--dart-define=API_BASE_URL=...`, sin archivo `.env` ni variantes de build declaradas en Dart.
- Secretos de plataforma (Google Maps API key) fuera del código Dart, inyectados a nivel Gradle con un archivo local gitignored + placeholder versionado (ver `arquitectura.md`).

### Git / commits observables

- Todos los mensajes de commit recientes siguen Conventional Commits (`feat:`, `fix:`, `chore:`), en inglés, en imperativo/presente (`feat: add driver availability and ride offer matching`, `fix: harden driver session and use real ride gps`). El primer commit del historial es `df124b6 Primer commit` (español, sin prefijo) — único caso que no sigue el patrón.
- Historial es lineal (20 commits, sin merges visibles en `git log --oneline -20`), consistente con desarrollo directo sobre `main` o rebase antes de mergear.
- No hay convención de ramas observable en este checkout: solo existe `main` localmente (no se listaron otras branches remotas para confirmar un patrón de naming de ramas).

---

## [PENDIENTE: no se pudo determinar]

- No hay `CONTRIBUTING.md`, `CODEOWNERS` ni guía de estilo escrita: las convenciones de este documento son inferidas del código, no declaradas por el equipo.
- No hay evidencia de un formateador/linter ejecutado en CI (no hay `.github/workflows`) que haga cumplir estas convenciones automáticamente más allá de `flutter analyze` local.
- No se pudo determinar el proceso de revisión de código (PR template, checklist de review) porque no hay artefactos de GitHub/GitLab en el repo.
- No hay convención escrita sobre cuándo dividir un archivo de pantalla grande en varios; el tamaño de `driver_home_screen.dart`/`driver_active_ride_screen.dart` sugiere que, en la práctica, no se ha aplicado un límite.
