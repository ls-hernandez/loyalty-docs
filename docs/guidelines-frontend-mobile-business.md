# Lineamientos — Frontend Mobile Negocios (Flutter)

Aplican a `mobile/loyalty_business` (app del dueño del negocio). Parten de `guidelines-frontend-mobile.md`:
todo lo que este documento no cambia se mantiene igual que en la app cliente.
Fuente de verdad visual: `Vuelve — Design System para Flutter.md` + `design-system-business.md`.
Contrato real del backend: `docs/api-business-audit.md`.

> Referencia visual y de comportamiento: `docs/prototypes/vuelve-negocios.html` (abrir en el navegador).
> Si el prototipo y estos documentos no coinciden, mandan los documentos.
> Del prototipo NO se toma: las cuentas de prueba, el selector "Teléfono / Tablet", los datos de ejemplo,
> el uso de localStorage ni la lógica de negocio calculada en el cliente (horas libres, premios, inasistencias):
> en la app real eso lo decide el backend.

## 1. Proyecto

| Elemento | Valor |
|---|---|
| Carpeta | `mobile/loyalty_business/` |
| Paquete Dart | `loyalty_business` |
| Android applicationId | `com.qodev.loyalty_business` |
| iOS bundle id | `com.qodev.loyaltyBusiness` |
| Nombre visible | `AppBrand.name` = "Vuelve Negocios" |
| Flutter / Dart | 3.38 / 3.10 mínimo (`sdk: ^3.10.0`), probado con 3.41.5 / 3.11.3 (Material 3) |
| Estado | `flutter_riverpod` 3.x (sin code-gen) |
| Navegación | `go_router` (`StatefulShellRoute` para las pestañas) |
| HTTP | `dio` |
| Formatos | `intl`, locale `es_CO` |
| Íconos | `lucide_icons_flutter` |
| Orientación | Vertical en teléfono; vertical y horizontal en tablet |

## 2. Código compartido entre las dos apps

El design system y la infraestructura transversal no se copian: se extraen a paquetes locales del monorepo
que usan las dos apps.

```
mobile/
├── packages/
│   ├── loyalty_design_system/   # tokens, tema, componentes (lo que hoy es lib/design_system/)
│   └── loyalty_core/            # network, session base, formatters, validation, ErrorReason
├── loyalty_customer/            # app cliente
└── loyalty_business/            # app negocios
```

- Las apps dependen de los paquetes con `path:` en `pubspec.yaml`, dentro de un workspace de pub
  (`resolution: workspace`).
- **El design system no tiene textos.** Todo componente recibe sus etiquetas como parámetros (`String`), y cada
  app las saca de su propio `.arb`. Así el paquete no depende de la l10n de ninguna app.
- `loyalty_core` no tiene UI. Contiene: `dioProvider`, `TraceInterceptor`, `ApiException`, `ErrorReason`,
  `Formatters`, `Validators`, `RetryPolicy` y `TokenStorage` base.
- Lo propio de cada app (rutas, shell, `AuthInterceptor`, `SessionController`, `AppBrand`) se queda en la app.
- Un cambio en un paquete compartido se prueba en las dos apps antes del commit.
- Los paquetes se crearon junto con `loyalty_business`. Cuando la app cliente entre al workspace, su
  `lib/design_system/` se reemplaza por el paquete en un commit `[refactor]`, sin cambios de comportamiento.

## 3. Naming

- Igual que en la app cliente: **loyalty** en código y la marca solo en `AppBrand`, `android:label` y
  `CFBundleDisplayName`.
- Prefijo `L` en componentes del design system (`LButton`, `LChip`, `LSheet`). El documento de diseño todavía
  usa el prefijo `V` (`VButton`): en código se traduce a `L`.
- Componentes firma sin prefijo (`StampCard`, `BookingTile`) y los nuevos de esta app (`DayStrip`,
  `WeeklyScheduleEditor`, `RewardUnlockedSheet`).

## 4. Arquitectura (feature-first)

```
lib/
├── main.dart
├── app/
│   ├── app.dart
│   ├── router/               # app_router.dart, app_routes.dart
│   └── shell/                # BusinessShell: header + tab bar o NavigationRail
├── core/
│   ├── config/               # Env, AppBrand
│   ├── business/             # BusinessContext: businessId, tipo y capacidades
│   ├── l10n/                 # context.l10n, ErrorReasonL10n, BusinessTypeL10n, ResourceKindL10n
│   └── session/              # OwnerSessionController, AuthInterceptor
└── features/
    ├── auth/                 # login, PIN, desbloqueo
    ├── agenda/               # día, recursos, acciones de reserva, nueva reserva
    ├── stamps/               # buscar cliente, sumar sello, canjear premio
    ├── programs/             # CRUD de programas
    ├── customers/            # lista, detalle, perdonar inasistencias
    ├── stats/                # resumen
    └── settings/             # negocio, horarios, reservas, recursos, servicios, excepciones, cuenta
```

Cada feature tiene `<feature>_dependencies.dart`, `domain/`, `data/` y `presentation/` con las mismas reglas de
dependencias que la app cliente. Una feature no importa otra: lo compartido sube a `core/` o a los paquetes.

### Mapeo con el backend

| Feature | Endpoints (`/api/business/{bizId}` salvo indicación) | Estado |
|---|---|---|
| `auth` | `POST /api/auth/login` | Listo (sin refresh) |
| negocio (`core/business`) | `GET /` y `PUT /` sobre `/api/business/{bizId}` | Listo (fase A) e implementado (Etapa 5) |
| `agenda` | `GET /agenda?date=yyyy-MM-dd`, `PATCH /bookings/{id}/status?status=`, `POST /bookings` | Listo; atendida devuelve `stampedMemberships` y `unlockedRewards` (fase A) |
| `agenda` | `GET /slots` | Falta (fase B) |
| `stamps` | `POST /stamps` (body `customerIdentifier`, `programId`), `POST /rewards/redeem?code=` | Listo; identificador solo por celular |
| `stamps` | `GET /rewards/{code}`, `GET /customers?q=`, `POST /customers` | Falta (fase B) |
| `programs` | `GET/POST/PUT/DELETE /programs`, `PATCH /programs/{id}/status?targetStatus=` | Listo |
| `customers` | `GET /customers`, `DELETE /customers/{custId}/no-shows` | Listo, sin búsqueda ni paginación |
| `customers` | `GET /customers/{custId}` | Falta (fase B) |
| `stats` | `GET /stats` | Listo (premios corregidos en fase A) |
| `settings` | `GET/PUT /booking-config`, `GET/PUT /business-hours`, `GET/POST/DELETE /resources`, `GET/PUT /resources/{id}/schedule`, `GET/PUT /resources/{id}/services`, `POST/DELETE /services`, `GET/POST/DELETE /schedule-exceptions` | Listo |

Los pendientes están detallados en `requirements-frontend-mobile-business.md` §10.

## 5. Negocio actual y capacidades

- `businessContextProvider` (en `core/business/`) expone `businessId`, `BusinessType`, `timeZone` y las
  capacidades. Se carga tras el login con `GET /api/business/{bizId}` y antes de mostrar el shell.
- `RemoteBusinessRepository` implementa `BusinessRepository` sobre ese endpoint (Etapa 5). La tabla local
  temporal ya no existe.
- **Colores e íconos de categoría:** salen de la paleta del documento de diseño (peluquería `#F2B630`, cancha
  `#8FD3B6`, tienda `#F4A7B0`, restaurante `#A9C4F5`, café `#E3C29B`), expuestos como extensión de `BusinessType`
  (`cardColor`, `icon`). No se mapean a tokens semánticos como `teal` o `sun`.
- "Hoy" se calcula con la zona horaria del negocio (`timeZone`, hoy `America/Bogota` fija en el backend),
  no con la del dispositivo.
- Los datasources toman el `businessId` de ahí. **Nunca** de un parámetro de la UI ni de una ruta.
- Qué pestañas y pantallas de Ajustes aparecen se decide con las banderas (`hasBooking`, `hasResources`,
  `hasServices`), nunca con un `switch` sobre el tipo. Así un tipo nuevo en el backend no obliga a cambiar la app.
- El nombre visible del recurso viene de `ResourceKind` traducido en `core/l10n/` (`STAFF` → "Profesionales").
- Un `403` por `businessId` que no coincide cierra la sesión: el token es de otro negocio.

## 6. Sesión

- `OwnerSessionController` con los mismos 3 estados que la app cliente: `signedOut` → `/login`,
  `locked` → `/unlock`, `signedIn`.
- Si el claim `role` no es `OWNER`, se trata como `signedOut` y se muestra `errorNotOwner`.
- **Token único de 24 h, sin refresh** (decisión 2026-10-08). Se guarda en `flutter_secure_storage`.
  `OwnerSessionController` lee `exp` del JWT y programa el cierre de sesión al vencer; al volver a primer plano
  revisa si ya venció.
- `AuthInterceptor` solo agrega `Authorization: Bearer`. Ante `401` cierra la sesión (no intenta refresh).
  Ante un `403` de seguridad en `/api/business/{bizId}/**` también cierra la sesión; un `403` de negocio
  (por ejemplo `REWARD_WRONG_BUSINESS`) no la cierra.
- El JWT no trae `userId`: el usuario se identifica por `sub` (correo).
- PIN local opcional de 4 dígitos, ofrecido tras el primer login, y bloqueo tras 5 min en segundo plano.
- **Bloqueo con PIN en un controller aparte (`AppLockController`), no como estado de `SessionState`**
  (decisión 2026-10-19). La sesión (`OwnerSessionController`) sigue con sus 3 estados de token; el PIN es
  una capa independiente con su propio estado (`unlocked` / `locked` / espera). El `redirect` del router
  combina ambos: `SignedOut → /login`; `SignedIn` **y bloqueado → `/unlock`**; `SignedIn` y desbloqueado → la
  app. Así la sesión y los datos del negocio no dependen del PIN.
  - El PIN se guarda **hasheado** (PBKDF2-HMAC-SHA256, sal aleatoria, cálculo en un `Isolate`, comparación en
    tiempo constante) en `flutter_secure_storage`, nunca en claro; las iteraciones viajan con el hash.
  - Se ofrece **una sola vez** tras el primer login (con "Ahora no"); si se omite, se marca y queda disponible
    en Ajustes → Cuenta → "Crear PIN".
  - Intentos fallidos: espera de 30 s con cuenta regresiva en el 3.º y 4.º; en el 5.º, `signOut` completo
    (borra token, PIN, intentos, espera y la marca de "ofrecido") y vuelta al login. Al desbloquear se vuelve a
    la ruta donde estaba el usuario.
  - Reloj inyectable para los tests (umbral de 5 min en segundo plano, espera de 30 s).
- Si más adelante el backend agrega refresh (fase D), se reutiliza el esquema de la app cliente
  (`TokenStorage` y el refresh único con reintento) desde `loyalty_core`.

## 6.1 Contrato con el backend

- **Fechas:** `String` `yyyy-MM-dd`. **Horas:** minutos desde medianoche (`startMin`, `endMin`, `durationMin`),
  salvo `booking-config`, que usa horas enteras (`openHour`, `closeHour`, `nightFromHour`): se convierte en su DTO.
- **Días de la semana:** `DayOfWeek` (`MONDAY` … `SUNDAY`), mapeados a un enum Dart.
- **Dinero:** entero en pesos (`price`, `rewardValue`, `dayPrice`, `nightPrice`), sin centavos. En Dart, `int`.
- **Enums:** en `SCREAMING_SNAKE_CASE`, mapeados a enums Dart en camelCase en el DTO:
  `BookingStatus`, `StampUnit` (`VISITS`, `BOOKINGS`, `PURCHASES`, `SERVICES`), `RewardType` (`FREE_SERVICE`,
  `PERCENTAGE_DISCOUNT`, `SECOND_ITEM_DISCOUNT`, `AMOUNT_DISCOUNT`, `GIFT`), `ProgramStatus`, `ResourceKind`,
  `ResourceStatus`, `ServiceStatus`, `Role`.
- **Fechas con hora** (`createdAt`, `expiresAt`, `timestamp`): `LocalDateTime` sin zona. Se interpretan en la
  zona del negocio.
- **Errores:** `{status, reason, source, timestamp}`. En `VALIDATION_ERROR`, `source` llega `null`. Desde la fase A,
  `401` (`UNAUTHORIZED`) y `403` (`FORBIDDEN`) también traen ese cuerpo; `ApiException` sigue aceptando respuestas
  vacías por si acaso y decide la sesión por el código HTTP.
- **Query params en vez de body** en `POST /resources?name=`, `POST /services?name=&durationMin=&price=`,
  `PATCH /bookings/{id}/status?status=`, `PATCH /programs/{id}/status?targetStatus=` y `POST /rewards/redeem?code=`.
- **Sin paginación** en `GET /customers` ni `GET /agenda`: llegan listas completas.
- **Sin idempotencia:** los botones de sellar, canjear y reservar se deshabilitan mientras la petición está en curso.
- **Fixtures:** los ejemplos JSON de `docs/api-business-audit.md` §3 se copian a `test/fixtures/` y cada DTO tiene
  un test que los parsea.

## 7. Layout responsive

| Ancho | Navegación | Contenido |
|---|---|---|
| < 600 px | `NavigationBar` inferior | Una columna, margen 20 |
| 600–839 px | `NavigationBar` inferior | Una columna con ancho máximo 640, centrada |
| ≥ 840 px | `NavigationRail` a la izquierda | Lista y detalle lado a lado en Agenda y Clientes |

- Los breakpoints viven en `LBreakpoints` (design system), no en las features.
- Bottom sheets en tablet: ancho máximo de 560, centrados.

## 8. Reglas propias de esta app

- **Acciones del mostrador primero.** Sumar sello y marcar atendida no pueden tardar más de un toque tras
  encontrar al cliente: sin pantallas intermedias.
- **Toda acción que avisa al cliente pide confirmación** (cancelar reserva, confirmar abono) y el diálogo lo dice:
  "Se le avisará a Laura."
- **Respuestas que desbloquean premio:** si la respuesta de `stamps` o de `bookings/{id}/status` trae un premio,
  se muestra `RewardUnlockedSheet`. La app no calcula si se completó la tarjeta.
- Tras una mutación se invalidan los providers afectados. Por ejemplo, atender una reserva invalida `agenda`,
  `customers` y `stats`. Para avisar entre features sin importarse se usa una señal en `core/`
  (`businessDataChangesProvider`).
- Los editores largos (horario semanal, excepciones) son pantallas completas dentro de Ajustes, con
  "Guardar" fijo abajo. Si hay cambios sin guardar, salir pide confirmación.
- Las horas se manejan en minutos desde medianoche (`startMin`, `endMin`) igual que el backend, y solo se
  formatean en la UI con `Formatters`.

## 9. i18n

Igual que la app cliente (`gen-l10n`, `app_es.arb` como plantilla, `app_en.arb`). Prefijos de clave por feature:

| Feature | Prefijo | Ejemplo |
|---|---|---|
| agenda | `agenda` | `agendaFreeAllDay`, `agendaPendingCount` |
| stamps | `stamps` | `stampsAdded`, `stampsRedeemInvalid` |
| programs | `programs` | `programsDeleteConfirm` |
| customers | `customers` | `customersForgiveNoShows` |
| stats | `stats` | `statsNoShowRate` |
| settings | `settings` | `settingsBusinessHours`, `settingsExceptionReason` |

Plurales con ICU: `"agendaPendingCount": "{count, plural, =0{Sin reservas} =1{1 reserva} other{{count} reservas}}"`.

## 10. Widgets, estado, modelos, código, testing y commits

Igual que `guidelines-frontend-mobile.md` §4, §6, §7.1, §9, §10 y §11. Adicional:

- Widget test de `DayStrip`, `WeeklyScheduleEditor` y `RewardUnlockedSheet` en el paquete del design system.
- Unit test de la lógica de franjas del editor de horario: solapamientos, franjas fuera del horario del
  negocio y días cerrados.
- Test de navegación: un negocio sin `hasBooking` no muestra Agenda y abre en Sellos.

## Checklist de PR

- [ ] Ningún archivo de widget supera 300 líneas
- [ ] Sin colores, tamaños ni fuentes literales fuera del paquete `loyalty_design_system`
- [ ] El design system no contiene textos visibles: los recibe por parámetro
- [ ] Ninguna feature importa otra feature
- [ ] "Vuelve" no aparece fuera de `AppBrand` y los nombres de plataforma
- [ ] Sin textos visibles literales: todo en `.arb` (es y en)
- [ ] Pestañas y ajustes condicionados por capacidades, no por `BusinessType`
- [ ] `businessId` solo desde `businessContextProvider`
- [ ] Estados vacío, carga y error cubiertos
- [ ] Probado en teléfono y en tablet (≥ 840 px)
- [ ] `flutter analyze` y `flutter test` en verde en las dos apps si se tocó un paquete compartido
