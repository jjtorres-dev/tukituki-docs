# flujo-de-trabajo.md

Repositorio:
tukituki-admin-web

Branch analizada:
main

Commit analizado:
cd2df561dc37e7fcac87529658fcdba3fa500477

Última actualización:
2026-08-16

Fuente de verdad:
Este documento es contexto auxiliar. Si contradice al código actual,
el código y los tests tienen prioridad.

---

## Instalación

```bash
cp .env.example .env.local
npm ci
```

Respaldado por `README.md:23-25` y `package-lock.json` (npm). No hay
`.nvmrc` ni campo `engines` en `package.json`.
[PENDIENTE: definir versión mínima de Node.js requerida.]

Variables de entorno (`.env.example`):

```
TUKITUKI_API_BASE_URL=http://localhost:3001/api/v1
TUKITUKI_API_TIMEOUT_MS=10000
TUKITUKI_ADMIN_LOGIN_PATH=/auth/admin/login
```

`TUKITUKI_API_BASE_URL` es exclusiva del servidor (no debe llevar prefijo
`NEXT_PUBLIC_`); el navegador solo habla con las rutas BFF de este
proyecto (`README.md:36-38`).

## Ejecución local

```bash
npm run dev
```

Ejecuta `next dev` (`package.json:6`). Requiere que el backend esté
disponible en `TUKITUKI_API_BASE_URL` con un usuario administrativo
válido; el panel no incluye credenciales de demostración ni datos
simulados (`README.md:40-41`).

## Lint

```bash
npm run lint
```

Ejecuta `eslint` (`package.json:9`, flat config `eslint.config.mjs`).
**Verificado sobre el commit analizado: pasa sin errores.**

## Type-check

No hay script `type-check` en `package.json`. Se ejecuta manualmente:

```bash
npx tsc --noEmit
```

**Verificado sobre el commit analizado: FALLA con 9 errores** (`TS2307:
Cannot find module 'lucide-react'`) — estado del checkout original, antes
de reconciliar `node_modules`. Causa raíz y detalle en
`errores-conocidos.md`. **Actualización (ADMIN-DRIVER-R1B)**: tras correr
`npm install` (sin tocar `package.json`/`package-lock.json`), pasa sin
errores en este entorno.

## Build

```bash
npm run build
```

Ejecuta `next build` (`package.json:7`), que en Next 16.3.0 usa Turbopack
por defecto. **Verificado sobre el commit analizado: FALLA** originalmente
por el mismo motivo que el type-check (módulo `lucide-react` no
resuelto); tras `npm install` ese problema desaparece, pero en esta
máquina Windows específica una directiva de Control de aplicaciones
bloquea el binario nativo de Turbopack (`@next/swc-win32-x64-msvc`) y el
build falla igual por eso — workaround verificado:

```bash
npx next build --webpack
```

(mismo mecanismo para `next dev`: `npx next dev --webpack`). No se
modificó `package.json` por esta limitación local — ver
`errores-conocidos.md`.

## Start (producción)

```bash
npm run start
```

Ejecuta `next start` (`package.json:8`); requiere un build previo exitoso.
No verificado en esta sesión porque `npm run build` falla actualmente.

## Tests

[PENDIENTE: no existe script `test` en `package.json`, ni archivos
`*.test.*`/`*.spec.*`, ni framework de testing (Jest, Vitest, Playwright,
etc.) instalado.]

## Migraciones / seed

No aplica: este repositorio no tiene acceso a base de datos ni ORM. La
gestión de datos vive en el backend (`tukituki-backend`, repositorio
separado).

## Git

- Historial: 6 commits en `main`, incluyendo un merge de PR (`cd2df56`
  "Merge pull request #1 from jjtorres-dev/carlos").
- Dos autores han contribuido (`Juanjo`/`jjtorres-dev`, `Carlitos-Omar`).
- Flujo observado: rama de feature (`carlos`) → Pull Request → merge a
  `main`. No hay evidencia de reglas de branch protection, CODEOWNERS ni
  plantilla de PR en el repo.
- Mensajes de commit mayormente en inglés con prefijo `feat:`/`fix:` (ver
  `convenciones.md`).
[PENDIENTE: definir proceso formal de revisión de PR del equipo — número
de aprobaciones requeridas, checks obligatorios, etc.]

## Deploy

[PENDIENTE: no hay pipeline de CI/CD (`.github/` no existe), ni
`Dockerfile`, ni `vercel.json` en el repo. No hay evidencia de una
plataforma de despliegue elegida.]

---

## Checklist técnico antes de considerar un cambio terminado

Basado únicamente en lo que los scripts del repo permiten verificar hoy:

- [ ] `npm run lint` pasa sin errores.
- [ ] `npx tsc --noEmit` pasa sin errores (requiere `node_modules`
      reconciliado con el lockfile — ver `errores-conocidos.md`).
- [ ] `npm run build` completa sin errores. En esta máquina Windows usar
      `npx next build --webpack` si Turbopack falla por la directiva de
      Control de aplicaciones (ver `errores-conocidos.md`).
- [ ] `npm run dev` (o `npx next dev --webpack`) levanta la app y la
      página modificada carga en el navegador sin errores de consola.

No incluido por falta de soporte en el repo:

- [PENDIENTE: no hay tests automatizados que correr.]
- [PENDIENTE: no hay pipeline de CI que valide el cambio antes de
  mergear.]
- [PENDIENTE: no hay proceso de revisión de PR documentado más allá del
  uso observado de Pull Requests en GitHub.]
