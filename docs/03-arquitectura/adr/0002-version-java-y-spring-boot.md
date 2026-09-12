# ADR-0002 — Versiones de Java y Spring Boot

**Estado:** Aceptado (revisado el 2026-09-12) · **Fase:** F00

## Contexto

En la máquina hay JDK 17, 21 y 25. Al abrir el generador de Spring Boot de IntelliJ se comprobó
un hecho decisivo: **start.spring.io ya no ofrece ninguna versión 3.x**. Las únicas versiones
disponibles el 2026-09-12 son:

```
4.2.0 (SNAPSHOT) · 4.2.0 (M1) · 4.1.2 (SNAPSHOT) · 4.1.1 (por defecto) · 4.0.9 (SNAPSHOT) · 4.0.8
```

Spring retira de Initializr las líneas que salen del soporte comunitario (OSS). Que 3.5.x haya
desaparecido significa que esa línea ya solo recibe soporte comercial.

Además se verificó que `spring-boot-starter-parent:4.1.1` declara `java.version=17` como mínimo y
hereda `maven.compiler.release=${java.version}`, por lo que Java 21 es plenamente compatible.

## Decisión anterior (descartada)

La primera versión de este ADR elegía **Java 21 + Spring Boot 3.5.x**, con el argumento de que era
la combinación dominante en producción corporativa y la que más material de consulta tiene, dejando
la migración a 4.x como ejercicio final (Fase 18).

**Por qué se descarta:** iniciar un proyecto nuevo, de varios meses de duración, sobre una línea
fuera de soporte comunitario es un mal consejo de ingeniería. El argumento del material de consulta
no compensa arrancar con deuda de versión el día uno.

## Decisión

**Java 21 (LTS) + Spring Boot 4.1.1.**

- Java 21 y no 25: es LTS, es la versión mayoritaria en producción, y cubre todo el Java moderno
  que interesa aprender (records, sealed, pattern matching, virtual threads).
- Spring Boot 4.1.1: es la versión estable actual y la que el generador propone por defecto.

## Consecuencias

### Positivas
- El proyecto nace en una versión soportada y actual.
- `maven.compiler.release` viene heredado del parent: un ajuste manual menos.
- Lo que se aprenda es directamente aplicable a proyectos nuevos del mercado.

### Negativas (aceptadas)
- **Buena parte del material de la comunidad (tutoriales, respuestas de StackOverflow, cursos)
  sigue escrito para Spring Boot 3.x.** Habrá diferencias al copiar ejemplos de internet.
  - *Mitigación*: la fuente primaria de este proyecto es la **documentación oficial de Spring Boot 4
    y las notas de migración 3.x → 4.x**, no los tutoriales. Cuando aparezca una diferencia, se
    verifica contra la documentación oficial y se anota en la bitácora de la fase.
  - Esto es, además, una habilidad profesional en sí misma: trabajar con una versión más nueva que
    el material disponible es la situación normal en el mercado.

### Efecto sobre el roadmap
La **Fase 18** dejaba de tener sentido como "migración a 4.x". Se redefine como
**mantenimiento evolutivo**: actualizar dependencias, leer notas de versión, resolver
incompatibilidades apoyándose en la batería de pruebas y saldar deuda técnica. Se conserva el
objetivo pedagógico original —actualizar un sistema real confiando en sus pruebas— con un alcance
más realista.

### Cómo se revertiría
Cambiar la versión del `<parent>` a 3.5.16 y resolver las diferencias de API. Barato al inicio del
proyecto, caro más adelante: por eso se decide ahora.
