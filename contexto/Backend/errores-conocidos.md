# errores-conocidos

Repositorio:
tukituki-backend

Branch analizada:
main

Commit analizado:
29fe187aa31f5aad2db20ecd49534a218b358ab6

Última actualización:
2026-08-18 (refrescado tras el cierre de `CROSS-APP-R4.3`, fast-forward de `test/r4-ride-identities` a `main`)

Fuente de verdad:
Este documento es contexto auxiliar. Si contradice al código actual,
el código y los tests tienen prioridad.

---

## Baseline verificado en este análisis

Ejecutado directamente sobre el commit `1e029a3e`, sin dependencias instaladas ni código modificado (`node_modules` ya presente):

- `npm test -- --runInBand`: **72 suites, 401 tests, todos en verde** (27.3 s).
- `npm run lint:check` (ESLint con `--max-warnings=0`): **sin errores ni warnings**.
- `npm run build` (`nest build`): **compila sin errores de TypeScript**.
- `git status --short` después de estos tres comandos: **sin cambios** (los comandos no tocan archivos versionados; `dist/` está en `.gitignore`).

No se ejecutaron `test:db:smoke` ni `test:e2e` en este análisis porque requieren una base PostgreSQL/Redis desechable levantada (`docker compose up -d`) fuera del alcance de esta revisión de solo lectura; su comportamiento esperado está documentado en `flujo-de-trabajo.md`.

## Baseline `main`@`ede503bc` (DRIVER-ONBOARDING-R2/R2.3, ya fusionado)

`npm test -- --runInBand` → **85 suites, 568 tests, todos en verde** (+3 suites/+26 tests sobre el baseline anterior de 82/542 de `main`@`7517c5ea`); `npm run lint:check` y `npm run build` limpios — validado localmente sobre `test/driver-onboarding-r2-backend` (2026-08-16) y re-confirmado sin cambios de código tras el fast-forward a `main` (`DRIVER-ONBOARDING-R2.3`). `test:db:smoke` local no se ejecutó en ningún punto — Docker sin daemon disponible en este entorno (`INFRA-NOT-AVAILABLE`), igual que en checkpoints previos de Storage; sí se validó la migración nueva (`1786950000000-PrepareDriverOnboardingR2.ts`) por dos vías reales: GitHub CI contra Postgres limpio (`DRIVER-ONBOARDING-R2.1`, PR sin fusionar) y Railway STAGING con datos históricos reales (`DRIVER-ONBOARDING-R2.2`, backward compatibility confirmada por JuanJo). **Este es ahora el baseline oficial de `main`** (82/542 → 85/568). `MAIN CI` del push específico a `main` en `R2.3`: pendiente de confirmación manual (sin `gh`/API de GitHub en este entorno).

## Baseline `main`@`f147a664` (OTP-R2/R2.3, ya fusionado)

`npm test -- --runInBand` → **88 suites, 601 tests, todos en verde** (+3 suites/+33 tests sobre el baseline de 85/568 de `main`@`ede503bc`); `npm run lint:check` y `npm run build` limpios — validado localmente sobre `test/otp-r2-hardening` (2026-08-16) y re-confirmado sin cambios de código tras el fast-forward a `main` (`OTP-R2.3`). Sin migración nueva (OTP vive enteramente en Redis), por lo que `test:db:smoke` no aplica a este checkpoint. `test:e2e` no se re-ejecutó localmente (requiere Postgres/Redis reales, mismo `INFRA-NOT-AVAILABLE` de checkpoints previos); sí se validó en GitHub CI real sobre el commit TEST (`OTP-R2.1`, Backend CI #32, 7 checks en verde) y en runtime real de Railway STAGING (`OTP-R2.2`, desplegado desde la rama TEST). **Este es ahora el baseline oficial de `main`** (85/568 → 88/601). `MAIN CI` del push específico a `main` en `R2.3`: pendiente de confirmación manual (sin `gh`/API de GitHub en este entorno).

## Baseline `main`@`29fe187a` (CROSS-APP-R4.3, ya fusionado)

`npm test -- --runInBand` → **629/629 tests, todos en verde**, `npm run lint:check` limpio — validado sobre `test/r4-ride-identities` (2026-08-18) y re-confirmado sin cambios de código tras el fast-forward a `main` (checkpoint de cierre `CROSS-APP-R4.3`, 2026-08-18). Incluye `deriveLastNameInitial()` (identidad compacta de ride) e `isUserOperationallyEnabled()` (política MVP de teléfono verificado — ver `decisiones.md`). Confirmado además por smoke físico final de JuanJo sobre Railway STAGING desde `main` (7/7 PASS). **Este es ahora el baseline oficial de `main`.** `test/r4-ride-identities` eliminada local y remotamente tras el cierre; `test/otp-demo-staging` permanece intacta.

## Logs de warning esperados durante `npm test` (no son fallos)

La suite unitaria imprime deliberadamente logs de `ERROR`/`WARN` de Nest como parte de tests que ejercitan rutas de fallo controlado (reintentos, fallbacks, timeouts simulados). Confirmado por lectura de los specs correspondientes:

- `[RideDispatchWorker] El late-join matching falló para el conductor driver-1: boom` — test que verifica manejo de errores en `runLateJoinOnce`.
- `[OutboxWorker] Evento ... falló: fallo simulado` — test de reintento del worker de outbox.
- `[GoogleGeocodingService] ... se usa fallback` (varias líneas) — tests del fallback de geocodificación cuando Google Places/Geocoding no responde o no está configurado.
- `[DriverDailyStatsService] ... Rides COMPLETED en múltiples monedas` — test de agregación con monedas mixtas.

Ninguno de estos logs indica una prueba fallida; los 401 tests terminan en verde.

## Configuraciones delicadas verificadas en código

- **`test:db:smoke` exige una base literalmente vacía y con nombre que contenga `test` o `smoke`** (`test/database-readiness.e2e-spec.ts:71-92`); si se apunta a una base con datos o nombre distinto, el test lanza error explícito en lugar de aplicar migraciones a ciegas.
- **(RESUELTO, `RELEASE-R2.0.1`, ya en `main` — commit `7517c5ea`) El smoke test verificaba un conteo exacto de migraciones aplicadas hardcodeado (`28`)** (`test/database-readiness.e2e-spec.ts:135`). Se rompió en Backend CI apenas se agregó `AddStorageObjectKeys1786860000000` (elevó el conteo real a 29) — confirmado en el run "feat: add dedicated admin login #27" de GitHub Actions: todas las migraciones se aplicaron correctamente, solo falló la aserción. Fix: el conteo esperado ahora se deriva contando los archivos de migración reales en `src/database/migrations/` en cada corrida (`countMigrationSourceFiles()`), en vez de un número fijo — agregar una migración futura ya **no** exige tocar este test. Validado con GitHub CI en verde (run "test: make migration smoke count resilient #28", incluido este mismo paso).
- **`COMMISSION_MODE` por defecto es `DISABLED`**; el código de comisiones existe y se ejecuta, pero no es exigible hasta cambiar explícitamente a `ENFORCED` (ver `decisiones.md`). Cambiar este valor en producción sin coordinación de negocio activaría cobros no esperados.
- **`DATABASE_SSL_REJECT_UNAUTHORIZED` debe mantenerse en `true` en producción** (README, sección "Operación y seguridad"); `DATABASE_SSL_CA_BASE64` solo debe usarse si el proveedor entrega una CA privada.
- **Migraciones nunca se ejecutan automáticamente al iniciar una réplica** (`docker-compose.prod.yml`, README) — es un paso manual (`migration:run:prod`) antes de `up -d`. Desplegar sin ejecutar este paso deja el esquema desactualizado sin error visible al arranque.
- **Variables condicionalmente requeridas por Joi**: `IZIPAY_MERCHANT_CODE`/`IZIPAY_API_KEY`/`IZIPAY_KEY_HASH`/`IZIPAY_PUBLIC_KEY` solo son obligatorias si `IZIPAY_ENABLED=true`; `FIREBASE_SERVICE_ACCOUNT_BASE64` solo si `FCM_ENABLED=true` (`src/config/env.validation.ts`). Activar el flag sin las credenciales correspondientes falla el arranque de la aplicación (validación Joi al boot), no en tiempo de uso.

## Trabajo divergente entre ramas (histórico — ya cerrado en `BRANCH-CLEANUP-R1`)

- **(RESUELTO, `BRANCH-CLEANUP-R1`, 2026-08-16) `origin/carlos` (Backend) fue eliminada del remoto.** Contenía una migración (`1786233600000-AddPassengerRideCounterOffers.ts`) y cambios de negociación de tarifa multi-ronda que nunca estuvieron en `main` (la cual tiene su propia implementación independiente de negociación de una sola ronda: `1786147200000-AddRideNegotiation.ts`). `BRANCH-AUDIT-R1` (2026-08-16) auditó en detalle el commit `6fb5c56e` ("feat: complete passenger fare negotiation flow") y confirmó que agregaba una capacidad de producto real (contraoferta del Passenger de vuelta a un Driver, estado `PASSENGER_COUNTERED`) — no un duplicado técnico de lo que ya existía. JuanJo decidió explícitamente **no adoptar** esa dirección de producto (mantener negociación de una sola ronda); el commit quedó marcado `REJECTED-BY-PRODUCT-DECISION`, no `OBSOLETE` ni "perdido". El otro contenido exclusivo de `origin/carlos` — el commit `60cccd9d` de endurecimiento de login administrativo — ya había sido reconciliado (portado, no fusionado) en `ADMIN-DRIVER-R1A` (commit `4a537437`, ya en `main` desde `RELEASE-R2`). Con ambos commits resueltos (uno reconciliado, otro rechazado por decisión de producto) y ningún commit nuevo detectado en la re-auditoría de `BRANCH-CLEANUP-R1`, la rama se eliminó del remoto (`git push origin --delete carlos`, sin `--force`; no existía copia local). Ver `decisiones.md` para el detalle de la decisión de negociación.
- `test/storage-r2-railway-buckets` (commit `4a537437`, sobre `050e5657`) contiene la implementación Backend de **STORAGE-R2** (módulo `storage`, migración `1786860000000-AddStorageObjectKeys`), el hardening de privacidad **STORAGE-R2.1** (capability token para avatares) y, desde **ADMIN-DRIVER-R1A**, `POST /auth/admin/login` con throttling/lockout dedicado (ver `arquitectura.md`/`decisiones.md`). Baseline verificado en esa rama: **82 suites, 542 tests, todos en verde** (518 + 24 nuevos de ADMIN-DRIVER-R1A), lint limpio, build limpio — mismo resultado reconfirmado en `main` tras `RELEASE-R2`. Igual que en R2/R2.1, no se ejecutaron `test:db:smoke`/`test:e2e` en esta rama (Docker instalado pero el daemon no está disponible en este entorno). `docs/openapi.json`/`docs/tukituki.postman_collection.json` no se regeneraron localmente (el script requiere una conexión real a PostgreSQL, no disponible aquí); CI los regenera en su propio pipeline. **`POST /auth/admin/login` y `GET /storage/admin/documents/:id/download-url` sí se probaron en vivo contra Railway staging (ADMIN-DRIVER-R1A.1, mismo commit, sin cambio de código): PASS con una cuenta `SUPER_ADMIN` real — ya no queda `TEST CREDENTIAL REQUIRED` pendiente para staging.** **Actualización (`RELEASE-R2`, 2026-08-16): esta rama se integró a `main` por fast-forward (`main`@`4a537437`). `main` no es producción** — ver `docs/contexto/estado-proyecto.md` sección 10.

## Limitaciones técnicas demostrables

- El patrón de reintento de comisión/matching/outbox se basa en **polling sobre PostgreSQL**, no en un broker de mensajería; la latencia de procesamiento de eventos está acotada por `OUTBOX_POLL_INTERVAL_MS` (default 1000 ms) y `RIDE_DISPATCH_POLL_INTERVAL_MS` (default 2000 ms), no es instantánea por diseño.
- La validación de formato telefónico peruano (`/^\+519\d{8}$/` o equivalente) está **duplicada literalmente** en al menos 6 archivos DTO distintos (`src/modules/auth/dto/*.dto.ts`, `src/modules/admin-drivers/dto/admin-driver-query.dto.ts`, `src/modules/passengers/dto/create-passenger-profile.dto.ts`) en vez de un validador compartido; un cambio de formato requiere editar todos los puntos.
- **Ni `tukituki-driver-app` ni `tukituki-passenger-app` pueden hacer un fetch de imagen autenticado** (confirmado por auditoría de código en STORAGE-R2.1): ambas apps cargan cualquier `photoUrl` con `Image.network(url)` plano, sin `headers`, sin `cached_network_image` ni ningún cliente de imágenes envuelto en Dio. Esto es una restricción real de diseño para cualquier futuro endpoint de foto/imagen: no puede exigir `Authorization` en el header sin antes modificar Flutter — la autorización tiene que viajar en la propia URL (query param firmado, como el capability token de avatares) o el flujo se rompe silenciosamente (imagen no carga, sin error visible más allá del `errorBuilder` de cada widget).
- No se encontraron comentarios `TODO`/`FIXME`/`HACK`/`XXX` en `src/` en este análisis (búsqueda exhaustiva con `Grep`); no hay deuda técnica marcada explícitamente en el código para priorizar.

## [PENDIENTE: no se pudo determinar...]

- Comportamiento real de `test:e2e`/`test:db:smoke` en este commit: **[PENDIENTE: ejecutar contra una base Postgres/Redis desechable para confirmar]**.
- Cobertura de código (`npm run test:cov`) no se ejecutó en este análisis: **[PENDIENTE: no hay número de cobertura verificado para este commit]**.
- Compatibilidad de la versión instalada de `typeorm` (`1.1.0`, ver `package-lock.json`) con el resto del ecosistema TypeORM a largo plazo: **[PENDIENTE: no hay evidencia en el repositorio de por qué se fijó este rango de versión ni de problemas conocidos con ella; el build y los tests pasan hoy]**.
