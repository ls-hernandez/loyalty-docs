# Requerimientos — Backend (Java + Spring Boot)

## Stack
- Java + Spring Boot
- PostgreSQL
- Spring Security (auth básico usuario/contraseña + JWT)
- Arquitectura hexagonal — proyecto único

---

## Arquitectura — Hexagonal (un solo proyecto)

```
com.vuelve
├── domain
│   ├── model          # Entidades de dominio puras (sin anotaciones JPA)
│   ├── port
│   │   ├── in         # Casos de uso (interfaces que llama el controller)
│   │   └── out        # Repositorios (interfaces que implementa infra)
│   └── service        # Implementación de casos de uso + strategies
├── application
│   └── usecase        # Orquestación, validaciones de negocio
├── infrastructure
│   ├── persistence    # Entidades JPA, repositorios Spring Data
│   ├── web
│   │   ├── customer   # Controllers /api/customer/
│   │   ├── business   # Controllers /api/business/
│   │   └── bo         # Controllers /api/bo/
│   └── security       # Spring Security config, JWT filter
└── shared             # Excepciones, utilidades, mappers
```

---

## Diseño de BD

### Configuración por tipo de negocio (seed data)
```sql
business_type_config
  business_type   VARCHAR PK  -- HAIR_SALON, SPORTS_COURT, STORE, RESTAURANT, CAFE
  has_booking     BOOLEAN
  has_staff       BOOLEAN
  has_resources   BOOLEAN
  has_services    BOOLEAN
  booking_unit    VARCHAR     -- PER_SLOT, PER_HOUR (null si no aplica)
  stamp_unit      VARCHAR     -- VISITS, BOOKINGS, PURCHASES, SERVICES
```

### Negocios y recursos
```sql
business
  id, name, business_type, address, status, created_at

booking_config
  id, business_id FK, open_hour, close_hour,
  remind_hours, deposit_after_no_shows, created_at

resource                        -- profesional, cancha o mesa
  id, business_id FK, name, status,
  kind (STAFF | COURT | TABLE),  -- lo decide la BookingStrategy según business_type
  capacity (nullable)            -- solo mesas (futuro TableBookingStrategy)

business_hours                  -- horario semanal del negocio, varias franjas por día (V3)
  id, business_id FK, day_of_week (1=lunes..7=domingo), start_min, end_min
  -- sin filas: se usa booking_config.open_hour/close_hour los 7 días
  -- con filas: un día sin franjas = cerrado

resource_schedule               -- horario semanal de un recurso; los huecos son descansos (V3)
  id, resource_id FK, day_of_week, start_min, end_min
  -- sin filas: hereda el horario del negocio; siempre se recorta al horario del negocio

schedule_exception              -- festivos, cierres, vacaciones y días libres (V3)
  id, business_id FK, resource_id FK (null = todo el negocio),
  date_from, date_to, start_min, end_min (ambos null = día completo), reason, created_at

resource_service                -- qué servicios hace cada profesional (V3)
  resource_id FK, service_id FK, duration_min (nullable), price (nullable)  -- PK compuesta
  -- duration_min/price propios: columnas listas, aún no se usan (paso 10 del diseño)

offered_service                 -- servicio ofrecido (aplica según business_type_config)
  id, business_id FK, name, duration_min, price, status
```

### Auth y usuarios
```sql
app_user                        -- dueño del negocio (Vuelve Negocios)
  id, name, email (unique), password_hash, role, business_id FK, status, created_at
  -- role: OWNER (MVP). Futuro: EMPLOYEE, BO_ADMIN
  -- auth: JWT con role=OWNER + businessId claim
```

### Clientes
```sql
customer                        -- cliente final (Vuelve app)
  id, name, phone (unique), email (unique, nullable), password_hash (nullable), created_at
  -- phone se crea al primer sello; email+password se agregan al registrarse en la app

no_show_record
  id, customer_id FK, business_id FK, count
```

### Fidelización
```sql
loyalty_program
  id, business_id FK, name, goal, unit, reward_type,
  reward_value, reward_text, expiry_days, reset_on_complete, status

program_booking_link
  program_id FK, business_id FK

membership
  id, customer_id FK, program_id FK, stamp_count, nudged_at

stamp_history
  id, membership_id FK, type (STAMP/REWARD), created_at

reward
  id, customer_id FK, program_id FK, code (6 chars, unique),
  status (AVAILABLE/REDEEMED), created_at, expires_at, redeemed_at
  -- EXPIRED se calcula en runtime: expires_at < now
```

### Reservas
```sql
booking
  id, business_id FK, customer_id FK, resource_id FK,
  service_id FK (nullable), date, start_min, duration_min,
  price, status, reminded, stamped, created_at
  -- status: CONFIRMED, PENDING_DEPOSIT, ATTENDED, NO_SHOW, CANCELLED
```

### Notificaciones
```sql
notification
  id, customer_id FK, business_id FK, kind, text, read, created_at
  -- kind: REMINDER, PRIZE, ALMOST, COME_BACK,
  --       BOOKING_CONFIRMED, BOOKING_CANCELLED, DEPOSIT_CONFIRMED
```

---

## Estrategias por tipo de negocio

### BookingStrategy (implementado, `application/service/booking/strategy/`)
```java
public interface BookingStrategy {
  Set<BusinessType> supports();
  boolean bookable();
  Optional<ResourceKind> resourceKind();      // kind de los recursos nuevos
  void validateNewResource(String name);
  BookingPlan plan(BookingContext context, BookingRequest request);
}
```

| Strategy | Tipos | Recursos | Plan |
|----------|-------|----------|------|
| `SalonBookingStrategy` | HAIR_SALON | STAFF | exige `serviceId`; solo STAFF activo que hace el servicio (`resource_service`); paso 30 min; `ServicePricing` |
| `CourtBookingStrategy` | SPORTS_COURT | COURT | exige `resourceId`; bloques de 60 min; `DayNightPricing` (`nightFromHour`) |
| `NoBookingStrategy` | RESTAURANT | TABLE | `400 BOOKING_NOT_SUPPORTED` (después vendrá `TableBookingStrategy` con `partySize`) |
| `NoBookingStrategy` | STORE, CAFE | — (no admite recursos) | `400 BOOKING_NOT_SUPPORTED` |

- `BookingStrategyRegistry` (EnumMap) falla al arrancar si a un `BusinessType` le falta strategy o tiene dos.
- `BookingPlanResolver.resolve(businessId, serviceId, resourceId)`: negocio activo → strategy →
  `BOOKING_NOT_SUPPORTED` si no es reservable → `booking_config` → `strategy.plan(...)`.
- `SlotCalculator` usa `WorkingHoursResolver` (disponibilidad por recurso) y `TimeWindows`
  (intersección, resta y unión de franjas):
  `disponible = (horario del recurso ese día, o el del negocio si no tiene) ∩ horario del negocio
  − excepciones del negocio − excepciones del recurso − reservas`.
- Agregar un tipo nuevo = nueva strategy + bean; el registry obliga a cubrir todos los tipos.

### StampStrategy (port/in)
```java
interface StampStrategy {
    boolean shouldAutoStamp(Booking booking);
    String getStampUnit(BusinessType type);
}
```

---

## Auth

- Spring Security con JWT (usuario/contraseña → token)
- Token contiene: `userId`, `businessId`, `role`
- Rutas `/api/customer/**` — auth por `customerId` (futuro; MVP puede ser abierto o token simple)
- Rutas `/api/business/**` — requieren JWT válido con `businessId` coincidente
- Rutas `/api/bo/**` — requieren JWT con `role = BO_ADMIN`

### Endpoints de auth
| Método | Endpoint | Descripción |
|--------|----------|-------------|
| POST | `/api/auth/login` | Login usuario/contraseña → JWT |
| POST | `/api/auth/refresh` | Renovar token |

---

## Endpoints — Configuración (público/startup)
| Método | Endpoint | Descripción |
|--------|----------|-------------|
| GET | `/api/config/business-types` | Capacidades por tipo (frontend lo carga al inicio) |

---

## Endpoints — Cliente (`/api/customer/`)
| Método | Endpoint | Descripción |
|--------|----------|-------------|
| GET | `/api/customer/me/memberships` | Tarjetas y progreso |
| GET | `/api/customer/me/rewards` | Premios |
| GET | `/api/customer/me/bookings` | Reservas propias |
| POST | `/api/customer/me/bookings` | Crear reserva |
| DELETE | `/api/customer/me/bookings/{id}` | Cancelar reserva |
| POST | `/api/customer/programs/{id}/join` | Unirse a programa |
| GET | `/api/customer/me/notifications` | Avisos in-app |
| PATCH | `/api/customer/me/notifications/read` | Marcar leídos |
| GET | `/api/customer/businesses` | Explorar negocios |
| GET | `/api/customer/businesses/{id}/slots` | Horas disponibles |

---

## Endpoints — Negocio (`/api/business/`)
| Método | Endpoint | Descripción |
|--------|----------|-------------|
| GET | `/api/business/{bizId}/agenda` | Reservas del día |
| PATCH | `/api/business/{bizId}/bookings/{id}/status` | Cambiar estado reserva |
| POST | `/api/business/{bizId}/bookings` | Crear reserva desde panel |
| POST | `/api/business/{bizId}/stamps` | Sumar sello manual |
| POST | `/api/business/{bizId}/rewards/{code}/redeem` | Canjear premio |
| GET/POST/PUT/DELETE | `/api/business/{bizId}/programs` | CRUD programas |
| GET/PUT | `/api/business/{bizId}/booking-config` | Config de reservas |
| GET/POST/DELETE | `/api/business/{bizId}/resources` | Listar y gestionar profesionales/canchas/mesas (kind por tipo de negocio) |
| GET/PUT | `/api/business/{bizId}/resources/{id}/schedule` | Horario semanal del recurso (descansos = huecos) |
| GET/PUT | `/api/business/{bizId}/resources/{id}/services` | Servicios que hace un profesional (solo STAFF) |
| GET/PUT | `/api/business/{bizId}/business-hours` | Horario semanal del negocio |
| GET/POST/DELETE | `/api/business/{bizId}/schedule-exceptions` | Festivos, cierres, vacaciones, días libres |
| POST/DELETE | `/api/business/{bizId}/services` | Gestionar servicios (uno nuevo se asigna a todo el STAFF activo) |
| GET | `/api/business/{bizId}/customers` | Clientes con progreso |
| PATCH | `/api/business/{bizId}/customers/{custId}/no-shows/clear` | Perdonar inasistencias |
| GET | `/api/business/{bizId}/stats` | Métricas resumen |

---

## Endpoints — Back Office (`/api/bo/`)

> Solo accesible con rol `BO_ADMIN`. Permite a nosotros (el equipo) gestionar la plataforma.

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| POST | `/api/bo/businesses` | Crear nuevo negocio |
| POST | `/api/bo/users` | Crear usuario OWNER para un negocio |
| GET | `/api/bo/businesses` | Listar todos los negocios |
| PATCH | `/api/bo/businesses/{id}/active` | Activar/desactivar negocio |
| GET | `/api/bo/users` | Listar usuarios admin |

---

## Jobs periódicos (Spring @Scheduled)
- **Recordatorios** (cada minuto): reservas CONFIRMED, `reminded=false`, `startTime - now <= remindHours` → notificación REMINDER
- **Reactivación** (diario): memberships con `stampCount > 0`, último sello hace >30 días → notificación COME_BACK
