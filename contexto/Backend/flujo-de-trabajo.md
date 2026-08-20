# flujo-de-trabajo

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

## Instalación

```bash
docker compose up -d      # Postgres+PostGIS (5433->5432) y Redis (6379)
npm ci
npm run migration:run
npm run seed:development
```

Requiere `.env` (copiado de `.env.example`) con todos los secretos reemplazados. El seed operativo (`seed:development`) solo se ejecuta si `NODE_ENV=development` o `test` (`src/database/seeds/seed-environment.guard.ts`); falla explícitamente en cualquier otro ambiente.

Requisitos declarados en `README.md`: Node.js 22, PostgreSQL 18 con PostGIS, Redis 8.

## Ejecución local

```bash
npm run start:dev     # nest start --watch
npm run start:debug   # con inspector
npm start             # nest start (sin watch)
npm run start:prod    # node dist/main (requiere build previo)
```

Puerto y prefijo de API configurables por entorno (`PORT`, `API_PREFIX`, default `3001` / `api/v1`). Swagger disponible en `/docs` cuando `SWAGGER_ENABLED=true` (default `true`, pero `false` en CI y en `docker-compose.prod.yml`).

## Tests

```bash
npm test                    # unitarios (Jest, src/**/*.spec.ts)
npm run test:watch
npm run test:cov            # con cobertura -> ./coverage
npm run test:debug          # unitarios con --inspect-brk
npm run test:db:smoke       # migraciones + seed doble sobre base desechable
npm run test:e2e            # app.e2e-spec.ts + ride-financial-flow.e2e-spec.ts
npm run test:integration    # test:e2e con --runInBand
```

`test:db:smoke` exige explícitamente una base de datos vacía cuyo nombre contenga `test` o `smoke` (falla con error si no, ver `test/database-readiness.e2e-spec.ts`); aplica todas las migraciones compiladas y corre el seed dos veces para verificar idempotencia. `test:e2e` usa `test/jest-e2e.json` (`testRegex: .e2e-spec.ts$`) y valida readiness HTTP y el flujo cotización → viaje → pago → comisión → liquidación (`ride-financial-flow.e2e-spec.ts`, README).

## Lint / formato

```bash
npm run lint          # eslint --fix sobre src, apps, libs, test
npm run lint:check    # eslint --max-warnings=0 sobre src, apps, libs, test, scripts (usado en CI)
npm run format        # prettier --write sobre src/**/*.ts y test/**/*.ts
```

Reglas relevantes en `eslint.config.mjs`: `typescript-eslint` recomendado con chequeo de tipos (`recommendedTypeChecked`), `prettier` integrado como regla de error, `no-explicit-any` desactivado, `no-floating-promises` y `no-unsafe-argument` como `warn` (no bloquean CI, que corre con `--max-warnings=0` — **cualquier `warn` de ESLint sí falla `lint:check`**, ya que ESLint no distingue nivel de severidad para ese flag).

## Build

```bash
npm run build   # nest build -> dist/
```

`tsconfig.build.json` extiende `tsconfig.json`. El build es prerequisito de `start:prod`, de todos los scripts `migration:*` y `seed:*` (compilan antes de ejecutar contra `dist/`), y del smoke de base de datos.

## Migraciones

```bash
npm run migration:generate   # build + typeorm migration:generate
npm run migration:run        # build + typeorm migration:run -d dist/database/data-source.js
npm run migration:run:prod   # typeorm migration:run -d dist/database/data-source.js (sin build previo)
npm run migration:revert
npm run migration:show
```

Las migraciones son un paso manual y explícito, nunca automático al iniciar una instancia (ver `README.md` y `docker-compose.prod.yml`, que no invoca ninguna migración en el `command`/`entrypoint` del servicio `api`). En despliegue, se ejecutan una sola vez antes de levantar réplicas: `docker compose -f docker-compose.prod.yml run --rm api npm run migration:run:prod`.

## Seed

```bash
npm run seed:super-admin     # build + node dist/database/seeds/super-admin.seed.js
npm run seed:development     # build + node dist/database/seeds/development-data.seed.js
```

Ambos seeds validan `NODE_ENV` antes de escribir datos (`assertDevelopmentSeedEnvironment`). El seed de desarrollo es idempotente (verificado por `test:db:smoke`, que lo ejecuta dos veces consecutivas y compara conteos).

## Contratos de API (OpenAPI / Postman)

```bash
npm run openapi:export   # ts-node scripts/export-openapi.ts
```

Requiere Postgres y Redis disponibles y variables de entorno configuradas. Genera/valida `docs/openapi.json` y `docs/tukituki.postman_collection.json` (tags, resúmenes, `operationId` y esquemas de seguridad se validan antes de escribir). CI re-ejecuta este paso en cada build para detectar contratos desactualizados.

## Git

- Rama principal: `main`. Working tree limpio al momento de este análisis.
- Estilo de commit observado: Conventional Commits en inglés (`feat:`, `fix:`, `test:`, `chore:`) — ver `convenciones.md`.
- No hay plantilla de PR, `CONTRIBUTING.md` ni hooks de commit versionados en el repo: **[PENDIENTE: definir proceso de revisión y merge del equipo]**.

## Deploy

```bash
docker build --target production -t tukituki-backend:local .
docker compose -f docker-compose.prod.yml build
docker compose -f docker-compose.prod.yml run --rm api npm run migration:run:prod
docker compose -f docker-compose.prod.yml up -d
```

GitHub Actions (`.github/workflows/cd.yml`) publica la imagen a `ghcr.io/<org>/tukituki-backend` únicamente en push de tags `v*` o disparo manual (`workflow_dispatch`); no despliega automáticamente a ningún ambiente. El pipeline de CI (`.github/workflows/ci.yml`) corre en cada `pull_request` y push a `main`/`master`: lint, tests unitarios, build, smoke de migraciones, e2e, export de contratos, build de imagen Docker.

---

## Checklist técnico antes de considerar un cambio terminado

Basado únicamente en lo que CI ejecuta (`.github/workflows/ci.yml`) y en los scripts de `package.json`:

- [ ] `npm run lint:check` sin errores ni warnings.
- [ ] `npm test -- --runInBand` (unitarios) en verde.
- [ ] `npm run build` sin errores de TypeScript.
- [ ] Si se agregó/modificó una migración: `npm run test:db:smoke` en verde contra una base desechable (nombre con `test`/`smoke`), y actualizar la aserción de conteo de migraciones en `test/database-readiness.e2e-spec.ts` si corresponde.
- [ ] `npm run test:e2e -- --runInBand` en verde.
- [ ] `npm run openapi:export` regenerado sin errores de validación si se tocó un controller/DTO expuesto por Swagger.
- [ ] Si se agregó una variable de entorno: reflejarla en `src/config/env.validation.ts` y en `.env.example`.
- [ ] `docker build --target production` construye correctamente si el cambio afecta dependencias o estructura de build.

[PENDIENTE: definir proceso humano de revisión de código, aprobaciones requeridas, y criterios de "definition of done" más allá de lo que CI valida automáticamente — no hay evidencia de esto en el repositorio]
