# ADR-0003 — Maven como herramienta de build

**Estado:** Aceptado · **Fecha:** 2026-09-12 · **Fase:** F00

## Contexto
El entorno tiene Maven 3.9.16 y Gradle 9.6.1. Hay que elegir uno.

## Opciones consideradas
1. **Maven** — XML declarativo y verboso; ciclo de vida fijo; dominante en entornos corporativos
   Java (banca, seguros, gobierno). Todo es explícito.
2. **Gradle (Kotlin DSL)** — más conciso y rápido (caché, builds incrementales); DSL programable.
   Más común en startups y Android; su magia esconde detalles que este proyecto quiere mostrar.

## Decisión
**Maven.**

La verbosidad del `pom.xml` es aquí una **ventaja pedagógica**: obliga a leer y entender cada
dependencia, cada plugin y cada fase del ciclo de vida. Además es lo que el estudiante encontrará
con más probabilidad en un empleo en el sector financiero colombiano.

## Consecuencias
- Builds algo más lentos que con Gradle: irrelevante a esta escala.
- El estudiante debe dominar el ciclo de vida (`validate → compile → test → package → verify →
  install → deploy`) y comandos como `mvn dependency:tree`, `mvn help:effective-pom`.
- Se usará el **Maven Wrapper** (`mvnw`) para fijar la versión del build.
