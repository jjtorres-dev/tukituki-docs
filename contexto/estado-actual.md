# TukiTuki — Estado del proyecto

Última actualización:
2026-08-20

Estado:
ACTIVO

Proyecto:
TukiTuki

Documento:
Contexto transversal de producto y estado del proyecto — **solo el presente**

Fuente de verdad técnica:
Repositorios + tests + configuración + Git

**Nota (2026-08-20)**: este documento se dividió en dos. `estado-actual.md`
(este archivo) es solo el presente — qué existe hoy en cada componente,
qué falta, qué está bloqueado y por qué, qué decisiones de producto
faltan. El registro completo de checkpoints (qué se hizo, cuándo, con
qué evidencia, desde el inicio del proyecto) vive en
`historial-checkpoints.md`, que solo crece por el final y nunca se edita
hacia atrás. Cuando un bloque de este documento mezclaba estado vigente
con la historia de cómo se llegó a él, la historia completa se trasladó
íntegra a `historial-checkpoints.md` y aquí quedó una frase corta con el
hecho vigente, remitiendo al historial para el detalle.

---

## 1. Fuente de verdad

- El código, los tests, la configuración y el historial Git de los tres repositorios (`tukituki-backend`, `tukituki-driver-app`, `tukituki-passenger-app`) son la **verdad técnica actual**. Ante cualquier duda, se verifica ahí, no en este documento.
- `docs/contexto/Backend/*`, `docs/contexto/App-driver/*` y `docs/contexto/App-passenger/*` son **contexto auxiliar técnico por repositorio**, generado por lectura de código en el commit indicado en cada uno.
- `estado-proyecto.md` (este documento) es **contexto transversal de producto**: además del estado técnico, registra decisiones de producto aprobadas que todavía no están reflejadas en el código.
- Regla de todo el documento: cuando el código actual y una decisión de producto difieren, se registran **ambas**, explícitamente etiquetadas `ACTUAL` (lo que hace el código hoy) y `OBJETIVO APROBADO` (lo decidido, aún no implementado). Nunca se presenta un objetivo aprobado como si ya estuviera implementado.

## 2. Repositorios y versiones oficiales

| Componente | Repo | Branch | Commit oficial |
|---|---|---|---|
| Backend | `tukituki-backend` | `main` | `29fe187aa31f5aad2db20ecd49534a218b358ab6` (`29fe187a`) |
| Admin Web | `tukituki-admin-web` | `main` | `989ffc42faef5788c18455993d2462285b4db18d` (`989ffc42`) |
| Driver | `tukituki-driver-app` | `main` | `2743274900ef33374a76606b62ff841d476c232f` (`2743274`) |
| Passenger | `tukituki-passenger-app` | `main` | `7d717ff4b908dd1c0d5f932fca30c280b03b5f8f` (`7d717ff`) |

✅ **Verificado (2026-08-20)** contra `git branch --show-current` + `git rev-parse HEAD` reales de los cuatro repos, working tree limpio en los cuatro. Backend y Admin Web coincidían con lo que ya estaba registrado; Driver y Passenger se actualizaron — ambos habían avanzado el mismo día con la fusión de `R4.4B` (ver `historial-checkpoints.md`).

El detalle de cómo se llegó a cada uno de estos commits (checkpoints,
fast-forwards, smoke tests, limpieza de ramas) está en
`historial-checkpoints.md`.

## 3. Arquitectura general

```
Passenger (Flutter, Android)  ↔  Backend (NestJS + PostgreSQL/PostGIS + Redis)  ↔  Driver (Flutter, Android)
```

- Backend es la única fuente de verdad de negocio y de tiempo (tarifas, estados de viaje, disponibilidad, pagos, comisiones) — confirmado tanto por el propio Backend (`autoLoadEntities`, migraciones, `synchronize: false`) como por comentarios explícitos en ambas apps ("Backend es la única autoridad de tiempo").
- Persistencia: PostgreSQL con PostGIS (geolocalización) + Redis (disponibilidad geoespacial de conductores, presencia). Ninguna app móvil tiene base de datos local ni caché offline estructurada.
- **Transporte cliente↔servidor**: Backend expone tanto REST/JSON como WebSocket (namespaces `rides` y `safety` con autenticación JWT, `src/modules/rides/realtime/`, `src/modules/safety/realtime/`, ver `Backend/arquitectura.md`). **Ninguna de las dos apps móviles consume ese WebSocket todavía**: Driver y Passenger actualizan todo su estado en vivo por *polling* HTTP (`Timer.periodic`, 1–30 s según pantalla). Es la brecha más relevante para las notificaciones pendientes (sección 8 y 17).
- Servicios externos confirmados por los docs técnicos del Backend: Google Routes API y Google Places/Geocoding (cotización y direcciones), Izipay (pago digital, deshabilitado por defecto), Firebase Cloud Messaging (push, deshabilitado por defecto — y aun habilitándose, ninguna app cliente registra actualmente un listener de push).

## 4. Flujo de viaje vigente

Confirmado por el Backend (`rides` module) y ambas apps:

1. Passenger cotiza destino (`FareQuote`, vía Google Routes) y envía una oferta de precio (`passengerOfferFare`).
2. Backend busca conductores (`RideDispatchWorker`, radio creciente 2→5→10 km, ver sección 6) y crea `RideOffer` para cada uno.
3. Driver ve la solicitud, puede **aceptar** (mismo precio) o **contraofertar** (`PROPOSED`); esto **no asigna al conductor todavía** (`RideOfferStatus.PROPOSED`: "Todavía NO significa que ganó el viaje" — comentario textual en `ride-offer-status.enum.ts`).
4. Passenger elige una propuesta final → `Ride.agreedFare` se congela → recién ahí `RideOfferStatus.ACCEPTED` y el Ride pasa a `DRIVER_ASSIGNED` (Driver pasa a `BUSY`).
5. Transiciones siguientes, validadas por GPS/PostGIS: `DRIVER_ARRIVING` → `DRIVER_ARRIVED` → (código PIN de inicio) → `IN_PROGRESS` → `COMPLETED`.
6. Al completar: tarifa final, `RidePayment` (efectivo hoy), y si `COMMISSION_MODE=ENFORCED` (hoy `DISABLED`), acumulación de comisión.

Sobre el paso 3→4: el mecanismo de negociación descrito arriba **ya está alineado** con la decisión de producto "aceptar/proponer no debe asignar inmediatamente al Driver" (sección 5 de decisiones de producto) — es el comportamiento observado hoy en el modelo de datos del Backend, no un objetivo pendiente.

## 5. Pricing y pagos

📌 **DECISIÓN DE PRODUCTO APROBADA**: TukiTuki no es intermediario financiero del pago del viaje. El pago es directo Passenger ↔ Driver; una futura comisión de plataforma sería informativa/contractual al Driver, no un descuento automático al Passenger.

- ✅ **IMPLEMENTADO** (Backend + ambas apps): pago en efectivo como flujo principal (`PaymentMethod.CASH`); Passenger crea el ride con `paymentMethod: CASH` fijo (la app aún no ofrece elegir método).
- 🟡 **PARCIAL**: Izipay (pasarela digital) existe en el Backend (`payments/gateways/izipay-payment.gateway.ts`) pero está deshabilitado por defecto (`IZIPAY_ENABLED=false`) y ninguna app cliente lo integra en UI.
- 🔴 **PENDIENTE**: Yape P2P directo — no hay evidencia de esto en ningún repositorio; solo se menciona `YAPE`/`PLIN` como valores de enum de `PaymentMethod` en Backend, sin flujo real detrás.
- ✅ **IMPLEMENTADO**: comisión de plataforma modelada (`CommissionPolicy`, `RideCommission`) pero **no exigible** por defecto (`COMMISSION_MODE=DISABLED`); esto ya coincide con la intención de que la comisión no descuente automáticamente el pago del Passenger.
- ✅ **IMPLEMENTADO**: `passengerOfferFare` → negociación → `agreedFare` como precio final base (ver sección 4).
- 📌 **DECISIÓN DE PRODUCTO APROBADA, no verificada a nivel de copy de pantalla**: no mostrar "Precio recomendado TukiTuki" ni prellenar automáticamente la oferta del Passenger con `estimatedFare`. Los docs técnicos de `App-passenger` confirman que `FareEstimate.estimatedFare` existe como dato interno (`lib/features/fare/domain/fare_estimate.dart`), pero no documentan el texto/copy exacto de la pantalla Home lo suficientemente en detalle como para confirmar si algún texto tipo "recomendado" sigue visible hoy. **[PENDIENTE: verificar el copy exacto de `home_screen.dart` contra esta regla]**.

## 6. Matching

✅ **IMPLEMENTADO** en Backend, confirmado por código y por commits recientes:

- Radio de búsqueda progresivo: **2 km → 5 km → 10 km** (`RIDE_SEARCH_RADII_METERS`, `src/modules/rides/ride-matching.constants.ts`), y tras alcanzar el radio máximo, el sistema **sigue reintentando en 10 km** en vez de detenerse (comentario "G3C-lite"; commits `c1fab96b`, `009b50d9`).
- Duración total de búsqueda: `RIDE_SEARCH_TTL_MS = 5 * 60 * 1000` → **5 minutos** (`Ride.searchExpiresAt`).
- **Late-join**: un conductor que se conecta después de que el Passenger ya envió la solicitud puede recibir esa solicitud vigente — implementado vía lease en Redis (`DriverAvailabilityRedisService`, comentarios "G3A") y `RideDispatchWorker.runLateJoinOnce()` (commit `fdee792c`).

Este comportamiento coincide exactamente con la decisión de producto de matching descrita en el prompt de esta tarea — no hay brecha ACTUAL vs OBJETIVO aquí.

## 7. Passenger

### Implementado

- ✅ Registro corto (celular + contraseña) y login (`auth/register`, `auth/login`), con verificación de teléfono por OTP **desconectada** del flujo activo (código huérfano, `App-passenger/decisiones.md`).
- ✅ Perfil de pasajero obligatorio como paso separado tras autenticarse (`/complete-profile`: nombre, apellido) antes de entrar a `/home` — salvo que ya haya un ride activo en curso, que siempre tiene prioridad sobre este paso; la regla aplica igual a cuentas nuevas y antiguas, sin excepciones. Ver `App-passenger/decisiones.md` e `historial-checkpoints.md` (`R4.2`) para el detalle de cómo se corrigió.
- ✅ Cotización automática (`FareQuote`) + selección de destino + negociación de oferta en la pantalla Home (`home_screen.dart`).
- ✅ Ciclo completo de seguimiento del viaje por polling (búsqueda, ofertas de conductores, asignación, PIN de inicio, recibo, calificación).
- ✅ Recibo de viaje con desglose de tarifa y estado de pago.

### Pendiente

- 🔴 "Has llegado a tu destino" con confirmación del Passenger al completar el Driver — no hay evidencia de esta pantalla/flujo en `App-passenger` (ver sección 13, no debe bloquear el cierre del Ride si el Passenger no confirma).
- 🔴 Estado del viaje en background/pantalla bloqueada tipo navegación activa — no existe (la app solo actualiza en foreground vía `Timer.periodic`).
- 🟡 Interfaz Passenger completa: partes existen (Home, búsqueda, recibo) pero el propio proceso de mejora del producto la marca como parcial (ver tabla de la sección 11).
- ~~Mapas/live tracking fluido~~ — **animación del marcador del Driver hecha** (`R4.4B`, historial-checkpoints.md, 2026-08-20): el marcador interpola suavemente entre posiciones de polling en vez de saltar. Ver estado y pendientes exactos en el checkpoint `R4.4B`. La polyline de la ruta sí se decodifica y se dibuja en pantalla (confirmado en `TukiTuki-Designer-Handoff-R1.md` sección 8) — la duda original sobre `routePolyline` ya estaba resuelta antes de esta corrección, solo no se había actualizado aquí.

### Onboarding futuro

📌 **DECISIÓN DE PRODUCTO APROBADA**: mantener el registro corto. Campos actuales importantes: celular, contraseña, nombre, apellidos. Foto: opcional. Contacto de emergencia: opcional/recomendado.

Campos **considerados** para ampliación futura (NO decididos como obligatorios): correo electrónico, DNI, dirección domiciliaria. Explícito: DNI y dirección domiciliaria **no deben volverse obligatorios** del alta inicial sin una nueva decisión de producto — hoy no existen en el registro (`App-passenger/arquitectura.md`, módulo `passenger`).

## 8. Driver

### Implementado

- ✅ Login por teléfono + password con verificación de rol `DRIVER` (`GET auth/me`).
- ✅ Estado operativo (online/offline/heartbeat), ubicación GPS en vivo.
- ✅ Ofertas de viaje: ver, aceptar, contraofertar (`DriverOffersRepository`).
- ✅ Ciclo de vida del viaje activo: llegada (con validación GPS/PostGIS), inicio por código, cobro en efectivo, finalización, cancelación, espera/no-show.
- ✅ Estadísticas diarias (`DriverDailyStats`).
- ✅ Bottom navigation: **[PENDIENTE: verificar contra el código de `tukituki-driver-app` si ya existen las 4 secciones "Inicio / Solicitudes / Ingresos / Perfil"]** — `App-driver/arquitectura.md` documenta rutas `/home`, `/active-ride`, `/cash-payment`, `/completed-payment`, sin describir explícitamente una barra de navegación con esas 4 etiquetas ni una pantalla "Perfil" independiente.

### Onboarding aprobado (✅ IMPLEMENTADO Y FUSIONADO A `main` — ver `historial-checkpoints.md` para la corrección registrada el 2026-08-20)

📌 Estructura oficial deseada (6 pasos): 1) Tu cuenta (celular, contraseña) → 2) Sobre ti (foto, nombre, apellidos, DNI/CE, fecha de nacimiento, correo opcional — **sin dirección**) → 3) Tu mototaxi (placa, marca, modelo, año, color, propio/alquilado) → 4) Tus documentos (licencia, SOAT, TIV) → 5) Revisar y enviar → 6) Solicitud en revisión.

📌 **Decisión de producto de JuanJo (`DRIVER-ONBOARDING-R2`, 2026-08-16): NO pedir dirección domiciliaria durante el onboarding.** `DriverProfile.address` se conserva en el modelo con sus datos existentes, pero deja de ser obligatoria (nullable en DB y en `CreateDriverProfileDto`) y no aparece como requisito del paso 2. Admin puede seguir mostrándola cuando exista.

✅ **ACTUAL**: el onboarding de Driver de 6 pasos está implementado y fusionado a `main` de `tukituki-driver-app` — pantallas reales en `lib/features/driver/presentation/onboarding/`. Ver `historial-checkpoints.md` (`DRIVER-ONBOARDING-R3.x`) para los commits y el detalle completo del cierre.

Explícito: no pedir manualmente número de motor, número de chasis ni VIN. Desde `DRIVER-ONBOARDING-R2` (**ya en `main`**), `DriverVehicle.engineNumber`/`chassisNumber` son nullable en DB y opcionales en `CreateDriverVehicleDto`; se agregó `ownership` (`VehicleOwnership.OWNED`/`RENTED`), requerido en el DTO de creación para vehículos nuevos aunque nullable en DB (vehículos existentes en STAGING no tienen este dato y no se les inventó uno).

### Documentos

**ACTUAL en `main`@`ede503bc`** (Backend, `src/modules/drivers/driver-application-submission.service.ts`): reducido a **3 documentos del expediente**: `DRIVER_LICENSE` (licencia), `SOAT`, `VEHICLE_REGISTRATION` (tarjeta de propiedad/TIV), mediante una constante única de dominio (`REQUIRED_DRIVER_APPLICATION_DOCUMENT_TYPES`) reutilizada por `DriverApplicationSubmissionService` y `AdminDriverReviewService` para que no puedan volver a divergir. `DNI_FRONT`/`DNI_BACK`/`PROFILE_PHOTO` se conservan en el enum y en registros legacy, pero dejan de ser obligatorios. La foto de perfil pasa a exigirse como `DriverProfile.photoObjectKey`/`photoUrl` (nunca un `DriverDocument`) **antes de permitir `POST /drivers/me/submit`**; `AdminDriverReviewService.approve` revalida la misma condición como última barrera.

### Revisión administrativa

- ✅ **IMPLEMENTADO en Backend**: `DriverStatus` (`DRAFT` → `PENDING_REVIEW` → `APPROVED`/`REJECTED`, + `SUSPENDED`), módulo `admin-drivers` con `admin-driver-review.service.ts` que permite aprobar/rechazar. El conductor no queda habilitado automáticamente tras el registro — el modelo de estados ya refleja esto.
- ✅ **IMPLEMENTADO, ya en `main`**: el panel administrativo es el cuarto repositorio `tukituki-admin-web` (confirmado por `ADMIN-STORAGE-AUDIT`). Desde `RELEASE-R2` (`main`@`989ffc42faef5788c18455993d2462285b4db18d`) permite a un ADMIN/SUPER_ADMIN revisar datos personales, mototaxi, foto y los 3 documentos objetivo (licencia/SOAT/TIV) con preview privado vía Railway Storage, y decidir Aprobar/Rechazar/Suspender (checkpoint `ADMIN-DRIVER-R1B`, validado manualmente por JuanJo contra staging desplegada desde `main` — ver historial-checkpoints.md, `RELEASE-R2.1`).
- Regla de privacidad ya soportada por el modelo de roles del Backend (`UserRole.ADMIN`/`SUPER_ADMIN`, `RolesGuard`): los documentos privados solo deberían ser visibles para su propietario y ADMIN/SUPER_ADMIN — **[PENDIENTE: no se auditaron en este documento los endpoints de documentos uno por uno para confirmar que ningún endpoint expone `fileUrl` a otros Drivers/Passengers; ver Backend/errores-conocidos.md sobre `fileUrl` como URL arbitraria]**.

### Pendiente

- 🔴 Notificación inmediata + vibración de nueva solicitud, funcionando aunque el Driver no esté mirando la app — bloqueado hoy porque la app usa solo polling HTTP en foreground, sin push ni WebSocket conectado (ver sección 3).

## 9. Backend

Resumen de capacidades (detalle completo en `Backend/arquitectura.md`, no se duplica aquí):

- ✅ Autenticación JWT + sesiones con refresh rotativo, autorización por rol (`RolesGuard`, `SUPER_ADMIN` con bypass).
- ✅ Ciclo de vida completo de Ride, matching con radio progresivo y late-join, negociación bidireccional de tarifa, cancelaciones/no-show con cargos configurables, calificaciones mutuas.
- ✅ Pagos en efectivo; Izipay integrado pero apagado por defecto.
- ✅ Comisiones y liquidaciones a conductor modeladas, comisión no exigible por defecto.
- ✅ Seguridad: SOS/incidentes, contactos de emergencia, enlaces de viaje compartido.
- ✅ Notificaciones: outbox pattern (polling sobre PostgreSQL) + push FCM (apagado por defecto) + notificaciones in-app; **WebSocket de rides/safety implementado pero sin consumidores móviles aún** (ver sección 3).
- ✅ Persistencia: PostgreSQL+PostGIS, `synchronize: false`, todo por migración (28 migraciones al commit analizado).
- ✅ **En `main`, validado en STAGING**: la integración de storage de archivos (S3/Railway) — STORAGE-R2 + STORAGE-R2.1 + STORAGE-R3, los tres PASS — está desde `RELEASE-R2` en `main`@`7517c5ea` (Backend), y desde `RELEASE-R2.1` STAGING quedó redesplegado desde `main` y validado en runtime real (health, login SUPER_ADMIN, Storage Admin — ver sección 10). **Sigue sin estar en producción**, que permanece sin iniciar.

## 10. Railway Storage

### Decisión

📌 Proveedor: **Railway Storage Buckets**, bucket previsto `tukituki-media`. Objetivo: no guardar binarios en PostgreSQL ni en el filesystem persistente del Backend. Flutter nunca debe recibir `ACCESS_KEY_ID`/`SECRET_ACCESS_KEY`. Arquitectura deseada: `Flutter → Backend autenticado → presigned upload → Railway Storage Bucket → complete/HeadObject → Backend vincula objectKey`. Documentos privados vía presigned GET/autorización. Categorías necesarias: `PASSENGER_PROFILE_PHOTO`, `DRIVER_PROFILE_PHOTO`, `DRIVER_LICENSE`, `SOAT`, `VEHICLE_REGISTRATION` — explícitamente **sin** `DNI_FRONT`/`DNI_BACK` como requisito nuevo.

### Estado actual

✅ **ACTUAL**: Railway Storage (S3-compatible) implementado y en `main` — presign/PUT/complete, capability tokens firmados para privacidad de avatares/documentos, categorías `PASSENGER_PROFILE_PHOTO`/`DRIVER_PROFILE_PHOTO`/`DRIVER_LICENSE`/`SOAT`/`VEHICLE_REGISTRATION`. Validado con pruebas reales contra Railway STAGING. Ver `historial-checkpoints.md` (`STORAGE-R1`–`R3`, `RELEASE-R2`) para el detalle completo.

### Diseño aprobado

Estrategia implementada: `objectKey` nuevo (server-side, nunca elegido por el cliente) + compatibilidad gradual con `fileUrl`/`photoUrl` legacy; AWS SDK v3 modular; flujo presign PUT + upload directo + `complete` + `HeadObject`; documentos privados (owner/ADMIN/SUPER_ADMIN vía URL presignada temporal); avatares mediante una URL estable del propio Backend (`/storage/avatars/driver|passenger/:id`) que redirige (302) a una presigned GET fresca — nunca se persiste una presigned URL como dato canónico. **Corrección de STORAGE-R2.1**: esa URL estable ahora requiere además un capability token firmado (`?token=...`) emitido únicamente por el propio Backend dentro de un contexto ya autorizado; el `profileId` en la ruta ya no es, por sí solo, suficiente para resolver la foto (política de producto: "no security by obscurity"). `photoUrl` ya no se persiste como valor estático — se resuelve al vuelo en cada lectura, por lo que los mismos ~10 puntos del código que exponían `driverProfile.photoUrl`/`passengerProfile.photoUrl` (incluido el share-link público de un viaje) siguen funcionando, ahora emitiendo un token fresco en cada respuesta.

### Siguiente checkpoint

`main` **no es producción** — Railway sigue sin recibir este código hasta que JuanJo cambie manualmente el origen de deploy de STAGING. 🔴 **Sigue pendiente**: decidir/ejecutar la configuración de Railway para el ambiente de **producción** (bucket, credenciales, `STORAGE_ENABLED=true`, `STORAGE_AVATAR_TOKEN_SECRET` propios de producción — nunca reutilizar los de staging) — milestone separado, sin fecha ni autorización todavía.

## 11. Mejoras originales

| # | Mejora | Estado |
|---|---|---|
| 1 | Solicitud visible para Driver que se activa después (late-join) | ✅ IMPLEMENTADO |
| 2 | Mejorar interfaz Passenger completa | 🟡 PARCIAL |
| 3 | Mostrar punto de recogida Passenger al Driver | ✅ IMPLEMENTADO |
| 4 | Solicitud vigente 5 minutos | ✅ IMPLEMENTADO |
| 5 | Estados visuales claros del viaje | 🟡 PARCIAL / PENDIENTE |
| 6 | Mapas/live tracking más fluidos | 🟡 PARCIAL / PENDIENTE |
| 7 | Rango de solicitudes 2→5→10 km | ✅ IMPLEMENTADO |
| 8 | "Has llegado a tu destino" + confirmación opcional | 🔴 PENDIENTE |
| 9 | Estado del viaje en segundo plano/pantalla bloqueada | 🔴 PENDIENTE |
| 10 | Notificación/vibración de nueva solicitud Driver | 🔴 PENDIENTE |
| 11 | Mejorar registro/perfil Passenger | 🔴 PENDIENTE |
| 12 | Onboarding/autoregistro Driver completo | 🔴 PENDIENTE |
| 13 | Railway Storage para fotos/documentos | 🟡 EN PROCESO — STORAGE-R1/R2/R2.1/R3 PASS, validación real contra staging, y **fusión a `main` completada (RELEASE-R2)**; configuración de Railway para producción sigue pendiente de autorización |

## 12. Decisiones de producto vigentes

- Marca visible: **TukiTuki** (no "Mi TukiTuki"/"Tu TukiTuki" salvo petición explícita).
- Pago Passenger↔Driver directo; TukiTuki no retiene el dinero del viaje; comisión futura informativa/contractual, no descuento automático.
- Precio final = `agreedFare` tras negociación; asignación del Driver ocurre solo tras la selección final del Passenger; `estimatedFare` no se presenta como "recomendado".
- Matching: 2→5→10 km, ventana total 5 minutos, con late-join.
- Passenger Home: flujo cerrado (destino → cotización automática → oferta del Passenger → "Ofrecer y buscar conductor"), sin botón "Calcular tarifa" ni "Precio recomendado".
- Driver: onboarding de 6 pasos aprobado (no implementado); 3 documentos definitivos (licencia, SOAT, TIV); revisión administrativa obligatoria antes de habilitar.
- Almacenamiento de archivos: Railway Storage Buckets, presigned upload, sin binarios en Postgres/filesystem, sin credenciales AWS en Flutter.
- **Política de avatares (STORAGE-R2.1)**: la foto de un Passenger es de acceso controlado/contextual (propio Passenger, ADMIN/SUPER_ADMIN, o un Driver únicamente en un contexto de Ride real ya existente — nunca ampliado). La foto de un Driver es igualmente de acceso controlado/contextual (propio Driver, ADMIN/SUPER_ADMIN, un Passenger en un contexto real de oferta/ride/historial, o un share-link de viaje válido como excepción acotada). **No existe endpoint público genérico de avatar por `profileId`**: el `profileId` por sí solo nunca es suficiente, se exige además un capability token firmado y de corta duración emitido por el Backend dentro de una respuesta ya autorizada.
- **Política MVP de verificación telefónica (`CROSS-APP-R4.3`, 2026-08-18, historial-checkpoints.md)**: `status != ACTIVE` bloquea siempre. Cuentas `PASSENGER`/`DRIVER` con `ACTIVE`: `isPhoneVerified=false` es suficiente durante el MVP (aprobación admin, login, JWT, refresh) mientras `OTP-R3`/SMS real sigue `DEFERRED`. Cuentas `ADMIN`/`SUPER_ADMIN` (incluido rol mixto): siguen exigiendo `ACTIVE` + `isPhoneVerified=true` en todos esos gates, no solo en `loginAdmin` — la seguridad administrativa no se relaja. `OTP-R2` y toda la infraestructura OTP quedan intactos.
- **Prioridad de producto (`DEMO-PRIORITY-DECISION-R1`, 2026-08-16)**: JuanJo decidió priorizar terminar una **demo funcional** de TukiTuki antes de las gestiones/afiliaciones y de las adquisiciones que dependen de ellas (dominio corporativo, correo corporativo, proveedor SMS). Consecuencia directa: `OTP-R3` (SmsProvider/transporte SMS real) queda **DEFERRED/PAUSED — no cancelado** hasta que la demo esté lista; `DRIVER-ONBOARDING-R3` vuelve a ser la prioridad inmediata (ver sección 14). `tukitukiapp.pe` fue identificado como **candidato de dominio**, **sin comprar todavía** — no se afirma propiedad ni adquisición. `LabsMobile` fue evaluado como **candidato de proveedor SMS**, **sin contratar** (sin cuenta corporativa completa, sin API token, sin integración, sin pago); otros transportistas SMS pueden evaluarse más adelante. Para demostraciones antes de tener un proveedor SMS real, se usarán **cuentas sintéticas QA previamente verificadas** — explícitamente **no** se creará ningún bypass de OTP de producción, OTP hardcodeado, validación local en Flutter, secreto embebido en el APK, ni se habilitará `OTP_DEBUG_ENABLED`/un proveedor fake en un ambiente de tipo producción. El requisito de producto se mantiene sin cambios para la versión real: verificación OTP telefónica obligatoria antes de completar la habilitación correspondiente (ver `OTP-AUDIT-R1`/`OTP-R2`, historial-checkpoints.md).

## 13. No implementar / No asumir

- No mostrar "Precio recomendado TukiTuki" visible al Passenger.
- No autoasignar al Driver al aceptar/proponer una oferta (ya alineado con el código actual, ver sección 4).
- No inventar ETA si el Backend/API no la provee.
- No inventar polyline/ruta real si el Backend/API todavía no la entrega en ese punto del flujo.
- No almacenar documentos de forma pública ni sin control de acceso.
- No exponer un endpoint público genérico de avatar por `profileId` (un UUID conocido/adivinado no es autorización — corregido en STORAGE-R2.1, ver sección 10).
- No exigir seis documentos de Driver en el diseño **futuro** (el Backend actual sí los exige hoy — ver sección 8).
- No pedir número de motor, chasis ni VIN manualmente en el onboarding.
- No convertir a TukiTuki en intermediario de pago del viaje.
- No asumir que el panel administrativo de revisión de documentos ya existe dentro de estos tres repositorios.
- No asumir que Driver o Passenger consumen el WebSocket del Backend — ambas apps operan por polling HTTP hoy.
- No actualizar `docs/contexto/*` (los 18 documentos técnicos ni este documento) durante una implementación provisional — solo al cerrar un checkpoint o al declararse una decisión de producto vigente.

## 14. Roadmap inmediato

📌 **Orden de prioridad vigente desde `DEMO-PRIORITY-DECISION-R1`** (2026-08-16, decisión de producto de JuanJo — sustituye el orden anterior de esta sección):

1. 🔴 **`DRIVER-ONBOARDING-R3`** (prioridad inmediata): construir en `tukituki-driver-app` las pantallas/flujo del onboarding de Driver que hoy no existen (Backend foundation ya `CLOSED` desde `DRIVER-ONBOARDING-R2`, historial-checkpoints.md).
2. 🔴 Completar la demo funcional Driver/Passenger necesaria (alcance exacto a definir en `DRIVER-ONBOARDING-R3` y checkpoints siguientes).
3. 🔴 Revisión funcional de la demo completa.
4. 🔴 Afiliaciones/gestiones (fuera del alcance técnico de estos repositorios).
5. 🔴 Dominio corporativo — candidato identificado (`tukitukiapp.pe`), **no comprado todavía**.
6. 🔴 Proveedor SMS — candidato evaluado (LabsMobile), **no contratado todavía**; otros transportistas pueden evaluarse.
7. 🔴 `OTP-R3` — transporte SMS real (**DEFERRED/PAUSED, no cancelado** — núcleo OTP ya `CLOSED` y propiedad de TukiTuki desde `OTP-R2`, historial-checkpoints.md).
8. 🔴 Integración OTP final (gate obligatorio del Paso 1 del onboarding de Driver con SMS real).
9. 🔴 Producción futura (milestone separado, sin fecha ni autorización).

Pendientes técnicos previos a esta decisión, sin resolver y no reordenados por ella (siguen abiertos, retomar según corresponda):

- Configurar Railway para **producción** (Storage: bucket, credenciales y `STORAGE_AVATAR_TOKEN_SECRET` propios, nunca reutilizar los de staging) — requiere autorización explícita de JuanJo. (STORAGE-R1/R2/R2.1/R3 ya en PASS, ver sección 10.)
- Actualizar al menos un cliente (Flutter) para que suba archivos reales usando el flujo presign/PUT/complete ya validado en staging.
- Mejorar registro/perfil Passenger (sin saturar el alta inicial, ver sección 7).
- Continuar mejoras pendientes del flujo de viaje/notificaciones/mapas (ítems 8-10 y 5-6 de la tabla de la sección 11).

No se registran fechas: no hay evidencia de un calendario comprometido en ningún repositorio ni en este prompt.

## 15. Método de trabajo

- **ChatGPT**: analiza problema/producto/UX/riesgos; genera prompts técnicos para Claude Code.
- **Claude Code**: audita el repositorio real, programa, ejecuta tests/builds, revisa Git, entrega reporte.
- **JuanJo**: realiza pruebas físicas de APK, revisa resultado visual/funcional, aprueba el checkpoint.

Flujo normal: análisis → auditoría → implementación → tests → APK → prueba física → correcciones → aprobación → commit/push → cierre.

Reglas operativas vigentes: no publicar a `main` antes de las validaciones acordadas; no usar `git reset`/`git restore`/`git clean`/`git checkout` de forma destructiva para "arreglar" un working tree si implica descartar trabajo; preservar cambios no relacionados; no actualizar dependencias salvo que la tarea lo requiera; no introducir secretos al repositorio.

**Aclaración formal de JuanJo (2026-08-16, vigente desde `RELEASE-R2`): `main` NO significa producción.** Flujo oficial durante esta etapa del proyecto:

1. `main` = estado oficial/estable/aprobado del código — no implica que esté desplegado en producción.
2. Una tarea nueva nace desde `main` en una rama `test/*`.
3. Implementación + tests en esa rama.
4. STAGING puede apuntar temporalmente a la rama `test/*` mientras se valida.
5. JuanJo valida (local y/o contra STAGING real).
6. Con PASS y aprobación explícita de JuanJo, `test/*` se integra a `main` (fast-forward cuando sea posible, sin merge commit innecesario, sin rebase/squash/cherry-pick).
7. STAGING vuelve a apuntar a `main` (cambio manual de JuanJo en Railway).
8. Se valida `main` en STAGING (health/runtime).
9. Se cierra el checkpoint (documentación, limpieza de rama TEST si corresponde).
10. Producción solo mediante autorización/milestone separado — nunca implícita por llegar a `main`.

Precedente: `RELEASE-R1` (auditoría de fast-forward) → `RELEASE-R2` (ejecución del fast-forward de Backend y Admin Web a `main`, ver historial-checkpoints.md) siguió exactamente este flujo.

## 17. Pendientes / preguntas abiertas

Solo pendientes reales, no decisiones ya tomadas:

- Configurar Railway para **producción** (bucket, credenciales y `STORAGE_AVATAR_TOKEN_SECRET` propios, nunca reutilizar los de staging) — requiere autorización explícita de JuanJo, milestone separado de `RELEASE-R2`. Luego actualizar el/los clientes móviles para que suban archivos reales.
- **Verificación de teléfono por OTP en el onboarding de Driver: DIFERIDA para el MVP** (`DRIVER-ONBOARDING-R3.3`, 2026-08-17, historial-checkpoints.md, **aprobada físicamente en `R3.3.3`**) — decisión explícita de producto de JuanJo. `isPhoneVerified=false` ya no bloquea el onboarding; `phoneE164` sigue único en Backend/DB; registro duplicado se rechaza con `409` y la app lo muestra en un modal centrado aprobado físicamente (título, X, CTA "Iniciar sesión"); `OTP-R2` permanece intacto; `OTP-R3` (SMS real, con proveedor) se implementará posteriormente, antes de producción. La secuencia `OTP-DEMO-R1` (cuentas QA con `OTP_DEMO_ENABLED`) queda preservada sin fusionar, sin descartarse como opción futura, pero no fue la vía usada por este MVP.
- Definir e implementar: "Has llegado a tu destino" con confirmación opcional del Passenger; estado del viaje en background/pantalla bloqueada; notificación + vibración inmediata al Driver por nueva solicitud — los tres bloqueados hoy, al menos en parte, porque ninguna app consume el WebSocket ya expuesto por el Backend (sección 3).
- Firma de release Android: tanto `tukituki-driver-app` como `tukituki-passenger-app` generan hoy el build `release` firmado con la clave de **debug** (confirmado en ambos `errores-conocidos.md`) — no apto para publicación tal cual; no se conoce plan de firma de producción en ningún repositorio.
- Verificar el copy exacto de la pantalla Home de Passenger contra la regla "no mostrar precio recomendado" (sección 5) — no confirmado a nivel de texto de UI en este análisis.
- Conectar un proveedor real de SMS al OTP ya endurecido (`OTP-R3`): definir abstracción `SmsProvider`, elegir proveedor (LabsMobile/Clicklab/otro) y decidir la estrategia `FakeSmsProvider` para dev/test. Sigue sin existir ningún `SmsProvider`/transporte SMS real — es el pendiente que bloquea cerrar el gate obligatorio de OTP en el Paso 1 del onboarding de Driver, pero **queda `DEFERRED/PAUSED` por decisión explícita de producto (`DEMO-PRIORITY-DECISION-R1`, 2026-08-16, ver `historial-checkpoints.md`) hasta terminar la demo funcional** — no es un pendiente activo hoy.
- **`OTP-DEMO-R1` (2026-08-16, historial-checkpoints.md)**: `IMPLEMENTED ON TEST — READY-FOR-CI` en `test/otp-demo-staging`@`fd9ba8b3`, no fusionado a `main`, no desplegado en Railway. Secuencia pendiente explícita, en orden: (1) `OTP-DEMO-R1.1` — sincronización documental, cerrada por este mismo registro; (2) `OTP-DEMO-R1` CI sobre la rama TEST; (3) Railway STAGING apuntando temporalmente a `test/otp-demo-staging`; (4) configurar `OTP_DEMO_ENABLED`/`OTP_DEMO_ALLOWED_PHONE_E164` reales en STAGING (nunca en producción); (5) validar el flujo demo real (request con `debugOtp` real → `verify` → `isPhoneVerified=true`); (6) `DRIVER-ONBOARDING-R3.3` (Flutter Driver) consumiéndolo; (7) prueba física por JuanJo; (8) integración a `main` solo si todo lo anterior está en PASS y JuanJo autoriza explícitamente. Es infraestructura temporal — se retira cuando `OTP-R3` (SMS real) esté disponible, no reemplaza `OTP-R2` ni a un futuro `SmsProvider`.

No se reabren aquí preguntas ya resueltas por decisión de producto (p. ej. cuántos documentos exige el Driver: la decisión vigente es **3**, aunque el Backend hoy todavía exija 6).
