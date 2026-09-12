# ADR-0010 — Usar el generador de Spring Boot de IntelliJ, con revisión obligatoria del POM

**Estado:** Aceptado · **Fecha:** 2026-09-12 · **Fase:** F00
**Reemplaza:** la instrucción "no uses Spring Initializr" de la primera versión de `fase-00-entorno.md`

## Contexto
La Fase 00 original exigía escribir el `pom.xml` a mano, sin asistentes, para forzar la comprensión
de cada línea. El estudiante señaló que uno de los objetivos declarados del proyecto es
**aprovechar al máximo las ayudas de las herramientas** (IntelliJ, DBeaver), y que rechazar el
asistente contradice ese objetivo.

El argumento es correcto: en un entorno laboral real, nadie escribe el `pom.xml` inicial a mano.
Se genera con Spring Initializr (directamente o desde el IDE) y **luego se revisa y se ajusta**.
Prohibirlo enseña una práctica que no existe en el trabajo.

## Opciones consideradas
1. **Escribir el POM completamente a mano** — máxima exposición a la sintaxis XML; pero es una
   práctica que no se usa profesionalmente y consume tiempo en algo de bajo valor.
2. **Generar con el asistente y no revisar** — rápido, pero deja un `pom.xml` que el estudiante no
   entiende: exactamente la dependencia que el proyecto busca eliminar.
3. **Generar con el asistente + revisión línea por línea obligatoria** — se usa la herramienta como
   en el trabajo real, y la comprensión se garantiza con una revisión explícita.

## Decisión
**Opción 3.** Se usa `New Project → Spring Boot` de IntelliJ IDEA Ultimate.

Condiciones obligatorias tras generar:
1. Revisar el `pom.xml` generado línea por línea y poder explicar cada elemento.
2. Ajustar a mano lo que el asistente no acierta: `maven.compiler.release`, versión del proyecto,
   convenciones de nombres.
3. Marcar **solo** las dependencias de la fase actual. Las demás se agregan a mano en su fase,
   que es donde se aprende para qué sirven.
4. No marcar Lombok (ADR-0004) ni DevTools (añade comportamiento implícito que estorba al aprender).

## Consecuencias

### Positivas
- Coherente con el objetivo de dominar las herramientas.
- Refleja el flujo de trabajo profesional real.
- La estructura de directorios, el Maven Wrapper y el `.gitignore` quedan correctos desde el inicio.

### Negativas (aceptadas)
- Se pierde la práctica de teclear el XML inicial — de escaso valor frente al tiempo que cuesta.
- Riesgo de aceptar valores por defecto sin entenderlos → mitigado por la revisión obligatoria y
  por las preguntas de control de la Fase 00.

### Nota general
El mismo criterio aplica al resto del proyecto: **el asistente que genera andamiaje está permitido;
el asistente que escribe lógica de negocio, no.** La línea está en quién toma las decisiones.
