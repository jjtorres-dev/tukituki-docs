# glosario

Repositorio:
tukituki-passenger-app

Branch analizada:
main

Commit analizado:
a5d2411e791c391d4dca72a79e21cf6e73395335

Última actualización:
2026-08-15

Fuente de verdad:
Este documento es contexto auxiliar. Si contradice al código actual,
el código y los tests tienen prioridad.

---

## Entidades de dominio

- **PublicUser** — usuario autenticado (`id`, `phoneE164`, `roles`,
  `status`, `isPhoneVerified`, `createdAt`). Devuelto por `auth/me`.
  (`lib/features/auth/domain/public_user.dart`)
- **PassengerRide** — el viaje del pasajero: origen/destino, tarifas,
  conductor asignado, timestamps del ciclo de vida, cancelación.
  Entidad central del módulo `ride`.
  (`lib/features/ride/domain/passenger_ride.dart`)
- **PassengerRideOffer** — propuesta de tarifa de un conductor durante
  la búsqueda (`SEARCHING_DRIVER`); puede ser igual a la oferta del
  pasajero o una contraoferta (`isCounterOffer`).
  (`lib/features/ride/domain/passenger_ride_offer.dart`)
- **AssignedDriver** / **AssignedDriverVehicle** — conductor y vehículo
  asignados a un ride una vez aceptada una oferta.
  (`lib/features/ride/domain/assigned_driver.dart`)
- **DriverLocation** — posición GPS del conductor (`latitude`,
  `longitude`, `heading`, `speed`, `accuracy`, `recordedAt`), validada
  por rango antes de aceptarse.
  (`lib/features/ride/domain/driver_location.dart`)
- **PassengerRideStartCode** — PIN/código que el pasajero muestra al
  conductor para iniciar el viaje cuando este llega
  (`code`, `expiresAt`, `remainingSeconds`, `remainingAttempts`).
  (`lib/features/ride/domain/passenger_ride_start_code.dart`)
- **RideReceipt** — recibo del viaje completado: direcciones, tiempos
  reales, distancia real, tarifa estimada, notas de cierre, pago.
  (`lib/features/ride/domain/ride_receipt.dart`)
- **RideReceiptPayment** — estado y montos del pago del recibo
  (`method`, `status`, `amountDue`, `grossAmount`, `discountAmount`,
  `cashReceived`, `changeGiven`, `confirmedAt`).
- **RideReceiptFare** — desglose de la tarifa final (`baseFare`,
  `distanceAmount`, `timeAmount`, `bookingFee`, `subtotal`,
  `adjustmentMultiplier`, `calculatedFinalFare`, `finalFare`,
  `fareCapAmount`, `fareWasCapped`).
- **FareEstimate** — cotización previa a crear un ride
  (`quoteId`, `quoteStatus`, `distanceMeters`, `durationSeconds`,
  `estimatedFare`, `expiresAt`, `routePolyline`).
  (`lib/features/fare/domain/fare_estimate.dart`)
- **PlacePrediction** / **PlaceDetails** — resultado de autocompletar
  direcciones y su detalle geocodificado (proxy del backend a Google
  Places). (`lib/features/places/domain/`)

## Estados de `PassengerRide.status`

Valores usados/consultados directamente en el código
(`ride_searching_screen.dart`, `ride_repository_test.dart`):

- **SEARCHING_DRIVER** — buscando conductor; se muestran/refrescan
  `PassengerRideOffer`.
- **DRIVER_ASSIGNED** — el pasajero aceptó/seleccionó una oferta.
- **DRIVER_ARRIVING** — conductor en camino (usado en tests de
  responsividad; sin lógica propia adicional detectada más allá del
  render de seguimiento).
- **DRIVER_ARRIVED** — conductor llegó al punto de origen; dispara la
  obtención automática del `PassengerRideStartCode` (PIN).
- **IN_PROGRESS** — viaje iniciado (tras validar el PIN en el
  backend); deja de mostrarse el código de inicio.
- **COMPLETED** — viaje terminado; el cliente detiene el polling y
  navega al recibo (`/ride/:id/receipt`).
- **CANCELLED** — viaje cancelado (por pasajero, conductor, admin o
  sistema — ver `cancelledBy`).
- **EXPIRED** — la búsqueda/cotización venció sin conductor asignado.
- **UNKNOWN** — valor local por defecto cuando el backend no envía
  `status` (fallback del parser, no un estado real del backend).

## `cancelledBy` (valor crudo de `RideCancellationActor`)

`PASSENGER` / `DRIVER` / `ADMIN` / `SYSTEM` — quién originó la
cancelación. El cliente **no distingue** una cancelación normal del
conductor de un no-show confirmado (ambos llegan como `DRIVER`; ver
`decisiones.md`).

## Estados de `RideReceiptPayment.status`

`PENDING` (default de fallback), `PAID`, `FAILED`, `EXPIRED`,
`DISPUTED`, `VOIDED`. Los últimos cuatro (junto a `PAID`) se tratan
como estados terminales que detienen el polling de pago.

## Otros enums / valores fijos observados

- **paymentMethod**: `CASH` — único valor enviado al crear un ride
  (`RideRepository.createRide`); el modelo de recibo admite el campo
  pero el cliente no ofrece otra opción hoy.
- **currency**: `PEN` — moneda por defecto usada como fallback en
  varios `fromJson` (soles peruanos).
- **Tags de calificación** (`ride_receipt_screen.dart`, constante
  `_tags`): `SAFE_DRIVING`, `FRIENDLY`, `CLEAN_VEHICLE`, `PUNCTUAL`,
  `GOOD_COMMUNICATION`, `RESPECTFUL`, `CLEAR_PICKUP_POINT`.
- **roles** de `PublicUser`: al menos `PASSENGER` es un valor conocido
  y verificado por la app (`splash_screen.dart`); no hay evidencia en
  este repo de qué otros roles existen.
- **status** de `PublicUser`: al menos `ACTIVE` es el único valor
  verificado explícitamente por la app para considerar la sesión
  válida.

## Siglas / conceptos internos

- **OTP** — One-Time Password, código de verificación telefónica.
  Endpoints (`auth/otp/request`, `auth/otp/verify`) y pantalla
  (`OtpScreen`) existen en el código pero no están conectados a ningún
  flujo activo (ver `decisiones.md`).
- **PIN / start code** — término usado indistintamente en comentarios
  y tests para referirse a `PassengerRideStartCode.code`.
- **E164** (`phoneE164`) — formato internacional de número telefónico
  (`+<código país><número>`), estándar ITU-T E.164.
- **G4B-R\<n\>**, **G1P** — identificadores de requisito/checkpoint
  usados en comentarios de código y nombres de tests (p. ej.
  `G4B-R5.1`, `G4B-R3-10`, "Checkpoint G1P"). No hay documento en este
  repo que defina su significado; parecen referenciar una
  especificación o backlog externo.
- **quoteId / fareQuoteId** — identificador de una `FareEstimate`; se
  reenvía al crear el ride (`createRide(fareQuoteId: ...)`).
