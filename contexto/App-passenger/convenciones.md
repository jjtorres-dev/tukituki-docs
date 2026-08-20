# convenciones

Repositorio:
tukituki-passenger-app

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

## PATRONES OBSERVADOS

### Naming

- Archivos: `snake_case.dart` (`fare_repository.dart`,
  `passenger_ride_offer.dart`).
- Clases de dominio: `PascalCase` sustantivo (`PassengerRide`,
  `AssignedDriverVehicle`).
- Repositorios: sufijo `Repository` (`RideRepository`,
  `AuthRepository`, `PassengerProfileRepository`).
- Providers Riverpod: sufijo `Provider`, `camelCase`
  (`rideRepositoryProvider`, `dioProvider`, `healthReadyProvider`).
- Constantes de storage/config: clases privadas con constructor
  `._()` y solo miembros `static const` (`StorageKeys`, `AppConfig`).
- Colores de UI: `static const Color _nombreColor` privados por
  pantalla (`_darkGreen`, `_ctaYellow`, `_cream`...) — **repetidos
  literalmente** en cada `presentation/*.dart` (splash, login,
  register, home, ride_searching, ride_receipt), no centralizados en
  un `ThemeData`/tokens compartido.

### Organización de archivos

- Patrón por feature: `features/<nombre>/{data,domain,presentation}/`.
  Se cumple en `auth`, `fare`, `passenger`, `places`, `ride`.
- Excepciones documentadas: `health/` no separa en subcarpetas
  (`health_repository.dart` y `health_screen.dart` al mismo nivel);
  `home/` solo tiene `home_screen.dart` sin `data/`/`domain/` propios
  (reutiliza repos de `fare`, `places`, `ride`).
- Un archivo de dominio = una entidad (o una entidad + sus tipos
  anidados pequeños, ej. `ride_receipt.dart` agrupa `RideReceipt`,
  `RideReceiptPayment`, `RideReceiptFare`).

### Convenciones de DTOs / modelos de dominio

- Todos los modelos que vienen del backend son clases inmutables
  (`const` constructor, todos los campos `final`) con factory
  `fromJson(Map<String, dynamic> json)`. No usan `json_serializable`
  ni `freezed` (no están en `pubspec.yaml`) — el parseo es manual.
- Parseo **defensivo por convención**: casi todos los campos usan
  `json['x']?.toString() ?? 'valorPorDefecto'` en vez de asumir el tipo
  o lanzar si falta. Ejemplos: `PassengerRide.fromJson`,
  `PassengerRideOffer.fromJson`, `RideReceipt.fromJson`.
- Excepción al patrón defensivo: `PublicUser.fromJson` y
  `PassengerRideStartCode.fromJson` usan `as String`/`as bool` directos
  (fallan con excepción si el campo falta o tiene otro tipo) — no
  siguen el mismo nivel de tolerancia que el resto de `ride/domain`.
- Coordenadas siempre se validan por rango (`-90..90`, `-180..180`) y
  finitud antes de aceptarse (`PassengerRide.fromJson`,
  `DriverLocation.tryParse`) — patrón repetido, no una sola función
  compartida.
- Getters derivados en el propio modelo de dominio en vez de lógica en
  la UI: `PassengerRideOffer.hasDifferentProposedFare`,
  `PassengerRideOffer.driverDisplayName`, `AssignedDriver.hasRating`.
- Comparación/normalización de montos monetarios (strings tipo
  `"7.00"`) centralizada en `lib/features/ride/domain/fare_amount.dart`
  (`fareAmountInCents`, `fareAmountsDiffer`,
  `normalizePassengerOfferFare`) — nunca se comparan como `double`
  directamente por riesgo de error de precisión; se normaliza a
  centavos enteros primero.

### Services / repositorios

- Un repositorio por feature, recibe `Dio` por constructor (inyección
  manual, no vía `get_it`/similar), expuesto por un único `Provider`
  Riverpod.
- Todos los métodos de red son `Future<T>` async/await directos —no hay
  capa de "use case" intermedia entre repositorio y widget.
- Manejo de 404 como "recurso no existe todavía" (no como error):
  `RideRepository.getActiveRide()`, `PassengerProfileRepository.getMyProfile()`
  capturan `DioException` con `statusCode == 404` y devuelven `null`
  en vez de propagar la excepción.
- Respuesta vacía (`response.data == null`) se trata como error de
  programación del backend y lanza `Exception('El backend devolvió una
  respuesta vacía.')` — mensaje repetido casi literal en varios
  repositorios.

### Presentation / widgets

- Todas las pantallas con estado son `ConsumerStatefulWidget` +
  `ConsumerState`, nunca `HookConsumerWidget` (no se usa
  `flutter_hooks`).
- Polling de estado con `Timer.periodic` guardado en un campo `Timer?`
  y cancelado en `dispose()`; se usa un contador de "generación"
  (`_stateGeneration` en `ride_searching_screen.dart`) para descartar
  respuestas tardías de una request obsoleta tras cambios de estado
  concurrentes.
- Errores de red mapeados a mensajes en español por `statusCode`
  (`if (error.response?.statusCode == 401) ...`) repetidos por pantalla
  — no hay un mapper de errores centralizado.
- Comentarios en español explicando el *por qué* de una decisión no
  obvia (ver ejemplos en `passenger_ride.dart`, `home_screen.dart`,
  `ride_searching_screen.dart`) — no hay comentarios que solo describan
  el *qué* del código.
- Algunos comentarios referencian identificadores de requisito interno
  con el patrón `G<n><letra?>-R<n>(.<n>)?(-<n>)?` (p. ej. `G4B-R5.1`,
  `G4B-R3-10`, checkpoint `G1P`) — aparecen tanto en comentarios de
  producción (`home_screen.dart`) como en nombres de tests. No hay en
  este repo un documento que defina qué es "G4B" o el checkpoint "G1P".

### Tests

- `flutter_test` puro (widget tests + unit tests), sin `mocktail`,
  `mockito` ni paquetes de testing adicionales en
  `pubspec.yaml` — los dobles de `Dio` se construyen a mano con
  `InterceptorsWrapper` (`ride_repository_test.dart`).
- Nombres de test largos, descriptivos, en español, a veces con el
  identificador de requisito como prefijo:
  `'G4B-R5.1-1: destino con cotización ya vencida al llegar: se
  renueva sola sin ningún botón manual'`.
- Tests de responsividad explícitos por tamaño de pantalla
  (`360x640`, `390x844`, `412x915`) verificando ausencia de overflow
  (`ride_searching_screen_test.dart`, `ride_receipt_screen_test.dart`).
- Un test por archivo de código relevante, misma estructura de carpeta
  que `lib/` (`test/features/<feature>/<capa>/<archivo>_test.dart`).
- 155 tests, todos en verde en este commit (`flutter test`, ver
  `errores-conocidos.md`).

### Migrations

[PENDIENTE: no aplica — este repo no tiene base de datos ni carpeta de
migraciones; toda la persistencia de dominio vive en el backend, que
no forma parte de este repositorio]

### Error handling

- Patrón consistente: `try { ... } on DioException catch (error) { ...
  mapear por statusCode ... } catch (_) { ... mensaje genérico ... }`.
- Errores no bloqueantes (best-effort) explícitamente documentados con
  comentario, ejemplo `AuthRepository.logout()`: si el logout remoto
  falla, la sesión local se limpia igual (`finally`).
- `debugPrint` se usa como logging de diagnóstico en rutas de error
  (no hay logger estructurado ni paquete de logging en
  `pubspec.yaml`).

### Validación

- Validación de formularios con `TextFormField.validator` +
  `_formKey.currentState!.validate()` (Form API estándar de Flutter),
  no hay librería de formularios externa.
- Reglas concretas observadas: teléfono `^[0-9]{9}$` (9 dígitos,
  prefijo `+51` fijo agregado por la app), OTP `^[0-9]{6}$`, password
  mínimo 8 / máximo 64 caracteres y debe incluir minúscula, mayúscula y
  número (`register_screen.dart:396-410`).
- Montos de oferta del pasajero validados y normalizados por
  `normalizePassengerOfferFare` (máx. 2 decimales, > 0, ≤ 999999.99).

### Configuración

- Una sola variable de entorno de build: `API_BASE_URL` vía
  `--dart-define`, leída en `AppConfig` (`lib/core/config/app_config.dart`);
  falla rápido (`StateError`) si no se define.
- La API key de Google Maps se inyecta solo a nivel Android
  (`android/local.properties` → Gradle → manifest placeholder), nunca
  en Dart/`--dart-define`.

## Patrones Git observables

- Un solo branch remoto (`main`), sin ramas de feature visibles en el
  repo local ni tags.
- 24 commits en total, historia lineal (sin merges visibles en
  `git log --oneline`).
- Mensajes de commit en inglés, formato Conventional Commits:
  `feat: ...`, `fix: ...`, `chore: ...` (excepción: el primer commit es
  literal `"Primer commit"`, en español).
- Los commits `feat:` recientes suelen traer implementación +
  su propio test file en el mismo commit (ver `git show --stat` de
  `a5d2411` y `8890396`), consistente con el volumen de tests (155)
  encontrado.
- No hay convención de *scope* en los commits (`feat(ride): ...`), solo
  `tipo: descripción` en imperativo/gerundio corto.

## [PENDIENTE: no se pudo determinar...]

- No hay `CONTRIBUTING.md`, plantillas de PR/issue, ni configuración de
  code owners en el repo — no es posible confirmar el proceso humano de
  revisión de código desde el repositorio.
- No hay definición documentada en el repo de qué significan los
  identificadores `G4B-R...` / checkpoints (`G1P`) usados en comentarios
  y nombres de test — probablemente referencian un backlog o
  especificación externa no incluida aquí.
- No hay linter de commits (commitlint/husky) configurado en el repo;
  el formato Conventional Commits observado parece disciplina manual,
  no forzada por herramienta.
