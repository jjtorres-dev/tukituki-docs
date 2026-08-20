# flujo-de-trabajo

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

## Instalación

No hay script propio de setup (sin `Makefile`, sin `*.sh` de bootstrap). El flujo es el estándar de Flutter:

```
flutter pub get
```

Requiere Flutter SDK compatible con Dart `^3.12.2` (`pubspec.yaml:8`). Verificado en este entorno: Flutter 3.44.6 / Dart 3.12.2 stable.

Para Android, además hace falta crear `android/secrets.properties` con una clave real de `MAPS_API_KEY` (ver `android/local.defaults.properties` para el formato); sin ese archivo, el build usa el placeholder `DEFAULT_API_KEY` versionado y el mapa no funcionará con datos reales.

## Ejecución local

```
flutter run --dart-define=API_BASE_URL=<url-del-backend>
```

`API_BASE_URL` es obligatorio: `AppConfig.normalizedApiBaseUrl` lanza `StateError` en tiempo de ejecución si no se definió (`lib/core/config/app_config.dart`). No hay valor por defecto ni archivo `.env` que lo sustituya.

`[PENDIENTE: definir proceso del equipo]` — qué URL(s) de backend usar por ambiente (dev/staging/prod) no está documentado en el repo.

## Tests

```
flutter test
```

Verificado sobre el commit analizado: **804 tests, todos pasando** (`flutter test`, sin failures). No hay comando separado para tests unitarios vs. widget tests; ambos corren juntos con `flutter_test`.

No existe `integration_test/` en el repo: no hay tests end-to-end/de integración configurados.

## Lint / análisis estático

```
flutter analyze
```

Verificado sobre el commit analizado: **"No issues found!"**. Configuración en `analysis_options.yaml`, que solo incluye `package:flutter_lints/flutter.yaml` sin reglas propias añadidas ni exclusiones.

Nota operativa: la primera vez que se corre `flutter analyze` (o `flutter test`/`flutter run`) tras un clone, Flutter puede resolver/descargar dependencias (`Resolving dependencies... Downloading packages...`) de forma transparente si `pubspec.lock` no está satisfecho localmente; esto no modificó `pubspec.lock` en la verificación de este documento (`git status` limpio tras correrlo).

## Build

Estándar de Flutter, sin wrapper propio:

```
flutter build apk --dart-define=API_BASE_URL=<url>
```

(o `appbundle` para Play Store). El build de `release` firma con las llaves de **debug** por defecto — ver `errores-conocidos.md` y `decisiones.md` ("Signing de release Android pendiente").

## Migraciones / seed

No aplica: este repo no tiene base de datos ni ORM (ver `arquitectura.md`, sección Persistencia). No hay carpeta de migraciones ni seeds.

## Git

Patrón observado (no necesariamente una regla escrita del equipo):

- Commits recientes en Conventional Commits, en inglés (`feat:`, `fix:`, `chore:`) — ver `convenciones.md`.
- Historial lineal en `main` (sin merge commits — toda integración de una rama de feature se hace con `merge --ff-only`, nunca `--no-ff`/squash).

### Workflow de checkpoint demostrado (onboarding de Driver, `R3.3`–`R3.10.1`)

Patrón real seguido de punta a punta para llevar una feature grande (el onboarding completo) desde cero hasta `main`, sin tocar Backend/Railway/producción en ningún punto intermedio:

1. Rama de trabajo dedicada desde `main` (`test/driver-onboarding-r3`).
2. Implementación por checkpoints incrementales, cada uno con su propio ciclo `flutter analyze` + `flutter test` en verde antes de continuar.
3. Build de APK `release` (firmado con llaves de **debug**, ver `errores-conocidos.md`) contra el backend de **staging** (`--dart-define=API_BASE_URL=https://tukituki-backend-staging.up.railway.app/api/v1`), nunca contra producción.
4. Prueba física del APK — solo tras la aprobación física explícita se cierra el checkpoint.
5. Un único commit por checkpoint cerrado (nunca varios commits sueltos sin aprobación), publicado a la rama de test remota.
6. Al completar todos los checkpoints: **auditoría final integral de solo lectura** sobre toda la rama acumulada (routing, seguridad de subida de archivos, PII/logging, dependencias, dead code, cobertura de tests) antes de considerar la integración.
7. Limpieza de los hallazgos MINOR de esa auditoría (si los hay), en un checkpoint separado, también commiteado y publicado.
8. Integración a `main` mediante **`git merge --ff-only`** exclusivamente — nunca merge normal/squash/rebase; se aborta si `main` dejó de ser ancestro directo de la rama de test.
9. Revalidación completa (`flutter analyze` + `flutter test`) sobre `main` ya avanzado, **antes** de `git push origin main`.
10. Build de un nuevo APK `release`, esta vez **desde `main`** (no desde la rama de test), para el smoke test físico final.
11. Smoke test físico corto sobre `main` (no se repite toda la batería de pruebas de cada checkpoint individual, ya que `main` es el mismo commit ya aprobado pieza por pieza).
12. Solo tras el smoke test físico en verde: se actualiza la documentación externa que seguía describiendo el `main` anterior, y se elimina la rama de test (local y remota) — nunca antes.

`[PENDIENTE: definir proceso del equipo]` — no hay `CONTRIBUTING.md` que documente este flujo como regla escrita del equipo; lo anterior es el patrón real seguido, no una política declarada.

## Deploy

No hay pipeline de CI/CD ni script de deploy en el repo (sin `.github/workflows`, sin `Dockerfile`, sin `fastlane/`). El único artefacto de "deploy" configurado es el build de Android descrito arriba, con signing de debug.

`[PENDIENTE: definir proceso del equipo]` — cómo se distribuye la app a conductores (Play Store interno, APK directo, TestFlight equivalente) no está documentado en el repo.

---

## Checklist técnico antes de considerar un cambio terminado

Solo pasos respaldados por scripts/configuración verificable en este repo:

1. `flutter analyze` sin issues nuevos.
2. `flutter test` pasa completo (804 tests en el baseline de este documento — cualquier reducción en el conteo o failure nuevo debe investigarse).
3. Si el cambio toca un repositorio (`data/*_repository.dart`) o un modelo (`domain/*.dart`), agregar/actualizar el test correspondiente en la ubicación espejo bajo `test/features/...` (patrón observado en el 100% de los módulos existentes).
4. Si el cambio toca una pantalla grande (`driver_home_screen.dart`, `driver_active_ride_screen.dart`), correr los tests "Responsive" existentes de esa pantalla para descartar overflow en los viewports ya cubiertos.
5. Si el cambio introduce un nuevo endpoint, seguir el patrón existente: método de repositorio que devuelve el modelo de dominio parseado (no `Map` crudo), con manejo de 404→`null` solo donde ya es el contrato esperado.

`[PENDIENTE: definir proceso del equipo]` para cualquier paso humano no verificable desde el repo: aprobación de PR, checklist de QA manual, criterios de "listo para producción" más allá de analyze/test.
