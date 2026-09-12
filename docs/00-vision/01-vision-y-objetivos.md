# 01 — Visión y objetivos

## 1. El problema que resuelve este proyecto

Años de experiencia delegando la escritura de código a agentes producen un efecto medible:

- Se conserva el **criterio conceptual** (sé qué es una transacción, qué es hexagonal, qué es un índice).
- Se pierde la **fluidez de producción** (no recuerdo la sintaxis exacta de `@ManyToOne`, no sé dónde
  poner el breakpoint, me quedo en blanco frente a un editor vacío).
- Se pierde la **capacidad de depuración a mano** (leer un stacktrace de 200 líneas y saber en qué
  línea 7 está la causa raíz).

La consecuencia real: dependencia. En una entrevista técnica, en un pair programming, o cuando el
agente se equivoca y hay que corregirlo, la diferencia entre "sé de esto" y "sé hacer esto" se vuelve
visible.

## 2. Objetivo general

Construir **a mano**, de cero y en el orden real de un proyecto profesional, una plataforma de
originación y gestión de crédito con facturación electrónica y pagos, de forma que al terminar el
estudiante pueda:

1. Abrir un editor vacío y escribir una aplicación Spring Boot completa sin plantillas ni asistentes.
2. Leer código Java/Spring ajeno y entenderlo sin traducirlo mentalmente.
3. Depurar con breakpoints, watches y stacktraces, no con `System.out.println` a ciegas.
4. Diseñar una arquitectura y defender por qué eligió esa y no otra.
5. Escribir pruebas que de verdad protejan el negocio.
6. Manejar PostgreSQL con criterio (índices, planes de ejecución, transacciones).
7. Entender el negocio financiero, la facturación electrónica DIAN y las pasarelas de pago.
8. Desplegar la aplicación en AWS con infraestructura como código.

## 3. Objetivos de aprendizaje por área

### 3.1 Sintaxis y lenguaje Java

| Tema | Nivel objetivo |
|---|---|
| Clases, interfaces, herencia, polimorfismo | Escribir sin consultar |
| `record`, `sealed`, `enum` con comportamiento | Escribir sin consultar |
| Genéricos, wildcards, tipos comodín | Leer y escribir casos comunes |
| Streams, `Optional`, lambdas, referencias a método | Escribir sin consultar |
| Excepciones (checked/unchecked), `try-with-resources` | Escribir sin consultar |
| Colecciones y su complejidad | Elegir con criterio |
| `BigDecimal` y aritmética monetaria | Dominio total (es un proyecto financiero) |
| Fechas: `LocalDate`, `Instant`, `ZoneId` | Dominio total |
| Concurrencia básica, `@Async`, virtual threads | Comprensión y uso guiado |

### 3.2 Spring Boot

| Tema | Nivel objetivo |
|---|---|
| Contenedor IoC, beans, ciclo de vida, inyección | Explicar y depurar |
| Autoconfiguración y `spring.factories` / `AutoConfiguration.imports` | Explicar por qué "funciona solo" |
| Perfiles, `@ConfigurationProperties`, externalización | Escribir sin consultar |
| Spring MVC: controladores, binding, validación, errores | Escribir sin consultar |
| Spring Data JPA: repositorios, queries, proyecciones | Escribir sin consultar |
| Transacciones: propagación, aislamiento, rollback | Dominio total |
| Spring Security: filtros, JWT, autorización | Escribir con guía |
| Cliente HTTP, resiliencia, reintentos | Escribir con guía |
| Actuator, métricas, health checks | Configurar sin consultar |
| Testing: slices, contexto, Testcontainers | Escribir sin consultar |

### 3.3 Arquitectura y diseño

- Diferencias reales entre capas, hexagonal, Clean Architecture, DDD y microservicios.
- Cuándo **no** usar hexagonal (sobre-ingeniería).
- Modelado de dominio rico vs modelo anémico.
- Eventos de dominio, transaccionalidad y consistencia eventual.
- Patrones: Repository, Adapter, Strategy, Factory, State, Outbox, Saga (conceptual).

### 3.4 Datos

- Diseño relacional normalizado y cuándo desnormalizar.
- Migraciones versionadas con Flyway (nunca `ddl-auto=update` en serio).
- Índices, `EXPLAIN ANALYZE`, N+1, paginación, bloqueos optimistas/pesimistas.
- DBeaver como herramienta de trabajo diario, no solo de consulta.

### 3.5 Calidad

- Clean Code aplicado, no recitado.
- Pirámide de pruebas: unitarias, de rodaja (slice), de integración, end-to-end.
- TDD en al menos un módulo completo (cálculo de amortización).
- Cobertura con criterio (JaCoCo), análisis estático, revisión de código.

### 3.6 Negocio

- Vocabulario financiero: capital, interés, tasa E.A. vs M.V., amortización, mora, cartera.
- Facturación electrónica en Colombia: DIAN, UBL 2.1, CUFE, rangos, notas crédito.
- Pasarelas de pago: tokenización, 3-D Secure, webhooks, idempotencia, conciliación.

### 3.7 Infraestructura

- Contenedores, imágenes multi-stage, healthchecks.
- LocalStack para S3, SQS y Secrets Manager sin costo.
- AWS real: VPC, RDS, ECS Fargate, ALB, Secrets Manager, CloudWatch.
- Terraform y GitHub Actions.

### 3.8 Herramientas

- IntelliJ IDEA: navegación, refactors, live templates, depurador, HTTP client, análisis.
- DBeaver: ER, editor SQL, plan de ejecución, generación de datos.
- Maven: ciclo de vida, dependencias, `dependency:tree`, perfiles, plugins.
- Git: ramas, commits atómicos, rebase, historia legible.

## 4. Qué NO es este proyecto

- No es un curso pasivo de ver videos.
- No es un repositorio donde un agente escribe y el estudiante revisa.
- No es un microservicio de juguete: el dominio tiene reglas reales, dinero real y normativa real.
- No se busca "terminar rápido": se busca **no quedar en blanco nunca más**.

## 5. Criterio de éxito del proyecto completo

El proyecto se considera exitoso cuando el estudiante pueda, **sin ayuda y desde un IDE vacío**:

1. Crear un proyecto Spring Boot con Maven y explicar cada línea del `pom.xml`.
2. Modelar un agregado de dominio con sus invariantes y persistirlo.
3. Exponerlo por REST con validación y manejo de errores estandarizado.
4. Escribir sus pruebas unitarias y de integración.
5. Migrar el esquema con Flyway.
6. Depurar un fallo de producción a partir de un log.
7. Explicar ante otra persona por qué la arquitectura es la que es.

## 6. Duración estimada

18 fases. Ritmo sugerido: 1 a 3 sesiones por fase. No hay fecha límite: la métrica es la
**comprensión demostrada**, verificada con las preguntas de control de cada fase.
