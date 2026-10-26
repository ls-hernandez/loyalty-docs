# Design System — Anexo Negocios

Anexo de `Vuelve — Design System para Flutter.md` para la app `loyalty_business`. **No cambia ningún token**:
colores, tipografía, espaciado, radios, íconos y movimiento son los mismos. Este anexo define la navegación de la
app de negocios, el layout en tablet y los componentes que el prototipo no tenía.

En código, los componentes `V…` del documento base se llaman `L…` (`VButton` → `LButton`).

## 1. Qué cambia respecto al modo Admin del prototipo

| Prototipo (modo Admin) | App de negocios |
|---|---|
| Selector de modo Cliente / Admin en el header | No existe: es otra app |
| Selector de negocio con Editar y Nuevo | No existe: un negocio por usuario; el nombre va en el header |
| "Nuevo negocio" | Lo hace el Back Office |
| Configurar reservas en un sheet desde Agenda | Pantallas de Ajustes, desde el engranaje del header |
| Horario único de apertura y cierre | Horario semanal por día, descansos por profesional y excepciones |
| Agenda siempre visible | Solo si el negocio tiene reservas (`hasBooking`) |
| "Restablecer datos de ejemplo" en Resumen | Se elimina |
| Casilla sellada "perforada" en `background` | Círculo sólido `cardInk` con número en el color de categoría (decisión del 2026-10-02) |

## 2. Shell

**Header (`BusinessHeader`)**

- Izquierda: ícono de la categoría en un cuadro de 38 px con el color de categoría, el nombre del negocio en
  `headlineSmall` y debajo la categoría y la dirección en `bodyMedium` `muted`.
- Derecha: botón de engranaje de 38 × 38 (`settings`, borde `line`, fondo `surface`) que abre Ajustes.
- Sin campana ni selector de modo.

**Navegación**

| Pestaña | Ícono | Condición |
|---|---|---|
| Agenda | `calendar` | `hasBooking` |
| Sellos | `stamp` | Siempre |
| Programas | `list` | Siempre |
| Clientes | `users` | Siempre |
| Resumen | `chart-column` | Siempre |

- Teléfono: `LTabBar` inferior (4 o 5 pestañas).
- Tablet (≥ 840 px): `LNavigationRail` a la izquierda, fondo `surface` con borde derecho `line`, ícono 22 y
  etiqueta `labelSmall`. La activa va en `teal` con fondo `tealSoft` en una píldora de 56 × 32.
- La pestaña Sellos cambia el ícono `plus` del prototipo por `stamp`, porque `plus` se reserva para "crear".

## 3. Layout por ancho

| Ancho | Margen | Columnas | Sheets |
|---|---|---|---|
| < 600 px | 20 | 1 | Ancho completo |
| 600–839 px | 24 | 1, máximo 640 centrada | Máximo 560, centrados |
| ≥ 840 px | 24 | Lista + detalle (Agenda, Clientes) | Máximo 560, centrados |

En lista + detalle, la lista ocupa 360 px fijos y el detalle el resto, separados por 1 px en `line`.

## 4. Componentes nuevos

### Selector de días (`DayStrip`)

- Fila de 7 chips con scroll horizontal: "Hoy", "Mañana" y luego el día abreviado ("vie 3 oct").
- Cada chip muestra a la derecha la cantidad de reservas en 500 `muted`. Si es 0, no se muestra número.
- Un día cerrado por horario o excepción muestra el chip con borde punteado y la etiqueta "Cerrado".
- Seleccionado: igual que `LChip` seleccionado.

### Grupo de recurso en la agenda (`ResourceAgendaGroup`)

- `LList` con encabezado de grupo (nombre del recurso) y un `BookingTile` por reserva, ordenadas por hora.
- Sin reservas: un ítem "Libre todo el día" en `muted`.
- Recurso que no trabaja ese día: un ítem "No trabaja hoy" con el motivo de la excepción ("Vacaciones") y el
  ícono `calendar-off` en `muted`.
- Resumen sobre los grupos: "5 reservas · 2 por atender" en `bodyMedium` `muted`.

### `BookingTile` en modo negocio

Igual que en el documento base, con dos ajustes:

- El título es el **cliente**, no el negocio, y el subtítulo es el servicio con el profesional, o la cancha, más
  el valor.
- La columna de hora muestra el inicio en `titleSmall` y el fin en 12/500 `muted` ("3:00 / 3:30 p. m.").

### Buscador de cliente (`CustomerSearchField`)

- `LSearchField` con un placeholder como "Nombre, celular o código".
- Debajo, hasta 5 resultados en `LList`: nombre en el título, celular y código en el subtítulo y, a la derecha,
  el progreso del programa elegido (`RewardBadge` "4/10").
- Sin resultados: un ítem con "No encontramos a nadie con ese dato" y el botón fantasma "Registrar cliente nuevo".
- El cliente elegido queda fijo como un ítem con botón "Cambiar".

### Campo de código de premio (`RewardCodeField`)

- Una celda por carácter. El largo y el alfabeto entran por parámetro desde una sola fuente en `loyalty_core`:
  `RewardCode.length` (hoy **8**) y `RewardCode.alphabet` (hoy **`0–9 A–F`**, hexadecimal en mayúscula, que es lo que
  genera el backend: un `UUID` recortado a 8 en mayúsculas). **El formato definitivo está pendiente de decidir con la
  app cliente**; cuando cambie en el backend, es un cambio de una línea en `RewardCode`.
- Celdas de 44 × 52 con radio 12, fondo `surface` y borde `line`, en `titleMedium`.
- **Normalización** (`RewardCode.normalize`), igual al teclear de a un carácter o al pegar el código completo:
  minúsculas → mayúsculas; confusiones al dictar `O → 0`, `I` y `L → 1`; cualquier carácter fuera del alfabeto se
  ignora sin borrar lo demás. (Nota: el diseño original pedía excluir `O 0 I 1`, pero el backend sí emite `0` y `1`,
  así que se acepta el alfabeto real y solo se corrigen las letras que se confunden.)
- La celda activa lleva borde de 3 px en `sun`.
- Al completar todas las celdas se valida solo; con error, todas las celdas se marcan con borde `rose` y el mensaje va
  debajo en 13 `rose`.

### Premio desbloqueado (`RewardUnlockedSheet`)

- `LSheet` con título "¡Tarjeta completa!", el nombre del cliente y el premio en 15/600.
- Un bloque de código del documento base por cada premio desbloqueado.
- Botones: "Guardar para después" (fantasma) y "Canjear ahora" (positivo), en fila con flex 1 cada uno.
- Al abrirse, la `StampCard` de la vista previa reproduce la animación de sello en la casilla del premio.

### Editor de horario semanal (`WeeklyScheduleEditor`)

Pantalla completa dentro de Ajustes. Se usa para el horario del negocio y para el de cada profesional o cancha.

- Una fila por día (lunes a domingo) en `LList`, con el nombre del día y un `LSwitch` de abierto o cerrado.
- Día abierto: debajo, sus franjas como chips "8:00 a. m. – 12:00 p. m." con una "×" para quitarlas, y el botón
  pequeño "Agregar franja". Los huecos entre franjas son los descansos.
- Al tocar una franja se abre un sheet con dos selectores de hora (inicio y fin), en pasos de 30 min.
- Acción "Copiar a todos los días" en el menú de la fila.
- En un recurso, se muestra arriba la nota (`LNote` `tealSoft`): "El horario se recorta al del negocio". Las
  franjas que caen fuera del horario del negocio se pintan con borde punteado `rose`.
- Validación en vivo: franjas que se solapan o con fin menor que inicio se marcan con `rose` y deshabilitan
  "Guardar".
- Recurso sin horario propio: la pantalla dice "Usa el horario del negocio" y ofrece "Personalizar".

### Excepciones (`ScheduleExceptionTile` y formulario)

- Ítem de lista: fechas ("24 – 26 dic" o "8 dic") en `titleSmall`, motivo en el título, alcance en el subtítulo
  ("Todo el negocio" o "Julián") y la franja si es parcial ("Cerrado de 2:00 a 6:00 p. m.").
- Las excepciones ya pasadas van en una sección "Anteriores", colapsada.
- Formulario en sheet:
  - alcance: todo el negocio o un recurso (chips)
  - fecha desde y hasta
  - día completo o parcial (switch); si es parcial, hora de inicio y fin
  - motivo, con chips de sugerencia: Festivo, Vacaciones, Día libre, Mantenimiento
- Si la excepción choca con reservas confirmadas, se avisa cuántas antes de guardar: "Hay 3 reservas en esas
  fechas. Revisa la agenda para avisar a los clientes."

### Servicios de un profesional (`ResourceServicesSelector`)

- Chips de selección múltiple con todos los servicios del negocio. Los elegidos van en el estilo seleccionado.
- Si no se elige ninguno, una nota `sunSoft`: "Sin servicios, este profesional no aparecerá al reservar."

### Ajustes (`SettingsSection`)

- Grupos de `LList` con título de sección encima.
- Cada ítem lleva un ícono en cuadro, el título, un resumen del valor actual en el subtítulo
  ("Lun a sáb, 8:00 a. m. – 7:00 p. m.") y un chevron.
- Grupos: Negocio (Información, Horario), Reservas (Reservas, Profesionales/Canchas, Servicios, Excepciones) y
  Cuenta (Correo, PIN, Tema, Cerrar sesión). Cerrar sesión va en `rose`, sin chevron.

### Barra de guardado (`LSaveBar`)

- Fija abajo en editores de pantalla completa, con fondo `background` y borde superior `line`.
- "Guardar" en botón positivo de ancho completo, deshabilitado si no hay cambios o hay errores.
- Si hay cambios sin guardar, encima del botón aparece "Cambios sin guardar" en 13 `muted`.

### Diálogo de confirmación (`LConfirmDialog`)

- Fondo `background`, radio 22 y padding 20.
- Título en `titleMedium`, explicación en `bodyLarge` y, si avisa al cliente, una línea `muted` que lo dice.
- Botones: "Volver" (fantasma) y la acción, en peligro si destruye ("Cancelar reserva", "Eliminar") o en
  positivo si confirma ("Confirmar abono").

## 5. Íconos adicionales

| Uso | Ícono Lucide |
|---|---|
| Pestaña Sellos | `stamp` |
| Ajustes | `settings` |
| Recurso no disponible | `calendar-off` |
| Horario | `clock` |
| Excepciones | `calendar-x` |
| Servicios | `scissors` (peluquería) o `list` |
| Mesas | `armchair` |
| Cerrar sesión | `log-out` |

## 6. Contenido

- Las acciones que afectan al cliente lo nombran: "Se le avisará a Laura", "Camila lleva 2 inasistencias".
- Los resúmenes de ajustes dicen el valor actual, no la instrucción: "Recordatorio 24 horas antes", no
  "Configura el recordatorio".
- Los errores de horario explican el conflicto: "Esta franja termina antes de empezar", "Se cruza con 10:00 a. m. –
  12:00 p. m."
