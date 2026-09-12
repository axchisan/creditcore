# ADR-0007 — Mapeo manual entre modelos (sin MapStruct por ahora)

**Estado:** Aceptado · **Fecha:** 2026-09-12 · **Fase:** F04

## Contexto
La arquitectura hexagonal implica tres representaciones del mismo concepto: DTO de API, modelo de
dominio y entidad JPA. Hay que traducir entre ellas.

## Opciones consideradas
1. **MapStruct** — genera los mappers en compilación. Rápido y sin boilerplate. Contra: el código
   generado es invisible mientras no se inspeccione `target/`; los errores de mapeo aparecen como
   fallos de compilación crípticos; oculta conversiones que aquí son parte del aprendizaje
   (`BigDecimal` → `Dinero`, enum → string).
2. **Mapeo manual en clases `*Mapper` dedicadas** — explícito, depurable, sin magia.
3. **Constructores/factories en las propias clases** — mezcla responsabilidades.

## Decisión
**Mapeo manual en clases `*Mapper` dedicadas**, una por dirección y por capa.

Se reevaluará (nuevo ADR) si el volumen de mapeos se vuelve inmanejable; el momento natural sería
después de la Fase 11.

## Consecuencias
- Más código que escribir: coherente con el objetivo del proyecto.
- Los mappers son clases normales y **se prueban con pruebas unitarias** — lo que además detecta
  campos olvidados.
- Al depurar, se puede poner un breakpoint dentro del mapeo y ver exactamente qué se transforma.
- Riesgo: olvidar mapear un campo nuevo. Mitigación: prueba de mapeo que compara todos los campos.
