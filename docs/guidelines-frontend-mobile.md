# Lineamientos — Frontend Mobile (Flutter)

Aplican a `mobile/loyalty_customer` (app del cliente) y a cualquier app móvil futura (`mobile/loyalty_business`).
Complementan `Vuelve — Design System para Flutter.md`, que es la fuente de verdad visual.

## 1. Proyecto

| Elemento | Valor |
|---|---|
| Carpeta | `mobile/loyalty_customer/` |
| Paquete Dart | `loyalty_customer` |
| Android applicationId | `com.qodev.loyalty_customer` |
| iOS bundle id | `com.qodev.loyaltyCustomer` |
| Flutter / Dart | 3.38 / 3.10 (Material 3) |
| Estado | `flutter_riverpod` 3.x (sin code-gen) |
| Navegación | `go_router` (`StatefulShellRoute` para las pestañas) |
| HTTP | `dio` |
| Formatos | `intl`, locale `es_CO` |
| Íconos | `lucide_icons_flutter` |

## 2. Naming: `loyalty` en código, marca solo en lo visible

- Todo el código usa **loyalty**: clases (`LoyaltyApp`, `LoyaltyTokens`), archivos (`loyalty_theme.dart`) y paquetes.
- La marca comercial ("Vuelve") **solo** aparece en lo que ve el usuario y vive en un único lugar:
  - `lib/core/config/app_brand.dart` → `AppBrand.name`
  - `android:label` en `AndroidManifest.xml` y `CFBundleDisplayName` en `Info.plist`
- Cambiar la marca = tocar esos 3 puntos. Nunca escribir "Vuelve" literal en un widget.
- Componentes del design system con prefijo **`L`**: `LButton`, `LChip`, `LSheet`, `LSpace`, `LRadius`.
  Componentes firma sin prefijo: `StampCard`, `GlowCard`, `BookingStepper`, `BookingTile`.

## 3. Arquitectura por dominio (feature-first)

```
lib/
├── main.dart                 # bootstrap: locale, ProviderScope
├── app/                      # composición de la app
│   ├── app.dart              # MaterialApp.router + temas
│   ├── router/               # app_router.dart, app_routes.dart
│   └── shell/                # MainShell (header + tab bar)
├── core/                     # transversal, sin UI
│   ├── config/               # Env (--dart-define), AppBrand
│   ├── domain/               # enums compartidos (BusinessType)
│   ├── format/               # Formatters (moneda, hora, día relativo)
│   ├── l10n/                 # context.l10n, mapeo de errores y enums
│   ├── network/              # dioProvider, AuthInterceptor, ApiException, ErrorReason
│   ├── session/              # SessionController (JWT), TokenStorage (secure storage)
│   └── validation/           # Validators de formularios (mensajes desde l10n)
├── design_system/            # ÚNICA fuente de estilos y widgets base
│   ├── theme/                # tokens, spacing, text, theme, category_style
│   ├── components/           # buttons/ feedback/ layout/ navigation/ ...
│   └── design_system.dart    # barrel: un solo import
└── features/
    └── <feature>/            # auth, cards, booking, rewards, explore, notifications
        ├── <feature>_dependencies.dart  # wiring: provider del repositorio (único que importa data/)
        ├── domain/
        │   ├── entities/     # modelos inmutables del negocio
        │   └── repositories/ # contratos abstractos
        ├── data/
        │   ├── models/       # DTOs: fromJson/toJson + toEntity()
        │   ├── datasources/  # llamadas HTTP con Dio
        │   └── repositories/ # implementación del contrato
        └── presentation/
            ├── providers/    # Notifier / AsyncNotifier de Riverpod
            ├── pages/        # una pantalla por archivo
            └── widgets/      # piezas privadas de la feature
```

### Regla de dependencias

```
presentation → domain ← data
        ↓
   design_system, core
```

- `domain` no importa Flutter, Dio ni JSON. Solo Dart puro.
- `presentation` nunca importa `data/` directamente: usa el provider de `<feature>_dependencies.dart`,
  que expone el repositorio como su interfaz de `domain`.
- **Una feature no importa a otra feature.** Lo compartido sube a `core/` (lógica) o `design_system/` (UI).
  La navegación entre features se hace por ruta (`AppRoutes`), no por import de páginas.
- Solo `app/router` conoce las páginas de todas las features.
- Use cases solo si la lógica combina varios repositorios o tiene reglas no triviales;
  si solo delega al repositorio, el Notifier llama al repositorio.

### Mapeo con el backend

| Feature | Endpoints |
|---|---|
| `auth` | `/api/auth/` |
| `cards`, `rewards`, `booking`, `explore`, `notifications` | `/api/customer/` (pendiente en backend) |
| `config` compartida | `/api/config/` |

## 4. Widgets

- **Máximo 250–300 líneas por archivo de widget.** Si crece, se extraen sub-widgets.
- Un widget público por archivo. Los sub-widgets privados (`_Logo`) son válidos si el archivo cumple el límite;
  si se reutilizan dentro de la feature van a `presentation/widgets/`.
- `build()` corto y legible: si un bloque necesita comentario para entenderse, es un widget aparte.
- Preferir `StatelessWidget` / `ConsumerWidget`. `StatefulWidget` solo para estado local efímero
  (controllers de animación, `TextEditingController`, `PageController`).
- Prohibidos los métodos `Widget _buildX()`: se extraen como clases (rebuilds selectivos y `const`).
- Constructores `const` siempre que se pueda.
- Las páginas componen; la lógica vive en providers. Una página no llama a Dio ni parsea JSON.

## 5. Design system (obligatorio)

- **Todo widget usa el design system.** Nada de `Color(0x…)`, `Colors.*`, `fontSize` ni `EdgeInsets` con números sueltos en features.
  - Colores: `context.tokens.<token>` o constantes de `LoyaltyTokens`.
  - Texto: `context.text.<estilo>` y `copyWith` solo para variar peso o color.
  - Espaciado y radios: `LSpace`, `LRadius`, `LSize`.
  - Categoría de comercio: `businessType.cardColor` / `.icon` / `.label`.
- Si falta un componente, se crea en `design_system/components/` siguiendo la sección *Componentes* del
  documento de diseño, **no** dentro de la feature.
- Importar con el barrel: `import '.../design_system/design_system.dart';`.
- Elevación 0, sin sombras; formularios en bottom sheet; feedback con `LToast`.
- Decisión de producto (2026-10-02): en `StampCard` la casilla sellada es un círculo **sólido `cardInk`**
  (reemplaza la "perforada" en `background` del design system, que en modo claro parecía vacía), con su
  **número visible** en el color de categoría de la propia tarjeta (contraste ≥ 8:1 sobre `cardInk`), igual
  que las casillas vacías, para contar de un vistazo. La última casilla (premio) mantiene el regalo, sin número.
- Respetar `disableAnimations`, objetivos táctiles de 44 px y escala de texto hasta 1,3×.

## 6. Estado (Riverpod)

- `Provider` para dependencias (Dio, repositorios), `NotifierProvider` / `AsyncNotifierProvider` para estado de pantalla.
- Los providers de una feature viven en `features/<feature>/presentation/providers/`.
- Estados de carga y error con `AsyncValue.when(...)`; cada pantalla maneja vacío, carga y error.
- `ref.watch` en `build`, `ref.read` en callbacks.
- Filtros y derivaciones baratas (búsqueda, categoría) se calculan en el widget o en el modelo
  (`filter.apply(list)`), **no** con un `Provider` derivado de un `FutureProvider`: en Riverpod 3,
  al volver a una pestaña pausada el provider derivado se invalida durante el build y lanza
  `setState() called during build`.
- Reintentos: `ProviderScope(retry: RetryPolicy.retry)` (`core/network/retry_policy.dart`). Errores HTTP
  no se reintentan (se muestra el error con "Reintentar" al instante); solo fallos de red, máx 2 veces.
  El retry por defecto de Riverpod 3 dejaba la pantalla en "cargando" ~40 s ante un 404.
- Comunicación entre features sin importarse: señales en `core/` (ej. `membershipChangesProvider`:
  Explorar la emite al unirse; Tarjetas, Premios y Reservar la escuchan para recargar).
- Si el backend expone una decisión de negocio (ej. `hasBooking`), el app la usa tal cual; no la deduce.
- Tras una mutación (ej. unirse a un programa) se hace `ref.invalidate` de los providers afectados;
  el `AsyncValue` conserva el dato anterior mientras recarga, sin parpadeo.

## 7. Sesión y autenticación

- Endpoints cliente: `POST /api/customer/auth/register`, `/login` y `/password-reset/confirm` envían `deviceId` + `deviceName`
  y responden access (15 min) + refresh rotado (60 días deslizantes); el resto de `/api/customer/**` usa `Authorization: Bearer`.
- `SessionStorage` (`flutter_secure_storage`) guarda access, refresh, `deviceId` (UUID por instalación), PIN (PBKDF2 + salt),
  intentos, huella y "Ahora no"; se lee en `main()` antes de arrancar. Si la lectura falla se borra todo y se pide login.
- `sessionProvider` (core) es la única fuente de verdad con 3 estados: `signedOut` → `/login`, `locked` → `/unlock`,
  `signedIn` (sin PIN y sin "Ahora no" → `/pin/create`). `SessionLifecycleGuard` bloquea tras 5 min en segundo plano.
- `AuthInterceptor`: ante 401 de una petición con Bearer hace un solo refresh a la vez y reintenta una vez;
  `SESSION_EXPIRED` → `signOut`. 403 y 400 no tocan la sesión. Cerrar sesión llama `/auth/logout` (best effort).
- PIN local opcional de 4 dígitos (rechaza triviales), 5 fallos → se cierra la sesión (espera de 30 s en los fallos 3–4).
- Los controllers de formularios son `Notifier<AsyncValue<void>>` autoDispose; tras un `await`
  verifican `ref.mounted` antes de asignar estado (la redirección puede desmontar la pantalla).

## 7.1 Modelos y red

- Entidades: clases inmutables `final` con `copyWith` (equivalente a los records con `withXxx()` del backend).
- DTOs en `data/models/` con `fromJson`/`toJson` manuales y `toEntity()`.
- Enums del backend (`HAIR_SALON`, `ACTIVE`…) se mapean a enums Dart en camelCase en el DTO.
- Errores: el backend responde `{status, reason, source, timestamp}`. Los datasources convierten
  `DioException` en `ApiException` (con `ErrorReason` tipado) y la UI muestra `exception.message(context.l10n)`.
- Trazabilidad: `TraceInterceptor` envía un `qodevTraceId` nuevo en cada petición (el backend lo reutiliza
  y lo devuelve); en debug imprime `[api] GET /ruta 200 45ms trace=…`. `ApiException.traceId` lo conserva
  incluso sin respuesta, para buscar la petición en los logs del backend.
- URL base por `--dart-define=API_BASE_URL=...` (default emulador Android: `http://10.0.2.2:8080`).

## 8. Internacionalización (i18n)

Se usa `gen-l10n` oficial de Flutter (`flutter_localizations` + `intl`), configurado en `l10n.yaml`.

```
lib/l10n/app_es.arb          # plantilla (fuente de verdad), español
lib/l10n/app_en.arb          # inglés
lib/l10n/app_localizations*.dart   # GENERADO — no editar a mano
lib/core/l10n/l10n.dart      # context.l10n + mapeos (errores, categorías)
```

- **Ningún texto visible literal en widgets.** Todo label, título, botón, vacío o error va al `.arb`
  y se usa con `context.l10n.<clave>`.
- Claves en camelCase con prefijo de feature o tipo: `cardsEmpty`, `bookingTitle`, `tabRewards`, `errorSlotNotAvailable`.
- Nueva clave: se agrega primero en `app_es.arb` y luego en todos los demás `.arb`. `flutter gen-l10n`
  (o `flutter run`/`build`) regenera; una clave faltante rompe la compilación.
- Variables y plurales con ICU, nunca concatenando strings:
  `"stampsRemaining": "{remaining, plural, =0{¡Lista!} =1{Te falta 1} other{Te faltan {remaining}}}"`.
- **Errores del backend:** cada valor de `ErrorReason` (espejo del enum Java) se mapea a una clave
  `errorXxx` en `ErrorReasonL10n`. Si el backend agrega un reason, se agrega al enum Dart, al `.arb` y al `switch`
  (el `switch` exhaustivo obliga a hacerlo). Códigos desconocidos caen en `errorUnknown`; sin respuesta, `errorNetwork`.
- Enums de dominio con texto visible (ej. `BusinessType`) se traducen con una extensión en `core/l10n/`, no en el enum.
- Idioma: el del dispositivo si está soportado; si no, español.
- La marca (`AppBrand.name`) **no** va al `.arb`: no se traduce.
- Formatos de moneda y fecha siguen en `Formatters` (`es_CO`), independientes del idioma.

## 9. Código y estilo

- Código en inglés; textos de UI en español (tuteo, formato de oración).
- Sin comentarios innecesarios.
- `dart format` + `flutter analyze` sin warnings antes de cada commit (`analysis_options.yaml` con reglas estrictas).
- Archivos `snake_case.dart`, clases `PascalCase`, nombres explícitos (no `r`, `s`, `res`).
- Formatos de moneda y fecha solo vía `Formatters`.

## 10. Testing

- Widget test por cada componente del design system y por cada página principal.
- Unit test para mapeos DTO ↔ entidad y Notifiers.
- Estructura de `test/` espejo de `lib/`.

## 11. Commits

Mismo formato que el backend: `[feat]`, `[fix]`, `[refactor]`, `[chore]`, `[docs]` + descripción corta en inglés,
pequeños y atómicos. Ramas `development` y `master`.

## Checklist de PR

- [ ] Ningún archivo de widget supera 300 líneas
- [ ] Sin colores, tamaños ni fuentes literales fuera de `design_system/`
- [ ] Ninguna feature importa otra feature
- [ ] "Vuelve" no aparece fuera de `AppBrand` y los nombres de plataforma
- [ ] Sin textos visibles literales: todo en `.arb` (es y en)
- [ ] Estados vacío, carga y error cubiertos
- [ ] `flutter analyze` y `flutter test` en verde
