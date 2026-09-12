# ADR-0002 — Versiones de Java y Spring Boot

**Estado:** Aceptado · **Fecha:** 2026-09-12 · **Fase:** F00

## Contexto
En la máquina hay JDK 17 y JDK 25 (Temurin). En Maven Central, la última versión estable de
`spring-boot-starter-parent` es **4.1.1** (verificado el 2026-09-12), y la línea 3.5.x sigue siendo
la más extendida en producción corporativa. El objetivo del proyecto es doble: aprender la sintaxis
y el ecosistema, y parecerse a lo que el estudiante encontrará en un entorno laboral.

## Opciones consideradas

1. **Java 25 + Spring Boot 4.1.x** — lo más actual. Contra: menos material de referencia, muchos
   ejemplos de la comunidad aún en 3.x, y cambios de API (Spring Framework 7, nullability con
   JSpecify, cambios en el cliente HTTP) que añaden fricción a quien está recuperando sintaxis.
2. **Java 21 + Spring Boot 3.5.x** — combinación dominante hoy en banca y fintech. Ecosistema de
   documentación, tutoriales y respuestas enorme. Java 21 trae records, patrones, sealed y virtual
   threads: todo lo moderno que interesa aprender.
3. **Java 17 + Spring Boot 3.2** — demasiado conservador; se perdería parte del Java moderno.

## Decisión
**Java 21 (Temurin) + Spring Boot 3.5.x**, y una **fase final dedicada a migrar a Spring Boot 4.x**
(Fase 18).

La migración de versión mayor no es un rodeo: es una tarea real y frecuente en el trabajo. Hacerla
al final, con el proyecto cubierto de pruebas, es el mejor escenario posible para aprenderla — y
deja el proyecto en la versión actual.

## Consecuencias

### Positivas
- Máxima disponibilidad de material de consulta mientras se recupera fluidez.
- Java 21 cubre todo el lenguaje moderno relevante.
- La Fase 18 enseña a migrar versiones apoyándose en la batería de pruebas.

### Negativas
- Requiere instalar Temurin 21 (los JDK 17 y 25 presentes no se usan para este proyecto).
- Durante el desarrollo se trabaja una versión por detrás de la última.

### Notas
- Se fija la versión del JDK en el `pom.xml` (`maven.compiler.release`) para que el build no dependa
  del JDK por defecto del sistema.
- Gestión de múltiples JDK con SDKMAN! o con la configuración de JDK de IntelliJ (ver Fase 00).
