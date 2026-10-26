# Stack tecnológico — Vuelve

| Capa | Tecnología |
|------|-----------|
| App móvil | Flutter |
| Backend | Java + Spring Boot |
| Base de datos | Por definir (PostgreSQL recomendado) |
| Frontend web (fase 2) | Angular |

## Orden de desarrollo
1. Backend (Java/Spring) — API REST
2. App móvil (Flutter) — cliente y admin
3. Frontend web (Angular) — panel admin web (fase posterior)

## Comunicación
- Mobile y web consumen la misma API REST
- Notificaciones: in-app por polling (extensible a FCM/SMS)
