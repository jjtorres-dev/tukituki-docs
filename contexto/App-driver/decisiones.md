# decisiones

Repositorio:
tukituki-driver-app

Branch analizada:
main

Última actualización:
2026-09-02

Fuente de verdad:
Este documento es contexto auxiliar. Si contradice al código actual,
el código y los tests tienen prioridad.

---

## Riverpod solo como inyección de dependencias, no como gestor de estado

Estado:
ACTIVA

Qué se decidió:
Usar `flutter_riverpod` únicamente para exponer instancias (`Provider<Dio>`, `Provider<FlutterSecureStorage>`, `Provider<XRepository>`) y mantener el estado de cada pantalla en `setState` sobre `ConsumerStatefulWidget`/`StatefulWidget`.

Por qué:
[PENDIENTE: no hay comentario ni commit que explique la elección frente a `StateNotifier`/`AsyncNotifier`; se infiere solo de la consistencia del patrón en todo `lib/`]

Alternativas descartadas:
[PENDIENTE: sin evidencia de que se haya evaluado `StateNotifier`/BLoC/otro gestor]

Evidencia:
Ausencia total de `StateNotifier`/`Notifier`/`AsyncNotifier` en `lib/` (grep sin resultados); todos los providers en `lib/features/*/data/*_repository.dart` y `lib/core/*` son `Provider<T>` simples.

---

## Polling HTTP en vez de WebSockets/push para todo el estado en vivo

Estado:
ACTIVA

Qué se decidió:
Todas las actualizaciones de estado en tiempo casi-real (ofertas nuevas, estado del viaje, cuenta regresiva de espera, presencia del conductor) se obtienen re-consultando HTTP con `Timer.periodic`, no con un canal push.

Por qué:
[PENDIENTE: no hay comentario que justifique la elección frente a WebSockets/SSE]

Alternativas descartadas:
[PENDIENTE: sin evidencia]

Evidencia:
`driver_home_screen.dart` (offers cada 3s, heartbeat cada 10s, reloj de conexión cada 1min), `driver_active_ride_screen.dart` (estado del ride cada 3s, actividad cada 10s, waiting poll cada 3s + tick cada 1s), `driver_post_ride_presence.dart` (heartbeat cada 30s). Ausencia de `web_socket_channel`/similar en `pubspec.yaml`.

---

## Backend como única fuente de verdad temporal y de negocio (no se derivan reglas localmente)

Estado:
ACTIVA

Qué se decidió:
El cliente nunca calcula localmente disponibilidad, tiempos de espera restantes, si se puede reportar no-show, ni estadísticas — siempre transporta lo que Backend ya calculó.

Por qué:
Documentado explícitamente en el código como principio de diseño repetido (evitar drift de reloj/red y duplicar reglas de negocio en dos lugares).

Alternativas descartadas:
[PENDIENTE: no se documenta si se consideró calcular localmente]

Evidencia:
Comentarios explícitos: `driver_ride_waiting.dart` ("Backend es la única autoridad de tiempo... nunca calcula `remainingWaitingSeconds` ni `canReportNoShow` por su cuenta"), `driver_daily_stats.dart` ("Backend es la única fuente autoritativa: no se calculan ni se derivan localmente").

---

## Montos de dinero como `String`, nunca `double`

Estado:
ACTIVA

Qué se decidió:
Todos los campos monetarios en los modelos `domain/` (`estimatedFare`, `agreedFare`, `amountDue`, `grossAmount`, etc.) se tipan como `String` que se pasa tal cual desde/hacia Backend. Donde se necesita aritmética real (cambio en efectivo), se convierte explícitamente a centavos enteros (`int`) en `driver_money.dart`, nunca a `double`.

Por qué:
Documentado explícitamente: evitar errores de redondeo de punto flotante en dinero.

Alternativas descartadas:
Uso de `double` para representar montos (descartado explícitamente en comentarios).

Evidencia:
`driver_daily_stats.dart:20-22` ("Se conserva como String... para no introducir errores de redondeo con dinero"), `driver_money.dart:1-6` ("Nunca se usa `double` para reglas monetarias... todo pasa por `int` (centavos)").

---

## Validación del cliente más estricta que la de Backend en `reasonDetail` de cancelación

Estado:
ACTIVA

Qué se decidió:
Limitar `reasonDetail` a 300 caracteres en el cliente (`driverCancelDetailMaxLength`), aunque Backend acepta hasta 500 en ese endpoint.

Por qué:
Documentado: Backend copia ese texto a una columna denormalizada `rides.cancellationReason varchar(300)`; limitar en cliente evita depender de un truncamiento silencioso del lado servidor.

Alternativas descartadas:
Dejar el límite en 500 (el máximo que el endpoint acepta) y confiar en el truncamiento de Backend — descartado explícitamente como "decisión defensiva del cliente".

Evidencia:
`driver_cancel_ride_flow.dart:20-27`.

---

## Un único cliente `Dio` compartido con interceptor de auth y coalescing de refresh

Estado:
ACTIVA

Qué se decidió:
Una sola instancia de `Dio` (`dioProvider`) para toda la app, con un `AuthInterceptor` que añade el bearer token, intercepta 401, y coalesce refresh concurrentes (`_refreshFuture`) para que múltiples requests fallando a la vez disparen un solo `POST auth/refresh`.

Por qué:
[PENDIENTE: no hay comentario explícito sobre la razón del coalescing, pero el patrón (`_refreshFuture` compartido, `finally` que lo limpia solo si sigue siendo el mismo `Future`) es deliberado y no accidental]

Alternativas descartadas:
[PENDIENTE: sin evidencia]

Evidencia:
`lib/core/network/api_client.dart:89-236` (`AuthInterceptor`, método `_refreshSession`).

---

## Configuración de entorno vía `--dart-define`, sin archivos `.env`

Estado:
ACTIVA

Qué se decidió:
`API_BASE_URL` se lee con `String.fromEnvironment` y lanza `StateError` si no se definió al compilar/ejecutar; no existe mecanismo de archivo de configuración por entorno en Dart.

Por qué:
[PENDIENTE: sin comentario explícito; es el mecanismo estándar de Flutter para esto, pero no hay evidencia textual de por qué se eligió sobre alternativas]

Alternativas descartadas:
[PENDIENTE: sin evidencia]

Evidencia:
`lib/core/config/app_config.dart`.

---

## Secreto de Google Maps fuera de git, con placeholder versionado para builds sin clave real

Estado:
ACTIVA

Qué se decidió:
La clave real de Google Maps vive en `android/secrets.properties` (gitignored vía `android/.gitignore`), y `android/local.defaults.properties` (versionado) provee un placeholder (`MAPS_API_KEY=DEFAULT_API_KEY`) para que builds sin la clave real no rompan.

Por qué:
Documentado: evitar exponer la clave real en el repositorio, mientras se mantiene un build funcional para CI/otros desarrolladores.

Alternativas descartadas:
[PENDIENTE: sin evidencia]

Evidencia:
`android/app/build.gradle.kts:8-16`, `android/local.defaults.properties`, `android/.gitignore:18`.

---

## Signing de release Android pendiente (usa las llaves de debug)

Estado:
PENDIENTE

Qué se decidió:
El `buildType release` firma con `signingConfigs.getByName("debug")`, con un comentario `// TODO: Add your own signing config for the release build.` explícito en el propio archivo generado por `flutter create`.

Por qué:
[PENDIENTE: no hay evidencia de que sea una decisión deliberada; parece configuración por defecto de scaffolding aún no reemplazada]

Alternativas descartadas:
[PENDIENTE]

Evidencia:
`android/app/build.gradle.kts:39-44`.

---

## Testing de repositorios sin librería de mocking (fake `HttpClientAdapter` manual)

Estado:
ACTIVA

Qué se decidió:
No se agregó `mockito`/`mocktail` como dependencia; los tests de repositorios construyen un `HttpClientAdapter` fijo que devuelve JSON predefinido.

Por qué:
[PENDIENTE: sin comentario explícito sobre por qué se evitó una librería de mocking]

Alternativas descartadas:
[PENDIENTE: sin evidencia]

Evidencia:
`test/driver_offers_repository_test.dart:9-11` (comentario "sin red real ni paquetes de mocking adicionales"), ausencia de `mockito`/`mocktail` en `pubspec.yaml`.

---

## Puntos de inyección para tests como variables globales *override*, no parámetros de constructor

Estado:
ACTIVA

Qué se decidió:
Para poder testear lógica que depende de `Geolocator` sin acoplar la pantalla al paquete dentro de los tests, se usan variables globales tipo `driverHomeGpsFetcherOverride` que los tests pueden reemplazar.

Por qué:
Documentado explícitamente: evitar acoplar `Home` a `Geolocator` en los tests sin agregar paquetes nuevos (de mocking/DI adicional).

Alternativas descartadas:
Inyección por constructor o un provider Riverpod dedicado para el fetcher de GPS — no implementado.

Evidencia:
`lib/features/driver/presentation/driver_home_screen.dart:60-71`.

---

## Rama `origin/carlos` — prototipo demo obsoleto, eliminada — DECISIÓN CERRADA, NO PORTAR

Estado:
RESUELTA (`DRIVER-BRANCH-CLEANUP-R1`, 2026-08-16)

Qué existía:
La rama remota `origin/carlos` (2 commits, `4d9a1d9` "crear demo de la experiencia del conductor" y `50c5e85` "agregar flujo de solicitud y carrera activa", ramificada desde el "Primer commit" del scaffold y nunca reconciliada con `main`) contenía una app demo temprana llamada "MotoSeguro" (Tarapoto): selección de rol, home/ganancias/historial/perfil con datos hardcodeados, mapa dibujado a mano (`CustomPaint`, sin Google Maps), y una solicitud/viaje activo simulados por un botón de notificación (sin negociación, sin backend). Sin auth, sin sesión, sin onboarding, sin ningún archivo compatible con la arquitectura actual (`dio`/`flutter_riverpod`/`go_router`/`geolocator`/`google_maps_flutter` ausentes de su `pubspec.yaml`).

Por qué se descartó:
`DRIVER-BRANCH-AUDIT-R1` (2026-08-16, solo lectura) comparó ambos commits función por función contra `main`: ninguna función de `origin/carlos` es igual o superior a la ya implementada en `main` (login/splash/isDriver gate, ofertas con contraoferta, viaje activo con GPS/PIN/cobro, bottom nav real, mapa real vía `google_maps_flutter`, 4 repositorios reales — todo ausente en `origin/carlos`). No contenía nada relacionado con registro/onboarding/perfil/vehículo/documentos de conductor (grep de esos términos sin resultados relevantes). Veredicto: `OBSOLETE`, cherry-pick no seguro para ninguno de los 2 commits, `R3-CLEAR` — no bloquea ni informa `DRIVER-ONBOARDING-R3`.

Decisión (JuanJo, `DRIVER-BRANCH-CLEANUP-R1`, 2026-08-16):
Eliminar `origin/carlos` del remoto. No existía rama local `carlos`. No se perdió ninguna funcionalidad aprobada ni pendiente de reconciliar.

Consecuencia:
`origin/carlos` (Driver) ya no existe en ningún checkout. Este documento y `DRIVER-BRANCH-AUDIT-R1` son la referencia histórica de qué existía y por qué se descartó, para que no se re-evalúe por accidente como "pendiente de rescatar".

Evidencia:
`git log --oneline origin/main..origin/carlos` (antes de borrar: `4d9a1d9`, `50c5e85`, sin commits adicionales); reporte completo de `DRIVER-BRANCH-AUDIT-R1` (tabla comparativa función por función); `estado-proyecto.md` sección 2 y 16.

---

## MVP sin bloqueo de OTP telefónico — `DRIVER-ONBOARDING-R3.3`

Estado:
ACTIVA

Qué se decidió:
Para el MVP/demo de TukiTuki, la verificación de número por OTP queda **DIFERIDA** hasta contar con un proveedor SMS real. El flujo Driver de cuenta nueva pasa a ser Tu cuenta → login → Sobre ti (foundation), **sin** pasar por ninguna pantalla de verificación de teléfono. `isPhoneVerified=false` ya NO bloquea `resolveDriverApplicationState`/`resolveSessionState`: una cuenta puede seguir su onboarding (incluida cualquier `DriverStatus`) sin tener el teléfono verificado.

Por qué:
Decisión explícita de producto de JuanJo: no usar el mecanismo `OTP_DEMO_ENABLED`/cuentas QA (`OTP-DEMO-R1`, ver `estado-proyecto.md` sección 16) para completar cuentas Driver en el MVP — se prefiere no exigir verificación en absoluto hasta tener un proveedor SMS real, en vez de mantener infraestructura de demo temporal.

Alternativas descartadas:
- Completar la secuencia `OTP-DEMO-R1` (Railway STAGING con `OTP_DEMO_ENABLED`, teléfono QA allowlisted) — descartada para este checkpoint, no eliminada como opción futura si se necesitara antes de tener SMS real.
- Falsificar `isPhoneVerified=true` localmente o vía un bypass de Backend — **explícitamente prohibido**, no se implementó.

Explícitamente NO tocado por esta decisión:
- `phoneE164` sigue siendo **único** en Backend/DB (`UQ_users_phone_e164`, `src/modules/users/entities/user.entity.ts`); `POST auth/register/passenger` sigue respondiendo `409 ConflictException` ("Ya existe una cuenta registrada con este teléfono") si el número ya existe — confirmado leyendo `auth.service.ts` directamente, no supuesto.
- `OTP-R2` (núcleo OTP endurecido) permanece intacto en Backend `main`@`f147a664` — CSPRNG, HMAC-SHA256, Redis, TTL, intentos, cooldown, rate limit por IP/teléfono, invalidación. `POST auth/otp/request`/`auth/otp/verify` en `AuthRepository` (Driver App) tampoco se tocaron: siguen preparados para cuando exista transporte SMS real (`OTP-R3`, sigue `DEFERRED/PAUSED`).
- `DriverStatus` routing (`NO_PROFILE`/`DRAFT` → onboarding, `REJECTED` → rejected, `PENDING_REVIEW` → review, `APPROVED`+`DRIVER` → home, `SUSPENDED` → suspended, `APPROVED` sin `DRIVER` o status desconocido → state-error) se preserva exactamente igual — el único cambio es que ya no depende de `isPhoneVerified`.

Consecuencia en código (`tukituki-driver-app`):
- `DriverSessionKind.phoneVerificationRequired` eliminado del enum (quedaba inalcanzable con la nueva lógica) y con él la pantalla provisional `PhoneVerificationScreen` (`lib/features/auth/presentation/phone_verification_screen.dart`) y su ruta `/onboarding/phone-verification` — código muerto, exclusivo de la implementación provisional de `DRIVER-ONBOARDING-R3.2`, sin nada reutilizable para el futuro OTP-R3 (el futuro flujo de verificación real necesitará una pantalla distinta, con input de código).
- `resolveDriverApplicationState` (`driver_session_state.dart`) ya no lee `user.isPhoneVerified`.
- `AuthRepository.resolveSessionState()` siempre llama `GET drivers/me`, sin condicionarlo a `isPhoneVerified`.
- `CreateAccountScreen`: mensaje de teléfono duplicado (409) actualizado a "Este número de celular ya está registrado. Inicia sesión para continuar.", con acción directa (`SnackBarAction`) para ir a Login — nunca se muestra el error crudo de Backend ("Ya existe una cuenta registrada con este teléfono") ni jerga de "Passenger" en la UI.
- `DriverOnboardingStartScreen` (foundation de "Sobre ti", pasos 2-5 aún sin formulario real): copy ajustado para no afirmar que el teléfono está verificado.

Evidencia:
`lib/features/auth/domain/driver_session_state.dart`, `lib/features/auth/data/auth_repository.dart`, `lib/features/auth/presentation/create_account_screen.dart`, `lib/core/router/driver_onboarding_routes.dart`, `lib/core/router/app_router.dart` (rama `test/driver-onboarding-r3`). Backend verificado por lectura directa (`src/modules/auth/auth.service.ts`, `src/modules/users/entities/user.entity.ts`), sin modificar ningún archivo del repo Backend.

**Actualización (`DRIVER-ONBOARDING-R3.3.2`, 2026-08-17) — UX de teléfono duplicado, versión final aprobada:**
El `SnackBar` original de "teléfono ya registrado" se reemplazó por un modal centrado (`Dialog`), tras feedback físico de JuanJo probando el APK de `R3.3.1` (el `SnackBar` resultaba invasivo/grande). El modal usa el lenguaje visual existente de Driver App (`DriverPalette`: fondo crema, verde oscuro principal, acento amarillo, bordes redondeados 24px): título "Número ya registrado", mensaje "Este número de celular ya tiene una cuenta en TukiTuki. Puedes iniciar sesión para continuar." (sin `409`/`ConflictException`/"Passenger"/stack traces), botón X en la esquina superior derecha (cierra sin navegar ni reenviar el registro) y CTA "Iniciar sesión" (cierra el modal y navega a Login vía el routing existente). Este reemplazo es exclusivo del caso 409/duplicado — errores de red, inesperados y validaciones locales conservan el `SnackBar` de siempre. `_DuplicatePhoneDialog` en `create_account_screen.dart`.

**Actualización (`DRIVER-ONBOARDING-R3.3.3`, 2026-08-17) — Aprobación física y cierre del checkpoint:**
JuanJo probó físicamente el conjunto `R3.2`+`R3.3`+`R3.3.2` en un APK Android real y confirmó **PHYSICAL-REVIEW-PASS**: pantalla "Tu cuenta" funciona; teléfono ya registrado → Backend rechaza correctamente y el modal centrado se muestra tal como se diseñó (título, X, CTA); registro con número nuevo funciona; cuenta se crea correctamente; OTP no aparece en ningún punto del flujo; `isPhoneVerified=false` no bloquea el onboarding; tras el registro avanza correctamente a la foundation del Paso 2 "Sobre ti". Con esa aprobación, se creó un único commit (`feat: add driver onboarding account flow`, commit `3fec449460d2a857f45daeb3bf360a11c48b2472`) con todo el trabajo acumulado de `R3.2`+`R3.3`+`R3.3.2`, publicado en `origin/test/driver-onboarding-r3` — **todavía sin fusionar a `main`**, fusión pendiente de autorización explícita de JuanJo en un checkpoint separado.

---

## Paso 2 "Sobre ti" — foto obligatoria, cámara + galería (`DRIVER-ONBOARDING-R3.4`)

Estado:
ACTIVA — **IMPLEMENTED LOCALLY / PENDING PHYSICAL REVIEW** (2026-08-17). No se marca como checkpoint cerrado hasta que JuanJo confirme la revisión física.

Qué se decidió (decisiones de producto definitivas de JuanJo):

1. **Selección de foto: cámara + galería.** La persona debe poder tomar una foto nueva o elegir una existente de galería — ambas opciones, no solo una.
2. **La foto es obligatoria para completar el Paso 2.** Sin una foto seleccionada y subida correctamente, no se puede avanzar al Paso 3.
3. Campos visibles del Paso 2: Foto, Nombre, Apellidos, Tipo de documento, Número de documento, Fecha de nacimiento, Correo electrónico (opcional). **Sin dirección.**
4. Tipos de documento visibles en la UI: únicamente **DNI** y **CE**. `PASSPORT` sigue soportado técnicamente por Backend (`IdentityDocumentType`) pero no se ofrece como opción en este formulario.

Por qué:
Decisión explícita de producto de JuanJo, tomada después de la auditoría `DRIVER-ONBOARDING-R3.4A` (que confirmó el contrato real de Backend sin encontrar impedimentos técnicos para ninguna de las dos).

Hallazgo técnico que condicionó la implementación (no una decisión de producto, un contrato real de Backend verificado en `R3.4A`):
`POST /storage/uploads/presign` para la categoría `DRIVER_PROFILE_PHOTO` exige que el `DriverProfile` ya exista (el `objectKey` se construye con `driverProfileId`). Por lo tanto el orden real es: `POST drivers/me` (crea el DRAFT, sin foto) → `presign` → `PUT` al bucket → `complete` (Backend persiste `photoObjectKey` sobre el perfil ya creado) — nunca al revés. Si la foto falla después de crear el DRAFT, un reintento nunca repite el `POST` (evitaría un 409 innecesario); esto se resuelve con estado local en la pantalla, no con un mecanismo nuevo de Backend.

No se agregó ningún campo, endpoint, DTO ni regla de validación por diferenciación DNI/CE — Backend usa el mismo regex (`^[A-Z0-9-]{8,20}$`) para los tres tipos de documento; no hay una decisión de producto que diferencie longitudes por tipo todavía.

Alternativas descartadas:
- Solo cámara o solo galería — descartado explícitamente, se ofrecen ambas.
- Permitir continuar al Paso 3 sin foto y resolverla después — descartado explícitamente, la foto bloquea el Paso 2.

Explícitamente NO tocado por esta decisión:
- Backend, Railway, producción.
- El routing por `DriverStatus` (preservado exactamente).
- `isPhoneVerified`/MVP sin OTP (preservado exactamente).
- El registro de número nuevo, login automático y `auth/register/passenger` (preservados exactamente).

Consecuencia en código (`tukituki-driver-app`, rama `test/driver-onboarding-r3`, **sin commit todavía**):
- Nueva pantalla real `DriverOnboardingAboutYouScreen` reemplaza, para `DriverSessionKind.noProfile`, la foundation/placeholder que antes compartía con DRAFT — `driver_onboarding_about_you_screen.dart`.
- `DriverSessionKind.noProfile` ahora enruta a `/onboarding/about-you` (antes iba a `/onboarding/start` igual que DRAFT); `DriverSessionKind.draft` sigue yendo a `/onboarding/start`, cuyo `DriverOnboardingProgress` avanzó de `currentStep: 2` a `currentStep: 3` (el Paso 2 ya está completo cuando existe un `DriverProfile`).
- Nuevos repositorios: `DriverProfileRepository` (`POST drivers/me`), `DriverStorageRepository` (`POST storage/uploads/presign`/`complete`), `DriverPhotoUploader` (el `PUT` real al bucket, con un `Dio()` aislado sin `AuthInterceptor` — nunca se filtra el Bearer token de TukiTuki a un dominio de bucket externo).
- `DriverApplication` (dominio) extendido con `documentType`/`documentNumber`/`birthDate`/`email`/`photoUrl` (los campos reales de `DriverProfileResponseDto`) y el nuevo enum `IdentityDocumentType`.
- Dependencia nueva: `image_picker: ^1.2.3` (única agregada; sin cambios de configuración Android — el paquete funciona sin permisos/manifest adicionales según su propia documentación oficial).

Evidencia:
`lib/features/driver/presentation/onboarding/driver_onboarding_about_you_screen.dart`, `lib/features/driver/data/driver_profile_repository.dart`, `lib/features/driver/data/driver_storage_repository.dart`, `lib/features/driver/data/driver_photo_uploader.dart`, `lib/features/driver/domain/driver_application.dart`, `lib/core/router/driver_onboarding_routes.dart`. Backend verificado por lectura directa (`src/modules/drivers/drivers.controller.ts`, `drivers.service.ts`, `dto/create-driver-profile.dto.ts`, `src/modules/storage/storage.service.ts`, `storage-category.policy.ts`), sin modificar ningún archivo del repo Backend. `flutter analyze` limpio, 601/601 tests en verde (baseline previo 555/555 de `R3.3.2`).

**Actualización (`DRIVER-ONBOARDING-R3.4.2`, 2026-08-17) — DatePicker en español:**
Feedback físico de JuanJo sobre el APK de `R3.4`: el `DatePicker` de "Fecha de nacimiento" aparecía en inglés. Causa raíz: `MaterialApp.router` (`lib/app.dart`) no declaraba `locale`/`supportedLocales`/`localizationsDelegates` — sin `GlobalMaterialLocalizations.delegate`, Flutter usa el locale/traducciones por defecto (inglés) para todo widget Material oficial, incluido `showDatePicker`. Fix quirúrgico, sin calendario personalizado: se agregó la dependencia oficial `flutter_localizations: sdk: flutter` y se configuró `locale: Locale('es', 'PE')`, `supportedLocales: [Locale('es', 'PE'), Locale('es')]` y las tres delegates oficiales (`GlobalMaterialLocalizations`, `GlobalWidgetsLocalizations`, `GlobalCupertinoLocalizations`) en `TukiTukiDriverApp`. Flutter resuelve los recursos de Material vía el bundle base `es` (no existe una variante `es_PE` propia) — comportamiento esperado, no un bug. Solo cambia la presentación: la UI sigue mostrando/enviando `DD/MM/YYYY` al usuario y `YYYY-MM-DD` a Backend sin cambios; la regla de 18 años tampoco se tocó. Validado con tests: locale/delegates declaradas en `MaterialApp`, y el `DatePickerDialog` real (sin el override usado en el resto de los tests) resuelve `Localizations.localeOf(...).languageCode == 'es'`.

**Actualización (`DRIVER-ONBOARDING-R3.4.3`, 2026-08-17) — Aprobación física y cierre del checkpoint:**
JuanJo probó físicamente el APK con el fix de idioma y confirmó **PHYSICAL-REVIEW-PASS** para el conjunto `R3.4`+`R3.4.2`: Paso 2 "Sobre ti" completo (cámara, galería, preview, cambiar foto, foto obligatoria, Nombre, Apellidos, DNI/CE, fecha de nacimiento, regla 18+, correo opcional, `POST drivers/me`, `DriverProfile` DRAFT, presign/PUT/complete de Storage, foto asociada, navegación a la foundation del Paso 3) funcionando de punta a punta, y el calendario ya en español real de Perú (confirmado por JuanJo: "dom, 17 ago", "agosto de 2008", "Cancelar", "ACEPTAR"). Con esa aprobación se creó un único commit (`feat: add driver personal details onboarding`, commit `543e4a8dc101dd7880f463701fff943fe616789a`) con todo el trabajo acumulado de `R3.4`+`R3.4.2`, publicado en `origin/test/driver-onboarding-r3` — **todavía sin fusionar a `main`**, fusión pendiente de autorización explícita de JuanJo en un checkpoint separado. `PASSPORT` se mantiene fuera de la UI (decisión vigente, sin cambios). Limitación ya conocida y sin resolver todavía: no existe `onboardingStep` en Backend — la reanudación completa de un `DriverProfile` en `DRAFT` entre sesiones se resolverá coordinadamente cuando se implemente el Paso 3 "Tu mototaxi" (`DRIVER-ONBOARDING-R3.5`), no antes.

---

## Paso 3 "Tu mototaxi" + reanudación DRAFT por vehículo (`DRIVER-ONBOARDING-R3.5`)

Estado:
ACTIVA — **IMPLEMENTED LOCALLY / PENDING PHYSICAL REVIEW** (2026-08-17). No se marca como checkpoint cerrado hasta que JuanJo confirme la revisión física.

Qué se decidió:
Campos del Paso 3: **Placa, Marca, Modelo, Año, Color, Propio/Alquilado (`ownership`)**. Nunca se pide ni se envía número de motor, número de chasis, VIN, ni `vehicleType` (Backend lo fija automáticamente a `MOTOTAXI`, único valor del enum — confirmado en la auditoría `DRIVER-ONBOARDING-R3.5A`, sin decisión de producto pendiente al respecto).

Por qué:
Decisión de producto ya cerrada desde antes de este checkpoint (ver estructura de 6 pasos, sección 8 de `estado-proyecto.md`); `R3.5A` confirmó que el contrato real de `drivers/me/vehicle` la soporta sin brechas ni ambigüedad (`CreateDriverVehicleDto` no tiene siquiera un campo `vehicleType`).

**Reanudación DRAFT — arquitectura de routing (lo más importante de este checkpoint):**
No existe `onboardingStep` en Backend, ni se creó ninguno nuevo. La única señal disponible para distinguir "Paso 3 pendiente" de "Paso 3 completo" es `GET drivers/me/vehicle` (relación 1:1 con `DriverProfile`, confirmada en `R3.5A`): `404` → Paso 3 todavía no se completó; `200` → ya existe, avanzar a la foundation de Paso 4. Esto se modeló extendiendo `DriverSessionKind` (nunca con un booleano ambiguo, tal como pidió JuanJo): el valor único `draft` se dividió en **`draftNoVehicle`** (→ pantalla real de "Tu mototaxi") y **`draftWithVehicle`** (→ foundation, ahora representando el inicio de Paso 4). `AuthRepository.resolveSessionState()` solo llama `GET drivers/me/vehicle` cuando `DriverProfile.status == DRAFT` — nunca para `PENDING_REVIEW`/`APPROVED`/`REJECTED`/`SUSPENDED`, evitando una llamada de red innecesaria (verificado con test que confirma qué paths se solicitaron).

**Duplicate plate / vehículo ya existente (409):** Backend puede devolver `409` por dos motivos distintos con el mismo status code (`getUniqueConstraintException` en `driver-vehicles.service.ts`): placa duplicada (mensaje exacto `"La placa ya está registrada"`, verificado por lectura del código) → error mostrado en el campo Placa, nunca el modal de "Número ya registrado" de Paso 1 (ese es exclusivo de cuenta duplicada); o "El conductor ya tiene un vehículo registrado" → se resuelve consultando `AuthRepository.getVehicle()` antes de tratarlo como error fatal, y si confirma que el vehículo ya existe, se navega a la foundation de Paso 4 en vez de ocultar la inconsistencia.

Explícitamente NO tocado por esta decisión:
- Backend, Railway, producción.
- El registro de cuenta (Paso 1), "Sobre ti" (Paso 2), MVP sin OTP, `isPhoneVerified`.
- El modal de "Número ya registrado" (exclusivo de Paso 1, no reutilizado aquí).

Consecuencia en código (`tukituki-driver-app`, rama `test/driver-onboarding-r3`, **sin commit todavía**):
- Nueva pantalla real `DriverOnboardingVehicleScreen` (`driver_onboarding_vehicle_screen.dart`), Paso 3.
- `DriverVehicle`/`VehicleOwnership`/`VehicleStatus` (dominio, `driver_vehicle.dart`) y `DriverVehicleRepository` (`createVehicle`/`updateVehicle` — el `PATCH` queda preparado para una futura edición de vehículo `REJECTED`, ninguna pantalla lo invoca todavía).
- `AuthRepository.getVehicle()` (`GET drivers/me/vehicle`, mismo contrato 404→null que `getDriverProfile()`) — deliberadamente NO duplicado en `DriverVehicleRepository`, mismo patrón exacto que `getDriverProfile()` en R3.4 (una única fuente de lectura, reutilizada tanto por el routing como por la recuperación tras un 409 en la propia pantalla).
- `DriverOnboardingStartScreen` (foundation): `currentStep` avanzó de 3 a 4, copy actualizado a "Ya registraste los datos de tu mototaxi... el siguiente paso será agregar tus documentos."
- Tests: 644/644 en verde (baseline previo 606/606 de `R3.4.2`), `flutter analyze` sin issues.

Evidencia:
`lib/features/driver/presentation/onboarding/driver_onboarding_vehicle_screen.dart`, `lib/features/driver/data/driver_vehicle_repository.dart`, `lib/features/driver/domain/driver_vehicle.dart`, `lib/features/auth/domain/driver_session_state.dart`, `lib/features/auth/data/auth_repository.dart`, `lib/core/router/driver_onboarding_routes.dart`. Backend verificado por lectura directa (`src/modules/drivers/driver-vehicles.controller.ts`, `driver-vehicles.service.ts`, `dto/create-driver-vehicle.dto.ts`, `entities/driver-vehicle.entity.ts`), sin modificar ningún archivo del repo Backend.

**Actualización (`DRIVER-ONBOARDING-R3.5.2`, 2026-08-17) — Aprobación física y cierre del checkpoint:**
JuanJo probó físicamente el APK de `R3.5.1` en Android y confirmó **PHYSICAL-REVIEW-PASS**: pantalla "Tu mototaxi" completa (campos, progress, validaciones de placa/año/campos vacíos/ownership vacío, selector Propio/Alquilado de opción única, `POST drivers/me/vehicle` real, loading "Guardando...") y navegación a la foundation de Paso 4 con el copy y progress correctos. **La arquitectura de reanudación DRAFT (diseñada en `R3.5`, sin `onboardingStep`) quedó confirmada en runtime real en ambos sentidos**: cerrar la app con `DriverProfile` DRAFT sin vehículo y reabrirla → `GET drivers/me/vehicle` 404 → vuelve a "Tu mototaxi"; cerrarla después de registrar el vehículo y reabrirla → `GET drivers/me/vehicle` 200 → va directo a la foundation de Paso 4. Sustituye por completo la necesidad de un `onboardingStep` en Backend para distinguir el Paso 3 completado.

**Placa duplicada — cobertura, no `PHYSICAL-PASS`:** la revisión física no incluyó registrar una placa ya existente (no se creó una cuenta extra solo para probar este caso). El comportamiento (409 "La placa ya está registrada" → error mostrado en el campo Placa, nunca el modal de cuenta duplicada de Paso 1) queda validado como **`TEST-COVERED`** por los tests automatizados de `R3.5`, pendiente de una confirmación física futura si JuanJo la considera necesaria.

Con la aprobación física confirmada, se creó un único commit (`feat: add driver vehicle onboarding`, commit `081c5fe2bd5ad712b1494134f5a36266963b03ec`) con todo el trabajo de `R3.5`, publicado en `origin/test/driver-onboarding-r3` — **todavía sin fusionar a `main`**, fusión pendiente de autorización explícita de JuanJo en un checkpoint separado.

---

## Paso 4 "Tus documentos" + reanudación DRAFT por documentos (`DRIVER-ONBOARDING-R3.6`)

Estado:
ACTIVA — **IMPLEMENTED LOCALLY / PENDING PHYSICAL REVIEW** (2026-08-17). No se marca como checkpoint cerrado hasta que JuanJo confirme la revisión física.

Auditoría previa (`DRIVER-ONBOARDING-R3.6A`, solo lectura): confirmó que `drivers/me/documents` está técnicamente completo (`POST`/`GET`/`GET :id`/`PATCH`/`DELETE`), sin brecha de Backend. Hallazgo crítico: el flujo de Storage (`completeDocumentUpload()`, usado por presign/PUT/complete) nunca toca `documentNumber`/`issuedAt`/`expiresAt` — solo reemplaza `fileObjectKey` y revierte `REJECTED`→`DRAFT`. Un reemplazo de archivo preserva la metadata existente. Veredicto: `DECISION-REQUIRED` — Backend no imponía ninguna decisión de UX, así que se le pidieron 3 decisiones explícitas a JuanJo antes de implementar.

Qué se decidió (JuanJo, verbatim resumido):
1. **Selector**: para cada uno de los 3 documentos (Licencia, SOAT, Tarjeta de propiedad/TIV), ofrecer *Tomar una foto* / *Elegir de galería* / *Elegir archivo PDF*. `image_picker` (ya usado en R3.4) cubre cámara/galería; `file_picker` se agregó como único paquete nuevo, exclusivamente para el selector de PDF.
2. **Metadata dentro del Paso 4**: `documentNumber`/`issuedAt` (+ `expiresAt` solo para Licencia/SOAT) se capturan junto con cada archivo, dentro de este mismo paso — nunca diferidos al Paso 5.
3. **Sin preview de PDF**: ningún renderizador visual de PDF — solo ícono + nombre de archivo + estado + botón "Cambiar". Las imágenes sí pueden mostrar una miniatura local (`Image.memory` del archivo recién elegido).

Regla de completitud del Paso 4 (no basta con que existan 3 archivos):
Cada uno de los 3 documentos requeridos necesita archivo + `documentNumber` no vacío + `issuedAt` válido (parseable, no futuro) + (para Licencia/SOAT) `expiresAt` válido (parseable, posterior a `issuedAt`, no vencido). Replicado como función pura `isDriverDocumentComplete()` en `lib/features/driver/domain/driver_document.dart`, con el mismo criterio exacto que `DriverApplicationSubmissionService.validateDocumentData` de Backend (que sigue siendo la fuente de verdad final, aplicada recién en el futuro `POST drivers/me/submit`, fuera de alcance de este checkpoint).

**Reanudación DRAFT — arquitectura de routing (extensión de la de `R3.5`):**
`DriverSessionKind.draftWithVehicle` se dividió en **`draftDocumentsIncomplete`** (→ pantalla real de "Tus documentos") y **`draftDocumentsComplete`** (→ foundation, ahora representando el inicio de Paso 5 "Revisar y enviar"). `AuthRepository.resolveSessionState()` solo llama `GET drivers/me/documents` cuando `DriverProfile.status == DRAFT` **y** ya existe vehículo — nunca antes, evitando una llamada de red innecesaria (verificado con test que confirma qué paths se solicitaron).

**Guardado por documento — orden y fallo parcial:** si hay un archivo nuevo elegido, se hace presign→PUT (`DriverPhotoUploader`, reutilizado tal cual de R3.4, **sin renombrar** — se evaluó generalizarlo pero se descartó para no tocar código ya aprobado/comiteado innecesariamente, "no sobrearquitecturar")→complete→`GET drivers/me/documents` para descubrir el `id` real del documento (`CompleteUploadResponseDto` no lo expone) y actualizar la tarjeta de inmediato; **recién después** se hace `PATCH drivers/me/documents/:id` con la metadata. Si el `complete` tiene éxito pero el `PATCH` falla, la tarjeta conserva el archivo (nunca vuelve a "Agregar documento") y muestra "El archivo se cargó, pero faltan datos por guardar." — un reintento de "Guardar" en ese estado solo repite el `PATCH`, nunca vuelve a subir el archivo. El reemplazo de archivo **no usa `DELETE`** — Backend reemplaza automáticamente al hacer `complete` sobre la misma categoría (confirmado en `R3.6A`), preservando la metadata ya guardada.

Explícitamente NO tocado por esta decisión:
- Backend, Railway, producción.
- Los Pasos 1-3, MVP sin OTP, `isPhoneVerified`.
- `POST drivers/me/submit` (Paso 5 real, fuera de alcance de este checkpoint).

Consecuencia en código (`tukituki-driver-app`, rama `test/driver-onboarding-r3`, **sin commit todavía**):
- Nueva pantalla real `DriverOnboardingDocumentsScreen` (`driver_onboarding_documents_screen.dart`), Paso 4.
- `DriverDocument`/`DriverDocumentType`/`DriverDocumentStatus`/`isDriverDocumentComplete` (dominio, `driver_document.dart`) y `DriverDocumentRepository` (solo `updateDocumentMetadata` — el `GET` vive únicamente en `AuthRepository.getMyDocuments()`, mismo patrón deliberado que `getVehicle()` en R3.5: una única fuente de lectura, sin duplicar la llamada entre el routing y la propia pantalla).
- `storageCategoryForDriverDocumentType()` en `driver_storage_repository.dart` — mapeo 1:1 de cada tipo de documento del Paso 4 a su categoría real de Storage.
- `DriverOnboardingStartScreen` (foundation): `currentStep` avanzó de 4 a 5, copy actualizado a "Ya completaste tus datos, tu mototaxi y tus documentos... en el siguiente paso podrás revisar toda tu información antes de enviar tu solicitud."
- Tests: 711/711 en verde (baseline previo 644/644 de `R3.5.2`), `flutter analyze` sin issues.

Evidencia:
`lib/features/driver/presentation/onboarding/driver_onboarding_documents_screen.dart`, `lib/features/driver/data/driver_document_repository.dart`, `lib/features/driver/domain/driver_document.dart`, `lib/features/driver/data/driver_storage_repository.dart`, `lib/features/auth/domain/driver_session_state.dart`, `lib/features/auth/data/auth_repository.dart`, `lib/core/router/driver_onboarding_routes.dart`. Backend verificado por lectura directa (`src/modules/drivers/driver-documents.controller.ts`, `driver-documents.service.ts`, `driver-application-submission.service.ts`, `dto/create-driver-document.dto.ts`, `dto/update-driver-document.dto.ts`, `entities/driver-document.entity.ts`), sin modificar ningún archivo del repo Backend.

**Actualización (`DRIVER-ONBOARDING-R3.6.2`, 2026-08-17) — Aprobación física y cierre del checkpoint:**
JuanJo probó físicamente el APK de `R3.6.1` en Android y confirmó **PHYSICAL-REVIEW-PASS**: pantalla "Tus documentos" completa (progress Pasos 1-3 completos/4 actual, únicamente los 3 documentos objetivo visibles — nunca los tipos legacy, selector cámara/galería/PDF por documento, metadata correcta por tipo — Licencia y SOAT con vencimiento, TIV sin él —, PDF sin preview visual, preview local de imagen funcionando, calendario `es_PE`, "Continuar" bloqueado con un documento incompleto, tarjeta pasando a "Documento cargado"/"Cambiar" al completarse) y navegación a la foundation de Paso 5 con progress correcto (Pasos 1-4 completos, 5 actual) al completar los 3 documentos.

**Reanudación por documentos — confirmada físicamente en ambos sentidos** (extiende la arquitectura de reanudación de `R3.5`, ahora usando `GET drivers/me/documents` además de `GET drivers/me/vehicle`): cerrar la app con documentos parcialmente completos y reabrirla → vuelve a "Tus documentos" con los ya guardados preservados y el pendiente todavía pendiente (**`PARTIAL RESUME PHYSICAL PASS`**); cerrar la app con los 3 documentos completos y reabrirla → va directo a la foundation de Paso 5, nunca vuelve a Paso 4 (**`COMPLETE RESUME PHYSICAL PASS`**). Confirma en runtime real que `draftDocumentsIncomplete`/`draftDocumentsComplete` distinguen correctamente ambos casos sin necesidad de `onboardingStep` en Backend.

Con la aprobación física confirmada, se creó un único commit (`feat: add driver documents onboarding`, commit `48c8156f08c1c05347b1575ddd2e285ff18269eb`) con todo el trabajo de `R3.6`, publicado en `origin/test/driver-onboarding-r3` — **todavía sin fusionar a `main`**, fusión pendiente de autorización explícita de JuanJo en un checkpoint separado.

---

## Paso 5 "Revisar y enviar" + edición desde el resumen + submit real (`DRIVER-ONBOARDING-R3.7`)

Estado:
ACTIVA — **IMPLEMENTED LOCALLY / PENDING PHYSICAL REVIEW** (2026-08-17). No se marca como checkpoint cerrado hasta que JuanJo confirme la revisión física.

Auditoría previa (`DRIVER-ONBOARDING-R3.7A`, solo lectura): confirmó que `POST drivers/me/submit` está completo (sin body, `200`, `DriverProfileResponseDto`, solo permitido en `DRAFT`/`REJECTED`, transacción con `pessimistic_write`, `missingRequirements`/`invalidRequirements` como arrays estructurados, sin `reviewedAt` en `DriverProfile`) y que `profile`/`vehicle`/`documents` comparten exactamente `[DRAFT, REJECTED]` como únicos estados editables — sin brecha de Backend. Encontró además que `DriverSessionKind.rejected` no distingue si lo rechazado fue el perfil, el vehículo o un documento puntual (Backend sí guarda esa granularidad); se registró como pendiente `DRIVER-ONBOARDING-R3.8`, deliberadamente fuera de alcance de este checkpoint.

Qué se decidió (JuanJo, verbatim resumido):
1. **DNI/CE completo**, sin enmascarar — pantalla propia y autenticada del solicitante.
2. **Documentos en el resumen**: Licencia/SOAT muestran número + fecha de vencimiento; TIV solo número. `issuedAt` nunca se muestra en el resumen (sigue disponible al editar).
3. **Editar por sección**: "Sobre ti", "Tu mototaxi" y "Tus documentos" tienen cada una un botón "Editar", que reutiliza la pantalla real de ese paso en un nuevo modo EDIT — nunca un formulario duplicado.
4. **Sin "Ver documento"**: cero dependencias nuevas (se descartó `url_launcher`/un visor de PDF).
5. **Modal de confirmación** antes del `POST /submit` ("¿Enviar tu solicitud?" / "Mientras tu solicitud esté en revisión no podrás modificar tus datos"), respaldado por la editabilidad real confirmada en `R3.7A`.
6. **CTA exacto**: "Enviar solicitud" (no "Enviar para revisión" ni variantes).

**Arquitectura de la pantalla de revisión:** nueva ruta `/onboarding/review` (`DriverOnboardingRoutes.review`) — `draftDocumentsComplete` ya no va a `start`. Pantalla nueva `DriverOnboardingSubmitReviewScreen` (`driver_onboarding_submit_review_screen.dart`) — **deliberadamente no** `driver_onboarding_review_screen.dart`/`DriverOnboardingReviewScreen`, nombre que ya pertenece a la pantalla de `PENDING_REVIEW` existente (evitar esa colisión de nombres fue una corrección explícita, detectada y corregida durante la implementación antes de tocar nada más). `goToDriverSessionRoute()` ahora pasa el `DriverSessionState` completo como `extra` cuando el destino es `review` (mismo criterio que `application` para rechazada/suspendida) — la pantalla recibe `application`+`vehicle`+`documents` ya resueltos por `resolveSessionState()`, sin 3 GETs adicionales. Si `initialState` es `null` o su `kind` ya cambió, resuelve de nuevo y se autocorrige navegando adonde corresponda.

**Modo EDIT reutilizando los Pasos 2-4** (`DriverOnboardingAboutYouScreenArgs`/`DriverOnboardingVehicleScreenArgs`, parámetro `args` opcional en ambas pantallas — la Documentos no necesita `args`, ya se autocarga): la sola presencia de `args` decide CREATE (`POST`) vs EDIT (`PATCH`), sin booleanos adicionales. La foto/vehículo/documentos existentes ya satisfacen el requisito sin obligar a re-elegir nada. La pantalla de Documentos suma un botón "Editar datos" por tarjeta ya completa, que permite corregir `documentNumber`/`issuedAt`/`expiresAt` con un `PATCH` puro — sin volver a pasar por presign/PUT/complete. Tras guardar en cualquiera de los tres, siempre se hace `resolveSessionState()` + `goToDriverSessionRoute()` — nunca una ruta hardcodeada — así el usuario vuelve a Revisar y enviar con datos frescos, o al paso que corresponda si algo dejó de estar completo.

**Botón de volver sin cerrar sesión en modo EDIT:** `Navigator.of(context).canPop()` distingue si la pantalla llegó empujada (`context.push`, con pila — modo EDIT, siempre desde Revisar y enviar) o vía el routing normal (`context.go`, sin pila — modo CREATE/flujo normal): con pila, el botón de volver solo hace `pop()`; sin pila, mismo comportamiento de siempre (logout). Aplicado también a "Tus documentos", que ahora puede abrirse desde ambos contextos.

**Corrección indispensable encontrada durante la implementación (no una decisión de producto nueva):** `DriverOnboardingAboutYouScreen`/`DriverOnboardingVehicleScreen` navegaban con `context.go(DriverOnboardingRoutes.start)` hardcodeado tras un `POST` exitoso — arrastrado sin corregir desde antes de `R3.5`/`R3.6`, cuando el significado de `start` cambió dos veces sin que estas dos pantallas se actualizaran. Reemplazado por el mismo patrón `resolveSessionState()` + `goToDriverSessionRoute()`: ahora "Sobre ti" navega correctamente a "Tu mototaxi" y "Tu mototaxi" a "Tus documentos" tras un create exitoso, en vez de saltar directo (e incorrectamente) al inicio del Paso 5.

**Submit real:** `DriverProfileRepository.submitApplication()` (`POST drivers/me/submit`, sin body — mismo repository que ya tenía `createProfile`, ahora también `updateProfile`, sin repository nuevo). Pipeline: confirmación → `submitApplication()` → si `status == PENDING_REVIEW`, navega a `review-status` (pantalla `DriverOnboardingReviewScreen` ya existente, **sin cambios**, copy ya aprobado por auditoría). Errores 400 mapeados desde `missingRequirements`/`invalidRequirements` a mensajes amigables con una acción "Revisar..." hacia la sección correspondiente — nunca arrays/JSON crudo; `DRIVER_LICENSE_EXPIRED`/`SOAT_EXPIRED` con mensaje específico. Un 400 sin arrays estructurados (carrera de doble-submit: la solicitud ya no está en `DRAFT`/`REJECTED`), o cualquier error de red/timeout/5xx, dispara una única recuperación vía `GET drivers/me`: si ya está `PENDING_REVIEW`, se trata como envío exitoso; si no, error amigable sin reintento ciego ni polling. Si el `GET` de recuperación también falla, mensaje de incertidumbre explícito (nunca se asume éxito ni fracaso definitivo). 401 limpia sesión y navega a Login (mismo mecanismo que Splash). Doble tap bloqueado en Flutter (`_submitting`) además del `pessimistic_write` real de Backend.

Explícitamente NO tocado por esta decisión:
- Backend, Railway, producción.
- `DriverOnboardingReviewScreen` (`PENDING_REVIEW`) y `DriverOnboardingStartScreen` (`start`, sigue viva solo como destino de "Corregir solicitud" en `REJECTED`) — ninguna de las dos se modificó.
- Corrección granular de una solicitud `REJECTED` (perfil/vehículo/documentos individualmente) — diferida a `DRIVER-ONBOARDING-R3.8`.

Consecuencia en código (`tukituki-driver-app`, rama `test/driver-onboarding-r3`, **sin commit todavía**):
- Nueva pantalla real `DriverOnboardingSubmitReviewScreen` (`driver_onboarding_submit_review_screen.dart`), Paso 5.
- `DriverProfileRepository.updateProfile()`/`submitApplication()` (`PATCH`/`POST drivers/me`, `POST drivers/me/submit`).
- `DriverOnboardingAboutYouScreenArgs`/`DriverOnboardingVehicleScreenArgs` + modo EDIT en `driver_onboarding_about_you_screen.dart`/`driver_onboarding_vehicle_screen.dart`; botón "Editar datos" por tarjeta en `driver_onboarding_documents_screen.dart`.
- `driver_onboarding_routes.dart`: ruta `review` nueva, `draftDocumentsComplete → review`, `goToDriverSessionRoute()` extendido para pasar el `DriverSessionState` completo como `extra` en ese caso.
- Tests: 757/757 en verde (baseline previo 711/711 de `R3.6.2`), `flutter analyze` sin issues. **0 dependencias nuevas.**

Evidencia:
`lib/features/driver/presentation/onboarding/driver_onboarding_submit_review_screen.dart`, `lib/features/driver/presentation/onboarding/driver_onboarding_about_you_screen.dart`, `lib/features/driver/presentation/onboarding/driver_onboarding_vehicle_screen.dart`, `lib/features/driver/presentation/onboarding/driver_onboarding_documents_screen.dart`, `lib/features/driver/data/driver_profile_repository.dart`, `lib/core/router/driver_onboarding_routes.dart`, `lib/core/router/app_router.dart`. Backend verificado por lectura directa (`src/modules/drivers/drivers.controller.ts`, `driver-application-submission.service.ts`, `drivers.service.ts`, `driver-vehicles.service.ts`, `driver-documents.service.ts`, `src/modules/admin-drivers/admin-driver-review.service.ts`, `src/modules/storage/storage.controller.ts`), sin modificar ningún archivo del repo Backend.

**Actualización (`DRIVER-ONBOARDING-R3.7.2`, 2026-08-17) — Hallazgo físico y hotfix, `R3.7` sigue sin cerrarse:**
JuanJo probó físicamente el APK de `R3.7.1`: el resumen y el submit final funcionaron correctamente, pero encontró que el flujo Editar tenía un bug de refresco — al editar un campo desde cualquier sección, Backend persistía el cambio correctamente (confirmado en el panel Admin), pero "Revisar y enviar" seguía mostrando el valor anterior al volver. Root cause confirmada (reproducida primero con un test que falla contra el código sin corregir): `DriverOnboardingSubmitReviewScreen` solo aplicaba `initialState` dentro de `initState()`; como las pantallas de edición navegan de vuelta con `context.go(review, extra: freshState)` (no `context.pop()`) hacia la misma ruta, Flutter/`go_router` puede reutilizar el `State` existente en vez de recrearlo, y `initState()` nunca se repetía. Fix: `didUpdateWidget()` agregado a esa pantalla para detectar un `initialState` nuevo y reemplazar los tres campos (`application`/`vehicle`/`documents`) juntos. La arquitectura de navegación (state fresco vía `extra`, sin GETs redundantes en la entrada normal) no cambió — ya era correcta, solo faltaba consumirla en el caso de `State` reutilizado. 761/761 tests en verde (+4 de regresión). **Sin commit todavía** — pendiente de reprueba física del flujo Editar → Revisar y enviar.

**Actualización (`DRIVER-ONBOARDING-R3.7.4`, 2026-08-17) — Aprobación física integral y cierre del checkpoint:**
JuanJo completó la prueba física integral de `R3.7` (Paso 5 "Revisar y enviar", los tres resúmenes por sección, modal de confirmación, CTA, submit real completo hasta `PENDING_REVIEW`/"Solicitud en revisión") y confirmó **PHYSICAL-REVIEW-PASS**. Con el APK del hotfix (`R3.7.3`), reprobó específicamente el flujo Editar → Guardar → Revisar y enviar en las tres secciones (Sobre ti, Tu mototaxi, Tus documentos) y confirmó que los datos frescos aparecen de inmediato al volver, sin cerrar la app ni reiniciar sesión — **`REFRESH-AFTER-EDIT PHYSICAL PASS`**, cerrando también `R3.7.2`.

Con la aprobación física confirmada (checkpoint principal + hotfix), se creó un único commit (`feat: add driver onboarding review and submission`, commit `0201b42cdf880f406c0b80ffb7ad02d9bb1fecf3`) con todo el trabajo de `R3.7` + el hotfix de `R3.7.2`, publicado en `origin/test/driver-onboarding-r3` — **todavía sin fusionar a `main`**, fusión pendiente de autorización explícita de JuanJo en un checkpoint separado.

## `DRIVER-ONBOARDING-R3.8` — Correcciones requeridas + reenvío tras rechazo (2026-08-18, 🟡 IMPLEMENTED LOCALLY / PENDING PHYSICAL REVIEW, sin commit todavía)

Auditoría previa (`R3.8A`, solo lectura) confirmó los hechos de Backend que sostienen esta implementación: `PATCH admin/drivers/:driverProfileId/reject` acepta `{ profileReason?, vehicleReason?, documents?: [{documentId, reason}] }`; `profile.status = REJECTED` es **incondicional** en cualquier reject, aunque el admin solo haya observado el vehículo o un documento — por lo tanto `application.status == REJECTED` nunca es, por sí solo, señal de "hay algo que corregir"; cada recurso (perfil/vehículo/documento) se resetea a `DRAFT` de forma independiente en su propio `PATCH` exitoso (el de perfil también limpia `submittedAt: null`); Backend no exige que el conductor haya corregido efectivamente un recurso `REJECTED` antes de reenviar (solo valida status enum + estructura); Backend no conserva historial de rechazos.

**Decisiones de producto de JuanJo (A-H, verbatim, base de toda la implementación):**
- **A.** Sí, construir una pantalla central "Correcciones requeridas".
- **B.** Mostrar siempre las 5 secciones (Sobre ti, Tu mototaxi, Licencia, SOAT, TIV) — las observadas "⚠ Requiere corrección", el resto "✓ Sin observaciones".
- **C.** No se puede editar una sección NO observada desde esta pantalla — el CTA "Corregir" solo aparece en las realmente observadas.
- **D.** Un "Corregir" sobre un documento observado abre Paso 4 **enfocado directamente** en ese documento.
- **E.** El reenvío está deshabilitado mientras haya cualquier corrección pendiente. Regla exacta (nunca `profile.status == REJECTED`): `pendingCorrections = profile.rejectionReason?.trim().isNotEmpty == true OR vehicle.status == REJECTED OR algún documento TARGET.status == REJECTED` (TARGET = `DRIVER_LICENSE`/`SOAT`/`VEHICLE_REGISTRATION`).
- **F.** Tras corregir todo, no se crea una segunda pantalla de resumen — se reutiliza `DriverOnboardingSubmitReviewScreen` en modo "resubmission", alcanzada con un CTA "[ Revisar y reenviar ]" desde "Correcciones requeridas".
- **G.** El CTA del modo reenvío del resumen debe decir exactamente "Reenviar solicitud" (nunca "Enviar solicitud").
- **H.** El texto de la observación del admin se muestra exacto — sin resumir, reinterpretar ni modificar (solo se acepta trim de espacios externos).

**`DriverSessionKind.correctionsRequired`** reemplaza a `rejected`. Se llega aquí desde dos casos: `application.status == REJECTED` (siempre, cargando `vehicle`/`documents` igual que `DRAFT`), o `application.status == DRAFT` cuando `hasPendingDriverCorrections()` es `true` **o** el marcador local de reenvío sigue activo — este segundo caso cubre el hueco de contexto que Backend no puede llenar: un perfil corregido vuelve a `DRAFT` sin ningún rastro de que viniera de un rechazo, así que sin el marcador local la app perdería el hilo del ciclo de reenvío en cuanto el usuario corrigiera el último recurso observado (pasaría a verse como un `DRAFT` recién creado). El marcador (`StorageKeys.driverResubmissionContext`, mismo `flutter_secure_storage` ya usado para tokens de sesión — **0 dependencias nuevas**) nunca guarda la razón de rechazo ni ningún dato del expediente, solo un booleano; `AuthRepository.markResubmissionContext()` lo activa la primera vez que `resolveSessionState()` ve `REJECTED`, y se limpia en `clearResubmissionContext()` (tras cualquier submit exitoso, inicial o de reenvío) y en `clearSession()` (evita que sobreviva a un cambio de cuenta en el mismo dispositivo).

**`hasPendingDriverCorrections({application, vehicle, documents})`** (función pura, `driver_session_state.dart`) replica la regla E exactamente: `application?.rejectionReason` no vacío (trim), `vehicle?.status == VehicleStatus.rejected`, o algún documento de `requiredDriverOnboardingDocumentTypes` con `status == DriverDocumentStatus.rejected` — los tipos legacy nunca cuentan. Nunca lee `application.status`.

**Se descartó explícitamente** un enum `DriverOnboardingReturnTarget` (`normalFlow`/`review`/`corrections`) que el prompt original sugería para decidir a dónde volver tras una edición: el patrón ya existente desde `R3.7` — "al guardar, `resolveSessionState()` fresco + `goToDriverSessionRoute()`" — ya es puramente derivado de datos (estado real de Backend + marcador local), así que enruta solo, sin necesidad de que ninguna pantalla recuerde "de dónde vine". Añadir el enum habría sido sobrearquitectura sin beneficio real (instrucción explícita de JuanJo: "no sobrearquitecturar").

**Pantalla nueva `DriverOnboardingCorrectionsScreen`** (`driver_onboarding_corrections_screen.dart`, ruta `/onboarding/corrections`, `DriverOnboardingRoutes.corrections`): mismo patrón `initState`/`didUpdateWidget`/`_isUsable`/`_assignFromState` que `DriverOnboardingSubmitReviewScreen` (relectura explícita de la lección `R3.7.2`: si esta pantalla se reabre por `context.go` a la misma ruta mientras `go_router` reutiliza el `State`, `initState()` no vuelve a correr y hace falta `didUpdateWidget()` para no quedarse con datos obsoletos). Renderiza las 5 secciones siempre (decisión B); cada una muestra el texto exacto de `rejectionReason` (perfil/vehículo) o `document.rejectionReason` (decisión H) y un botón "Corregir" solo si está observada (decisión C). "Corregir" en Sobre ti/Tu mototaxi empuja (`context.push`) a esas pantallas con sus `Args` existentes de `R3.7`, reutilizadas tal cual; en un documento, empuja a Documentos con `DriverOnboardingDocumentsScreenArgs(focusDocumentType: tipo, editableDocumentTypes: {tipo})` (decisión D). Al volver de cualquiera, siempre recarga vía `resolveSessionState()`. El botón "Revisar y reenviar" está deshabilitado mientras `hasPendingDriverCorrections()` sea `true` sobre el snapshot local, pero **antes de navegar** vuelve a resolver el estado real contra Backend y solo entonces decide si ir a `review` (extra: el `DriverSessionState` fresco) — nunca confía ciegamente en el snapshot ya cargado (decisión E, gate no negociable pese a que Backend técnicamente permitiría el reenvío sin la corrección real).

**`DriverOnboardingDocumentsScreen`** gana un parámetro `args` opcional (`DriverOnboardingDocumentsScreenArgs { focusDocumentType, editableDocumentTypes }`, `null` = comportamiento sin restricción de `R3.6`/`R3.7`). `focusDocumentType` hace `Scrollable.ensureVisible` sobre la tarjeta correspondiente tras el primer frame post-carga. `editableDocumentTypes` no nulo vuelve solo-lectura (sin selector de archivo ni "Editar datos") cualquier tarjeta fuera del set — implementado como una tarjeta alternativa simplificada (`_buildReadOnlyCard`) en vez de deshabilitar botones individuales, ya que en el flujo de corrección los otros documentos siempre están completos.

**`DriverOnboardingSubmitReviewScreen`** gana modo dual sin una segunda pantalla (decisión F): `_isUsable()` acepta `draftDocumentsComplete` (siempre) o `correctionsRequired` (solo si `!hasPendingDriverCorrections`); `_isResubmission` se deriva de `state.kind` dentro de `_assignFromState()`, nunca de un parámetro explícito. Con `_isResubmission == true`: título "Revisar y reenviar", CTA "Reenviar solicitud"/"Reenviando..." (decisión G), modal "¿Reenviar tu solicitud?", y los tres botones "Editar" no se renderizan en absoluto (`_ReviewSectionCard.onEdit` ahora nullable — las correcciones granulares solo se hacen desde "Correcciones requeridas", nunca desde el resumen de reenvío). `clearResubmissionContext()` se llama en todo camino de éxito (incluyendo el de recuperación tras fallo ambiguo), sin importar si fue envío inicial o reenvío — idempotente, no debería quedar activo por error.

Explícitamente NO tocado por esta decisión:
- Backend, Railway, producción.
- `DriverOnboardingReviewScreen` (`PENDING_REVIEW`, sin cambios) y el submit real de `R3.7` (`submitApplication()` reutilizado tal cual, sin endpoint nuevo).
- Historial de rechazos anteriores y notificaciones — fuera de alcance, ninguna de las dos existe hoy.
- `DriverOnboardingRejectedScreen`/`DriverOnboardingStartScreen` **eliminadas** (junto con sus tests) — dejaron de ser destino funcional de ningún `DriverSessionKind`, reemplazadas por completo por `DriverOnboardingCorrectionsScreen`.

Consecuencia en código (`tukituki-driver-app`, rama `test/driver-onboarding-r3`, **sin commit todavía**):
- Nueva pantalla `driver_onboarding_corrections_screen.dart` (`DriverOnboardingCorrectionsScreen`).
- Eliminadas `driver_onboarding_rejected_screen.dart`/`driver_onboarding_start_screen.dart` y sus tests.
- `driver_session_state.dart`: `DriverSessionKind.correctionsRequired` (reemplaza `rejected`), `hasPendingDriverCorrections()`, `resolveDriverApplicationState()` extendido con `hasResubmissionContext`.
- `secure_storage.dart`: `StorageKeys.driverResubmissionContext`.
- `auth_repository.dart`: `resolveSessionState()` carga `vehicle`/`documents` también en `REJECTED`; `markResubmissionContext()`/`hasResubmissionContext()`/`clearResubmissionContext()` (scoped por `userId` desde `R3.8B`, ver actualización debajo; `clearSession()` deliberadamente no lo toca).
- `driver_onboarding_documents_screen.dart`: `DriverOnboardingDocumentsScreenArgs` (`focusDocumentType`/`editableDocumentTypes`), scroll-to, tarjetas solo-lectura.
- `driver_onboarding_submit_review_screen.dart`: modo dual (`_isResubmission`), `_ReviewSectionCard.onEdit` nullable, `clearResubmissionContext()` en todo camino de éxito.
- `driver_onboarding_routes.dart`/`app_router.dart`: ruta `corrections` nueva, rutas `rejected`/`start` eliminadas.
- Tests: **793/793 en verde** (baseline previo 761/761 de `R3.7.2`), `flutter analyze` sin issues. **0 dependencias nuevas** en `dependencies` (`flutter_secure_storage_platform_interface` agregado solo como `dev_dependency`, para simular `flutter_secure_storage` en memoria en tests vía el propio helper `TestFlutterSecureStoragePlatform` que expone ese paquete — sin canal de plataforma real).

Evidencia:
`lib/features/driver/presentation/onboarding/driver_onboarding_corrections_screen.dart`, `lib/features/driver/presentation/onboarding/driver_onboarding_documents_screen.dart`, `lib/features/driver/presentation/onboarding/driver_onboarding_submit_review_screen.dart`, `lib/features/auth/domain/driver_session_state.dart`, `lib/features/auth/data/auth_repository.dart`, `lib/core/storage/secure_storage.dart`, `lib/core/router/driver_onboarding_routes.dart`, `lib/core/router/app_router.dart`. Backend verificado por lectura directa en la auditoría `R3.8A` (`admin-driver-review.service.ts`, `drivers.service.ts`, `driver-vehicles.service.ts`, `driver-documents.service.ts`), sin modificar ningún archivo del repo Backend.

**Actualización (`DRIVER-ONBOARDING-R3.8B`, 2026-08-18) — Hotfix pre-review: marcador de reenvío scoped por cuenta, `R3.8` sigue sin cerrarse:**
JuanJo encontró, antes del APK de prueba física, que el marcador `driverResubmissionContext` (agregado en `R3.8`) tenía dos problemas: era una **key global** de `flutter_secure_storage` compartida por toda la instalación (no por cuenta), y `clearSession()` (logout) lo borraba incondicionalmente. Caso crítico concreto: un conductor corrige toda su solicitud rechazada (perfil vuelve a `DRAFT`, sin `rejectionReason`, vehículo/documentos sin observaciones) pero cierra sesión **antes** de tocar "Reenviar solicitud" — al volver a iniciar sesión con la misma cuenta, Backend ya no tiene ningún rastro de que ese `DRAFT` viene de un rechazo (no conserva historial), así que sin el marcador local la app mostraría por error el resumen inicial ("Enviar solicitud") en vez del de reenvío. Un segundo riesgo, más grave: al ser global, dos conductores compartiendo el mismo dispositivo heredarían el contexto de reenvío del otro.

Fix: el marcador ahora vive scoped por cuenta — `StorageKeys.driverResubmissionContext(userId)` construye la key a partir de `AuthenticatedUser.id` (identificador estable ya expuesto por `GET auth/me`, nunca DNI/email/teléfono, nunca PII). `markResubmissionContext`/`hasResubmissionContext`/`clearResubmissionContext` ahora exigen `userId` explícito. `clearSession()` **ya no toca el marcador**: cerrar sesión termina la sesión, no el ciclo administrativo de la solicitud — el marcador sobrevive logout/login de la misma cuenta y reinicios de la app, exactamente como debía funcionar desde `R3.8`. Se agregó además una limpieza defensiva dentro de `resolveSessionState()`: cualquier estado en el que Backend confirma sin ambigüedad que el ciclo terminó (`PENDING_REVIEW`/`APPROVED`/`SUSPENDED`) limpia el marcador de esa cuenta — refuerza, de forma idempotente, la limpieza explícita que ya hacía `DriverOnboardingSubmitReviewScreen` tras un submit exitoso (útil si ese paso nunca llegó a ejecutarse, p.ej. la app se cerró a mitad del submit).

Ningún cambio de UX: las 5 secciones de "Correcciones requeridas", el gate de reenvío, el modo "resubmission" de "Revisar y enviar" y el texto exacto del admin siguen igual — este hotfix es puramente de persistencia. 801/801 tests en verde (+8 de regresión: aislamiento entre cuentas, supervivencia a logout/login de la misma cuenta, y limpieza en los tres estados terminales). **Sin commit todavía** — `R3.8` (incluyendo este hotfix) sigue `PENDING PHYSICAL REVIEW`.

**Actualización (`DRIVER-ONBOARDING-R3.8.1`, 2026-08-18) — APK release para prueba física:**
Con `R3.8`+`R3.8B` en verde (801/801, `flutter analyze` limpio), se generó `TukiTuki-Conductor-Onboarding-R3.8-Correcciones-Test.apk` (llaves de debug, uso interno) para la prueba física del flujo completo de "Correcciones requeridas"/reenvío. Sin cambios de código ni documentación en este paso.

**Actualización (`DRIVER-ONBOARDING-R3.8.2`, 2026-08-18) — Hallazgo físico y hotfix: "Revisar y reenviar" no se habilitaba hasta reiniciar la app, `R3.8` sigue sin cerrarse:**
JuanJo probó físicamente el APK de `R3.8.1` con un caso real: Admin observó únicamente el SOAT. Las 5 secciones se mostraron correctamente (solo SOAT con "⚠ Requiere corrección" + texto exacto + "Corregir", el resto "Sin observaciones") y "Revisar y reenviar" apareció deshabilitado — **PASS** hasta ahí. Tras corregir la metadata del SOAT (sin reemplazar archivo) y volver, la pantalla sí actualizó las 5 secciones a "Sin observaciones" — pero "Revisar y reenviar" **siguió deshabilitado** hasta cerrar y reabrir la app por completo.

Root cause confirmada (reproducida primero con un test que falla contra el código sin corregir — mismo criterio que `R3.7.2`, no asumir la causa): no era un problema de datos obsoletos (las tarjetas ya reflejaban el estado fresco correctamente) ni del marcador de reenvío de `R3.8B`. Era el flag local `_busy` de `DriverOnboardingCorrectionsScreen` (pensado para evitar doble-tap mientras una corrección está en curso) quedando **permanentemente `true`**: `DriverOnboardingDocumentsScreen` vuelve a "Correcciones requeridas" a través de su botón real "Continuar", que llama `goToDriverSessionRoute()` (`context.go`, el mismo patrón ya usado en `R3.7`) — no un `Navigator.pop()` imperativo. Cuando el destino de ese `go()` coincide con la ruta que originó el `context.push()`, GoRouter puede completar esa navegación reutilizando el `State` (vía `didUpdateWidget`, que sí trae los datos frescos — de ahí que las tarjetas se vieran bien) sin que eso garantice que el `Future` del `push()` original llegue a resolverse a tiempo. Nada reseteaba `_busy` fuera de esa continuación, así que el CTA quedaba atado a un flag atascado, completamente desacoplado de `pending` (que sí era correcto). Es un bug específico de esta pantalla nueva — `DriverOnboardingSubmitReviewScreen` no tiene un flag equivalente porque nunca deshabilita sus botones "Editar" mientras navega.

Fix: `_assignFromState()` — fuente única usada por `initState`, `didUpdateWidget` y `_loadFresh()` — ahora también resetea `_busy = false`. Recibir un estado fresco y usable es, por definición, prueba de que ya no queda ninguna navegación de corrección en curso, sin depender de que la continuación del `push()` original se ejecute. Se auditó también, explícitamente, que `application.status == REJECTED` con `rejectionReason == null` (perfil nunca observado, solo un hijo — exactamente el caso físico, porque Backend deja `profile.status = REJECTED` incondicionalmente en cualquier reject) **nunca** bloquea el CTA: esa regla ya vivía correcta en `hasPendingDriverCorrections()` desde `R3.8` (nunca lee `application.status`), ahora cubierta también con tests explícitos a nivel de pantalla.

Ningún cambio de UX — el fix es puramente de reactividad de un flag interno. 804/804 tests en verde (+3: el caso físico exacto con retorno vía `context.go`, y los dos casos `REJECTED`+`rejectionReason` nulo/no nulo). Los tests de aislamiento entre cuentas y persistencia logout/login de `R3.8B` siguen en verde sin cambios. **Sin commit todavía** — `R3.8` (incluyendo `R3.8B` y este hotfix) sigue `PENDING PHYSICAL REVIEW`, pendiente de reprueba específica de este flujo.

**Actualización (`DRIVER-ONBOARDING-R3.8.3`, 2026-08-18) — APK release para reprueba del hotfix CTA:**
Con `R3.8`+`R3.8B`+`R3.8.2` en verde (804/804, `flutter analyze` limpio), se generó `TukiTuki-Conductor-Onboarding-R3.8.3-CTARefresh-Test.apk` (llaves de debug, uso interno) para la reprueba física específica de que "Revisar y reenviar" se habilita de inmediato tras la última corrección. Sin cambios de código ni documentación en este paso.

**Actualización (`DRIVER-ONBOARDING-R3.8.4`, 2026-08-18) — Aprobación física integral y cierre del checkpoint:**
JuanJo completó la prueba física integral de `R3.8` (con el APK de `R3.8.3`, que incluye el hotfix `R3.8.2` sobre `R3.8`+`R3.8B`) y confirmó **PHYSICAL-REVIEW-PASS** de punta a punta: las 5 secciones de "Correcciones requeridas" con un único recurso observado (solo la sección observada con el texto exacto del admin + "Corregir", el resto "Sin observaciones" sin acción — nunca se ofreció edición en una sección no observada); "Corregir" abrió Paso 4 con el documento correcto editable y los otros dos en solo-lectura; la corrección de metadata (sin reemplazar archivo) se guardó vía `PATCH` y Backend confirmó el reset del recurso (`REJECTED → DRAFT`, motivo limpio); "Revisar y reenviar" permaneció deshabilitado mientras hubo observación pendiente.

**`CTA-REFRESH PHYSICAL PASS`**: tras corregir la última observación y volver, el botón se habilitó de inmediato, sin cerrar la app ni reiniciar sesión — confirma físicamente el fix de `R3.8.2`.

**`RESUBMISSION-CONTEXT LOGOUT/LOGIN PHYSICAL PASS`**: con todo corregido pero antes de reenviar, cerrar sesión y volver a iniciarla con la misma cuenta recuperó correctamente el flujo de reenvío (nunca el onboarding inicial) — confirma físicamente el marcador scoped por cuenta de `R3.8B`.

El resumen de reenvío (`DriverOnboardingSubmitReviewScreen` reutilizada, sin botones "Editar" generales, CTA "Reenviar solicitud") y el reenvío real (`POST drivers/me/submit → PENDING_REVIEW → "Solicitud en revisión"`, pipeline de `R3.7` sin cambios) se confirmaron funcionando correctamente. **`RESUBMIT-REOPEN PHYSICAL PASS`**: tras el reenvío exitoso, cerrar y reabrir la app por completo llevó directo a "Solicitud en revisión", nunca de regreso a "Correcciones requeridas" — confirma que el marcador se limpia correctamente al confirmarse `PENDING_REVIEW`.

**Reglas finales registradas** (checkpoint cerrado):
- `pendingCorrections = application.rejectionReason no vacío OR vehicle.status == REJECTED OR algún documento objetivo (DRIVER_LICENSE/SOAT/VEHICLE_REGISTRATION).status == REJECTED` — nunca `application.status == REJECTED` como criterio único.
- Marcador de reenvío: local, no sensible (nunca PII), scoped por cuenta (nunca global), sobrevive navegación/restart/logout-login de la misma cuenta, nunca se comparte entre cuentas, se limpia al confirmarse un estado terminal aplicable (`PENDING_REVIEW`/`APPROVED`/`SUSPENDED`).

Con la aprobación física confirmada, se cerró el checkpoint: `flutter analyze` (limpio) y `flutter test` (804/804) re-verificados una última vez, documentación externa actualizada (este documento y `estado-proyecto.md`), y se creó **un único commit** en `tukituki-driver-app` con todo el trabajo de `R3.8`+`R3.8B`+`R3.8.2` (`feat: add driver rejected application corrections`, commit `d0abe57664040cd383940c5d6e14e71522a4e76f`), publicado con `git push` a `origin/test/driver-onboarding-r3` — **todavía sin fusionar a `main`**, fusión pendiente de autorización explícita de JuanJo en un checkpoint separado.

## `DRIVER-ONBOARDING-R3.9A`/`R3.9B`/`R3.10`/`R3.10.1` — Auditoría final, limpieza, integración a `main` y cierre definitivo (2026-08-18)

**`R3.9A` (auditoría de solo lectura)**: antes de integrar `test/driver-onboarding-r3` a `main`, se auditó todo el onboarding acumulado (`R3.3`–`R3.8`) de punta a punta: routing (sin loops, sin rutas inaccesibles), seguridad de subida de archivos (confirmado `Dio` aislado sin `Authorization` en ambos call sites — foto de perfil y documentos), PII/logging (sin campos sensibles en ningún `debugPrint`), dependencias (sin huérfanas, sin `url_launcher`/visor PDF colados), dead code, calidad estructural de tests. Checklist funcional de 25 ítems: 23 `PHYSICAL PASS` (evidencia ya registrada en este documento por cada checkpoint `R3.3`–`R3.8`), 2 `TEST-COVERED ONLY` (placa duplicada de `R3.5`, aislamiento cross-account del marcador — ninguno con datos físicos reales). Veredicto: `BLOCKERS: 0`, `MAJOR: 0`, `MINOR: 2` → `READY-FOR-MAIN-INTEGRATION`.

**`R3.9B` (limpieza)**: se corrigieron exclusivamente los 2 `MINOR` — un comentario doc de `DriverOnboardingDocumentsScreen` que todavía referenciaba la ruta `start` eliminada en `R3.8`, y 4 stubs muertos de ruta `/onboarding/start` en routers locales de test (el cuarto, en `create_account_screen_test.dart`, se detectó al reconfirmar el hallazgo, no estaba en el listado original de `R3.9A`). Sin cambios funcionales, 804/804 tests sin cambio de conteo. Commit `chore: clean up obsolete onboarding routes`, publicado en `origin/test/driver-onboarding-r3`.

**`R3.10` (integración a `main`)**: reconfirmado `main == origin/main == b562d24564ecab65cb7833cde0716924f7192c66` y que seguía siendo ancestro directo de la rama de test, se hizo `git checkout main` + `git merge --ff-only test/driver-onboarding-r3` (fast-forward puro, sin merge commit) → `main` avanzó exactamente al commit ya aprobado `2857ff227086442f1589a24f31626dc1b801d426`. Revalidado en `main` antes de publicar (`flutter analyze` limpio, 804/804), `git push origin main` confirmado. `test/driver-onboarding-r3` se preservó intacta (local y remota) hasta el smoke test físico. Se generó un APK `release` **desde `main`** (`TukiTuki-Conductor-Onboarding-Main-R3-SmokeTest.apk`) para ese smoke test.

**`R3.10.1` (smoke test físico y cierre definitivo)**: JuanJo instaló el APK construido desde `main` y confirmó **`MAIN-PHYSICAL-SMOKE-PASS`**: login/sesión; `PENDING_REVIEW` sobrevive cerrar/reabrir la app; routing de cuenta `DRAFT` correcto en sus 4 sub-casos (sin perfil/sin vehículo/documentos incompletos/todo completo); cuenta `REJECTED` lleva a "Correcciones requeridas" con las observaciones correspondientes. Con el smoke test en verde, se actualizó la documentación externa que seguía describiendo el `main` anterior — incluidos los 5 documentos snapshot de `App-driver/` (`arquitectura.md`/`convenciones.md`/`errores-conocidos.md`/`flujo-de-trabajo.md`/`glosario.md`), congelados desde `main@b562d24` y ahora refrescados contra el `main` real (`2857ff2`) — y se eliminó `test/driver-onboarding-r3`, local y remota, único borrado de rama de todo el bloque `R3`.

**Reglas y arquitectura finales del onboarding, ya integradas y físicamente probadas en `main`**: las mismas registradas en la actualización de `R3.8.4` arriba (regla `pendingCorrections`, marcador de reenvío scoped por cuenta) — sin cambios adicionales en `R3.9`/`R3.10`, solo verificación, limpieza cosmética e integración.

Explícitamente NO tocado en todo este bloque de cierre (`R3.9A`–`R3.10.1`): Backend, Railway, Admin, Passenger, producción.

---

## `CROSS-APP-R4.3A` — Identidad compacta real del Passenger en solicitud y viaje activo (2026-08-18)

Estado:
**FINAL-CLOSED-ON-MAIN.** Integrado a `main`@`9c2a7f75da3abe375dd16a3336507145aece7455` mediante fast-forward puro (`CROSS-APP-R4.3J`, 2026-08-18) y confirmado por el smoke físico final de JuanJo sobre un APK construido específicamente desde `main` contra Backend STAGING desde `main` (7/7 PASS, incluye Driver viendo la identidad real del Passenger en solicitud y viaje activo). `flutter analyze` limpio, 824/824 tests. `test/r4-ride-identities` eliminada (local y remota) tras confirmar contención total en `main`.

Qué se decidió:
Nuevo `lib/core/display_name.dart` (`displayCompactNameFromInitial()`) — mismo helper que `tukituki-passenger-app` implementa en paralelo para el mismo checkpoint cruzado, copiado independientemente en cada repo (no existe un package Dart compartido entre Driver y Passenger). `AssignedPassenger.lastNameInitial` y `DriverRideOffer.passengerLastNameInitial` (ambos nuevos, ya expuestos por Backend en `CROSS-APP-R4.3`) se consumen en la tarjeta de solicitud entrante (`driver_home_screen.dart`) y en la pantalla de viaje activo (`driver_active_ride_screen.dart`, tarjeta del Passenger y encabezado de estado "llegaste"), reemplazando el `firstName` solo por "Nombre + inicial" (p. ej. "Juan P."). Sin campos adicionales de PII, sin cambio de lógica de ride.

Por qué:
Contraparte de `CROSS-APP-R4.3` en `tukituki-passenger-app`: la identidad real bidireccional (Passenger↔Driver) requiere que ambas apps consuman el mismo campo `lastNameInitial` ya agregado por Backend a las cuatro DTO de identidad de ride.

Evidencia:
`lib/core/display_name.dart`, `lib/features/driver/domain/driver_assigned_passenger.dart` (`lastNameInitial`), `lib/features/driver/domain/driver_ride_offer.dart` (`passengerLastNameInitial`), `lib/features/driver/presentation/driver_active_ride_screen.dart`, `lib/features/driver/presentation/driver_home_screen.dart`. `flutter analyze` limpio, 824/824 tests (sin cambio de conteo — solo consumo de un campo ya expuesto por Backend). **`PHYSICAL-REVIEW-PASS`** confirmado por JuanJo: Driver recibe la solicitud mostrando la identidad real del Passenger como "Nombre + inicial" (formato aprobado, p. ej. "Juan P."), sin texto genérico.

**Estado final: `DRIVER-ONBOARDING-R3` — `FINAL-CLOSED-ON-MAIN`.** `main` de `tukituki-driver-app` queda en `2857ff227086442f1589a24f31626dc1b801d426` como única rama activa del repo, con el onboarding completo de Driver (registro → perfil → vehículo → documentos → revisión → envío → revisión del admin → correcciones → reenvío) integrado, probado automatizada y físicamente, y documentado.

---

## `R4.4B` — Cadencia de GPS del viaje activo sube de 10s a 3s (2026-08-20)

Estado:
**FINAL-CLOSED-ON-MAIN.** Integrado a `main`@`2743274900ef33374a76606b62ff841d476c232f` mediante fast-forward puro (2026-08-20), tras `PHYSICAL-REVIEW-PASS` de JuanJo (dispositivo real, campus universitario, terreno abierto — ver `estado-proyecto.md` checkpoint `R4.4B` para el detalle completo, incluyendo que el umbral de salto grande de 300m del lado Passenger no se probó en señal difícil). `flutter analyze` limpio, **829/829 tests en verde** (824→829, +5 del grupo `Checkpoint R4.4B: cadencia GPS de viaje activo (3s)`). `test/r4-smooth-driver-marker` eliminada local y remotamente tras confirmar contención total.

Qué se decidió:
`_startActivityTimer()` en `driver_active_ride_screen.dart` cambia el `Timer.periodic` que publica ubicación GPS + heartbeat durante un viaje activo (`DriverActiveRideScreen`) de **10 segundos a 3 segundos**. La cadencia de `Online` sin viaje activo, en `driver_home_screen.dart` (heartbeat cada 10s), **no cambia** — es un `Timer` completamente distinto, en una pantalla distinta. El guard `_activityInFlight` ya existente (previo a este checkpoint) sigue impidiendo que un tick nuevo del `Timer` dispare una publicación GPS mientras la anterior sigue en vuelo — no fue necesario tocarlo para soportar la cadencia más rápida.

Por qué:
Contraparte en Driver de la interpolación del marcador en Passenger (ver `App-passenger/decisiones.md`, checkpoint `R4.4B`): para que la animación del marcador en el mapa del Passenger se vea fluida necesita posiciones más frecuentes de origen. Subir la cadencia de publicación GPS del Driver de 10s a 3s (alineada con la cadencia de polling de 3s que ya usa Passenger para refrescar el ride) le da a esa animación datos más frescos entre los que interpolar. Se acotó el cambio exclusivamente al viaje activo — la pantalla `Online` sin viaje (`DriverHomeScreen`) no participa de esta animación (el Passenger no ve la ubicación de un Driver todavía no asignado a su ride), así que subir también esa cadencia habría aumentado la carga de requests sin ningún beneficio visible.

Alternativas descartadas:
Subir también la cadencia de heartbeat de `Online` sin viaje (`DriverHomeScreen`, hoy 10s) — descartado porque esa ubicación no alimenta ninguna animación visible al Passenger (el conductor todavía no está asignado a un ride), así que el único efecto habría sido más carga de requests sin beneficio.

Evidencia:
`lib/features/driver/presentation/driver_active_ride_screen.dart` (`_startActivityTimer()`); `test/features/driver/presentation/driver_active_ride_screen_test.dart`, grupo `'Checkpoint R4.4B: cadencia GPS de viaje activo (3s)'` (5 casos: cadencia real de 3s, guard contra fetch GPS concurrente, recuperación tras error de GPS, cancelación del timer en `dispose`, un fallo de `complete` (409) no duplica el scheduler). Commit `2743274900ef33374a76606b62ff841d476c232f`, rama `origin/test/r4-smooth-driver-marker`.

**`PHYSICAL-REVIEW-PASS`** confirmado por JuanJo (2026-08-20, dispositivo real, campus universitario, terreno abierto y buena señal GPS): movimiento del marcador fluido y sin tirones en el caso base. Con esa aprobación, `test/r4-smooth-driver-marker` se fusionó a `main` por fast-forward puro — ver checkpoint `R4.4B` en `estado-proyecto.md` sección 16 para el detalle completo de la integración.

---

## `PAYMENT-METHOD-DRIVER-R1` — El conductor ve el método de pago referencial antes de decidir (2026-08-27)

Estado:
**FINAL-CLOSED-ON-MAIN.** Integrado a `main`@`b1644518ecde354c9855fe513755cc7a0961eb7d` mediante fast-forward puro, tras verificación en **teléfono real** de JuanJo (no solo emulador). `flutter analyze` limpio, 840/840 tests en verde. `test/payment-method-driver` eliminada local y remotamente tras confirmar contención total. Segundo tramo de la cadena de tres repos del método de pago (Backend cerró el primero en `PAYMENT-METHOD-CONTRACT-R1`); el selector real del pasajero sigue pendiente.

Qué se decidió (decisiones de producto de JuanJo):

1. **El método de pago se muestra en la parte SIEMPRE VISIBLE de la tarjeta de solicitud entrante**, sin necesidad de expandirla ni hacer scroll. Razón: el conductor tiene que verlo *antes* de aceptar o contraofertar. Si el pasajero eligió Yape y el conductor no tiene Yape, no debería tomar ese viaje — y esa decisión ocurre en la tarjeta compacta, no después de expandirla.
2. **Si Backend no envía `paymentMethod`, no se muestra nada.** NO se asume `CASH` por defecto: inventar un valor sería peor que omitirlo (el conductor tomaría una decisión con un dato falso). `paymentMethod` es `String?` con parseo defensivo; vacío/nulo → el chip devuelve `SizedBox.shrink()`.
3. **Se reutilizó el mapeo de etiquetas `paymentMethodLabel`** que ya existía en la pantalla de finalización (`driver_ride_completion_view.dart`), en vez de duplicarlo. La pantalla de finalización y la de cobro en efectivo no se tocaron — solo se importó su helper.
4. **El chip usa la familia ámbar del conductor** (`DriverPalette.orangeDeep` para texto/ícono sobre un lavado de `DriverPalette.amber` al 18 %), el mismo par que ya usa `_RideStatusHeader` — nunca el verde del pasajero. Contraste verificado en pantalla real bajo condiciones de uso.

Dónde se muestra:
- Tarjeta de solicitud entrante (`driver_home_screen.dart`, `_buildOfferCard`): en la zona compacta siempre visible, versión `dense`.
- Viaje activo (`driver_active_ride_screen.dart`): bajo `_FareCard` / "TARIFA ACORDADA" en los tres estados de la pantalla.

Omisión aceptada a propósito:
La hoja de propuestas pendientes (`_buildProposalsSheet`, "Propuesta enviada — Esperando que el pasajero elija a su conductor") NO muestra el método, aunque Backend sí lo envía en `DriverPendingProposalResponseDto`. `DriverPendingProposal` no tiene el campo ni lo parsea. En ese estado el conductor ya decidió involucrarse con el viaje; el método pesa más antes de aceptar. Ver `errores-conocidos.md` para el detalle y cómo cerrarlo si se quisiera.

Explícitamente NO tocado:
Backend, `PaymentMethod` (sigue con sus 4 valores), el flujo de `RidePayment`/cobro en efectivo, la pantalla de finalización, `DriverPendingProposal`, Passenger App, Admin Web, Railway, producción.

Evidencia:
`lib/features/driver/presentation/driver_payment_method_chip.dart` (widget nuevo), `lib/features/driver/domain/driver_ride_offer.dart` (`paymentMethod`), `lib/features/driver/domain/driver_active_ride.dart` (`paymentMethod` + `_tryParseNonEmptyString`), `lib/features/driver/presentation/driver_home_screen.dart`, `lib/features/driver/presentation/driver_active_ride_screen.dart`. Tests: `test/features/driver/presentation/driver_payment_method_chip_test.dart` (nuevo), casos añadidos en `test/driver_ride_offer_test.dart` y `test/features/driver/domain/driver_active_ride_test.dart`. Commit `b164451`, `main`@`b1644518ecde354c9855fe513755cc7a0961eb7d`.

---

## `DRIVER-COMPLETION-EXIT-R1` — Salida "Volver al inicio" en "Viaje completado" para todo caso sin cobro pendiente (2026-08-28)

Estado:
**FINAL-CLOSED-ON-MAIN.** Integrado a `main`@`7581a1af562f090038b9afdd40158adcc8d02520` mediante fast-forward puro, tras verificación en **teléfono real** de JuanJo en los cuatro casos (YAPE en vivo, efectivo pendiente sin regresión, efectivo ya pagado, restore tras cerrar/reabrir con YAPE pendiente). `flutter analyze` limpio, 846/846 tests en verde (840 baseline + 6). `test/driver-completion-exit` eliminada local y remotamente tras confirmar contención total.

Qué se decidió:
La pantalla "Viaje completado" (`driver_ride_completion_view.dart`) muestra un botón "Volver al inicio" (`context.go('/home')`, back stack limpio) siempre que no haya una acción de cobro pendiente: junto al aviso de no-efectivo cuando el método ≠ `CASH`, y solo —sin el aviso— cuando el método es `CASH` pero su estado ya no es `PENDING` (`PAID`/`FAILED`/`VOIDED`). El botón reutiliza el estilo del CTA de cierre que ya existía en `driver_cash_payment_screen.dart` (`FilledButton.icon` verde, ícono de casa, label con padding vertical 16). El caso `CASH` + `PENDING` no cambia: sigue mostrando únicamente "Cobrar efectivo". La navegación se pasa por callback (`onGoHome`), sin importar `go_router` dentro de la vista compartida — mismo patrón que ya usaba `onCollectCash`.

Por qué:
Antes, esa pantalla solo tenía una acción para el caso `CASH` + `PENDING`. En cualquier otro caso no ofrecía forma de salir y el conductor quedaba atrapado: la única salida era forzar el cierre de la app. (Reabrir sí devolvía a Home porque el restore de `driver_home_screen.dart` filtra por `selectMostRecentCashPendingPayment`, que para no-efectivo da `null`.)

Alternativas descartadas:
Cubrir solo el caso no-efectivo (YAPE/PLIN), dejando el botón dentro de la rama `else if (paymentMethod != 'CASH')`. Descartado a favor de un fallback general (Opción B): el mismo dead-end existe también para un viaje en efectivo cuyo pago ya no está `PENDING` (`PAID`/`FAILED`/`VOIDED`), un caso raro pero real; un `else` final cubre toda la clase de estados "sin acción de cobro" con un único widget, en vez de dejar un segundo hueco equivalente sin resolver.

Explícitamente NO tocado:
`driver_home_screen.dart` y su lógica de restore, el router, el caso `CASH` + `PENDING`, la pantalla de cobro en efectivo (solo se replicó su estilo de botón), Backend, Passenger App, Admin Web, Railway, producción.

Evidencia:
`lib/features/driver/presentation/driver_ride_completion_view.dart` (`onGoHome`, `_GoHomeButton`, ramas de acción reestructuradas), `lib/features/driver/presentation/driver_active_ride_screen.dart` (`_buildCompletionScreen`), `lib/features/driver/presentation/driver_completed_payment_screen.dart` (`build`). Tests: `test/features/driver/presentation/driver_active_ride_screen_test.dart` (test `J` renombrado y ampliado, más `J2`/`J3`/`J4`/`K`), `test/features/driver/presentation/driver_completed_payment_screen_test.dart` (dos casos nuevos para el restore no-efectivo). Commit `7581a1a`, `main`@`7581a1af562f090038b9afdd40158adcc8d02520`.

---

## `DRIVER-PUSH-R1` — Firebase Cloud Messaging: sin des-registro en logout, y el tap de la notificación no necesitó pantalla nueva (2026-09-01)

Estado:
**FINAL-CLOSED-ON-MAIN.** Integrado a `main`@`c11b48f4337914969cd4440c20b5c8382e05f6b9` mediante fast-forward puro (rama `test/driver-push-r1` desde `main`@`7581a1a`, un solo commit), tras verificación en **físico** (Samsung A35 5G) de JuanJo en foreground, pantalla bloqueada y app cerrada; registro de dispositivo confirmado en la base de datos de STAGING. `flutter analyze` limpio, 873/873 tests. `test/driver-push-r1` eliminada local y remotamente tras confirmar contención total. Depende de `FCM-ENABLE-R1` (FCM activado en Railway STAGING el mismo día — ver `Backend/decisiones.md`).

Qué se decidió:

1. **No hay des-registro de dispositivo en logout — y es seguro.** Confirmado por lectura directa del Backend: el upsert de `POST /me/devices` está respaldado por un índice único parcial sobre `pushToken` en las filas activas (`WHERE revoked_at IS NULL`) y revoca cualquier fila activa preexistente con el mismo `pushToken` **sin filtrar por `userId`**. Consecuencia: si el mismo dispositivo físico inicia sesión con otra cuenta, el primer `POST /me/devices` de la cuenta nueva revoca la fila del dispositivo viejo y crea la suya — nunca quedan dos filas activas apuntando al mismo token, y una cuenta no recibe push destinado a otra. Por eso `clearSession()` **no** llama a ningún endpoint de des-registro: sería un request de más, best-effort, que puede fallar en silencio, para resolver un problema que el upsert del Backend ya resuelve del lado correcto.

2. **El tap de la notificación (Etapa 3) no necesitó código.** El conductor no tiene una pantalla de detalle de una oferta individual — la lista de propuestas vive en Home y se refresca por polling de 3s. El tap de la notificación de `RIDE_OFFER_CREATED` solo tiene que traer la app al frente, que es el comportamiento por defecto de Android al tocar una notificación cuyo content-intent apunta a la `MainActivity`. Un handler de `onMessageOpenedApp` para navegar a algún lado habría sido código sin destino. Se dejó explícitamente fuera de alcance; recién tendrá sentido si en el futuro existe una pantalla de oferta individual.

3. **El aviso en foreground filtra por `data['screen']` (originalmente `data['route']`), no por `eventType`.** El Backend manda `screen: 'ride-offer'` en el `data` de `RIDE_OFFER_CREATED`; `PushMessageHandler` solo materializa un aviso local para ese valor exacto y descarta el resto con `debugPrint`. ID de notificación fijo: una propuesta nueva reemplaza el aviso anterior en vez de apilarse. (La clave era `route` en la primera versión de este checkpoint; se renombró a `screen` pocos días después — ver la entrada del fix `route`→`screen` más abajo.)

Alternativas descartadas:
- Des-registrar el dispositivo en `clearSession()` con un `DELETE` best-effort — descartada por innecesaria (ver punto 1).
- Construir el handler de tap con navegación "para dejarlo listo" — descartada: no hay pantalla a la que navegar, sería código muerto.

Explícitamente NO tocado:
El polling de 3s de la lista de propuestas (sigue siendo la fuente de verdad — la push es solo el aviso), sonido personalizado ("tuki"), ícono de notificación dedicado, fallback in-app si el permiso está denegado (los tres pospuestos a checkpoints futuros, cuando existan los archivos de audio/diseño), Backend, Passenger App, Admin Web, producción.

Evidencia:
`lib/features/notifications/data/device_id_store.dart`, `push_messaging_service.dart`, `push_registration_repository.dart`, `push_registration_coordinator.dart`, `push_message_handler.dart`, `local_notifications_service.dart`; `lib/features/auth/presentation/driver_splash_screen.dart` (disparo del registro tras confirmar sesión); `lib/main.dart` (init best-effort + canal `ride_offers`); `android/app/build.gradle.kts` (`coreLibraryDesugaring`), `android/settings.gradle.kts`, `android/app/src/main/AndroidManifest.xml` (`POST_NOTIFICATIONS`, `default_notification_channel_id`). Tests: `device_id_store_test.dart`, `push_registration_coordinator_test.dart`, `push_registration_repository_test.dart`, `push_message_handler_test.dart`, `driver_splash_screen_test.dart`. Commit `c11b48f`, `main`@`c11b48f4337914969cd4440c20b5c8382e05f6b9`.

---

## `DRIVER-PUSH-R1` (fix) — clave `route`→`screen` en el payload de notificaciones + `errorBuilder` de red de seguridad (2026-09-02)

Estado:
**FINAL-CLOSED-ON-MAIN.** Integrado a `main`@`2fdfd60717e9de996cf95f7f8df9a1104e129309` mediante fast-forward puro (rama `test/fcm-screen-key-fix` desde `c11b48f`, un solo commit `2fdfd60`). `flutter analyze` limpio, `flutter test` 875/875. Verificado en emulador con force-stop del proceso + simulación del intent real de tap de notificación. `test/fcm-screen-key-fix` eliminada local y remotamente tras confirmar contención. Parte del fix cross-repo coordinado con Backend (`main`@`48d797bb`) y `tukituki-passenger-app` (`main`@`e17d760`) — ver `historial-checkpoints.md`, entrada propia del fix.

Qué se decidió:

1. **Leer `data['screen']` en vez de `data['route']`.** La clave `route` dentro del objeto `data` de una notificación FCM colisiona con `EXTRA_INITIAL_ROUTE`, constante reservada del embedding de Flutter en Android: con la app completamente terminada, Android copia cada entrada de `data` como extra del intent de arranque, y Flutter usa el extra llamado literalmente `"route"` como `initialRoute`, salteándose la navegación normal. Como `'ride-offer'` no es una ruta declarada, la app arrancaba con `GoException: no routes for location`. El Backend renombró la clave a `screen` en todas sus notificaciones (`main`@`48d797bb`); esta app se alinea: `resolveRideUpdateKind()` y el `debugPrint` de `_handleMessage()` leen `data['screen']`, mismos valores comparados. Corte limpio, sin compatibilidad con el nombre viejo (STAGING sin usuarios reales).

2. **`errorBuilder` de red de seguridad en el `GoRouter` → `RouteNotFoundScreen`.** Independientemente del renombre, se agregó una pantalla de rescate: ante cualquier ruta que `go_router` no logre resolver (por la causa que sea, ahora o en el futuro), el conductor ve una pantalla con un botón real que navega a `/splash` (el resolver de sesión). Reemplaza al `ErrorScreen` por defecto de `go_router`, cuyo botón "Home" apunta a `/` — ruta que no existe en esta app y dejaba al usuario atrapado sin salida (fue exactamente lo que agravó el bug de arriba en las pruebas físicas).

Por qué las dos cosas juntas:
El renombre elimina la causa conocida; el `errorBuilder` es el cinturón por si aparece otra causa. Se decidió no confiar solo en el renombre: una ruta inválida que deje al usuario sin salida es un dead-end lo bastante grave como para tener una malla de contención permanente, no solo la corrección puntual.

Alternativas descartadas:
- Mantener `route` y filtrar/renombrar el extra en el lado nativo (Android) antes de que Flutter lo lea — descartada: frágil, específico de plataforma, y no ayuda a las otras claves de `data`; renombrar en el origen (Backend) es la solución correcta y única.
- Solo renombrar, sin `errorBuilder` — descartada: deja la app sin malla de contención ante cualquier ruta inválida futura de otra causa.

Explícitamente NO tocado:
El resto del flujo de push (registro, aviso en foreground), el polling, Backend salvo la coordinación del renombre, Passenger App salvo la coordinación, Admin Web, producción.

Evidencia:
`lib/features/notifications/data/push_message_handler.dart` (`resolveRideUpdateKind()`, `_handleMessage()`), `lib/core/router/app_router.dart` (`errorBuilder`), `lib/core/router/route_not_found_screen.dart` (nuevo). Tests: `push_message_handler_test.dart` (clave `route`→`screen` en todos los casos), `test/core/router/route_not_found_screen_test.dart` (nuevo — ruta inválida muestra el fallback sin crash, el botón navega a `/splash`). Commit `2fdfd60`, `main`@`2fdfd60717e9de996cf95f7f8df9a1104e129309`. Backend coordinado: `main`@`48d797bb556a63664ee2a79adb10f50839fb1688`.
