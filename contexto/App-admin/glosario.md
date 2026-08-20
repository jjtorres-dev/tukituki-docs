# glosario.md

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

## Entidades (`src/lib/api/types.ts`)

- **PublicUser** — usuario del backend expuesto al panel: `id`,
  `phoneE164`, `roles`, `status`, `isPhoneVerified`, `createdAt`.
- **AdminIdentity** — subconjunto de `PublicUser` guardado en la cookie de
  sesión del panel (`id`, `phoneE164`, `roles`).
- **DriverProfile** — expediente principal de un conductor: datos
  personales, documento de identidad, estado, motivo de rechazo/suspensión,
  fechas de envío/aprobación/suspensión y quién aprobó/suspendió.
- **DriverVehicle** — la mototaxi asociada al conductor: placa, marca,
  modelo, año, color, número de motor y chasis, estado.
- **DriverDocument** — un documento individual del expediente (ver
  `DriverDocumentType`), con archivo, vigencia y estado de revisión.
- **AdminDriverListItem** / **AdminDriverListResponse** — fila resumida de
  la bandeja de conductores y su respuesta paginada (`page`, `limit`,
  `total`, `totalPages`).
- **AdminDriverDetail** — expediente completo: `user` + `profile` +
  `vehicle` + `documents[]`.
- **DriverOverviewResponse** — resumen agregado para el Centro de control:
  `counts` por estado, `recentPending`, `generatedAt`.
- **RejectDriverPayload** — cuerpo de la observación de un expediente:
  `profileReason`, `vehicleReason` y/o `documents[]` (cada uno con
  `documentId` + `reason`).
- **LoginResponse** — respuesta del backend al autenticar: tokens, sesión y
  `PublicUser`.
- **ApiErrorPayload** — forma esperada de un error del backend
  (`message`, `error`, `statusCode`).

## Enums / estados

- **UserRole**: `PASSENGER` | `DRIVER` | `ADMIN` | `SUPER_ADMIN`.
- **UserStatus**: `PENDING` | `ACTIVE` | `SUSPENDED` | `DELETED`.
- **DriverStatus**: `DRAFT` | `PENDING_REVIEW` | `APPROVED` | `REJECTED` |
  `SUSPENDED` — ciclo de vida del expediente de un conductor.
- **VehicleStatus**: `DRAFT` | `PENDING_REVIEW` | `APPROVED` | `REJECTED` |
  `SUSPENDED` — ciclo de vida de la mototaxi registrada.
- **DriverDocumentStatus**: `DRAFT` | `PENDING_REVIEW` | `APPROVED` |
  `REJECTED` — ciclo de vida de cada documento.
- **DriverDocumentType**: `DNI_FRONT`, `DNI_BACK`, `DRIVER_LICENSE`,
  `VEHICLE_REGISTRATION`, `SOAT`, `PROFILE_PHOTO` — los seis tipos que el
  backend legacy todavía puede devolver. En `main`@`cd2df561` (commit
  original de este documento) la UI todavía los mostraba todos ("X de 6
  documentos registrados", `driver-detail.tsx:206` en ese commit). Desde
  `RELEASE-R2` (`main`@`989ffc42`), la UI ya filtra a solo 3 — ver
  `TARGET_DOCUMENT_TYPES` más abajo y `decisiones.md`.
- **documentType** (de `DriverProfile`, distinto de `DriverDocumentType`):
  `DNI` | `FOREIGNER_CARD` | `PASSPORT` — tipo de documento de identidad
  del conductor.
- **vehicleType**: único valor observado, `MOTOTAXI`.

## Conceptos internos / siglas

- **BFF (Backend for Frontend)** — capa de route handlers en
  `src/app/api/` que media entre el navegador y el backend real; ver
  `arquitectura.md`.
- **Expediente** — término de dominio usado en la UI en español para
  referirse al conjunto perfil + mototaxi + documentos de un conductor
  (`AdminDriverDetail`).
- **Conductor** — usuario con perfil `DriverProfile`, candidato u operador
  de mototaxi.
- **Administrador / Superadministrador** — roles `ADMIN` / `SUPER_ADMIN`
  con acceso al panel; `AppShell` distingue visualmente ambos ("AD"/"SA").
- **DNI** — Documento Nacional de Identidad (Perú), uno de los tipos de
  `documentType`/`DriverDocumentType`.
- **SOAT** — Seguro Obligatorio de Accidentes de Tránsito (Perú), uno de
  los `DriverDocumentType`.
- **phoneE164** — número de teléfono en formato E.164; en este repo se
  restringe además al patrón peruano `+51` seguido de 9 dígitos que
  empiezan en `9` (`normalizePhone`, `src/lib/api/validation.ts`).
- **Cookies de sesión**: `tt_admin_access`, `tt_admin_refresh`,
  `tt_admin_identity` (`SESSION_COOKIES`, `src/lib/api/backend.ts`).

## Preview de documentos Storage (ya en `main` desde `RELEASE-R2`, commit `989ffc42`)

- **TARGET_DOCUMENT_TYPES** — constante (`src/lib/driver-documents.ts`)
  con los 3 tipos de documento objetivo del expediente, en el orden en
  que se muestran: `DRIVER_LICENSE`, `SOAT`, `VEHICLE_REGISTRATION`.
- **DocumentPreviewDialog** — modal (`src/components/drivers/document-preview-dialog.tsx`)
  que pide la URL temporal de un documento al pulsar "Ver" y la muestra
  dentro del expediente (`<img>` o `<iframe>` según `Content-Type`).
- **BFF document download URL** — ruta `GET /api/admin/documents/:documentId/download-url`
  (`src/app/api/admin/documents/[documentId]/download-url/route.ts`);
  intermedia entre el navegador y `GET /storage/admin/documents/:id/download-url`
  del backend, agregando `contentType` a la respuesta.
- **isLegacyUrl** — campo de `AdminDocumentDownloadUrlResponse` (igual que
  en el backend): `true` si el documento todavía no migró a Railway
  Storage y `downloadUrl` es el `fileUrl` legacy tal cual, sin expiración
  administrada.
