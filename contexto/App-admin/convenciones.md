# convenciones.md

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

## PATRONES OBSERVADOS

### Naming y organización de archivos

- Archivos de convención de Next.js en minúscula (`page.tsx`, `layout.tsx`,
  `loading.tsx`, `route.ts`), impuestos por el framework.
- Componentes propios en **kebab-case** (`driver-list.tsx`,
  `driver-review-dialog.tsx`, `app-shell.tsx`, `status-badge.tsx`,
  `brand-logo.tsx`), exportando una función **PascalCase** con el mismo
  nombre semántico (`DriverList`, `AppShell`, `StatusBadge`).
- Route groups entre paréntesis para separar layouts sin afectar la URL:
  `(admin)/`, `(auth)/` (`src/app/`).
- Rutas API bajo `src/app/api/` en inglés (`session/login`,
  `admin/drivers/[driverProfileId]/approve`), mientras que las rutas de
  UI están en español (`conductores`, `analitica`) porque son visibles al
  usuario final.
- Alias de imports `@/*` → `./src/*` (`tsconfig.json:22`), usado de forma
  consistente en todo el repo en vez de rutas relativas largas.

### Componentes cliente vs. servidor

- `"use client"` como primera línea en todo componente con estado o
  interacción (`login-form.tsx`, `app-shell.tsx`, `driver-*.tsx`).
  Páginas y layouts que solo leen cookies/redirigen se mantienen como
  Server Components (`(admin)/layout.tsx`, `page.tsx` raíz).
- Los componentes de datos (`driver-list.tsx`, `driver-detail.tsx`,
  `driver-overview.tsx`) siguen el mismo patrón: `useState` con una unión
  discriminada `{ status: "loading" | "error" | "ready", ... }`, `fetch`
  con `cache: "no-store"`, y redirección a `/login` si la respuesta es
  401/403.

### Capa `lib/api`

- `backend.ts` marcado `import "server-only"` — concentra toda la lógica
  de sesión (cookies), llamadas al backend y helpers de respuesta
  (`apiUnavailableResponse`, `passthroughJson`, `readErrorMessage`). No se
  llama al backend desde ningún otro archivo fuera de `route.ts`.
- `types.ts` — únicamente `interface`/`type` planos que reflejan las
  respuestas del backend; no hay clases ni decoradores.
- `validation.ts` — validación manual con `typeof`, regex y funciones
  puras (`isUuidV4`, `normalizePhone`, `cleanReason`, `parseRejectPayload`,
  `parseSuspendPayload`); no se usa ninguna librería de esquemas (zod, yup,
  class-validator).

### Route handlers (BFF)

Patrón repetido en los 8 route handlers de `src/app/api/`:

1. Mutaciones (`PATCH`/`POST`) primero verifican
   `isSameOriginMutation(request)` y devuelven 403 si falla.
2. Params dinámicos (`driverProfileId`) se validan con `isUuidV4` antes de
   tocar el backend, devolviendo 400 si no son válidos.
3. Toda llamada al backend está envuelta en `try/catch`, cayendo a
   `apiUnavailableResponse()` (503) si falla la conexión.
4. La respuesta exitosa se retransmite tal cual con `passthroughJson()`
   (status + body del backend), salvo en `login` y `overview`, que
   construyen su propia respuesta.

### Error handling

- No hay `error.tsx` ni `not-found.tsx` en `src/app/`. Los errores se
  manejan por componente vía el estado `{ status: "error", message }` y un
  botón "Reintentar".
- Mensajes de error orientados al usuario final, siempre en español.

### Validación

- Teléfonos: formato peruano estricto `+51` + 9 dígitos empezando en `9`
  (`normalizePhone`, `src/lib/api/validation.ts:10-15`).
- Contraseña: longitud 8–64 (`validPassword`).
- Motivos de observación/suspensión: longitud mínima 3 o 5, máxima 500
  (`cleanReason`).
- IDs de recurso: UUID v4 estricto (`isUuidV4`).

### Configuración

- Variables de entorno leídas directamente con `process.env.NOMBRE` en
  `backend.ts` y `route.ts` (sin librería de validación de env como
  `zod`/`t3-env`), con valores por defecto hardcodeados como fallback
  (`DEFAULT_API_BASE_URL`, `DEFAULT_TIMEOUT_MS`).
- `TUKITUKI_API_BASE_URL` se documenta explícitamente en `README.md` como
  variable **exclusiva del servidor**: no debe llevar prefijo
  `NEXT_PUBLIC_` porque el navegador solo debe hablar con las rutas BFF.

### Patrones Git observables

- Dos autores: `Juanjo`/`jjtorres-dev` (commit inicial y merge de PR) y
  `Carlitos-Omar` (feature de onboarding y fixes posteriores).
- Flujo con rama de feature (`carlos`) fusionada a `main` mediante Pull
  Request #1 (`cd2df56 Merge pull request #1 from jjtorres-dev/carlos`).
- Mensajes de commit en inglés con prefijo tipo Conventional Commits en 4
  de 6 commits: `feat: add admin driver onboarding dashboard`,
  `fix: allow dashboard access from local network`,
  `fix: harden admin login flow`, `feat: use dedicated admin authentication`.
  El primer commit ("Primer commit") está en español y no sigue el
  formato — es el `create-next-app` inicial, no representativo del patrón
  posterior.
- Commits sin body/footer; el asunto describe la intención en pocas
  palabras.

## [PENDIENTE: no se pudo determinar...]

- Convenciones de DTOs más allá de interfaces TS planas (no hay
  clases, decoradores ni esquemas de validación runtime del lado del
  contrato de API).
- Convenciones de tests: no existe ningún test en el repo.
- Convenciones de migrations: no aplica (sin base de datos en este repo).
- Naming de ramas más allá de un único ejemplo (`carlos`) — insuficiente
  para inferir una convención de equipo.
- Proceso de revisión de PR (aprobaciones requeridas, checks obligatorios):
  no hay configuración de branch protection ni plantillas de PR en el
  repo.
