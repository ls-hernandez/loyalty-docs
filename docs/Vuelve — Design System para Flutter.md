# Vuelve — Design System para Flutter

Oct 1, 2026 · @Sebastian

## Introducción

Este documento traduce el prototipo web de Vuelve a tokens y componentes listos para Flutter, en modo claro y oscuro, para los dos roles de la app: Cliente y Admin. Todo valor aquí sale del prototipo; si algo cambia en el código, se cambia primero en este documento.

**Principios de diseño**

- **La tarjeta de sellos es la protagonista.** Es el único elemento con color fuerte, muescas laterales y perforaciones. El resto de la interfaz es sobria para que la tarjeta resalte.
- **El color identifica la categoría del comercio.** Peluquería, cancha, tienda, restaurante y café tienen cada uno su color de tarjeta, el mismo en claro y oscuro.
- **Una sola animación protagonista por pantalla.** La perforación al sumar un sello y la luz que gira en Nueva reserva. El resto del movimiento solo responde a acciones del usuario.
- **Lenguaje de barrio, claro y directo.** Verbos concretos ("Sumar sello", "Confirmar reserva"), sin tecnicismos.
- **Pensado para una mano.** Acciones principales abajo, botones de ancho completo y objetivos táctiles de mínimo 44 px.

**Cómo usarlo en Flutter**

1. Los colores, tipografías, espaciados y radios viven en una `ThemeExtension` llamada `VuelveTokens`, con una instancia para claro y otra para oscuro.
2. `ThemeData` se construye a partir de esos tokens, así los widgets de Material heredan el estilo sin configuración extra.
3. Los componentes propios (tarjeta de sellos, stepper, carrusel, tarjeta con luz) leen los tokens con `context.tokens` y nunca usan colores literales.
4. El cambio de modo lo controla `ThemeMode.system`, con opción de forzar claro u oscuro desde ajustes.

## Color

La paleta tiene 12 tokens semánticos que cambian entre claro y oscuro, 5 colores de categoría que no cambian y 4 colores fijos para contenido sobre tarjetas. El amarillo `sun` es el acento de marca y es igual en ambos modos.

**Tokens semánticos**

| Token | Uso | Claro | Oscuro |
| --- | --- | --- | --- |
| `background` | Fondo de pantalla (Scaffold), relleno de las perforaciones | `#F6F8F5` | `#151C2B` |
| `canvas` | Fondo exterior en tablet o web, alrededor del contenido | `#E9EDE8` | `#0F1522` |
| `surface` | Listas, campos, chips, tarjetas blancas, barra de pestañas | `#FFFFFF` | `#1C2536` |
| `ink` | Texto principal, botón primario, chip seleccionado | `#17223A` | `#EEF1F6` |
| `muted` | Texto secundario, etiquetas de campo, íconos inactivos | `#5E6878` | `#9AA4B5` |
| `line` | Bordes, divisores, switch apagado, puntos del carrusel | `#D6DCD4` | `#2A3446` |
| `sun` | Acento de marca, botón destacado, foco, luz giratoria | `#F2B630` | `#F2B630` |
| `sunSoft` | Fondo de estado "Pendiente de abono" y notas de aviso | `#FBEBC2` | `#3B321A` |
| `teal` | Acción positiva, pestaña activa, switch encendido, slot elegido | `#1E7A6E` | `#4FB8A8` |
| `tealSoft` | Fondo de estado "Confirmada", avisos no leídos, ítem elegido | `#D5ECE7` | `#1D3A3A` |
| `rose` | Cancelar, eliminar, inasistencias, contador de avisos | `#C8475B` | `#E36C7E` |
| `roseSoft` | Fondo de estado "No asistió" | `#F7DDE1` | `#3D2229` |

El botón primario invierte colores según el modo: fondo `ink` y texto `background`. En claro es azul tinta con texto claro; en oscuro es casi blanco con texto oscuro.

**Colores de categoría (iguales en ambos modos)**

| Categoría | Color de tarjeta | Ícono |
| --- | --- | --- |
| Peluquería | `#F2B630` | Tijeras |
| Cancha | `#8FD3B6` | Balón |
| Tienda | `#F4A7B0` | Bolsa |
| Restaurante | `#A9C4F5` | Cubiertos |
| Café | `#E3C29B` | Taza |

Son tonos claros a propósito: sobre ellos el contenido siempre va en `cardInk` (`#17223A`), aun en modo oscuro. Así la tarjeta se ve igual en ambos modos y se lee como un objeto físico.

**Colores fijos**

| Token | Valor | Uso |
| --- | --- | --- |
| `cardInk` | `#17223A` | Texto e íconos sobre tarjetas de categoría, fondo del badge de premio |
| `onTeal` | `#FFFFFF` | Texto sobre botones y slots `teal` |
| `onSun` | `#17223A` | Texto sobre botones `sun` |
| `cardGlass` | `#FFFFFF` al 50 % | Fondo del ícono dentro de la tarjeta |
| `holeEmpty` | `#17223A` al 35 % | Borde punteado de casilla vacía |
| `scrim` | `#0A0F19` al 45 % | Fondo detrás de bottom sheets |

**Estados de reserva**

| Estado | Fondo | Texto |
| --- | --- | --- |
| Confirmada | `tealSoft` | `teal` |
| Pendiente de abono | `sunSoft` | `ink` |
| Atendida | `ink` | `background` |
| No asistió | `roseSoft` | `rose` |
| Cancelada | `line` | `muted` |

**Reglas de contraste**

- `sun` nunca se usa como color de texto sobre `surface` o `background` claros; solo como fondo con texto `onSun`, o como borde y acento.
- Texto normal: mínimo 4,5:1. `teal` sobre blanco y `#4FB8A8` sobre `#1C2536` cumplen.
- `muted` solo para texto de 13 px o más; nunca para información crítica como precios o estados.
- El estado nunca se comunica solo con color: el badge siempre lleva su palabra.

## Tipografía

Vuelve usa dos familias: **Bricolage Grotesque** para títulos, nombres de comercio, horas y códigos, y **Figtree** para todo el texto de interfaz. Ambas están en Google Fonts; se recomienda empaquetarlas como assets en lugar de descargarlas en tiempo de ejecución, para que la app funcione bien con mala señal.

| Familia | Pesos a incluir | Rol |
| --- | --- | --- |
| Bricolage Grotesque | 500, 700, 800 | Display: títulos, nombre del comercio en tarjeta, horas, códigos |
| Figtree | 400, 500, 600, 700 | Texto: cuerpo, botones, chips, campos, etiquetas |

Los títulos llevan un tracking de −0,01 em (−0,2 px aproximadamente). Ningún texto va en mayúsculas sostenidas.

**Escala tipográfica y mapeo a `TextTheme`**

| Estilo Flutter | Familia | Tamaño (px) | Peso | Interlineado | Uso en la app |
| --- | --- | --- | --- | --- | --- |
| `displaySmall` | Bricolage | 34 | 800 | 1,1 | Código de cliente y de premio (tracking +4 px) |
| `headlineMedium` | Bricolage | 28 | 800 | 1,1 | Título de pantalla ("Hola, Laura", "Reservar"), cifras de Resumen |
| `headlineSmall` | Bricolage | 22 | 800 | 1,2 | Marca "Vuelve" en el header, título de bottom sheet (700) |
| `titleLarge` | Bricolage | 20 | 800 | 1,1 | Nombre del comercio en la tarjeta de sellos |
| `titleMedium` | Bricolage | 18 | 700 | 1,2 | Títulos de sección ("Historial", "Programas") |
| `titleSmall` | Bricolage | 15 | 700 | 1,15 | Hora en listas de reservas y agenda |
| `bodyLarge` | Figtree | 15 | 400 | 1,45 | Texto base y descripciones |
| `bodyMedium` | Figtree | 13 | 400 | 1,45 | Texto secundario de ítems, notas, reglas del programa |
| `labelLarge` | Figtree | 15 | 700 | 1,2 | Botones normales, título de ítem (600) |
| `labelMedium` | Figtree | 13 | 600 | 1,2 | Chips, etiquetas de campo, botones pequeños (700) |
| `labelSmall` | Figtree | 11,5 | 600 | 1,2 | Etiquetas de la barra de pestañas, badges de estado (700) |

Cuando un uso pide otro peso que el de la tabla (por ejemplo, el título de ítem en 600), se aplica con `copyWith(fontWeight: …)` sobre el estilo base, sin crear un estilo nuevo.

Se respeta el tamaño de texto del sistema (`MediaQuery.textScaler`), con tope en 1,3 para que la tarjeta de sellos y la barra de pestañas no se rompan.

## Espaciado, radios, bordes y elevación

Vuelve es una interfaz plana: separa con bordes de 1 px y fondos, casi nunca con sombras. Los radios crecen con el tamaño del elemento, de 10 px en botones pequeños a 26 px en bottom sheets.

**Espaciado**

| Token | Valor (px) | Uso típico |
| --- | --- | --- |
| `space4` | 4 | Separación entre título y subtítulo de un ítem |
| `space6` | 6 | Separación entre chips y entre botones pequeños |
| `space8` | 8 | Separación entre botones en una fila, interior de chips |
| `space10` | 10 | Separación entre secciones de un formulario, gap de listas compactas |
| `space12` | 12 | Gap entre ícono y texto en ítems, entre tarjetas del carrusel |
| `space14` | 14 | Padding horizontal de ítems de lista |
| `space16` | 16 | Padding de tarjetas internas, margen bajo listas |
| `space18` | 18 | Padding de la tarjeta de sellos |
| `space20` | 20 | Margen lateral de pantalla, separación antes de títulos de sección |
| `space24` | 24 | Padding inferior de pantalla y de bottom sheets |

Margen lateral de todas las pantallas: 20 px. Ítem de lista: 13 px vertical y 14 px horizontal. Título de sección: 20 px arriba y 10 px abajo.

**Radios**

| Token | Valor (px) | Elemento |
| --- | --- | --- |
| `radiusS` | 10 | Botón pequeño, campo compacto |
| `radiusM` | 12 | Campo de texto, slot de hora, ícono en cuadro (dot) |
| `radiusL` | 14 | Botón normal, buscador |
| `radiusXL` | 16 | Tarjeta de estadística, resumen de reserva |
| `radius2XL` | 18 | Lista, carrusel, interior de la tarjeta con luz |
| `radius3XL` | 20 | Contenedor del stepper, borde de la tarjeta con luz |
| `radiusCard` | 22 | Tarjeta de sellos |
| `radiusSheet` | 26 | Esquinas superiores de bottom sheets |
| `radiusPill` | 999 | Chips, badges, switch, selector de modo |

**Bordes**

- Bordes estándar: 1 px en `line`, en listas, campos, chips, botones fantasma y tarjetas de estadística.
- Casilla de sello vacía: 2 px punteado en `holeEmpty`. Casilla del premio: 2 px sólido al 60 %.
- Foco de teclado: 3 px en `sun` con 2 px de separación.
- Tarjeta con luz: 2 px de anillo, que es donde gira la luz.
- Ítem elegido en el stepper: barra interna izquierda de 3 px en `teal`.

**Elevación**

- Elevación 0 en toda la app. No se usan sombras de Material en tarjetas, listas ni botones.
- La única sombra es interna: la casilla perforada lleva una sombra interior de 3 px hacia abajo, 6 px de desenfoque y negro al 25 %, que la hace ver hundida.
- Los bottom sheets se separan del contenido con el `scrim`, no con sombra.

**Tamaños de componentes**

| Elemento | Tamaño (px) |
| --- | --- |
| Objetivo táctil mínimo | 44 × 44 |
| Ícono en cuadro (dot) | 38 × 38, ícono de 20 |
| Botón de avisos (campana) | 38 × 38, ícono de 19 |
| Flechas del carrusel | 32 × 32 |
| Switch | 44 × 26, perilla de 20 |
| Muesca de la tarjeta | 22 de diámetro |
| Barra del stepper | 5 de alto |
| Ícono de pestaña | 22 |

## Iconografía y movimiento

Los íconos son de línea, trazo de 2 px y puntas redondeadas, sobre una cuadrícula de 24 px. El paquete [Lucide para Flutter](https://pub.dev/packages/lucide_icons_flutter) tiene casi todos con el mismo estilo; el balón de cancha se exporta del prototipo como SVG propio y se usa con `flutter_svg`.

| Uso | Ícono Lucide | Dónde aparece |
| --- | --- | --- |
| Peluquería | `scissors` | Tarjeta, Explorar, stepper |
| Cancha | SVG propio (balón) | Tarjeta, Explorar, stepper |
| Tienda | `shopping-bag` | Tarjeta, Explorar |
| Restaurante | `utensils` | Tarjeta, Explorar |
| Café | `coffee` | Tarjeta, Explorar |
| Premio | `gift` | Última casilla, Premios, beneficios |
| Tarjetas | `wallet-cards` | Pestaña Tarjetas |
| Reservar y Agenda | `calendar` | Pestañas, próxima reserva |
| Explorar | `compass` | Pestaña Explorar, dirección |
| Sellos | `plus` | Pestaña Sellos (Admin) |
| Programas | `list` | Pestaña Programas (Admin) |
| Clientes | `users` | Pestaña Clientes (Admin) |
| Resumen | `chart-column` | Pestaña Resumen (Admin) |
| Avisos | `bell` | Header en modo Cliente |
| Recordatorio | `clock` | Avisos, horario del comercio |
| Cliente | `user` | Lista de clientes, teléfono |

Tamaño de 20 px dentro de cuadros (dot) y de 22 px en la barra de pestañas. El ícono toma el color del texto que lo acompaña; nunca lleva un color propio.

**Movimiento**

| Animación | Duración | Curva | Detalle |
| --- | --- | --- | --- |
| Perforación de sello | 600 ms | `easeOut` | La casilla nueva pasa de escala 0,4 a 1,15 y vuelve a 1 |
| Luz giratoria en Nueva reserva | 3200 ms por vuelta | `linear`, repetición infinita | Barrido amarillo que recorre el borde de 2 px |
| Apagado de la luz al abrir | 350 ms | `easeOut` | Opacidad de la luz a 0; el stepper se abre a los 320 ms |
| Entrada de bottom sheet | 250 ms | `easeOut` | Sube 40 px y pasa de opacidad 0,4 a 1 |
| Cambio de reserva en el carrusel | 300 ms | `ease` | Desplazamiento horizontal de una página completa |
| Punto activo del carrusel | 200 ms | `ease` | El punto se alarga de 8 a 22 px |
| Switch | 200 ms | `ease` | Desplazamiento de la perilla y cambio de color |
| Cambio de paso del stepper | 200 ms | `easeOut` | Fundido con `AnimatedSwitcher` |
| Toast | 2600 ms visible | — | Aparece sobre la barra de pestañas y se retira solo |

Las dos animaciones firma son la perforación y la luz giratoria. No se agregan entradas animadas a listas ni a secciones: el movimiento solo responde a lo que hace el usuario, salvo la luz.

**Reducción de movimiento**

Si `MediaQuery.of(context).disableAnimations` es verdadero, la luz no gira y el borde queda en `sun` fijo, el sello aparece sin rebote y los cambios de paso y de carrusel son inmediatos.

## Componentes

Cada componente se construye una sola vez como widget propio y lee sus colores de los tokens; el nombre sugerido del widget va entre paréntesis.

### Tarjeta de sellos (`StampCard`)

El componente firma. Simula una tarjeta física perforada con el color de la categoría del comercio.

- **Contenedor:** color de categoría, radio 22, padding 18 arriba y a los lados y 16 abajo. Dos muescas circulares de 22 px en color `background`, centradas verticalmente y salidas a medias por izquierda y derecha.
- **Encabezado:** nombre del comercio en `titleLarge` y regla del programa en `bodyMedium` ("Cada 10 visitas: tu corte número 11 es gratis") al 82 % de opacidad, ambos en `cardInk`. A la derecha, el ícono de categoría en un cuadro de 38 px con fondo `cardGlass`.
- **Cuadrícula:** gap de 9 px y casillas circulares. Son 5 columnas si la meta es de 5 o menos, múltiplo de 5 o mayor que 12; en otro caso, el mínimo entre 6 y la meta.
- **Pie:** "4 de 10 visitas" a la izquierda y "Te faltan 6", "Te falta 1" o "¡Lista!" a la derecha, en 13/600.

| Estado de casilla | Aspecto |
| --- | --- |
| Vacía | Borde punteado de 2 px en `holeEmpty` y número en 11/700 al 45 % |
| Perforada | Relleno `background` con sombra interior, sin número |
| Premio pendiente | Borde sólido al 60 % con ícono `gift` al 55 % del tamaño |
| Premio alcanzado | Relleno `cardInk` con ícono `gift` en `sun` |

Tiene dos variantes. La interactiva, en Tarjetas y Explorar, abre el detalle al tocarla. La estática se usa en vistas previas: el formulario de programa, la vista previa del cliente en Sellos y el detalle.

### Botones (`VButton`)

| Variante | Fondo | Texto | Borde | Uso |
| --- | --- | --- | --- | --- |
| Primario | `ink` | `background` | — | Acción general ("Listo", "Validar y canjear") |
| Destacado | `sun` | `onSun` | — | "Mostrar mi código", "Nuevo programa" |
| Positivo | `teal` | `onTeal` | — | Confirmar, guardar, sumar sello, reservar, atendida |
| Fantasma | transparente | `ink` | 1 px `line` | Cancelar, editar, acciones secundarias |
| Peligro | transparente | `rose` | 1 px `rose` | Cancelar reserva, eliminar |

| Tamaño | Padding (px) | Radio | Texto |
| --- | --- | --- | --- |
| Normal | 12 × 16 | 14 | 15/700 |
| Pequeño | 7 × 12 | 10 | 13/700 |

Puede ocupar el ancho completo. Deshabilitado va al 45 % de opacidad, sin cambio de color.

### Chips (`VChip`)

- Forma de píldora, fondo `surface`, borde `line`, padding 7 × 12 y texto en `labelMedium`.
- Seleccionado: fondo `ink`, texto `background` y sin borde visible.
- Puede llevar un texto secundario más liviano ("30 min · $20.000") en 500 y `muted`, que pasa al 75 % cuando el chip está seleccionado.
- En fila con salto de línea para opciones, o en fila con scroll horizontal para días y categorías.

### Campos de texto y buscador (`VTextField`, `VSearchField`)

- La etiqueta va arriba en 13/600 `muted`, con 5 px de separación del campo.
- Campo: fondo `surface`, borde `line`, radio 12 y padding 11 × 12. El foco agrega un anillo de 3 px en `sun`.
- El buscador lleva una lupa a 14 px del borde izquierdo, padding izquierdo de 40, radio 14 y fondo `background` cuando está dentro de un contenedor `surface`.

### Switch (`VSwitch`)

De 44 × 26 px con perilla blanca de 20. Apagado en `line` y encendido en `teal`. Siempre lleva una etiqueta accesible con el nombre del programa.

### Listas e ítems (`VList`, `VListItem`)

- **Contenedor:** fondo `surface`, borde `line`, radio 18 y recorte de contenido. Divisor de 1 px en `line` entre ítems.
- **Ítem:** padding 13 × 14, gap 12. Inicio opcional: cuadro de 38 px y radio 12 con ícono. Centro: título en 15/600 y subtítulo en 13 `muted`. Final: badge, botón, precio o switch.
- **Variantes:**
  - Seleccionable: al elegirlo toma fondo `tealSoft` y una barra interna izquierda de 3 px en `teal`.
  - No leído: fondo `tealSoft`, se usa en Avisos.
  - Encabezado de grupo: fondo `background`, texto 13/700 `muted`, padding 9 × 14. Se usa para separar profesionales o canchas en la agenda.

### Badges (`StatusBadge`, `RewardBadge`, `CountBadge`)

- **Estado:** píldora con padding 2 × 8 y texto en 11,5/700, con los colores de la tabla de estados de reserva.
- **Premio:** fondo `cardInk` y texto `sun`, en 12/700, por ejemplo "Premio" o "4/10".
- **Contador:** círculo `rose` de mínimo 18 px con número blanco en 11/700, sobre la campana de avisos.

### Ítem de reserva (`BookingTile`)

- Columna de hora de 62 px: hora en `titleSmall` y debajo "p. m." y el día ("Mañana") en 12/500 `muted`.
- Centro: nombre del comercio (o del cliente, en Admin), servicio con el valor ("Corte con Julián · $20.000"), badge de estado y, si aplica, "Sello sumado".
- Acciones debajo, en botones pequeños:
  - Cliente: "Cancelar reserva" en reservas activas.
  - Admin con reserva confirmada: "Atendida", "No asistió" y "Cancelar".
  - Admin con reserva pendiente de abono: "Confirmar abono" y "Cancelar".

### Carrusel de próximas reservas (`UpcomingCarousel`)

- Un `PageView` con una reserva por página; cada página es una lista de un solo `BookingTile`, con radio 18.
- Encabezado: el título "Mis próximas reservas" y a la derecha el contador ("1 de 3") y las flechas de 32 px. Cada flecha se deshabilita al llegar a su extremo.
- Debajo, puntos de 8 px en `line`; el activo se alarga a 22 px en `teal`. Tocar un punto lleva a esa reserva.
- Se navega deslizando, con las flechas o con los puntos. Con una sola reserva no se muestran flechas ni puntos.
- Sin reservas: un ítem con ícono de calendario, "No tienes reservas activas" y "Crea una arriba en cuatro pasos".

### Tarjeta con luz (`GlowCard`)

Es la versión colapsada de Nueva reserva y la segunda animación firma.

- **Anillo:** contenedor de radio 20 con padding de 2 px y fondo `line`. Detrás gira un `SweepGradient`: transparente de 0° a 250°, `sun` al 15 % en 280°, `sun` en 330°, `#FFF6D8` en 350° y transparente en 360°.
- **Interior:** fondo `surface`, radio 18 y padding 16. Título en Bricolage 17/700, subtítulo en 13 `muted` y un chevron "›" de 24 px en `muted`.
- **Textos:** sin borrador, "Nueva reserva" y "Elige comercio, servicio y hora en 4 pasos". Con borrador, "Continuar reserva" y, por ejemplo, "Barbería El Faro · paso 2 de 4".
- **Al tocar:** la luz se desvanece en 350 ms, la tarjeta se deshabilita y a los 320 ms se reemplaza por el stepper.

### Stepper de reserva (`BookingStepper`)

- **Contenedor:** fondo `surface`, borde `line`, radio 20 y padding 14. Las listas internas usan fondo `background` para diferenciarse.
- **Progreso:** 4 segmentos en cuadrícula con gap de 6. Cada uno es una barra de 5 px con su etiqueta debajo en 11,5/600.
  - Completado: barra `teal`.
  - Actual: barra `sun` y etiqueta en `ink`.
  - Pendiente: barra `line` y etiqueta en `muted`.
  - Se puede volver a un paso anterior tocándolo.
- **Título:** "Paso 2 de 4 · Servicio" en Bricolage 19/700.
- **Pie:** "Atrás" (fantasma, flex 1) y "Siguiente" (positivo, flex 2), deshabilitado hasta completar el paso. En el último paso, "Confirmar reserva".

| Paso | Contenido |
| --- | --- |
| 1. Comercio | Buscador, chips de categoría, contador de resultados, lista seleccionable de 5 comercios y "Mostrar 5 más". Al elegir uno avanza solo |
| 2. Servicio o Cancha | Peluquería: lista de servicios con duración y precio, más chips de profesional con "Cualquiera". Cancha: lista de canchas y las tarifas diurna y nocturna |
| 3. Día y hora | Chips de 7 días con scroll horizontal. Cuadrícula de horas libres en 3 columnas con gap de 7; cada hora es una celda de radio 12, padding 10 y texto 13,5/600, que toma fondo `teal` y texto `onTeal` al elegirla |
| 4. Confirmar | Lista resumen (Comercio, Servicio, Fecha y hora, Valor), cada dato con "Cambiar", y las notas de sello y de abono |

### Bottom sheet (`VSheet`)

- Fondo `background`, esquinas superiores de radio 26 y altura máxima del 92 % de la pantalla.
- Asa de 40 × 5 px en `line`, padding 10 × 20 y 24 más el área segura abajo.
- Título en `headlineSmall` 700. Fondo detrás con `scrim`; tocarlo cierra el sheet.
- Se usa para mostrar códigos, avisos, formularios (programa, comercio, cliente, configuración de reservas), confirmaciones y premio desbloqueado.

### Toast (`VToast`)

Fondo `ink`, texto `background` en 14/600, radio 12 y padding 10 × 16. Va flotando y centrado a 86 px del borde inferior, con ancho máximo del 85 %. Es la versión propia de `SnackBar` con `behavior: floating`.

### Barra de pestañas (`VTabBar`)

- Fondo `surface` con borde superior en `line`. Ícono de 22 sobre la etiqueta en `labelSmall`.
- Inactiva en `muted`, activa en `teal`.
- Cliente tiene 4 pestañas y Admin 5 (ver Patrones por modo). Se implementa con `NavigationBar` sin indicador o con un widget propio.

### Header (`VHeader`)

- **Logo:** círculo de 34 px en `ink` con un círculo interno `sun` a 10 px del borde, seguido de "Vuelve" en `headlineSmall`.
- **Derecha:** la campana de avisos (solo en modo Cliente) y el selector de modo Cliente / Admin. El selector es una píldora `surface` con borde; el segmento activo va en `ink` con texto `background`, en 13/600.

### Notas, estadísticas, código y estados vacíos

- **Nota:** fondo `sunSoft` para advertencias (abono) o `tealSoft` para información (sello al asistir), texto 13, radio 12 y padding 10 × 12.
- **Estadística:** contenedor `surface` con borde, radio 16 y padding 14. Cifra en Bricolage 28/800 y etiqueta en 13 `muted`. Se usa en cuadrícula de 2 columnas con gap de 10.
- **Código:** `displaySmall` centrado sobre `surface`, con borde punteado de 2 px en `line`, radio 16 y padding 16. Se usa para el código de cliente y el de premio.
- **Estado vacío:** texto centrado en `muted` con padding 22 × 10. Explica qué aparecerá ahí y, cuando aplica, ofrece una acción ("Ir a Explorar").

## Patrones por modo

Cliente y Admin comparten tokens y componentes; cambian la navegación, el header y el tipo de tarea. Cliente consulta y reserva. Admin opera en el mostrador, rápido y con una mano.

En el prototipo los dos modos se cambian con el selector del header. En la app real conviene que sean dos roles de una misma app, o dos apps (Vuelve y Vuelve Comercios) con el mismo design system. El selector solo se muestra a usuarios que tienen ambos roles.

**Modo Cliente: 4 pestañas**

| Pestaña | Ícono | Contenido | Componentes |
| --- | --- | --- | --- |
| Tarjetas | `wallet-cards` | Saludo, premios listos, próxima reserva, "Mostrar mi código" y las tarjetas de sellos de los comercios a los que pertenece | `StampCard`, `VListItem`, `VButton` destacado |
| Reservar | `calendar` | Nueva reserva (colapsada con luz o abierta como stepper), carrusel de próximas reservas e historial con las 5 más recientes | `GlowCard`, `BookingStepper`, `UpcomingCarousel`, `BookingTile` |
| Premios | `gift` | Premios disponibles, usados y vencidos, con el código para canjear | `VList`, `VSheet` con código |
| Explorar | `compass` | Todos los comercios con buscador y filtro por categoría, y detalle de cada comercio | `VSearchField`, `VChip`, `VList`, vista de detalle |

- **Header:** logo, campana de avisos con contador y selector de modo.
- **Detalle de comercio:** tarjeta de presentación con el color de categoría y su descripción, botones Reservar y Unirme, e Información (dirección, horario, teléfono, reservas). Luego tus próximas reservas en ese comercio, Servicios o Canchas y tarifas, Lo que ofrece y Beneficios para clientes.
- **Avisos:** bottom sheet con la lista de avisos. Los no leídos van en `tealSoft` y se marcan como leídos al abrir.
- **Volver:** en vistas de detalle hay un enlace "‹ Explorar" o "‹ Mis tarjetas" en `teal` arriba a la izquierda, además del gesto atrás del sistema.

**Modo Admin: 5 pestañas**

| Pestaña | Ícono | Contenido | Componentes |
| --- | --- | --- | --- |
| Agenda | `calendar` | Reservas del día agrupadas por profesional o cancha, con chips de 7 días, y los botones Configurar y Nueva reserva | `VChip`, `VList` con encabezados de grupo, `BookingTile` con acciones |
| Sellos | `plus` | Buscar cliente, elegir programa, sumar sello con vista previa de la tarjeta, registrar cliente nuevo y canjear premio por código | `VTextField`, `StampCard` estática, `VButton` positivo |
| Programas | `list` | Lista de programas con switch de activo, editar y Nuevo programa | `VListItem`, `VSwitch`, `VSheet` con formulario |
| Clientes | `users` | Clientes del comercio con progreso, reservas e inasistencias | `VListItem` |
| Resumen | `chart-column` | Estadísticas, clientes a punto de completar y restablecer datos de ejemplo | Tarjeta de estadística, `VList` |

- **Cabecera de cada pestaña:** nombre del comercio en `headlineMedium`, categoría y dirección, y debajo el selector de negocio con los botones Editar y Nuevo.
- **Sin campana de avisos:** los avisos son para el cliente final.
- **Acciones destructivas:** cancelar y eliminar piden confirmación en un diálogo. Cancelar una reserva avisa al cliente.
- **Premio desbloqueado:** al sumar el sello que completa la tarjeta, se abre un bottom sheet con el código y las opciones "Guardar para después" y "Canjear ahora".

**Comportamientos compartidos**

- Toda acción con efecto muestra un toast con el resultado, usando el mismo verbo del botón ("Sumar sello" produce "Sello sumado: 5 de 10").
- Los formularios van en bottom sheets, nunca en pantallas nuevas, para no perder el contexto.
- Los estados vacíos siempre dicen qué aparecerá ahí y, si se puede, ofrecen la acción para llenarlos.

## Implementación en Flutter

El tema se arma en cuatro archivos: los tokens de color como `ThemeExtension`, las constantes de espaciado y radios, el `TextTheme` y el `ThemeData` de cada modo. El código asume Flutter 3.27 o superior, Material 3 y Dart 3.

**Estructura de carpetas**

```
lib/
  theme/
    vuelve_tokens.dart      // colores claro/oscuro + fijos + categorías
    vuelve_spacing.dart     // VSpace y VRadius
    vuelve_text.dart        // TextTheme
    vuelve_theme.dart       // buildTheme(Brightness)
  widgets/
    stamp_card.dart  glow_card.dart  booking_stepper.dart
    upcoming_carousel.dart  booking_tile.dart  status_badge.dart
    v_button.dart  v_chip.dart  v_sheet.dart  v_header.dart
  features/
    cliente/   tarjetas/  reservar/  premios/  explorar/
    admin/     agenda/  sellos/  programas/  clientes/  resumen/
assets/fonts/  BricolageGrotesque-*.ttf  Figtree-*.ttf
```

**Fuentes en `pubspec.yaml`**

```yaml
flutter:
  fonts:
    - family: BricolageGrotesque
      fonts:
        - asset: assets/fonts/BricolageGrotesque-Medium.ttf
          weight: 500
        - asset: assets/fonts/BricolageGrotesque-Bold.ttf
          weight: 700
        - asset: assets/fonts/BricolageGrotesque-ExtraBold.ttf
          weight: 800
    - family: Figtree
      fonts:
        - asset: assets/fonts/Figtree-Regular.ttf
          weight: 400
        - asset: assets/fonts/Figtree-Medium.ttf
          weight: 500
        - asset: assets/fonts/Figtree-SemiBold.ttf
          weight: 600
        - asset: assets/fonts/Figtree-Bold.ttf
          weight: 700
```

**Tokens de color (`vuelve_tokens.dart`)**

```dart
import 'package:flutter/material.dart';

@immutable
class VuelveTokens extends ThemeExtension<VuelveTokens> {
  const VuelveTokens({
    required this.background, required this.canvas, required this.surface,
    required this.ink, required this.muted, required this.line,
    required this.sun, required this.sunSoft, required this.teal,
    required this.tealSoft, required this.rose, required this.roseSoft,
  });

  final Color background, canvas, surface, ink, muted, line;
  final Color sun, sunSoft, teal, tealSoft, rose, roseSoft;

  // Fijos: no cambian entre modos
  static const cardInk = Color(0xFF17223A);
  static const onTeal = Color(0xFFFFFFFF);
  static const onSun = Color(0xFF17223A);
  static const cardGlass = Color(0x80FFFFFF); // blanco 50 %
  static const holeEmpty = Color(0x5917223A); // cardInk 35 %
  static const scrim = Color(0x730A0F19);     // 45 %

  static const categories = <String, Color>{
    'peluqueria': Color(0xFFF2B630),
    'cancha': Color(0xFF8FD3B6),
    'tienda': Color(0xFFF4A7B0),
    'restaurante': Color(0xFFA9C4F5),
    'cafe': Color(0xFFE3C29B),
  };

  static const light = VuelveTokens(
    background: Color(0xFFF6F8F5), canvas: Color(0xFFE9EDE8),
    surface: Color(0xFFFFFFFF), ink: Color(0xFF17223A),
    muted: Color(0xFF5E6878), line: Color(0xFFD6DCD4),
    sun: Color(0xFFF2B630), sunSoft: Color(0xFFFBEBC2),
    teal: Color(0xFF1E7A6E), tealSoft: Color(0xFFD5ECE7),
    rose: Color(0xFFC8475B), roseSoft: Color(0xFFF7DDE1),
  );

  static const dark = VuelveTokens(
    background: Color(0xFF151C2B), canvas: Color(0xFF0F1522),
    surface: Color(0xFF1C2536), ink: Color(0xFFEEF1F6),
    muted: Color(0xFF9AA4B5), line: Color(0xFF2A3446),
    sun: Color(0xFFF2B630), sunSoft: Color(0xFF3B321A),
    teal: Color(0xFF4FB8A8), tealSoft: Color(0xFF1D3A3A),
    rose: Color(0xFFE36C7E), roseSoft: Color(0xFF3D2229),
  );

  @override
  VuelveTokens copyWith({Color? background, Color? surface, Color? ink}) =>
      VuelveTokens(
        background: background ?? this.background, canvas: canvas,
        surface: surface ?? this.surface, ink: ink ?? this.ink,
        muted: muted, line: line, sun: sun, sunSoft: sunSoft,
        teal: teal, tealSoft: tealSoft, rose: rose, roseSoft: roseSoft,
      );

  @override
  VuelveTokens lerp(ThemeExtension<VuelveTokens>? other, double t) {
    if (other is! VuelveTokens) return this;
    Color l(Color a, Color b) => Color.lerp(a, b, t)!;
    return VuelveTokens(
      background: l(background, other.background), canvas: l(canvas, other.canvas),
      surface: l(surface, other.surface), ink: l(ink, other.ink),
      muted: l(muted, other.muted), line: l(line, other.line),
      sun: l(sun, other.sun), sunSoft: l(sunSoft, other.sunSoft),
      teal: l(teal, other.teal), tealSoft: l(tealSoft, other.tealSoft),
      rose: l(rose, other.rose), roseSoft: l(roseSoft, other.roseSoft),
    );
  }
}

extension VuelveContext on BuildContext {
  VuelveTokens get tokens => Theme.of(this).extension<VuelveTokens>()!;
}
```

**Espaciado y radios (`vuelve_spacing.dart`)**

```dart
abstract final class VSpace {
  static const s4 = 4.0, s6 = 6.0, s8 = 8.0, s10 = 10.0, s12 = 12.0;
  static const s14 = 14.0, s16 = 16.0, s18 = 18.0, s20 = 20.0, s24 = 24.0;
  static const screen = 20.0; // margen lateral
}

abstract final class VRadius {
  static const s = 10.0, m = 12.0, l = 14.0, xl = 16.0, x2l = 18.0;
  static const x3l = 20.0, card = 22.0, sheet = 26.0, pill = 999.0;
}
```

**Tipografía (`vuelve_text.dart`)**

```dart
import 'package:flutter/material.dart';

TextTheme buildTextTheme(Color ink) {
  const d = 'BricolageGrotesque', b = 'Figtree';
  TextStyle s(String f, double size, FontWeight w, double h, [double ls = 0]) =>
      TextStyle(fontFamily: f, fontSize: size, fontWeight: w, height: h,
          letterSpacing: ls, color: ink);
  return TextTheme(
    displaySmall: s(d, 34, FontWeight.w800, 1.1, 4),
    headlineMedium: s(d, 28, FontWeight.w800, 1.1, -0.3),
    headlineSmall: s(d, 22, FontWeight.w800, 1.2, -0.2),
    titleLarge: s(d, 20, FontWeight.w800, 1.1, -0.2),
    titleMedium: s(d, 18, FontWeight.w700, 1.2, -0.2),
    titleSmall: s(d, 15, FontWeight.w700, 1.15),
    bodyLarge: s(b, 15, FontWeight.w400, 1.45),
    bodyMedium: s(b, 13, FontWeight.w400, 1.45),
    labelLarge: s(b, 15, FontWeight.w700, 1.2),
    labelMedium: s(b, 13, FontWeight.w600, 1.2),
    labelSmall: s(b, 11.5, FontWeight.w600, 1.2),
  );
}
```

**Tema por modo (`vuelve_theme.dart`)**

```dart
import 'package:flutter/material.dart';
import 'vuelve_tokens.dart';
import 'vuelve_text.dart';
import 'vuelve_spacing.dart';

ThemeData buildTheme(Brightness brightness) {
  final t = brightness == Brightness.light ? VuelveTokens.light : VuelveTokens.dark;
  final text = buildTextTheme(t.ink);
  final selected = WidgetState.selected;
  OutlineInputBorder border(Color c, [double w = 1]) => OutlineInputBorder(
      borderRadius: BorderRadius.circular(VRadius.m),
      borderSide: BorderSide(color: c, width: w));

  return ThemeData(
    useMaterial3: true,
    brightness: brightness,
    scaffoldBackgroundColor: t.background,
    canvasColor: t.background,
    textTheme: text,
    extensions: [t],
    colorScheme: ColorScheme(
      brightness: brightness,
      primary: t.ink, onPrimary: t.background,
      secondary: t.teal, onSecondary: VuelveTokens.onTeal,
      tertiary: t.sun, onTertiary: VuelveTokens.onSun,
      error: t.rose, onError: Colors.white,
      surface: t.surface, onSurface: t.ink, onSurfaceVariant: t.muted,
      outline: t.line, outlineVariant: t.line,
      surfaceTint: Colors.transparent,
    ),
    dividerTheme: DividerThemeData(color: t.line, thickness: 1, space: 1),
    cardTheme: CardThemeData(
      elevation: 0, color: t.surface, margin: EdgeInsets.zero,
      shape: RoundedRectangleBorder(
          borderRadius: BorderRadius.circular(VRadius.x2l),
          side: BorderSide(color: t.line)),
    ),
    filledButtonTheme: FilledButtonThemeData(
      style: FilledButton.styleFrom(
        backgroundColor: t.ink, foregroundColor: t.background,
        minimumSize: const Size(44, 44), elevation: 0,
        padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 12),
        shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(VRadius.l)),
        textStyle: text.labelLarge,
      ),
    ),
    outlinedButtonTheme: OutlinedButtonThemeData(
      style: OutlinedButton.styleFrom(
        foregroundColor: t.ink, side: BorderSide(color: t.line),
        minimumSize: const Size(44, 44),
        padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 12),
        shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(VRadius.l)),
        textStyle: text.labelLarge,
      ),
    ),
    inputDecorationTheme: InputDecorationTheme(
      filled: true, fillColor: t.surface,
      contentPadding: const EdgeInsets.symmetric(horizontal: 12, vertical: 11),
      border: border(t.line), enabledBorder: border(t.line),
      focusedBorder: border(t.sun, 3),
      hintStyle: text.bodyLarge?.copyWith(color: t.muted),
      labelStyle: text.labelMedium?.copyWith(color: t.muted),
    ),
    chipTheme: ChipThemeData(
      backgroundColor: t.surface, selectedColor: t.ink,
      side: BorderSide(color: t.line), shape: const StadiumBorder(),
      showCheckmark: false,
      padding: const EdgeInsets.symmetric(horizontal: 12, vertical: 7),
      labelStyle: text.labelMedium,
      secondaryLabelStyle: text.labelMedium?.copyWith(color: t.background),
    ),
    switchTheme: SwitchThemeData(
      thumbColor: const WidgetStatePropertyAll(Colors.white),
      trackColor: WidgetStateProperty.resolveWith(
          (s) => s.contains(selected) ? t.teal : t.line),
      trackOutlineColor: const WidgetStatePropertyAll(Colors.transparent),
    ),
    bottomSheetTheme: BottomSheetThemeData(
      backgroundColor: t.background, modalBarrierColor: VuelveTokens.scrim,
      showDragHandle: true, dragHandleColor: t.line,
      dragHandleSize: const Size(40, 5), elevation: 0,
      shape: const RoundedRectangleBorder(
          borderRadius: BorderRadius.vertical(top: Radius.circular(VRadius.sheet))),
    ),
    snackBarTheme: SnackBarThemeData(
      behavior: SnackBarBehavior.floating, backgroundColor: t.ink, elevation: 0,
      contentTextStyle: text.labelMedium?.copyWith(color: t.background, fontSize: 14),
      shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(VRadius.m)),
    ),
    navigationBarTheme: NavigationBarThemeData(
      backgroundColor: t.surface, indicatorColor: Colors.transparent,
      elevation: 0, height: 64,
      labelTextStyle: WidgetStateProperty.resolveWith((s) => text.labelSmall!
          .copyWith(color: s.contains(selected) ? t.teal : t.muted)),
      iconTheme: WidgetStateProperty.resolveWith((s) => IconThemeData(
          size: 22, color: s.contains(selected) ? t.teal : t.muted)),
    ),
  );
}
```

**Uso en `MaterialApp`**

```dart
MaterialApp(
  theme: buildTheme(Brightness.light),
  darkTheme: buildTheme(Brightness.dark),
  themeMode: ThemeMode.system, // o el que elija el usuario en ajustes
  builder: (context, child) =>
      MediaQuery.withClampedTextScaling(maxScaleFactor: 1.3, child: child!),
  home: const HomeScreen(),
);
```

**Ejemplo de componente propio: `GlowCard`**

Muestra cómo un componente firma consume los tokens y respeta la reducción de movimiento.

```dart
import 'dart:math' as math;
import 'package:flutter/material.dart';
import '../theme/vuelve_tokens.dart';
import '../theme/vuelve_spacing.dart';

class GlowCard extends StatefulWidget {
  const GlowCard({super.key, required this.title, required this.subtitle, required this.onOpen});
  final String title, subtitle;
  final VoidCallback onOpen;
  @override
  State<GlowCard> createState() => _GlowCardState();
}

class _GlowCardState extends State<GlowCard> with SingleTickerProviderStateMixin {
  late final AnimationController _spin =
      AnimationController(vsync: this, duration: const Duration(milliseconds: 3200));
  bool _off = false;

  bool get _reduce => MediaQuery.of(context).disableAnimations;

  @override
  void didChangeDependencies() {
    super.didChangeDependencies();
    if (_reduce) {
      _spin.stop();
    } else if (!_spin.isAnimating) {
      _spin.repeat();
    }
  }

  @override
  void dispose() {
    _spin.dispose();
    super.dispose();
  }

  Future<void> _tap() async {
    if (_off) return;
    if (_reduce) return widget.onOpen();
    setState(() => _off = true);
    await Future.delayed(const Duration(milliseconds: 320));
    if (mounted) widget.onOpen();
  }

  @override
  Widget build(BuildContext context) {
    final t = context.tokens;
    final text = Theme.of(context).textTheme;
    return Semantics(
      button: true,
      label: '${widget.title}. ${widget.subtitle}',
      child: GestureDetector(
        onTap: _tap,
        child: ClipRRect(
          borderRadius: BorderRadius.circular(VRadius.x3l),
          child: Stack(children: [
            Positioned.fill(child: ColoredBox(color: _reduce ? t.sun : t.line)),
            if (!_reduce)
              Positioned.fill(
                child: AnimatedOpacity(
                  opacity: _off ? 0 : 1,
                  duration: const Duration(milliseconds: 350),
                  curve: Curves.easeOut,
                  child: AnimatedBuilder(
                    animation: _spin,
                    builder: (_, __) =>
                        CustomPaint(painter: _SweepPainter(_spin.value, t.sun)),
                  ),
                ),
              ),
            Padding(
              padding: const EdgeInsets.all(2),
              child: Container(
                padding: const EdgeInsets.all(VSpace.s16),
                decoration: BoxDecoration(
                    color: t.surface,
                    borderRadius: BorderRadius.circular(VRadius.x2l)),
                child: Row(children: [
                  Expanded(
                    child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
                      Text(widget.title, style: text.titleMedium?.copyWith(fontSize: 17)),
                      const SizedBox(height: 2),
                      Text(widget.subtitle, style: text.bodyMedium?.copyWith(color: t.muted)),
                    ]),
                  ),
                  Text('›', style: TextStyle(fontSize: 24, color: t.muted)),
                ]),
              ),
            ),
          ]),
        ),
      ),
    );
  }
}

class _SweepPainter extends CustomPainter {
  _SweepPainter(this.turn, this.sun);
  final double turn;
  final Color sun;

  @override
  void paint(Canvas canvas, Size size) {
    final rect = Rect.fromCircle(center: size.center(Offset.zero), radius: size.longestSide);
    final paint = Paint()
      ..shader = SweepGradient(
        colors: [Colors.transparent, Colors.transparent, sun.withValues(alpha: .15),
                 sun, const Color(0xFFFFF6D8), Colors.transparent],
        stops: const [0, .694, .778, .917, .972, 1], // 250°, 280°, 330°, 350°
        transform: GradientRotation(turn * 2 * math.pi),
      ).createShader(rect);
    canvas.drawRect(rect, paint);
  }

  @override
  bool shouldRepaint(_SweepPainter old) => old.turn != turn || old.sun != sun;
}
```

**Notas para los demás componentes**

- **Perforación del sello:** usa un `TweenSequence` de escala, de 0,4 a 1,15 (60 % del tiempo) y de 1,15 a 1 (40 %), en 600 ms.
- **Sombra interior de la casilla:** `BoxShadow` de Flutter no tiene modo interior. Se logra con el paquete `flutter_inset_shadow` o pintando un degradado radial oscuro dentro del círculo.
- **Muescas de la tarjeta:** dos círculos de 22 px en color `background` posicionados con `Positioned` a −11 px por cada lado, dentro de un `Stack` con `clipBehavior: Clip.hardEdge`.
- **Carrusel:** `PageView` con `PageController`; las flechas usan `animateToPage` con 300 ms y `Curves.ease`.
- **Botón destacado y positivo:** variantes de `FilledButton.styleFrom` con `sun`/`onSun` y `teal`/`onTeal`, expuestas como constructores con nombre de `VButton`.

## Contenido, formatos y accesibilidad

Vuelve habla como el dueño de un negocio de barrio: directo, cercano y sin tecnicismos, tuteando al cliente. Los textos usan formato de oración (solo la primera letra en mayúscula) y nunca mayúsculas sostenidas.

**Reglas de escritura**

- Los botones dicen exactamente lo que pasa al tocarlos, y la acción mantiene el nombre en todo el flujo: "Sumar sello" produce "Sello sumado" y "Canjear ahora" produce "Premio canjeado".
- El progreso se dice en lo que falta, no en porcentajes: "Te faltan 6", "Te falta 1", "¡Lista!".
- Los errores dicen qué pasó y cómo seguir, sin disculpas: "Ese código no existe. Revisa las letras."
- Los estados vacíos invitan a actuar: "Aún no tienes tarjetas. Entra a Explorar y únete a tus comercios favoritos."
- Se nombran las cosas como las entiende el usuario: "Mostrar mi código", no "ID de cliente".

**Formatos (locale `es_CO`)**

| Dato | Formato | Ejemplo | Flutter (`intl`) |
| --- | --- | --- | --- |
| Moneda | Símbolo $ y punto de miles, sin decimales | $20.000 | `NumberFormat.currency(locale: 'es_CO', symbol: '\$', decimalDigits: 0)` |
| Hora | 12 horas con a. m. / p. m. | 3:00 p. m. | `DateFormat('h:mm a', 'es_CO')`, ajustando el sufijo a "a. m." y "p. m." |
| Día relativo | Hoy, Mañana o día abreviado | Hoy, Mañana, vie 3 oct | `DateFormat('EEE d MMM', 'es_CO')` |
| Fecha corta | Día y mes abreviado | 1 oct | `DateFormat('d MMM', 'es_CO')` |
| Tiempo restante | Minutos, horas o días | En 3 horas, En 2 días | Función propia |
| Celular | 10 dígitos sin espacios | 3104567890 | — |
| Código de premio | 6 caracteres sin O, 0, I ni 1 | K7M2QX | Generado en el backend |

Se inicializa el locale con `initializeDateFormatting('es_CO')` antes de mostrar fechas.

**Accesibilidad**

- **Contraste:** 4,5:1 mínimo en texto normal y 3:1 en íconos y bordes de controles; los tokens de esta guía cumplen en ambos modos.
- **Objetivos táctiles:** de 44 × 44 px como mínimo. Los botones pequeños mantienen esa área con `materialTapTargetSize` o con padding invisible.
- **Lectores de pantalla:**
  - La tarjeta de sellos se anuncia como "Barbería El Faro, 4 de 10 visitas".
  - Las flechas del carrusel dicen "Reserva anterior" y "Reserva siguiente".
  - El stepper anuncia "Paso 2 de 4, Servicio".
  - El switch dice qué programa activa.
- **Color:** ningún estado depende solo del color; los badges llevan siempre su palabra.
- **Texto:** se respeta el tamaño del sistema hasta 1,3×.
- **Movimiento:** se respeta la reducción de movimiento (ver Iconografía y movimiento).
- **Foco:** anillo de 3 px en `sun`, visible en navegación por teclado o switch de accesibilidad.
- **Modo oscuro:** sigue la configuración del sistema, con opción de forzar claro u oscuro. Las tarjetas de categoría no cambian de color entre modos.
