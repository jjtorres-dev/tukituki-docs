# decisiones.md

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

## Preview de documentos Driver: solo 3 documentos objetivo, ocultos los legacy

Estado:
ACTIVA (checkpoint ADMIN-DRIVER-R1B; **ya en `main`** desde `RELEASE-R2`,
commit `989ffc42`)

Qué se decidió:
El expediente Driver muestra únicamente `DRIVER_LICENSE`, `SOAT` y
`VEHICLE_REGISTRATION`, en ese orden (Licencia de conducir, SOAT, Tarjeta
de propiedad/TIV). `DNI_FRONT`, `DNI_BACK` y `PROFILE_PHOTO` quedan
excluidos de la UI (lista de documentos y del diálogo de rechazo por
documento), aunque el backend legacy siga devolviéndolos y sin borrar
esos registros.

Por qué:
Decisión de producto ya cerrada en `docs/contexto/estado-proyecto.md`
(reducción del expediente Driver de 6 a 3 documentos objetivo).

Evidencia:
`src/lib/driver-documents.ts` (`TARGET_DOCUMENT_TYPES`,
`filterTargetDocuments`), usado en `driver-detail.tsx` y pasado a
`driver-review-dialog.tsx`.

---

## Preview de documentos: modal interno, sin botón Descargar

Estado:
ACTIVA (ya en `main` desde `RELEASE-R2`, commit `989ffc42`; implementada originalmente en `test/admin-driver-storage-preview`)

Qué se decidió:
Al pulsar "Ver" sobre un documento, se abre un modal (`DocumentPreviewDialog`)
que renderiza el archivo dentro del propio expediente (`<img>` para
imágenes, `<iframe>` para PDF) en vez de navegar fuera. El modal ofrece
"Cerrar" y "Abrir en otra pestaña" (`window.open(url, "_blank",
"noopener,noreferrer")`); deliberadamente **no** ofrece ningún botón
"Descargar" ni usa el atributo `download`.

Por qué:
Decisión UX cerrada explícitamente para este checkpoint: no incentivar la
descarga de documentos de identidad/vehículo desde el panel.

Alternativas descartadas:
Abrir el `fileUrl`/`downloadUrl` directamente en pestaña nueva como único
mecanismo (comportamiento previo de `DocumentCard`, sin preview interno)
— descartado porque sacaba al admin del expediente y no distinguía
imagen de PDF.

Evidencia:
`src/components/drivers/document-preview-dialog.tsx`.

---

## URL de descarga: siempre bajo demanda ("lazy"), nunca persistida

Estado:
ACTIVA (ya en `main` desde `RELEASE-R2`, commit `989ffc42`; implementada originalmente en `test/admin-driver-storage-preview`)

Qué se decidió:
La URL temporal de un documento se solicita únicamente cuando el admin
pulsa "Ver" — nunca al cargar la lista de conductores ni el detalle del
expediente. Se mantiene solo en el estado de React del modal; se
descarta al cerrar (el componente se desmonta) y se vuelve a solicitar
desde cero al reabrir. No se guarda en `localStorage`, `sessionStorage`,
cookies, ni en ningún estado persistente del lado del servidor.

Por qué:
Minimiza URLs firmadas innecesarias y exposición, coherente con el TTL de
15 minutos que ya maneja el backend (`STORAGE_DOWNLOAD_URL_TTL_SECONDS`).

Evidencia:
`src/components/drivers/document-preview-dialog.tsx` (estado `useState`
sin persistencia), `src/app/api/admin/documents/[documentId]/download-url/route.ts`
(`Cache-Control: no-store`).

---

## Content-Type del documento: leído del recurso real, nunca inferido

Estado:
ACTIVA (ya en `main` desde `RELEASE-R2`, commit `989ffc42`; implementada originalmente en `test/admin-driver-storage-preview`)

Qué se decidió:
El backend no expone MIME/`Content-Type` en ningún punto del contrato de
Storage (ni en `DriverDocumentResponseDto` ni en la respuesta de
`download-url`). En vez de inferirlo desde el `fileObjectKey` (su
extensión) o desde `DriverDocumentType` (las 3 categorías objetivo
aceptan los mismos 4 MIME types, así que el tipo de documento no alcanza
para decidir), el BFF pide `Range: bytes=0-0` a la propia `downloadUrl`
temporal y lee el `Content-Type` real de la respuesta.

Por qué:
Evita "inventar" el tipo de archivo; usa el mismo dato que el navegador
vería si cargara el recurso directamente, sin necesidad de descargarlo
completo ni de tocar el contrato del backend.

Alternativas descartadas:
Inferir MIME desde la extensión del `objectKey` — descartado
explícitamente (el propio `objectKey` no debería ser un dato que Admin
Web necesite consumir). Pedir a Backend que agregue MIME al contrato —
evaluado y descartado por innecesario una vez confirmado que `Range:
bytes=0-0` funciona de forma fiable contra Railway Storage (S3-compatible).

Evidencia:
`src/app/api/admin/documents/[documentId]/download-url/route.ts`
(`peekContentType`).

---

## Decisión: patrón Backend for Frontend (BFF)

Estado:
ACTIVA

Qué se decidió:
Todas las llamadas al backend pasan por route handlers de Next.js bajo
`src/app/api/`, ejecutados server-side. El navegador nunca llama
directamente a `TUKITUKI_API_BASE_URL` ni recibe el access token.

Por qué:
Declarado explícitamente en `README.md`: "Backend for Frontend para evitar
exponer tokens al navegador".

Alternativas descartadas:
[PENDIENTE: sin evidencia de que se evaluara consumir el backend
directamente desde el cliente.]

Evidencia:
`README.md:10`, `src/lib/api/backend.ts:1` (`import "server-only"`),
todos los archivos `src/app/api/**/route.ts`.

---

## Decisión: sesión en cookies `HttpOnly`, no en localStorage

Estado:
ACTIVA

Qué se decidió:
Access token, refresh token e identidad del admin se guardan en cookies
`HttpOnly`, `SameSite=Lax`, `secure` en producción.

Por qué:
`README.md:9` declara "JWT de acceso y renovación almacenados en cookies
`HttpOnly`" como parte del alcance del proyecto.

Alternativas descartadas:
[PENDIENTE: sin evidencia de que se evaluara localStorage/sessionStorage.]

Evidencia:
`src/lib/api/backend.ts:35-44` (`cookieOptions`), `:50-73`
(`saveLoginSession`).

---

## Decisión: endpoint de login administrativo dedicado

Estado:
ACTIVA

Qué se decidió:
El login del panel usa `/auth/admin/login` (configurable vía
`TUKITUKI_ADMIN_LOGIN_PATH`), en lugar del `/auth/login` genérico usado en
la primera versión.

Por qué:
[PENDIENTE: el commit no incluye una justificación textual; el mensaje
solo describe el cambio.]

Alternativas descartadas:
El endpoint genérico `/auth/login`, usado previamente (ver diff).

Evidencia:
Commit `32c35db` "feat: use dedicated admin authentication" — cambia
`backendPublicFetch("/auth/login", ...)` por
`backendPublicFetch(ADMIN_LOGIN_PATH, ...)` en
`src/app/api/session/login/route.ts`, y agrega
`TUKITUKI_ADMIN_LOGIN_PATH` a `.env.example`.

---

## Decisión: allowlist de roles ADMIN/SUPER_ADMIN verificada en el servidor

Estado:
ACTIVA

Qué se decidió:
Tras un login exitoso contra el backend, el BFF verifica
`isAdminUser(login.user)`; si el usuario autenticado no tiene rol `ADMIN`
ni `SUPER_ADMIN`, se cierra la sesión recién creada en el backend
(`POST /auth/logout`) y se responde 403, sin guardar cookies.

Por qué:
[PENDIENTE: sin comentario explícito; se infiere del propio código que es
una segunda barrera de autorización más allá de lo que el backend permita
autenticar.]

Alternativas descartadas:
[PENDIENTE: sin evidencia.]

Evidencia:
`src/lib/api/backend.ts:17,46-48` (`ADMIN_ROLES`, `isAdminUser`),
`src/app/api/session/login/route.ts:132-139`.

---

## Decisión: `isSameOriginMutation` como protección tipo CSRF

Estado:
ACTIVA

Qué se decidió:
Toda ruta que muta estado (`login`, `logout`, `approve`, `reject`,
`suspend`) rechaza la petición con 403 si las cabeceras `sec-fetch-site`
u `origin` no coinciden con el host público de la petición, en lugar de
usar un token CSRF explícito.

Por qué:
[PENDIENTE: sin comentario en el código; consistente con el endurecimiento
de login del commit `21ea9f0`.]

Alternativas descartadas:
[PENDIENTE: sin evidencia de que se evaluara un token CSRF dedicado.]

Evidencia:
`src/lib/api/backend.ts:202-229`, usado en los 5 route handlers de
mutación.

---

## Decisión: soporte de envío nativo de formulario además de `fetch` JSON

Estado:
ACTIVA

Qué se decidió:
`POST /api/session/login` acepta tanto `application/json` (usado por
`login-form.tsx` vía `fetch`) como `application/x-www-form-urlencoded` /
`multipart/form-data` (envío nativo del `<form>`), respondiendo con
redirect 303 en el segundo caso.

Por qué:
[PENDIENTE: sin comentario explícito; el commit que lo introduce se llama
"fix: harden admin login flow", sugiriendo robustez ante fallos de
JavaScript, pero el motivo exacto no está documentado.]

Alternativas descartadas:
[PENDIENTE: sin evidencia.]

Evidencia:
Commit `21ea9f0` "fix: harden admin login flow";
`src/app/api/session/login/route.ts:36-61,63-72` (`readLoginRequest`,
`nativeRedirect`).

---

## Decisión: `allowedDevOrigins` con IP fija para acceso desde red local

Estado:
ACTIVA (config de desarrollo)

Qué se decidió:
`next.config.ts` declara `allowedDevOrigins: ["10.70.82.181"]` para que
`next dev` acepte peticiones desde esa IP de red local.

Por qué:
Mensaje del commit: "fix: allow dashboard access from local network".

Alternativas descartadas:
[PENDIENTE: sin evidencia.]

Evidencia:
Commit `ed537cb` "fix: allow dashboard access from local network",
`next.config.ts:4`.

Nota:
La IP está hardcodeada; solo sirve para quien desarrolla desde esa
dirección específica. Ver `errores-conocidos.md`.

---

## Decisión: proxy.ts como reemplazo de middleware.ts

Estado:
ACTIVA

Qué se decidió:
El control de rutas protegidas se implementa en `src/proxy.ts`, no en
`middleware.ts`.

Por qué:
Requisito del framework: Next.js 16 renombró y deprecó la convención
`middleware.ts` en favor de `proxy.ts`.

Alternativas descartadas:
`middleware.ts` (convención deprecada).

Evidencia:
`src/proxy.ts`,
`node_modules/next/dist/docs/01-app/03-api-reference/03-file-conventions/proxy.md:11`
("The `middleware` file convention is deprecated and has been renamed to
`proxy`").

---

## Decisión: sin librería de validación de esquemas ni de estado remoto

Estado:
ACTIVA (por omisión)

Qué se decidió:
La validación de payloads se hace a mano (`src/lib/api/validation.ts`) y
el fetching de datos usa `fetch` + `useState`/`useEffect` directamente,
sin zod/yup ni SWR/React Query/Redux/Zustand.

Por qué:
[PENDIENTE: sin evidencia de una decisión explícita; se infiere de la
ausencia de estas dependencias en `package.json`.]

Alternativas descartadas:
[PENDIENTE: sin evidencia.]

Evidencia:
`package.json` (solo `next`, `react`, `react-dom`, `lucide-react` como
dependencias de producción).

---

## Decisión: Analítica & BI sin datos simulados

Estado:
ACTIVA

Qué se decidió:
La página `/analitica` muestra únicamente estructura visual y roadmap;
explícitamente no renderiza datos de ejemplo ni mock mientras el backend
no exponga agregados reales.

Por qué:
`README.md:15`: "Estructura visual preparada para Analítica & BI, sin
datos simulados".

Alternativas descartadas:
Mostrar datos de ejemplo/mock mientras se define el contrato del backend
(descartado explícitamente).

Evidencia:
`README.md:15`, `src/app/(admin)/analitica/page.tsx` (badge "Sin datos
simulados", copy "Esperando contrato de datos").

---

## Decisión pendiente: plataforma y pipeline de deploy

Estado:
PENDIENTE

Qué se decidió:
No hay decisión registrada.

Por qué:
No aplica (decisión no tomada).

Alternativas descartadas:
No aplica.

Evidencia:
Ausencia de `.github/workflows`, `Dockerfile`, `vercel.json` en el repo.

---

## Decisión pendiente: framework de testing

Estado:
PENDIENTE

Qué se decidió:
No hay framework de testing elegido ni instalado.

Por qué:
No aplica (decisión no tomada).

Alternativas descartadas:
No aplica.

Evidencia:
`package.json` no declara ningún framework de test; no existe ningún
archivo `*.test.*`/`*.spec.*` en el repo.
