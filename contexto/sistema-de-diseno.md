# Sistema de diseño — TukiTuki

Definido el 20/08/2026 sobre las pantallas de login, registro y completar perfil de la app Passenger.

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

### Estados

| Nombre | Hex | Uso |
|---|---|---|
| `exito` | `#1F7A3E` | Confirmaciones, pago completado |
| `alerta` | `#E8951A` | Advertencias, pago pendiente |
| `error` | `#E05B4F` | Errores de validación, cancelaciones |
| `destino` | `#D8542C` | Marcador de destino en el mapa |

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

**Ningún elemento tocable mide menos de 44 de lado.** Los campos de 56 y los botones de 58 cumplen con margen.

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
