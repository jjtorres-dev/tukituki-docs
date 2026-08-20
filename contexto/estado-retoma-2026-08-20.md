# Informe de retoma — 20/08/2026

## 1. Resumen ejecutivo (máximo 10 líneas)

La tarea "movimiento fluido del conductor en el mapa" **ya tiene una implementación completa**, con nombre interno `R4.4B`, pero está **sin commitear** en dos repos: `tukituki-driver-app` y `tukituki-passenger-app`, ambos actualmente parados en una rama local `test/r4-smooth-driver-marker` (sin commits propios — todo el trabajo es working tree sucio). El código implementa animación lineal del marcador del conductor en Passenger (`ride_searching_screen.dart`) y sube la cadencia de reporte GPS del conductor de 10s a 3s en Driver (`driver_active_ride_screen.dart`), con tests nuevos ya escritos y en verde localmente (no verificado por este informe si `flutter test` corre limpio). Ningún otro repo tiene cambios sin commitear ni stashes. El repo `tukituki-backend` tiene una rama `test/otp-demo-staging` con 1 commit sin fusionar (`OTP-DEMO-R1`, ya documentado como pendiente explícito). Se encontró una **inconsistencia relevante**: `estado-proyecto.md` (actualizado 2026-08-18) afirma que el onboarding de Driver pasos 2-5 no existe o sigue sin fusionar en una rama que en la práctica **ya no existe** — el código real en `main` de `tukituki-driver-app` ya tiene esos pasos implementados y enrutados. No se encontraron comentarios `TODO`/`FIXME` reales en ningún repo (solo la palabra "pendiente" como texto normal en español). Los cuatro proyectos y sus 26 documentos de contexto existen; ninguno falta.

## 2. Estado de cada proyecto (backend, driver, passenger, admin)

### Backend (`tukituki-backend`)
- Rama actual: `main`. Ramas locales: `feat/google-routes-fare-estimates`, `main`, `test/otp-demo-staging`.
- Último commit en `main`: `29fe187a` (2026-08-18) "fix: align driver phone verification with MVP policy".
- Cambios sin commitear: **ninguno** (`git status --short` limpio).
- Stash: ninguno.
- Rama `test/otp-demo-staging` tiene 1 commit no fusionado a `main`: `fd9ba8b3` "feat: add guarded staging otp demo flow" — esto coincide con lo documentado (`OTP-DEMO-R1`, `estado-proyecto.md` línea 600): implementado y probado en la rama, pendiente de una secuencia explícita de pasos (CI, STAGING, validación física) antes de fusionar.

### Driver (`tukituki-driver-app`)
- Rama actual: `test/r4-smooth-driver-marker` (sin commits propios respecto a `main` — `git log main..test/r4-smooth-driver-marker` no devuelve nada).
- Último commit real (= HEAD de `main`): `9c2a7f7` (2026-08-18) "feat: show compact passenger identity during rides".
- Cambios sin commitear:
  - `lib/features/driver/presentation/driver_active_ride_screen.dart` (modificado)
  - `test/features/driver/presentation/driver_active_ride_screen_test.dart` (modificado, +183 líneas)
- Stash: ninguno.

### Passenger (`tukituki-passenger-app`)
- Rama actual: `test/r4-smooth-driver-marker` (sin commits propios respecto a `main`).
- Último commit real (= HEAD de `main`): `5c4f0f9` (2026-08-18) "feat: add driver identity and photo identification".
- Cambios sin commitear:
  - `lib/features/ride/presentation/ride_searching_screen.dart` (modificado, +202/-4 líneas)
  - `test/features/ride/presentation/ride_searching_screen_test.dart` (modificado, +422 líneas)
- Stash: ninguno.

### Admin Web (`tukituki-admin-web`)
- Rama actual: `main`. Ramas locales: solo `main` (`origin/carlos` existe en remoto, preservada deliberadamente según `estado-proyecto.md`, sin rama local).
- Último commit: `989ffc4` (2026-08-16) "feat: add secure driver document preview".
- Cambios sin commitear: ninguno.
- Stash: ninguno.
- No está tocado por la tarea del marcador fluido — el admin web no muestra mapas de conductor en vivo.

## 3. La tarea del movimiento fluido del conductor

**a) ¿Existe algo implementado?** Sí, una implementación completa y no trivial, identificada internamente como checkpoint `R4.4B` (referenciada en comentarios de código, sin documento externo que la describa todavía — ver sección 5).

- **Passenger** — `tukituki-passenger-app/lib/features/ride/presentation/ride_searching_screen.dart` (working tree, sin commitear):
  - Nueva clase `_RideRouteMapState` con `SingleTickerProviderStateMixin` y un `AnimationController` (`_driverMarkerAnimationController`, duración 2800ms, línea ~3211).
  - Método `_handleDriverLocationUpdate()` (línea ~3277) que decide cómo reaccionar a cada nueva posición recibida por polling (cada 3s): interpola linealmente entre la posición visual actual y la nueva (no desde el último "target" si una animación estaba en curso, para no saltar hacia atrás); resetea sin animar si cambia `rideId` o `driverProfileId`; conserva el último marker válido si la posición nueva es `null` transitorio; hace "snap" directo sin animar si el salto supera 300 metros (guard anti-teleport, vía distancia Haversine, función `_driverMarkerDistanceMeters` al final del archivo); ignora diferencias de coordenada insignificantes para evitar parpadeo.
  - El widget `_RideRouteMap` ahora recibe `rideId` y `driverProfileId` además de `driverLocation` (línea ~3199), necesarios para detectar cambios de identidad.
  - Tests nuevos: `test/features/ride/presentation/ride_searching_screen_test.dart`, grupo `'R4.4B: interpolación visual del marker del Driver'` (línea 2075), con 9 casos: primera posición directa, animación A→B suave con punto intermedio verificado, interrupción de animación por una nueva posición C sin salto hacia atrás, coordenada igual sin reiniciar animación, `driverLocation` null transitorio, cambio de conductor, cambio de ride, salto >300m con snap directo, y dispose sin dejar Timer/Ticker colgado.

- **Driver** — `tukituki-driver-app/lib/features/driver/presentation/driver_active_ride_screen.dart` (working tree, sin commitear):
  - `_startActivityTimer()` (línea ~235) cambia la cadencia de publicación de GPS/heartbeat durante el viaje activo de **10 segundos a 3 segundos**, para alimentar la animación del Passenger con actualizaciones más frecuentes.
  - Comentario explícito referencia "R4.4B" y aclara que la cadencia de `Online` sin viaje en `DriverHomeScreen` (10s) no cambia — solo la del viaje activo.
  - Tests nuevos: `test/features/driver/presentation/driver_active_ride_screen_test.dart`, grupo `'Checkpoint R4.4B: cadencia GPS de viaje activo (3s)'` (línea 3091), 5 casos: cadencia real de 3s, guard contra fetch GPS concurrente, recuperación tras error de GPS, cancelación del timer en `dispose`, y que un fallo de `complete` (409) no duplique el scheduler.

**b) ¿Está completa, a medias, o solo planificada en la documentación?**
- **En el código**: técnicamente completa para el caso principal (interpolación lineal + cadencia GPS más frecuente + hardening ante saltos/identidad/nulls), con cobertura de test extensa para los casos borde identificados. No se ejecutó `flutter test`/`flutter analyze` en esta auditoría (modo solo lectura) — no hay confirmación externa de que los tests pasen hoy.
- **En la documentación**: `estado-proyecto.md` (línea 580, última actualización registrada 2026-08-18) describe `R4.4` únicamente como **"NEXT"** — el próximo checkpoint, sin alcance definido más allá de exclusiones explícitas ("sin 'Por llegar', sin nuevo estado de ride, sin ETA simulado, sin polyline/ruta simulada. Alcance exacto pendiente de definir por JuanJo"). Ningún documento de `docs/contexto/App-driver/` o `docs/contexto/App-passenger/` menciona `R4.4` o `R4.4B`. Es decir: **la documentación todavía no sabe que este trabajo existe** — está más avanzado en el código que en los documentos.

**c) ¿Quedó código sin commitear o en una rama aparte?**
Sí, en ambos repos simultáneamente, en una rama local llamada `test/r4-smooth-driver-marker` que **no tiene ningún commit propio** — es una copia de `main` con cambios de working tree encima, no publicada a `origin` (no existe `origin/test/r4-smooth-driver-marker` en ningún repo, confirmado con `git fetch --dry-run`). Si se perdiera el working tree de estos dos repos sin haber hecho commit, este trabajo se perdería por completo.

**d) ¿Qué falta para cerrarla?**
1. Confirmar que `flutter analyze` y `flutter test` pasan en ambos repos sobre el working tree actual (no verificado en esta auditoría de solo lectura).
2. Prueba física en dispositivo real (patrón usado en checkpoints anteriores del proyecto: `PHYSICAL-REVIEW-PASS`/smoke físico).
3. Commit y push de ambos repos a una rama publicada (hoy no existe en `origin`).
4. Actualizar `estado-proyecto.md` y los `decisiones.md`/`errores-conocidos.md` de `App-driver`/`App-passenger` describiendo `R4.4`/`R4.4B` como lo hacen los checkpoints anteriores (`R4.3`, etc.) — hoy no hay ningún registro externo de este trabajo.
5. Fusión a `main` de ambos repos, siguiendo el mismo patrón de fast-forward + limpieza de rama que se usó en checkpoints previos (ver sección 16 de `estado-proyecto.md`).
6. El propio código deja una nota abierta: el umbral de salto grande (300m) está marcado como "ajustable tras prueba física" (`_driverMarkerLargeJumpMetersThreshold`, comentario en `ride_searching_screen.dart` línea ~3218) — sujeto a validarse o ajustarse con datos reales.

## 4. Otras tareas abiertas detectadas

- **Backend — `OTP-DEMO-R1`** (`test/otp-demo-staging`, commit `fd9ba8b3`, sin fusionar): mecanismo de demo para entregar el código OTP real en respuestas de `staging` bajo lista blanca de teléfono. Documentado en `estado-proyecto.md` línea 600 con una secuencia de 8 pasos pendientes antes de fusionar a `main`. No hay evidencia de que se haya avanzado más allá del commit ya existente.
- **Driver — Onboarding pasos 2-5 ("Sobre ti", "Tu mototaxi", "Tus documentos", "Revisar y enviar")**: la documentación (`estado-proyecto.md`, `TukiTuki-Designer-Handoff-R1.md`) los describe como pendientes o sin fusionar, pero el código en `main` de `tukituki-driver-app` ya los tiene completos y enrutados (ver discrepancia en sección 5). Como tarea real pendiente, esto parece **ya cerrado**, no abierto — pero requiere confirmación humana porque contradice la documentación vigente.
- **Backend — `OTP-R3`** (proveedor SMS real): explícitamente `DEFERRED/PAUSED` por decisión de producto hasta terminar la demo funcional. No es una tarea activa hoy.
- **Passenger/Driver — "Has llegado a tu destino" con confirmación del Passenger**: no implementado en ningún repo, sin modelo en Backend. Bloqueada por decisión de producto (sección 23 de `TukiTuki-Designer-Handoff-R1.md`).
- **Ambas apps — SOS/emergencia visible, compartir viaje/share-link visible, disputa de pago en efectivo visible**: Backend ya tiene los endpoints/modelos; ninguna app los expone en UI. Sin decisión de alcance/prioridad de producto.
- **Producción**: dominio corporativo, proveedor SMS, firma de release Android (hoy ambas apps firman `release` con clave de debug), configuración de Storage/Railway para producción — todos explícitamente no iniciados.
- **Admin Web**: varios `[PENDIENTE]` documentales, no de código — decisión de plataforma/pipeline de deploy (`docs/contexto/App-admin/decisiones.md:373`), framework de testing (`decisiones.md:392`), ausencia de CI/CD (`flujo-de-trabajo.md:137`). Son huecos de documentación/proceso, no funcionalidad de producto sin construir.
- **Sin ningún comentario `TODO`/`FIXME`/`HACK`/`XXX` real** en ningún repo (`tukituki-backend`, `tukituki-driver-app`, `tukituki-passenger-app`, `tukituki-admin-web`) — confirmado por búsqueda directa en esta auditoría y coincide con lo ya documentado en `Backend/errores-conocidos.md:74` y `App-admin/errores-conocidos.md:75-81`.

## 5. Inconsistencias entre la documentación y el código real

1. **Onboarding de Driver pasos 2-5 (la más relevante encontrada)**: `estado-proyecto.md` sección 8 (líneas 128-129) afirma que `tukituki-driver-app` "no tiene ninguna pantalla de registro/onboarding de conductor" y que solo existe la foundation en Backend. La sección 17 (línea 592) agrega que el trabajo de onboarding (`DRIVER-ONBOARDING-R3.x`) está "commiteado y publicado en `origin/test/driver-onboarding-r3`, todavía sin fusionar a main". Verificado en esta auditoría:
   - `origin/test/driver-onboarding-r3` **no existe** en el remoto (`git fetch --dry-run` sobre `tukituki-driver-app` solo muestra `origin/main`).
   - El router real (`lib/core/router/app_router.dart`) en `main`@`9c2a7f7` ya define rutas para los pasos "Sobre ti" (`driver_onboarding_about_you_screen.dart`), "Tu mototaxi" (`driver_onboarding_vehicle_screen.dart`), "Tus documentos" (`driver_onboarding_documents_screen.dart`) y "Revisar y enviar" (`driver_onboarding_submit_review_screen.dart`), todos presentes como archivos reales en `lib/features/driver/presentation/onboarding/`.
   - El propio historial de commits de `main` confirma esto: `3fec449`, `543e4a8`, `081c5fe`, `48c8156`, `0201b42`, `d0abe57` (2026-08-17 y 2026-08-18) — "add driver onboarding account flow", "add driver personal details onboarding", "add driver vehicle onboarding", "add driver documents onboarding", "add driver onboarding review and submission", "add driver rejected application corrections".
   - Conclusión: el código está más adelantado que lo que describe `estado-proyecto.md` en esta área específica — el documento no se actualizó tras la fusión real de este trabajo, aunque su fecha de "última actualización" (2026-08-18) sugiere que sí debería reflejarlo.
2. **Checkpoint `R4.4`/`R4.4B` no documentado**: el código ya usa el nombre `R4.4B` en comentarios y nombres de grupos de test en dos repos, y un comentario en `ride_searching_screen.dart` (línea ~3278) remite a "AGENTS.md R4.4B" — pero **no existe ningún `AGENTS.md`** en `tukituki-driver-app` ni en `tukituki-passenger-app` (confirmado, ambos "No such file or directory"). El comentario referencia un documento que no está en el repositorio.
3. Nada más relevante detectado; el resto de la documentación de `estado-proyecto.md` y `TukiTuki-Designer-Handoff-R1.md` (matching, negociación de tarifa, pricing, etc.) es internamente consistente y ya está marcada explícitamente por sus propios autores como verificada línea por línea contra el código en el momento de escritura.

## 6. Riesgos o cosas que conviene revisar antes de seguir

- **Riesgo de pérdida de trabajo**: el código de `R4.4B` en Driver y Passenger vive únicamente en el working tree local de una rama sin commits ni publicación remota. Cualquier `git checkout`, `git reset --hard`, limpieza de working tree, o simplemente trabajar en otra máquina, lo perdería sin posibilidad de recuperación. Se recomienda commitear (aunque sea en la rama de trabajo) antes de continuar.
- **No se corrió la suite de tests** de ninguno de los dos repos durante esta auditoría (modo solo lectura, sin builds). El estado "tests en verde" de `R4.4B` es una inferencia a partir del código de los tests nuevos, no una confirmación de ejecución real.
- **Umbral de salto GPS (300m) sin validar físicamente**: el propio código lo marca como sujeto a ajuste tras prueba real; convendría probarlo en condiciones reales (zonas con mala señal GPS, esquinas cerradas) antes de dar el checkpoint por cerrado.
- **Actualizar `estado-proyecto.md` antes de seguir generando más checkpoints**: dado que ya está desalineado en la sección de onboarding de Driver, seguir escribiendo checkpoints nuevos sobre una base desactualizada aumenta el riesgo de que la próxima persona (o el propio JuanJo en otra sesión) tome decisiones basadas en información incorrecta sobre qué existe y qué no.
- **`test/otp-demo-staging` (Backend) sigue con 1 commit sin fusionar** desde 2026-08-16 — revisar si sigue siendo relevante para la demo o si ya se decidió otra vía, antes de retomar cualquier trabajo de OTP.
- El nombre de rama `test/r4-smooth-driver-marker` en dos repos distintos sin ningún commit propio (solo working tree) es un patrón distinto a como se manejaron los checkpoints anteriores del proyecto (que sí generaban commits reales en la rama `test/*` antes de fusionar) — vale la pena confirmar si esto fue intencional o si el commit se quedó pendiente a media sesión.
