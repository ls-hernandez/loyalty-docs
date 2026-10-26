# Diseño — Recursos, horarios y BookingStrategy

Fecha: 2026-10-02. Estado: **aprobado**; pasos 1–9 y 13 implementados en `development` (V3). Paso 11 hecho en `fix/business-booking-strategy`. Pendientes: 10 y 12.

## Problema
- `resource(id, business_id, name, status)` es genérico: peluqueros, canchas y mesas son lo mismo y solo tienen nombre.
- La disponibilidad sale de `booking_config.open_hour/close_hour`: mismo horario los 7 días, sin descansos, sin días libres, sin horario por profesional y sin relación profesional ↔ servicio.
- `CreateBusinessBookingService` (reserva del dueño) no valida horario, tipo de negocio ni servicio del profesional.
- No hay tests del cálculo de slots.

## 1. Modelo: una sola tabla `resource` + `kind`
- `ResourceKind { STAFF, COURT, TABLE }` y `capacity` (nullable, mesas). Lo específico por tipo va en tablas satélite.
- El `kind` lo decide la strategy según el `BusinessType`, no el dueño (`POST /resources?name=` no cambia).
- Descartado: tablas separadas por tipo, porque obligan a una FK polimórfica en `booking`, triplican horarios y agenda, y rompen el `resourceId` que usa Flutter.

Records de dominio (uno por archivo): `ResourceKind`, `TimeWindow`, `WorkingInterval(day, startMin, endMin)`,
`ScheduleException(businessId, resourceId|null, dateFrom, dateTo, startMin?, endMin?, reason)`,
`ResourceService(resourceId, serviceId, durationMin?, price?)`.

## 2. Horarios
Disponibilidad = (horario del recurso ese día, o el del negocio si el recurso no tiene) ∩ horario del negocio
− excepciones del negocio − excepciones del recurso − reservas.
- `business_hours`: varias franjas por día; si el negocio no tiene filas, se usa `open/close_hour` los 7 días.
- `resource_schedule`: horario semanal por profesional con descansos; si no tiene filas, hereda el del negocio.
- `schedule_exception`: vacaciones, festivos y cierres (día completo o parcial).
- `resource_service`: qué servicios hace cada profesional, con duración y precio propios opcionales.
- Colaborador `WorkingHoursResolver` + helper `TimeWindows` (intersección y resta de franjas).

## 3. BookingStrategy por tipo de negocio
`application/service/booking/strategy/`:
- `BookingStrategy { supports(); resourceKind(); validateNewResource(); plan(ctx, request) }`, `PricingRule`.
- `SalonBookingStrategy` (exige serviceId, filtra el STAFF que hace el servicio, paso 30, `ServicePricing`).
- `CourtBookingStrategy` (exige resourceId, bloque 60, `DayNightPricing`).
- `NoBookingStrategy` (STORE, CAFE, RESTAURANT por ahora → `BOOKING_NOT_SUPPORTED`). Después vendrá `TableBookingStrategy` con `partySize`.
- `BookingStrategyRegistry` (EnumMap) que falla al arrancar si a algún `BusinessType` le falta strategy.
- `BookingPlanResolver` mantiene su firma y delega; `SlotCalculator` usa `WorkingHoursResolver`.

## 4. Migración V3 (`V3__resource_kind_and_schedules.sql`)
`resource.kind` + `capacity`; tablas `business_hours`, `resource_schedule`, `schedule_exception`, `resource_service`
(backfill: todos los STAFF × servicios activos). Detalle SQL en la propuesta original del agente.

## 5. API
Dueño (`/api/business/{bizId}`): `GET /resources` (nuevo); `GET|PUT /resources/{id}/schedule`;
`GET|PUT /resources/{id}/services`; `GET|PUT /business-hours`; `GET|POST|DELETE /schedule-exceptions`.
Cada grupo en su controller. Cliente: no cambia ninguna URL; `slots` agrega `durationMin` por slot y el detalle de
Explore agrega `kind` y `serviceIds` por recurso.

## 6. Plan (un commit por paso)
1. Tests de caracterización de `SlotCalculator`/`BookingPlanResolver`.
2. Strategies + registry (sin cambiar el comportamiento).
3. Migración V3 + persistencia.
4. Alta de recursos con kind/validación de la strategy + `GET /resources`.
5. `WorkingHoursResolver` con el fallback.
6. Horario del negocio. 7. Horario por recurso. 8. Excepciones.
9. Servicios por profesional. 10. Duración y precio por profesional.
11. ✅ **Hecho** (`fix/business-booking-strategy`): `CreateBusinessBookingService` valida con la strategy
    reutilizando `BookingPlanResolver`/`SlotCalculator`/`DepositPolicy` (mismo stack que la reserva del
    cliente y `GET /slots`). `serviceId` opcional; reglas por tipo, precio día/noche, horario, excepciones,
    `SLOT_NOT_AVAILABLE`/`CUSTOMER_BOOKING_OVERLAP`/`BOOKING_NOT_SUPPORTED`. Ver `docs/api-business-audit.md`.
12. `kind` y `serviceIds` en Explore. 13. Documentación.
