# glosario

Repositorio:
tukituki-backend

Branch analizada:
main

Commit analizado:
1e029a3ea5aded92cac21dcd4f5996e7bc167bff

Última actualización:
2026-08-16

Fuente de verdad:
Este documento es contexto auxiliar. Si contradice al código actual,
el código y los tests tienen prioridad.

---

## Entidades de dominio

- **User**: cuenta con uno o más `UserRole`, autenticada por teléfono E.164. `src/modules/users/entities/user.entity.ts`.
- **PassengerProfile**: perfil de pasajero asociado a un `User`.
- **DriverProfile**: perfil de conductor, con estado de aprobación (`DriverStatus`).
- **DriverVehicle**: vehículo (mototaxi) registrado por un conductor.
- **DriverDocument**: documento de verificación de conductor (DNI, licencia, SOAT, etc.).
- **DriverOperationalState**: estado operativo en vivo del conductor (`OFFLINE`/`AVAILABLE`/`BUSY`).
- **DriverLocation**: última ubicación conocida del conductor (PostGIS).
- **ServiceZone**: zona geográfica donde opera TukiTuki, con reglas de tarifa asociadas.
- **FareRule**: regla de tarifa (base, por km, por minuto, booking fee) de una `ServiceZone`.
- **FareQuote**: cotización de tarifa generada antes de crear un `Ride`.
- **Ride**: viaje; entidad central del dominio. Contiene tarifa, negociación, cancelación, comisión y geolocalización.
- **RideOffer**: oferta de un `Ride` enviada a un conductor específico durante el matching.
- **RideStartCode**: código de un solo uso que el pasajero entrega al conductor para iniciar el viaje.
- **RideCancellation**: registro de la cancelación de un `Ride` (quién, por qué, cuándo).
- **RideRating**: calificación mutua (pasajero ↔ conductor) al finalizar un viaje.
- **RideLocationSample**: muestra de ubicación GPS durante un viaje en curso.
- **RideProgressMetrics**: métricas agregadas de avance de un viaje.
- **RideWaiting**: registro del tiempo de espera del conductor en el punto de origen.
- **RideStatusHistory**: historial de transiciones de estado de un `Ride`.
- **RideFinalFare**: desglose de la tarifa final calculada al completar un viaje.
- **CancellationPolicy**: política de cargos por cancelación aplicable a un `Ride`.
- **UserFinancialObligation**: cargo pendiente sobre un usuario (cuota de cancelación, no-show).
- **CommissionPolicy**: política vigente de comisión de plataforma.
- **RideCommission**: comisión acumulada sobre un `Ride` específico.
- **RidePayment**: pago de un `Ride` (efectivo o digital).
- **DigitalPaymentAttempt**: intento de pago digital vía pasarela (Izipay).
- **DriverSettlement** / **DriverSettlementItem**: liquidación periódica de saldos entre plataforma y conductor.
- **Promotion** / **PromotionRedemption**: cupón/promoción y su canje en un viaje.
- **EmergencyContact**: contacto de emergencia registrado por un usuario.
- **EmergencyContactAlert**: alerta enviada a un contacto de emergencia.
- **RideSafetyIncident**: incidente de seguridad reportado durante o después de un viaje.
- **RideShareLink**: enlace público temporal para compartir el seguimiento en vivo de un viaje.
- **RideShareAccessLog**: registro de accesos a un `RideShareLink`.
- **UserDevice**: dispositivo registrado de un usuario para push notifications (FCM).
- **UserNotification**: notificación in-app persistida para un usuario.
- **OutboxEvent**: evento de negocio pendiente de procesar por el worker de outbox.
- **AdminAuditLog**: registro de auditoría de acciones administrativas.
- **AuthSession**: sesión de autenticación con refresh token rotativo.

## Roles y estados

- **UserRole**: `PASSENGER`, `DRIVER`, `ADMIN`, `SUPER_ADMIN`.
- **UserStatus**: `ACTIVE`, `PENDING`, `SUSPENDED`, `BLOCKED`.
- **RideStatus**: `SEARCHING_DRIVER`, `DRIVER_ASSIGNED`, `DRIVER_ARRIVING`, `DRIVER_ARRIVED`, `IN_PROGRESS`, `COMPLETED`, `CANCELLED`, `EXPIRED`.
- **RideOfferStatus**: `OFFERED` (enviada al conductor) → `PROPOSED` (conductor respondió, con o sin contraoferta) → `ACCEPTED` (pasajero eligió esta oferta), o `REJECTED`/`EXPIRED`/`CANCELLED`.
- **DriverOperationalStatus**: `OFFLINE`, `AVAILABLE`, `BUSY`.
- **DriverStatus** / **VehicleStatus** / **DriverDocumentStatus**: `DRAFT`, `PENDING_REVIEW`, `APPROVED`, `REJECTED` (+ `SUSPENDED` en `DriverStatus`/`VehicleStatus`).
- **RideCancellationType**: `PASSENGER_CANCELLED`, `DRIVER_CANCELLED`, `PASSENGER_NO_SHOW`, `DRIVER_NO_SHOW_CANCELLED`.
- **RidePaymentStatus**: `PENDING`, `PROCESSING`, `PAID`, `FAILED`, `EXPIRED`, `DISPUTED`, `VOIDED`.
- **RideCommissionStatus**: `ACCRUED`, `ALLOCATED`, `HELD`, `SETTLED`, `REVERSED`.
- **SettlementStatus**: `DRAFT`, `APPROVED`, `SETTLED`, `CANCELLED`.
- **SafetyIncidentStatus**: `OPEN`, `ACKNOWLEDGED`, `IN_REVIEW`, `RESOLVED`, `FALSE_ALARM`.
- **SafetyIncidentSeverity**: `MEDIUM`, `HIGH`, `CRITICAL`.
- **RideShareLinkStatus**: `ACTIVE`, `REVOKED`, `EXPIRED`.
- **PromotionStatus**: `ACTIVE`, `PAUSED`. **PromotionRedemptionStatus**: `RESERVED`, `APPLIED`, `RELEASED`.
- **OutboxEventStatus**: `PENDING`, `PROCESSING`, `PROCESSED`, `DEAD`.

## Siglas y conceptos internos

- **OTP**: One-Time Password, código de verificación telefónica (`auth/otp.service.ts`).
- **E.164**: formato internacional de número telefónico (`+51987654321`) usado en `User.phoneE164`.
- **PostGIS**: extensión geoespacial de PostgreSQL; tipo `geography`/`Point` (`srid: 4326`) usado en `Ride.originPosition`, `DriverLocation`, etc.
- **BPS**: basis points (centésimas de punto porcentual), unidad de `Ride.platformCommissionRateBps`.
- **PEN**: Sol peruano (código de moneda), usado en nombres de variables de entorno de tarifas (`CANCELLATION_FEE_ASSIGNED_PEN`).
- **G3A / G3B1 / G3B2 / G3C-lite**: nombres internos de iteraciones del sistema de matching, usados como referencia en comentarios de código (`ride-matching.constants.ts`, `driver-availability-redis.service.ts`) para documentar qué responsabilidad resuelve cada cambio. No hay documento externo que los defina; su significado se infiere únicamente de los comentarios donde aparecen.
- **FCM**: Firebase Cloud Messaging, proveedor de push notifications (HTTP v1, `notifications/fcm-push.service.ts`).
- **Outbox (patrón)**: escritura de eventos de dominio en una tabla (`OutboxEvent`) para procesamiento asíncrono garantizado, en vez de publicarlos directamente a un broker externo.
- **Late-join (matching)**: reintento de matching para un conductor que se vuelve disponible después de que un `Ride` ya estaba `SEARCHING_DRIVER`.
- **Dispatch round**: número de ronda de intento de matching automático de un `Ride` (`Ride.dispatchRound`).
- **Ride start code**: código de inicio de viaje que el pasajero muestra al conductor (`RideStartCode`), con intentos y regeneraciones limitadas.
- **Advisory lock**: bloqueo consultivo de PostgreSQL usado para evitar procesar el mismo `Ride` concurrentemente (`ride-advisory-lock.util.ts`).

## Storage (STORAGE-R2 / STORAGE-R2.1 — ya en `main` desde `RELEASE-R2`, commit `4a537437`)

- **objectKey**: ruta única de un archivo dentro del bucket de Railway Storage, generada siempre por el Backend (nunca por el cliente), con forma `<prefijo-por-owner>/<uuid>.<extensión>` (`src/modules/storage/storage-object-key.util.ts`).
- **StorageCategory**: enum cerrado con las 5 categorías de Storage vigentes (`PASSENGER_PROFILE_PHOTO`, `DRIVER_PROFILE_PHOTO`, `DRIVER_LICENSE`, `SOAT`, `VEHICLE_REGISTRATION`); deliberadamente no incluye `DNI_FRONT`/`DNI_BACK`/`PROFILE_PHOTO` de `DriverDocumentType` (`src/modules/storage/enums/storage-category.enum.ts`).
- **Presigned URL**: URL temporal firmada por el Backend que autoriza una operación puntual (PUT para subir, GET para leer) directamente contra el bucket, sin exponer credenciales reales al cliente.
- **HeadObject**: operación S3 usada en `POST /storage/uploads/complete` para confirmar que el archivo subido existe, y comprobar su `ContentType`/`ContentLength` reales antes de vincularlo a un `DriverProfile`/`PassengerProfile`/`DriverDocument`.
- **photoObjectKey / fileObjectKey**: columnas nullable (`PassengerProfile`/`DriverProfile`/`DriverDocument`) que guardan el `objectKey` canónico en Storage, coexistiendo con las columnas legacy `photoUrl`/`fileUrl`.
- **isLegacyUrl**: campo de la respuesta de descarga de documento (`DownloadUrlResponseDto`) que distingue si la URL devuelta es una presigned GET fresca (`false`) o el `fileUrl` legacy tal cual, sin expiración administrada por el Backend (`true`).
- **Capability token (avatar token)** *(STORAGE-R2.1)*: token HMAC-SHA256 firmado con `STORAGE_AVATAR_TOKEN_SECRET`, con `kind` (`driver`/`passenger`) + `profileId` + `expiresAt` codificados en el payload firmado. Es el único mecanismo de autorización de `GET /storage/avatars/driver|passenger/:id`; el `profileId` de la ruta, por sí solo, no alcanza (`src/modules/storage/avatar-url.util.ts`: `mintAvatarToken`/`verifyAvatarToken`).
- **AvatarUrlResolverService** *(STORAGE-R2.1)*: único punto donde se resuelve el `photoUrl` a exponer en una respuesta ya autorizada — mintea un capability token si hay `photoObjectKey`, o conserva el `photoUrl` legacy si no (`src/modules/storage/avatar-url-resolver.service.ts`).
- **AdminLoginSecurityService** *(ADMIN-DRIVER-R1A, solo en `test/storage-r2-railway-buckets`)*: throttling/lockout de `POST /auth/admin/login` con contadores en Redis por cuenta e IP, identificados por fingerprints HMAC pseudonimizados (nunca el teléfono/IP en claro como clave); bloquea la cuenta tras `ADMIN_LOGIN_ACCOUNT_MAX_FAILURES` fallos y responde 429 sin revelar cuál límite se alcanzó (`src/modules/auth/admin-login-security.service.ts`).
