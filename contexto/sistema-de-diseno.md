# Sistema de diseño — TukiTuki

Definido el 20/08/2026 sobre las pantallas de login, registro y completar perfil de la app Passenger.

Extendido el 26/08/2026 (`DESIGN-SYSTEM-R2`) a partir de la auditoría de `home_screen.dart` — primera pantalla del sistema con mapa a pantalla completa, chips tocables y texto sobre superficie oscura fuera del header. Los tokens de esta extensión están marcados `[R2]` donde corresponde. `DESIGN-SYSTEM-R2` solo agrega tokens — no migra ninguna pantalla.

Este documento es la fuente de verdad para todo lo visual. Cualquier color, tamaño o medida que se use en las apps tiene que salir de aquí. Si algo que necesitas no está definido, se decide y se agrega a este documento antes de escribirlo en el código.

---

## 1. Principio general

Las dos apps son la misma marca. Comparten fondo, tipografía, formas y medidas.

Lo único que las distingue es el **acento dominante**: la app del pasajero tira a verde, la del conductor tira a ámbar. Esto no es decoración — sirve para que el pasajero, al subirse de noche, reconozca de un vistazo que la pantalla que le muestra el conductor es realmente la app del conductor.

Todo lo demás debe verse igual en ambas.

---

## 2. Colores

Cada color tiene un nombre y un uso. No se usan colores fuera de esta lista.

### Base — compartidos por las dos apps

| Nombre | Hex | Uso |
|---|---|---|
| `verdeMarca` | `#123B26` | Header, bordes de campo activo, etiquetas de campo |
| `verdeProfundo` | `#0B2517` | Borde exterior del degradado del header |
| `verdeClaro` | `#1D5433` | Centro del degradado del header |
| `crema` | `#FFF9EC` | Fondo de toda la pantalla |
| `blanco` | `#FFFFFF` | Fondo de campos de texto |
| `textoPrimario` | `#16241C` | Títulos y texto que se escribe |
| `textoSecundario` | `#5A6B5C` | Subtítulos y texto de apoyo |
| `textoTenue` | `#7C8A79` | Pistas, notas al pie, íconos secundarios |
| `placeholder` | `#8A9487` | Texto de ejemplo dentro de un campo vacío |
| `bordeCampo` | `#C3CDBE` | Borde de campo en reposo |
| `bordeSuave` | `#E7E0CB` | Separadores y líneas divisorias |
| `inactivo` | `#E2DCC9` | Barras y elementos apagados |

**Un solo gris.** `textoSecundario` `#5A6B5C` reemplaza a los tres grises que existían antes (`#7C8A79`, `#6F7E72`, `#9A8F6E`). El único gris adicional permitido es `textoTenue`, y solo para texto que debe pesar menos que el resto.

### Acento — distinto por app

| Nombre | Passenger | Driver | Uso |
|---|---|---|---|
| `acento` | `#1F7A3E` (verde) | `#E8951A` (ámbar) | Enlaces, elementos activos, confirmaciones |

### Acción — compartido

| Nombre | Hex | Uso |
|---|---|---|
| `amarilloCTA` | `#FFC72C` | Botón principal. **Uno solo por pantalla.** |

El amarillo es exclusivo del botón principal. No se usa para fondos, íconos ni decoración. Su valor está en que el usuario aprenda que amarillo significa "aquí se toca".

**Excepción documentada — FAB de recentrar en Home (`DESIGN-SYSTEM-R2`, 2026-08-26).** Home tiene dos elementos en `amarilloCTA`: el botón principal de la hoja ("Ofrecer y buscar conductor" y estados relacionados) y el botón flotante de "centrar en mi ubicación" sobre el mapa. Es una excepción deliberada a la regla de arriba, no un olvido: el FAB de recentrar no compite con el CTA por la misma decisión — nunca son "la siguiente acción" al mismo tiempo, uno vive sobre el mapa como utilidad de navegación y el otro en la hoja como el cierre del flujo — y darle un color distinto al FAB no tenía, a priori, un beneficio claro de claridad que compensara el trabajo de definir uno nuevo. Se deja así **por ahora**; no está resuelto de forma permanente. Si se retoma, revisar si el FAB debería moverse a un color neutro (p. ej. blanco con ícono `verdeMarca`, como ya usa el botón de menú) en vez de a otro tono cálido nuevo.

### Estados

| Nombre | Hex | Uso |
|---|---|---|
| `exito` | `#1F7A3E` | Confirmaciones, pago completado |
| `alerta` | `#E8951A` | Advertencias, pago pendiente |
| `error` | `#E05B4F` | Errores de validación, cancelaciones |
| `destino` | `#D8542C` | Marcador de destino en el mapa |
| `aviso` [R2] | `#B8641E` | Ícono y texto de estados vencidos o erróneos que no son error de formulario (p. ej. "sin ubicación", "cotización vencida") |
| `fondoAviso` [R2] | `#FFF0E8` | Fondo de la caja de aviso que acompaña a `aviso` |

**`aviso` es un color nuevo, no una reutilización de `alerta` ni de `destino` (`DESIGN-SYSTEM-R2`, 2026-08-26).** Antes de esta extensión, `home_screen.dart` usaba `destino` (`#D8542C`, pensado como "marcador de destino en el mapa") también para el aviso de "sin ubicación" — mezclaba dos significados distintos en un mismo color. La alternativa obvia, reutilizar `alerta` (`#E8951A`), se descartó porque ese valor es **idéntico al acento de la app del Conductor** (ver sección 2, "Acento"): usarlo de forma prominente en Home — la pantalla de mapa a pantalla completa que el pasajero ve todo el viaje — arriesgaba la señal de reconocimiento de marca de la sección 1 ("el pasajero, al subirse de noche, reconoce de un vistazo que la pantalla que le muestra el conductor es realmente la app del conductor"). `aviso` (`#B8641E`) comparte la familia cálida de `destino`/`alerta`/`amarilloCTA` (coherencia de paleta) pero es visiblemente más oscuro/ocre que los tres — distinguible a simple vista, no solo en el valor hex.

**PROVISIONAL (aprobado como tal por JuanJo, 2026-08-26).** No se pudo validar en pantalla en `DESIGN-SYSTEM-R2` porque ese checkpoint solo agrega tokens — no se usan en ninguna vista todavía. Se valida cuando se aplique en la migración de `home_screen.dart`, prestando atención especial al caso de "cotización vencida": ese texto va sobre `verdeMarca` (verde oscuro), no sobre `crema` — un ocre oscuro como `aviso` sobre un fondo oscuro puede quedar con poco contraste, a diferencia del uso sobre `fondoAviso`/`crema` (claro), donde el contraste es más fácil de lograr. Si falla ese caso, el valor del token se corrige en un solo lugar (`passenger_colors.dart`).

### Texto sobre fondo oscuro [R2]

El sistema se definió sobre pantallas claras (login, registro, completar perfil) y no cubría texto sobre una superficie oscura fuera del header degradado. `DESIGN-SYSTEM-R2` (2026-08-26) agrega dos tokens, tomados literalmente de valores que ya estaban en uso como literales en `home_screen.dart`:

| Nombre | Hex | Uso |
|---|---|---|
| `textoTenueSobreOscuro` | `#8FA891` | Texto micro sobre `verdeMarca` (p. ej. la etiqueta "TE RECOGEMOS EN" del marcador de origen) |
| `textoSecundarioSobreOscuro` | `#B9C8BC` | Texto de apoyo sobre `verdeMarca` (p. ej. ayuda y estado de vigencia dentro de la tarjeta de oferta) |

Nombrados por rol — paralelos a `textoTenue` y `textoSecundario` de la sección 2, con calificador de superficie — no por su valor de color, para que el nombre siga siendo válido si el tono exacto cambia más adelante.

### Superposición sobre el mapa [HOME-DESIGN-R1]

Categoría nueva, agregada el 2026-08-26 al planificar la migración de `home_screen.dart`: colores que se dibujan directamente **sobre el mapa** (rutas, overlays), no sobre una superficie de interfaz (`crema`/`blanco`/`verdeMarca`). Un color de mapa se valida contra el mapa real (Google Maps, distintos niveles de zoom, distinto terreno), no contra el resto de la paleta — puede no tener ninguna relación visual con los demás tokens, y no tiene por qué tenerla.

| Nombre | Hex | Uso |
|---|---|---|
| `lineaRuta` | `#5C8A17` | Línea de la ruta trazada sobre el mapa entre origen y destino |

**No reutiliza `acento` (`#1F7A3E`).** El valor de `lineaRuta` ya estaba en uso como literal (`_secondaryGreen` en `home_screen.dart`) y funciona: es legible sobre el mapa real. Migrarlo a `acento` lo oscurecería sin haber validado ese cambio en calle — se conserva el valor existente, con nombre propio, en vez de forzarlo a un token de interfaz que nunca fue pensado para dibujarse sobre un mapa. Si en el futuro aparecen más colores de este tipo (otro trazo, un overlay), esta es la sección donde viven — no la de "Estados" ni la de "Base".

---

## 3. Tipografía

**Familia única: Manrope.** Gratuita, disponible en Google Fonts.

Se eligió por tres razones: las minúsculas son grandes en proporción, así que se lee con la pantalla en movimiento; los números son claros y no se confunden entre sí, importante al mostrar tarifas; y su peso 800 da firmeza a los títulos sin verse agresivo.

| Uso | Tamaño | Peso |
|---|---|---|
| Título de pantalla | 26 | 800 |
| Título de sección | 20 | 700 |
| Botón principal | 17 | 700 |
| Cuerpo y contenido de campos | 16 | 400 |
| Subtítulo y texto de enlace | 15 | 400 / 700 |
| Etiqueta de campo | 13 | 600 |
| Pista y nota al pie | 12–13 | 400 / 500 |

**Nada por debajo de 12.** En la app del conductor, el cuerpo sube a 17 porque se lee en movimiento.

Los títulos de pantalla llevan `letter-spacing: -0.6`. Los botones, `-0.2`. El interlineado del texto corrido es 1.5.

### Escalas del campo de precio [R2] — provisionales

Agregadas en `DESIGN-SYSTEM-R2` (2026-08-26) a partir de `home_screen.dart` ("¿Cuánto quieres ofrecer?"), el único lugar de la app con una cifra editable en tamaño grande:

| Uso | Tamaño | Peso |
|---|---|---|
| Valor del monto (`montoOferta`) | 27 | 800 |
| Prefijo "S/ " (`prefijoMoneda`) | 19 | 700 |

**Provisionales.** El campo de precio se rediseña por completo en el checkpoint de tarifa sugerida (ver `App-passenger/errores-conocidos.md`, "Campo de precio acepta texto libre") — estas dos escalas pueden cambiar o desaparecer cuando eso ocurra. No forman parte de la escala general de la tabla de arriba porque son específicas de ese único campo, no reutilizables en otro contexto.

---

## 4. Medidas

| Elemento | Valor |
|---|---|
| Alto de campo de texto | 56 |
| Alto de botón principal | 58 |
| Radio de campo y botón | 14 |
| Radio del logo en el header | 20–28 según tamaño |
| Radio de la hoja crema | 26 (solo esquinas superiores) |
| Margen lateral de pantalla | 20 |
| Separación entre campos | 18–20 |
| Separación antes del botón principal | 24 |
| Alto de barra de progreso | 5 |
| Lado del checkbox | 24, radio 7 |
| Radio de píldora/chip [R2] | 30 |
| Radio de etiqueta de marcador [R2] | 11 |

**Ningún elemento tocable mide menos de 44 de lado.** Los campos de 56 y los botones de 58 cumplen con margen.

Los dos radios `[R2]` se agregaron en `DESIGN-SYSTEM-R2` (2026-08-26): el sistema solo contemplaba 14 (campos y botones) y 26 (hoja) porque se definió sobre pantallas sin mapa ni chips tocables. `radioPildora` (30) es la forma completamente redondeada de los chips sobre el mapa (métricas, destino sugerido). `radioEtiquetaMarcador` (11) es propio de la etiqueta que acompaña al marcador de origen — más chica y menos redondeada que una píldora, no reutilizable en otro elemento hoy.

---

## 5. Componentes

### Header

Degradado radial con el punto claro detrás del logo:

```dart
BoxDecoration(
  gradient: RadialGradient(
    center: Alignment(0, -0.36),
    radius: 1.0,
    colors: [Color(0xFF1D5433), Color(0xFF123B26), Color(0xFF0B2517)],
    stops: [0.0, 0.55, 1.0],
  ),
)
```

El valor de `center` se ajusta según la altura del header, para que el punto claro siempre caiga sobre el logo. En headers bajos se acerca a `Alignment(0, -0.08)`.

**Por qué radial y no diagonal:** el degradado diagonal que existía antes llevaba el punto claro a la esquina inferior derecha, donde no hay nada que mirar, y el salto de color era tan amplio que producía bandas visibles en pantallas de gama baja. El radial se mueve menos y dirige el ojo al logo.

La hoja crema monta 18–20 px sobre el header, con las esquinas superiores redondeadas. Eso da profundidad sin usar sombras.

### Campo de texto

Fondo blanco siempre. Nunca crema sobre crema — ese era el defecto principal del diseño anterior: con sol sobre la pantalla los campos desaparecían.

| Estado | Borde |
|---|---|
| Reposo | 1 px `bordeCampo` |
| Activo | 1.5 px `verdeMarca` |
| Error | 1.5 px `error` |

La etiqueta va **encima** del campo, en 13/600 `verdeMarca`. Así el interior del campo queda libre para el valor que el usuario escribe, y el usuario puede ver qué escribió sin recordar de qué campo se trataba.

El placeholder muestra un ejemplo real del formato esperado (`987 654 321`), no una repetición de la etiqueta.

### Barra de búsqueda [R2]

Variante propia, distinta de "Campo de texto" — agregada en `DESIGN-SYSTEM-R2` (2026-08-26) a partir de la barra de búsqueda de destino de Home. **Decidido: no se fuerza al patrón de "Campo de texto"/`TukiTextField`.** Ese patrón (etiqueta encima, alto fijo 56, campo aislado) es el de un formulario — Login y Registro. La barra de búsqueda es otro patrón de uso: una única entrada flotante con lupa, sin etiqueta, cuyo contenido (resultados de autocompletado) crece debajo de ella.

Diferencias deliberadas frente a "Campo de texto":

| | Campo de texto | Barra de búsqueda |
|---|---|---|
| Etiqueta | Encima, siempre visible | Ninguna — el ícono de lupa cumple ese rol |
| Alto | Fijo, 56 | Variable, según el padding vertical del contenido (no un número fijo del sistema) |
| Ícono | Solo en `prefix`/`sufix` opcionales | Lupa fija a la izquierda; a la derecha, spinner de carga o botón de limpiar, mutuamente excluyentes |
| Radio | 14 (`radioCampoBoton`) | Mismo `radioCampoBoton` (14) — no se define un radio nuevo para esta variante; el valor que usaba `home_screen.dart` (15) era un literal sin intención de diseño, no una decisión distinta |
| Fondo | Blanco siempre | Crema secundario, sin token propio todavía (mismo criterio de "nunca crema sobre crema" — la barra debe distinguirse del fondo de la hoja) |

Borde, igual criterio que "Campo de texto":

| Estado | Borde |
|---|---|
| Reposo | 1 px `bordeCampo` |
| Activo | 1.5 px `verdeMarca` |

**Corrección respecto al código actual**: `home_screen.dart` usa hoy el borde activo en `acento` (verde de acción), no en `verdeMarca` como dicta esta sección para "Campo de texto" — la barra de búsqueda debe usar `verdeMarca`, igual que cualquier otro campo, cuando se migre.

**Pendiente de decidir en el checkpoint de implementación** (no en este, que solo documenta): el fondo exacto de la barra en reposo — hoy `home_screen.dart` usa un crema secundario (`#FBF7EA`) sin token propio; la migración debe forzarlo a `crema` o a un color ya existente, no inventar uno nuevo solo para esto salvo que se vea mal en pantalla.

**Recomendación de implementación — parámetro de `TukiTextField` vs. componente separado**: recomiendo **componente separado** (p. ej. `TukiSearchBar`), no un parámetro nuevo en `TukiTextField`. Razón: `TukiTextField` construye su `Container` sobre un contrato fijo — alto 56 constante, layout de una sola fila con slots `prefix`/`suffix` genéricos, y un bloque de `errorText` condicional debajo. La barra de búsqueda no solo cambia valores (alto, radio) sino la estructura interna: necesita alto flexible, un ícono de posición fija (no un slot genérico), y dos estados mutuamente excluyentes a la derecha (carga/limpiar) en vez de un `suffix` libre — meterlo como flag booleano (`isSearchBar: true`) obligaría a `TukiTextField` a ramificar buena parte de su `build()` según ese flag, lo que en la práctica sería mantener dos componentes dentro de un mismo archivo con un `if` en el medio. Un widget separado que reutiliza los mismos tokens de color (`bordeCampo`, `verdeMarca`, `radioCampoBoton`) y el mismo criterio de foco/error es más simple de leer y de testear por separado. **Costo de la alternativa descartada** (parámetro en `TukiTextField`): evita un archivo nuevo, pero acopla dos formas visuales distintas a un mismo widget y hace más frágil cualquier cambio futuro a "Campo de texto" (hay que verificar que no rompa la rama de búsqueda). Si en la práctica la barra de búsqueda termina necesitando más comportamiento propio (debounce, lista de predicciones acoplada), un componente separado también da más margen para crecer sin forzar ese crecimiento dentro de `TukiTextField`.

### Botón principal

Fondo `amarilloCTA`, texto `textoPrimario` en 17/700, alto 58, radio 14, sin sombra.

**Estado deshabilitado:** sin relleno, borde 1.5 px `#D5CFBA`, texto `#A79F8A`. Debajo, siempre, una línea de 12/500 explicando por qué está deshabilitado.

Un botón apagado que no dice por qué es un callejón sin salida en móvil, donde el usuario no puede "pasar el mouse por encima" para averiguarlo.

### Enlace

Color `acento`, peso 700. Dentro de una frase, el texto que lo rodea queda en `textoSecundario` peso 400 para que el enlace resalte.

### Indicador de pasos

Barras de 5 px con el número al lado (`1 de 2`). Paso actual en `amarilloCTA`, pasos pendientes en `inactivo`, pasos completados en `acento`.

Se usa siempre que un flujo tenga más de una pantalla. Sin él, el usuario que ya creó su cuenta y recibe otra pantalla pidiéndole datos no sabe cuánto falta — y ahí es donde abandona.

---

## 6. Reglas de escritura

Estas reglas valen tanto como los colores.

**Explica para qué sirve el dato, no qué es.** "Tu conductor verá tu nombre cuando acepte el viaje" en vez de "Estos datos serán parte de tu perfil de pasajero".

**Ayuda, no juzgues.** Una pista de contraseña dice el requisito ("Mínimo 8 caracteres, con al menos un número"), no una calificación sin salida ("Débil").

**Un solo título por pantalla.** No repetir el título en el header y en el cuerpo.

**Nada de flechas que no llevan a ningún lado.** Si un paso es obligatorio, la pantalla no lleva flecha de volver.

**No prometas lo que no cumples.** Una nota como "tu número no se comparte con el conductor" solo va si la implementación lo garantiza.

---

## 7. Decisiones tomadas y descartadas

**Botón de registro con Google — descartado.** La identidad del usuario en TukiTuki es el número de celular: la verificación es por OTP, el conductor y el pasajero se identifican por teléfono, y el pago va por Yape o Plin, que también son por número. Entrar con Google da un correo y aun así habría que pedir el celular después, así que agrega una pantalla en vez de ahorrarla. Se revisaría solo si el correo pasa a ser identidad — por ejemplo con cuentas institucionales de una universidad.

**Nombre de marca bajo el logo — descartado.** El logo ya contiene el texto integrado.

**Barra de fortaleza de contraseña — descartada.** Coincidía visualmente con el indicador de pasos y generaba confusión. El texto con el requisito cumple la misma función con menos ruido.

**Pie con los tres distritos — descartado.** No aportaba al usuario que ya está usando la app.

**Fondo blanco en vez de crema — descartado.** El crema es cálido y distingue a TukiTuki de las apps de movilidad grandes, que son blanco y azul. Se conserva, con los campos en blanco para garantizar el contraste.

---

## 8. Pendientes

- **Modo oscuro.** No definido. Decidir si se implementa antes de salir al público.
- **Iconografía.** Hoy se usan glifos genéricos de Material. Un set propio, o al menos una selección consistente, está sin decidir.
- **Sistema de sombras.** No existe y por ahora no hace falta: la profundidad se logra con la superposición de la hoja crema sobre el header.
- **Contraste verificado.** Ninguna combinación de las definidas aquí ha sido medida contra un estándar de accesibilidad. Conviene hacerlo antes de publicar.
- **Aplicación al Driver.** Este documento nace de las pantallas del Passenger. Falta revisar los componentes que ya existen en Driver (`DriverOnboardingScaffold`, `DriverOnboardingInfoCard`, `DriverOnboardingProgress`) y alinearlos.
