# flujo-de-trabajo

Repositorio:
tukituki-passenger-app

Branch analizada:
main

Commit analizado:
a5d2411e791c391d4dca72a79e21cf6e73395335

Última actualización:
2026-09-03 (agregada la nota operativa sobre el bug intermitente de atribución de IA de Claude Code y el procedimiento de verificación obligatoria de commits)

Fuente de verdad:
Este documento es contexto auxiliar. Si contradice al código actual,
el código y los tests tienen prioridad.

---

## Instalación

Proyecto Flutter estándar, sin scripts propios de bootstrap:

```
flutter pub get
```

Requiere Flutter `stable` (verificado en este entorno: 3.44.6, Dart
3.12.2 — coincide con el constraint `sdk: ^3.12.2` de `pubspec.yaml`).
No hay `.tool-versions`/`fvm` en el repo que fije una versión exacta de
Flutter.

Para Android, además hace falta crear `android/local.properties` con
`MAPS_API_KEY=<clave>` (no versionado, ver `arquitectura.md`); sin esa
clave el mapa de Google no renderiza pero la app compila igual.

## Ejecución local

```
flutter run --dart-define=API_BASE_URL=<url-del-backend>
```

`API_BASE_URL` es **obligatorio**: `main.dart` llama a
`AppConfig.normalizedApiBaseUrl` antes de `runApp`, y esa llamada
lanza `StateError` si la variable no fue definida — la app no arranca
sin backend configurado. No hay un valor por defecto ni un modo
offline/mock.

### Contra el backend de staging

```
flutter run --dart-define=API_BASE_URL=https://tukituki-backend-staging.up.railway.app/api/v1
```

`API_BASE_URL` debe incluir el prefijo `/api/v1` (`API_PREFIX` en
`tukituki-backend/.env`) — el repo concatena el path relativo del
endpoint (p. ej. `auth/me`) directamente sobre esta base
(`lib/core/network/api_client.dart`), sin agregarlo por su cuenta.

Solo hay target Android disponible en este repo (no `ios/`).

## Tests

```
flutter test
```

En este commit: **155 tests, todos en verde**, sin skips detectados
(ver `errores-conocidos.md` para el detalle de la corrida). No hay
`integration_test/`, solo unit tests (`test/features/**/domain`,
`test/features/**/data`) y widget tests
(`test/features/**/presentation`, `test/widget_test.dart`).

No hay script de cobertura configurado en el repo (se podría correr
`flutter test --coverage` de forma estándar, pero no hay `Makefile`/CI
que lo haga).

## Lint / análisis estático

```
flutter analyze
```

Usa `analysis_options.yaml` → `package:flutter_lints/flutter.yaml`
sin reglas adicionales habilitadas/deshabilitadas. En este commit:
**sin issues**.

## Build

```
flutter build apk --dart-define=API_BASE_URL=<url>
```

(o `appbundle`). No hay flavors (`--flavor`) configurados en
`android/app/build.gradle.kts` — un solo `applicationId`
(`pe.tukituki.passenger`) para todos los builds. El build `release`
reutiliza la firma de `debug` (ver `decisiones.md`), por lo que un APK
"release" generado desde este repo tal cual **no está listo para
publicación en Play Store**.

## Migraciones / seed

[PENDIENTE: no aplica — no hay base de datos ni migraciones en este
repositorio; la persistencia de dominio vive en un backend externo no
incluido aquí]

## Git

Patrones observables (ver `convenciones.md` para el detalle):
mensajes `feat:`/`fix:`/`chore:` en inglés, historia lineal en `main`,
sin ramas remotas adicionales ni tags. No hay `CONTRIBUTING.md` ni
plantillas de PR en el repo.

## Nota operativa — atribución de IA en los commits (Claude Code)

**Bug confirmado en vivo (2026-09-03, durante `PROFILE-EDIT-R1`).** Aunque `~/.claude/settings.json` tenga `attribution.commits = false`, **algunas ejecuciones de Claude Code igual agregan trailers de atribución** (`Co-Authored-By: Claude …` y/o `Claude-Session: …`) al mensaje de commit. Ese día ocurrió **2 veces**, en sesiones nuevas (no reutilizadas) y en 2 repositorios distintos (`tukituki-backend` y `tukituki-passenger-app`). Es un bug del lado de la herramienta, sin fix conocido al momento de escribir esto — la configuración no es garantía.

**Procedimiento obligatorio, sin excepciones:**

1. Después de **cada** commit (incluido cada `--amend`), correr:
   ```
   git log -1 --format="%an <%ae>%n%B"
   ```
2. Si aparece cualquier trailer `Co-Authored-By:` de una IA o `Claude-Session:` (o similar), corregirlo **antes de hacer push**:
   ```
   git commit --amend -m "<mensaje limpio, sin trailers>"
   ```
   y volver a verificar con el `git log -1` de arriba.
3. El autor debe quedar siempre `Juanjo <jjtorres.devtech@gmail.com>` y el cuerpo del mensaje sin ninguna línea de atribución de herramienta.
4. Recién con la verificación limpia se hace `git push` / el fast-forward a `main`.

Aplica a los tres repos del proyecto. Esta verificación va **antes** de cualquier publicación a un remoto, sea rama `test/*` o `main`.

## Deploy

No hay pipeline de deploy en el repo (sin `.github/workflows`, sin
Dockerfile, sin Fastlane/Codemagic). El build/firma/publicación a
tiendas es un proceso manual externo al repositorio, o
[PENDIENTE: definir proceso del equipo].

---

## Checklist técnico antes de considerar un cambio terminado

Solo pasos respaldados por scripts/configuración presentes en el
repo:

- [ ] `flutter analyze` sin issues nuevos.
- [ ] `flutter test` en verde (155 tests en este commit — cualquier
      regresión debe quedar explicada, no silenciada).
- [ ] Si se tocó un modelo de dominio en `lib/features/*/domain/` con
      `fromJson`, revisar que exista/actualice su test correspondiente
      en `test/features/*/domain/` (patrón observado en cada commit
      `feat:` reciente).
- [ ] Si se tocó una pantalla con estados de `PassengerRide.status` o
      `RideReceiptPayment.status`, revisar los tests de responsividad
      existentes (`responsive ... en 360x640/390x844/412x915`) en
      `ride_searching_screen_test.dart` / `ride_receipt_screen_test.dart`.
- [ ] `flutter build apk --dart-define=API_BASE_URL=<url>` compila sin
      errores (valida que no se rompió el bootstrap de configuración).

## [PENDIENTE: definir proceso del equipo]

- Proceso de revisión de código (¿PR obligatorio?, ¿cuántos
  aprobadores?) — no inferible desde el repo (rama única, sin
  historial de PRs disponible localmente).
- Convención de versionado de release (`pubspec.yaml` tiene
  `version: 1.0.0+1` fijo en este commit) y cadencia de publicación.
- Qué significan los identificadores `G4B-R<n>` / checkpoints usados en
  comentarios y tests, y dónde vive esa especificación.
- Estrategia de entornos (staging vs producción) más allá de pasar un
  `API_BASE_URL` distinto al build.
