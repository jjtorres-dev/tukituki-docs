# arquitectura.md

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

## Stack tecnológico

- **Next.js 16.3.0** (App Router, `src/app/`), sin carpeta `pages/`.
- **React 19.2.4** / **react-dom 19.2.4**.
- **TypeScript 5.9.3**, `strict: true` (`tsconfig.json:7`).
- **Tailwind CSS v4.3.3** vía `@tailwindcss/postcss`, sin `tailwind.config.*`
  (no requerido en v4); único punto de entrada `src/app/globals.css`.
- **lucide-react ^1.31.0** — única librería de iconos usada en todo el repo.
- **ESLint 9.39.5**, flat config (`eslint.config.mjs`) sobre
  `eslint-config-next/core-web-vitals` + `eslint-config-next/typescript`.
- Gestor de paquetes: npm (`package-lock.json` presente).

## Estructura y responsabilidad de carpetas

- `src/app/(admin)/` — rutas protegidas del panel: `dashboard/`,
  `conductores/` (lista y detalle `[driverProfileId]/`), `analitica/`,
  con `layout.tsx` compartido que exige sesión (ver Autenticación).
- `src/app/(auth)/login/` — página y formulario de inicio de sesión.
- `src/app/api/` — capa **BFF** (Backend for Frontend): route handlers que
  reciben peticiones del navegador y las reenvían al backend real, sin
  exponer tokens al cliente. Incluye `session/login`, `session/logout` y
  `admin/drivers/**`.
- `src/components/` — componentes de UI (`app-shell.tsx`, `status-badge.tsx`,
  `brand-logo.tsx`) y `components/drivers/` con la lógica de la bandeja,
  el expediente y el diálogo de revisión.
- `src/lib/api/` — `backend.ts` (server-only: sesión, cookies, fetch al
  backend), `types.ts` (interfaces TS que reflejan las respuestas del
  backend) y `validation.ts` (validadores manuales de payloads/params).
- `src/lib/format.ts` — formateo de fecha/hora (`es-PE`, zona
  `America/Lima`), teléfono e iniciales.
- `src/proxy.ts` — proxy de Next.js 16 (reemplazo de `middleware.ts`,
  confirmado en `node_modules/next/dist/docs/01-app/03-api-reference/03-file-conventions/proxy.md`)
  que protege `/dashboard`, `/conductores`, `/analitica` y redirige
  `/login` si ya hay sesión.

## Módulos principales

- **Onboarding de conductores**: bandeja con filtros/paginación/búsqueda
  (`driver-list.tsx`), expediente completo con perfil/mototaxi/documentos
  (`driver-detail.tsx`), y acciones administrativas de aprobar/observar
  (rechazar)/suspender (`driver-review-dialog.tsx`).
- **Centro de control** (`dashboard/`): contadores agregados
  (pendientes/aprobados/observados/suspendidos) y últimas solicitudes
  pendientes (`driver-overview.tsx`), calculados en el propio BFF
  (`api/admin/drivers/overview/route.ts`) combinando varias llamadas al
  backend.
- **Analítica & BI** (`analitica/page.tsx`): página estática de "próxima
  fase", sin datos reales ni simulados; solo estructura visual y copy.
- **Autenticación**: login con teléfono peruano (+51) y contraseña,
  soporta tanto `fetch` JSON (progressive enhancement) como envío nativo
  de formulario (`<form method="post" action="/api/session/login">`).

## Integración con Railway Storage — ya en `main` (RELEASE-R2, 2026-08-16)

`main`@`cd2df561` (el commit original de este documento) no tenía esta
integración. Se implementó y validó manualmente contra Backend
**staging** en la rama `test/admin-driver-storage-preview` (checkpoint
ADMIN-DRIVER-R1B) y **ya se integró a `main` por fast-forward** en
`RELEASE-R2` (`main`@`989ffc42faef5788c18455993d2462285b4db18d`) — ver
`decisiones.md`. **`main` no es producción**: Railway/STAGING solo pasa a
desplegar desde `main` cuando JuanJo lo cambie manualmente; producción
sigue sin ningún cambio.

- Preview del expediente Driver reducido a los 3 documentos objetivo
  (`DRIVER_LICENSE`, `SOAT`, `VEHICLE_REGISTRATION`) más la foto del
  perfil (`profile.photoUrl`, ya resuelta por el backend).
- Nueva ruta BFF `GET /api/admin/documents/:documentId/download-url`
  (`src/app/api/admin/documents/[documentId]/download-url/route.ts`):
  llama a `backendAdminFetch('/storage/admin/documents/:id/download-url')`
  y, además, hace un segundo `fetch` server-side a la propia `downloadUrl`
  con `Range: bytes=0-0` únicamente para leer el `Content-Type` real (sin
  descargar el archivo), porque el backend no expone MIME en ningún otro
  punto del contrato.
- Componente `DocumentPreviewDialog` (`src/components/drivers/document-preview-dialog.tsx`):
  pide la URL temporal solo al pulsar "Ver" (nunca al cargar la lista ni
  el detalle), la mantiene únicamente en estado de componente (nunca en
  `localStorage`/`sessionStorage`/cookies), la descarta al cerrar el
  modal y pide una nueva al reabrir. Renderiza `<img>` para
  `image/jpeg|png|webp` o `<iframe>` para `application/pdf` (y como
  fallback genérico si el `Content-Type` no se pudo determinar). Ofrece
  "Abrir en otra pestaña" (`window.open(..., "noopener,noreferrer")`);
  deliberadamente **sin** botón "Descargar".
- El navegador carga la `downloadUrl` (presignada, S3/Railway Storage)
  **directamente**, sin pasar por el BFF — el BFF nunca reenvía el
  archivo en sí, solo la URL temporal.

## Entidades principales (`src/lib/api/types.ts`)

`PublicUser`, `AdminIdentity`, `DriverProfile`, `DriverVehicle`,
`DriverDocument`, `AdminDriverListItem` / `AdminDriverListResponse`,
`AdminDriverDetail` (agrega `user` + `profile` + `vehicle` + `documents`),
`DriverOverviewResponse`, `RejectDriverPayload`, `LoginResponse`,
`ApiErrorPayload`. Ver `glosario.md` para enums de estado.
`AdminDocumentDownloadUrlResponse` (`downloadUrl`, `expiresAt`,
`isLegacyUrl`, `contentType`) existe solo en
`test/admin-driver-storage-preview` (ver sección de Storage arriba).

## Flujo de datos

Navegador → route handler bajo `src/app/api/**` (BFF, siempre server-side)
→ `backendAdminFetch`/`backendPublicFetch` (`src/lib/api/backend.ts`) →
API REST del backend en `TUKITUKI_API_BASE_URL`. El navegador **nunca**
llama directamente al backend ni ve el access token; solo interactúa con
las rutas `/api/**` de este mismo proyecto. Las respuestas del backend se
retransmiten con `passthroughJson()` preservando status y body.

## Servicios externos

- Únicamente la **API REST del backend de TukiTuki** (`TUKITUKI_API_BASE_URL`,
  por defecto `http://localhost:3001/api/v1`). No se encontró integración
  con ningún otro servicio externo (sin SDK de analítica, error tracking,
  pagos, mapas, email, etc. en `package.json` ni en el código).

## Persistencia

No hay base de datos ni ORM en este repositorio. El único estado
persistido localmente es la **sesión en cookies `HttpOnly`**:
`tt_admin_access`, `tt_admin_refresh`, `tt_admin_identity`
(`SESSION_COOKIES`, `src/lib/api/backend.ts:11-15`). `tt_admin_identity`
guarda un JSON (`id`, `phoneE164`, `roles`) codificado en base64url, sin
firmar ni cifrar.

## Autenticación / autorización

- Login exclusivo para roles `ADMIN` y `SUPER_ADMIN` (`isAdminUser`,
  `src/lib/api/backend.ts:46-48`); si el backend autentica a un usuario sin
  esos roles, el BFF cierra la sesión recién creada en el backend y
  responde 403 (`src/app/api/session/login/route.ts:132-139`).
- Endpoint de login dedicado para administradores:
  `TUKITUKI_ADMIN_LOGIN_PATH` (por defecto `/auth/admin/login`), distinto
  del login general del backend.
- Autorización real de cada llamada ocurre en el backend vía Bearer token;
  `backendAdminFetch` refresca automáticamente el access token con el
  refresh token si recibe 401, y limpia la sesión si igual falla o recibe
  403 (`src/lib/api/backend.ts:136-176`).
- `src/proxy.ts` hace un chequeo **superficial** (presencia de cookies) para
  decidir si redirige a `/login`; no valida el token. La autorización de
  datos siempre depende del backend.
- Mutaciones (`login`, `logout`, `approve`, `reject`, `suspend`) verifican
  `isSameOriginMutation(request)` como protección tipo CSRF basada en
  cabeceras `sec-fetch-site`/`origin` (`src/lib/api/backend.ts:202-229`).

## Deploy

[PENDIENTE: no se encontró `Dockerfile`, carpeta `.github/workflows`,
`vercel.json` ni ningún otro artefacto de CI/CD en el repositorio. No hay
evidencia de una plataforma de despliegue elegida.]

## Cosas importantes que actualmente NO existen

Verificado con búsqueda en el repo (excluyendo `node_modules`/`.next`):

- **Sin tests**: no hay archivos `*.test.*`/`*.spec.*`, ni script `test`
  en `package.json`, ni framework de testing instalado.
- **Sin CI/CD**: no existe `.github/`, `Dockerfile` ni `vercel.json`.
- **Sin módulos de pasajeros, viajes ni finanzas**: el `README.md` los
  declara explícitamente como evolución futura, no habilitados aún; no
  hay rutas ni componentes para ellos.
- **Sin datos reales en Analítica & BI**: la página existe pero es un
  placeholder visual; no consume ningún endpoint de analítica.
- **Sin versión de Node.js fijada**: no hay `.nvmrc` ni campo `engines`
  en `package.json`.
- **Instalación desincronizada (reconciliada localmente, no commiteada)**:
  `lucide-react` estaba declarado en `package.json`/`package-lock.json`
  pero ausente de `node_modules` en el checkout original; se corrigió
  corriendo `npm install` durante ADMIN-DRIVER-R1B (sin tocar
  `package.json`/`package-lock.json`) — ver `errores-conocidos.md` para
  el detalle y el riesgo remanente en un clon nuevo del repo.
