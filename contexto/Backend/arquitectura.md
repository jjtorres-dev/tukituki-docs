# arquitectura

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

## Stack tecnológico

- **Runtime**: Node.js 22 (`Dockerfile`: `node:22-bookworm-slim`; README exige Node 22).
- **Framework**: NestJS 11 (`@nestjs/core` y `@nestjs/common` instalados en `11.1.28`, rango `^11.0.1` en `package.json`).
- **Lenguaje**: TypeScript `^5.7.3` en `package.json` (instalado `5.9.3`), `strictNullChecks` y `noImplicitAny` activos en `tsconfig.json`.
- **ORM**: TypeORM instalado en `1.1.0` (`node_modules/typeorm/package.json`), migraciones manuales (`synchronize: false` fijado explícitamente en `src/app.module.ts`).
- **Base de datos**: PostgreSQL con PostGIS (`docker-compose.yml` usa `postgis/postgis:18-3.6`); tipos `geography`/`Point` con `srid: 4326` en entidades (p. ej. `Ride.originPosition`).
- **Cache / estructuras efímeras**: Redis 8 (`redis:8.2.7-alpine`), cliente `redis@^6.1.0` vía `src/infrastructure/redis/redis.service.ts`. Se usa GEO (`GEOADD`) para disponibilidad de conductores.
- **Auth**: `@nestjs/passport` + `passport-jwt`, JWT propio (no OAuth de terceros).
- **Validación**: `class-validator` / `class-transformer` + `Joi` (`joi@^18`) para variables de entorno (`src/config/env.validation.ts`).
- **Tiempo real**: `@nestjs/websockets` + `socket.io` (namespaces de rides y safety).
- **Documentación API**: `@nestjs/swagger`, expuesta en `/docs` si `SWAGGER_ENABLED=true`; export a `docs/openapi.json` y `docs/tukituki.postman_collection.json` vía `scripts/export-openapi.ts`.
- **Seguridad HTTP**: `helmet`, `@nestjs/throttler` (rate limit global vía `APP_GUARD`).
- **Testing**: Jest 30 + `ts-jest` para unitarios (`src/**/*.spec.ts`, 72 archivos), Jest + Supertest para e2e (`test/*.e2e-spec.ts`).

## Estructura de carpetas (`src/`)

- `app.module.ts` / `bootstrap.ts` / `main.ts`: composición raíz, configuración transversal (CORS, Helmet, ValidationPipe, filtro de excepciones, Swagger) y arranque HTTP.
- `common/`: filtro de excepciones global (`filters/http-exception.filter.ts`) y middleware de contexto de request (`middleware/request-context.middleware.ts`, agrega `X-Request-Id` y loguea cada request en JSON).
- `config/`: validación de entorno (Joi), CORS, SSL de base de datos, throttling, Swagger.
- `database/`: `data-source.ts` (TypeORM CLI), `migrations/` (29 migraciones al commit `7517c5ea`; el conteo real ya no está hardcodeado en ningún test — ver `errores-conocidos.md` sobre `RELEASE-R2.0.1`), `seeds/` (`super-admin.seed.ts`, `development-data.seed.ts`, con guard de entorno).
- `infrastructure/redis/`: cliente Redis y servicio de disponibilidad geoespacial de conductores.
- `modules/`: un módulo NestJS por dominio de negocio (22 módulos, ver abajo).

## Módulos principales (`src/modules`)

`auth`, `authorization` (guard de roles), `auth-sessions` (sesiones/refresh tokens), `users`, `passengers`, `drivers`, `admin-drivers`, `driver-operations` (estado operativo y ubicación en vivo), `service-zones`, `fares` (cotizaciones, reglas de tarifa, Google Routes/Geocoding), `rides` (el módulo más grande: matching, ciclo de vida, cancelaciones, calificaciones, WebSocket, ride-start-code), `admin-rides`, `commissions`, `payments` (efectivo + Izipay), `settlements` (liquidaciones a conductores), `promotions`, `safety` (SOS, contactos de emergencia, viaje compartido), `notifications` (push FCM + notificaciones in-app), `outbox` (event sourcing saliente), `operations` (auditoría admin y métricas), `places` (búsqueda de direcciones), `health`.

Cada submódulo de dominio suele separar controllers por actor: `passenger-*.controller.ts`, `driver-*.controller.ts`, `admin-*.controller.ts` (ver `convenciones.md`).

**Ya en `main` desde `RELEASE-R2`** (2026-08-16, fast-forward, `main`@`4a537437`): agrega un módulo `storage` (presigned upload/complete a Railway Storage Buckets, avatares por capability token — no por `profileId` público, descarga privada de documentos) y un `S3StorageService` en `src/infrastructure/storage`, implementados originalmente en `test/storage-r2-railway-buckets` (checkpoints STORAGE-R2 + STORAGE-R2.1). `main` no es producción — ver `decisiones.md` y `docs/contexto/estado-proyecto.md` sección 10.

## Entidades principales

`User` (roles múltiples vía enum array), `PassengerProfile`, `DriverProfile`/`DriverVehicle`/`DriverDocument`, `DriverOperationalState`/`DriverLocation`, `ServiceZone`, `FareRule`/`FareQuote`, `Ride` (entidad central, ~50 columnas: tarifa, negociación, cancelación, comisión, geolocalización), `RideOffer`, `RideStartCode`, `RideCancellation`, `RideRating`, `RideLocationSample`, `RidePayment`/`DigitalPaymentAttempt`, `CommissionPolicy`/`RideCommission`, `DriverSettlement`/`DriverSettlementItem`, `Promotion`/`PromotionRedemption`, `EmergencyContact`/`RideSafetyIncident`/`RideShareLink`, `UserDevice`/`UserNotification`, `OutboxEvent`, `AdminAuditLog`, `AuthSession`.

## Flujo de datos (viaje típico)

1. Pasajero cotiza (`fares`, usa Google Routes para distancia/duración) y crea el ride (`SEARCHING_DRIVER`).
2. `RideDispatchWorker` (polling configurable, `RIDE_DISPATCH_POLL_INTERVAL_MS`) busca conductores disponibles vía Redis GEO (`drivers:available`) dentro de un radio creciente (`RIDE_SEARCH_RADII_METERS` = 2/5/10 km) y crea `RideOffer`.
3. El conductor responde (acepta, contraoferta o rechaza) vía `driver-ride-offers`; el pasajero elige (`passenger-rides`).
4. Transiciones de estado (`DRIVER_ASSIGNED` → `DRIVER_ARRIVING` → `DRIVER_ARRIVED` → `IN_PROGRESS` → `COMPLETED`) validan GPS contra PostGIS.
5. Cambios de estado se notifican en vivo por WebSocket (`rides.gateway.ts`) y se encolan como `OutboxEvent` para procesamiento asíncrono (`outbox.worker.ts` → notificaciones push/in-app).
6. Al completar, se calcula tarifa final, se genera `RidePayment` (efectivo o Izipay) y, si `COMMISSION_MODE=ENFORCED`, se acumula `RideCommission` para liquidación posterior (`settlements`).

## Servicios externos

- **Google Routes API** (`fares/google-routes.service.ts`): distancia/duración/polyline para cotizaciones.
- **Google Places / Geocoding** (`fares/google-geocoding.service.ts`, `places/`): resolución de direcciones.
- **Izipay** (`payments/gateways/izipay-payment.gateway.ts`): pasarela de pago digital (sandbox por defecto), deshabilitada salvo `IZIPAY_ENABLED=true`.
- **Firebase Cloud Messaging (HTTP v1)** (`notifications/fcm-push.service.ts`): push nativo, deshabilitado salvo `FCM_ENABLED=true` con credenciales de service account.

Los tres últimos son opcionales y están apagados por defecto en `.env.example`; el código valida explícitamente su configuración cuando se habilitan (`env.validation.ts`, reglas `Joi.when`).

## Persistencia

PostgreSQL vía TypeORM, `autoLoadEntities: true`, `synchronize: false` (comentario explícito en `app.module.ts`: "Nunca dependeremos de synchronize en TukiTuki"). Todo cambio de esquema pasa por migración explícita (`npm run migration:generate` / `migration:run`). Redis se usa solo como cache/estado efímero (disponibilidad geoespacial, presencia), no como fuente de verdad transaccional.

## Autenticación / autorización

- JWT de acceso (`JWT_ACCESS_SECRET`, TTL configurable) validado en `JwtStrategy` (`src/modules/auth/strategies/jwt.strategy.ts`), que además verifica estado del usuario (`ACTIVE`) y sesión activa (`auth-sessions`, refresh tokens rotativos).
- Autorización por rol: decorador `@Roles()` + `RolesGuard` (`src/modules/authorization`), evaluado sobre `UserRole` (`PASSENGER`, `DRIVER`, `ADMIN`, `SUPER_ADMIN`). `SUPER_ADMIN` tiene bypass explícito en el guard.
- WebSocket usa el mismo JWT vía `WsJwtAuthGuard` (`src/modules/rides/realtime/ws-jwt-auth.guard.ts`), extrayendo el token de `handshake.auth`, headers o query.
- Verificación de teléfono por OTP (`auth/otp.service.ts`), con secreto de hash y máximo de intentos configurables. **Ya en `main`@`f147a664` desde `OTP-R2.3`, validado en runtime real de Railway STAGING desde `main` en `OTP-R2.4`** (checkpoint `OTP-R2`, fast-forward de `test/otp-r2-hardening`): rate limit dedicado por IP/teléfono en `POST /auth/otp/request`, enumeración de cuentas cerrada, cooldown preservado al agotar intentos, y guard de arranque que impide `OTP_DEBUG_ENABLED=true` en `NODE_ENV=production` — ver `decisiones.md`. `main` no es producción. Sigue sin existir ningún `SmsProvider`/transporte SMS real (`OTP-R3`, pendiente).
- **Ya en `main` desde `RELEASE-R2`** (commit `4a537437`, checkpoint ADMIN-DRIVER-R1A): agrega `POST /auth/admin/login`, exclusivo para `ADMIN`/`SUPER_ADMIN`, con `AdminLoginSecurityService` (throttling/lockout por cuenta e IP vía Redis, fingerprints HMAC pseudonimizados) y `PasswordService.verifyWithFallback` (comparación bcrypt ficticia para evitar canal lateral de tiempo cuando el usuario no existe) — ver `decisiones.md`.

## Deploy

- Imagen Docker multi-stage (`Dockerfile`): build con `npm ci` + `nest build`, imagen final `node:22-bookworm-slim` como usuario no root (`USER node`), healthcheck HTTP embebido.
- `docker-compose.yml` (desarrollo local: Postgres+PostGIS y Redis) y `docker-compose.prod.yml` (agrega el servicio `api`, `read_only: true`, `security_opt: no-new-privileges`).
- Migraciones son un paso manual explícito antes de levantar el contenedor (`migration:run:prod`), nunca automáticas al iniciar una réplica (ver README).
- CI (`.github/workflows/ci.yml`): lint, tests unitarios, build, smoke de migraciones sobre base limpia, e2e, export de OpenAPI, build de imagen Docker — en cada PR/push a `main`/`master`.
- CD (`.github/workflows/cd.yml`): publica imagen a GHCR solo en tags `v*` o `workflow_dispatch`; no despliega automáticamente a producción.

## Cosas importantes que actualmente NO existen

Verificado por ausencia de archivos/dependencias relevantes:

- No hay carpeta `apps/` ni estructura de monorepo Nest — es una única aplicación NestJS.
- No hay ningún cliente GraphQL ni módulo GraphQL (`@nestjs/graphql` no está en `package.json`).
- No hay colas de mensajería externas (RabbitMQ, Kafka, SQS): el patrón outbox (`modules/outbox`) usa polling sobre PostgreSQL, no un broker.
- No hay ningún proveedor de pago digital adicional a Izipay (`PaymentProvider` enum solo define `IZIPAY`).
- No hay Sentry ni APM externo configurado (`grep` sobre `src/` no encontró referencias); el logging es `ConsoleLogger` de Nest con formato JSON opcional.
- No hay `synchronize: true` en ningún ambiente — confirmado en `app.module.ts`.
- No se encontraron comentarios `TODO`/`FIXME`/`HACK` en `src/` (búsqueda exhaustiva; solo coincidencias falsas en mensajes de validación de formato telefónico).
- No hay tests de carga/performance en el repo (solo unitarios y e2e funcionales).
- [PENDIENTE: no se pudo determinar si existe documentación de arquitectura previa fuera de este repositorio — no se buscó fuera de `tukituki-backend` por alcance de la tarea]
