# ADR-0008 — PostgreSQL real en pruebas con Testcontainers

**Estado:** Aceptado · **Fecha:** 2026-09-12 · **Fase:** F05

## Contexto
Las pruebas de integración necesitan una base de datos. Las opciones habituales son H2 en memoria o
un contenedor con el motor real.

## Opciones consideradas
1. **H2 en modo compatibilidad PostgreSQL** — arranque instantáneo. Contra: **no es PostgreSQL**.
   No soporta índices parciales, `JSONB`, tipos ni funciones específicas; el SQL que pasa en H2 puede
   fallar en producción. Da una falsa sensación de seguridad.
2. **PostgreSQL local compartido** — real, pero el estado se contamina entre ejecuciones y no es
   reproducible en CI.
3. **Testcontainers con la imagen oficial de PostgreSQL 17** — el mismo motor que en producción,
   aislado y reproducible, tanto local como en CI.

## Decisión
**Testcontainers con PostgreSQL 17**, con contenedor reutilizable entre clases de prueba
(patrón *singleton container*) para no pagar el arranque en cada clase.

Esta decisión es la razón por la que el proyecto requiere un runtime de contenedores (OrbStack).

## Consecuencias
- Las pruebas de integración validan el SQL real, incluidas las migraciones Flyway.
- Se prueban de verdad las restricciones (`UNIQUE` parcial, `CHECK`) que implementan reglas de negocio.
- Cada ejecución tarda unos segundos más: aceptable, y mitigado con el contenedor singleton.
- Requiere Docker disponible en local y en CI (GitHub Actions lo soporta de fábrica).
- Las pruebas unitarias de dominio **no** usan Testcontainers: siguen corriendo en milisegundos.
