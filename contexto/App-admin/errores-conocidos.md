# errores-conocidos.md

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

## Baseline de tests

No aplica: no existe ningún test ni script `test` en `package.json`.

## Baseline de TypeScript — FALLA

Ejecutado `npx tsc --noEmit` sobre el commit analizado: **9 errores**,
todos `TS2307: Cannot find module 'lucide-react' or its corresponding
type declarations`, en cada archivo que importa el paquete:
`(admin)/analitica/page.tsx`, `(auth)/login/login-form.tsx`,
`(auth)/login/page.tsx`, `app-shell.tsx`, `brand-logo.tsx`,
`driver-detail.tsx`, `driver-list.tsx`, `driver-overview.tsx`,
`driver-review-dialog.tsx`.

**Causa raíz verificada**: `lucide-react` está declarado en
`package.json` (`^1.31.0`) y en `package-lock.json`
(`node_modules/lucide-react` con versión `1.31.0` en el árbol de
paquetes), pero **no existe físicamente** en `node_modules/` de este
checkout (`node_modules/lucide-react`: No such file or directory), aunque
`node_modules` sí contiene 287 paquetes y las dependencias principales
(`next`, `react`, `react-dom`, `typescript`, `eslint`, `tailwindcss`). Es
una instalación desincronizada respecto al lockfile, no un problema del
código fuente.

Reproducible con:
```bash
npx tsc --noEmit
```

No se ejecutó `npm install`/`npm ci` para corregirlo (fuera del alcance
de esta tarea de documentación).

**Actualización (ADMIN-DRIVER-R1B)**: se ejecutó `npm install` en este
checkout para poder validar `lint`/`build` de la integración de Storage;
reconcilió `node_modules` (gitignored, no commiteado) contra el
`package-lock.json` ya existente sin modificar `package.json` ni
`package-lock.json` (`git diff` vacío sobre ambos). `lint`/`build`/`tsc`
pasan limpios en este entorno desde entonces. Sigue siendo un riesgo
latente para un clon nuevo del repo si quien lo configura no corre
`npm ci`/`npm install` antes de `next build`.

## Baseline de build — FALLA

Ejecutado `npm run build` sobre el commit analizado: **falla** con el
mismo error de módulo no encontrado (`Module not found: Can't resolve
'lucide-react'`), en la misma lista de archivos. Mismo origen que el
fallo de TypeScript.

## Baseline de lint

Ejecutado `npm run lint` sobre el commit analizado: **pasa sin errores ni
warnings**. El lint no depende de la resolución de tipos de
`lucide-react` de la misma forma que `tsc`/`next build`.

## Comentarios FIXME/TODO

Búsqueda sobre `src/` (excluyendo `node_modules`/`.next`) por los
patrones `TODO`, `FIXME`, `XXX`, `HACK`: **sin resultados relevantes**
(las dos coincidencias de "todo" son texto en español dentro de strings
de UI — "Todos", "revisaste todo el expediente" —, no marcadores de
pendientes).

## Gotchas verificables

- **`lucide-react` no instalado pese a estar declarado** — ver arriba;
  bloquea `tsc` y `next build` hoy. Cualquiera que clone el repo y corra
  `npm ci` limpio probablemente no vería este problema; es específico del
  estado actual de `node_modules` en este checkout.
- **`AGENTS.md`/`CLAUDE.md` advierten explícitamente** que esta versión
  de Next.js (16.3.0) puede tener APIs y convenciones distintas a las que
  un modelo de IA "recuerda" de su entrenamiento, y piden leer
  `node_modules/next/dist/docs/` antes de escribir código nuevo. El propio
  `AGENTS.md` aclara que ese bloque lo regenera `next dev`
  (`node_modules/next/dist/server/lib/generate-agent-files.js`): quitarlo
  de un diff solo recrea el cambio sin commitear; conviene comitearlo tal
  cual si aparece modificado.
- **`allowedDevOrigins` en `next.config.ts` tiene una IP fija**
  (`10.70.82.181`, agregada en el commit `ed537cb`). Solo permite acceso
  a `next dev` desde esa dirección de red local específica; otro
  desarrollador en una LAN distinta necesitará agregar su propia IP para
  reproducir el mismo fix.
- **`README.md:64` referencia una ruta con un typo de carpeta**:
  `../tuki-backent/docs/backend-readiness-dashboard-bi.md`. El repositorio
  hermano observado en el sistema de archivos se llama `tukituki-backend`,
  no `tuki-backent`; el enlace relativo tal como está escrito no resuelve.
- **La cookie `tt_admin_identity` no está firmada ni cifrada** (es JSON en
  base64url). `readAdminIdentity()` sí valida su forma (`id`,
  `phoneE164`, `roles`, y que `isAdminUser` sea verdadero) antes de
  confiar en ella, pero un valor manipulado por el cliente que pase esa
  validación de forma podría alterar qué se muestra en la UI (p. ej. el
  badge "SA"). La autorización real de cada operación sigue dependiendo
  del access token Bearer validado por el backend, no de esta cookie.
- **Windows Application Control bloquea binarios nativos no firmados/no
  aprobados en esta máquina de desarrollo** (visible como "Una directiva
  de Control de aplicaciones bloqueó este archivo"). Verificado con dos
  síntomas distintos: (1) bloquea `node_modules/@next/swc-win32-x64-msvc`,
  forzando a Next a caer a bindings WASM; Turbopack (`next dev`/`next
  build` por defecto en Next 16) requiere bindings nativos y falla duro
  con ese fallback — el workaround verificado es agregar `--webpack`
  (`npx next dev --webpack`, `npx next build --webpack`); (2) bloquea la
  DLL nativa `_imaging` de Pillow (Python) tras instalarlo con `pip`. No
  es un bug de ninguno de los dos proyectos — es una política del sistema
  operativo de esta máquina específica, y puede no aplicar en otro
  entorno (CI, otra máquina Windows, macOS/Linux).
- **(RESUELTO, ya en `main` desde `RELEASE-R2` — commit `989ffc42`) El
  modal de preview de documentos podía quedar colgado indefinidamente en
  "Obteniendo documento..."**. Causa raíz: la ruta BFF
  `src/app/api/admin/documents/[documentId]/download-url/route.ts` hacía
  `fetch(downloadUrl, {method:"GET"})`, leía solo las cabeceras y llamaba
  `response.body.cancel()` sin drenar el resto — verificado de forma
  aislada (fuera de este repo) que ese patrón deja el socket TLS en un
  estado inconsistente en esta versión de Node/undici (el proceso llega a
  abortar con un fallo de aserción nativo de libuv al cerrarlo). Fix:
  pedir `Range: bytes=0-0` (respuesta mínima, siempre se drena entera con
  `arrayBuffer()`, nunca queda una lectura a medias) más
  `AbortSignal.timeout(5000)` en el BFF y `AbortSignal.timeout(15000)` en
  el cliente como red de seguridad adicional. Validado manualmente contra
  Railway staging con un PDF y dos JPG/PNG sintéticos reales: ambos casos
  cargan correctamente, sin cuelgue.

## Configuraciones delicadas

- **`secure` de las cookies de sesión depende de `NODE_ENV==="production"`**
  (`cookieOptions`, `src/lib/api/backend.ts:35-44`). Correr `npm run
  start` localmente sin HTTPS marcaría las cookies como `secure` y el
  navegador las descartaría, rompiendo el login. No verificado en esta
  sesión porque el build actual falla antes de llegar a `start`.
- **`TUKITUKI_API_BASE_URL` no debe llevar prefijo `NEXT_PUBLIC_`**
  (advertencia explícita en `README.md:36-38`); si alguien la renombra
  con ese prefijo por costumbre de Next.js, el valor quedaría expuesto en
  el bundle del cliente sin que el resto del BFF lo use de esa forma.
- **Tailwind v4 sin archivo de configuración explícito**: el proyecto
  depende del comportamiento por defecto de `@tailwindcss/postcss`. Toda
  personalización de tema deberá decidir entre introducir
  `tailwind.config.*` o usar el mecanismo `@theme` de Tailwind v4 dentro
  de `globals.css`.

## Incompatibilidades

[PENDIENTE: no se detectó ninguna incompatibilidad de versiones entre
dependencias declaradas; el único problema verificado es la instalación
desincronizada de `lucide-react` descrita arriba, que es un problema de
estado de `node_modules`, no de compatibilidad entre versiones.]

## Limitaciones técnicas demostrables

- No hay tests automatizados de ningún tipo.
- No hay CI/CD ni Dockerfile.
- El módulo de Analítica & BI no tiene datos reales conectados; es
  intencionalmente un placeholder (ver `decisiones.md`), no un bug, pero
  cualquier trabajo sobre esa página debe asumir que no hay contrato de
  API todavía.
- No se observa límite de intentos de login (rate limiting/lockout) en
  `src/app/api/session/login/route.ts`; cada intento simplemente reenvía
  la petición al backend.
