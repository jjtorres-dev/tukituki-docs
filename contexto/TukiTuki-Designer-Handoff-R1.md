# TukiTuki — Designer Handoff R1

Fecha: 2026-08-16
Estado: READY-FOR-DESIGNER (ver veredicto al final)
Alcance: fotografía de solo lectura del producto real (Backend + Driver + Passenger), para que Claude Designer diseñe/rediseñe pantallas sin adivinar comportamiento.

Este documento **no sustituye** `estado-proyecto.md` ni los 18 documentos técnicos de `docs/contexto/Backend|App-driver|App-passenger`. Es un resumen orientado a UI/UX, derivado de ellos y verificado contra el código real al momento de esta auditoría.

Regla de este documento, heredada de `estado-proyecto.md`: cuando el código y una decisión de producto difieren, se marcan ambas — `ACTUAL` (lo que hace el código hoy) vs `OBJETIVO APROBADO` (decidido, no implementado). Nunca se presenta un objetivo como si ya existiera.

---

## 0. Estado de los repositorios al momento de esta auditoría

| Repo | Branch en disco | HEAD | vs origin/main | Working tree |
|---|---|---|---|---|
| `tukituki-backend` | `test/otp-demo-staging` | `fd9ba8b3` | 1 commit adelante de `origin/main` (`f147a664`) | limpio (commit ya hecho, no fusionado) |
| `tukituki-driver-app` | `test/driver-onboarding-r3` | `b562d24` (= `origin/main`) | igual | **con cambios locales SIN COMMITEAR** (Driver Onboarding R3, ver sección 5) |
| `tukituki-passenger-app` | `main` | `a5d2411` (= `origin/main`) | igual | limpio |

Ninguno de los tres repos fue modificado por esta auditoría. El único archivo nuevo en el sistema de archivos es este mismo documento.

---

## 1. Resumen del producto

**TukiTuki** es una app de mototaxi bajo demanda para una ciudad peruana (referencias de código apuntan a Tarapoto), con tres roles de cliente y un backend propio:

- **Passenger** (app Flutter Android): pide viajes, propone una tarifa, elige entre las respuestas del conductor, sigue el viaje, paga en efectivo, califica.
- **Driver** (app Flutter Android): recibe solicitudes, acepta o contraoferta una vez, atiende el viaje (llegada validada por GPS, inicio por PIN, cobro en efectivo), ve sus estadísticas del día.
- **Admin** (app web Next.js, cuarto repositorio `tukituki-admin-web`, fuera del alcance de esta auditoría salvo referencias puntuales del Backend): revisa y aprueba/rechaza/suspende conductores, ve documentos privados vía Storage.
- **Backend** (NestJS + PostgreSQL/PostGIS + Redis): única fuente de verdad de negocio y de tiempo — tarifas, estados de viaje, disponibilidad, matching, pagos, comisiones.

**Objetivo actual de la demo** (`DEMO-PRIORITY-DECISION-R1`, 2026-08-16): terminar una demo funcional Driver+Passenger antes de gestiones/afiliaciones (dominio corporativo, proveedor SMS, producción). La prioridad inmediata es construir en `tukituki-driver-app` las pantallas de onboarding de conductor que hoy no existen del todo — trabajo que, de hecho, ya está en marcha sin commitear en `test/driver-onboarding-r3` (ver sección 5).

Branding: **TukiTuki** siempre. Nunca "Mi TukiTuki" / "Tu TukiTuki" salvo copy natural de pantalla que lo requiera explícitamente.

---

## 2. Arquitectura funcional simple

```
Passenger (Flutter, Android)  ──HTTP(polling)──┐
                                                 ├──▶  Backend (NestJS + PostgreSQL/PostGIS + Redis)
Driver (Flutter, Android)     ──HTTP(polling)──┘            │
                                                              ├──▶ Admin Web (Next.js) — revisión de conductores
                                                              ├──▶ Railway Storage (fotos/documentos privados)
                                                              └──▶ WebSocket real (namespaces /rides, /shared-rides) — NINGUNA app móvil lo consume todavía
```

- **Ninguna app móvil tiene base de datos local ni caché offline.** Todo el estado en vivo se actualiza por *polling* HTTP (`Timer.periodic`, 1–30 s según pantalla), nunca WebSocket ni push, aunque el Backend ya expone ambos (WS real, FCM apagado por defecto).
- **Sin conexión = sin actualización.** No hay modo offline, no hay estado en background/pantalla bloqueada.
- Google Routes/Places para cotización y direcciones. Izipay integrado en Backend pero apagado (`IZIPAY_ENABLED=false`), ninguna app lo muestra. Yape/Plin solo existen como valores de enum, sin flujo real.

---

## 3. Reglas de negocio que afectan UX (verificadas contra código)

1. **Pago directo Passenger↔Driver, en efectivo.** TukiTuki no retiene el dinero del viaje. `paymentMethod` se crea fijo en `CASH` — la app Passenger no ofrece elegir método (no hay UI para eso).
2. **Negociación de una sola ronda**, confirmada en el enum real (`RideOfferStatus`: `OFFERED → PROPOSED → ACCEPTED` + `REJECTED/EXPIRED/CANCELLED`, 6 valores, sin `PASSENGER_COUNTERED`). Passenger propone (`passengerOfferFare`) → Driver acepta (mismo precio) o contraoferta una vez → Passenger elige una propuesta → `agreedFare` se congela → **recién ahí** el Driver pasa a `BUSY` y el Ride a `DRIVER_ASSIGNED`. Una variante de negociación multi-ronda existió en una rama (`origin/carlos`) y fue **descartada explícitamente por decisión de producto** — no debe reaparecer en el diseño.
3. **`estimatedFare` es interno**, literalmente comentado en el código como "recomendación TukiTuki" — nunca se expone como label ni aparece la palabra "recomendado"/"sugerido" en ninguna respuesta de API ni en el copy real de la pantalla Home del Passenger (verificado línea por línea, ver sección 8). El campo de oferta del Passenger arranca **vacío**, no prellenado con `estimatedFare`.
4. **Matching**: radio de búsqueda progresivo 2 km → 5 km → 10 km (`RIDE_SEARCH_RADII_METERS`); tras llegar a 10 km, sigue reintentando en 10 km (nunca se rinde ni inventa un radio mayor). Ventana total de búsqueda: 5 minutos (`RIDE_SEARCH_TTL_MS`). Ciclo de despacho cada 60 s (`RIDE_DISPATCH_INTERVAL_MS`). **Late-join**: un Driver que se conecta después de enviada la solicitud puede recibir esa solicitud vigente.
5. **Una cuenta puede tener rol PASSENGER + DRIVER simultáneamente.** De hecho, la app Driver reutiliza literalmente `POST auth/register/passenger` para crear la cuenta de un nuevo conductor — no existe `auth/register/driver` (decisión de producto explícita, comentada en el propio código Driver). La palabra "Passenger" nunca se muestra al usuario en ese flujo.
6. **Un Driver `APPROVED` mantiene la posibilidad de ser Passenger** — confirmado por arquitectura de roles (`roles[]` en `User`, no excluyentes). No hay ningún mecanismo que retire el rol PASSENGER al aprobar como Driver.
7. **Documentos de expediente del Driver: exactamente 3** — Licencia (`DRIVER_LICENSE`), SOAT, Tarjeta de propiedad/TIV (`VEHICLE_REGISTRATION`), reforzado por una única constante de dominio compartida entre envío y aprobación (no pueden volver a divergir). DNI/CE se pide como **dato de texto** en el perfil (documentType + documentNumber), no como foto/documento adjunto. La foto de perfil es un campo del perfil, no un documento del expediente.
8. **No se pide dirección domiciliaria** en el onboarding de Driver (decisión de producto `DRIVER-ONBOARDING-R2`). No se pide número de motor/chasis/VIN manualmente (quedaron opcionales/legacy).
9. **Revisión administrativa obligatoria** antes de habilitar a un Driver — el registro nunca autoaprueba.
10. **Fotos/documentos privados**, nunca URLs públicas. Un `profileId` conocido/adivinado nunca es suficiente por sí solo — se exige un capability token firmado de corta duración emitido por el propio Backend dentro de un contexto ya autorizado.

---

## 4. OBJETIVO APROBADO vs ACTUAL — tabla de brechas más relevantes para Designer

| Tema | OBJETIVO APROBADO | ACTUAL (código) |
|---|---|---|
| Onboarding Driver Paso 1 (Tu cuenta) | Implementado | ✅ Implementado — **sin commitear**, en `test/driver-onboarding-r3` |
| Onboarding Driver Paso 2-5 (Sobre ti / Tu mototaxi / Tus documentos / Revisar y enviar) | Implementado | 🔴 NO existe ninguna pantalla — placeholder "próximamente" |
| Gate de estado (DRAFT/PENDING_REVIEW/REJECTED/APPROVED/SUSPENDED) | Implementado | ✅ Implementado — máquina de estados completa, sin commitear |
| Verificación de teléfono real por SMS | Obligatoria antes de habilitar | 🔴 Diferida (`OTP-R3` DEFERRED). Pantalla actual es un gate explícitamente provisional, sin campo de código, con copy "todavía no está disponible" |
| Driver tab "Ingresos" | Con contenido | 🔴 Placeholder "coming soon" |
| Driver tab "Perfil" | Con contenido | 🔴 Placeholder "coming soon" |
| "Has llegado a tu destino" + confirmación Passenger | Pendiente de diseñar | 🔴 No existe en Backend ni en ninguna app |
| SOS / emergencia visible al usuario | Backend lo soporta (`safety` module) | 🔴 Ninguna app (Driver ni Passenger) tiene UI para esto |
| Ride sharing / share-link visible al usuario | Backend lo soporta (`RideShareLink`) | 🔴 Ninguna app tiene UI para esto |
| Disputa de pago en efectivo | Backend tiene endpoint (`POST passenger/rides/:id/payment/cash/dispute`) | 🔴 Ninguna app expone esta acción en UI |
| WebSocket para notificación instantánea | Deseado (vibración/push inmediata) | 🔴 Backend expone `/rides` y `/shared-rides` reales; ninguna app los consume — todo es polling |

---

## 5. Driver Onboarding — estado real por paso

Estructura de 6 pasos aprobada por producto:

| Paso | Nombre | Backend foundation | Pantalla Flutter | Estado |
|---|---|---|---|---|
| 1 | Tu cuenta (celular, contraseña) | ✅ `auth/register/passenger` + `auth/login` | `CreateAccountScreen` | ✅ **IMPLEMENTED** (sin commitear) |
| — | Verificación de teléfono (gate antes del paso 2) | ✅ OTP core listo (`OTP-R2` CLOSED), SMS real diferido | `PhoneVerificationScreen` | 🟡 **PROVISIONAL POR DISEÑO** — no hay campo de código; muestra número enmascarado + mensaje "el equipo TukiTuki verificará tu cuenta" + botón "Ya verifiqué mi número" (re-consulta estado) |
| 2 | Sobre ti (foto, nombres, apellidos, DNI/CE, fecha nacimiento, correo opcional) | ✅ `DriverProfile` acepta todos estos campos | ninguna | 🔴 **NOT IMPLEMENTED** |
| 3 | Tu mototaxi (placa, marca, modelo, año, color, propio/alquilado) | ✅ `DriverVehicle` acepta todos estos campos | ninguna | 🔴 **NOT IMPLEMENTED** |
| 4 | Tus documentos (Licencia, SOAT, TIV) | ✅ presign/PUT/complete de Storage ya validado en staging | ninguna | 🔴 **NOT IMPLEMENTED** |
| 5 | Revisar y enviar | ✅ `POST drivers/me/submit` | ninguna | 🔴 **NOT IMPLEMENTED** |
| 6 | Solicitud en revisión | ✅ `DriverStatus.PENDING_REVIEW` | `DriverOnboardingReviewScreen` | ✅ **IMPLEMENTED** (sin commitear) |

Pantallas de estado adicionales, todas **implementadas sin commitear**:
- `DriverOnboardingRejectedScreen` (`/onboarding/rejected`) — muestra `rejectionReason` si existe, CTA "Corregir solicitud" (hoy vuelve al placeholder del paso 2, porque los pasos 2-5 no existen todavía).
- `DriverOnboardingSuspendedScreen` (`/suspended`) — muestra `suspensionReason` si existe, sin canal de apelación (no inventado).
- `DriverOnboardingStateErrorScreen` (`/onboarding/state-error`) — red de seguridad para dos casos: (a) `APPROVED` sin rol `DRIVER` en la cuenta, (b) un `status` que el cliente no reconoce. Nunca deja pasar a Home silenciosamente.

**Máquina de estados de ruteo** (`resolveDriverApplicationState`, `lib/features/auth/domain/driver_session_state.dart`, nueva, sin commitear) — 9 resultados posibles, orden de evaluación:

1. `!isPhoneVerified` → pantalla de verificación de teléfono (bloquea todo lo demás — "sin teléfono verificado no hay Paso 2 en adelante")
2. `GET drivers/me` → 404 (sin perfil aún) → pantalla de inicio de onboarding (placeholder)
3. `status == DRAFT` → misma pantalla placeholder que el caso anterior
4. `status == REJECTED` → pantalla de rechazo
5. `status == PENDING_REVIEW` → pantalla de revisión
6. `status == SUSPENDED` → pantalla de suspensión
7. `status == APPROVED` + rol `DRIVER` presente → **Home** (único camino a Home)
8. `status == APPROVED` sin rol `DRIVER` → pantalla de error de estado (red de seguridad)
9. `status` no reconocido → pantalla de error de estado (red de seguridad)

Esta lógica de ruteo está completa y correcta según el modelo aprobado. Lo que falta es el **contenido** de los pasos 2-5 — ahí es donde debe concentrarse el trabajo de diseño con mayor prioridad.

---

## 6. Driver — flujo operacional (Driver ya aprobado)

Bottom navigation real, confirmado por código (`NavigationBar` Material 3, 4 tabs): **Inicio · Solicitudes · Ingresos · Perfil**.

- **Inicio**: toggle online/offline, mapa (`google_maps_flutter` real), estadísticas del día, ubicación GPS enviada por `PUT drivers/me/location`, heartbeat cada 10 s.
- **Solicitudes**: lista de ofertas de viaje, badge con conteo en vivo, polling cada 3 s, aceptar / contraofertar (diálogo aparte) / rechazar.
- **Ingresos**: 🔴 placeholder "coming soon" — sin pantalla real.
- **Perfil**: 🔴 placeholder "coming soon" — sin pantalla real.

Viaje activo (una sola pantalla-contenedor que cambia de contenido según `Ride.status`, no pantallas separadas):
- `DRIVER_ASSIGNED` / `DRIVER_ARRIVING` — acciones "iniciar llegada" / "he llegado".
- `DRIVER_ARRIVED` — entrada de PIN de 4 dígitos + sub-flujo de espera/no-show.
- `IN_PROGRESS` — viaje en curso.
- `COMPLETED` — entrega a cobro en efectivo / pantalla de finalización.

Pantallas separadas: `driver_cash_payment_screen.dart` (cobro, calculadora de vuelto en soles con montos rápidos S/5–S/500) y `driver_completed_payment_screen.dart` (recibo). Sin SOS, sin ninguna feature de emergencia en la app Driver — confirmado por búsqueda exhaustiva en el repo, cero resultados.

---

## 7. Ride lifecycle (confirmado en Backend, `RideStatus` enum real — 8 valores)

```
SEARCHING_DRIVER
   ↓ (Passenger elige una propuesta → agreedFare fijo → Driver pasa a BUSY)
DRIVER_ASSIGNED
   ↓ (Driver inicia desplazamiento)
DRIVER_ARRIVING
   ↓ (Driver confirma llegada, validado por GPS: distancia ≤150 m y precisión ≤100 m del punto de origen)
DRIVER_ARRIVED
   ↓ (Passenger ve PIN de 4 dígitos → lo dice de palabra → Driver lo ingresa)
IN_PROGRESS
   ↓ (Driver completa, con validación de destino por GPS)
COMPLETED

Rutas alternativas: CANCELLED (en cualquier punto anterior a COMPLETED) · EXPIRED (búsqueda venció sin match, 5 min)
```

Correspondencia pantalla ↔ estado:

| Estado | Passenger ve | Driver ve | Acciones reales |
|---|---|---|---|
| `SEARCHING_DRIVER` | Progreso de búsqueda + lista de propuestas de conductores conforme llegan | Nueva solicitud en "Solicitudes" | Passenger: elegir propuesta / cancelar. Driver: aceptar / contraofertar (una vez) / rechazar |
| `DRIVER_ASSIGNED` / `DRIVER_ARRIVING` | Conductor asignado, datos del vehículo/foto/rating, mapa | Botones "iniciar llegada" / "he llegado" | Cancelar (ambos lados, con cargos configurables) |
| `DRIVER_ARRIVED` | **PIN de 4 dígitos** con cuenta regresiva (TTL 15 min) e intentos restantes (máx. 5), opción de regenerar (máx. 3 veces, bloqueo tras agotar intentos) | Campo para ingresar el PIN que el Passenger le dice de palabra | Driver ingresa PIN → inicia viaje. Sub-flujo de espera/no-show con temporizador |
| `IN_PROGRESS` | Seguimiento en curso | Vista de viaje en curso | Driver: completar viaje |
| `COMPLETED` | Recibo (tarifa final, desglose, estado de pago), calificación al Driver | Pantalla de cobro efectivo → recibo de finalización | Passenger: calificar. Driver: confirmar cobro efectivo (con vuelto) |
| `CANCELLED` | Mensaje de cancelación (motivo si existe) | igual | — |

No existe "Has llegado a tu destino" con confirmación del Passenger — Backend no lo modela, ninguna app lo implementa. No inventar ETA: solo hay distancia/duración de la cotización inicial, nunca un tiempo estimado de llegada del conductor.

---

## 8. Negociación de tarifa — flujo exacto

```
Passenger propone (passengerOfferFare, sobre una cotización FareQuote/estimatedFare interno)
   ↓
Backend crea RideOffer (OFFERED) para cada Driver candidato dentro del radio de búsqueda
   ↓
Driver ve la solicitud → ACEPTAR (mismo precio) → RideOfferStatus.ACCEPTED*
                        → CONTRAOFERTAR (una sola vez) → RideOfferStatus.PROPOSED
                                                            (comentario textual en el código: "Todavía NO significa que ganó el viaje")
   ↓
Passenger ve todas las propuestas recibidas (aceptadas y contraofertadas) y ELIGE UNA
   ↓
Ride.agreedFare se congela ← recién en este momento
   ↓
Driver elegido pasa a BUSY, Ride → DRIVER_ASSIGNED
```

*Importante para Designer*: **aceptar u ofertar NO asigna al Driver.** El Driver solo se vuelve `BUSY` y queda asignado cuando el Passenger selecciona explícitamente su propuesta entre potencialmente varias. Esto está confirmado en el código real (no es solo un objetivo de producto) — verificar que ningún diseño muestre "conductor asignado" antes de que el Passenger elija.

No existe negociación multi-ronda (el Passenger no puede volver a contraofertar al Driver) — una variante que sí lo permitía existió en una rama descartada y **no debe reaparecer**.

Detalle en la pantalla Home del Passenger (`home_screen.dart`, verificado línea por línea):
- Destino: campo de texto con autocompletado (proxy de Backend a Google Places), deshabilitado hasta obtener ubicación GPS.
- Mapa real (`GoogleMap`) con marcadores y la **polyline de la ruta sí se decodifica y se dibuja** (esto corrige una duda abierta en `estado-proyecto.md` — sí está implementado).
- Chips de distancia (`X.X km`) y duración (`X min`), y expiración de cotización ("Cotización válida hasta HH:MM" / "Cotización vencida", se renueva automáticamente).
- `estimatedFare` **nunca se lee ni se muestra** en ningún punto de esta pantalla — confirmado por búsqueda en el archivo (cero coincidencias).
- Campo de oferta: `TextField` numérico, centrado, prefijo `S/`, con **hint (no valor) `5.00`**, ayuda: *"Este es el monto que verán los conductores."* — arranca vacío. **Cero coincidencias de "recomend"/"sugerid" en todo el archivo.**
- CTA principal, texto exacto: **"Ofrecer y buscar conductor"**.
- Tras crear el ride: navega a la pantalla de búsqueda/seguimiento (`/ride/:rideId`) — no hay pantalla intermedia de "ride creado".

---

## 9. Payment flow

- **Método único hoy: efectivo (`CASH`)**, fijado por el cliente al crear el ride — no hay selector de método en ninguna UI.
- Campos reales del pago (`RidePayment`): `amountDue`, `grossAmount`, `discountAmount`, `cashReceived`, `changeGiven`, `status` (PENDING/PROCESSING/PAID/FAILED/EXPIRED/DISPUTED/VOIDED).
- Tarifa: `passengerOfferFare` (lo que el Passenger tipeó) → `agreedFare` (congelada al elegir) → `finalFare` (fijada al completar, respeta `agreedFare`). `estimatedFare` es interno, nunca se muestra.
- Driver: pantalla de cobro con calculadora de vuelto (montos rápidos S/5 a S/500), luego pantalla de recibo/finalización con tarifa final, descuento si aplica, y estado de pago.
- Passenger: pantalla de recibo con desglose de tarifa y estado de pago, seguido de calificación al Driver.
- **Existe en Backend un endpoint de disputa de pago en efectivo** (`POST passenger/rides/:rideId/payment/cash/dispute`, motivos: monto incorrecto / no se pagó / no dieron el vuelto / otro) que **ninguna de las dos apps expone en su UI hoy**. Es una función real y llamable, pero no hay pantalla — queda como candidata a diseño futuro, no como algo ya construido.
- Izipay existe en Backend, apagado por defecto, sin UI en ninguna app. Yape/Plin solo enum, sin flujo.
- Comisión de plataforma modelada pero no exigible por defecto (`COMMISSION_MODE=DISABLED`) — no visible al usuario final hoy.

---

## 10. OTP — estado actual

- **Núcleo OTP: `CLOSED` y en producción de código** (`main`@`f147a664`, checkpoint `OTP-R2`). TukiTuki es dueño completo del ciclo: generación (`crypto.randomInt`, 6 dígitos), hash HMAC-SHA256 + comparación de tiempo constante, Redis con TTL, límite de intentos, cooldown de reenvío (preservado incluso al agotar intentos), rate limit dedicado por IP y por teléfono, invalidación de un solo uso, actualización de `isPhoneVerified`, enumeración de cuentas cerrada (mismo 200 para teléfono registrado o no).
- **Sin proveedor SMS real.** No existe ningún `SmsProvider`, ninguna integración con LabsMobile/Clicklab/otro. El único mecanismo de entrega sigue siendo `debugOtp` en la respuesta, condicionado a entornos no productivos.
- **`OTP-R3`** (transporte SMS real) está **DEFERRED/PAUSED, no cancelado**, hasta terminar la demo funcional.
- **Nuevo — no documentado aún en `estado-proyecto.md`**: existe un commit **sin fusionar a `main`** (`test/otp-demo-staging`, `fd9ba8b3`, "feat: add guarded staging otp demo flow") que agrega un mecanismo de demo: si `OTP_DEMO_ENABLED=true` **y** el entorno Railway es exactamente `staging` **y** el teléfono solicitado coincide con un único número en lista blanca (`OTP_DEMO_ALLOWED_PHONE_E164`), la respuesta de `otp/request` incluye el código real (el mismo generado por el flujo normal, no uno falso/hardcodeado) para facilitar demos sin SMS real. Falla el arranque del Backend si se intenta activar fuera de `staging`. **Verificado como conforme** con la prohibición explícita de `DEMO-PRIORITY-DECISION-R1` (nada de bypass de producción, nada hardcodeado, nada del lado Flutter, nada embebido en el APK) — no cambia ningún contrato de UI, es invisible para ambas apps.
- **En la app Driver**: la pantalla de verificación de teléfono (`PhoneVerificationScreen`, sin commitear, ver sección 5) es deliberadamente **provisional** — no tiene campo de código, solo informa que el equipo TukiTuki verificará la cuenta manualmente durante esta etapa. Los métodos `requestPhoneOtp()`/`verifyPhoneOtp()` existen en el repositorio Driver pero **no los invoca ninguna pantalla** — están listos para cuando llegue SMS real.
- **En la app Passenger**: existe una pantalla OTP (`/otp`, `otp_screen.dart`) pero está **huérfana** — nada navega hacia ella, confirmado por búsqueda en el repo. El registro de Passenger no pasa por verificación de teléfono en el flujo activo.
- **Regla de producto vigente para demos**: usar únicamente cuentas sintéticas QA verificadas manualmente. Prohibido explícitamente: bypass de producción, OTP hardcodeado, validación local en Flutter, secretos embebidos en el APK, `OTP_DEBUG_ENABLED` en un ambiente productivo.

---

## 11. Screen inventory — Passenger (`tukituki-passenger-app`)

8 rutas reales + 1 utilidad no enrutada. **No hay una pantalla separada por cada estado del viaje** — una sola pantalla (`RideSearchingScreen`) cambia de contenido internamente según `PassengerRide.status`.

| # | Pantalla | Archivo | Ruta | Propósito | Backend | Estado |
|---|---|---|---|---|---|---|
| 1 | Splash | `lib/features/auth/presentation/splash_screen.dart` | `/splash` (inicial) | Recuperar/validar sesión | `auth/me` | ✅ APROBADA/ESTABLE |
| 2 | Login | `lib/features/auth/presentation/login_screen.dart` | `/login` | Login teléfono+contraseña | `auth/login` | ✅ APROBADA/ESTABLE |
| 3 | Register | `lib/features/auth/presentation/register_screen.dart` | `/register` | Registro corto | `auth/register` → login directo | ✅ APROBADA/ESTABLE |
| 4 | OTP | `lib/features/auth/presentation/otp_screen.dart` | `/otp` | Código de verificación | `auth/otp/verify` | 🔴 **HUÉRFANA** — existe pero nada navega ahí |
| 5 | Complete profile | `lib/features/passenger/presentation/complete_profile_screen.dart` | `/complete-profile` | Nombre + apellido | `passengers/me` (crear) | ✅ APROBADA/ESTABLE — solo 2 campos, sin foto ni contacto de emergencia |
| 6 | Home | `lib/features/home/home_screen.dart` | `/home` | Destino, cotización, oferta, crear ride | `fares/estimate`, `places/autocomplete`, `places/:id`, `rides` (crear) | ✅ APROBADA/ESTABLE — ver detalle sección 8 |
| 7 | Ride searching/tracking | `lib/features/ride/presentation/ride_searching_screen.dart` | `/ride/:rideId` | Búsqueda, propuestas, selección, seguimiento del conductor, llegada, PIN, viaje en curso, cancelación | `rides/:id` (polling 3 s), ofertas, `rides/:id/start-code` | ✅ APROBADA/ESTABLE — una sola pantalla para todo el ciclo |
| 8 | Ride receipt / rating | `lib/features/ride/presentation/ride_receipt_screen.dart` | `/ride/:rideId/receipt` | Recibo, estado de pago, calificación | `rides/:id/receipt` (polling 2 s), `submitRating` | ✅ APROBADA/ESTABLE |
| — | Health | `lib/features/health/health_screen.dart` | no enrutada (utilidad) | Chequeo de Backend | `health` | Herramienta de desarrollo, no producto |

**Confirmado NOT IMPLEMENTED** (Passenger): SOS/emergencia, gestión de contactos de emergencia, compartir viaje/share-link, "Has llegado a tu destino", selección de método de pago, paradas múltiples, viajes programados, favoritos, códigos promocionales.

---

## 12. Screen inventory — Driver (`tukituki-driver-app`)

Incluye trabajo **sin commitear** de Driver Onboarding R3 (ver sección 5 para el detalle paso a paso).

| # | Pantalla | Archivo | Ruta | Propósito | Backend | Estado |
|---|---|---|---|---|---|---|
| 1 | Splash | `lib/features/auth/presentation/driver_splash_screen.dart` | `/splash` (inicial) | Resolver sesión → máquina de estados | `auth/me`, `drivers/me` (vía `resolveSessionState`) | ✅ ESTABLE, rewired (sin commitear) |
| 2 | Login | `lib/features/auth/presentation/login_screen.dart` | `/login` | Login teléfono+contraseña | `auth/login` | ✅ ESTABLE, con nuevo CTA "Crea tu cuenta" (sin commitear) |
| 3 | Crear cuenta (Paso 1) | `lib/features/auth/presentation/create_account_screen.dart` | `/onboarding/account` | Celular + contraseña | `auth/register/passenger` + `auth/login` | ✅ **NUEVA, IMPLEMENTED** (sin commitear) |
| 4 | Verifica tu número | `lib/features/auth/presentation/phone_verification_screen.dart` | `/onboarding/phone-verification` | Gate de teléfono, sin campo de código | ninguna activa (métodos preparados, no invocados) | 🟡 **NUEVA, PROVISIONAL POR DISEÑO** (sin commitear) |
| 5 | Inicio de onboarding (placeholder pasos 2-5) | `lib/features/driver/presentation/onboarding/driver_onboarding_start_screen.dart` | `/onboarding/start` | Anuncia próximos pasos, sin formularios | ninguna | 🔴 **PLACEHOLDER** (sin commitear) |
| 6 | Rechazada | `.../driver_onboarding_rejected_screen.dart` | `/onboarding/rejected` | Motivo de rechazo + reintentar | — (usa datos ya cargados) | ✅ **NUEVA, IMPLEMENTED** (sin commitear) |
| 7 | En revisión | `.../driver_onboarding_review_screen.dart` | `/onboarding/review-status` | Confirma envío, sin SLA inventado | — | ✅ **NUEVA, IMPLEMENTED** (sin commitear) |
| 8 | Suspendida | `.../driver_onboarding_suspended_screen.dart` | `/suspended` | Motivo de suspensión, sin apelación | — | ✅ **NUEVA, IMPLEMENTED** (sin commitear) |
| 9 | Error de estado | `.../driver_onboarding_state_error_screen.dart` | `/onboarding/state-error` | Red de seguridad ante estado inconsistente/desconocido | `resolveSessionState` (reintentar) | ✅ **NUEVA, IMPLEMENTED** (sin commitear) |
| 10 | Home (con bottom nav) | `lib/features/driver/presentation/driver_home_screen.dart` | `/home` | Online/offline, mapa, tabs Inicio/Solicitudes/Ingresos/Perfil | ofertas, ubicación, heartbeat, stats | ✅ ESTABLE — Inicio y Solicitudes con contenido real; **Ingresos y Perfil son placeholders** |
| 11 | Viaje activo | `lib/features/driver/presentation/driver_active_ride_screen.dart` | `/active-ride` | Llegada (GPS), PIN, viaje en curso, no-show | llegada/arribo/inicio/completar/cancelar | ✅ ESTABLE |
| 12 | Cobro en efectivo | `lib/features/driver/presentation/driver_cash_payment_screen.dart` | `/cash-payment/:rideId` | Calculadora de vuelto, confirmar cobro | pago pendiente, confirmar | ✅ ESTABLE |
| 13 | Pago completado | `lib/features/driver/presentation/driver_completed_payment_screen.dart` | `/completed-payment/:rideId` | Recibo de finalización | — | ✅ ESTABLE |

**Confirmado NOT IMPLEMENTED** (Driver): Sobre ti / Tu mototaxi / Tus documentos / Revisar y enviar (Pasos 2-5), pestaña Ingresos real, pestaña Perfil real, SOS/emergencia (ninguna evidencia en todo el repo).

---

## 13. Backend data available — matriz por pantalla/entidad

| Dato | Passenger lo ve | Driver lo ve | Admin lo ve | Backend lo tiene |
|---|---|---|---|---|
| Nombre/apellido del Passenger | propio | del pasajero, en oferta/viaje activo | sí | sí |
| Nombre/apellido del Driver | del conductor, en oferta/viaje activo | propio | sí | sí |
| Foto del Driver | contextual (oferta/viaje/historial/share-link, vía token) | propio | sí | sí |
| Foto del Passenger | propio | contextual (oferta/viaje) | sí | sí |
| Placa/marca/modelo/año/color del vehículo | durante viaje/oferta activa | propio | sí | sí |
| `ownership` (propio/alquilado) | no | propio | sí | sí |
| Rating del Driver | durante oferta/viaje | propio | sí | sí |
| Rating del Passenger | propio | posible durante oferta (no confirmado a nivel endpoint) | sí | sí |
| Teléfono | propio | propio | sí | sí — **nunca se muestra el teléfono de la otra parte en ninguna pantalla auditada** |
| Correo (Driver, opcional) | no | propio | sí | sí |
| DNI/CE (Driver) | no | propio | sí, solo admin | sí — **nunca a Passenger** |
| Fecha de nacimiento (Driver) | no | propio | sí | sí |
| Dirección (Driver, legacy/opcional) | no | propio | sí | sí, ya no obligatoria |
| Motor/chasis (Driver, legacy) | no | propio | sí | sí, ya no obligatoria |
| Contacto de emergencia (Passenger) | propio (edita) | no | no en flujo normal | sí |
| `passengerOfferFare` / `agreedFare` / `finalFare` | sí | sí | sí | sí |
| `estimatedFare` | **NUNCA se muestra** | **NUNCA se muestra** | posible | sí — interno, "recomendación TukiTuki" en comentario de código |
| Estado del viaje (`RideStatus`) | sí | sí | sí | sí, dirige la pantalla |
| PIN de inicio de viaje | **solo Passenger** (Driver lo recibe de palabra) | no lo ve en pantalla, lo ingresa | no | sí, generado server-side |
| `RidePayment` (método/estado/vuelto) | propio viaje | propio viaje | sí | sí |
| Motivo de rechazo/suspensión (Driver) | no aplica | propio | sí | sí |

Reglas de privacidad ya soportadas por roles (`ADMIN`/`SUPER_ADMIN` vía `RolesGuard`); no se auditó endpoint por endpoint que ninguno exponga `fileUrl` a terceros — sigue como pendiente heredado de `estado-proyecto.md`.

---

## 14. Design system actual (auditado, no inventado)

**Ambas apps comparten el mismo patrón de inconsistencia**: el `ThemeData` global de Flutter (`MaterialApp`) es un seed Material 3 genérico (`colorSchemeSeed: Colors.amber`), **desconectado** de la paleta de marca real, que vive como constantes privadas repetidas por archivo (no un sistema de tokens centralizado). Esto es seguro de formalizar sin cambiar el resultado visual, ya que los valores ya coinciden entre archivos.

### Passenger — paleta real en uso

| Uso | Hex |
|---|---|
| Verde oscuro (marca / headers / texto sobre amarillo) | `#123B26` |
| Verde (acento, éxito, links) | `#1F7A3E` |
| Verde secundario | `#5C8A17` |
| Amarillo CTA (botón principal) | `#FFC72C` |
| Crema (fondo) | `#FFF9EC` |
| Crema secundaria (cards/inputs) | `#FBF7EA` |
| Borde | `#E7E0CB` / `#EFE8D4` |
| Texto primario | `#16241C` |
| Texto secundario | `#7C8A79` (Home) / `#6F7E72` (auth/ride — pequeña inconsistencia, no un solo constante compartido) |
| Acento destino (marcador) | `#D8542C` |
| Pago pendiente | texto `#B8790C` sobre `#FFF3D9` |
| Pago pagado (fondo) | `#E3F1E7` |
| Estado terminal/cancelado (fondo) | `#FBE7DE` |

Componentes: `FilledButton`/`FilledButton.icon`, `TextField`/`TextFormField` con `OutlineInputBorder` (radio 14-15), "chips" de métrica (ícono + texto), sin uso de `Card` (usan `Container`+`BoxDecoration`), mapa vía `google_maps_flutter` directo.

### Driver — paleta real en uso (`DriverPalette`, `lib/core/theme/driver_palette.dart`)

| Token | Hex |
|---|---|
| `amberLight` | `#FFDD6B` |
| `amber` | `#FFC72C` |
| `orange` | `#E8951A` |
| `orangeDeep` | `#C97313` |
| `greenPrimary` | `#123B26` |
| `greenAvailable` | `#24A24A` |
| `cream` | `#FFF9EC` |
| `brown` | `#6B4E14` |
| `coral` (errores) | `#E05B4F` |

Colores adicionales usados inline, no incluidos en `DriverPalette`: `#E7E0CB` (bordes), `#FBF7EA` (fondo de inputs), `#7C8A79` (labels), `#9A8F6E` (número inactivo del step-tracker).

Componentes: `FilledButton` (radio 17, 54px alto, sin elevación), inputs con `OutlineInputBorder` radio 16, tarjetas de estado (`DriverOnboardingInfoCard`: label naranja mayúscula + cuerpo verde), badge de ícono 72×72 (radio 20, fondo verde, ícono ámbar) — patrón nuevo introducido por el onboarding, primer conjunto de componentes realmente reutilizables del repo Driver. Step tracker: `DriverOnboardingProgress`, círculos 26×26.

### Diferencias Driver vs Passenger que Designer debe conocer (no corregidas en esta auditoría, solo reportadas)

- Verde de marca (`#123B26`) coincide entre ambas apps — es consistente.
- Passenger usa `#1F7A3E`/`#5C8A17` como acentos verdes; Driver usa `#24A24A` — tonos de verde distintos entre apps para un rol visual similar.
- El texto secundario tiene 3 variantes de gris-verde ligeramente distintas entre los dos repos y hasta dentro del mismo repo (`#7C8A79` / `#6F7E72` en Passenger; `#7C8A79` en Driver).
- Radio de borde de botones: 14-15 en Passenger (implícito) vs 17 explícito en Driver.
- Ninguna de las dos apps tiene una librería de componentes compartida (`Button`/`TextField`/`Card` propios) — cada pantalla repite su propio `InputDecoration`/`ButtonStyle`.

---

## 15. Assets

| Repo | Asset | Path | Estado |
|---|---|---|---|
| Driver | Logo | `assets/images/tukituki_driver_logo.png` | ACTIVE — único asset declarado en `pubspec.yaml` |
| Passenger | Logo | `assets/images/tukituki_logo.png` | ACTIVE — único asset declarado en `pubspec.yaml` |

**Ninguna de las dos apps tiene**: set de íconos propio, ilustraciones, íconos de vehículo, avatares placeholder como archivo (los avatares/placeholders se construyen en código con iniciales/íconos, no con imágenes). Todos los "íconos" visibles son glifos `Icons.*` de Material, no arte custom. Este es un vacío real de diseño visual — cualquier ilustración, ícono custom o placeholder de avatar que el Designer proponga es, por definición, contenido nuevo a crear, no un reemplazo de algo existente.

---

## 16. Pantallas estables (KEEP/POLISH) — no rediseñar desde cero

El historial de commits reciente de ambos repos (`redesign passenger ride searching experience`, `redesign passenger completed ride payment flow`, `build passenger driver tracking flow`, `redesign driver arrived and pin flow`, `redesign driver in-progress ride flow`, `redesign driver paid ride completion flow`, etc.) indica que estas pantallas **ya pasaron por una ronda de rediseño reciente y aprobada** — tratarlas como KEEP/POLISH, no REDESIGN:

**Passenger**: Splash, Login, Register, Complete profile, Home, Ride searching/tracking (todo el ciclo), Ride receipt/rating.

**Driver**: Splash, Login, Home (tabs Inicio/Solicitudes), Active ride (todo el ciclo: llegada/PIN/en curso), Cash payment, Completed payment.

**POLISH sugerido** (no bloqueante, cosmético): unificar los grises de texto secundario y el verde de acento entre Driver y Passenger (sección 14); formalizar la paleta de cada app en un archivo de tokens central en vez de constantes repetidas por archivo — sin cambiar ningún valor real.

---

## 17. Pantallas que requieren rediseño/mejora activa

- **Driver — Pasos 2 a 5 del onboarding** (Sobre ti / Tu mototaxi / Tus documentos / Revisar y enviar): no existen, máxima prioridad (ver briefs en sección 24).
- **Driver — pestaña Ingresos**: placeholder, sin contenido real todavía definido por producto más allá de `DriverDailyStats` ya existente en Backend.
- **Driver — pestaña Perfil**: placeholder, sin contenido real todavía definido.
- **Driver — pantalla "Verifica tu número"**: funcional pero deliberadamente mínima (sin campo de código); si el diseño quiere mejorarla visualmente, debe mantenerse fiel a que **no hay verificación real por SMS todavía** — no diseñar un flujo de "ingresa el código" que no existe.
- **Passenger — Home**: estable funcionalmente, pero el copy de estados secundarios del CTA (loading/error) no fue confirmado línea por línea en esta auditoría — recomendable revisar antes de tocar visualmente.

---

## 18. Pantallas que faltan por completo (NEW SCREEN NEEDED)

- Driver: Sobre ti, Tu mototaxi, Tus documentos, Revisar y enviar, Ingresos (contenido real), Perfil (contenido real).
- Passenger: "Has llegado a tu destino" + confirmación opcional.
- Ambas apps: SOS/emergencia visible al usuario (Backend ya soporta incidentes/contactos), compartir viaje/share-link visible al usuario (Backend ya soporta `RideShareLink`).
- Passenger: pantalla/acción de disputa de pago en efectivo (Backend ya tiene el endpoint).

Ninguna de estas tiene mockups o decisiones visuales previas — son diseño desde cero, respetando los datos y contratos reales descritos en este documento.

---

## 19. Restricciones funcionales (mobile constraints)

- Flutter, **solo Android** (ninguna de las dos apps tiene carpeta iOS).
- `google_maps_flutter` real en ambas apps — cualquier pantalla con mapa debe considerar el mapa nativo de Google, no un placeholder.
- Sin WebSocket/push consumido — cualquier actualización en vivo en el diseño debe asumir polling (con el retraso correspondiente, 1-30 s según pantalla), nunca "tiempo real instantáneo".
- Sin caché/DB local — cualquier pantalla debe contemplar estado de carga tras cada navegación, no placeholders con datos previos.
- Teclado: los formularios (login, crear cuenta, oferta de tarifa, futuros pasos 2-5) deben funcionar con teclado abierto en pantallas pequeñas Android — no se documentó explícitamente el manejo de `scroll`/`resize` pero es requisito implícito de cualquier form Flutter Android.
- Firma de release: ambas apps compilan `release` firmado con clave de **debug** hoy — no apto para publicación tal cual (fuera del alcance de diseño, pero relevante si el Designer entrega assets pensando en distribución real).
- `Money` siempre se maneja como `String`/centavos enteros en el dominio, nunca `double` — no afecta el diseño visual pero confirma que los montos que se muestran vienen ya formateados del cliente, no requieren redondeo especial en la propuesta visual.

---

## 20. DESIGN DO / DON'T

**DESIGNER MUST:**
- Respetar el flujo de negociación de una sola ronda (sección 8) y el ciclo de vida del viaje (sección 7) tal cual están implementados.
- Mantener el branding **TukiTuki**.
- Priorizar simplicidad — usuarios de mototaxi, contexto de uso real (calle, sol, una mano ocupada).
- Considerar pantallas Android pequeñas y el teclado abierto.
- Diseñar explícitamente los estados loading/error/empty en toda pantalla nueva, dado que no hay caché local.
- Mantener el CTA principal claro y único por pantalla (patrón ya usado: un botón primario grande, texto imperativo — "Ofrecer y buscar conductor", "Crear cuenta", etc.).
- Para Driver Onboarding pasos 2-5, seguir el lenguaje visual ya establecido por los componentes nuevos: `DriverOnboardingScaffold` (badge de ícono + título + subtítulo), `DriverOnboardingInfoCard` (tarjeta de estado), `DriverOnboardingProgress` (tracker de 5 pasos) — ya existen y están commiteados como el lenguaje visual del onboarding.

**DESIGNER MUST NOT:**
- Inventar features (chat, llamada telefónica, wallet, tarjeta de crédito, Yape integrado, códigos promo, paradas múltiples, viajes programados, favoritos).
- Cambiar el modelo de negociación (una sola ronda, sin contraoferta del Passenger de vuelta al Driver).
- Mostrar "Precio recomendado TukiTuki" o cualquier variante de `estimatedFare` como sugerencia visible.
- Agregar pagos integrados/pasarela visible (Izipay/Yape) — no están activos.
- Inventar ETA de llegada del conductor.
- Inventar/asumir un chat o llamada telefónica Passenger↔Driver — no existe ningún canal de comunicación en la app; el PIN se comunica **verbalmente** fuera de la app.
- Cambiar la cantidad de documentos del Driver (son 3: Licencia, SOAT, TIV — no 6).
- Saltarse los estados de aprobación del Driver (DRAFT → PENDING_REVIEW → APPROVED/REJECTED/SUSPENDED) ni permitir llegar a Home sin `APPROVED` + rol `DRIVER`.
- Diseñar un campo de código OTP funcional para Driver o Passenger como si SMS real ya existiera — el gate de teléfono en Driver es intencionalmente un mensaje de verificación manual, no un input de código.
- Convertir ningún prototipo visual en contrato de Backend — cualquier pantalla nueva (Sobre ti, Tu mototaxi, Tus documentos, SOS, share-link, disputa de pago) debe usar exactamente los campos/entidades reales listados en la sección 13, no inventar campos nuevos.

---

## 21. UI state matrix (pantallas clave)

| Pantalla | EMPTY | LOADING | SUCCESS | ERROR | EXPIRED/OTRO |
|---|---|---|---|---|---|
| Passenger Home (destino) | campo vacío, mapa sin ruta | spinner en autocompletado/cotización | ruta + chips + oferta habilitada | error de red ("no se pudo conectar") | cotización vencida → renueva automático |
| Passenger Ride searching | — | buscando conductor (spinner/progreso) | lista de propuestas | — | búsqueda vencida (5 min) sin match |
| Passenger PIN | — | — | PIN visible con cuenta regresiva | intentos agotados → bloqueado, requiere regenerar | TTL vencido (15 min) → requiere regenerar |
| Driver Solicitudes | sin ofertas visibles | polling (3 s) | lista con badge de conteo | error de red | oferta expirada se retira de la lista |
| Driver Onboarding (cualquier paso) | — | validando estado (`resolveSessionState`) | pantalla correspondiente al estado | 400/409/5xx con mensajes específicos ("ya existe una cuenta", "TukiTuki no disponible") | — |
| Driver Cash payment | — | confirmando cobro | recibo | — | — |

---

## 22. Roadmap de diseño priorizado

1. **Driver Onboarding Paso 2 — Sobre ti** (foto, nombres, apellidos, DNI/CE, fecha nacimiento, correo opcional).
2. **Driver Onboarding Paso 3 — Tu mototaxi** (placa, marca, modelo, año, color, propio/alquilado).
3. **Driver Onboarding Paso 4 — Tus documentos** (upload de Licencia/SOAT/TIV vía presign/PUT/complete ya validado en staging).
4. **Driver Onboarding Paso 5 — Revisar y enviar** (resumen + confirmación + `POST drivers/me/submit`).
5. **Driver — pestaña Ingresos** (contenido real sobre `DriverDailyStats`).
6. **Driver — pestaña Perfil** (contenido real).
7. **Passenger — "Has llegado a tu destino"** (si producto decide construirlo; hoy no hay soporte de Backend, requiere decisión de producto primero — ver sección 23).
8. **SOS/emergencia visible** en ambas apps (Backend ya listo).
9. **Compartir viaje/share-link visible** (Backend ya listo).
10. **Disputa de pago en efectivo visible** en Passenger (Backend ya listo).
11. Polish cosmético cross-app (unificar grises/verdes entre Driver y Passenger, formalizar tokens de color).

---

## 23. Preguntas de producto abiertas (PRODUCT DECISION REQUIRED)

Solo las que siguen genuinamente sin resolver — no se repiten aquí decisiones ya cerradas en `estado-proyecto.md`:

- **"Has llegado a tu destino" + confirmación opcional del Passenger**: Backend no tiene ningún modelo para esto todavía. Opciones reales: (a) construir el endpoint/estado en Backend primero y luego la UI, o (b) mantenerlo fuera de la demo actual. Bloquea: cualquier diseño de esta pantalla necesita que producto decida el contrato de datos antes de que Designer proponga la interacción exacta.
- **Disputa de pago en efectivo**: el endpoint ya existe en Backend pero ninguna app lo usa. ¿Se agrega a la demo actual o queda para después? Bloquea: si Designer diseña esta pantalla, debe confirmarse primero si entra en el alcance de la demo funcional priorizada.
- **SOS/share-link visibles al usuario**: Backend completo, sin UI en ninguna app. Mismo tipo de pregunta — ¿entra en el alcance de la demo actual o es "futuro"? No es una decisión de UX sino de alcance/prioridad de producto.
- Todo lo demás (producción, dominio, proveedor SMS, negociación single-round, 3 documentos, sin dirección en onboarding) ya está decidido y documentado en `estado-proyecto.md` — no se reabre aquí.

---

## 24. Screen Design Briefs

Formato: `SCREEN / PURPOSE / USER / ENTRY / DATA AVAILABLE / PRIMARY CTA / SECONDARY CTA / STATES / NEXT / MUST SHOW / MUST NOT SHOW / BACKEND DEPENDENCY / CURRENT IMPLEMENTATION FILE`.

### Driver — Onboarding

**SCREEN: Tu cuenta**
- PURPOSE: crear la cuenta del conductor (celular + contraseña).
- USER: aspirante a conductor, sin cuenta.
- ENTRY: desde Login ("Crea tu cuenta") o primer uso de la app.
- DATA AVAILABLE: ninguno todavía — es el primer paso.
- PRIMARY CTA: "Crear cuenta".
- SECONDARY CTA: volver a Login.
- STATES: loading ("Creando cuenta..."), error (409 cuenta existente / 400 datos inválidos / 5xx / sin red).
- NEXT: verificación de teléfono.
- MUST SHOW: indicador de progreso (paso 1 de 5).
- MUST NOT SHOW: campo de código OTP en esta pantalla (no aplica aquí).
- BACKEND DEPENDENCY: `POST auth/register/passenger`, `POST auth/login`.
- CURRENT IMPLEMENTATION FILE: `lib/features/auth/presentation/create_account_screen.dart` — **YA IMPLEMENTADA** (sin commitear), solo referencia/polish si aplica.

**SCREEN: Verifica tu número**
- PURPOSE: gate de verificación de teléfono antes de continuar el onboarding.
- USER: conductor recién registrado, `isPhoneVerified == false`.
- ENTRY: automático tras registro/login si el teléfono no está verificado.
- DATA AVAILABLE: número de teléfono (mostrar enmascarado).
- PRIMARY CTA: "Ya verifiqué mi número" (re-consulta el estado).
- SECONDARY CTA: "Cerrar sesión".
- STATES: consultando estado, sin cambios (permanece en esta pantalla).
- NEXT: inicio de onboarding (si se verificó) o permanece aquí.
- MUST SHOW: mensaje honesto de que la verificación es manual por el equipo TukiTuki en esta etapa.
- MUST NOT SHOW: campo de código de 6 dígitos, botón "Reenviar código", cualquier UI que implique SMS real activo.
- BACKEND DEPENDENCY: `GET auth/me` (campo `isPhoneVerified`); `POST auth/otp/request`/`verify` existen pero **no deben conectarse** en esta etapa.
- CURRENT IMPLEMENTATION FILE: `lib/features/auth/presentation/phone_verification_screen.dart` — **YA IMPLEMENTADA** (sin commitear), deliberadamente mínima.

**SCREEN: Sobre ti**
- PURPOSE: capturar datos personales del conductor.
- USER: conductor con teléfono verificado, sin perfil o en `DRAFT`.
- ENTRY: desde el placeholder de inicio de onboarding.
- DATA AVAILABLE: ninguno propio todavía (primer formulario de datos).
- PRIMARY CTA: "Continuar" (a Tu mototaxi).
- SECONDARY CTA: volver / cerrar sesión.
- STATES: vacío, validando campos, error de envío.
- NEXT: Tu mototaxi.
- MUST SHOW: foto (upload), nombres, apellidos, tipo+número de documento (DNI/CE), fecha de nacimiento, correo (opcional), indicador de progreso (paso 2 de 5).
- MUST NOT SHOW: dirección domiciliaria, número de motor/chasis/VIN (excluidos por decisión de producto).
- BACKEND DEPENDENCY: `DriverProfile` (firstName, lastName, documentType, documentNumber, birthDate, email opcional, photoUrl/photoObjectKey vía Storage presign/PUT/complete).
- CURRENT IMPLEMENTATION FILE: no existe — **NEW SCREEN NEEDED**.

**SCREEN: Tu mototaxi**
- PURPOSE: capturar datos del vehículo.
- USER: mismo conductor, tras completar Sobre ti.
- ENTRY: desde Sobre ti.
- DATA AVAILABLE: datos ya ingresados en Sobre ti (no mostrados aquí necesariamente).
- PRIMARY CTA: "Continuar" (a Tus documentos).
- SECONDARY CTA: volver.
- STATES: vacío, validando, error (ej. placa duplicada).
- NEXT: Tus documentos.
- MUST SHOW: placa, marca, modelo, año, color, propio/alquilado (`ownership`), indicador de progreso (paso 3 de 5).
- MUST NOT SHOW: número de motor/chasis/VIN como campos obligatorios.
- BACKEND DEPENDENCY: `DriverVehicle` (plate, brand, model, year, color, ownership requerido).
- CURRENT IMPLEMENTATION FILE: no existe — **NEW SCREEN NEEDED**.

**SCREEN: Tus documentos**
- PURPOSE: subir los 3 documentos del expediente.
- USER: mismo conductor, tras Tu mototaxi.
- ENTRY: desde Tu mototaxi.
- DATA AVAILABLE: ninguno propio (upload puro).
- PRIMARY CTA: "Continuar" (a Revisar y enviar).
- SECONDARY CTA: volver.
- STATES: vacío por documento, subiendo (progreso por archivo), subido, error de subida/tipo de archivo/tamaño.
- NEXT: Revisar y enviar.
- MUST SHOW: exactamente 3 documentos — Licencia, SOAT, Tarjeta de propiedad/TIV; indicador de progreso (paso 4 de 5).
- MUST NOT SHOW: DNI frontal/posterior como documentos separados (el DNI ya se capturó como texto en Sobre ti), foto de perfil (ya es parte de Sobre ti, no un documento del expediente), un 4º/5º/6º documento.
- BACKEND DEPENDENCY: flujo Storage presign→PUT→complete ya validado en staging; tipos `DRIVER_LICENSE`, `SOAT`, `VEHICLE_REGISTRATION`.
- CURRENT IMPLEMENTATION FILE: no existe — **NEW SCREEN NEEDED**.

**SCREEN: Revisar y enviar**
- PURPOSE: resumen final antes de enviar la solicitud.
- USER: mismo conductor, tras completar los 3 pasos de datos.
- ENTRY: desde Tus documentos.
- DATA AVAILABLE: todo lo capturado en pasos 2-4 (resumen editable/no editable, a definir por Designer).
- PRIMARY CTA: "Enviar solicitud".
- SECONDARY CTA: editar cada sección (volver al paso correspondiente).
- STATES: revisando, enviando, error de envío (ej. foto de perfil faltante — el Backend la exige antes de permitir el envío).
- NEXT: Solicitud en revisión.
- MUST SHOW: resumen de los 3 bloques de datos, indicador de progreso (paso 5 de 5).
- MUST NOT SHOW: ningún campo nuevo no capturado en pasos anteriores.
- BACKEND DEPENDENCY: `POST drivers/me/submit` (exige `photoObjectKey`/`photoUrl` presente como última barrera).
- CURRENT IMPLEMENTATION FILE: no existe — **NEW SCREEN NEEDED**.

**SCREEN: Solicitud en revisión**
- PURPOSE: informar que el expediente fue enviado y está pendiente de aprobación.
- USER: conductor con `status == PENDING_REVIEW`.
- ENTRY: automático tras envío exitoso, o al reabrir la app en este estado.
- DATA AVAILABLE: ninguno adicional.
- PRIMARY CTA: ninguno de avance (es un estado de espera).
- SECONDARY CTA: "Cerrar sesión".
- STATES: estático (solo este estado).
- NEXT: (fuera de la app) admin aprueba/rechaza → próxima vez que abre la app, pasa a Home o a Rechazada.
- MUST SHOW: confirmación de envío, mensaje sin promesa de tiempo (no hay SLA real).
- MUST NOT SHOW: estimación de tiempo tipo "24 horas" (no existe SLA).
- BACKEND DEPENDENCY: `DriverProfile.status == PENDING_REVIEW`.
- CURRENT IMPLEMENTATION FILE: `lib/features/driver/presentation/onboarding/driver_onboarding_review_screen.dart` — **YA IMPLEMENTADA** (sin commitear).

**SCREEN: Solicitud rechazada**
- PURPOSE: informar el rechazo y permitir corregir.
- USER: conductor con `status == REJECTED`.
- ENTRY: automático al abrir la app en este estado.
- DATA AVAILABLE: `rejectionReason` (si el admin lo indicó).
- PRIMARY CTA: "Corregir solicitud" (hoy vuelve al placeholder de inicio, porque los pasos 2-5 no existen — al construirlos, debería llevar de vuelta al formulario real).
- SECONDARY CTA: "Cerrar sesión".
- STATES: con motivo / sin motivo.
- NEXT: nuevo envío (una vez existan los pasos 2-5).
- MUST SHOW: motivo si está disponible.
- MUST NOT SHOW: culpar al usuario sin motivo — si no hay `rejectionReason`, no inventar uno genérico ofensivo.
- BACKEND DEPENDENCY: `DriverProfile.status == REJECTED`, `rejectionReason`.
- CURRENT IMPLEMENTATION FILE: `lib/features/driver/presentation/onboarding/driver_onboarding_rejected_screen.dart` — **YA IMPLEMENTADA** (sin commitear).

**SCREEN: Suspendido**
- PURPOSE: informar la suspensión de la cuenta.
- USER: conductor con `status == SUSPENDED`.
- ENTRY: automático al abrir la app en este estado.
- DATA AVAILABLE: `suspensionReason` (si existe).
- PRIMARY CTA: ninguno (no puede continuar).
- SECONDARY CTA: "Cerrar sesión".
- STATES: con motivo / sin motivo.
- NEXT: ninguno dentro de la app.
- MUST SHOW: motivo si está disponible.
- MUST NOT SHOW: canal de apelación inventado (no existe hoy).
- BACKEND DEPENDENCY: `DriverProfile.status == SUSPENDED`, `suspensionReason`.
- CURRENT IMPLEMENTATION FILE: `lib/features/driver/presentation/onboarding/driver_onboarding_suspended_screen.dart` — **YA IMPLEMENTADA** (sin commitear).

### Driver — Operación

**SCREEN: Home**
- PURPOSE: hub principal del conductor aprobado — disponibilidad, mapa, navegación a Solicitudes/Ingresos/Perfil.
- USER: conductor `APPROVED` con rol `DRIVER`.
- ENTRY: único destino posible tras la máquina de estados de sesión.
- DATA AVAILABLE: estado online/offline, ubicación GPS propia, estadísticas del día (`DriverDailyStats`).
- PRIMARY CTA: toggle online/offline.
- SECONDARY CTA: navegación por bottom nav.
- STATES: restaurando sesión, error, recuperando viaje en curso (`busyRecovery`), offline, disponible.
- NEXT: Solicitudes (ver ofertas) o Viaje activo (si hay uno en curso).
- MUST SHOW: mapa real, conteo de solicitudes pendientes (badge).
- MUST NOT SHOW: nada de otros conductores (privacidad).
- BACKEND DEPENDENCY: ubicación (`PUT drivers/me/location`), heartbeat, `DriverDailyStats`.
- CURRENT IMPLEMENTATION FILE: `lib/features/driver/presentation/driver_home_screen.dart` — **YA IMPLEMENTADA/ESTABLE**, solo pulir si aplica.

**SCREEN: Solicitudes**
- PURPOSE: ver y responder ofertas de viaje.
- USER: conductor disponible.
- ENTRY: tab del bottom nav.
- DATA AVAILABLE: por oferta — nombre del pasajero, `passengerOfferFare`, origen, destino, distancia.
- PRIMARY CTA: "Aceptar".
- SECONDARY CTA: "Contraofertar" (una vez), "Rechazar".
- STATES: sin ofertas (empty), lista con badge, oferta expirando/expirada.
- NEXT: si el Passenger elige esta propuesta → Viaje activo.
- MUST SHOW: aviso claro de que aceptar/contraofertar **no asigna el viaje todavía** (el Passenger decide).
- MUST NOT SHOW: negociación multi-ronda, contraoferta ilimitada.
- BACKEND DEPENDENCY: lista de ofertas (polling 3 s), aceptar/contraofertar/rechazar.
- CURRENT IMPLEMENTATION FILE: `driver_home_screen.dart` (`_buildSolicitudesTab`) + `driver_counter_offer_dialog.dart` — **YA IMPLEMENTADA/ESTABLE**.

### Passenger — pantallas que podrían necesitar polish (según hallazgos de esta auditoría)

**SCREEN: Home (revisión de copy de estados secundarios)**
- PURPOSE: confirmar que todos los estados del CTA (no solo el primario) cumplen la regla "sin precio recomendado" y mantienen el texto exacto.
- CURRENT IMPLEMENTATION FILE: `lib/features/home/home_screen.dart` — recomendable una lectura dirigida de las líneas ~1450-1500 (estados alternativos del botón) antes de tocar visualmente esta pantalla, ya que esta auditoría solo confirmó con certeza el estado primario ("Ofrecer y buscar conductor").

---

## 25. Referencias de archivos citados en este documento

**Backend** (`tukituki-backend`, branch `test/otp-demo-staging`@`fd9ba8b3`, 1 commit sin fusionar sobre `origin/main`@`f147a664`):
`src/modules/rides/enums/ride-status.enum.ts`, `src/modules/rides/enums/ride-offer-status.enum.ts`, `src/modules/rides/ride-matching.constants.ts`, `src/modules/drivers/enums/driver-status.enum.ts`, `src/modules/drivers/driver-application.constants.ts`, `src/modules/drivers/entities/driver-profile.entity.ts`, `src/modules/drivers/entities/driver-vehicle.entity.ts`, `src/modules/passengers/entities/passenger-profile.entity.ts`, `src/modules/rides/entities/ride.entity.ts`, `src/modules/rides/ride-lifecycle.constants.ts`, `src/modules/rides/ride-start-codes.service.ts`, `src/modules/payments/entities/ride-payment.entity.ts`, `src/modules/payments/passenger-cash-payments.controller.ts`, `src/modules/rides/realtime/rides.gateway.ts`, `src/modules/safety/realtime/shared-rides.gateway.ts`, `src/modules/safety/*` (emergency contacts, incidents, share links), `src/modules/auth/otp.service.ts`, `src/config/env.validation.ts`.

**Driver** (`tukituki-driver-app`, branch `test/driver-onboarding-r3`@`b562d24` + cambios locales sin commitear):
`lib/core/router/app_router.dart`, `lib/core/router/driver_onboarding_routes.dart` (nuevo), `lib/features/auth/presentation/driver_splash_screen.dart`, `lib/features/auth/presentation/login_screen.dart`, `lib/features/auth/presentation/create_account_screen.dart` (nuevo), `lib/features/auth/presentation/phone_verification_screen.dart` (nuevo), `lib/features/auth/domain/authenticated_user.dart` (nuevo), `lib/features/auth/domain/driver_session_state.dart` (nuevo), `lib/features/auth/data/auth_repository.dart`, `lib/features/driver/domain/driver_application.dart` (nuevo), `lib/features/driver/presentation/onboarding/*` (7 archivos nuevos), `lib/features/driver/presentation/driver_home_screen.dart`, `lib/features/driver/presentation/driver_active_ride_screen.dart`, `lib/features/driver/presentation/driver_cash_payment_screen.dart`, `lib/features/driver/presentation/driver_completed_payment_screen.dart`, `lib/core/theme/driver_palette.dart`, `lib/app.dart`.

**Passenger** (`tukituki-passenger-app`, `main`@`a5d2411`):
`lib/core/router/app_router.dart`, `lib/features/auth/presentation/splash_screen.dart`, `lib/features/auth/presentation/login_screen.dart`, `lib/features/auth/presentation/register_screen.dart`, `lib/features/auth/presentation/otp_screen.dart`, `lib/features/passenger/presentation/complete_profile_screen.dart`, `lib/features/home/home_screen.dart`, `lib/features/ride/presentation/ride_searching_screen.dart`, `lib/features/ride/presentation/ride_receipt_screen.dart`, `lib/app.dart`.

---

# CLAUDE DESIGNER — NON-NEGOTIABLE PRODUCT RULES

Lee esto en menos de 2 minutos antes de diseñar cualquier pantalla de TukiTuki.

1. **TukiTuki** es una app de mototaxi bajo demanda: Passenger pide, Driver atiende, Admin aprueba conductores, Backend es la única autoridad de negocio.
2. **Passenger pide un viaje así**: elige destino → ve cotización (distancia/duración, mapa con ruta) → escribe **su propia oferta de precio** (campo vacío, sin sugerencia de "precio recomendado") → toca "Ofrecer y buscar conductor".
3. **Negocia así**: Driver ve la oferta y puede ACEPTAR (mismo precio) o CONTRAOFERTAR **una sola vez**. Passenger ve todas las respuestas y **elige una**. Recién ahí se fija el precio (`agreedFare`) y **recién ahí** el Driver queda asignado/ocupado. Nunca hay negociación multi-ronda ni el Passenger vuelve a contraofertar.
4. **Onboarding de Driver, 6 pasos aprobados**: 1) Tu cuenta 2) Sobre ti (foto, nombres, apellidos, DNI/CE, fecha nacimiento, correo opcional — **sin dirección**) 3) Tu mototaxi (placa, marca, modelo, año, color, propio/alquilado — **sin motor/chasis/VIN**) 4) Tus documentos (**solo 3**: Licencia, SOAT, TIV) 5) Revisar y enviar 6) Solicitud en revisión. Hoy solo el paso 1 y el paso 6 (más rechazado/suspendido/error) tienen pantalla real; **pasos 2-5 son la prioridad de diseño número uno**.
5. **Estados de Driver**: DRAFT → PENDING_REVIEW → APPROVED/REJECTED, más SUSPENDED. Solo `APPROVED` + rol `DRIVER` en la cuenta lleva a Home. Nunca diseñes una ruta que salte este gate.
6. **Estados de viaje**: SEARCHING_DRIVER → DRIVER_ASSIGNED → DRIVER_ARRIVING → DRIVER_ARRIVED (con **PIN de 4 dígitos** que el Passenger ve y le dice de palabra al Driver) → IN_PROGRESS → COMPLETED (o CANCELLED/EXPIRED). No hay pantalla de "llegaste a tu destino" con confirmación — no existe hoy.
7. **Pago**: solo **efectivo**, directo Passenger↔Driver. TukiTuki no retiene el dinero. No hay Yape/tarjeta/Izipay activo en ninguna app.
8. **OTP/SMS**: el núcleo de verificación está listo en Backend, pero **no hay SMS real todavía** (diferido hasta después de la demo). No diseñes un campo de "ingresa el código" — hoy el gate de teléfono en Driver es un mensaje de verificación manual.
9. **Features que NO existen y no debes dar por hechas**: chat, llamada telefónica, ETA de llegada, wallet, tarjeta de crédito, Yape/Izipay activo, códigos promocionales, paradas múltiples, viajes programados, favoritos, SOS visible al usuario, compartir viaje visible al usuario, disputa de pago visible, notificación push/vibración instantánea (todo es polling HTTP), analítica avanzada de ingresos para Driver.
10. Cuando falte un dato para diseñar algo, **no lo inventes** — usa la matriz de datos disponibles (sección 13 del documento completo) o pregunta.
