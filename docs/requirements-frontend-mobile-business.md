# Requerimientos — Frontend Mobile Negocios (Flutter)

App para el dueño del negocio (`role = OWNER`). Equivale al modo **Admin** del prototipo, ajustado a lo que el
backend ya soporta (recursos con `kind`, horarios, excepciones y servicios por profesional).
Proyecto: `mobile/loyalty_business`. La app del cliente final está en `requirements-frontend-mobile.md`.

Estado del backend: según `docs/api-business-audit.md` (2026-10-08). Lo que todavía no existe está en §10, con la
fase del backend en que se construye. Referencia visual y de comportamiento: `docs/prototypes/vuelve-negocios.html`.

## 0. Alcance del MVP

- Un usuario `OWNER` administra **un solo negocio**: el `businessId` viene en el JWT. No hay selector de negocio.
- No hay modo cliente ni selector de modo dentro de esta app.
- La app no crea negocios ni usuarios: eso lo hace el equipo desde el Back Office (`/api/bo/`).
- No se envían avisos al dueño: la campana es solo de la app del cliente.

Fuera de alcance del MVP: empleados con permisos (`EMPLOYEE`), varios negocios por usuario, escaneo de QR,
notificaciones push y el panel web (fase posterior, `requirements-frontend-web.md`).

## 1. Capacidades según el tipo de negocio

Tras el login, la app carga `GET /api/business/{bizId}`, que trae los datos del negocio y sus capacidades
(`capabilities`: `hasBooking`, `hasStaff`, `hasResources`, `hasServices`, `resourceKind`, `stampUnit`), y con eso
muestra u oculta módulos. **No deduce nada del tipo**: usa las banderas tal como llegan.

> Ese endpoint aún no existe (fase A del backend, §10). Mientras tanto, las capacidades se leen de una tabla local
> temporal en `core/business/` con los valores de abajo, marcada para reemplazarse por la llamada real.

| Tipo | Agenda y reservas | Recursos (`kind`) | Servicios | Precio |
|---|---|---|---|---|
| `HAIR_SALON` | Sí, pasos de 30 min | Profesionales (`STAFF`) | Sí, con duración y precio | Por servicio |
| `SPORTS_COURT` | Sí, bloques de 60 min | Canchas (`COURT`) | No | Diurno y nocturno |
| `RESTAURANT` | No por ahora (`BOOKING_NOT_SUPPORTED`) | Mesas (`TABLE`), con capacidad | No | — |
| `STORE`, `CAFE` | No | No | No | — |

El nombre visible del recurso sale del `kind`: "Profesionales", "Canchas" o "Mesas".

## 2. Navegación

| Pestaña | Visible cuando | Contenido |
|---|---|---|
| Agenda | `hasBooking` | Reservas por día y recurso, acciones y nueva reserva |
| Sellos | Siempre | Sumar sello en mostrador y canjear premios |
| Programas | Siempre | CRUD de programas de fidelización |
| Clientes | Siempre | Clientes del negocio con progreso e inasistencias |
| Resumen | Siempre | Métricas y clientes a punto de completar |

- Pestaña inicial: **Agenda** si el negocio tiene reservas; si no, **Sellos**.
- **Ajustes** se abre desde el ícono de engranaje del header y es una pila de pantallas propia.
  Ahí van la información del negocio, el horario, las reservas, los recursos, los servicios, las excepciones
  y la cuenta.

## 3. Acceso y sesión

- Login con correo y contraseña: `POST /api/auth/login` → `{token, email, role, businessId}`.
  El JWT trae `sub` (= correo), `role`, `businessId` y `exp`. No hay `userId`.
- **Sesión de 24 h sin refresh (decisión 2026-10-08).** No existe `/api/auth/refresh` para el dueño. La app lee `exp`
  del token y, al vencer, cierra la sesión y pide login de nuevo con el mensaje "Tu sesión terminó. Inicia sesión de
  nuevo." El correo queda precargado.
- Cualquier `401` cierra la sesión. Un `403` en `/api/business/{bizId}/**` también: el token no corresponde al negocio.
  Esas respuestas llegan **sin cuerpo JSON**, así que se decide solo por el código HTTP.
- Si `role` no es `OWNER`, no se entra. Credenciales incorrectas o usuario inactivo: `401 INVALID_CREDENTIALS`.
- Recuperar contraseña: aún no existe para el dueño (fase D). El enlace "Olvidé mi contraseña" muestra cómo
  contactar al equipo.
- PIN local opcional de 4 dígitos y bloqueo tras 5 min en segundo plano, igual que la app cliente. En un mostrador
  el teléfono o la tablet suele quedar a la vista de otros, así que la app ofrece crear el PIN tras el primer login.
- Cerrar sesión desde Ajustes → Cuenta.

## 4. Agenda (`hasBooking`)

- Selector de días: hoy y los 6 siguientes, en chips con la cantidad de reservas no canceladas de cada día.
- Reservas del día agrupadas por recurso. Un recurso sin reservas muestra "Libre todo el día". Si el recurso no
  trabaja ese día por horario o excepción, muestra "No trabaja hoy" con el motivo, si lo hay.
- Resumen del día: "N reservas · M por atender".
- Por reserva: hora de inicio y fin, cliente, servicio (o cancha), precio, estado y la marca "Sello sumado".

**Acciones por estado**

| Estado | Acciones |
|---|---|
| `CONFIRMED` | Atendida · No asistió · Cancelar |
| `PENDING_DEPOSIT` | Confirmar abono · Cancelar |
| `ATTENDED`, `NO_SHOW`, `CANCELLED` | Ninguna |

- **Atendida:** suma sello en los programas vinculados a reservas. Si completa una tarjeta, abre el sheet de premio
  desbloqueado con el código y las opciones "Guardar para después" y "Canjear ahora".
- **No asistió:** suma una inasistencia. Si el cliente llega al umbral de abono, el toast lo indica:
  "Laura lleva 2 inasistencias: sus próximas reservas pedirán abono."
- **Cancelar** pide confirmación y avisa al cliente (lo hace el backend).
- **Confirmar abono** pasa la reserva a `CONFIRMED` y avisa al cliente.

**Regla de tiempo (la decide el backend).** Cada reserva trae `attendedAllowedFrom` y `noShowAllowedFrom`
(zona del negocio). El front **no** recalcula la regla: deshabilita "Atendida"/"No asistió" mientras el reloj
del negocio sea anterior a la fecha correspondiente, con la nota "Disponible desde las 3:10 p. m."
(con `Formatters`). Con la recarga de cada 60 s se habilitan solas. Si el backend igual recibe la acción
antes de tiempo responde `409 NO_SHOW_TOO_EARLY` o `ATTENDED_TOO_EARLY`, que se muestran como toast.
El tiempo de espera para "No asistió" se configura en **Ajustes → Reservas** con "Tiempo de espera antes de
marcar inasistencia" (0 a 120 min, es `noShowGraceMin` de `booking-config`; "Atendida" se habilita siempre
60 min antes del inicio).

**Nueva reserva (desde el negocio)**

- Para clientes que llaman o escriben por WhatsApp. Se abre en un bottom sheet.
- Pasos en una sola pantalla:
  1. Cliente: se busca por nombre, celular o código, o se registra uno nuevo con nombre y celular.
  2. Servicio y profesional (peluquería) o cancha.
  3. Día y hora libre.
- Las horas salen del backend con el mismo cálculo que ve el cliente (horario, descansos, excepciones y reservas).
- Si el cliente alcanzó el umbral de inasistencias, se avisa que la reserva quedará pendiente de abono.
- Muestra la nota de los programas en los que sumará sello al asistir.

## 5. Sellos y canje

- Buscar cliente por nombre, celular o código de cliente (`V-XXXX..`). Mientras se escribe se muestran hasta
  5 coincidencias.
- Si no existe, se ofrece "Registrar cliente nuevo" (nombre + celular de 10 dígitos). El celular es único.
- Elegir un programa activo del negocio y ver la tarjeta del cliente en tiempo real (`StampCard` estática).
- **Sumar sello:** animación de sello en la casilla nueva y toast "Sello sumado: 5 de 10". Si completa la meta,
  se abre el sheet de premio desbloqueado.
- **Reglas de búsqueda (privacidad):** el cliente es global en la plataforma, así que por **nombre** solo se buscan
  clientes que ya tienen relación con el negocio; por **celular completo o código** se busca en toda la plataforma
  con coincidencia exacta, para encontrar a quien viene por primera vez.
- **Canjear premio:** campo de código en mayúsculas.
  - **Formato real (hoy):** 8 celdas, alfabeto hexadecimal `0–9 A–F` (es lo que genera el backend: un `UUID` recortado
    a 8 en mayúsculas). El largo y el alfabeto salen de una sola constante en `loyalty_core` (`RewardCode.length` y
    `RewardCode.alphabet`), para cambiarlos en un solo lugar. **El formato definitivo está pendiente de decidir con la
    app cliente.** Al teclear o pegar se normaliza: minúsculas → mayúsculas, `O → 0`, `I`/`L → 1`, y se ignora lo que
    no esté en el alfabeto. (El diseño pedía excluir `O 0 I 1`; como el backend sí usa `0` y `1`, solo se corrigen las
    letras confundibles.)
  - Al validar se muestra el cliente y el premio, y se pide confirmación (requiere `GET /rewards/{code}`, fase B).
  - Errores, ya soportados por `POST /rewards/redeem?code=`: `REWARD_NOT_FOUND` (no existe),
    `REWARD_WRONG_BUSINESS` (es de otro negocio), `REWARD_ALREADY_REDEEMED` (ya canjeado), `REWARD_EXPIRED` (vencido).
- **Doble toque:** mientras una petición de sellar, canjear o reservar está en curso, el botón queda deshabilitado.
  El backend aún no tiene idempotencia (fase D).
- Si el negocio no tiene programas activos, se invita a crear uno.

## 6. Programas

- Lista de programas con su regla ("Cada 10 visitas: …"), los días de vigencia del premio, la marca "Suma con
  reservas" y un switch de activo.
- Crear y editar en bottom sheet:
  - nombre y meta (2 a 30)
  - unidad: visitas, reservas, compras o servicios, preseleccionada con `stampUnit` de las capacidades (en el backend: `VISITS`, `BOOKINGS`, `PURCHASES`, `SERVICES`)
  - tipo de premio: gratis, porcentaje, segundo artículo, monto en pesos o regalo
  - valor (solo para porcentaje, segundo artículo y monto)
  - texto que ve el cliente, que se autocompleta hasta que el dueño lo edita
  - días de vigencia y si la tarjeta se reinicia o termina al completarse
- Vista previa de la tarjeta en vivo mientras se edita.
- Eliminar pide confirmación y advierte que se pierde el progreso de los clientes.
- Pausar un programa lo oculta a los clientes pero no borra el progreso.

## 7. Clientes

- Clientes que tienen membresía en algún programa o reservas en el negocio.
- Por cliente: nombre, celular, progreso por programa (`4/10`), número de reservas e inasistencias.
- Búsqueda por nombre o celular. Orden: actividad más reciente primero.
- **Perdonar inasistencias** con confirmación, que pone el contador en 0 (`DELETE /customers/{custId}/no-shows`).
- Al tocar un cliente se abre su detalle: tarjetas, premios disponibles e historial de reservas.

## 8. Resumen

- Clientes con tarjeta, sellos entregados, premios por canjear y premios canjeados.
- Si el negocio tiene reservas: reservas de los próximos 7 días y porcentaje de inasistencias.
- Avisos automáticos enviados a clientes.
- Clientes a 2 sellos o menos del premio, con acceso a sumarles sello.
- Se quita "Restablecer datos de ejemplo" del prototipo.

## 9. Ajustes

| Pantalla | Visible cuando | Contenido | Endpoint | Estado |
|---|---|---|---|---|
| Información del negocio | Siempre | Nombre, dirección, teléfono, descripción, lo que ofrece; categoría solo lectura | `GET/PUT /api/business/{bizId}` | Falta (fase A) |
| Horario del negocio | Siempre | Franjas por día de la semana; un día sin franjas = cerrado | `GET/PUT /business-hours` | Listo |
| Reservas | `hasBooking` | Recordatorio (horas antes), abono tras N inasistencias, programas vinculados, tarifas de cancha | `GET/PUT /booking-config` | Listo |
| Profesionales / Canchas / Mesas | `hasResources` | Alta y baja; por profesional: horario semanal con descansos y servicios que hace | `GET/POST/DELETE /resources`, `GET/PUT /resources/{id}/schedule`, `GET/PUT /resources/{id}/services` | Listo (capacidad de mesa: falta) |
| Servicios | `hasServices` | Nombre, duración, precio; uno nuevo se asigna a todo el STAFF activo | `POST /services?name=&durationMin=&price=`, `DELETE /services/{id}` | Listo |
| Excepciones | `hasBooking` | Festivos, cierres, vacaciones y días libres; de todo el negocio o de un recurso; día completo o parcial | `GET/POST/DELETE /schedule-exceptions` | Listo |
| Cuenta | Siempre | Correo, PIN, tema (sistema, claro, oscuro), cerrar sesión | Local | Listo |

Detalles del contrato en estas pantallas:
- `booking-config` usa **horas enteras** (`openHour`, `closeHour`, `nightFromHour`), no minutos. El resto del backend
  usa minutos desde medianoche. La conversión se hace solo en el DTO de esta pantalla.
- `POST /resources?name=` y `POST /services?…` reciben **query params**, no body. El `kind` del recurso lo decide el
  backend; la capacidad de una mesa todavía no se puede fijar.
- `GET /business-hours` trae `defaultHours: true` cuando el negocio no tiene horario propio (usa apertura y cierre
  de `booking-config`). `GET /resources/{id}/schedule` trae `inheritsBusinessHours`.
- Los días llegan como `DayOfWeek` (`MONDAY` … `SUNDAY`).
- "Lo que ofrece" (`offers`) es una **lista de texto libre** del negocio (≤10, cada una ≤40 car.) e **independiente**
  de Servicios. Para no reescribir, el campo muestra los nombres de los **Servicios** como chips sugeridos (tocar =
  agregar como texto); se excluyen los que ya estén en la lista. No se sincroniza al renombrar un servicio.

Reglas:
- No se puede quitar un recurso con reservas pendientes; se avisa y se ofrece ir a la agenda.
- Los horarios del recurso siempre se recortan al horario del negocio; la pantalla lo explica.
- Si el negocio no tiene horario semanal propio, se usa apertura y cierre de `booking_config` los 7 días.
- Las tablas `duration_min` y `price` por profesional existen pero no se usan aún: la UI no las muestra.

## 10. Dependencias de backend

Estado al 2026-10-08 (`docs/api-business-audit.md`). Los cambios del backend van por fases; el front construye primero
lo que ya está listo.

| Fase | Cambio en el backend | Desbloquea en el front |
|---|---|---|
| A | `unlockedReward` en la respuesta de `PATCH /bookings/{id}/status` (hoy el caso de uso lo calcula y el controller lo descarta) | Agenda → Atendida con premio |
| A | `GET/PUT /api/business/{bizId}` con datos, `capabilities` y `timeZone`; columnas `phone`, `description`, `offers` | Arranque, Ajustes → Información |
| A | Corregir `rewardsPending` y `rewardsRedeemed` en `/stats` (hoy fijos en 0) | Resumen |
| A | Exigir `ROLE_OWNER` también en `/api/business/{bizId}/**` | Seguridad |
| B | Buscar (`GET /customers?q=`), registrar (`POST /customers`) y detalle (`GET /customers/{id}`) de clientes; `customerIdentifier` por celular o código | Sellos, Nueva reserva, Clientes |
| B | `GET /api/business/{bizId}/slots` | Nueva reserva |
| B | `GET /api/business/{bizId}/rewards/{code}` para validar antes de canjear | Canje |
| C | `/agenda` con franjas y motivo de "No trabaja hoy" por recurso; conteo de reservas por día; conteo de reservas por cliente | Agenda completa, Clientes |
| C | `/stats` con clientes a punto de completar y avisos enviados | Resumen |
| D | Idempotencia en sellos, reservas y canje; recuperar contraseña; refresh del dueño | Uso diario y robustez |
| Por decidir | Largo del código de premio: 6 (diseño) u 8 (backend actual), coordinado con la app cliente | Canje |

**Qué se puede construir ya:** proyecto base, design system, login y sesión, shell (con capacidades locales
temporales), Programas y Ajustes (horario, reservas, recursos, servicios, excepciones).

## 11. UX general

- Tema claro, oscuro y del sistema, con selector en Ajustes → Cuenta.
- Formularios cortos en bottom sheets. Los editores largos (horario semanal, excepciones) van en pantalla completa
  dentro de Ajustes.
- Feedback con toasts usando el mismo verbo del botón.
- Diálogo de confirmación en toda acción destructiva o que avisa al cliente.
- Funciona en teléfono y en tablet: en pantallas anchas la navegación pasa a riel lateral.
- Estados de carga, vacío y error en cada pantalla, con "Reintentar".
