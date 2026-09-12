# ADR-0005 — Identificadores UUID v7 generados en la aplicación

**Estado:** Aceptado · **Fecha:** 2026-09-12 · **Fase:** F03

## Contexto
Hay que decidir el tipo de clave primaria: secuencial de base de datos o UUID; y quién la genera.

## Opciones consideradas
1. **`BIGSERIAL` de PostgreSQL** — compacto, índices eficientes. Contra: el objeto no tiene identidad
   hasta que se persiste (fuerza `flush`), expone el volumen de negocio al exterior y complica
   pruebas y sistemas distribuidos.
2. **UUID v4 generado en la aplicación** — identidad antes de persistir, no revela nada. Contra:
   aleatorio, fragmenta los índices B-tree y empeora la localidad de escritura.
3. **UUID v7 generado en la aplicación** — ordenado por tiempo; conserva las ventajas del v4 y
   recupera buena parte de la eficiencia de índice.

## Decisión
**UUID v7 generado en la aplicación**, almacenado en columnas `UUID` de PostgreSQL.

## Consecuencias
- Los agregados son válidos y probables antes de tocar la base de datos.
- Las URLs no filtran cuántos créditos existen.
- 16 bytes por clave en vez de 8: aceptable a esta escala.
- Se necesita una utilidad de generación de UUID v7 (Java 21 no la trae en `java.util.UUID`; se
  implementa en el kernel compartido o se usa una librería pequeña). **Es un buen ejercicio de
  manipulación de bits** en la Fase 03.
- Para el usuario final, además del UUID técnico, los créditos y solicitudes tienen un **número de
  negocio legible** (`numero`), que es lo que se muestra y se comunica.
