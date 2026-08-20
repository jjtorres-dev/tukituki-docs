# TukiTuki — Documentación de contexto

Este repositorio contiene la documentación de contexto transversal del
proyecto **TukiTuki** (app de mototaxis bajo demanda), separada de los
cuatro repositorios de código:

- `tukituki-backend`
- `tukituki-driver-app`
- `tukituki-passenger-app`
- `tukituki-admin-web`

Todo el contenido vive bajo `contexto/`.

## Qué hay en cada archivo/carpeta

- **`contexto/estado-proyecto.md`** — Contexto transversal de producto y
  estado del proyecto: qué está implementado, qué es solo una decisión
  de producto aprobada aún sin código, checkpoints cerrados, y
  pendientes reales. Es el documento que más cambia; ante cualquier
  duda sobre el estado actual, empezar por aquí.
- **`contexto/TukiTuki-Designer-Handoff-R1.md`** — Fotografía de solo
  lectura del producto (Backend + Driver + Passenger) orientada a
  UI/UX, para diseñar/rediseñar pantallas sin adivinar comportamiento.
- **`contexto/Backend/`, `contexto/App-driver/`, `contexto/App-passenger/`,
  `contexto/App-admin/`** — Contexto auxiliar técnico por repositorio,
  generado por lectura directa de código en un commit puntual de cada
  uno. Cada carpeta tiene los mismos 6 archivos:
  - `arquitectura.md` — estructura del repo, módulos, stack.
  - `convenciones.md` — patrones de código y estilo ya en uso.
  - `decisiones.md` — decisiones técnicas y de producto tomadas, con
    su motivo y evidencia (archivo/línea/commit); incluye el log de
    checkpoints cerrados de ese repositorio.
  - `errores-conocidos.md` — baseline de tests/análisis estático,
    bugs conocidos, limitaciones técnicas demostrables.
  - `flujo-de-trabajo.md` — cómo se desarrolla, prueba y despliega ese
    repositorio.
  - `glosario.md` — términos y siglas específicos de ese repositorio.

## Fuente de verdad

El código, los tests, la configuración y el historial Git de cada
repositorio son la verdad técnica actual. Estos documentos son
contexto auxiliar: si contradicen al código real, el código gana. Ver
la sección 1 de `contexto/estado-proyecto.md` para el detalle completo
de esta regla.

## Cómo se usa en el flujo de trabajo

Antes de retomar trabajo en cualquiera de los cuatro repositorios (uno
mismo o un agente), se lee primero `estado-proyecto.md` y, si aplica,
el `decisiones.md` del repositorio correspondiente, para no repetir
trabajo ya hecho ni contradecir una decisión de producto ya tomada.
Estos documentos se actualizan como parte del cierre de cada
checkpoint — no son notas informales, son el registro oficial de qué
se decidió, por qué, y con qué evidencia.
