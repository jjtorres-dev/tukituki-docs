# glosario

Repositorio:
tukituki-driver-app

Branch analizada:
main

Commit analizado:
2857ff227086442f1589a24f31626dc1b801d426

Última actualización:
2026-08-18

Fuente de verdad:
Este documento es contexto auxiliar. Si contradice al código actual,
el código y los tests tienen prioridad.

---

## Entidades (domain/)

**DriverActiveRide** — Viaje activo del conductor: id, status, tarifas (`estimatedFare`/`agreedFare`), origen/destino (dirección + coordenadas), distancias, y el `AssignedPassenger`. `lib/features/driver/domain/driver_active_ride.dart`.

**AssignedPassenger** — Resumen mínimo del pasajero asignado a un ride (`profileId`, `firstName`, rating). Deliberadamente no incluye apellido/teléfono/email/documento porque Backend no los expone en este contrato. `driver_assigned_passenger.dart`.

**DriverRideOffer** — Una oferta de viaje ofrecida al conductor (`OFFERED`) o ya propuesta por él (`PROPOSED`), con `estimatedFare` (precio sugerido por TukiTuki), `passengerOfferFare` (precio inicial elegido por el pasajero) y `proposedFare` (contraoferta del conductor). `driver_ride_offer.dart`.

**DriverPendingProposal** — Una `DriverRideOffer` en estado `PROPOSED`, ya enviada por el conductor y pendiente de que el pasajero decida; se obtiene por separado vía `GET drivers/me/ride-offers/proposals/pending`. `driver_pending_proposal.dart`.

**DriverOperationalState** / **DriverOperationalStatus** — Estado de disponibilidad del conductor. Valores: `offline`, `available`, `busy`, `unknown` (mapeados de los strings backend `OFFLINE`/`AVAILABLE`/`BUSY`). `driver_operational_state.dart`.

**DriverDailyStats** — Estadísticas del día del conductor: `businessDate`, `completedRides`, `grossAmount`, calculadas y devueltas siempre por Backend. `driver_daily_stats.dart`.

**DriverPendingPayment** — Un ride `COMPLETED` cuyo `RidePayment` sigue `PENDING` (de cualquier método, no solo efectivo); se usa para restaurar la pantalla de cobro tras reiniciar la app. `driver_pending_payment.dart`.

**DriverRideCompletion** — Resultado de completar un viaje (`POST .../complete`): tarifa final, descuento, monto que debe el pasajero, si la tarifa fue "capeada" (`fareWasCapped`), método y estado de pago. `driver_ride_completion.dart`.

**DriverRidePayment** — El pago asociado a un ride: `method` (CASH/YAPE/PLIN/CARD), `status` (PENDING/PAID/...), montos, y (si ya se cobró) `cashReceived`/`changeGiven`/`confirmedAt`. `driver_ride_payment.dart`.

**DriverRideWaiting** — Estado de la espera del conductor en el punto de recojo (Checkpoint G2): tiempos requeridos/transcurridos/restantes y si ya se puede reportar `no-show`, todo calculado por Backend. `driver_ride_waiting.dart`.

**DriverCancellationReason** (enum) — Motivos de cancelación que el conductor puede enviar realmente a Backend: `passengerRequestedCancel`, `cannotReachPickup`, `vehicleProblem`, `safetyConcern`, `emergency`, `other`. Nota: el enum de Backend también incluye `PASSENGER_NOT_FOUND`, pero ese valor está deliberadamente excluido aquí porque Backend lo rechaza en este endpoint (exige el flujo waiting/no-show en su lugar). `driver_cancellation_reason.dart`.

## Entidades del onboarding (domain/)

**AuthenticatedUser** — Usuario autenticado tal como lo devuelve `GET auth/me`: `id`, `phoneE164`, `roles`, `status` de cuenta, `isPhoneVerified`. `authenticated_user.dart`.

**DriverSessionState** / **DriverSessionKind** (enum) — Resultado del routing del onboarding, no un recurso de Backend: se calcula con la función pura `resolveDriverApplicationState()` a partir de `AuthenticatedUser` + `DriverApplication`/`DriverVehicle`/`DriverDocument` ya resueltos. Valores: `noProfile`, `draftNoVehicle`, `draftDocumentsIncomplete`, `draftDocumentsComplete`, `correctionsRequired`, `pendingReview`, `approved`, `approvedRoleMismatch`, `suspended`, `unknownApplicationStatus`. `driver_session_state.dart`.

**DriverApplication** — El perfil del conductor (`GET drivers/me`): datos personales, foto, documento de identidad, y `status` (`DriverApplicationStatus`). `driver_application.dart`.

**DriverVehicle** — El mototaxi del conductor (`GET drivers/me/vehicle`): placa, marca/modelo, año, color, `VehicleOwnership`, `status` (`VehicleStatus`). `driver_vehicle.dart`.

**DriverDocument** — Un documento del expediente (`GET drivers/me/documents`): `type` (`DriverDocumentType`), `status` (`DriverDocumentStatus`), archivo, metadata, `rejectionReason`. `driver_document.dart`.

## Estados de una `DriverApplication`/`DriverVehicle` (valores de `status`)

`DRAFT` — en construcción, todavía editable por el conductor.
`PENDING_REVIEW` — enviada, esperando revisión del admin; ya no editable.
`REJECTED` — el admin observó algo (perfil y/o vehículo y/o algún documento); vuelve a ser editable en el/los recurso(s) observado(s). Backend marca `REJECTED` en el perfil ante cualquier rechazo, aunque solo se haya observado el vehículo o un documento puntual — nunca usar `status == REJECTED` en solitario como señal de "hay algo que corregir" (ver `hasPendingDriverCorrections` en `arquitectura.md`).
`APPROVED` — aprobada; solo entra a Home si el usuario ya tiene el rol `DRIVER`.
`SUSPENDED` — cuenta de conductor suspendida tras haber estado activa.
`UNKNOWN` — Backend devolvió un valor que el cliente todavía no reconoce (fallback defensivo, nunca un estado real de Backend).

## Estados de un `DriverDocument` (`DriverDocumentStatus`)

Mismos 4 valores base que arriba (`DRAFT`/`PENDING_REVIEW`/`REJECTED`/`APPROVED`), sin `SUSPENDED` — un documento no se "suspende" independientemente del perfil.

## Tipos de documento objetivo (`DriverDocumentType`)

`DRIVER_LICENSE` (licencia de conducir), `SOAT`, `VEHICLE_REGISTRATION` (tarjeta de propiedad / TIV) — los únicos 3 que el Paso 4 "Tus documentos" ofrece y exige. Tipos legacy conservados solo para parsear defensivamente un documento antiguo si Backend lo devuelve, nunca ofrecidos como opción nueva: `DNI_FRONT`, `DNI_BACK`, `PROFILE_PHOTO`.

## Propiedad del vehículo (`VehicleOwnership`)

`OWNED` — Propio.
`RENTED` — Alquilado.

## Pantallas del onboarding (por nombre de producto)

**"Correcciones requeridas"** (`DriverOnboardingCorrectionsScreen`) — destino de `DriverSessionKind.correctionsRequired` mientras quede al menos una observación pendiente. Siempre muestra 5 secciones (Sobre ti, Tu mototaxi, Licencia, SOAT, TIV); solo las observadas ofrecen "Corregir" con el motivo exacto del admin.

**"Revisar y enviar"** (`DriverOnboardingSubmitReviewScreen`, modo inicial) — Paso 5 del flujo normal, resumen antes del primer `POST drivers/me/submit`.

**"Revisar y reenviar"** (la misma pantalla, modo "resubmission") — se alcanza solo desde "Correcciones requeridas" cuando ya no queda ninguna observación pendiente; sin botones "Editar" generales, CTA "Reenviar solicitud".

## Contexto de reenvío (resubmission context)

Marcador local no sensible (booleano, `flutter_secure_storage`, **scoped por cuenta** vía `AuthenticatedUser.id`) que indica que la solicitud actual sigue en un ciclo de corrección/reenvío tras un rechazo — necesario porque Backend no conserva historial de rechazos: un `DriverApplication` corregido vuelve a `DRAFT` sin ningún rastro de que viniera de `REJECTED`. Se marca al detectar `REJECTED`, se limpia al confirmarse un reenvío exitoso o un estado terminal (`PENDING_REVIEW`/`APPROVED`/`SUSPENDED`); sobrevive reinicios de la app y logout/login de la misma cuenta; nunca se comparte entre cuentas distintas en el mismo dispositivo.

## Estados de un Ride (valores de `status` observados en el código)

`DRIVER_ASSIGNED`, `DRIVER_ARRIVING`, `DRIVER_ARRIVED`, `IN_PROGRESS`, `COMPLETED` — nombres tal como aparecen en comentarios y lógica de `driver_rides_repository.dart` y las pantallas de `presentation/`. `UNKNOWN` es el valor de fallback cuando el cliente no reconoce el status recibido (no un estado real de Backend).

## Estados de una Offer / Proposal

`OFFERED` — oferta activa aún no respondida por el conductor.
`PROPOSED` — el conductor ya envió una contraoferta (`proposedFare`), pendiente de decisión del pasajero.

## Estados de un Payment

`PENDING` — pago aún no confirmado.
`PAID` — pago confirmado (tiene `confirmedAt`).
Métodos observados: `CASH` (único que este repo confirma activamente vía `confirmCashPayment`), `YAPE`, `PLIN`, `CARD` (mencionados como valores posibles que el cliente filtra pero no procesa).

## Otros conceptos internos

**Checkpoint** — Marcador informal usado en comentarios de código para referenciar hitos de desarrollo/iteraciones (p.ej. "Checkpoint D0", "Checkpoint G2", "Checkpoint F1"), presumiblemente ligados a un tracker externo al repo. `[PENDIENTE: no hay documento en este repo que liste o defina qué es cada checkpoint]`.

**RESTORE** (patrón, no una clase) — Lógica de arranque de `DriverHomeScreen` que reconstruye el estado (viaje activo, pagos pendientes, estado operativo) reconsultando Backend en vez de usar caché local; visible en logs de debug (`DRIVER RESTORE - ...`) y en el nombre de métodos internos de la pantalla.

**Distance-to-origin (`distanceToOriginMeters`)** — Distancia real (PostGIS del lado Backend) entre el conductor y el punto de recojo, recalculada por Backend en cada consulta; nunca calculada en el cliente.

**Quick amounts / cash ladder** — Montos sugeridos de efectivo (`quickAmountsForDueCents` en `driver_money.dart`) para agilizar el cobro: escalera fija (S/5, 10, 20, 50, 100, 200, 500) con overflow en múltiplos de S/100.

**E.164** — Formato de número telefónico usado en login (`phoneE164`), con prefijo `+51` (Perú) concatenado explícitamente en `login_screen.dart`.

**PEN** — Código de moneda por defecto (Soles peruanos) usado como fallback en todos los modelos cuando Backend no envía `currency`.
