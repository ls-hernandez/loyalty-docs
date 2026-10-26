# Progreso — Frontend Mobile Negocios

Fecha: 2026-10-08 (etapa inicial), actualizado 2026-10-26 (Etapa 9 — Agenda Fase C).

## Estado del proyecto (2026-10-26)

Cruce de `git` con `backend-changes-business-app.md`, `requirements-frontend-mobile-business.md` y
este documento. **Backend `development`:** sin ramas sin unir (Fase C ya integrada; las ramas ya
unidas se borraron). **Mobile `development`:** 1 rama sin unir (Etapa 9 — Agenda Fase C, pendiente de
aprobación). La Etapa 2 (PIN + Tema) ya se unió. Las ramas ya unidas se borraron (`git branch -d`);
los nombres se dejan como referencia histórica.

| Tarea | Repo | Estado | Rama |
|---|---|---|---|
| **Fase A** — app de negocios (premio al atender, capacidades + zona horaria, `/stats`, rol OWNER, JSON en 401/403) | backend | ✅ Hecho y unido | `feature/business-app-phase-a` |
| **Fase B** — flujo de mostrador (buscar/registrar/detalle de clientes, `GET /slots`, `GET /rewards/{code}`, `/stamps` por celular·código·id) | backend | ✅ Hecho y unido | `feature/business-app-phase-b` |
| Regla de tiempo **"No asistió"/"Atendida"** (`noShowGraceMin`, `*AllowedFrom`, `Clock`) | backend | ✅ Hecho y unido | `fix/booking-status-timing` |
| **Fix contrato de clientes** (`GET /customers` con y sin `?q=` = mismo DTO; no-shows por lote) | backend | ✅ Hecho y unido | `fix/customer-list-contract` |
| **Fix reservas con strategy (canchas)** — `serviceId` opcional; duración/precio del `BookingPlan` (sirve para COURT) | backend | ✅ Hecho y unido | `fix/business-booking-strategy` |
| **Spotless** (formato) | backend | ✅ Hecho y unido | `chore/spotless-format` |
| **Fase C** — agenda completa (`workingIntervals`/`unavailableReason`, `/agenda/summary`, conteo de reservas por cliente, stats "a punto de completar" y "avisos enviados") | backend | ✅ Hecho y unido | `feature/business-app-phase-c` |
| **Fase D** — antes de lanzar (idempotencia, recuperar contraseña del dueño, refresh token, capacidad al crear mesa) | backend | ⬜ Pendiente | — |
| **Perfil local de `application.yaml`** | backend | ❌ Descartado (se implementó y se revirtió por decisión; los ajustes locales quedan solo en el working tree, sin versionar) | — |
| **Base** — Login/sesión + shell y navegación | mobile | ✅ Hecho y unido | (en `development`) |
| **Etapa 3** — Programas | mobile | ✅ Hecho y unido | (en `development`) |
| **Etapa 4a/4b** — Ajustes (horario, reservas, recursos, servicios, excepciones) | mobile | ✅ Hecho y unido | (en `development`) |
| **Etapa 5** — Negocio real + Información | mobile | ✅ Hecho y unido | (en `development`) |
| **Etapa 6** — Agenda | mobile | ✅ Hecho y unido | (en `development`) |
| **"No asistió"/"Atendida"** por tiempo (front: gating por `*AllowedFrom`, espera de inasistencia) | mobile | ✅ Hecho y unido | (en `development`) |
| **Etapa 8a** — Sellos y canje | mobile | ✅ Hecho y unido | `feature/stage-8a-stamps` |
| **Etapa 8b** — Clientes | mobile | ✅ Hecho y unido | `feature/stage-8b-customers` |
| **Etapa 8c** — Nueva reserva (incluye la simplificación de la 8b y las franjas de hora) | mobile | ✅ Hecho y unido | `feature/stage-8c-new-booking` |
| **Etapa 7** — Resumen | mobile | ✅ Hecho y unido | `feature/stage-7-stats` |
| **Habilitar reservas de cancha en el front** (`kCourtBookingEnabled = true`) | mobile | ✅ Hecho y unido | `fix/court-booking-enabled` |
| **`dev.example.json` + perfil local del front** (gitignore de `env/dev.json` y `.vscode/launch.json`, plantillas y README) | mobile | ✅ Hecho y unido | `chore/env-example` |
| **Etapa 2** — PIN + Tema | mobile | ✅ Hecho y unido | `feature/stage-2-pin-theme` |
| **Etapa 9** — Agenda Fase C (consumo del contrato) | mobile | 🟡 En rama sin unir | `feature/stage-9-agenda-phase-c` |

> Nota: la cancha (COURT) ya es **creable** en backend y front (bandera `kCourtBookingEnabled` en
> `true`).

## Etapa 9 — Agenda Fase C (2026-10-26)

Rama `feature/stage-9-agenda-phase-c` desde `development`. Consume los contratos nuevos de la Fase C
(`docs/api-business-audit.md`) en la agenda, clientes y resumen. Un commit por paso; `flutter
analyze` sin warnings y suite en verde en cada uno.

- **Modelos y repositorio.** `Booking` trae `customerName`/`serviceName` desde `GET /agenda`;
  `AgendaGroup`/`ResourceAgenda` suman `workingIntervals` y `unavailableReason`. `getAgendaDay` hace
  **una sola** llamada a `/agenda` (se quitaron las llamadas extra a `/customers` y `/booking-config`
  que solo servían para resolver nombres; el precio ya venía en la reserva). Nuevo modelo
  `DaySummary` + `getAgendaSummary` para `GET /agenda/summary?from=&to=`. `CustomerCard`/
  `CustomerListItem` suman `bookingCount`; `BusinessStats` suma `customersNearReward` y
  `notificationsSent`.
- **Selector de 7 días.** `AgendaSummaryController` carga el conteo y la bandera de cerrado de los 7
  días visibles en **una** llamada, indexado por fecha, y observa `businessDataChangesProvider`: se
  recarga al crear/cancelar/cambiar el estado de una reserva y al guardar el horario del negocio, el
  de un recurso o una excepción (se agregó `bump()` en esos guardados). `DayStripItem` gana
  `closedLabel`: un día cerrado muestra chip punteado con "Cerrado" en vez del conteo; los abiertos
  muestran el conteo (oculto si 0).
- **ResourceAgendaGroup.** Se extrajo el grupo inline a un widget público y testeable con tres
  estados: no trabaja hoy (`calendar-off` + "No trabaja hoy" + motivo; `OUT_OF_SCHEDULE` → "Fuera de
  su horario", otro valor → el texto de la excepción), libre todo el día, y la lista de reservas. **No
  se pintan las franjas libres** (el diseño §4 no las pide). El resumen "N reservas · M por atender"
  sigue encima de los grupos.
- **BookingTile.** El subtítulo se arma con un helper parametrizado (`bookingSubtitle`, un parámetro y
  no dos widgets): dentro de un grupo no repite el profesional ("Servicio · $valor", o solo el valor
  en canchas); fuera de un grupo lo incluye ("Servicio con Profesional · $valor" o "Cancha · $valor").
- **Clientes.** Cada ítem de la lista muestra "N reservas" (`bookingCount`), oculto si 0.
- **Resumen.** Tarjeta "A punto de completar" con `customersNearReward`.

**Pendiente de backend (no se muestra todavía):**
- **`notificationsSent`** — el backend devuelve `0` fijo (aún sin tabla de notificaciones, Fase D). El
  Resumen **no** muestra esa tarjeta hasta que exista la persistencia.
- **`bookingCount` en el detalle del cliente** — `GET /customers/{custId}` **no** trae `bookingCount`
  en el contrato. Para no agregar llamadas extra, el conteo se muestra **solo en la lista**; queda
  pendiente que el backend lo exponga en el detalle.

## Etapa 2 — PIN + Tema (2026-10-19)

Rama `feature/stage-2-pin-theme` desde `development`. PIN local opcional y tema, según requirements
§3 y guidelines §6 (que se actualizó con la decisión del `AppLockController` aparte).

- **PIN** (4 dígitos): se **ofrece una sola vez** tras el primer login (con "Ahora no"; si se omite,
  se marca y queda en Ajustes → Cuenta → "Crear PIN"). Se rechazan triviales (iguales o secuencia).
  Se guarda **hasheado** (PBKDF2-HMAC-SHA256, sal aleatoria, cálculo en `Isolate`, comparación en
  tiempo constante) en `flutter_secure_storage`; las iteraciones viajan con el hash.
- **Bloqueo** (`AppLockController`, aparte de la sesión): pide el PIN **al abrir la app** y **tras 5
  min** en segundo plano. Intentos fallidos: espera de **30 s** con cuenta regresiva en el 3.º y 4.º;
  en el **5.º**, `signOut` completo (borra token, PIN, intentos, espera y la marca) y vuelta al login.
  Al **desbloquear vuelve a la ruta** donde estaba. "¿Olvidaste tu PIN?" → `signOut`. "Bloquear ahora"
  y "Cambiar PIN" (pide el actual) en Ajustes → Cuenta.
- **Tema**: Ajustes → Cuenta → Sistema/Claro/Oscuro (por defecto Sistema), persistido y aplicado sin
  reiniciar (`MaterialApp.themeMode`).
- **Core:** `PinHasher`/`PinPolicy`/`PinStorage` (dependencia nueva `cryptography`). **DS:** `PinPad`
  (reutilizado en desbloqueo, alta y cambio). El router gatea `SignedIn`+bloqueado → `/unlock` y
  oferta → `/setup-pin`.
- **Tests:** hash/verify + vector conocido de PBKDF2 + rechazo de triviales; contador de intentos y
  bloqueo por segundo plano (4:59 vs 5:00) con reloj fijo; cambio de tema; y widgets de desbloqueo,
  alta y Ajustes → Cuenta. Suite: **185** en verde. Sin merge a `development`.

## Etapa 7 — Resumen (2026-10-15)

Rama `feature/stage-7-stats` desde `development`. Pestaña **Resumen** (`/summary`) con las
métricas del negocio (requirements §8), según `GET /stats` de `api-business-audit.md`.

- **Tarjetas de estadística** en cuadrícula de **2 columnas (4 desde 840 px)**: Clientes con
  tarjeta, Sellos entregados, Premios por canjear y Premios canjeados; **si `hasBooking`**:
  Reservas próximos 7 días e **Inasistencias** (`noShowPercent` %, tal cual lo da el backend).
- **No** se muestran "A punto de completar" ni "Avisos enviados" (los expone el backend en la
  Fase C): no se muestran ni se calculan.
- Recarga al entrar a la pestaña, **pull to refresh** y al recibir `businessDataChangesProvider`
  (p. ej. tras sumar un sello). Estados de carga y error con reintentar.
- **Design system:** `LStatCard` (valor grande + etiqueta `muted`, sin texto propio).
- **Datos:** `features/summary` — `BusinessStats` (`GET /stats`), `StatsRepository`,
  `SummaryController`.
- **Tests:** DTO con el fixture de la auditoría, repositorio, Notifier (carga y recarga con
  `businessDataChanges`) y widget tests (con y sin `hasBooking`, y layout de tablet con 4
  columnas) + `LStatCard` en el design system. `analyze` limpio; app **154** y design system
  **97** en verde. Sin merge a `development`.

## Etapa 8c — Nueva reserva (2026-10-14)

Rama `feature/stage-8c-new-booking` desde `development`. Alta de reservas desde el negocio
(requirements §4), con el contrato de `GET /slots` y `POST /bookings` de la Fase B.

**Ajuste previo de la 8b:** el backend unificó `GET /customers` (con y sin `?q=`) a un solo DTO
(con `noShows` y `memberships` con `programName`/`goal`). Se eliminó el cruce con los programas en el
repositorio de clientes; `CustomerCard` trae `noShows` y las **inasistencias se resaltan también en
los resultados de búsqueda**.

- **Sheet (`LSheet`)** abierto desde la Agenda (botón "Nueva reserva", ya habilitado) y desde el
  detalle de un cliente (con ese cliente preelegido). Solo con `hasBooking`.
- En una pantalla: (1) **cliente** con `CustomerSearchField` reutilizado y "Registrar cliente nuevo"
  (sheet de registro compartido con Sellos); (2) **peluquería (STAFF):** servicio (con duración y
  precio) + profesional con "Cualquiera" (solo quienes hacen ese servicio); **cancha (COURT):** la
  cancha; (3) **día** con `DayStrip` de 7 días (zona del negocio) y **hora** en una cuadrícula con
  las horas libres de `GET /slots`. El front no calcula disponibilidad.
- Nota **sunSoft** si el backend pide abono ("<nombre> tiene N inasistencias. La reserva quedará
  pendiente de abono.") y nota **tealSoft** con los programas que suman sello (`stampProgramNames`).
- "Crear reserva" deshabilitado hasta tener cliente y hora, y mientras corre la petición. `409`
  (`SLOT_NOT_AVAILABLE`/`CUSTOMER_BOOKING_OVERLAP`) → mensaje y recarga de horas. Al crear: cierra el
  sheet, salta la Agenda al día de la reserva, toast ("Reserva creada. Se le avisó a <nombre>." o
  "pendiente de abono") y emite `businessDataChangesProvider`. Carga/empty ("No quedan horas libres
  ese día…")/error de horas con reintentar.

**Gap de backend — reservas de cancha (COURT).** `POST /bookings` exige `serviceId` (y deriva
duración/precio de un `OfferedService`), pero las canchas no tienen servicio: hoy **no se pueden
crear**. Se construyó la UI de cancha completa (selector + horas + precio día/noche, con tests), pero
"Crear reserva" queda **deshabilitado solo para COURT** con la nota "Las reservas de cancha estarán
disponibles pronto." La condición está en **un solo lugar** (`kCourtBookingEnabled` en
`bookings_repository.dart`): cuando el backend acepte canchas se habilita con un commit. El request de
cancha ya se arma **sin `serviceId`**. Pendiente: fix en el backend (`serviceId` opcional para canchas,
duración/precio del `CourtBookingStrategy`).

- **Datos:** `features/bookings` — `BookingAvailability` (`GET /slots`) y `BookingsRepository`
  (`slots`, `create`; `create` omite `serviceId` cuando es cancha).
- **Controller:** `NewBookingController` (cliente, servicio/profesional o cancha, día, hora; carga
  opciones y horas; crea).
- **Refactor:** `showRegisterCustomer` subido del feature Sellos a `customers` para compartirlo.
- **Tests:** DTOs de slots/reserva con fixtures de la auditoría, repositorio, Notifier (cambiar
  servicio/profesional/día/hora, conflicto 409) y widget tests del sheet (peluquería y cancha, con y
  sin abono, con cliente preseleccionado). Suite: **160 en verde**. Sin merge a `development`.

## Etapa 8b — Clientes (2026-10-14)

Rama `feature/stage-8b-customers` desde `development` (que ya incluye la 8a, mergeada con
`--no-ff`). Pestaña **Clientes** (`/customers`): lista con búsqueda y detalle del cliente. Usa
los endpoints de la Fase B tal como están en `api-business-audit.md`.

- **Lista:** nombre, celular, código, progreso por programa ("Corte gratis: 4/10") e
  **inasistencias resaltadas** (badge rose). Sin texto de búsqueda muestra la **lista completa**
  del negocio (`GET /customers`, con inasistencias); al escribir, busca en el backend con rebote de
  300 ms (`GET /customers?q=`) y **respeta la regla de privacidad del backend** (por nombre solo
  clientes del negocio; por celular completo o código, toda la plataforma). Orden: el que devuelve
  el backend. No se muestra el **conteo de reservas** por cliente (Fase C). Estados de carga, vacío
  ("Todavía no hay clientes…" / "Nadie coincide con esa búsqueda") y error con reintentar.
- **Detalle** (al tocar un cliente): encabezado (nombre, celular, código); **"Sumar sello"** abre
  Sellos con ese cliente ya elegido **por ruta** (`selectedCustomerProvider` + `context.go('/stamps')`,
  sin acoplar las features); **"Nueva reserva"** deshabilitado hasta la 8c y solo si `hasBooking`.
  Nota **rose** con las inasistencias y **"Perdonar"** con `LConfirmDialog`
  (`DELETE /customers/{id}/no-shows`). Una `StampCard` por tarjeta con el color de la categoría.
  Premios **disponibles / canjeados / vencidos** con su badge (estado calculado por el backend).
  Reservas recientes si el detalle las trae.
- **Tablet (≥ 840 px):** lista (360 px) y detalle lado a lado, separados por 1 px `line`. En
  teléfono el detalle se abre en ruta aparte (`/customers/:id`).
- Al **perdonar** se emite `businessDataChangesProvider`; **la lista y el detalle se recargan** con
  esa señal (p. ej. tras sumar un sello en Sellos).

**Discrepancia doc↔backend (anotada).** La lista que pide el doc combina progreso con nombre/meta de
programa **y** inasistencias, pero el backend lo parte: `GET /customers?q=` trae `programName`/`goal`
pero **no** `noShows`; `GET /customers` (sin `q`) trae `noShows` pero las membresías vienen sin
nombre ni meta. Solución fiel: la **lista completa** une el conteo con el nombre/meta desde la lista
de programas del negocio y muestra inasistencias; en **resultados de búsqueda** las inasistencias no
se resaltan porque ese DTO no las trae (el conteo de reservas por cliente sigue siendo Fase C).

- **Datos:** `features/customers` gana `CustomersRepository.list` (lista completa) y `clearNoShows`;
  modelos `CustomerSummary`, `CustomerListItem`/`CustomerProgress` (unifican lista y búsqueda).
- **Controllers:** `customersListController` (lista/búsqueda con rebote, recarga con
  `businessDataChanges`), `customerDetailController` (detalle del cliente abierto + perdonar),
  `selectedCustomerIdProvider`.
- **Tests:** repositorio (lista, `clearNoShows`), Notifiers (lista, búsqueda, detalle, perdonar),
  detalle (tarjeta, premio, perdonar, "sumar sello" navega) y lista + layout de tablet. Suite:
  **147 en verde**. Sin merge a `development` (pendiente de aprobación).

## Etapa 8a — Sellos y canje (2026-10-14)

Rama `feature/stage-8a-stamps` desde `development`. Pestaña **Sellos** (`/stamps`) con dos
modos segmentados: **Sumar sello** y **Canjear premio**. Usa los endpoints de la Fase B del
backend tal como están documentados en `api-business-audit.md`.

- **Sumar sello:** `CustomerSearchField` con búsqueda por nombre/celular/código (rebote 300 ms;
  la **regla de privacidad la aplica el backend**, el front no filtra). Sin resultados, se ofrece
  "Registrar cliente nuevo" (nombre + celular de 10 díg., `POST /customers`; si el celular ya
  existía, el backend lo devuelve con `200` y se selecciona sin error). Cliente fijo con "Cambiar";
  selector de programa si hay más de uno activo. Vista previa de `StampCard` con el progreso real
  del cliente y el color de la categoría; al sumar, **anima la casilla nueva** y muestra el toast
  "Sello sumado: N de M". Si la respuesta trae premio, abre `RewardUnlockedSheet` (reutiliza el del
  sheet de Agenda). Botón deshabilitado mientras la petición está en curso. Sin programas activos,
  invita a crear uno (acceso a Programas).
- **Canjear premio:** `RewardCodeField` valida con `GET /rewards/{code}` al completar las celdas y
  muestra cliente, premio y vencimiento antes de confirmar (`POST /rewards/redeem`). Cada `reason`
  de error (`REWARD_NOT_FOUND`, `REWARD_WRONG_BUSINESS`, `REWARD_ALREADY_REDEEMED`, `REWARD_EXPIRED`)
  tiene su mensaje.
- Tras sumar o canjear se emite `businessDataChangesProvider` (Clientes, Resumen y Agenda se
  refrescan cuando se reconstruyan).

**Formato del código de premio — pendiente de decidir con la app cliente.** Se implementó con el
formato **real de hoy**: 8 celdas, alfabeto hexadecimal `0–9 A–F` (lo que genera el backend: un
`UUID` recortado a 8 en mayúsculas). El largo y el alfabeto viven en una sola fuente
(`loyalty_core/RewardCode.length` y `RewardCode.alphabet`), así un cambio de formato en el backend
es de una línea. La normalización corrige confusiones al dictar (`O → 0`, `I`/`L → 1`), pasa a
mayúsculas e ignora lo que no esté en el alfabeto, tanto al teclear como al pegar. (El diseño pedía
excluir `O 0 I 1`, pero el backend sí emite `0` y `1`; se documentó la discrepancia en
`design-system-business.md` y `requirements-frontend-mobile-business.md` §5.)

- **Design system (nuevos):** `CustomerSearchField`, `RewardCodeField`; `StampCard` gana animación
  opcional (`animate`) de la casilla recién sellada. Sin textos ni red propios.
- **Datos:** `features/customers` (search/register/detail), `features/stamps` (addStamp),
  `features/rewards` (preview/redeem).
- **Tests:** `RewardCode` (core), `CustomerSearchField`/`RewardCodeField`/`StampCard` (design system,
  claro y oscuro), DTOs con los fixtures de la auditoría, repositorios, Notifiers y pantalla
  (buscar, registrar, sumar con y sin premio, canjear y error). Suite: **133 en verde**.

## Agenda — acciones por tiempo + espera de inasistencia (2026-10-13)

Soporte del cambio del backend (rama `fix/booking-status-timing`): la reserva trae `noShowAllowedFrom` y
`attendedAllowedFrom`, y booking-config trae `noShowGraceMin`.

- **ErrorReason** (core): `noShowTooEarly`, `attendedTooEarly` + `fromWire`; mensajes claros en los `.arb`
  (es/en) por si el backend devuelve el `409`.
- **Reloj del negocio:** `BusinessClock.nowNaive()` (hora de pared de la zona, sin offset) para comparar
  contra los `LocalDateTime` del backend sin mezclar husos.
- **DTO** `Booking`: `noShowAllowedFrom`/`attendedAllowedFrom` (parse naive; tolera ausentes).
- **Design system** `BookingTile`: `BookingAction` gana `enabled` + `note`; el botón se deshabilita y debajo
  muestra la nota en `muted`.
- **Agenda:** "Atendida" y "No asistió" se deshabilitan mientras `now < ...AllowedFrom`, con la nota
  "Disponible desde las 3:10 p. m." (`Formatters.time`). No se recalcula la regla; solo se usan esas fechas.
  Con la recarga de cada 60 s se habilitan solas.
- **Ajustes → Reservas:** campo "Tiempo de espera antes de marcar inasistencia" (0–120 min, `noShowGraceMin`),
  round-trip en el `PUT`.
- **Tests:** DTO (fechas nuevas, ausentes), `BookingTile` (acción deshabilitada con nota), pantalla de Agenda
  con reloj fijo (deshabilitada con nota antes de la hora; habilitada después), mensajes de error de los dos
  reasons y round-trip de `noShowGraceMin`. `analyze` limpio; suites en verde.

## Etapa 6 — Agenda (2026-10-13)

## Etapa 6 — Agenda (2026-10-13)

- **Pestaña Agenda** (solo `hasBooking`): reservas del día agrupadas por recurso, con selector de días,
  resumen, acciones por estado y el sheet de premio desbloqueado. Reemplaza el placeholder de la rama.
- **Design system (nuevos):** `DayStrip` (chips de días, conteo opcional —hoy `null`, Fase C—),
  `BookingTile` modo negocio (hora inicio/fin, cliente, servicio/cancha + valor, estado, "Sello sumado" y
  acciones deshabilitables con `busy`) y `RewardUnlockedSheet` ("¡Tarjeta completa!", un bloque de código
  por premio, "Guardar para después"/"Canjear ahora"; devuelve el código a canjear o `null`).
- **Feature** (`features/agenda/`): `Booking`/`BookingStatus`/`UnlockedReward`/`BookingStatusResult`
  (`PATCH …/status` trae `stampedMemberships` + `unlockedRewards`). El **repositorio** (no la UI) cruza
  `GET /agenda` con `GET /customers` (id→nombre; faltante → vacío, la UI muestra "Cliente") y
  `GET /booking-config` (servicios id→nombre); ordena por hora. Controller `AsyncNotifier` que observa el
  día seleccionado (`selectedAgendaDateProvider`, arranca en "hoy" de la zona del negocio); acciones
  `changeStatus`/`redeem` con set de reservas en curso (`agendaBusyProvider`), recarga tras cada acción y
  emite `businessDataChangesProvider` (nuevo, `core/sync/`) para Clientes y Resumen.
- **Acciones por estado:** `CONFIRMED` → Atendida / No asistió / Cancelar; `PENDING_DEPOSIT` → Confirmar
  abono / Cancelar; resto sin acciones. Atendida: si hay `unlockedRewards`, abre el sheet (Canjear ahora =
  `POST /rewards/redeem?code=`); si no, toast "Sello sumado a {nombre}" o "marcada como atendida". No
  asistió, Cancelar y Confirmar abono confirman con `LConfirmDialog`; Cancelar y Confirmar abono dicen "Se
  le avisará a {nombre}". Botón deshabilitado mientras corre la petición (no hay idempotencia).
- **Recarga:** pull-to-refresh, cada 60 s mientras la pantalla está montada y al volver a primer plano
  (lifecycle). Tablet ≥ 840 px: columnas por recurso lado a lado. Estados de carga, vacío ("Sin reservas
  este día") y error con reintentar.
- **Diferido (Fase C / Fase B):** conteo de reservas por día y "No trabaja hoy" en el selector; toast del
  umbral de abono al marcar No asistió (la respuesta no trae el conteo); botón "Nueva reserva"
  deshabilitado; animación de sellado del sheet.
- **Tests:** DTOs con fixtures de la auditoría (`agenda.json`, `booking_status_attended.json`), repositorio
  (cruce de nombres, cliente faltante, `PATCH`/redeem), controller (cambio de día, acción + recarga + bump,
  busy, redeem) y pantalla (vacío, datos, Atendida con y sin premio). DS: `DayStrip`, `BookingTile` y
  `RewardUnlockedSheet` claro/oscuro.

## Correcciones y ajustes (2026-10-13)

- **Fix — Horario del negocio no persistía al reingresar:** `businessHoursControllerProvider` no es
  `autoDispose`, así que conservaba en caché la semana sembrada (8–20) de la primera carga; el `PUT`
  guardaba en base pero el controller no se refrescaba y al reentrar se veían los valores por defecto.
  Ahora `save()` hace `invalidateSelf()` + `await future` para releer el horario persistido. Test de
  regresión con repo que simula persistencia.
- **Fix — Subtítulos de Ajustes desactualizados:** las filas Profesionales/Servicios/Excepciones leían
  un `...SummaryProvider` separado del controller de cada feature; al agregar/quitar se actualizaba el
  controller pero no ese provider de resumen, que seguía mostrando el conteo vacío ("Ninguno"/"Sin
  servicios"). Cada controller ahora invalida su summary al mutar. Test de regresión del resumen.
- **Ajuste — "Lo que ofrece" sugiere Servicios:** el campo sigue siendo texto libre (`offers`,
  independiente de Servicios, según §9), pero ahora muestra los nombres de los Servicios como chips
  sugeridos para agregarlos de un toque (excluye los ya agregados). No cambia el contrato del backend.
  Nuevo parámetro opcional `suggestions`/`onAddSuggestion` en `OffersField`.

## Etapa 4b — Profesionales/Canchas/Mesas, Servicios y Excepciones (2026-10-12)

- **Recursos** (`features/resources/`, solo `hasResources`): lista + alta (`POST ?name=`) y detalle por
  recurso. El detalle edita el **horario propio** con el `WeeklyScheduleEditor` o lo deja **heredado** del
  negocio; al personalizar, siembra la semana desde el horario del negocio y marca con borde punteado las
  franjas que caen **fuera** del horario del negocio (lógica `LScheduleLogic.isOutside`, pura y testeada).
  Para `STAFF`, selector de **servicios** del profesional (chips) con aviso si queda vacío. Quitar con
  confirmación; si el backend responde `RESOURCE_HAS_PENDING_BOOKINGS`, segundo diálogo con "Ir a la Agenda".
- **Servicios** (`features/services/`, solo `hasServices`): la lista sale de `booking-config.services`
  (no hay `GET /services`); alta (`POST ?name&durationMin&price`) y baja. El subtítulo muestra duración,
  precio y **nº de profesionales** que lo ofrecen (conteo N+1 sobre el `STAFF` activo). `ServiceItem` vive
  en `core/business/` para compartirlo entre Servicios y el detalle del recurso.
- **Excepciones** (`features/schedule_exceptions/`, solo `hasBooking`): `GET/POST/DELETE
  /schedule-exceptions`; alcance "todo el negocio" o un recurso, rango de fechas, día completo o parcial y
  motivo (con sugerencias). La lista separa **próximas** de **anteriores** usando "hoy" en la zona del
  negocio (`businessTodayProvider`).
- **Ajustes:** nuevas filas en "Negocio" —Profesionales/Canchas/Mesas (si `hasResources`), Servicios
  (si `hasServices`) y Excepciones (si `hasBooking`)— cada una con su resumen (nº de recursos/servicios y
  nº de excepciones próximas).
- **Lecturas transversales:** Servicios→recursos (conteo) y Excepciones/detalle→recursos y horario del
  negocio se resuelven vía providers de otras features (compromiso pragmático de capas, anotado en código).
- **Tests:** DTOs con fixtures (`resources.json`, `resource_schedule.json`, `schedule_exception.json`,
  servicios por `booking-config`), repos con adapter Dio falso (incl. `POST` por query y el `409`
  `RESOURCE_HAS_PENDING_BOOKINGS`) y widget de cada pantalla (lista/alta/baja y apertura de formularios).
  `analyze` sin warnings; **workspace en verde** (core 22, design system 61, business 89).

## Etapa 4a — Horario del negocio y Reservas (2026-10-10)

- **Design system:** `WeeklyScheduleEditor` (controlado, sin textos: etiquetas y formato de hora por
  parámetro) con filas por día (switch abierto/cerrado), franjas como chips editables, "Agregar franja",
  "Copiar a todos los días" y sheet de rango en pasos de 30 min; validación en vivo (franja inválida o
  solapada) vía `LScheduleLogic` (puro, testeado). Nuevo `LNote` (info/aviso).
- **Horario** (`features/business_hours/`): `GET/PUT /business-hours` con `Weekday` (DayOfWeek) y minutos
  desde medianoche; conversión semana↔`{day,startMin,endMin}` en el DTO. Si `defaultHours:true`, muestra
  `LNote` y **siembra** los 7 días con `openHour/closeHour` de `booking-config` (GET extra). Pantalla
  completa con `LSaveBar`, confirmación al salir y estados.
- **Reservas** (`features/booking_config/`, solo `hasBooking`): `GET/PUT /booking-config`; la conversión
  horas↔minutos vive en el DTO. Recordatorio, abono tras N (0 = nunca), programas vinculados como chips
  multi-selección (pausados marcados), y —solo si `resourceKind == COURT`— tarifa diurna/nocturna y hora
  de inicio nocturna; apertura/cierre solo si el negocio no tiene horario propio. Al guardar invalida
  `programsControllerProvider` (marca "Suma con reservas"). Round-trip de todos los campos en el PUT.
- **Ajustes:** filas "Horario" (siempre) y "Reservas" (si `hasBooking`) con **resumen del valor actual**
  en el subtítulo ("Desde 8:00 a. m.", "Recordatorio 24 h · abono tras 2").
- **Tests:** lógica de franjas (unit), `WeeklyScheduleEditor` claro/oscuro, DTOs con fixtures
  (`business_hours.json`, `booking_config.json`), repos (GET/PUT con adapter Dio falso) y widget de cada
  pantalla (carga, guardado). `analyze` sin warnings; `flutter test` en verde.

## Etapa 3 — Programas (2026-10-10)

Pestaña **Programas** (`features/programs/`) contra `GET/POST/PUT/DELETE /programs` y
`PATCH /programs/{id}/status`:
- **Lista** con la regla ("Cada 10 visitas: Un corte gratis"), vigencia ("Vence en 30 días"), la marca
  **"Suma con reservas"** (de `linkedProgramIds` de `booking-config`, solo si `hasBooking`) y el
  **switch** de activo/pausado. Estados carga / vacío / error+Reintentar. El toggle actualiza la lista
  en memoria (sin parpadeo).
- **Crear/editar** en bottom sheet (`LSheet`): nombre, meta (2–30), unidad (chips, preseleccionada con
  `stampUnit` de las capacidades), tipo de premio (chips), valor (solo %/segundo/monto), texto para el
  cliente **autocompletado** hasta que el dueño lo edita, días de vigencia y reinicio. **Vista previa
  `StampCard` en vivo** con `businessType.cardColor` + ícono. Guardar deshabilitado mientras guarda o
  con errores.
- **Eliminar** con `LConfirmDialog` que avisa que se pierde el progreso y sugiere pausar. **Pausar** con
  `PATCH status`, sin borrar progreso.
- Capa: `Program` + enums `RewardType`/`ProgramStatus`; `ProgramsRepository` (CRUD + status +
  `linkedProgramIds`); `ProgramsController` (`AsyncNotifier`). Extensiones l10n `StampUnitL10n` (core) y
  `RewardTypeL10n` (feature). La rama `/programs` del shell ya monta `ProgramsScreen`.
- **Tests:** DTOs con fixture `program.json`, repositorio (CRUD/status/linked con adapter Dio falso),
  controller (carga + toggle + create + delete) y widget (lista con marca/estado vacío/toggle y el sheet
  de alta con validación de meta). `analyze` sin warnings; `flutter test` en verde en todo el workspace.

## Reconciliación con el documento base de diseño (2026-10-10)

Llegó `docs/Vuelve — Design System para Flutter.md`. Comparé y corregí los tokens de
`loyalty_design_system` para que manden los valores del documento:
- **Color:** los 12 semánticos + `cardInk` ya coincidían. Agregué `canvas` (claro `#E9EDE8` / oscuro
  `#0F1522`) y los fijos `onTeal`, `onSun`, `cardGlass`, `holeEmpty`, `scrim`.
- **Tipografía:** reescribí `buildTextTheme` exacto al doc (p. ej. `headlineMedium` 28/800, `displaySmall`
  34/800 +4, `bodyLarge`/`bodyMedium` en 400, `labelMedium` 600). `LButton` pequeño usa 700 por `copyWith`.
- **Radios:** agregué el de 20 px (stepper/glow).
- **Tema:** `colorScheme.surfaceTint` transparente y `navigationBarTheme` (sin indicador, activo `teal`).
- **Paleta de categorías** como tokens (`LCategoryColors`: peluquería `#F2B630`, cancha `#8FD3B6`,
  tienda `#F4A7B0`, restaurante `#A9C4F5`, café `#E3C29B`) + extensión `BusinessType` (`cardColor`,
  `icon`). Se usa en `StampCard` (ícono en cuadro `cardGlass`) y en el header; se eliminó el mapeo a
  teal/sun/rose/ink (`category_visuals.dart` borrado). El contenido sobre esos colores va en `cardInk`.
- **Excepciones acordadas:** la casilla sellada de `StampCard` se queda círculo `cardInk` + número
  (decisión 2026-10-02), no "perforada"; los componentes conservan el prefijo `L`.
- **Desviación pendiente:** la cancha debería usar un **SVG propio del balón** (doc); por ahora usa
  `LucideIcons.volleyball` como sustituto (falta agregar el asset + `flutter_svg`).
- **Herencia (`guidelines-frontend-mobile.md`):** ese archivo **sigue faltando** en `docs/`, así que la
  revisión se hizo contra la guía de negocios + el doc base. Ajustes hechos: sin literales de color
  fuera del DS; `business_info_screen.dart` se dividió (`business_info_fields.dart`) para bajar de 300
  líneas.

## Ajustes → Cuenta (2026-10-10)

Sección "Cuenta" en Ajustes (debajo de "Negocio"): correo de la sesión (solo lectura) y **Cerrar
sesión** en `rose` con ícono `log-out`. Pide confirmación (`LConfirmDialog`: "Para volver a entrar
necesitas tu correo y contraseña"); al confirmar guarda el correo para precargarlo, borra el token de
`TokenStorage`, invalida `businessContextProvider` y el redirect de go_router lleva a `/login`. Es solo
local (el backend no tiene logout de dueño). PIN y tema quedan para la Etapa 2.

## Etapa 5 — Negocio real + Información (2026-10-10)

- **Repositorio real:** `LocalBusinessCapabilitiesSource` eliminado. `RemoteBusinessRepository`
  (`core/business/`) llama `GET/PUT /api/business/{bizId}` detrás del mismo `BusinessRepository`
  y `businessContextProvider`; la UI no cambió. DTOs en `core/business/data/`
  (`BusinessDetailDto.fromJson`, `BusinessUpdateRequest`). `BusinessContext` ahora expone también
  `phone`, `description`, `offers`, `status`.
- **Zona horaria:** paquete `timezone` + `tz.initializeTimeZones()` en `main`. `businessClockProvider`
  y `businessTodayProvider` calculan "hoy" con `businessContext.timeZone` (hoy `America/Bogota`), no
  con la del dispositivo.
- **Design system:** nuevo `LSaveBar` (sin textos) con test claro/oscuro.
- **Ajustes → Información del negocio** (`features/settings/`): editor de pantalla completa con
  nombre, dirección, teléfono, descripción y "lo que ofrece" (chips editables), categoría en solo
  lectura; **validaciones iguales al backend** (name ≤120 obligatorio, address ≤200, phone 10 díg.,
  description ≤500, offers ≤10×≤40); `LSaveBar` con aviso "Cambios sin guardar"; confirmación al
  salir con cambios (`PopScope` + `LConfirmDialog`); estados carga/error+reintentar. Al guardar hace
  `PUT`, invalida `businessContextProvider` y **el header se actualiza**. Entrada desde Ajustes →
  Negocio → Información del negocio (ruta `/settings/business-info`).
- **Tests (94 en total, +): ** DTO con el fixture `business_detail.json` de la auditoría,
  `RemoteBusinessRepository` (GET, PUT, mapeo 400→VALIDATION_ERROR con adapter falso de Dio),
  `BusinessClock` (offset −5), widget test de `BusinessInfoScreen` (carga, error+reintentar,
  validación de teléfono, guardado + volver), y `LSaveBar`. El `shell_navigation_test` pasó a usar
  un `FakeBusinessRepository` de test (ya no existe la fuente local).
- `flutter analyze` sin issues; `flutter test` en verde en todo el workspace (`--no-test-assets`).

## Qué quedó hecho

### 1. Estructura (`mobile/`)
- Repo git propio en `mobile/` (rama `main`). **Sin `git push`** y los commits se crearon al final (el repo aún no está en GitHub).
- Pub workspace con `resolution: workspace` y `sdk: ^3.10.0` en todos los `pubspec`:
  `packages/loyalty_core`, `packages/loyalty_design_system`, `loyalty_business`.
- `flutter create` con `--org com.qodev`, plataformas **android** e **ios**. Flutter instalado (3.41.5 / Dart 3.11.3), sin FVM.
- Nombre visible **"Vuelve Negocios"**: `AppBrand.name`, `android:label`, `CFBundleDisplayName`.
- l10n con `gen-l10n`: `app_es.arb` (plantilla) + `app_en.arb`, salida en `lib/l10n/gen/`.
- `analysis_options.yaml` estricto compartido (heredado por cada paquete). `flutter analyze` sin issues.
- `env/dev.json` con `API_BASE_URL = http://localhost:8080`, leído por `Env.load()` desde assets.
- HTTP sin cifrar **solo en debug**: Android `src/debug/res/xml/network_security_config.xml` + `usesCleartextTraffic` en el manifest de debug; iOS `NSAllowsLocalNetworking` (solo direcciones locales).
- Fuentes **Figtree** y **Bricolage Grotesque** (OFL) empaquetadas en el design system y declaradas.

### 2. `loyalty_core` (sin UI)
- `ApiException` + `ErrorReason` (todos los `reason` del audit + `network`/`timeout`/`cancelled`/`unknown`); maneja 401/403 con y sin cuerpo (decide por HTTP).
- `dioProvider`, `TraceInterceptor` (`X-Trace-Id`), `networkConfigProvider` y `extraInterceptorsProvider` (la app inyecta `AuthInterceptor`).
- `Formatters` es_CO (dinero, hora desde minutos, fechas, teléfono) y `Validators` (email, celular 10 dígitos, no vacío).
- `TokenStorage` (interfaz) + `SecureTokenStorage` sobre `flutter_secure_storage`.
- Tests: 22 (formatters, validators, api_exception).

### 3. `loyalty_design_system` (sin textos)
- Tokens claro/oscuro `LColors` (ThemeExtension), `LSpace`, `LRadius`, `LSize`, `LBreakpoints`, `TextTheme` y `buildTheme(Brightness)`.
- Componentes: `LButton` (primario, destacado, positivo, fantasma, peligro; normal y pequeño), `LChip`, `LTextField`, `LSearchField`, `LSwitch`, `LList`/`LListItem` (+`LLeadingIcon`), `StatusBadge`, `LSheet`, `LToast`, `LConfirmDialog`, `StampCard` (casilla sellada en `cardInk` con número en el color de categoría, decisión 2026-10-02).
- Widget test de cada componente en **claro y oscuro**. Tests: 42.

### 4. `loyalty_business`
- Sesión (`OwnerSessionController`, `Notifier`): token 24 h sin refresh, lee `exp`, programa cierre al vencer, revisa vencimiento al volver a primer plano, exige `role=OWNER`. `AuthInterceptor`: agrega `Bearer`, cierra ante 401 y ante 403 de **seguridad** en `/api/business/{bizId}/**` (un 403 de dominio como `REWARD_WRONG_BUSINESS` no cierra sesión).
- Login contra `POST /api/auth/login` con estados de carga y error; banner de "sesión terminada" / "no es dueño"; "olvidé mi contraseña" muestra ayuda.
- `businessContextProvider` + `LocalBusinessCapabilitiesSource` (tabla local temporal por tipo) detrás de `BusinessRepository`, listo para cambiarse por `GET /api/business/{bizId}`.
- `go_router` con `StatefulShellRoute`: header (categoría + nombre + dirección + engranaje de Ajustes), `NavigationBar` en teléfono y `NavigationRail` desde 840 px, pestañas según capacidades. Cada pestaña y Ajustes: pantalla vacía con su nombre.
- Tests: 16 — DTO de login con su fixture, `OwnerSessionController` (vencimiento, 401, 403, rol, restore), `AuthInterceptor`, y navegación (sin `hasBooking` no muestra Agenda y abre en Sellos; con `hasBooking` muestra Agenda y abre ahí).

## Decisiones tomadas

- **Faltan documentos fuente en el repo.** No existen `docs/Vuelve — Design System para Flutter.md` (valores de tokens), `docs/guidelines-frontend-mobile.md` (guía padre) ni `docs/fonts/`. Acordado con el equipo:
  - Tokens derivados del CSS del prototipo (única fuente). Si aparece el doc base y difiere, manda el doc.
  - Fuentes descargadas de Google Fonts (OFL) y empaquetadas.
- **API_BASE_URL = `http://localhost:8080`** (dev). En emulador Android, `localhost` no resuelve al host; para probar contra un backend en la máquina usar `http://10.0.2.2:8080` (el `network_security_config` de debug ya permite ambos).
- **Riverpod 3.x sin code-gen**; `go_router` 14.8.1; `lucide_icons_flutter` 3.1.22.
- **Color de categoría**: por ahora se mapea el tipo a un token existente del design system (teal/sun/rose/ink). Si el diseño define una paleta de categorías, agregarla como tokens.
- **`flutter test` en esta máquina** requiere `--no-test-assets`: el Control de aplicaciones de Windows (Smart App Control / WDAC) bloquea `impellerc.exe` al compilar el shader `ink_sparkle`. No afecta al código; es política de la máquina.

## Fuera de alcance de esta etapa (pendiente)

- PIN local y bloqueo tras segundo plano (sesión §6).
- Features completas: Agenda, Sellos (sumar/canjear), Programas, Clientes, Resumen y todas las pantallas de Ajustes (hoy son placeholders).
- `RemoteBusinessRepository` sobre `GET /api/business/{bizId}` (reemplaza la tabla local).
- Persistencia del correo precargado entre reinicios (hoy solo dentro de la sesión del proceso).
- Recuperación de contraseña del dueño (fase D del backend).

## Cómo verificar

```
cd mobile
flutter analyze
# Tests (ver nota de impellerc):
(cd packages/loyalty_core && flutter test --no-test-assets)
(cd packages/loyalty_design_system && flutter test --no-test-assets)
(cd loyalty_business && flutter test --no-test-assets)
flutter build apk --debug
```

### Resultado de la última corrida (2026-10-08)
- `flutter analyze`: **No issues found**.
- `flutter test` (workspace): **80 tests en verde** (core 22, design system 42, business 16) con `--no-test-assets`.
- `flutter build apk --debug`: **falla por política de la máquina, no por el código.** El Control de
  aplicaciones de Windows (Smart App Control / WDAC) bloquea `impellerc.exe` al compilar el shader
  `ink_sparkle.frag` que Flutter empaqueta siempre para Android:
  `Target debug_android_application failed: ProcessException: Una directiva de Control de aplicaciones
  bloqueó este archivo`. No hay flag equivalente a `--no-test-assets` para el build. Gradle descargó
  todo y resolvió bien; solo falla ese paso de shaders.
  - **Remediación (del lado del equipo/máquina):** permitir `impellerc.exe`
    (`C:\flutter\bin\cache\artifacts\engine\windows-x64\`) en la política de Control de aplicaciones,
    o desactivar Smart App Control (Seguridad de Windows → Control de aplicaciones y navegador). Tras
    eso, `flutter build apk --debug` debería completar; el proyecto en sí ya compila (analyze + tests
    en verde).
