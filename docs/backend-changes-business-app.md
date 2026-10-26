# Cambios del backend para la app de negocios

Fecha: 2026-10-08. Parte de `docs/api-business-audit.md` y de `docs/requirements-frontend-mobile-business.md` §10.
Los cambios van por fases. Cada cambio entra en su propio commit (`[feat]`, `[fix]`), con su test.

## Decisiones ya tomadas

- **Sesión del dueño:** token único de 24 h, sin refresh ni `deviceId`. No se cambia en estas fases.
- **Capacidades por tipo de negocio:** se exponen dentro de `GET /api/business/{bizId}`, no en un endpoint de
  configuración aparte.
- **Código de premio:** el largo (6 u 8) está **pendiente de decidir** con la app cliente. No se cambia en la fase A.

---

## Fase A — Cambios rápidos

### A1. Premio desbloqueado al marcar una reserva como atendida

Hoy `UpdateBookingStatusService` calcula el premio y `BookingController.updateStatus` lo descarta.

- `PATCH /api/business/{bizId}/bookings/{bookingId}/status?status=` devuelve un `BookingStatusResponse`:
  todos los campos actuales de `BookingResponse` más
  `stampedMemberships: [{membershipId, programId, stampCount}]` y
  `unlockedRewards: [{programId, programName, code, status, expiresAt}]`.
- Son **listas** porque una reserva puede estar vinculada a varios programas. Si el caso de uso hoy devuelve un solo
  premio, se adapta para devolver todos.
- Si no se sumó sello (estado distinto de `ATTENDED` o sin programas vinculados), ambas listas van vacías.
- Test de integración del controller: atender una reserva que completa una tarjeta devuelve el premio; atender una
  que no la completa devuelve la lista vacía; cancelar no suma sello.

### A2. Datos del negocio con capacidades

**`GET /api/business/{bizId}`**

```json
{
  "id": "b1",
  "name": "Barbería El Faro",
  "businessType": "HAIR_SALON",
  "address": "Cra 6 # 4-21, Centro",
  "phone": "3121234567",
  "description": "Barbería clásica con cortes modernos.",
  "offers": ["Cortes", "Barba"],
  "status": "ACTIVE",
  "timeZone": "America/Bogota",
  "capabilities": {
    "hasBooking": true,
    "hasStaff": true,
    "hasResources": true,
    "hasServices": true,
    "resourceKind": "STAFF",
    "stampUnit": "VISITS"
  }
}
```

- Las capacidades **se derivan de la `BookingStrategy` del tipo**, para no tener dos fuentes de verdad:
  `hasBooking = strategy.bookable()`, `resourceKind = strategy.resourceKind()` (o `null`),
  `hasResources = resourceKind != null`, `hasStaff = resourceKind == STAFF`, `hasServices = resourceKind == STAFF`.
- `stampUnit` por defecto: `HAIR_SALON` y `RESTAURANT` → `VISITS`; `SPORTS_COURT` → `BOOKINGS`;
  `STORE` y `CAFE` → `PURCHASES`. Vive junto a las capacidades, en un solo lugar del dominio.
- `timeZone` sale de la constante de `TimeConfig` hasta que exista zona por negocio.

**`PUT /api/business/{bizId}`**

- Body: `name` (obligatorio, máximo 120), `address` (máximo 200), `phone` (opcional, 10 dígitos),
  `description` (opcional, máximo 500), `offers` (opcional, máximo 10 elementos de máximo 40 caracteres).
- No permite cambiar `businessType` ni `status`: eso es del Back Office.
- Devuelve el mismo cuerpo que el `GET`.
- Errores: `400 VALIDATION_ERROR`; `404 BUSINESS_NOT_FOUND`.

**Migración** (siguiente versión de Flyway): columnas `phone`, `description` y `offers` en `business`. Para `offers`,
usar el tipo que sea consistente con la persistencia actual (por ejemplo `jsonb` o una tabla `business_offer`).

**Tests:** el `GET` de cada tipo devuelve las capacidades correctas; el `PUT` valida los campos; no se puede leer ni
editar otro negocio.

### A3. Corregir `GET /stats`

`rewardsPending` y `rewardsRedeemed` llegan fijos en 0.

- `rewardsPending`: premios `AVAILABLE` del negocio con `expires_at` en el futuro.
- `rewardsRedeemed`: premios `REDEEMED` del negocio.
- Test con datos que cubran disponible, canjeado y vencido.

### A4. Exigir `ROLE_OWNER` en `/api/business/{bizId}/**`

Hoy solo se valida que el `businessId` del token coincida con el de la ruta. Se agrega que el rol sea `OWNER`.
Test: un token con `businessId` correcto y otro rol recibe `403`.

### A5. Cuerpo JSON en 401 y 403

Hoy los errores de Spring Security llegan sin cuerpo. Se agregan un `AuthenticationEntryPoint` y un
`AccessDeniedHandler` que responden con el formato estándar `{status, reason, source, timestamp}`:
`401 UNAUTHORIZED` y `403 FORBIDDEN`. El front ya maneja ambos casos por código HTTP; esto solo hace el contrato
uniforme. Test de ambos casos.

---

## Fase B — Flujo de mostrador

Se implementa después de la fase A, con su propio plan.

- **Buscar clientes:** `GET /customers?q=&limit=` (por defecto 5, máximo 20).
  - **Regla de privacidad:** por nombre, solo entre clientes con membresía o reservas en el negocio; por celular
    completo (10 dígitos) o código de cliente, en toda la plataforma con coincidencia exacta.
  - Cada resultado trae `id`, `name`, `phone`, `code` y sus membresías en este negocio.
- **Registrar cliente:** `POST /customers` con `name` y `phone`. Si el celular ya existe, devuelve ese cliente
  (`200`) en lugar de fallar; si es nuevo, lo crea (`201`).
- **Detalle de cliente:** `GET /customers/{custId}` con membresías, premios del negocio (con estado, incluido
  vencido calculado), últimas reservas e inasistencias.
- **`POST /stamps`:** `customerIdentifier` acepta celular **o** código de cliente, o bien se agrega `customerId`.
- **Horas libres:** `GET /slots?date=&serviceId=&resourceId=`, con el mismo cálculo que el endpoint del cliente,
  incluyendo `depositRequired` para un `customerId` opcional.
- **Validar premio:** `GET /rewards/{code}` con cliente, programa, premio, estado y vencimiento, sin canjearlo.
  Mismos `reason` que el canje.

## Regla de tiempo al cambiar el estado de una reserva (rama `fix/booking-status-timing`)

Implementado y con tests. Evita marcar reservas demasiado pronto:

- **`booking-config` agrega `noShowGraceMin`** (int, default **10**, rango 0–120) en request y response.
  En el `PUT` es opcional (`Integer`); si llega `null` se usa 10, para no romper a los clientes que ya
  mandan el cuerpo sin ese campo. Migración `V10__booking_no_show_grace.sql`
  (`ADD COLUMN no_show_grace_min int NOT NULL DEFAULT 10`).
- **`PATCH /bookings/{id}/status`**:
  - `NO_SHOW` solo si `ahora ≥ inicio de la reserva + noShowGraceMin`; si no, `409 NO_SHOW_TOO_EARLY`.
  - `ATTENDED` solo si `ahora ≥ inicio de la reserva − 60 min`; si no, `409 ATTENDED_TOO_EARLY`.
  - "Ahora" y la hora de la reserva se calculan en la zona del negocio (`TimeConfig`, `America/Bogota`)
    usando un `Clock` inyectable (bean ya existente) para poder probar los bordes.
- **Las reservas** en `GET /agenda` y en la respuesta de `PATCH status` agregan `noShowAllowedFrom` y
  `attendedAllowedFrom` (`LocalDateTime` en la zona del negocio, sin offset), calculados en el backend, para
  que el front muestre/deshabilite las acciones sin recalcular la regla.
- **`ErrorReason`** agrega `NO_SHOW_TOO_EARLY` y `ATTENDED_TOO_EARLY`.
- Tests: unit de la regla en el borde exacto con `Clock` fijo (un minuto antes → 409; justo a la hora → OK,
  para NO_SHOW y ATTENDED) y `@WebMvcTest` del 409 con su cuerpo JSON y de los campos nuevos en la agenda y
  en la respuesta de status.

## Fase C — Agenda completa

- `GET /agenda?date=` agrega por recurso `workingIntervals` (franjas del día ya recortadas) y `unavailableReason`
  (motivo de la excepción, o `OUT_OF_SCHEDULE` si no trabaja ese día según su horario).
- `GET /agenda/summary?from=&to=`: por día, cantidad de reservas no canceladas y si el negocio está cerrado.
- `GET /customers` agrega el conteo de reservas por cliente.
- `GET /stats` agrega clientes a 2 sellos o menos del premio y avisos enviados.

## Fase D — Antes de lanzar

- Idempotencia con header `Idempotency-Key` en `POST /stamps`, `POST /bookings`, `PATCH /bookings/{id}/status` y
  `POST /rewards/redeem`.
- Recuperar contraseña del dueño en `/api/auth/`.
- Refresh del token del dueño, si los comercios piloto lo piden.
- Capacidad al crear una mesa (`POST /resources?name=&capacity=`).
