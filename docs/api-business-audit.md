# Auditoría de API — Lado Negocios

Fecha: 2026-10-08
Alcance: `/api/auth/**`, `/api/config/**` y `/api/business/**` del backend
`loyalty-backend/`, contrastado con `docs/requirements-frontend-mobile-business.md`
y `docs/guidelines-frontend-mobile-business.md`.

> No se modificó código. Todo dato importante cita el archivo de origen. Lo que no se
> pudo confirmar leyendo el código se marca como **(por verificar)**.
>
> **Nota:** No existen documentos de diseño dentro de `loyalty-backend/`
> (no hay `requirements-backend.md` ni `design-resources-schedules.md`; de hecho no hay
> ningún `.md` en el backend). La auditoría se basa solo en el código fuente.

---

## 1. Resumen

> **Actualización Fase A (2026-10-08, rama `feature/business-app-phase-a`).** Ya implementado y
> con tests: premio desbloqueado al atender (A1), `GET/PUT /api/business/{bizId}` con capacidades
> y zona horaria (A2, resuelve también el bloqueante de arranque — las capacidades no van en
> `/api/config`), conteos de premios en `/stats` (A3), exigir rol OWNER en rutas de negocio (A4) y
> cuerpo JSON en 401/403 (A5). El texto abajo describe el estado **original**; cada punto resuelto
> queda marcado ✅ en §2–§6.
>
> **Actualización Fase B (2026-10-09, rama `feature/business-app-phase-b` sobre la A).** Ya
> implementado y con tests el **flujo de mostrador**: buscar clientes con regla de privacidad
> (`GET /customers?q=&limit=`), registrar o reutilizar por celular (`POST /customers`), detalle de
> cliente (`GET /customers/{custId}`), horas libres desde el negocio (`GET /slots`, reutiliza el
> cálculo del cliente), previsualizar premio (`GET /rewards/{code}`) y `POST /stamps` ahora acepta
> celular, **código de cliente** o `customerId`. **Decisión:** el código de premio se queda en **8
> caracteres** (sin cambio de generador). Detalle en §2.3, §3, §4 (marcados 🅱️) y `backend-changes-business-app.md`.
> Siguen pendientes las Fases C y D.
>
> **Integración (2026-10-14).** Las Fases A y B ya están **mergeadas a `development`** (dos merge
> commits `--no-ff`: `[chore] merge Phase A …` y `[chore] merge Phase B …`), sin push. La regla de
> tiempo (`fix/booking-status-timing`) sigue en su propia rama, aún sin mergear.
>
> **Fix contrato de clientes (2026-10-14, rama `fix/customer-list-contract`).** `GET /customers` con y
> sin `?q=` devuelven ahora **el mismo DTO** (`CustomerCardResponse` con `noShows` + `memberships`
> con `programName`/`goal`/`stampCount`). Se eliminaron `CustomerSummaryResponse`/`CustomerSummary`.
> Las inasistencias se cargan por lote (`NoShowRepositoryPort.findByBusinessIdAndCustomerIdIn`) en
> lista, búsqueda y alta (sin N+1). Sin push, sin mergear.
>
> **Merge de `origin/development` (2026-10-26, rama `chore/merge-remote-development`).** Se integró
> el trabajo del compañero (app cliente / exploración pública) sobre la Fase C de negocio. Entró
> **sin conflictos**. Novedades, todas en el **dominio cliente/público** (no tocan el contrato de
> negocio salvo donde se indica):
> - **Endpoints públicos para invitados** (`web/publicapi`): explorar negocios, tipos de negocio
>   públicos, slots públicos y **reserva de invitado** (`POST` con rate limit). La reserva de
>   invitado reutiliza cliente por **celular** (`CustomerRepositoryPort.findByPhone(...).orElseGet(crear)`),
>   **la misma regla** que `POST /api/business/{bizId}/customers` (mostrador).
> - **Exploración de negocios** (`web/customer`) con **paginación keyset** (cursor) y filtros en
>   servidor.
> - **`business_type_config.status`** (`ACTIVE`/`INACTIVE`): migración `V11` (columna + CHECK) y `V12`
>   (lo convierte a enum Postgres `business_type_status`). El endpoint público de tipos oculta los
>   `INACTIVE` (`GetBusinessTypesService.findAllActive()`). `BusinessType` pasa a tiparse como enum
>   en el código de config.
> - **Migraciones:** quedan `V1`→`V12` **sin huecos ni números repetidos**; verificado contra BD
>   limpia que Flyway aplica las 12 en orden y todas en `success`.
> - **Contrato de negocio preservado (decisión 2026-10-26).** El refactor del compañero renombró el
>   parámetro de búsqueda `q`→`search`. En `GET /api/business/{bizId}/customers` **se mantiene `q`**
>   (este documento es la fuente de verdad y la app de negocio manda `q`); `search`/`searchText` viven
>   **solo** en los endpoints de `web/customer` y `web/publicapi`. Hay un test de regresión
>   (`BusinessCustomersControllerTest.searchParamIsNamedQ`) que falla si alguien vuelve a renombrarlo.
> - **Deuda técnica:** `business_type_config` arrastra desde `V2` columnas
>   `has_booking/has_staff/has_resources/has_services` que **duplican** los flags de la `BookingStrategy`.
>   Las capacidades de `GET /api/business/{bizId}` siguen saliendo **solo** de la `BookingStrategy`
>   (`BusinessCapabilities.from(type, strategyRegistry.forType(...))`, fuente única); esas columnas
>   **no** las lee ese camino. Quedan como deuda para no acabar con dos fuentes de verdad.

El backend tiene listos y con contrato estable casi todos los endpoints de **Ajustes**
(horario, reservas/booking-config, recursos, horario y servicios por recurso, servicios,
excepciones) y el **CRUD de Programas** completo. Login funciona pero devuelve un **único
token de 24 h sin refresh** y sin `userId`; el esquema por dispositivo del cliente
(`deviceId`, refresh rotado, sesiones) **no** aplica al dueño.

Hay un **bloqueante de arranque**: no existe `/api/config/business-types` ni ninguna
bandera de capacidades (`has_booking`, `has_staff`, `has_resources`, `has_services`,
`stamp_unit`); el enum `BusinessType` es solo un nombre. Sin eso la app no puede decidir
pestañas ni pantallas.

Están **a medias** los flujos de mostrador y agenda: `POST /stamps` sí devuelve el premio
desbloqueado, pero `PATCH /bookings/{id}/status` **calcula el premio y lo descarta en el
controller**; el sello solo acepta **teléfono** (no nombre ni código) y **no registra**
clientes; el canje existe pero **no hay endpoint para validar/previsualizar** el código;
`GET /customers` **no tiene búsqueda ni paginación** y no hay detalle de cliente; y `stats`
devuelve dos métricas **fijas en 0**.

**Faltan por completo**: capacidades (`/api/config`), info editable del negocio
(`GET/PUT /api/business/{bizId}` + columnas `phone/description/offers`), horas libres desde
el negocio (`GET /slots`), búsqueda/registro/detalle de clientes, validación de premio y
recuperación de contraseña del dueño.

**Lo que bloquea empezar el front**: (1) capacidades por tipo de negocio, (2) definir el
esquema de sesión del dueño, (3) búsqueda/registro de clientes para Sellos y Nueva reserva.

---

## 2. Endpoints existentes

Convenciones: todos los de `/api/business/{bizId}/**` validan que el `businessId` del JWT
sea igual (ignore-case) a `{bizId}` de la ruta vía
`OwnBusinessAuthorizationManager` (`infrastructure/security/OwnBusinessAuthorizationManager.java:20-26`);
si no coincide → **403** (sin cuerpo JSON, ver §5). El rol concreto (`OWNER`) **no** se
revalida en esas rutas: solo se exige que el principal sea de negocio con `businessId`
coincidente (`infrastructure/security/SecurityConfig.java:46-51`).

### 2.1 Auth (`/api/auth`)

| Método y ruta | Controller / caso de uso | Qué hace | Request | Response | HTTP / `reason` | Seguridad | Tests |
|---|---|---|---|---|---|---|---|
| `POST /api/auth/login` | `AuthController.login` (`infrastructure/web/auth/impl/AuthController.java:22`) → `LoginService` (`application/service/auth/LoginService.java`) | Login por correo/contraseña; emite JWT | Body: `email:String` (NotBlank, Email), `password:String` (NotBlank) (`.../auth/request/LoginRequest.java`) | `token:String`, `email:String`, `role:Role`, `businessId:String` (`.../auth/response/LoginResponse.java`) | `200`; `401` `INVALID_CREDENTIALS` (credenciales malas o usuario no `ACTIVE`) | Público (`permitAll`) | Sin test directo de `LoginService` (sí `JwtAuthFilterTest`, `OwnBusinessAuthorizationManagerTest`) |

**No existen** bajo `/api/auth`: `refresh`, `logout`, recuperación de contraseña. Esos
flujos solo existen para el cliente final en `/api/customer/auth/**`
(`CustomerAuthController`, `password-reset`, `email-verification`).

### 2.2 Config (`/api/config`)

**No existe ningún endpoint ni controller bajo `/api/config`** (y por decisión **no se
creará**). Las capacidades y el `stampUnit` por tipo se exponen ahora dentro de
`GET /api/business/{bizId}` → `capabilities` (ver §2.3 y §3, Fase A2). El enum
`domain/model/BusinessType.java` ganó `defaultStampUnit()`; las banderas se derivan de la
`BookingStrategy` del tipo.

### 2.3 Business (`/api/business/{bizId}`)

Ruta base `{bizId}` implícita en todas. Campos de tiempo en **minutos desde medianoche**
salvo donde se indique.

| Método y ruta | Controller / caso de uso | Qué hace | Request | Response | HTTP / `reason` | Tests |
|---|---|---|---|---|---|---|
| 🆕 `GET /` (raíz `{bizId}`) | `BusinessController.get` (`.../impl/BusinessController.java`) → `GetBusinessService` | Perfil del negocio + capacidades + zona horaria (Fase A2) | — | `BusinessDetailResponse` (ver §3) | `200`; `404` `BUSINESS_NOT_FOUND` | `BusinessControllerTest`, `GetBusinessServiceTest` |
| 🆕 `PUT /` (raíz `{bizId}`) | `BusinessController.update` → `UpdateBusinessService` | Edita perfil (no `businessType`/`status`) (Fase A2) | Body: `name:String`(NotBlank≤120), `address:String?`(≤200), `phone:String?`(10 díg.), `description:String?`(≤500), `offers:List<String>?`(≤10×≤40) | `BusinessDetailResponse` | `200`; `404` `BUSINESS_NOT_FOUND`; `400` `VALIDATION_ERROR` | `BusinessControllerTest`, `UpdateBusinessServiceTest` |
| `GET /agenda?date=yyyy-MM-dd` | `AgendaController` (`.../impl/AgendaController.java:22`) → `GetAgendaService` | Reservas del día agrupadas por recurso activo | Query: `date:String` (yyyy-MM-dd) | `List<AgendaResponse>`: `resourceId`, `resourceName`, 🆕 `workingIntervals:List<{startMin,endMin}>` (recurso ∩ negocio − excepciones), 🆕 `unavailableReason` (`OUT_OF_SCHEDULE` o texto de la excepción; `null` si trabaja), `bookings:List<BookingResponse>` (cada reserva incluye `noShowAllowedFrom`/`attendedAllowedFrom` y 🆕 `customerName`/`serviceName` —`serviceName` nulo en canchas—) | `200` | `AgendaControllerTest`, 🆕 `GetAgendaIntegrationTest` |
| 🆕 `GET /agenda/summary?from=&to=` | `AgendaController.summary` (`.../impl/AgendaController.java`) → `GetAgendaSummaryService` | Resumen por día para el selector de fechas: conteo de reservas no canceladas (consulta agrupada, sin N+1) y si el negocio está cerrado ese día | Query: `from:String`, `to:String` (yyyy-MM-dd, **máximo 31 días** inclusive) | `List<AgendaSummaryResponse>`: `date`, `bookingCount:long`, `closed:boolean` (un elemento por día del rango, incluidos los días sin reservas) | `200`; `400` `VALIDATION_ERROR` (`from > to` o rango mayor a 31 días) | 🆕 `AgendaControllerTest` (summary), `GetAgendaSummaryIntegrationTest` |
| `POST /bookings` | `BookingController.create` (`.../impl/BookingController.java:30`) → `CreateBusinessBookingService` | Crea reserva desde el negocio. ✏️ **Ahora valida con la `BookingStrategy`**: reutiliza `BookingPlanResolver`/`SlotCalculator`/`DepositPolicy` (mismo stack que la reserva del cliente y `GET /slots`), así que respeta horario del negocio/recurso, excepciones, precio diurno/nocturno y reservas existentes. La strategy decide qué exige: **peluquería** exige `serviceId` (duración y precio del servicio, solo STAFF que lo hace); **cancha** exige `resourceId`, ignora `serviceId`, bloques de 60 min y precio día/noche | Body: `customerId:String`, `resourceId:String`, ✏️ `serviceId:String?` (**opcional**; lo exige la strategy de peluquería), `date:String`(yyyy-MM-dd), `startMin:int` | `BookingResponse` (201; `serviceId`/`price` según la strategy) | `201`; `400` `VALIDATION_ERROR` (body inválido, fecha fuera de [hoy, hoy+7), o falta `serviceId`/`resourceId` que la strategy exige); `400` `BOOKING_NOT_SUPPORTED` (tipo sin reservas); `404` `CUSTOMER_NOT_FOUND` / `BUSINESS_NOT_FOUND` / `SERVICE_NOT_FOUND` / `RESOURCE_NOT_FOUND`; `409` `SLOT_NOT_AVAILABLE` (fuera de horario u ocupado); `409` `CUSTOMER_BOOKING_OVERLAP` | 🆕 `CreateBusinessBookingServiceTest`, `BookingControllerTest`; `CustomerOverlapTest` (capa booking) |
| `PATCH /bookings/{bookingId}/status?status=` | `BookingController.updateStatus` → `UpdateBookingStatusService` | Cambia estado; si `ATTENDED` autosella los programas activos vinculados. 🆕 **Regla de tiempo** (rama `fix/booking-status-timing`): `NO_SHOW` solo si `ahora ≥ inicio + noShowGraceMin`; `ATTENDED` solo si `ahora ≥ inicio − 60 min` (ahora en la zona del negocio, `Clock` inyectable) | Path: `bookingId`; Query: `status:BookingStatus` | ✏️ `BookingStatusResponse`: campos de `BookingResponse` (incluye 🆕 `noShowAllowedFrom`/`attendedAllowedFrom`) + `stampedMemberships[]` + `unlockedRewards[]` (ver §3) | `200`; `404` `BOOKING_NOT_FOUND`; `403` `FORBIDDEN`; 🆕 `409` `NO_SHOW_TOO_EARLY` / `ATTENDED_TOO_EARLY` | `UpdateBookingStatusServiceTest`, `BookingStatusControllerTest` |
| `GET /booking-config` | `BookingConfigController.get` (`.../impl/BookingConfigController.java:21`) | Config de reservas + recursos + servicios | — | `BookingConfigResponse` | `200` | **Sin test** |
| `PUT /booking-config` | `BookingConfigController.update` (`:30`) → `UpdateBookingConfigUseCase` | Actualiza config y devuelve la vigente | Body: `openHour:int`, `closeHour:int`, `remindHours:int`, `depositAfterNoShows:int`, `dayPrice:Long?`, `nightPrice:Long?`, `nightFromHour:Integer?`, `linkedProgramIds:List<String>`, 🆕 `noShowGraceMin:Integer?` (opcional, `@Min(0) @Max(120)`; `null` → default 10) | `BookingConfigResponse` | `200`; `400` `VALIDATION_ERROR` | **Sin test** |
| `GET /business-hours` | `BusinessHoursController.get` (`.../impl/BusinessHoursController.java:21`) | Horario semanal del negocio | — | `BusinessHoursResponse`: `defaultHours:boolean`, `intervals:[{day:DayOfWeek, startMin:int, endMin:int}]` | `200` | `BusinessHoursServicesTest` |
| `PUT /business-hours` | `BusinessHoursController.update` (`:27`) | Reemplaza el horario semanal | Body: `intervals:[{day:DayOfWeek, startMin:int, endMin:int}]` (NotNull) | `BusinessHoursResponse` | `200`; `400` `VALIDATION_ERROR` | `BusinessHoursServicesTest` |
| `GET /customers` | `BusinessCustomersController.list` → `ListBusinessCustomersUseCase` | Lista de clientes del negocio | — (**sin `q`, sin paginación**) | ✏️ `List<CustomerCardResponse>` (**mismo DTO que la búsqueda**, fix `customer-list-contract`): `id,name,phone,code,noShows:int,🆕 bookingCount:long` (reservas no canceladas del cliente en el negocio, consulta agrupada sin N+1)`,memberships:[{membershipId,programId,programName,goal,stampCount}]` | `200` | 🆕 `ListBusinessCustomersServiceTest`, `BusinessCustomersControllerTest`, `ListBusinessCustomersBookingCountIntegrationTest` |
| `DELETE /customers/{customerId}/no-shows` | `BusinessCustomersController.clearNoShows` (`:31`) → `ClearNoShowsUseCase` | Pone inasistencias a 0 | Path: `customerId` | `204` | `204` | **Sin test** |
| 🅱️ `GET /customers?q=&limit=` | `BusinessCustomersController.search` → `SearchBusinessCustomersUseCase` | Busca clientes con regla de privacidad (ver §4) | Query: `q:String`, `limit:int` (def. 5, máx 20) | ✏️ `List<CustomerCardResponse>` (**mismo DTO que la lista**, incluye `noShows` y 🆕 `bookingCount`): `id,name,phone,code,noShows:int,bookingCount:long,memberships:[{membershipId,programId,programName,goal,stampCount}]` | `200`; `400` `VALIDATION_ERROR` (q vacío) | `SearchBusinessCustomersServiceTest`, `BusinessCustomersControllerTest` |
| 🅱️ `POST /customers` | `BusinessCustomersController.create` → `CreateBusinessCustomerUseCase` | Registra un cliente; si el celular ya existe, lo devuelve | Body: `name:String`(NotBlank≤120), `phone:String`(10 díg.) | `CustomerCardResponse` | `201` (nuevo) / `200` (ya existía); `400` `VALIDATION_ERROR` | `CreateBusinessCustomerServiceTest`, `BusinessCustomersControllerTest` |
| 🅱️ `GET /customers/{customerId}` | `BusinessCustomersController.detail` → `GetBusinessCustomerUseCase` | Detalle: tarjetas, premios (estado calculado), últimas reservas e inasistencias | Path: `customerId` | `CustomerDetailResponse` (ver §3) | `200`; `404` `CUSTOMER_NOT_FOUND` | `GetBusinessCustomerServiceTest`, `BusinessCustomersControllerTest` |
| 🅱️ `GET /slots?date=&serviceId=&resourceId=&customerId=` | `BusinessSlotsController.slots` → `GetBookingAvailabilityUseCase` (el mismo del cliente: `SlotCalculator`/`BookingPlanResolver`/`DepositPolicy`) | Horas libres desde el negocio; `customerId` opcional (solo para `depositRequired`) | Query: `date`(yyyy-MM-dd), `serviceId?`, `resourceId?`, `customerId?` | `BookingAvailabilityResponse` (ver §3) | `200`; `400` `VALIDATION_ERROR` (fecha pasada), `400` `BOOKING_NOT_SUPPORTED` | `BusinessSlotsControllerTest` |
| `POST /services?name=&durationMin=&price=` | `OfferedServiceController.add` (`.../impl/OfferedServiceController.java:21`) | Alta de servicio (query params, **no body**) | Query: `name:String`, `durationMin:int`, `price:Long?`(opcional) | `ServiceItemResponse`: `id`,`name`,`durationMin:int`,`price:Long`,`status:ServiceStatus` (201) | `201` | `AddOfferedServiceServiceTest` |
| `DELETE /services/{serviceId}` | `OfferedServiceController.remove` (`:33`) | Baja de servicio | Path: `serviceId` | `204` | `204`; `404` `SERVICE_NOT_FOUND` **(por verificar el código exacto)** | `AddOfferedServiceServiceTest` (parcial) |
| `GET /programs` | `ProgramController.list` (`.../impl/ProgramController.java:44`) | Lista de programas | — | `List<ProgramResponse>` | `200` | **Sin test** |
| `POST /programs` | `ProgramController.create` (`:25`) | Crea programa | Body: `name:String`, `goal:int`(≥1), `unit:StampUnit`, `rewardType:RewardType`, `rewardValue:Long?`, `rewardText:String?`, `expiryDays:int`(≥1), `resetOnComplete:boolean` | `ProgramResponse` (201) | `201`; `400` `VALIDATION_ERROR` | **Sin test** |
| `PUT /programs/{programId}` | `ProgramController.update` (`:51`) | Edita programa (mismo body que crear) | Path: `programId`; Body: `CreateProgramRequest` | `ProgramResponse` | `200`; `404` `PROGRAM_NOT_FOUND` **(por verificar)** | **Sin test** |
| `DELETE /programs/{programId}` | `ProgramController.delete` (`:73`) | Elimina programa | Path: `programId` | `204` | `204` | **Sin test** |
| `PATCH /programs/{programId}/status?targetStatus=` | `ProgramStateController.changeStatus` (`.../impl/ProgramStateController.java:18`) | Activa/pausa programa | Path: `programId`; Query: `targetStatus:ProgramStatus` | `ProgramResponse` | `200` | **Sin test** |
| `GET /resources` | `ResourceController.list` (`.../impl/ResourceController.java:32`) | Lista recursos | — | `List<ResourceItemResponse>`: `id`,`name`,`status:ResourceStatus`,`kind:ResourceKind`,`capacity:Integer?` | `200` | `ListResourcesServiceTest` |
| `POST /resources?name=` | `ResourceController.add` (`:24`) → `AddResourceService` | Alta de recurso; `kind` se deriva del tipo de negocio, `capacity=null` | Query: `name:String` | `ResourceItemResponse` (201) | `201`; `404` `BUSINESS_NOT_FOUND`; `400` `BOOKING_NOT_SUPPORTED` (tipo sin reservas) | `AddResourceServiceTest` |
| `DELETE /resources/{resourceId}` | `ResourceController.remove` (`:39`) | Baja de recurso | Path: `resourceId` | `204` | `204`; `404` `RESOURCE_NOT_FOUND`; `409` `RESOURCE_HAS_PENDING_BOOKINGS` **(por verificar código/HTTP)** | `AddResourceServiceTest` (parcial) |
| `GET /resources/{resourceId}/schedule` | `ResourceScheduleController.get` (`.../impl/ResourceScheduleController.java:21`) | Horario semanal del recurso | Path: `resourceId` | `ResourceScheduleResponse`: `resourceId`,`inheritsBusinessHours:boolean`,`intervals:[{day,startMin,endMin}]` | `200` | `ResourceScheduleServicesTest` |
| `PUT /resources/{resourceId}/schedule` | `ResourceScheduleController.update` (`:30`) | Reemplaza horario del recurso (se recorta al del negocio) | Body: `intervals:[{day,startMin,endMin}]` | `ResourceScheduleResponse` | `200`; `400` `VALIDATION_ERROR` | `ResourceScheduleServicesTest` |
| `GET /resources/{resourceId}/services` | `ResourceServicesController.get` (`.../impl/ResourceServicesController.java:21`) | Servicios que hace el recurso | Path: `resourceId` | `ResourceServicesResponse`: `resourceId`,`services:[ServiceItemResponse]` | `200` | `ResourceServicesServicesTest` |
| `PUT /resources/{resourceId}/services` | `ResourceServicesController.update` (`:30`) | Asigna servicios al recurso | Body: `serviceIds:List<String>` (NotNull, cada uno NotBlank) | `ResourceServicesResponse` | `200`; `400` `VALIDATION_ERROR` | `ResourceServicesServicesTest` |
| `POST /rewards/redeem?code=` | `RewardRedemptionController.redeem` | Canjea un premio por código (usa `RewardLookup`) | Query: `code:String` | `204` (**sin cuerpo**; descarta el `Reward`) | `204`; `404` `REWARD_NOT_FOUND`; `409` `REWARD_ALREADY_REDEEMED`; `410` `REWARD_EXPIRED`; `403` `REWARD_WRONG_BUSINESS` | `RewardLookupTest` (validación) |
| 🅱️ `GET /rewards/{code}` | `RewardRedemptionController.preview` → `GetRewardPreviewUseCase` (`RewardLookup`) | **Solo consulta**, no canjea: cliente + programa + premio | Path: `code:String` | `RewardPreviewResponse` (ver §3) | `200`; mismos `reason` que el canje: `404` `REWARD_NOT_FOUND`; `409` `REWARD_ALREADY_REDEEMED`; `410` `REWARD_EXPIRED`; `403` `REWARD_WRONG_BUSINESS` | `RewardLookupTest`, `RewardRedemptionControllerTest` |
| `GET /schedule-exceptions?from=&to=` | `ScheduleExceptionController.list` (`.../impl/ScheduleExceptionController.java:26`) | Lista excepciones (filtro opcional por rango) | Query: `from:String?`, `to:String?` | `List<ScheduleExceptionResponse>` | `200` | `ScheduleExceptionServicesTest` |
| `POST /schedule-exceptions` | `ScheduleExceptionController.create` (`:37`) | Crea excepción (negocio o recurso; día completo o parcial) | Body: `resourceId:String?`, `dateFrom:String`(yyyy-MM-dd), `dateTo:String?`(yyyy-MM-dd), `startMin:Integer?`, `endMin:Integer?`, `reason:String?`(≤255) | `ScheduleExceptionResponse` (201) | `201`; `400` `VALIDATION_ERROR` | `ScheduleExceptionServicesTest` |
| `DELETE /schedule-exceptions/{exceptionId}` | `ScheduleExceptionController.delete` (`:55`) | Elimina excepción | Path: `exceptionId` | `204` | `204`; `404` `SCHEDULE_EXCEPTION_NOT_FOUND` **(por verificar)** | `ScheduleExceptionServicesTest` (parcial) |
| `POST /stamps` | `StampController.add` → `AddStampService` (`CustomerResolver`) | Suma un sello; si completa meta, crea premio | Body: 🅱️ `customerIdentifier:String?` (celular **o** código `V-XXXXXX`) **o** `customerId:String?` (al menos uno), `programId:String` | `StampResponse`: `membershipId`,`programId`,`stampCount:int`,`unlockedReward:{code,status,expiresAt}` o `null` | `200`; `400` `VALIDATION_ERROR` (sin identificador); `404` `CUSTOMER_NOT_FOUND` / `PROGRAM_NOT_FOUND`; `403` `FORBIDDEN` (programa de otro negocio) | 🅱️ `AddStampServiceTest` |
| `GET /stats` | `BusinessStatsController.stats` → `GetBusinessStatsService` | Métricas del resumen | — | `{customersWithCard:long, stampsDelivered:long, rewardsPending:long, rewardsRedeemed:long, bookingsNextSevenDays:long, noShowPercent:int, 🆕 customersNearReward:long` (clientes distintos a los que faltan 1–2 sellos en un programa activo)`, 🆕 notificationsSent:long` (**0 fijo**: aún no existe la tabla de notificaciones; pendiente Fase D)`}` | `200` | `GetBusinessStatsServiceTest`, 🆕 `BusinessStatsControllerTest`, `GetBusinessStatsIntegrationTest` |

**`BookingResponse`** (`.../business/response/BookingResponse.java`): `id`, `customerId`,
`resourceId`, `serviceId`, `date:String`, `startMin:int`, `durationMin:int`, `price:Long`,
`status:BookingStatus`, `stamped:boolean`, `createdAt:LocalDateTime`,
🆕 `noShowAllowedFrom:LocalDateTime`, `attendedAllowedFrom:LocalDateTime` (rama
`fix/booking-status-timing`; calculados en el backend en la zona del negocio — ver §5).

**`BookingConfigResponse`** (`.../business/response/BookingConfigResponse.java`): `id`,
`openHour:int`, `closeHour:int`, `remindHours:int`, `depositAfterNoShows:int`,
🆕 `noShowGraceMin:int` (rama `fix/booking-status-timing`; minutos de gracia tras el inicio
para poder marcar NO_SHOW; default 10, rango 0–120),
`dayPrice:Long?`, `nightPrice:Long?`, `nightFromHour:Integer?`, `linkedProgramIds:List<String>`,
`resources:[ResourceItemResponse]`, `services:[ServiceItemResponse]`.

**`ProgramResponse`** (`.../business/response/ProgramResponse.java`): `id`, `businessId`,
`name`, `goal:int`, `unit:StampUnit`, `rewardType:RewardType`, `rewardValue:Long?`,
`rewardText:String?`, `expiryDays:int`, `resetOnComplete:boolean`, `status:ProgramStatus`,
`createdAt:LocalDateTime`.

**`ScheduleExceptionResponse`** (`.../business/response/ScheduleExceptionResponse.java`):
`id`, `resourceId:String?`, `dateFrom:String`, `dateTo:String`, `fullDay:boolean`
(derivado: `startMin == null`), `startMin:Integer?`, `endMin:Integer?`, `reason:String?`.

**Enums** (valores en SCREAMING_SNAKE_CASE):
`BookingStatus = CONFIRMED | PENDING_DEPOSIT | ATTENDED | NO_SHOW | CANCELLED`;
reasons de tiempo (rama `fix/booking-status-timing`): `NO_SHOW_TOO_EARLY`, `ATTENDED_TOO_EARLY` (ambos `409`);
`StampUnit = VISITS | BOOKINGS | PURCHASES | SERVICES`;
`RewardType = FREE_SERVICE | PERCENTAGE_DISCOUNT | SECOND_ITEM_DISCOUNT | AMOUNT_DISCOUNT | GIFT`;
`RewardStatus = AVAILABLE | REDEEMED`; `ProgramStatus = ACTIVE | INACTIVE`;
`ResourceKind = STAFF | COURT | TABLE`; `ResourceStatus`/`ServiceStatus = ACTIVE | INACTIVE`;
`Role = OWNER | BO_ADMIN`.

### 2.4 Seguridad del JWT (resumen)

- El filtro `JwtAuthFilter` (`infrastructure/security/JwtAuthFilter.java:59-71`) construye un
  `BusinessPrincipal(email, businessId)` a partir del token y le asigna `ROLE_<role>`.
- Claims emitidos (`infrastructure/security/JwtService.java:34-43`): `sub = email`,
  `role`, `businessId`, `iat`, `exp`. **No incluye `userId`** (el requerimiento del front
  §3 lo asume; en realidad el identificador es el email en `sub`).
- `/api/business/{bizId}/**` → valida coincidencia de `businessId` (ignore-case) **y**
  ✅ **(Fase A4)** que el token tenga `ROLE_OWNER`. Un token de negocio con otro rol → `403`.
- `/api/business/**` (sin `{bizId}`) → exige `ROLE_OWNER`.
- ✅ **(Fase A5)** `401`/`403` ya devuelven cuerpo JSON `{status, reason, source, timestamp}`
  (`JsonAuthenticationEntryPoint` / `JsonAccessDeniedHandler`), ver §3 y §5.
- Expiración del token del dueño: `app.jwt.expiration-ms = 86400000` = **24 h**
  (`application.yaml:48`). **No hay refresh token** para el dueño.

---

## 3. Ejemplos JSON

> Valores de ejemplo coherentes con los DTOs. Fechas `yyyy-MM-dd`; tiempos en minutos desde
> medianoche; dinero en pesos enteros (`Long`); timestamps `LocalDateTime` ISO sin offset,
> p. ej. `"2026-10-08T09:00:00"` (confirmado por los tests `@WebMvcTest` de la Fase A).
>
> **Actualización Fase A (2026-10-08):** los ejemplos marcados con 🆕/✏️ reflejan lo implementado
> en la rama `feature/business-app-phase-a`.

**`POST /api/auth/login`**
```json
// request
{ "email": "dueno@barberia.co", "password": "secret123" }
// response 200
{ "token": "eyJhbGciOiJIUzI1NiJ9...", "email": "dueno@barberia.co", "role": "OWNER", "businessId": "018f3a2b-...-", }
```

**`POST /api/business/{bizId}/stamps`**
```json
// request
{ "customerIdentifier": "3001234567", "programId": "018f3a40-...." }
// response 200 (completa la tarjeta)
{ "membershipId": "018f3a55-....", "programId": "018f3a40-....", "stampCount": 0,
  "unlockedReward": { "code": "A1B2C3D4", "status": "AVAILABLE", "expiresAt": "2026-11-07T10:15:00" } }
// response 200 (aún no completa)
{ "membershipId": "018f3a55-....", "programId": "018f3a40-....", "stampCount": 5, "unlockedReward": null }
```

**`POST /api/business/{bizId}/bookings`**
```json
// request — peluquería (serviceId obligatorio para este tipo)
{ "customerId": "018f3a30-....", "resourceId": "018f3a31-....", "serviceId": "018f3a32-....",
  "date": "2026-10-10", "startMin": 600 }
// response 201 — duración y precio salen del servicio
{ "id": "018f3a60-....", "customerId": "018f3a30-....", "resourceId": "018f3a31-....",
  "serviceId": "018f3a32-....", "date": "2026-10-10", "startMin": 600, "durationMin": 30,
  "price": 25000, "status": "CONFIRMED", "stamped": false, "createdAt": "2026-10-08T09:00:00" }

// request — cancha (sin serviceId; el bloque es de 60 min y el precio lo pone la cancha)
{ "customerId": "018f3a30-....", "resourceId": "018f3a33-....",
  "date": "2026-10-10", "startMin": 1080 }
// response 201 — startMin 1080 (18:00) cae en horario nocturno → nightPrice; serviceId = null
{ "id": "018f3a61-....", "customerId": "018f3a30-....", "resourceId": "018f3a33-....",
  "serviceId": null, "date": "2026-10-10", "startMin": 1080, "durationMin": 60,
  "price": 110000, "status": "CONFIRMED", "stamped": false, "createdAt": "2026-10-08T09:00:00" }
// error 409 — la hora quedó fuera de horario o ya está ocupada
{ "status": 409, "reason": "SLOT_NOT_AVAILABLE", "source": "BOOKING", "timestamp": "2026-10-08T09:00:00" }
```

**✏️ `PATCH /api/business/{bizId}/bookings/{bookingId}/status?status=ATTENDED`** (Fase A1 + 🆕 `fix/booking-status-timing`)
```json
// response 200 — atender que completa una tarjeta (inicio 10:00; ATENDIDA desde 09:00, NO_SHOW desde 10:10)
{ "id": "018f3a60-....", "customerId": "018f3a30-....", "resourceId": "018f3a31-....",
  "serviceId": "018f3a32-....", "date": "2026-10-10", "startMin": 600, "durationMin": 30,
  "price": 25000, "status": "ATTENDED", "stamped": true, "createdAt": "2026-10-08T09:00:00",
  "noShowAllowedFrom": "2026-10-10T10:10:00", "attendedAllowedFrom": "2026-10-10T09:00:00",
  "stampedMemberships": [ { "membershipId": "018f3a55-....", "programId": "018f3a40-....", "stampCount": 0 } ],
  "unlockedRewards": [ { "programId": "018f3a40-....", "programName": "Corte gratis",
    "code": "A1B2C3D4", "status": "AVAILABLE", "expiresAt": "2026-11-07T09:00:00" } ] }
// 🆕 error 409 — demasiado pronto para el estado pedido (reason NO_SHOW_TOO_EARLY o ATTENDED_TOO_EARLY)
{ "status": 409, "reason": "NO_SHOW_TOO_EARLY", "source": "BOOKING", "timestamp": "2026-10-10T10:00:00" }
```

**`GET /api/business/{bizId}/agenda?date=2026-10-10`**
```json
[ { "resourceId": "018f3a31-....", "resourceName": "Juan",
    "workingIntervals": [ { "startMin": 540, "endMin": 1080 } ], "unavailableReason": null,
    "bookings": [ { "id": "018f3a60-....", "customerId": "018f3a30-....", "customerName": "Laura",
      "resourceId": "018f3a31-....", "serviceId": "018f3a32-....", "serviceName": "Corte",
      "date": "2026-10-10", "startMin": 600, "durationMin": 30,
      "price": 25000, "status": "CONFIRMED", "stamped": false, "createdAt": "2026-10-08T09:00:00",
      "noShowAllowedFrom": "2026-10-10T10:10:00", "attendedAllowedFrom": "2026-10-10T09:00:00" } ] },
  { "resourceId": "018f3a33-....", "resourceName": "Pedro",
    "workingIntervals": [], "unavailableReason": "Vacaciones", "bookings": [] } ]
```

**🆕 `GET /api/business/{bizId}/agenda/summary?from=2026-10-19&to=2026-10-21`**
```json
[ { "date": "2026-10-19", "bookingCount": 2, "closed": false },
  { "date": "2026-10-20", "bookingCount": 1, "closed": true },
  { "date": "2026-10-21", "bookingCount": 0, "closed": true } ]
```

**`GET /api/business/{bizId}/booking-config`**
```json
{ "id": "018f3a70-....", "openHour": 8, "closeHour": 20, "remindHours": 24,
  "depositAfterNoShows": 2, "noShowGraceMin": 10, "dayPrice": 40000, "nightPrice": 60000, "nightFromHour": 18,
  "linkedProgramIds": ["018f3a40-...."],
  "resources": [ { "id": "018f3a31-....", "name": "Cancha 1", "status": "ACTIVE", "kind": "COURT", "capacity": null } ],
  "services": [] }
```

**`GET /api/business/{bizId}/business-hours`**
```json
{ "defaultHours": false,
  "intervals": [ { "day": "MONDAY", "startMin": 480, "endMin": 1200 },
                 { "day": "TUESDAY", "startMin": 480, "endMin": 1200 } ] }
```

**`POST /api/business/{bizId}/programs`**
```json
// request
{ "name": "Corte gratis", "goal": 10, "unit": "VISITS", "rewardType": "FREE_SERVICE",
  "rewardValue": null, "rewardText": "Un corte gratis", "expiryDays": 30, "resetOnComplete": true }
// response 201
{ "id": "018f3a40-....", "businessId": "018f3a2b-....", "name": "Corte gratis", "goal": 10,
  "unit": "VISITS", "rewardType": "FREE_SERVICE", "rewardValue": null, "rewardText": "Un corte gratis",
  "expiryDays": 30, "resetOnComplete": true, "status": "ACTIVE", "createdAt": "2026-10-08T09:00:00" }
```

**✏️ `GET /api/business/{bizId}/customers`** y **🅱️ `GET /api/business/{bizId}/customers?q=lau&limit=5`**
(fix `customer-list-contract`, 2026-10-14): **ambas devuelven exactamente el mismo cuerpo**.
```json
[ { "id": "018f3a30-....", "name": "Laura", "phone": "3001234567", "code": "V-4567A3",
    "noShows": 2, "bookingCount": 5,
    "memberships": [ { "membershipId": "018f3a55-....", "programId": "018f3a40-....",
                       "programName": "Corte gratis", "goal": 10, "stampCount": 4 } ] } ]
```
> Privacidad (solo con `?q=`): por **nombre** solo clientes con membresía o reservas en este negocio;
> por **celular completo** (10 díg.) o **código** (`V-XXXXXX`), en toda la plataforma con coincidencia
> exacta. Las `memberships` devueltas siempre se recortan a los programas de este negocio. Las
> inasistencias (`noShows`) y el conteo de reservas (`bookingCount`, reservas no canceladas en este
> negocio) se cargan cada uno en una sola consulta por lote (sin N+1).

**🅱️ `POST /api/business/{bizId}/customers`** (Fase B)
```json
// request
{ "name": "Laura", "phone": "3001234567" }
// response 201 (nuevo) — o 200 si el celular ya existía (devuelve ese cliente)
{ "id": "018f3a30-....", "name": "Laura", "phone": "3001234567", "code": "V-4567A3",
  "noShows": 0, "bookingCount": 0, "memberships": [] }
```

**🅱️ `GET /api/business/{bizId}/customers/{custId}`** (Fase B)
```json
{ "id": "018f3a30-....", "name": "Laura", "phone": "3001234567", "code": "V-4567A3", "noShows": 2,
  "memberships": [ { "membershipId": "018f3a55-....", "programId": "018f3a40-....",
                     "programName": "Corte gratis", "stampCount": 4, "goal": 10 } ],
  "rewards": [ { "code": "A1B2C3D4", "programId": "018f3a40-....", "programName": "Corte gratis",
                 "status": "AVAILABLE", "expiresAt": "2026-11-07T09:00:00", "redeemedAt": null } ],
  "bookings": [ { "id": "018f3a60-....", "date": "2026-10-10", "startMin": 600, "durationMin": 30,
                  "price": 25000, "status": "ATTENDED", "resourceId": "018f3a31-....",
                  "serviceId": "018f3a32-....", "stamped": true } ] }
```
> `rewards[].status` se calcula: un premio `AVAILABLE` ya vencido se devuelve como `EXPIRED`.
> `rewards` y `bookings` se recortan a este negocio; `bookings` ordenadas desc, últimas 10.

**🅱️ `GET /api/business/{bizId}/slots?date=2026-10-10&customerId=...`** (Fase B — mismo cuerpo que el del cliente)
```json
{ "date": "2026-10-10", "durationMin": 30, "depositRequired": false,
  "stampProgramNames": ["Corte gratis"],
  "slots": [ { "startMin": 600, "resourceId": "018f3a31-....", "price": 25000 },
             { "startMin": 630, "resourceId": "018f3a31-....", "price": 25000 } ] }
```

**🅱️ `GET /api/business/{bizId}/rewards/{code}`** (Fase B — solo consulta)
```json
// response 200 (premio disponible)
{ "code": "A1B2C3D4", "status": "AVAILABLE", "expiresAt": "2026-11-07T09:00:00",
  "customer": { "id": "018f3a30-....", "name": "Laura", "phone": "3001234567", "code": "V-4567A3" },
  "program": { "id": "018f3a40-....", "name": "Corte gratis" },
  "reward": { "rewardType": "FREE_SERVICE", "rewardValue": null, "rewardText": "Un corte gratis" } }
// error 410 (vencido) — mismos reason que el canje
{ "status": 410, "reason": "REWARD_EXPIRED", "source": "REWARD", "timestamp": "2026-10-09T09:00:00" }
```

**🅱️ `POST /api/business/{bizId}/stamps`** (Fase B — identificador por celular, código o id)
```json
{ "customerIdentifier": "3001234567", "programId": "018f3a40-...." }   // celular (como hasta ahora)
{ "customerIdentifier": "V-4567A3",   "programId": "018f3a40-...." }   // código de cliente
{ "customerId": "018f3a30-....",      "programId": "018f3a40-...." }   // id directo
```

**`POST /api/business/{bizId}/schedule-exceptions`**
```json
// request (cierre de día completo de todo el negocio)
{ "resourceId": null, "dateFrom": "2026-12-25", "dateTo": "2026-12-25",
  "startMin": null, "endMin": null, "reason": "Navidad" }
// response 201
{ "id": "018f3a80-....", "resourceId": null, "dateFrom": "2026-12-25", "dateTo": "2026-12-25",
  "fullDay": true, "startMin": null, "endMin": null, "reason": "Navidad" }
```

**🆕 `GET /api/business/{bizId}`** (Fase A2)
```json
{ "id": "018f3a2b-....", "name": "Barbería El Faro", "businessType": "HAIR_SALON",
  "address": "Cra 6 # 4-21, Centro", "phone": "3121234567",
  "description": "Barbería clásica con cortes modernos.", "offers": ["Cortes", "Barba"],
  "status": "ACTIVE", "timeZone": "America/Bogota",
  "capabilities": { "hasBooking": true, "hasStaff": true, "hasResources": true,
    "hasServices": true, "resourceKind": "STAFF", "stampUnit": "VISITS" } }
```

**🆕 `PUT /api/business/{bizId}`** (Fase A2)
```json
// request (businessType y status no son editables)
{ "name": "Barbería El Faro", "address": "Cra 6 # 4-21, Centro", "phone": "3121234567",
  "description": "Barbería clásica con cortes modernos.", "offers": ["Cortes", "Barba"] }
// response 200 → mismo cuerpo que el GET
// error 400 (p. ej. phone con != 10 dígitos o name vacío)
{ "status": 400, "reason": "VALIDATION_ERROR", "source": null, "timestamp": "2026-10-08T09:00:00" }
```

**✏️ `GET /api/business/{bizId}/stats`** (Fase A3 — `rewardsPending`/`rewardsRedeemed` ya calculados)
```json
{ "customersWithCard": 42, "stampsDelivered": 310, "rewardsPending": 6, "rewardsRedeemed": 12,
  "bookingsNextSevenDays": 18, "noShowPercent": 7, "customersNearReward": 5, "notificationsSent": 0 }
```
> `rewardsPending` = premios `AVAILABLE` no vencidos; `rewardsRedeemed` = premios `REDEEMED`,
> ambos del negocio. Siguen faltantes "clientes a punto de completar" y "avisos enviados" (Fase C).

**✏️ Formato de error de seguridad 401 / 403** (Fase A5 — antes venían sin cuerpo)
```json
// 401 — sin token, token vencido o firma inválida
{ "status": 401, "reason": "UNAUTHORIZED", "source": null, "timestamp": "2026-10-08T09:00:00" }
// 403 — token de otro negocio o rol distinto de OWNER
{ "status": 403, "reason": "FORBIDDEN", "source": null, "timestamp": "2026-10-08T09:00:00" }
```

**Referencia — slots del cliente** (lo que debería existir en el negocio, hoy solo en
`/api/customer/businesses/{businessId}/slots`, `.../customer/response/BookingAvailabilityResponse.java`):
```json
{ "date": "2026-10-10", "durationMin": 30, "depositRequired": false,
  "stampProgramNames": ["Corte gratis"],
  "slots": [ { "startMin": 600, "resourceId": "018f3a31-....", "price": 25000 },
             { "startMin": 630, "resourceId": "018f3a31-....", "price": 25000 } ] }
```

---

## 4. Cruce con el front

Estado por módulo (requisitos `requirements-frontend-mobile-business.md`):

| Módulo del front | Estado | Qué falta exactamente |
|---|---|---|
| **Login y sesión** (§3) | Parcial | Login OK, pero: sin `POST /api/auth/refresh` (el refresh solo existe para cliente); token único de **24 h**; el JWT no trae `userId` (usa `email` en `sub`); sin recuperación de contraseña del dueño. |
| **Capacidades por tipo de negocio** (§1) | **Completo** | ✅ **Fase A2:** por decisión (`backend-changes-business-app.md`) las capacidades se exponen dentro de `GET /api/business/{bizId}` en `capabilities` (`hasBooking`, `hasStaff`, `hasResources`, `hasServices`, `resourceKind`, `stampUnit`), derivadas de la `BookingStrategy` + `BusinessType.defaultStampUnit()`. No se creó `/api/config/business-types`. |
| **Agenda** (§4) | **Completo** | ✅ **Fase A1:** "Atendida" devuelve `stampedMemberships` y `unlockedRewards`. ✅ **Fase C:** `GET /agenda` agrega `workingIntervals`, `unavailableReason` (`OUT_OF_SCHEDULE`/excepción) y `customerName`/`serviceName`; `GET /agenda/summary?from=&to=` da el conteo por día y la bandera de cerrado para el selector. |
| **Nueva reserva** (§4) | ✅ **(Fase B)** | `POST /bookings` + 🅱️ `GET /slots` desde el negocio (reutiliza `GetBookingAvailabilityUseCase`, con `depositRequired` y nota de programas) + 🅱️ buscar/registrar cliente. |
| **Sellos y canje → Sellos** (§5) | ✅ **(Fase B)** | `POST /stamps` devuelve `unlockedReward` ✅ y 🅱️ ahora resuelve el cliente por **celular, código o `customerId`** (`CustomerResolver`). El registro de clientes nuevos se hace con 🅱️ `POST /customers`. **Decisión:** el código de premio se queda en **8 caracteres** (el front toma el largo de `RewardCode.length`). |
| **Sellos y canje → Canje** (§5) | ✅ **(Fase B)** | `POST /rewards/redeem` + 🅱️ `GET /rewards/{code}` para validar/previsualizar (cliente + programa + premio) antes de confirmar, con los mismos `reason` de error. |
| **Programas** (§6) | **Completo** | CRUD (`GET/POST/PUT/DELETE /programs`) + `PATCH /programs/{id}/status`. Cubre nombre, meta, unidad, tipo/valor/texto de premio, vigencia y `resetOnComplete`. "Suma con reservas" se deriva de `booking-config.linkedProgramIds`. |
| **Clientes** (§7) | **Completo** | `GET /customers` (lista) + 🅱️ búsqueda `?q=&limit=` (regla de privacidad) + 🅱️ `GET /customers/{custId}` (detalle con premios e historial). ✏️ **Fix `customer-list-contract`:** lista y búsqueda comparten el mismo DTO (con `noShows` y progreso con nombre/meta). ✅ **Fase C:** el DTO agrega `bookingCount` (reservas no canceladas por cliente, consulta agrupada sin N+1). `clear no-shows` sigue como `DELETE .../no-shows`. |
| **Resumen** (§8) | **Completo** | ✅ **Fase A3:** `rewardsPending` (disponibles no vencidos) y `rewardsRedeemed`. ✅ **Fase C:** `customersNearReward` (clientes a 1–2 sellos del premio en un programa activo) y `notificationsSent` (**0 fijo**: aún sin tabla de notificaciones, pendiente Fase D). |
| **Ajustes → Información del negocio** (§9) | **Completo** | ✅ **Fase A2:** `GET/PUT /api/business/{bizId}` con `phone`, `description`, `offers` (ordenadas) + capacidades y `timeZone`. Migración `V9__business_contact_fields.sql`. `PUT` no cambia `businessType` ni `status`. |
| **Ajustes → Horario** (§9) | **Completo** | `GET/PUT /business-hours`. |
| **Ajustes → Reservas** (§9) | **Completo** | `GET/PUT /booking-config` incluye `remindHours`, `depositAfterNoShows`, `linkedProgramIds`, y tarifas `dayPrice`/`nightPrice`/`nightFromHour`. |
| **Ajustes → Recursos** (§9) | Parcial | `GET/POST/DELETE /resources` + `schedule` + `services`. `POST /resources` solo recibe `name`; el `kind` se deriva del tipo de negocio y `capacity` queda `null` (no se puede fijar la capacidad de una mesa). |
| **Ajustes → Servicios** (§9) | **Completo** | `POST/DELETE /services` (parámetros por query, no body). Alta asigna el servicio al STAFF activo (`StaffServiceAssigner`). |
| **Ajustes → Excepciones** (§9) | **Completo** | `GET/POST/DELETE /schedule-exceptions`, negocio o recurso, día completo o parcial. |

### Revisión punto por punto de §10 "Dependencias de backend"

| Necesidad (§10) | Propuesta del front | Estado | Detalle |
|---|---|---|---|
| Editar información del negocio | `GET/PUT /api/business/{bizId}` + columnas `phone/description/offers` | ✅ **Resuelto (A2)** | Endpoint y columnas creados (migración `V9`). `offers` ordenada (`@OrderColumn`). |
| Horas libres desde el negocio | `GET /api/business/{bizId}/slots` | ✅ **Resuelto (B)** | `BusinessSlotsController` reutiliza `GetBookingAvailabilityUseCase` (mismo cálculo del cliente); `customerId` opcional para `depositRequired`. |
| Buscar cliente | `GET /api/business/{bizId}/customers?q=&limit=` | ✅ **Resuelto (B)** | Regla de privacidad: nombre solo dentro del negocio; celular/código exactos en toda la plataforma. Membresías recortadas al negocio. |
| Registrar cliente nuevo | `POST /api/business/{bizId}/customers` | ✅ **Resuelto (B)** | Celular existente → devuelve ese cliente (`200`); nuevo → lo crea sin correo/contraseña (`201`). |
| Progreso de un cliente | `GET /api/business/{bizId}/customers/{custId}` | ✅ **Resuelto (B)** | Detalle con membresías, premios (estado calculado), últimas reservas e inasistencias. |
| Validar un premio antes de canjear | `GET /api/business/{bizId}/rewards/{code}` | ✅ **Resuelto (B)** | Previsualiza cliente + programa + premio sin canjear; mismos `reason` que el canje (comparten `RewardLookup`). |
| Premio desbloqueado en la respuesta | `POST /stamps` y `PATCH /bookings/{id}/status` devuelven membresía + premio | ✅ **Resuelto (A1)** | `POST /stamps` ya devolvía `unlockedReward`. `PATCH .../status` ahora devuelve `BookingStatusResponse` con `stampedMemberships` y `unlockedRewards` (listas; una por programa activo vinculado). |
| Programas vinculados a reservas | `programIds` en `booking-config` | **Completo** | `linkedProgramIds` en `BookingConfigResponse`/`UpdateBookingConfigRequest`. |
| Tarifas de cancha | `dayPrice`, `nightPrice`, `nightFromHour` en `booking-config` | **Completo** | Presentes en request y response. |
| Recuperar contraseña del dueño | Flujo en `/api/auth/` | **Falta** | Solo existe para el cliente (`/api/customer/auth/password-reset`). |
| Sesión por dispositivo | Confirmar si el dueño usa `deviceId`/refresh rotado | **Confirmado: no (y así se queda)** | El dueño usa un token único de 24 h, sin `deviceId`, sin refresh, sin `sid`/`ver`. Esquema **distinto** al del cliente. **Decisión 2026-10-08:** se acepta el token de 24 h tal cual; el front cierra sesión al expirar. |

---

## 5. Consistencia del contrato

- **Fechas y horas.** Fechas como `String "yyyy-MM-dd"` en todos lados (bookings, agenda,
  excepciones). Horas en **minutos desde medianoche** (`startMin`, `endMin`, `durationMin`)
  en agenda, horarios, excepciones y slots. **Inconsistencia:** `booking-config` usa
  **horas enteras** (`openHour`, `closeHour`, `nightFromHour`), no minutos
  (`UpdateBookingConfigRequest.java`, `BookingConfigResponse.java`). El front maneja todo en
  minutos (guidelines §8), así que tendrá que convertir solo en esta pantalla.
- **Dinero.** Siempre `Long` en **pesos enteros** (sin decimales): `price`, `rewardValue`,
  `dayPrice`, `nightPrice`. Consistente. **(Unidad COP sin centavos — por verificar si se
  esperan centavos en algún punto; el código no multiplica por 100.)**
- **Enums.** Serializados por nombre en SCREAMING_SNAKE_CASE, consistente en todos los
  endpoints.
- **Formato de error `{status, reason, source, timestamp}`.** Sí para errores de negocio
  (`ApiException`) y de validación, vía `GlobalExceptionHandler`
  (`shared/exception/ErrorResponse.java`, `GlobalExceptionHandler.java:17-50`). Matices:
  - En validación `source` llega **`null`** (`reason = VALIDATION_ERROR`).
  - `timestamp` es `LocalDateTime` **sin zona/offset**, serializado como string ISO
    (p. ej. `"2026-10-08T09:00:00"`); confirmado por los tests `@WebMvcTest` de la Fase A.
  - ✅ **Fase A5:** los **errores de Spring Security (401/403)** ahora **sí** devuelven el
    cuerpo estándar. `401 UNAUTHORIZED` (sin token, vencido o firma inválida) vía
    `JsonAuthenticationEntryPoint`; `403 FORBIDDEN` (otro negocio o rol distinto de OWNER) vía
    `JsonAccessDeniedHandler`. En ambos `source = null`.
- **Zona horaria del negocio.** Existe como **constante global** `America/Bogota`
  (`infrastructure/config/TimeConfig.java:11`), fijada como default de la JVM; **no es por
  negocio** (no hay columna `timezone` en `business`). "Hoy" se calcula con
  `LocalDate.now()` en esa zona (p. ej. `GetBusinessStatsService.java` para "próximos 7
  días" y "últimos 30 días").
- **Sumar sello / marcar atendida.** `POST /stamps` devuelve `membership` + `unlockedReward`
  ✅. ✅ **Fase A1:** "Atendida" (`PATCH /bookings/{id}/status`) ahora devuelve
  `stampedMemberships` + `unlockedRewards` (listas, una entrada por programa **activo**
  vinculado; una reserva ya sellada no vuelve a sumar).
- **Idempotencia / peticiones duplicadas.** **No existe.** No hay `Idempotency-Key` ni
  deduplicación; cada `POST /stamps` incrementa, cada `POST /bookings` crea. El único rate
  limit es en autenticación (`infrastructure/web/ratelimit/AuthRateLimitFilter.java`).
- **Hora ya tomada.** `409 SLOT_NOT_AVAILABLE` cuando el recurso ya está ocupado;
  `409 CUSTOMER_BOOKING_OVERLAP` si el mismo cliente se solapa (`CreateBusinessBookingService.java`).
- **Paginación en listados.** **No hay** en `GET /customers` ni en `GET /agenda`: devuelven
  listas completas.
- **Esquema de sesión del dueño.** Access token JWT único de **24 h**
  (`app.jwt.expiration-ms=86400000`, `application.yaml:48`), firmado HMAC, con
  `sub=email/role/businessId`. **Sin refresh, sin `deviceId`, sin sesiones.** Contrasta con
  el cliente: access de **15 min** (`customer-access-expiration-ms=900000`) + refresh
  rotado + `sid`/`ver` y listado/revocación de sesiones.

---

## 6. Cambios recomendados (priorizados por lo que bloquea primero)

> **Estado Fase A (2026-10-08, rama `feature/business-app-phase-a`):** resueltos los ítems
> #1 (capacidades, dentro de `GET /api/business/{bizId}`), #5, #7, parcialmente #10 (conteos de
> premios), más A4 (exigir rol OWNER) y A5 (cuerpo JSON en 401/403), que afectan a §5.
>
> **Estado Fase B (2026-10-09, rama `feature/business-app-phase-b` sobre la A):** resueltos #3
> (buscar/registrar clientes + identificador por celular/código), #4 (`GET /slots`), #6
> (`GET /rewards/{code}`) y #8 (`GET /customers/{custId}`). El #11 queda **decidido**: código de
> premio en **8** (sin cambio de generador).
> Pendientes: #2 (decisión, sin trabajo), #12–#15 (Fase D). #8, #9 y #10 ✅ hechos en la Fase C.

| # | Cambio propuesto | Motivo | Pantalla/módulo que desbloquea |
|---|---|---|---|
| 1 | ✅ **Hecho (A2):** capacidades expuestas en `GET /api/business/{bizId}` → `capabilities` (`hasBooking`, `hasStaff`, `hasResources`, `hasServices`, `resourceKind`, `stampUnit`). Se descartó `/api/config/business-types` por decisión del doc de cambios | Sin capacidades la app no decide pestañas ni Ajustes; se carga al iniciar | **Todo el arranque** (navegación, §1-§2) |
| 2 | **Sesión del dueño: DECIDIDO — se acepta el token único de 24 h tal cual.** El front cierra sesión al expirar y re-pide login; no se implementa refresh ni `deviceId`. Pendiente solo confirmar si se necesita `userId` en el claim (hoy el `sub` es el email) | Decisión tomada (2026-10-08): cero trabajo backend; la app maneja el vencimiento | Login y sesión (§3) |
| 3 | ✅ **Hecho (B):** `GET /customers?q=&limit=` (regla de privacidad) + `POST /customers` (crea o devuelve existente) + `POST /stamps` acepta celular, **código** o `customerId` (`CustomerResolver`) | Flujo de mostrador no funciona sin encontrar/crear al cliente | Sellos (§5), Nueva reserva (§4) |
| 4 | ✅ **Hecho (B):** `GET /api/business/{bizId}/slots` reutiliza `GetBookingAvailabilityUseCase` (mismo `BookingAvailabilityResponse`), `customerId` opcional | No hay horas libres desde el negocio | Nueva reserva (§4) |
| 5 | ✅ **Hecho (A1):** `PATCH /bookings/{id}/status` devuelve `BookingStatusResponse` con `stampedMemberships` y `unlockedRewards` | Corrección pequeña; habilita el sheet de premio al atender | Agenda → Atendida (§4) |
| 6 | ✅ **Hecho (B):** `GET /api/business/{bizId}/rewards/{code}` previsualiza (cliente + programa + premio) sin canjear; mismos `reason` que el canje (comparten `RewardLookup`) | El canje necesita confirmación con datos antes del `POST` | Canje (§5) |
| 7 | ✅ **Hecho (A2):** `GET/PUT /api/business/{bizId}` + columnas `phone`, `description`, `offers` (migración `V9`) | No hay forma de ver/editar la info del negocio | Ajustes → Información (§9) |
| 8 | ✅ **Hecho (B + C):** `GET /customers/{custId}` (membresías, premios con estado, últimas reservas, inasistencias) + **Fase C:** `bookingCount` por cliente en lista/búsqueda (consulta agrupada) | Detalle de cliente y métrica de reservas del front | Clientes (§7) |
| 9 | ✅ **Hecho (C):** `GET /agenda` agrega `workingIntervals`/`unavailableReason` (`OUT_OF_SCHEDULE` o excepción) y `customerName`/`serviceName`; `GET /agenda/summary?from=&to=` da conteo por día y bandera de cerrado (selector de 7 días) | Antes ignoraba horario y no daba conteos | Agenda (§4) |
| 10 | ✅ **Hecho (A3 + C):** `rewardsPending`/`rewardsRedeemed` (A3) + **Fase C:** `customersNearReward` (1–2 sellos del premio) y `notificationsSent` (0 fijo; falta tabla de notificaciones, Fase D) | Métricas del Resumen | Resumen (§8) |
| 11 | ✅ **Decidido (B):** código de premio se queda en **8 caracteres** (sin cambio de generador); el front toma el largo de `RewardCode.length`. Pendiente aún: considerar unificar `booking-config` a minutos desde medianoche | Evita desajustes de validación/formato | Canje (§5), Ajustes → Reservas (§9) |
| 12 | **Idempotencia** (`Idempotency-Key`) en `POST /stamps` y `POST /bookings` | En mostrador el doble toque es frecuente | Sellos (§5), Nueva reserva (§4) |
| 13 | **Recuperación de contraseña del dueño** en `/api/auth/` | Hoy solo existe para el cliente | Login (§3) |
| 14 | **Alinear `clear no-shows`**: el front/guidelines esperan `PATCH /customers/{id}/no-shows/clear`; hoy es `DELETE /customers/{id}/no-shows` | Evitar confusión de contrato | Clientes (§7) |
| 15 | Permitir fijar `capacity` (y quizás `kind`) al crear recurso tipo mesa | `POST /resources` solo recibe `name`; capacidad queda `null` | Ajustes → Recursos/Mesas (§9) |
