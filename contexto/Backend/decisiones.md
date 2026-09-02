# decisiones

Repositorio:
tukituki-backend

Branch analizada:
main

Commit analizado:
1e029a3ea5aded92cac21dcd4f5996e7bc167bff

Última actualización:
2026-09-02

Fuente de verdad:
Este documento es contexto auxiliar. Si contradice al código actual,
el código y los tests tienen prioridad.

---

## No usar `synchronize` de TypeORM

Estado:
ACTIVA

Qué se decidió:
El esquema de base de datos se gestiona exclusivamente con migraciones explícitas; `synchronize` queda fijado en `false` en la configuración de TypeORM.

Por qué:
Comentario explícito en el código: "Nunca dependeremos de synchronize en TukiTuki. Los cambios de base de datos se realizan mediante migraciones."

Evidencia:
`src/app.module.ts:73-80`; el README documenta `npm run migration:run` como paso obligatorio antes de arrancar.

---

## Comisión de plataforma desactivada por defecto (`COMMISSION_MODE`)

Estado:
ACTIVA (con interruptor explícito hacia otro estado)

Qué se decidió:
El sistema calcula y modela comisiones (`RideCommission`, `CommissionPolicy`) pero solo las hace exigibles cuando `COMMISSION_MODE=ENFORCED`. Por defecto es `DISABLED`. Existe una restricción de base de datos (`CHK_rides_platform_commission_rate`) que permite explícitamente una tasa de `0` además del rango normal `300-500` bps.

Por qué:
Comentario en `src/config/env.validation.ts`: "Durante la etapa demo las comisiones quedan desactivadas de forma segura. ENFORCED se habilitará únicamente cuando TukiTuki decida comenzar a cobrar comisión a los conductores."

Evidencia:
`src/config/env.validation.ts` (bloque `COMMISSION_MODE`); `src/modules/commissions/commission-runtime-mode.ts`; commit `b3990bf1` ("fix: allow disabled ride commission rate"); migración `1786312800000-AllowDisabledRideCommissionRate.ts`.

---

## Matching de conductores con radio de búsqueda creciente y continuación en el radio máximo

Estado:
ACTIVA

Qué se decidió:
El despacho automático (`RideDispatchWorker`) amplía el radio de búsqueda por rondas (`2 km → 5 km → 10 km`) y, tras alcanzar el radio máximo, sigue reintentando en ese mismo radio en vez de detenerse o inventar uno mayor.

Por qué:
Comentarios "G3B2"/"G3C-lite" en el código: separar "cuántas rondas se intentaron" del radio efectivo, y evitar que un viaje quede sin conductor tras agotar los radios definidos.

Alternativas descartadas:
Un tope de `dispatch_round` que detenía los intentos — el comentario indica explícitamente que "ya no existe" ese tope.

Evidencia:
`src/modules/rides/ride-matching.constants.ts` (`RIDE_SEARCH_RADII_METERS`, `getEffectiveSearchRadiusMeters`); commits `c1fab96b` ("expand ride matching window and search radii") y `009b50d9` ("continue ride matching at max search radius").

---

## Matching retroactivo para conductores que se conectan tarde ("late-join")

Estado:
ACTIVA

Qué se decidió:
Cuando un conductor pasa a ser "descubrible" (primera publicación de ubicación tras conectarse, o tras expirar su lease de presencia), se dispara una búsqueda de matching retroactiva para ese conductor específico, además del ciclo normal de despacho.

Por qué:
Evitar que un conductor recién disponible pierda viajes que ya estaban `SEARCHING_DRIVER` y no fueron reevaluados por él.

Evidencia:
`src/infrastructure/redis/driver-availability-redis.service.ts` (comentarios "G3A"); `src/modules/rides/ride-dispatch.worker.ts` (`runLateJoinOnce`); commit `fdee792c` ("support late-joining drivers in ride matching").

---

## Ofertas de conductor persistentes (`RideOffer`) en vez de estado efímero

Estado:
ACTIVA

Qué se decidió:
Cada oferta enviada a un conductor se persiste como fila `RideOffer` con su propio ciclo de estados (`OFFERED → PROPOSED/REJECTED/EXPIRED/CANCELLED → ACCEPTED`), desacoplado del contador de rondas del ride.

Por qué:
Comentario "G3B1" indica que la responsabilidad de expiración de la oferta (`RideOffer.expiresAt`) se separó explícitamente de la cadencia de despacho (`RIDE_DISPATCH_INTERVAL_MS`), que antes mezclaba ambas cosas.

Evidencia:
`src/modules/rides/entities/ride-offer.entity.ts`; `src/modules/rides/enums/ride-offer-status.enum.ts`; commit `a7a6a721` ("persist driver ride offers during matching").

---

## Negociación bidireccional de tarifa (pasajero propone, conductor contraoferta)

Estado:
ACTIVA

Qué se decidió:
`Ride` distingue `passengerOfferFare` (decisión del pasajero) de `estimatedFare` (recomendación del sistema) y de `agreedFare` (precio final acordado, congelado cuando el pasajero elige una propuesta). `RideOfferStatus.PROPOSED` permite que el conductor responda con el mismo precio o una contraoferta.

Por qué:
Comentarios en `Ride.entity.ts` documentan explícitamente la independencia entre `estimatedFare` y `passengerOfferFare`.

Evidencia:
`src/modules/rides/entities/ride.entity.ts:191-226`; `src/modules/rides/enums/ride-offer-status.enum.ts`; commits `6a94fde1` ("support bidirectional ride fare negotiation") y `f954e183` ("add negotiated ride pricing"); migración `1786147200000-AddRideNegotiation.ts`.

**Actualización (`BRANCH-CLEANUP-R1`, 2026-08-16) — NEGOCIACIÓN DE TARIFA: DECISIÓN CERRADA (una sola ronda)**: JuanJo confirmó explícitamente que este es el flujo definitivo de producto, no un paso intermedio hacia negociación multi-ronda. Flujo oficial: (1) Passenger propone `passengerOfferFare`; (2) Driver puede aceptar o contraofertar **una vez** (`RideOfferStatus.PROPOSED`); (3) Passenger recibe las propuestas; (4) Passenger selecciona una (`POST .../offers/:offerId/select`); (5) al seleccionar, `agreedFare` queda fijado; (6) el Driver queda asignado; (7) el Driver pasa a `BUSY` — nunca antes del paso 6. No existe ni se adoptará: contraoferta del Passenger de vuelta a un Driver específico, estado `PASSENGER_COUNTERED`, ni ninguna forma de negociación multi-ronda (ver la decisión "Segunda implementación de negociación de tarifa en rama divergente" para el commit `6fb5c56e`, explícitamente descartado por esta misma decisión). `estimatedFare` permanece interno (no se expone como "Precio recomendado TukiTuki" — confirmado sin ocurrencias del texto en ningún commit de `main` ni de la rama ya eliminada); `finalFare` respeta `agreedFare`.

---

## Notificaciones y efectos secundarios vía patrón Outbox (no broker externo)

Estado:
ACTIVA

Qué se decidió:
Los eventos de negocio (cambios de estado de ride, pagos, comisiones, incidentes de seguridad) se escriben como `OutboxEvent` en PostgreSQL y un worker (`outbox.worker.ts`) los procesa por polling, en vez de usar una cola de mensajería externa.

Por qué:
[PENDIENTE: no hay comentario ni documento explícito que declare el motivo; se infiere del código que evita una dependencia de infraestructura adicional, pero eso no está afirmado en el repositorio]

Evidencia:
`src/modules/outbox/outbox.worker.ts`, `src/modules/outbox/entities/outbox-event.entity.ts`; commit `677f5920` ("add matching workers outbox and push notifications"); commit `4e14d31e` ("fix: corrige procesamiento y cierre del outbox worker").

---

## Verificación telefónica opcional para pasajeros

Estado:
ACTIVA

Qué se decidió:
Un usuario puede operar como pasajero sin haber verificado su teléfono por OTP en ciertos flujos, mediante `isPassengerOnlyUser()`, mientras que otros roles sí exigen `isPhoneVerified`.

Evidencia:
`src/modules/auth/strategies/jwt.strategy.ts:44-47`; `src/modules/users/utils/user-auth-policy.util.ts`; commit `73bb3b12` ("make passenger phone verification optional").

---

## Endurecimiento de autenticación de administradores — ya en `main`

Estado:
RESUELTA respecto a `main` (checkpoint `RELEASE-R2`, 2026-08-16) — ver actualizaciones más abajo. Sigue pendiente solo respecto a **producción** (milestone separado).

Qué se decidió originalmente:
La rama remota `origin/carlos` contiene un commit `60cccd9d` ("feat: harden admin authentication") que agrega `admin-login-security.service.ts` y cambios en `auth.service.ts`/`auth.controller.ts` no presentes en `main` al momento de este análisis.

Por qué:
[PENDIENTE: no hay evidencia en `main` de por qué este trabajo no se fusionó]

Evidencia original:
`git log main..origin/carlos` (commit `60cccd9d`); no aparece en `git log --oneline -20` de `main`.

Actualización (ADMIN-DRIVER-R1A, commit `4a537437` sobre `test/storage-r2-railway-buckets`):
Se auditó `60cccd9d` (merge-base con `origin/carlos`: `1532fbec`; los archivos de auth afectados eran idénticos entre `HEAD` de la rama Storage y el padre de ese commit, sin divergencia real que reconciliar) y se portó su comportamiento de forma adaptada — no fusionado ni cherry-pickeado — sobre `test/storage-r2-railway-buckets`: `POST /auth/admin/login` (exclusivo ADMIN/SUPER_ADMIN, 401 genérico también para roles no administrativos), `AdminLoginSecurityService` (throttling/lockout por cuenta e IP vía Redis, `incrementWithTtl` nuevo en `RedisService`), y `PasswordService.verifyWithFallback` (comparación bcrypt ficticia contra usuario inexistente, aplicada también al `login()` genérico). Contrato de respuesta (`LoginResponseDto`) sin cambios. Suite: 79/518 → 82/542 tests, todos en verde; lint y build limpios. Publicado en `origin/test/storage-r2-railway-buckets` (commit `4a537437`).

**Actualización (`BRANCH-CLEANUP-R1`, 2026-08-16): `origin/carlos` fue eliminada del remoto** tras `BRANCH-AUDIT-R1` (auditoría) y la decisión explícita de producto de mantener negociación de una sola ronda (ver decisión siguiente) — el commit `6fb5c56e`, único otro contenido sustantivo de esa rama, quedó formalmente rechazado, no pendiente. No queda ninguna funcionalidad aprobada sin reconciliar en esa rama.

Actualización (ADMIN-DRIVER-R1A.1, mismo commit `4a537437`, sin cambio de código): validado en vivo contra Railway staging con una cuenta `SUPER_ADMIN` real — `POST /auth/admin/login` 200 con rol `SUPER_ADMIN`, `GET /admin/drivers` 200, `GET /storage/admin/documents/:id/download-url` sobre un `DriverDocument` sintético de STORAGE-R3 (`SOAT`) 200 con `isLegacyUrl: false`, descarga real de la URL presignada (`application/pdf`), `POST /auth/refresh` 200, `POST /auth/logout` 204, y confirmación de que el access token queda invalidado de inmediato tras el logout (401 en la siguiente llamada, no solo por expiración del JWT). Cierra el `TEST CREDENTIAL REQUIRED` de esta decisión para el ambiente **staging**.

Actualización (`RELEASE-R2`, 2026-08-16): `test/storage-r2-railway-buckets` se integró a `main` por fast-forward (`main`@`4a537437d8946c02c3e7c3855a6773e13b97c725`). **`main` ya tiene este endpoint.** `main` no es producción — Railway/STAGING solo pasa a desplegar desde `main` cuando JuanJo lo cambie manualmente; producción sigue sin este endpoint hasta un milestone separado y autorizado explícitamente.

---

## Segunda implementación de negociación de tarifa en rama divergente — DECISIÓN CERRADA, NO PORTAR

Estado:
REJECTED-BY-PRODUCT-DECISION (cerrada en `BRANCH-CLEANUP-R1`, 2026-08-16)

Qué se decidió originalmente (rama, ya eliminada):
La rama `origin/carlos` (Backend) contenía un commit `6fb5c56e` ("feat: complete passenger fare negotiation flow") con su propia migración `1786233600000-AddPassengerRideCounterOffers.ts` y cambios en `ride-offer.entity.ts`, `passenger-rides.service.ts`, etc. Agregaba un estado `PASSENGER_COUNTERED` y columnas `ride_offers.passenger_proposed_fare`/`passenger_proposed_at`, habilitando que el Passenger contraofertara de vuelta a un Driver específico tras recibir su propuesta — negociación **multi-ronda**. `main` implementó negociación bidireccional de forma independiente y posterior (`6a94fde1`, `f954e183`, migración `1786147200000-AddRideNegotiation.ts`), pero de una sola ronda: el Driver responde una vez (`accept`/`counter-offer`) y el Passenger solo puede `select`, sin contraofertar de vuelta.

`BRANCH-AUDIT-R1` (2026-08-16) comparó ambas implementaciones función por función y confirmó que **no son redundantes**: `6fb5c56e` añadía una capacidad de producto real que `main` nunca implementó, no un duplicado inferior. Cherry-pickearlo tal cual habría sido inseguro (DTOs/contrato incompatibles con lo que ambas apps móviles consumen hoy).

Decisión de producto (JuanJo, `BRANCH-CLEANUP-R1`, 2026-08-16):
**Opción A — mantener negociación de una sola ronda**, la ya implementada en `main`. Explícitamente **NO** adoptar: estado `PASSENGER_COUNTERED`, columna `passenger_proposed_fare` por oferta, contraoferta del Passenger de vuelta al Driver, ni negociación multi-ronda en ninguna forma.

Por qué:
Decisión de producto explícita de JuanJo — no es un rechazo técnico ni un error del commit; es una elección de diseño de UX/negocio (mantener la negociación simple: Passenger propone, Driver responde una vez, Passenger elige).

Consecuencia:
`6fb5c56e` queda marcado **NO PORTAR** — no debe cherry-pickearse, fusionarse ni reimplementarse salvo que una decisión de producto futura y explícita revierta esto. La rama `origin/carlos` que lo contenía fue eliminada (`BRANCH-CLEANUP-R1`); este documento es la referencia histórica de qué existía y por qué se descartó, para que no vuelva a evaluarse como "pendiente de rescatar" por accidente.

Evidencia:
`git show 6fb5c56e` (commit ya no alcanzable desde ninguna rama remota tras la eliminación, pero recuperable por hash mientras no se ejecute garbage collection agresivo); comparación función por función en el reporte de `BRANCH-AUDIT-R1`; estado actual de `main` sin `PASSENGER_COUNTERED` (`src/modules/rides/enums/ride-offer-status.enum.ts`, 6 valores) ni columnas `passenger_proposed_*` (`src/database/migrations/1786147200000-AddRideNegotiation.ts`).

---

## Almacenamiento de archivos: URL estable del Backend en vez de exponer URLs de S3 directamente

Estado:
LEGACY (diseño original de STORAGE-R2; el mecanismo de autorización fue corregido por STORAGE-R2.1 — ver la decisión siguiente. La idea de "URL estable del Backend en vez de URL de S3 directa" se mantiene, pero **ya no** se persiste como valor estático en `photoUrl` ni es resoluble solo con el `profileId`)

Qué se decidió (diseño original R2, ya corregido):
Al completar una subida a Railway Storage Buckets, el Backend escribía en el campo `photoUrl` existente una URL estable propia (`/storage/avatars/driver|passenger/:id`) que redirigía (302) a una presigned GET recién generada, resoluble por cualquiera que conociera el `profileId`. Los documentos de conductor (licencia/SOAT/TIV) siguen sin cambios: privados, con un endpoint dedicado que devuelve `{ downloadUrl, expiresAt }` bajo demanda.

Por qué:
Evitar persistir una presigned URL como dato canónico (caduca) y evitar tocar los ~10 puntos del código que ya leen `driverProfile.photoUrl`/`passengerProfile.photoUrl` directamente.

Por qué se corrigió:
Revisión de producto (STORAGE-R2.1): un `profileId` conocido/adivinado como único requisito de acceso es "security by obscurity", explícitamente rechazado. Ver la decisión "Avatares por capability token" a continuación.

Evidencia:
`src/modules/storage/avatar-url.util.ts`, `src/modules/drivers/drivers.service.ts`/`src/modules/passengers/passengers.service.ts` (`completeProfilePhotoUpload`) — estado tal como quedó tras el commit `050e5657`, ya con la corrección aplicada.

---

## Avatares por capability token firmado, minteado solo en contextos ya autorizados (STORAGE-R2.1)

Estado:
ACTIVA (implementada en commit `050e5657`; **ya en `main` desde `RELEASE-R2`**, `main`@`4a537437`)

Qué se decidió:
`GET /storage/avatars/driver/:id` y `GET /storage/avatars/passenger/:id` exigen un query param `?token=...`: un HMAC-SHA256 firmado con `STORAGE_AVATAR_TOKEN_SECRET`, con `kind`+`profileId`+`expiresAt` codificados en el payload firmado, verificado con `timingSafeEqual`. El `profileId` de la ruta ya no es suficiente por sí solo. El token se minta (`AvatarUrlResolverService`) únicamente dentro de código que ya validó una relación real: perfil propio (`toProfileResponse`), ride asignado (`RideViewService`), ofertas pendientes (`PassengerRidesService`), historial propio (`RideHistoryService`), snapshot de un incidente de seguridad (`RideSafetyService`), share-link de viaje válido (`RideShareLinksService`), o vista de administrador (`AdminRidesService`/`AdminDriversService`). `photoUrl` dejó de escribirse como valor estático en la base de datos en el momento del upload — se resuelve de nuevo en cada lectura, con un token fresco cada vez.

Por qué:
Decisión de producto explícita ("POLÍTICA A — FOTOS CON ACCESO CONTROLADO"): ni la foto del Passenger ni la del Driver deben quedar accesibles con solo conocer/adivinar un `profileId`. Un share-link de viaje válido sigue pudiendo mostrar la foto del Driver, pero como excepción acotada al contexto de ese share, no como apertura de un endpoint público general.

Alternativas descartadas:
Endpoint autenticado con `JwtAuthGuard` — descartado tras auditar ambas apps Flutter: ninguna adjunta el header `Authorization` a `Image.network`, no existe `cached_network_image` ni ningún precedente de carga de imagen autenticada vía Dio en ninguno de los dos repos (`tukituki-driver-app`, `tukituki-passenger-app`); exigir JWT habría roto la carga de avatares sin cambios en Flutter. Tabla de "upload intent"/capability persistida en DB — descartada por sobrediseño, igual que en la decisión de ownership de `objectKey` de STORAGE-R2.

Evidencia:
`src/modules/storage/avatar-url.util.ts` (`mintAvatarToken`, `verifyAvatarToken`, `resolveAvatarUrl`), `src/modules/storage/avatar-url-resolver.service.ts`, `src/modules/storage/storage.controller.ts` (`?token=` en ambas rutas de avatar), `src/modules/storage/storage.service.ts` (`assertAvatarToken`); consumidores: `src/modules/drivers/drivers.service.ts`/`src/modules/passengers/passengers.service.ts` (`toProfileResponse`), `src/modules/rides/ride-view.service.ts`, `src/modules/rides/passenger-rides.service.ts`, `src/modules/rides/ride-history.service.ts`, `src/modules/safety/ride-safety.service.ts`, `src/modules/safety/ride-share-links.service.ts`, `src/modules/admin-rides/admin-rides.service.ts`, `src/modules/admin-drivers/admin-drivers.service.ts`.

---

## DriverProfile.address deja de ser obligatoria en el onboarding (DRIVER-ONBOARDING-R2)

Estado:
ACTIVA (implementada en `main`@`ede503bc` desde DRIVER-ONBOARDING-R2.3, fast-forward de `test/driver-onboarding-r2-backend`)

Qué se decidió:
`DriverProfile.address` se conserva en el modelo con todos sus datos existentes, pero pasa de `NOT NULL`/obligatoria a nullable/opcional a nivel de columna DB y de `CreateDriverProfileDto`/`UpdateDriverProfileDto`. El paso 2 ("Sobre ti") del onboarding nuevo ya no la pide. Admin puede seguir mostrándola cuando exista.

Por qué:
Decisión de producto explícita de JuanJo, respondiendo al `BUSINESS-DECISION-REQUIRED` dejado abierto por `DRIVER-ONBOARDING-R1` (auditoría integral pre-implementación): **Opción A — no pedir dirección domiciliaria durante el onboarding**.

Alternativas descartadas:
Opción B (mantenerla obligatoria) — no elegida.

Evidencia:
`src/modules/drivers/entities/driver-profile.entity.ts` (`address!: string | null`), `src/modules/drivers/dto/create-driver-profile.dto.ts` (`address?: string`), migración `1786950000000-PrepareDriverOnboardingR2.ts`.

---

## DriverProfile.email opcional, sin unicidad (DRIVER-ONBOARDING-R2)

Estado:
ACTIVA (implementada en `main`@`ede503bc` desde DRIVER-ONBOARDING-R2.3, fast-forward de `test/driver-onboarding-r2-backend`)

Qué se decidió:
Se agrega `DriverProfile.email` (nullable, `varchar(255)`, validado con `@IsEmail` solo cuando viene informado). No se exige, no se mueve a `User`, no se agrega verificación por OTP ni restricción `UNIQUE`.

Por qué:
El paso 2 ("Sobre ti") del onboarding aprobado incluye "correo electrónico opcional". No existe ninguna decisión de producto que exija unicidad de email para conductores — inventarla habría sido alcance no autorizado.

Evidencia:
`src/modules/drivers/entities/driver-profile.entity.ts`, `src/modules/drivers/dto/create-driver-profile.dto.ts`, migración `1786950000000-PrepareDriverOnboardingR2.ts`.

---

## VehicleOwnership (OWNED/RENTED): nullable en DB, requerido en el DTO de creación (DRIVER-ONBOARDING-R2)

Estado:
ACTIVA (implementada en `main`@`ede503bc` desde DRIVER-ONBOARDING-R2.3, fast-forward de `test/driver-onboarding-r2-backend`)

Qué se decidió:
`DriverVehicle.ownership` (enum nuevo `VehicleOwnership`: `OWNED`/`RENTED`) queda nullable a nivel de columna DB, pero `CreateDriverVehicleDto` lo exige (`@IsEnum`, sin `@IsOptional`) para todo vehículo nuevo. Los vehículos ya existentes en STAGING quedan con `ownership: null` — no se les asigna `OWNED` ni ningún otro valor por defecto sin evidencia real de cuál es.

Por qué:
El paso 3 ("Mototaxi") del onboarding aprobado pide "propio/alquilado" como campo nuevo. STAGING ya tiene vehículos creados antes de esta decisión, sin este dato — inventar un valor (p. ej. asumir `OWNED` para todos) habría sido falsificar datos de negocio sobre expedientes reales de conductores.

Alternativas descartadas:
Backfill automático a `OWNED` para registros existentes — descartado explícitamente por el prompt de este checkpoint ("no inventar default OWNED para registros existentes si no hay evidencia").

Evidencia:
`src/modules/drivers/enums/vehicle-ownership.enum.ts`, `src/modules/drivers/entities/driver-vehicle.entity.ts` (`ownership!: VehicleOwnership | null`), `src/modules/drivers/dto/create-driver-vehicle.dto.ts` (`ownership!: VehicleOwnership`, sin `@IsOptional`), migración `1786950000000-PrepareDriverOnboardingR2.ts`.

---

## Expediente de documentos reducido de 6 a 3 + foto de perfil obligatoria antes de submit (DRIVER-ONBOARDING-R2)

Estado:
ACTIVA (implementada en `main`@`ede503bc` desde DRIVER-ONBOARDING-R2.3, fast-forward de `test/driver-onboarding-r2-backend`)

Qué se decidió:
`DriverApplicationSubmissionService` y `AdminDriverReviewService` ya no exigen 6 tipos de documento (`DNI_FRONT`, `DNI_BACK`, `DRIVER_LICENSE`, `VEHICLE_REGISTRATION`, `SOAT`, `PROFILE_PHOTO`), sino exactamente 3: `DRIVER_LICENSE`, `SOAT`, `VEHICLE_REGISTRATION`, mediante una única constante de dominio compartida (`REQUIRED_DRIVER_APPLICATION_DOCUMENT_TYPES`, `src/modules/drivers/driver-application.constants.ts`) para que ambos servicios no puedan volver a divergir. Los valores de enum legacy (`DNI_FRONT`/`DNI_BACK`/`PROFILE_PHOTO`) se conservan sin eliminar (compatibilidad con registros históricos), pero dejan de ser obligatorios. Además, `POST /drivers/me/submit` ahora exige que `DriverProfile` tenga foto (`photoObjectKey` o `photoUrl` legacy) — la foto de perfil pasa a ser un atributo del perfil, nunca un `DriverDocument` de tipo `PROFILE_PHOTO`. `AdminDriverReviewService.approve` revalida la misma condición como última barrera.

Por qué:
Decisión de producto aprobada (ya registrada en `estado-proyecto.md` antes de este checkpoint): el expediente objetivo del conductor son 3 documentos (licencia, SOAT, TIV), y la foto de perfil es una imagen de perfil separada, no un documento del expediente.

Evidencia:
`src/modules/drivers/driver-application.constants.ts`, `src/modules/drivers/driver-application-submission.service.ts` (`getMissingRequirements`), `src/modules/admin-drivers/admin-driver-review.service.ts` (`assertApprovalRequirements`), `src/modules/admin-drivers/dto/reject-driver-application.dto.ts` (`@ArrayMaxSize(3)`).

---

## TukiTuki dueño del ciclo OTP; proveedor SMS futuro limitado a transporte (OTP-AUDIT-R1 / OTP-R2)

Estado:
ACTIVA (auditoría en `OTP-AUDIT-R1`, endurecimiento **ya en `main`@`f147a664`** desde `OTP-R2.3`, fast-forward de `test/otp-r2-hardening`)

Qué se decidió:
El Backend de TukiTuki es dueño completo de la lógica OTP: genera el código (`crypto.randomInt`, 6 dígitos), lo hashea (HMAC-SHA256), lo almacena en Redis con TTL, controla intentos y cooldown, lo invalida y actualiza `User.isPhoneVerified`. Un proveedor SMS externo futuro (`OTP-R3`) estará limitado a transportar el mensaje ya generado — nunca será dueño de la lógica de verificación ni un "managed OTP service" (se descartó explícitamente ese enfoque, que hubiera duplicado lo que `OtpService` ya hace).

Por qué:
Decisión de producto de JuanJo, confirmada por `OTP-AUDIT-R1`: el núcleo OTP ya existente en el Backend (`src/modules/auth/otp.service.ts`) es técnicamente suficiente y no depende de ningún SDK/servicio externo para su lógica — solo le falta el transporte del SMS.

Evidencia:
`src/modules/auth/otp.service.ts`; ausencia total de SDKs de SMS/OTP administrado en `package.json` (confirmado por `OTP-AUDIT-R1`).

---

## Guard estructural: `OTP_DEBUG_ENABLED` estructuralmente imposible en producción (OTP-R2)

Estado:
ACTIVA (**ya en `main`@`f147a664`** desde `OTP-R2.3`, fast-forward de `test/otp-r2-hardening`)

Qué se decidió:
El schema Joi de `src/config/env.validation.ts` rechaza el arranque del Backend si `NODE_ENV=production` y `OTP_DEBUG_ENABLED=true` (`Joi.when('NODE_ENV', { is: 'production', then: Joi.boolean().valid(false)... })`). Fuera de producción (`development`/`test`) el comportamiento no cambia: `OTP_DEBUG_ENABLED=true` sigue permitiendo que `POST /auth/otp/request` devuelva `debugOtp` en la respuesta.

Por qué:
`OTP-AUDIT-R1` identificó que la única protección existente contra filtrar el código OTP en la respuesta en producción era disciplina humana (nadie debía dejar la variable en `true` en Railway) — sin ningún guard de arranque que lo impidiera, a diferencia de otros flags condicionales del proyecto (Izipay, FCM, Storage) que sí fallan el arranque si están mal configurados.

Alternativas descartadas:
Dejarlo como responsabilidad operativa de Railway/JuanJo sin cambio de código — rechazada explícitamente por el prompt de `OTP-R2` ("No depender de disciplina humana").

Evidencia:
`src/config/env.validation.ts` (bloque `OTP_DEBUG_ENABLED`); `src/config/env.validation.spec.ts`.

---

## Enumeración de teléfonos cerrada en `POST /auth/otp/request` (OTP-R2)

Estado:
ACTIVA (**ya en `main`@`f147a664`** desde `OTP-R2.3`, fast-forward de `test/otp-r2-hardening`)

Qué se decidió:
Un teléfono sin cuenta registrada recibe exactamente la misma respuesta 200 (mismo shape `{ expiresIn }`, mismo cooldown aplicado) que un envío real a un teléfono existente. Ya no existe el `404 NotFoundException` distintivo que antes revelaba si un número estaba o no registrado en TukiTuki. Internamente no se genera, hashea ni almacena ningún OTP para un teléfono sin cuenta.

Por qué:
`OTP-AUDIT-R1` marcó el 404 explícito como un vector real de enumeración de cuentas. El prompt de `OTP-R2` autorizó explícitamente preferir una respuesta uniforme si no había ningún consumidor dependiente del contrato anterior: se verificó que la Passenger App tiene su flujo OTP desconectado ("código huérfano", ver `estado-proyecto.md` sección 7) y que la Driver App todavía no integra OTP en absoluto — ningún consumidor real depende del 404.

Alternativas descartadas:
Mantener el 404 (rechazada por el riesgo de enumeración ya documentado); requerir una decisión de producto explícita nueva antes de tocarlo — no fue necesario porque el propio Backend ya tiene precedente idéntico en `AuthService.login` (mensaje genérico ante usuario inexistente o contraseña incorrecta, vía `PasswordService.verifyWithFallback`).

Evidencia:
`src/modules/auth/otp.service.ts` (`requestPhoneVerification`); `src/modules/auth/otp.service.spec.ts`.

---

## Cooldown preservado al agotar los intentos de verificación OTP (OTP-R2)

Estado:
ACTIVA (**ya en `main`@`f147a664`** desde `OTP-R2.3`, fast-forward de `test/otp-r2-hardening`)

Qué se decidió:
Al superar `OTP_MAX_ATTEMPTS` en `POST /auth/otp/verify`, el código y el contador de intentos se invalidan como antes, pero ahora se aplica un cooldown fresco y completo (`OTP_RESEND_COOLDOWN_SECONDS`) en vez de borrarlo. `OtpService.clearOtp` acepta un parámetro `{ preserveCooldown: true }` para este caso.

Por qué:
`OTP-AUDIT-R1` detectó que `clearOtp()` borraba las tres keys (código, intentos, cooldown) también al agotar intentos, permitiendo un loop instantáneo: agotar los 5 intentos y pedir un OTP nuevo de inmediato, sin ninguna espera real.

Evidencia:
`src/modules/auth/otp.service.ts` (`clearOtp`, `verifyPhone`); `src/modules/auth/otp.service.spec.ts`.

---

## Rate limit dedicado a `POST /auth/otp/request` por IP y por teléfono (OTP-R2)

Estado:
ACTIVA (**ya en `main`@`f147a664`** desde `OTP-R2.3`, fast-forward de `test/otp-r2-hardening`)

Qué se decidió:
Además del throttle global (`RATE_LIMIT_*`, compartido por toda la API), `POST /auth/otp/request` aplica dos límites dedicados usando `RedisService.incrementWithTtl` (atómico): `OTP_REQUEST_IP_LIMIT`/`OTP_REQUEST_IP_WINDOW_SECONDS` (default 20 solicitudes/hora por IP) y `OTP_REQUEST_PHONE_LIMIT`/`OTP_REQUEST_PHONE_WINDOW_SECONDS` (default 5 solicitudes/hora por teléfono). Los contadores se identifican por huella HMAC-SHA256 (con `OTP_HASH_SECRET`), nunca por el teléfono/IP en claro como key de Redis — mismo patrón que `AdminLoginSecurityService`.

Por qué:
`OTP-AUDIT-R1` identificó que el único freno de costo existente era el cooldown de 60s por teléfono, insuficiente como protección de ventana larga (permitía hasta 60 SMS/hora/teléfono de forma indefinida una vez hubiera un proveedor real conectado). Los defaults (20/hora por IP, 5/hora por teléfono) buscan ser conservadores sin bloquear uso normal: un usuario legítimo no necesita más de un puñado de códigos por hora.

Evidencia:
`src/modules/auth/otp.service.ts` (`enforceIpRequestLimit`, `enforcePhoneWindowRequestLimit`, `fingerprint`); `src/config/env.validation.ts`; `.env.example`.

---

## OTP-DEMO-R1: mecanismo temporal de demo OTP en Railway STAGING, separado de `OTP_DEBUG_ENABLED`

Estado:
TEMPORAL — IMPLEMENTED ON TEST (`test/otp-demo-staging`, commit `fd9ba8b3c4a25db23c339b1c5dffba1b05e1718e`, creada desde `main`@`f147a664`), **no fusionada a `main`, no desplegada en Railway**. No confundir con arquitectura final: se retira cuando `OTP-R3` (proveedor SMS real) esté disponible.

Qué se decidió:
Para poder probar físicamente el flujo real de verificación telefónica (Paso 1 del onboarding de Driver) en Railway STAGING mientras no exista proveedor SMS, se agregó un mecanismo separado de `OTP_DEBUG_ENABLED`: `OTP_DEMO_ENABLED` (default `false`) + `OTP_DEMO_ALLOWED_PHONE_E164` (un único teléfono QA). `POST /auth/otp/request` solo incluye `debugOtp` cuando las tres condiciones se cumplen a la vez: (1) `OTP_DEMO_ENABLED=true`; (2) `RAILWAY_ENVIRONMENT_NAME` (inyectada por Railway, nunca `NODE_ENV`) es exactamente `staging`; (3) el teléfono solicitado es exactamente el de la allowlist. El schema Joi de `env.validation.ts` hace estructuralmente imposible activar el flag fuera de `staging` (falla el arranque, no solo la respuesta HTTP). `debugOtp` sigue siendo el OTP real generado por `OtpService` (mismo `crypto.randomInt`, mismo hash HMAC-SHA256 en Redis, mismo TTL/intentos/cooldown/rate-limit de `OTP-R2`, mismo `POST /auth/otp/verify`) — no hay bypass, segundo código ni OTP hardcodeado.

Por qué:
`DEMO-PRIORITY-DECISION-R1` (sección 12/16 de `estado-proyecto.md`) aprobó explícitamente usar "cuentas sintéticas QA previamente verificadas" para demos previas al proveedor SMS, prohibiendo a la vez cualquier bypass de OTP de producción, OTP hardcodeado, validación local en Flutter, secreto embebido en el APK, o `OTP_DEBUG_ENABLED` habilitado en un ambiente de tipo producción. `OTP_DEBUG_ENABLED` no podía reutilizarse para esto porque Railway STAGING corre con `NODE_ENV=production` (el guard de `OTP-R2` lo bloquea ahí intencionalmente) y porque mezclar ambos flags habría debilitado el guard de producción existente.

Alternativas descartadas:
Reutilizar/relajar `OTP_DEBUG_ENABLED` en STAGING — rechazado explícitamente (debilitaría el guard de `OTP-R2`, que debe seguir bloqueando cualquier NODE_ENV=production real). Confiar en `NODE_ENV` para autorizar el modo demo — rechazado porque STAGING usa deliberadamente `NODE_ENV=production`; solo `RAILWAY_ENVIRONMENT_NAME` distingue el environment real de Railway. Allowlist por prefijo/wildcard de teléfonos — rechazado, solo un teléfono QA exacto.

Evidencia:
`src/config/env.validation.ts` (bloques `RAILWAY_ENVIRONMENT_NAME`/`OTP_DEMO_ENABLED`/`OTP_DEMO_ALLOWED_PHONE_E164`), `src/config/env.validation.spec.ts`, `src/modules/auth/otp.service.ts` (`isDemoOtpAllowed`), `src/modules/auth/otp.service.spec.ts`, `.env.example`; `docs/contexto/estado-proyecto.md` secciones 12, 16, 17 (`OTP-DEMO-R1`).

---

## Ownership de objectKey sin tabla de "upload intent": prefijo server-side + HeadObject

Estado:
ACTIVA (**ya en `main` desde `RELEASE-R2`**, `main`@`4a537437`)

Qué se decidió:
Para verificar que un `objectKey` recibido en `POST /storage/uploads/complete` pertenece al usuario autenticado, el Backend reconstruye el prefijo esperado (`passengers/{userId}/...` o `drivers/{driverProfileId}/...`) a partir de las reglas de negocio existentes (perfil propio, documento propio) y comprueba que el `objectKey` caiga dentro de ese prefijo con un sufijo `<uuid>.<extensión>` — sin persistir una tabla intermedia de "intención de subida".

Por qué:
El propio flujo ya es seguro sin esa tabla: el Backend nunca entrega credenciales del bucket al cliente, y solo puede existir un objeto en esa ruta exacta si alguien obtuvo antes un PUT presignado para esa key específica (emitido únicamente por este Backend al owner legítimo). Agregar una tabla de intención habría sido complejidad adicional sin una garantía de seguridad extra medible.

Alternativas descartadas:
Persistir un registro de "upload intent" (objectKey, owner, expiración) en una tabla nueva y validar contra esa tabla en `complete` — evaluado y descartado explícitamente por sobrediseño para las garantías que ya ofrece prefijo+UUID+HeadObject.

Evidencia:
`src/modules/storage/storage-object-key.util.ts` (`isObjectKeyWithinPrefix`), `src/modules/storage/storage.service.ts` (`resolveOwnerPrefix`, `completeUpload`).

---

## DRIVER MVP PHONE POLICY: `isPhoneVerified` deja de gatear cuentas de consumidor durante MVP; ADMIN/SUPER_ADMIN siguen exigiéndolo (R4.3-PRE2/PRE2B/PRE3)

Estado:
**FINAL-CLOSED-ON-MAIN.** Commit `29fe187aa31f5aad2db20ecd49534a218b358ab6` ("fix: align driver phone verification with MVP policy"), integrado a `main`@`29fe187aa31f5aad2db20ecd49534a218b358ab6` mediante fast-forward puro (`CROSS-APP-R4.3` cierre, 2026-08-18) y confirmado por el smoke físico final de JuanJo sobre APKs Passenger+Driver construidas desde `main` contra Backend STAGING desde `main` (7/7 PASS): una cuenta Driver `ACTIVE`+`isPhoneVerified=false` operó de punta a punta. 629/629 tests, lint limpio. `test/r4-ride-identities` eliminada (local y remota) tras confirmar contención total; `test/otp-demo-staging` permanece intacta, sin tocar.

Qué se decidió:
Mientras la verificación real de teléfono por SMS sigue diferida para MVP, `isPhoneVerified=false` deja de bloquear cuentas PASSENGER/DRIVER (onboarding, submit de solicitud Driver, aprobación admin, login genérico, JWT de request, refresh de sesión) — solo `User.status===ACTIVE` gatea su operación. Las cuentas con rol ADMIN o SUPER_ADMIN (incluso combinado con PASSENGER/DRIVER) mantienen la exigencia `isPhoneVerified===true` en esos mismos gates — el rol administrativo tiene precedencia de seguridad sobre la relajación MVP. Política centralizada en `isUserOperationallyEnabled(user)` (`src/modules/users/utils/user-auth-policy.util.ts`), reutilizada por `AdminDriverReviewService.assertUserCanBecomeDriver`, `JwtStrategy.validate`, `AuthService.login` y `AuthSessionsService.rotate`. `AuthService.loginAdmin` (endpoint dedicado del panel admin) no se modificó — ya exigía `ACTIVE && isPhoneVerified` de forma independiente.

Por qué:
`CROSS-APP-R4.3C-PRE1` detectó que un Driver recién aprobado por Admin quedaba igualmente bloqueado al autenticarse, porque `JwtStrategy` aplicaba la misma exigencia de teléfono verificado que el gate de aprobación — contradiciendo la decisión de producto de mantener el registro/onboarding de Driver sin OTP real durante MVP. La primera corrección (`PRE2`) relajó `isPhoneVerified` para todos los roles por igual, lo cual habría debilitado accidentalmente la seguridad de acceso de ADMIN/SUPER_ADMIN en los gates genéricos (login, JWT, refresh), dependiendo solo del endpoint `loginAdmin` para esa garantía. `PRE2B` corrigió esto acotando la relajación exclusivamente a cuentas sin rol administrativo.

Evidencia:
`src/modules/users/utils/user-auth-policy.util.ts`, `src/modules/admin-drivers/admin-driver-review.service.ts`, `src/modules/auth/strategies/jwt.strategy.ts`, `src/modules/auth/auth.service.ts`, `src/modules/auth-sessions/auth-sessions.service.ts` y sus specs correspondientes. 629/629 tests PASS, lint limpio.

---

## `G4B-CONTRACT-R1`: bandera explícita `isManualSelection` reemplaza la detección por texto exacto en el contrato de estimación de tarifa

Estado:
**FINAL-CLOSED-ON-MAIN.** Commit `aec51b42083eb7ce5ea65011b9019f92b30be09b` ("feat: add explicit manual-selection flag to fare estimate contract"), integrado a `main`@`cc4888af` mediante fast-forward puro (2026-08-24), seguido de un commit de formato (`cc4888af`, línea de test que excedía el ancho de Prettier — detectado por `lint:check` sobre `main` antes de publicar, corregido con `npm run lint --fix`). 89/89 test suites, 631/631 tests, lint limpio. `test/g4b-contract-r1` eliminada (local y remota) tras confirmar contención total en `main`.

Qué se decidió:
`FareEstimateLocationDto` gana un campo opcional `destination.isManualSelection: boolean` (`IsOptional`, `IsBoolean`). En `FaresService.estimate`, la señal para decidir si `destination` debe resolverse por reverse geocoding pasa a ser `isManualSelection === true`, en vez de comparar `destination.address` contra el literal hardcodeado `'Destino seleccionado en el mapa'` (`MANUAL_DESTINATION_PLACEHOLDER`). `origin` no se ve afectado — siempre se resuelve por reverse geocoding, sin condición, igual que antes.

Fallback legado (temporal, con condición de retiro explícita):
Cuando `isManualSelection` viene **ausente** (cliente pre-contrato), `FaresService` cae en el comportamiento anterior: compara `address` contra `MANUAL_DESTINATION_PLACEHOLDER`. Un cliente que mande `isManualSelection: false` de forma explícita **nunca** cae en este fallback, aunque su `address` coincida por casualidad con el literal legado — la bandera explícita siempre gana sobre el texto cuando está presente.

**Condición de retiro del fallback** (no depende de fecha): retirar `MANUAL_DESTINATION_PLACEHOLDER`, su uso en `requiresDestinationGeocoding` y el test que lo cubre en `fares.service.spec.ts` en cuanto se confirme que la única instalación pre-contrato (un APK de prueba en el celular físico de JuanJo, sin distribución pública — ver la decisión equivalente en `docs/contexto/App-passenger/decisiones.md` y `docs/contexto/App-passenger/errores-conocidos.md`) fue reinstalada con una build que ya envía `isManualSelection`.

Por qué:
La detección anterior por comparación de texto exacto contra un literal de copy de UI (`home_screen.dart` del lado Passenger) quedaba desincronizada en silencio si la app cambiaba ese copy sin tocar el Backend — un acoplamiento implícito entre dos repos sin contrato explícito que lo garantizara. `G4B-CONTRACT-R1` reemplaza esa heurística de texto por un campo de contrato explícito y tipado.

Alternativas descartadas:
Exigir `isManualSelection` como campo obligatorio desde el primer commit — descartado porque habría roto la única instalación existente sin el campo (el APK de prueba mencionado arriba) hasta que se reinstalara; se prefirió opcional con fallback temporal y acotado en vez de forzar un reinstall inmediato.

Evidencia:
`src/modules/fares/dto/fare-estimate-location.dto.ts`, `src/modules/fares/fares.service.ts` (`requiresDestinationGeocoding`, comentario "LEGACY FALLBACK (G4B-CONTRACT-R1)"), `src/modules/fares/fares.service.spec.ts`. Commit `aec51b42`.

---

## `ORIGIN-ADDRESS-R1`: endpoint liviano de reverse geocoding para la dirección de origen, con límite dedicado por usuario

Estado:
**FINAL-CLOSED-ON-MAIN.** Commit `06cad3f3` ("feat: add lightweight reverse geocoding endpoint for the origin address"), integrado a `main`@`06cad3f3` mediante fast-forward puro (2026-08-24). 90/90 test suites, 637/637 tests, lint limpio. `test/origin-address-r1` eliminada (local y remota) tras confirmar contención total en `main`.

Qué se decidió:
Nuevo `GET /fares/origin-address?latitude=..&longitude=..` en `FaresModule` (mismos guards de clase que el resto de `FaresController`: `JwtAuthGuard` + `RolesGuard(PASSENGER)`). Resuelve la dirección real de un punto GPS **sin generar un `FareQuote`** — reutiliza `GoogleGeocodingService.reverseGeocode(..., FALLBACK_ORIGIN_ADDRESS)`, el mismo servicio y el mismo fallback honesto que ya usa `origin` en cada cotización (`FaresService.estimate`). Existe para que Passenger pueda mostrar la dirección real del origen apenas obtiene el GPS, sin esperar a que el pasajero elija destino (ver la decisión equivalente en `docs/contexto/App-passenger/decisiones.md`).

Límite dedicado por usuario (protección de costo):
`OriginAddressService.resolve()` aplica un límite vía `RedisService.incrementWithTtl`, mismo patrón atómico que ya usa `OTP_REQUEST_IP_LIMIT`/`OTP_REQUEST_PHONE_LIMIT` (`otp.service.ts`), pero identificando la key directamente por `userId` (`fares:origin-address:user:${userId}`) — sin fingerprint HMAC, porque a diferencia del teléfono/IP, `userId` ya es un UUID interno opaco emitido por este mismo Backend, no un dato de contacto real. Env vars nuevas: `ORIGIN_ADDRESS_RATE_LIMIT_MAX` (default `30`) y `ORIGIN_ADDRESS_RATE_LIMIT_WINDOW_SECONDS` (default `3600`). Al superarse, `429`.

Por qué un límite dedicado cuando `places/autocomplete` (el otro endpoint que paga a Google por llamada) no tiene uno:
`places/autocomplete` se autolimita por la velocidad de tecleo humano; `origin-address` se dispara automáticamente en cuanto el cliente obtiene un GPS, sin ese freno natural — quedó registrado como candidato a la misma protección en `errores-conocidos.md`, sin arreglarlo en este checkpoint.

Alternativas descartadas:
Reutilizar el throttle global (`RATE_LIMIT_*`) sin límite dedicado, siguiendo el precedente de `places/autocomplete` al pie de la letra — descartado porque el patrón de disparo es distinto (ver arriba), y el objetivo explícito era acotar el costo por cuenta, no solo por IP (varios pasajeros pueden compartir NAT/red móvil).

Evidencia:
`src/modules/fares/dto/origin-address.dto.ts`, `src/modules/fares/origin-address.service.ts`, `src/modules/fares/origin-address.service.spec.ts`, `src/modules/fares/fares.controller.ts` (`getOriginAddress`), `src/config/env.validation.ts`, `.env.example`. Commit `06cad3f3`.

---

## `SUGGESTED-DESTINATIONS-R1`: coordenadas del destino expuestas en el historial de pasajero

Estado:
**FINAL-CLOSED-ON-MAIN.** Commit `540153ff` ("feat: expose destination coordinates in passenger ride history"), integrado a `main`@`540153ff` mediante fast-forward puro (2026-08-24). 90/90 test suites, 638/638 tests, lint limpio. `test/suggested-destinations-r1` eliminada (local y remota) tras confirmar contención total en `main`.

Qué se decidió:
`PassengerRideHistoryItemDto` (`GET passenger/rides/history`) gana `destinationLatitude`/`destinationLongitude` (`number | null`, opcionales), seleccionadas en `RideHistoryService.getPassengerHistory` con `ST_Y(ride.destination_position::geometry)`/`ST_X(...)` — mismo patrón ya usado en `admin-rides.service.ts` para exponer coordenadas de un `geography` Point por SQL crudo. Solo el lado pasajero del historial las expone; `getDriverHistory` no las necesita para este checkpoint.

**Verificado explícitamente que no existen viajes sin estas coordenadas**: `Ride.destinationPosition` es `geography NOT NULL` desde `1784766330109-CreateFareQuotesAndRides.ts`, la migración que creó la tabla — nunca se agregó después ni se relajó su nulidad, así que ningún ride, sin importar antigüedad, carece de ella. Los campos igual quedaron opcionales/nullable en el DTO, a pedido explícito de producto, como defensa ante cualquier fila futura o dato corrupto que no la tenga — no porque el caso exista hoy.

Por qué:
Motivado por el diseño de destinos sugeridos del lado Passenger (ver `docs/contexto/App-passenger/decisiones.md`, misma entrada): sin coordenadas, mostrar un destino del historial como sugerencia tocable habría exigido re-resolver el texto de `destinationAddress` vía Places (`autocomplete` + `getDetails`, dos llamadas a Google) cada vez que el pasajero tocara una sugerencia — y sin garantía de llegar al mismo lugar exacto que generó ese texto. Eso es lo grave: el pasajero toca "UPEU" y podría terminar fijando un punto distinto. Exponer las coordenadas ya calculadas elimina ambos problemas de raíz: cero llamadas nuevas a Google al tocar, y el punto es exactamente el mismo que el del viaje original.

Alternativas descartadas:
Dejar que el cliente re-resuelva el texto vía `places/autocomplete`+`getDetails` al tocar una sugerencia — es lo que se iba a implementar antes de decidir tocar Backend; descartado explícitamente por el riesgo de inexactitud y el costo de dos llamadas por toque, ambos evitables con datos que Backend ya tenía en PostGIS.

Evidencia:
`src/modules/rides/dto/passenger-ride-history-response.dto.ts`, `src/modules/rides/ride-history.service.ts` (`getPassengerHistory`), `src/modules/rides/ride-history.service.spec.ts`. Commit `540153ff`.

---

## Identidad compacta de ride (`lastNameInitial`) en las cuatro DTO de participante, sin exponer apellido completo ni PII adicional

Estado:
**FINAL-CLOSED-ON-MAIN.** Commit `f423ecbab8079687c080ce7db2ae40a54ddcde50` ("feat: expose compact ride participant identities"), integrado a `main`@`29fe187aa31f5aad2db20ecd49534a218b358ab6` mediante fast-forward puro junto con el commit de política de teléfono MVP (`CROSS-APP-R4.3` cierre, 2026-08-18), confirmado por el smoke físico final de JuanJo (7/7 PASS, ver checkpoint de cierre en `estado-proyecto.md`). `test/r4-ride-identities` eliminada (local y remota) tras confirmar contención total.

Qué se decidió:
Nuevo util `deriveLastNameInitial()` deriva server-side una inicial de apellido (p. ej. `"P."`) a partir del apellido real, nunca persistida, agregada a las cuatro DTO de identidad de ride: `DriverActiveRideResponseDto.passenger` y `DriverRideOfferResponseDto...passenger` (Passenger visto por Driver, pre y post asignación) y `PassengerRideResponseDto.driver`/`PassengerRideOfferResponseDto.driver` (Driver visto por Passenger, pre y post asignación). Ninguna de las cuatro expone el apellido completo, DNI, teléfono, correo ni dirección — verificado por lectura directa de cada DTO en `CROSS-APP-R4.3G`. La ficha del Driver post-asignación (`AssignedDriverResponseDto`) ya incluía `photoUrl` y `vehicle` (placa/marca/modelo/color) antes de este commit; aquí solo se agrega la inicial del apellido a ese contrato existente.

Por qué:
Primer consumidor real de negocio de la identidad compacta: Passenger y Driver App necesitaban mostrar "Nombre + inicial de apellido" (p. ej. "Juan P.") en vez de solo el primer nombre, sin exponer el apellido completo de ningún usuario a la contraparte de un ride.

Evidencia:
`src/modules/rides/utils/last-name-initial.util.ts`(`.spec.ts`), `src/modules/rides/dto/driver-active-ride-response.dto.ts`, `src/modules/rides/dto/driver-ride-offer-response.dto.ts`, `src/modules/rides/dto/passenger-ride-response.dto.ts`, `src/modules/rides/dto/passenger-ride-offer-response.dto.ts`, `src/modules/rides/ride-view.service.ts`(`.spec.ts`), `src/modules/rides/driver-ride-offers.service.ts`(`.spec.ts`), `src/modules/rides/passenger-rides.service.ts`. 629/629 tests PASS, lint limpio.

---

## `PAYMENT-METHOD-CONTRACT-R1`: el método de pago es referencial, no transaccional — expuesto al conductor antes de aceptar

Estado:
**FINAL-CLOSED-ON-MAIN.** Commit `3962379f76ae2320970938b4e59e1ca3f03bab12` ("feat: expose payment method in driver-facing ride offer contracts"), integrado a `main`@`3962379f` mediante fast-forward puro (2026-08-27). `npm run lint:check` limpio, 90/90 suites, 638/638 tests. Verificado desplegado en Railway STAGING: `paymentMethod` aparece en el esquema de `DriverRideOfferRideDto` en `/docs` con el enum `CASH | YAPE | PLIN | CARD`. `test/payment-method-contract` eliminada (local y remota) tras confirmar contención total. Ver `historial-checkpoints.md` para el detalle de integración.

Qué se decidió:

**1. El método de pago es puramente referencial — no hay pasarela ni intermediación.** El pasajero le paga directo al conductor (efectivo, Yape o Plin, persona a persona); TukiTuki no procesa ni retiene ese cobro. El campo `paymentMethod` que ahora recibe el conductor sirve solo para que sepa de antemano con qué método le van a pagar — la app registra el método elegido únicamente para llevar control, no para ejecutar ninguna transacción. **Izipay se mantiene apagado a propósito** (`IZIPAY_ENABLED` sin tocar en este checkpoint) — la infraestructura de cobro digital ya existe en el código (`DigitalPaymentsService`, ver auditoría previa), pero conectarla es una decisión de producto aparte, no implícita en exponer este campo.

**2. El enum `PaymentMethod` conserva `CARD` aunque el selector que se construya del lado Passenger solo vaya a mostrar `CASH`/`YAPE`/`PLIN`.** No se elimina el valor del enum ni se restringe a nivel de Backend.

Por qué:
El enum es el contrato de datos; el selector visual es interfaz — no tienen que coincidir 1:1. Quitar `CARD` del enum exigiría una migración (`ALTER TYPE ... DROP VALUE` no existe en Postgres de forma directa; requeriría recrear el tipo) para eliminar un valor que hoy no molesta a nadie. Si en el futuro se conecta una pasarela de tarjeta, el valor ya está disponible sin tocar el esquema otra vez.

**3. El conductor debe ver el método de pago ANTES de aceptar o contraofertar, no solo al completar el viaje.** Por eso el campo se expuso en dos DTO, no uno: `DriverRideOfferRideDto` (solicitud entrante, `GET drivers/me/ride-offers/active`) y `DriverPendingProposalResponseDto` (contraoferta ya hecha, esperando al pasajero, `GET drivers/me/ride-offers/proposals/pending`) — los dos puntos donde el conductor todavía puede decidir si le conviene el viaje. Ya llegaba correctamente en `RideCompletionResponseDto` (al completar) y en `DriverActiveRideResponseDto` (viaje ya asignado) desde antes de este checkpoint; ese tramo no se tocó.

Por qué:
Si el pasajero elige Yape y el conductor no tiene Yape, ese viaje no le conviene — pero enterarse recién al completar el viaje es demasiado tarde: ya lo aceptó, ya llegó, ya lo hizo. Mostrarlo antes de responder le permite declinar (o simplemente no contraofertar) un viaje cuyo método de pago no maneja.

Alternativas descartadas:

- Restringir el enum `PaymentMethod` a los 3 valores que el selector del Passenger va a mostrar (`CASH`/`YAPE`/`PLIN`), quitando `CARD` — descartada por requerir una migración para eliminar un valor que no genera ningún problema quedándose (ver punto 2).
- Exponer el método de pago únicamente en `DriverActiveRideResponseDto` (post-asignación), ya que técnicamente ahí ya llegaba sin cambios — descartada explícitamente: el momento en que más importa es antes de aceptar, no después (ver punto 3).
- Habilitar Izipay o construir lógica de cobro digital como parte de este checkpoint, ya que el enum incluye `YAPE`/`PLIN` — descartada explícitamente por alcance: este checkpoint es solo el contrato de lectura del conductor, no el cobro (ver punto 1).

Evidencia:
`src/modules/rides/dto/driver-ride-offer-response.dto.ts` (`DriverRideOfferRideDto.paymentMethod`), `src/modules/rides/dto/driver-pending-proposal-response.dto.ts` (`DriverPendingProposalResponseDto.paymentMethod`), `src/modules/rides/driver-ride-offers.service.ts` (`mapOffer`, `mapPendingProposal`). Commit `3962379f`. Auditoría previa de solo lectura que identificó el tramo roto (sin commit propio, investigación conversacional 2026-08-27): confirmó que `Ride.paymentMethod`/`RidePayment.method` ya existían persistidos, que el Passenger ya podía enviar el método al crear el ride (`CreatePassengerRideDto.paymentMethod`) pero la app mandaba `'CASH'` hardcodeado, y que `RideCompletionResponseDto`/`DriverActiveRideResponseDto` ya lo exponían correctamente antes de este checkpoint.

---

## `minimumFare` de la regla activa de tarifas bajado de 5.00 a 3.00 (STAGING, ajuste manual sin migración)

Estado:
ACTIVA (ajuste de datos, no de esquema — aplicado directamente en la base de STAGING el 2026-08-27).

Qué se decidió:
El campo `minimumFare` de la única `FareRule` activa en STAGING (nombre de la regla: **"Tarifa estándar Tarapoto Staging"**) se cambió de `5.00` a `3.00` (PEN) mediante `UPDATE` directo sobre la tabla `fare_rules` — **no** mediante una migración de TypeORM. El resto de la fórmula (`baseFare: 2.00`, `pricePerKm: 1.20`, `pricePerMinute: 0.10`, `bookingFee: 0.50`) no se tocó.

Por qué:
Decisión de producto: el pasajero negocia directamente el precio con el conductor (`passengerOfferFare`, ver la decisión de negociación bidireccional más arriba en este documento) — el piso no necesita ser conservador para "proteger" un precio sugerido que ya no se muestra (ver la entrada siguiente, tarifa sugerida diferida). `3.00` soles se consideró el piso realista para una carrera corta dentro de Tarapoto.

Por qué sin migración:
Es un cambio de **datos** de una fila de configuración administrable (`FareRule` ya tiene su propio CRUD vía `admin/fare-rules`, pensado exactamente para este tipo de ajuste), no un cambio de **esquema**. La regla "nunca `synchronize`, todo por migración" (ver la primera entrada de este documento) aplica a la estructura de las tablas, no a los valores de negocio que esas tablas administran — no hay ninguna migración de TypeORM que exista solo para poblar/actualizar una fila de configuración operativa.

Pendiente explícito:
**Producción necesitará su propia `FareRule`** cuando se monte ese ambiente — "Tarifa estándar Tarapoto Staging" es, por nombre y por dato, específica de STAGING. No hay todavía ninguna regla equivalente preparada para producción, ni un proceso definido de cómo se replicará este ajuste (u otro) allá.

Alternativas descartadas:
Crear una migración de datos (`INSERT`/`UPDATE` dentro de un archivo de migración de TypeORM) para dejar rastro versionado del cambio — no se usó en este ajuste puntual; queda como opción a considerar si este tipo de cambio de configuración empieza a repetirse con frecuencia y se vuelve valioso tener su historial en el propio control de versiones del esquema.

Evidencia:
Tabla `fare_rules` (columna `minimum_fare`), regla `"Tarifa estándar Tarapoto Staging"`, ambiente Railway STAGING. Sin commit ni archivo de migración asociado — cambio de datos puro, confirmado por JuanJo el 2026-08-27.

---

## Tarifa sugerida (`estimatedFare`): ya calculada y ya llega a la app, pero no se muestra — activación diferida a después de la prueba con usuarios reales

Estado:
ACTIVA — decisión de producto explícita, complementa (no reemplaza) la decisión ya registrada del lado Passenger de "no mostrar Precio recomendado TukiTuki" (ver `docs/contexto/App-passenger/decisiones.md` y `estado-actual.md` secciones 5/13).

Qué se decidió:
`estimatedFare` (`FareEstimateResponseDto`, `POST fares/estimate`) existe, se calcula en tiempo real a partir de la `FareRule` activa y la distancia/duración reales de Google Routes, y **ya llega** al modelo de dominio de la Passenger app (`FareEstimate.fromJson()` lo parsea sin problema). La app simplemente no lo lee en ningún punto de la UI de Home — confirmado por auditoría de código, sin ninguna referencia a `estimatedFare` en `home_screen.dart`. Esto no es un pendiente técnico: es la decisión de producto vigente, ahora verificada contra el código real (cerraba el `[PENDIENTE: verificar el copy exacto]` que quedaba abierto en `estado-actual.md`).

**No se activa mostrarlo todavía.** Se retoma después de la prueba con usuarios reales, cuando existan datos reales de qué precios ofrecieron los pasajeros y cuáles terminaron aceptando los conductores.

Por qué:
La negociación la hacen las personas directamente (pasajero y conductor, sin intermediación de TukiTuki en el precio — ver la decisión de negociación bidireccional). Un número sugerido visible ancla la decisión del pasajero hacia ese valor, incluso si la fórmula no refleja bien la realidad del mercado local todavía. Antes de decidir si mostrarlo (y con qué framing, para no repetir el efecto "precio recomendado" que ya se descartó una vez) hace falta ver qué precios se ofrecen y aceptan de verdad en la práctica — esos datos son mejor base para calibrar tanto la fórmula como la decisión de mostrarla que seguir iterando la fórmula a ciegas.

Alternativas descartadas:
Mostrar `estimatedFare` ahora, ya que técnicamente el dato ya está disponible sin cambios de Backend — descartada explícitamente: la disponibilidad técnica del dato no es la razón por la que no se muestra; la razón es de producto (anclaje de precio) y no cambia por que el campo ya llegue.

Evidencia:
`src/modules/fares/dto/fare-estimate-response.dto.ts` (`estimatedFare`), `src/modules/fares/fares.service.ts` (`estimate`, fórmula completa), `src/modules/fares/entities/fare-rule.entity.ts`. Auditoría de solo lectura 2026-08-27 (sin commit propio): confirmó que el campo llega al modelo `FareEstimate` de la Passenger app y que `home_screen.dart` nunca lo referencia.

---

## Multiplicador nocturno de `FareRule` configurado en 2.00 pero nunca aplicado — la app manda `isNight` hardcodeado en `false`

Estado:
CONFIGURADO Y MUERTO — ver detalle completo en `errores-conocidos.md` (esta entrada es solo el puntero desde decisiones, para quien busque por qué el multiplicador nocturno no afecta ninguna tarifa real hoy).

Qué se observó:
La `FareRule` activa en STAGING tiene `nightMultiplier: 2.00` configurado, pero `FaresService.estimate` solo lo aplica cuando el request trae `isNight: true` — y la Passenger app manda ese campo hardcodeado en `false` (`fare_repository.dart`). En la práctica, ninguna cotización real aplica hoy el multiplicador nocturno, sin importar la hora real del viaje.

Evidencia:
Ver `errores-conocidos.md`.

---

## `FCM-ENABLE-R1` — Firebase Cloud Messaging activado en STAGING (2026-09-01)

Estado:
ACTIVA — cambio de configuración de Railway STAGING, sin commit ni migración. Producción, cuando exista, necesitará su propio service account y su propio `FCM_ENABLED`.

Qué se decidió:
Se puso `FCM_ENABLED=true` en el servicio `tukituki-backend` del ambiente Railway **STAGING**, con `FIREBASE_SERVICE_ACCOUNT_BASE64` (JSON del service account de Firebase, base64) y `FIREBASE_PROJECT_ID` configurados como variables/secretos de Railway. Ningún cambio de código: todo el sistema de push del Backend ya estaba construido y apagado por flag desde antes — `FcmPushService` (`src/modules/notifications/fcm-push.service.ts`, HTTP v1), `NotificationEventHandler`, el registro de dispositivos (`UserDevice`, `POST /me/devices`, upsert por `pushToken` con índice único parcial sobre filas activas) y el consumo desde el Outbox worker.

Por qué:
Mismo patrón que Izipay (`IZIPAY_ENABLED`) y Storage (`STORAGE_ENABLED`): una integración externa completa vive en el código, desactivada por un flag booleano validado por Joi al arranque (activar el flag sin las credenciales correspondientes **falla el boot**, no en tiempo de uso — ver `errores-conocidos.md`). Activar FCM en STAGING era el prerrequisito para que `DRIVER-PUSH-R1` y `PASSENGER-PUSH-R1` (las dos apps registrando dispositivos y consumiendo push) tuvieran algo real contra qué probar.

Verificación:
Se confirmó que activar el flag no introdujo ninguna regresión — la suite del Backend no depende del transporte FCM real (los tests mockean `FcmPushService`). La validación funcional real ocurrió del lado de las apps: registro de dispositivo confirmado en la base de datos de STAGING, y entrega real de notificaciones probada con mensajes desde la consola de Firebase Cloud Messaging (ver `historial-checkpoints.md`, `DRIVER-PUSH-R1` / `PASSENGER-PUSH-R1`).

Alternativas descartadas:
Dejar FCM apagado y seguir probando las apps con `am start`/intents simulados por `adb` — descartada: un intent simulado no pasa por el pipeline interno de Firebase y no dispara `getInitialMessage()` correctamente, así que no permite validar el cold start real (que fue justamente donde apareció el bug `route`/`EXTRA_INITIAL_ROUTE`).

Evidencia:
`src/modules/notifications/fcm-push.service.ts`, `src/config/env.validation.ts` (`FCM_ENABLED`, `FIREBASE_SERVICE_ACCOUNT_BASE64` condicionalmente requerida), `Backend/arquitectura.md` (línea de FCM en integraciones externas). Ambiente Railway STAGING (`tukituki-backend-staging.up.railway.app`). Sin commit — cambio de configuración confirmado por JuanJo el 2026-09-01.

---

## Fix `route`→`screen` en el objeto `data` de todas las notificaciones push: colisión con `EXTRA_INITIAL_ROUTE` de Flutter/Android (2026-09-02)

Estado:
**FINAL-CLOSED-ON-MAIN**, `main`@`48d797bb556a63664ee2a79adb10f50839fb1688`. Parte de un fix coordinado en los tres repos — ver `historial-checkpoints.md`, entrada propia del fix cross-repo. Corte limpio, sin compatibilidad con el nombre viejo (STAGING sin usuarios reales todavía).

Qué se decidió:
La clave `route` dentro del objeto `data` de **todas** las notificaciones push se renombró a `screen`, en las **10 ocurrencias** de `src/modules/notifications/notification-event.handler.ts` donde se arma ese objeto, para **todos** los tipos de evento del sistema — no solo los que las apps consumen hoy:

`ride-offer` (nueva solicitud al conductor), `ride-detail` (cambios de estado del viaje), `ride-receipt` (viaje completado), `ride-rating` (pedido de calificación), `driver-settlement` (liquidación al conductor), `safety-incident` (incidente de seguridad), `emergency-contact-alert` (alerta a contacto de emergencia), `ride-share-links` (enlaces de viaje compartido).

Solo cambió el **nombre de la clave**; los valores (`'ride-offer'`, `'ride-receipt'`, etc.) no se tocaron.

Por qué — la colisión, para quien agregue un evento push en el futuro:
`route` es el nombre de un extra reservado por el **embedding de Flutter en Android** (`EXTRA_INITIAL_ROUTE`, `io.flutter.embedding.android.FlutterActivityLaunchConfigs`). Cuando la app está **completamente terminada** y el usuario toca una notificación, Android arranca la `MainActivity` copiando **cada entrada del `data` de la notificación como un extra del intent**. El embedding de Flutter lee el extra llamado literalmente `"route"` y lo pasa como `initialRoute` al motor de Dart — **antes** de que corra cualquier lógica de navegación de la app. `go_router` recibe entonces `'ride-offer'` (o `'ride-detail'`, etc.) como ubicación inicial, no encuentra ninguna ruta declarada con ese path y lanza `GoException: no routes for location`. El resultado para el usuario: la app "no abre" al tocar **cualquier** notificación con el proceso muerto — y, por un problema aparte del `ErrorScreen` por defecto de `go_router` (su botón apunta a `/`, ruta inexistente en ambas apps), quedaba atrapado sin salida.

**Regla permanente: NUNCA usar una clave llamada `route` en el objeto `data` de una notificación FCM destinada a Android.** También conviene evitar cualquier otro nombre que el embedding de Flutter trate como especial (p. ej. `initial_route`, `background_isolate_run` y similares del namespace `io.flutter.*`). Claves ya en uso y **confirmadas seguras**: `screen`, `eventType`, `rideId`, `offerId`, `status`. Ante la duda con una clave nueva, probar el cold start real (force-stop + notificación real desde la consola de Firebase, no un `adb am start` simulado).

Por qué corte limpio (sin doble clave `route` + `screen` durante una transición):
STAGING no tiene usuarios reales y las dos apps se actualizan en el mismo tramo de trabajo (`tukituki-driver-app` `main`@`2fdfd60`, `tukituki-passenger-app` `main`@`e17d760`). Mandar las dos claves "por las dudas" habría dejado `route` en el payload — es decir, no habría arreglado nada, porque el embedding de Flutter la seguiría leyendo.

Alternativas descartadas:
- Mandar `route` y `screen` a la vez durante una ventana de compatibilidad — descartada: mientras `route` siga en el `data`, el bug persiste (ver arriba).
- Renombrar solo en los eventos que las apps consumen hoy (`ride-offer`, `ride-detail`, `ride-receipt`, `ride-rating`) — descartada: el bug lo dispara **cualquier** notificación con el proceso muerto, incluidas las de seguridad y liquidación; dejar `route` en esas habría dejado el dead-end abierto para esos caminos.
- Resolver el problema del lado app interceptando el extra en Android nativo — descartada: es un fix por app, frágil y específico de plataforma; renombrar en el origen lo arregla para las dos apps y para cualquier cliente futuro de una sola vez.

Evidencia:
`src/modules/notifications/notification-event.handler.ts` (10 ocurrencias de la clave del objeto `data`). Commit `48d797bb`, `main`@`48d797bb556a63664ee2a79adb10f50839fb1688`. Coordinado con `tukituki-driver-app` (`main`@`2fdfd60717e9de996cf95f7f8df9a1104e129309`, lee `data['screen']` + `errorBuilder`) y `tukituki-passenger-app` (`main`@`e17d760253dbe47bf3acfa631268fec610867b19`, ídem).
