# Loyalty (Vuelve)

Plataforma de fidelización para comercios. Monorepo local:
- `loyalty-backend/`: Java + Spring Boot, arquitectura hexagonal, PostgreSQL, Flyway.
- `mobile/`: Flutter (workspace de pub). App `loyalty_business` (dueños de negocio) y paquetes
  `packages/loyalty_design_system` y `packages/loyalty_core`. La app cliente la hace otro compañero.
- `docs/`: fuente de verdad. Antes de cambiar algo, lee el documento que aplica.

## Documentos
- Requerimientos de la app de negocios: `docs/requirements-frontend-mobile-business.md`
- Lineamientos de Flutter: `docs/guidelines-frontend-mobile-business.md` (hereda de `docs/guidelines-frontend-mobile.md`)
- Design system: `docs/Vuelve — Design System para Flutter.md` + `docs/design-system-business.md`
- Contrato real del backend: `docs/api-business-audit.md`
- Cambios pendientes del backend: `docs/backend-changes-business-app.md`
- Prototipo (solo referencia visual y de comportamiento): `docs/prototypes/vuelve-negocios.html`

## Reglas
- Si un documento y el prototipo no coinciden, manda el documento.
- La lógica de negocio (horas libres, premios, inasistencias) la decide el backend; el front no la recalcula.
- Antes de implementar algo grande, muestra un plan y espera aprobación.
- Un commit por paso: `[feat]`, `[fix]`, `[refactor]`, `[chore]`, `[docs]` + descripción corta en inglés.
- Nunca hagas `git push`.
- Flutter: `flutter analyze` sin warnings y `flutter test` en verde antes de cada commit.
- Backend: antes de cada commit, `spotlessApply` y `spotlessCheck` en verde además de los tests.
- Código en inglés; textos de la UI en español, siempre desde los `.arb`.

## Ramas
- Backend y mobile usan el mismo esquema: `main` (estable), `development` (integración) y una
  rama `feature/<nombre>` o `fix/<nombre>` por tarea, creada desde `development` actualizado.
- Nunca se trabaja ni se hace commit directo en `main` ni en `development`.
- Al terminar una tarea: tests en verde y merge de la rama a `development`, solo cuando yo lo apruebe.
- Nunca `git push`.