# ADR-0006 — Migraciones con Flyway y `ddl-auto=validate`

**Estado:** Aceptado · **Fecha:** 2026-09-12 · **Fase:** F03

## Contexto
Hibernate puede generar el esquema automáticamente (`ddl-auto=update`/`create`). Es cómodo en
tutoriales y desastroso en producción.

## Opciones consideradas
1. **`ddl-auto=update`** — cero esfuerzo inicial. Contra: no versionado, no reproducible, no
   revisable, no reversible; genera índices y tipos subóptimos; jamás se usa en producción seria.
2. **Flyway con migraciones SQL versionadas + `ddl-auto=validate`** — el esquema es código, revisable
   en el PR, reproducible en cualquier entorno; Hibernate solo verifica que el mapeo coincide.
3. **Liquibase** — equivalente, con XML/YAML además de SQL. Más abstracto.

## Decisión
**Flyway con migraciones en SQL puro** y `spring.jpa.hibernate.ddl-auto=validate` en todos los
entornos, incluidos los de prueba.

SQL puro (y no el formato abstracto de Liquibase) porque este proyecto también quiere enseñar SQL
real de PostgreSQL: índices parciales, índices funcionales, `CHECK`, `JSONB`.

## Consecuencias
- Hay que escribir el DDL a mano: **es un objetivo de aprendizaje, no un inconveniente**.
- `validate` hace que un desajuste entre entidad y tabla rompa el arranque — falla rápido y claro.
- Reglas: una migración aplicada no se modifica nunca; cada cambio es una nueva versión.
- Las pruebas de integración corren las mismas migraciones sobre PostgreSQL real (Testcontainers),
  con lo que el esquema queda probado en cada build.
