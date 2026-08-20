# convenciones

Repositorio:
tukituki-backend

Branch analizada:
main

Commit analizado:
1e029a3ea5aded92cac21dcd4f5996e7bc167bff

Última actualización:
2026-08-15

Fuente de verdad:
Este documento es contexto auxiliar. Si contradice al código actual,
el código y los tests tienen prioridad.

---

## PATRONES OBSERVADOS

### Naming de archivos

- `kebab-case.tipo.ts`: `driver-ride-offers.service.ts`, `ride-cancellation-response.dto.ts`, `jwt-auth.guard.ts`.
- Sufijos consistentes por rol arquitectónico: `.controller.ts`, `.service.ts`, `.module.ts`, `.entity.ts`, `.dto.ts`, `.enum.ts`, `.guard.ts`, `.strategy.ts`, `.interface.ts`, `.util.ts`, `.worker.ts`, `.gateway.ts`, `.interceptor.ts`.
- Tests colocados junto al archivo que prueban: `x.service.ts` + `x.service.spec.ts` en la misma carpeta (no hay carpeta `__tests__`).
- Migraciones: `<timestamp-ms>-<PascalCaseDescriptivo>.ts` (p. ej. `1784766330109-CreateFareQuotesAndRides.ts`), clase homónima con sufijo del timestamp.

### Organización de archivos por módulo

- Un módulo NestJS por dominio bajo `src/modules/<dominio>/`, con subcarpetas `dto/`, `entities/`, `enums/`, `interfaces/`, `utils/` solo cuando aplican.
- **Controllers separados por actor**, no por recurso: en módulos con múltiples audiencias se ven archivos como `passenger-rides.controller.ts`, `driver-rides.controller.ts`, `admin-ride-cancellations.controller.ts`, `driver-ride-offers.controller.ts` en vez de un único `rides.controller.ts`. Mismo patrón en `payments` (`passenger-cash-payments`, `driver-cash-payments`, `admin-cash-payments`), `commissions`, `settlements`, `drivers`/`admin-drivers`, `service-zones`/`admin-service-zones`, `fares`/`admin-fare-rules`, `safety`/`admin-safety-incidents`.
- Módulos grandes (`rides`) no tienen un solo `rides.service.ts`; la lógica se reparte en servicios de responsabilidad única (`ride-dispatch.service.ts`, `ride-transitions.service.ts`, `ride-completion.service.ts`, `ride-cancellations.service.ts`, `ride-start.service.ts`, `ride-ratings.service.ts`, `ride-receipts.service.ts`, `ride-history.service.ts`, `ride-view.service.ts`, `cancellation-fee-calculator.service.ts`, `cancellation-policy.service.ts`).
- Tiempo real vive en subcarpeta `realtime/` dentro del módulo dueño del dominio (`rides/realtime`, `safety/realtime`), no en un módulo global de WebSockets.

### DTOs

- Un DTO por endpoint/forma de respuesta, nombre `Verb/Noun...Dto` (`CreatePassengerRideDto`, `DriverCancelRideDto`, `RideCompletionResponseDto`).
- Validación con `class-validator` (`@IsString`, `@Matches`, `@MinLength`, etc.), documentación con `@nestjs/swagger` (`@ApiProperty`) en el mismo decorador stack.
- Mensajes de error de validación en español, explícitos por regla (`message: '...'`), no genéricos.
- `ValidationPipe` global (`bootstrap.ts`) usa `whitelist: true` y `forbidNonWhitelisted: true`: cualquier campo no declarado en el DTO es rechazado, no ignorado silenciosamente.
- Formato de teléfono peruano validado con la misma regex (`/^\+519\d{8}$/` o variante) repetida en varios DTOs — ver `errores-conocidos.md` por si conviene centralizarla.

### Services / Controllers

- Controllers son delgados: reciben `@CurrentUser()` (decorador propio, `auth/decorators/current-user.decorator.ts`) y `@Param()/@Body()/@Query()`, delegan toda la lógica a uno o varios services inyectados.
- Guardas de ruta declaradas a nivel de clase: `@UseGuards(JwtAuthGuard, RolesGuard)` + `@Roles(UserRole.X)` en el controller completo cuando todos sus endpoints comparten audiencia (patrón dominante en controllers `driver-*`/`passenger-*`/`admin-*`).
- Swagger completo por endpoint: `@ApiOperation`, `@ApiOkResponse`/`@ApiCreatedResponse`, y las respuestas de error relevantes (`@ApiBadRequestResponse`, `@ApiConflictResponse`, `@ApiNotFoundResponse`, `@ApiUnauthorizedResponse`, `@ApiForbiddenResponse`) documentadas explícitamente, incluso códigos no estándar (`@ApiResponse({ status: 423, ... })` para "Locked").
- Workers de background implementan `OnApplicationBootstrap`/`OnApplicationShutdown` con `setInterval` propio, respetando `WORKERS_ENABLED` (`ride-dispatch.worker.ts`, `outbox.worker.ts`) y evitando solapamiento con una bandera `running`.

### Tests

- 72 archivos `*.spec.ts` bajo `src/`, colocados junto al código (Jest `rootDir: src`, `testRegex: .*\.spec\.ts$` en `package.json`).
- Tests e2e separados en `test/*.e2e-spec.ts` con su propia config (`test/jest-e2e.json`, `testRegex: .e2e-spec.ts$`), ejecutados contra una base real (no mocks de base de datos).
- Nombres de `describe`/`it` en español, describiendo comportamiento de negocio (visto en `enable-database-extensions.spec.ts`, `database-readiness.e2e-spec.ts`).
- Existe un test de "smoke" de base de datos dedicado (`test/database-readiness.e2e-spec.ts`) que verifica migraciones aplicadas e idempotencia del seed. Desde `RELEASE-R2.0.1` (ya en `main`) el conteo esperado de migraciones se deriva automáticamente contando los archivos en `src/database/migrations/` — agregar una migración **no** requiere tocar este test (ver `errores-conocidos.md`).

### Migrations

- Cada migración implementa `up`/`down` explícitos; se ha visto al menos un caso (`EnableDatabaseExtensions`) donde `down()` no recibe `QueryRunner` porque no revierte nada (extensiones compartidas de PostgreSQL no se eliminan).
- Nombres de migración describen la intención de negocio, no solo la tabla (`AddDriverSuspensionFields`, `AllowDisabledRideCommissionRate`, `AddRideNegotiation`).
- Los CHECK constraints de negocio se declaran en la entidad TypeORM con `@Check(...)` y se replican en la migración SQL (ver `Ride.platformCommissionRateBps`).

### Error handling

- Filtro global único (`HttpExceptionFilter`, `@Catch()` sin argumento) captura toda excepción, normaliza a `{ statusCode, error, message, requestId, timestamp, path }`, sanitiza query strings del mensaje y loguea en JSON (`warn` para 4xx, `error` para 5xx).
- Los services lanzan excepciones estándar de Nest (`BadRequestException`, `ConflictException`, `NotFoundException`, `ForbiddenException`, `UnauthorizedException`, `ServiceUnavailableException`) con mensajes en español orientados al usuario final.
- `RolesGuard` diferencia explícitamente "no autenticado" (`UnauthorizedException`) de "autenticado sin permiso" (`ForbiddenException`).

### Validación / configuración

- Toda variable de entorno se valida con un único schema Joi (`src/config/env.validation.ts`), con defaults explícitos y validaciones condicionales (`Joi.when`) para integraciones opcionales (Izipay, FCM).
- Convenciones de nombre de env var: `MAYUSCULAS_CON_GUION_BAJO`, agrupadas por dominio con comentario de sección (`# Cancelaciones avanzadas y no-show`, `# Seguridad, SOS y viaje compartido`) en `.env.example`.
- Comentarios de negocio en el propio schema explican por qué existe un default sensible (ver `COMMISSION_MODE` en `env.validation.ts`: por qué arranca en `DISABLED`).

### Patrones Git / estilo de commits

- 100% de los últimos ~61 commits en `main` siguen Conventional Commits: `feat:`, `fix:`, `test:`, `chore:` en minúscula, sin scope, mensaje en inglés e imperativo corto (`feat: expand ride matching window and search radii`).
- Los commits son incrementales y describen una capacidad de negocio completa (endpoint + entidad + migración + tests en el mismo commit), no cambios mecánicos aislados.
- `main` está actualmente limpio (`git status --short` vacío) y sincronizado con `origin/main` en el commit analizado.
- **Histórico**: existió una rama `origin/carlos` (autor `Carlitos-Omar`) con trabajo no fusionado: `feat: harden admin authentication` y `feat: complete passenger fare negotiation flow`. El *comportamiento* de `feat: harden admin authentication` llegó a `main` por una vía distinta (portado/reconciliado en `ADMIN-DRIVER-R1A`, commit `4a537437`, sin fusionar `origin/carlos` directamente). `feat: complete passenger fare negotiation flow` (`6fb5c56e`) fue auditado en detalle (`BRANCH-AUDIT-R1`) y descartado por decisión explícita de producto de JuanJo (mantener negociación de una sola ronda, no multi-ronda) — ver `decisiones.md`. Con ambos commits resueltos, `origin/carlos` se eliminó del remoto en `BRANCH-CLEANUP-R1` (2026-08-16); ya no existe ninguna rama con trabajo divergente sin resolver en este repositorio.

---

## [PENDIENTE: no se pudo determinar...]

- No hay archivo `CONTRIBUTING.md`, plantilla de PR, ni `CODEOWNERS` en el repositorio: **[PENDIENTE: definir proceso de revisión de código del equipo]**.
- No hay convención documentada de branching (`feature/`, `fix/`, etc.) más allá de lo observable en las 2 ramas no-`main` existentes: **[PENDIENTE: confirmar convención de nombres de rama]**.
- No se encontró configuración de commit hooks (`husky`, `lint-staged`) en `package.json` ni `.git/hooks` versionado: **[PENDIENTE: confirmar si el lint/format se fuerza antes de commit o solo en CI]**.
