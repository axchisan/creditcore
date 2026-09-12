# ADR-0004 — No usar Lombok

**Estado:** Aceptado · **Fecha:** 2026-09-12 · **Fase:** F00

## Contexto
Lombok genera getters, setters, constructores, `equals`, `hashCode`, `toString` y builders mediante
anotaciones y procesamiento en tiempo de compilación. Es muy popular en proyectos Java.

## Opciones consideradas
1. **Usar Lombok** — menos código repetitivo, ficheros más cortos.
2. **No usarlo** — todo explícito; se escribe más.
3. **Usarlo solo en entidades JPA** — punto intermedio.

## Decisión
**No usar Lombok en este proyecto.**

Razones:
- El objetivo declarado es **recuperar la sintaxis a mano**. Lombok oculta exactamente el código
  que se quiere practicar: constructores, `equals`/`hashCode`, inmutabilidad.
- `record` de Java 21 cubre la mayor parte del boilerplate para VO, DTO y eventos, de forma nativa.
- Lombok tiene fricciones reales conocidas: `@Data` en entidades JPA rompe `equals`/`hashCode` con
  relaciones lazy; `@Builder` permite construir objetos inválidos saltándose invariantes;
  depende de procesadores de anotaciones y del soporte del IDE.
- En el dominio, generar setters automáticamente **empuja al modelo anémico**, justo lo que la
  arquitectura elegida busca evitar.

## Consecuencias

### Positivas
- El código es exactamente lo que se lee; el depurador entra en cada método.
- Se practica `equals`/`hashCode`, constructores y factory methods.
- Los objetos de dominio solo exponen el comportamiento que deben exponer.

### Negativas (aceptadas)
- Clases más largas, sobre todo las entidades JPA.
- Más tecleo — que es, literalmente, el objetivo.

### Mitigación
IntelliJ genera getters, constructores y `equals`/`hashCode` con `⌘N` (Generate). Se usa esa
generación del IDE: enseña el código real, queda en el fichero y es revisable.

### Revisión
Si en algún momento el boilerplate de las entidades JPA resulta insoportable, se reevaluará
**solo para el paquete de persistencia**, nunca para el dominio, con un nuevo ADR.
