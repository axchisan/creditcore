# 00 — Roadmap: el plan de construcción

> Orden real de ingeniería: **cimientos → estructura → instalaciones → acabados**.
> No se salta ninguna fase. El orden es dependencia técnica, no capricho.

---

## Mapa general

```
BLOQUE I — CIMIENTOS                        BLOQUE III — EL NEGOCIO
├── F00  Entorno y herramientas             ├── F06  Productos y amortización (TDD)
├── F01  Java moderno: la sintaxis          ├── F07  Originación y desembolso
└── F02  Esqueleto de la aplicación         └── F08  Pagos, mora y cierre de día

BLOQUE II — ESTRUCTURA                      BLOQUE IV — INTEGRACIONES
├── F03  Persistencia y migraciones         ├── F09  Seguridad y auditoría
├── F04  Primera rodaja vertical            ├── F10  Pasarela de pagos
└── F05  Testing en serio                   └── F11  Facturación electrónica DIAN

BLOQUE V — PRODUCCIÓN                       BLOQUE VI — CIERRE
├── F12  Observabilidad y rendimiento       ├── F16  AWS real con Terraform
├── F13  Asincronía, eventos y colas        ├── F17  Hardening y pruebas de carga
├── F14  Documentación de API               └── F18  Migración a Spring Boot 4.x
└── F15  Contenedores y CI/CD
```

---

## Estado

| Fase | Nombre | Estado | Rama | Etiqueta |
|---|---|---|---|---|
| F00 | Entorno y herramientas | ⬜ Pendiente | `fase/00-entorno` | — |
| F01 | Java moderno: la sintaxis | ⬜ | `fase/01-java` | — |
| F02 | Esqueleto de la aplicación | ⬜ | `fase/02-esqueleto` | — |
| F03 | Persistencia y migraciones | ⬜ | `fase/03-persistencia` | — |
| F04 | Primera rodaja vertical | ⬜ | `fase/04-clientes` | — |
| F05 | Testing en serio | ⬜ | `fase/05-testing` | — |
| F06 | Productos y amortización | ⬜ | `fase/06-amortizacion` | — |
| F07 | Originación y desembolso | ⬜ | `fase/07-originacion` | — |
| F08 | Pagos, mora y cierre de día | ⬜ | `fase/08-pagos` | — |
| F09 | Seguridad y auditoría | ⬜ | `fase/09-seguridad` | — |
| F10 | Pasarela de pagos | ⬜ | `fase/10-pasarela` | — |
| F11 | Facturación electrónica DIAN | ⬜ | `fase/11-facturacion` | — |
| F12 | Observabilidad y rendimiento | ⬜ | `fase/12-observabilidad` | — |
| F13 | Asincronía, eventos y colas | ⬜ | `fase/13-asincronia` | — |
| F14 | Documentación de API | ⬜ | `fase/14-openapi` | — |
| F15 | Contenedores y CI/CD | ⬜ | `fase/15-cicd` | — |
| F16 | AWS real con Terraform | ⬜ | `fase/16-aws` | — |
| F17 | Hardening y pruebas de carga | ⬜ | `fase/17-hardening` | — |
| F18 | Migración a Spring Boot 4.x | ⬜ | `fase/18-migracion` | — |

Leyenda: ⬜ Pendiente · 🟡 En curso · ✅ Completada

---

# BLOQUE I — CIMIENTOS

## F00 — Entorno y herramientas
**Sesiones estimadas:** 1 · **Documento:** [`fase-00-entorno.md`](fase-00-entorno.md)

**Objetivo:** tener un entorno de desarrollo profesional y dominar las herramientas antes de escribir
código. Como afilar el hacha antes de talar.

**Contenido:**
- Instalar y gestionar JDK 21 (SDKMAN! o IntelliJ).
- Maven: ciclo de vida, `pom.xml`, wrapper, `dependency:tree`, repositorio local.
- IntelliJ IDEA: configuración, atajos esenciales, live templates, inspecciones, formato al guardar.
- OrbStack: primer contenedor, `docker compose`.
- PostgreSQL 17: crear la base de datos y el usuario del proyecto.
- DBeaver: conexión, editor SQL, diagrama ER.
- Git: configuración, repositorio remoto, primer flujo de rama.
- Crear el proyecto Maven **a mano**, sin Spring Initializr: entender cada línea del `pom.xml`.

**Entregable verificable:** `mvn clean verify` pasa; la aplicación arranca en `http://localhost:8080`;
`docker run hello-world` funciona; DBeaver conecta a la base `creditcore`.

---

## F01 — Java moderno: la sintaxis
**Sesiones estimadas:** 2–3 · **Documento:** `fase-01-java-moderno.md`

**Objetivo:** recuperar la fluidez sintáctica. Esta fase es **pura práctica de lenguaje**, sin Spring.

**Contenido:**
- Clases, constructores, `final`, `static`, visibilidad (con foco en package-private).
- `record`: constructor compacto, validación, métodos adicionales.
- `enum` con estado y comportamiento; enum con cuerpo por constante.
- `sealed` interfaces + pattern matching en `switch`.
- Genéricos: tipos, wildcards, límites.
- `Optional`: uso correcto y sus anti-patrones.
- Streams: `map`, `filter`, `reduce`, `collect`, `groupingBy`, `flatMap`; cuándo NO usar streams.
- Lambdas y referencias a método; interfaces funcionales.
- Excepciones: jerarquía propia, `try-with-resources`, encadenamiento de causas.
- `equals`/`hashCode`: el contrato y cómo romperlo sin darse cuenta.
- `BigDecimal`: escala, redondeo, `compareTo` vs `equals`.
- Fechas: `LocalDate`, `LocalDateTime`, `Instant`, `ZoneId`, `Period`, `Duration`, `ChronoUnit`.
- Colecciones: cuál elegir y por qué; inmutabilidad.

**Ejercicio integrador:** construir el kernel compartido completo (`Dinero`, `Tasa`, `Periodicidad`,
`NumeroDocumento`, `Plazo`) **con TDD**, sin Spring, con su batería de pruebas.

**Entregable verificable:** todas las pruebas del kernel pasan; el estudiante puede escribir un
`record` con validación, un enum con comportamiento y una cadena de streams sin consultar.

---

## F02 — Esqueleto de la aplicación
**Sesiones estimadas:** 1–2 · **Documento:** `fase-02-esqueleto.md`

**Objetivo:** levantar la aplicación Spring Boot y entender **qué pasa cuando arranca**.

**Contenido:**
- `@SpringBootApplication`: qué son realmente sus tres anotaciones.
- El contenedor IoC: beans, ciclo de vida, ámbitos, orden de creación.
- Inyección por constructor y por qué no por campo.
- Autoconfiguración: cómo Spring decide qué configurar (`@ConditionalOnClass`, `@ConditionalOnMissingBean`).
- Depurar el arranque: `--debug`, el *condition evaluation report*.
- Perfiles (`local`, `test`, `prod`) y precedencia de configuración.
- `@ConfigurationProperties` tipado y validado.
- Estructura de paquetes completa del proyecto (los directorios vacíos de los módulos).
- ArchUnit: primeras reglas de arquitectura, que fallan el build si se violan.
- Primer endpoint `/api/v1/ping` para verificar el circuito completo.

**Entregable verificable:** la app arranca con tres perfiles distintos; `ReglasArquitecturaTest` pasa;
el estudiante puede explicar qué beans se crearon y por qué.

---

# BLOQUE II — ESTRUCTURA

## F03 — Persistencia y migraciones
**Sesiones estimadas:** 2–3 · **Documento:** `fase-03-persistencia.md`

**Objetivo:** dominar la capa de datos, no solo hacerla funcionar.

**Contenido:**
- PostgreSQL: tipos, `NUMERIC` vs `FLOAT`, `TIMESTAMPTZ`, `JSONB`, `UUID`.
- Flyway: convenciones de nombres, historial, `validate`, `repair`, migraciones repetibles.
- Escribir el DDL a mano: `CHECK`, `UNIQUE`, índices parciales e índices funcionales.
- JPA: `@Entity`, `@Id`, `@Column`, `@Enumerated(STRING)`, `@Embedded`, `@Version`.
- Relaciones: `@OneToMany`, `@ManyToOne`, `@JoinColumn`, `mappedBy`, `cascade`, `orphanRemoval`.
- Lazy vs Eager y la `LazyInitializationException` (provocarla a propósito y depurarla).
- El *persistence context*, dirty checking y `flush`.
- Transacciones: `@Transactional`, propagación, `readOnly`, rollback.
- HikariCP: pool de conexiones, métricas.
- UUID v7: implementarlo en el kernel compartido.
- DBeaver: diagrama ER, `EXPLAIN ANALYZE`, generación de datos de prueba.

**Entregable verificable:** migraciones aplicadas desde cero; `ddl-auto=validate` arranca sin errores;
el estudiante provoca y explica una `LazyInitializationException`.

---

## F04 — Primera rodaja vertical: Clientes
**Sesiones estimadas:** 2–3 · **Documento:** `fase-04-clientes.md`

**Objetivo:** construir **una funcionalidad completa de arriba abajo**. Es la fase que fija el patrón
que repetirás en todo el proyecto.

**Contenido:**
- Agregado `Cliente` con invariantes en el constructor.
- Puerto de salida `ClienteRepository` (interfaz del dominio).
- Entidad JPA `ClienteEntity` y adaptador; mapeo manual con pruebas.
- Casos de uso: registrar, consultar, listar, actualizar contacto, inactivar.
- DTOs de entrada y salida; mapeo REST.
- Bean Validation: `@NotBlank`, `@Email`, `@Past`, validadores personalizados.
- `@RestControllerAdvice` y Problem Details (RFC 7807) — una vez, para todo el proyecto.
- Paginación, ordenamiento y filtros.
- Pruebas: unitarias de dominio, `@WebMvcTest`, `@DataJpaTest` con Testcontainers.
- IntelliJ HTTP Client: fichero `.http` con todas las peticiones.

**Entregable verificable:** los cuatro escenarios de HU-001 y los tres de HU-002 pasan;
`POST /api/v1/clientes` responde 201 con `Location`; el documento duplicado devuelve 409.

---

## F05 — Testing en serio
**Sesiones estimadas:** 2 · **Documento:** `fase-05-testing.md`

**Objetivo:** convertir las pruebas en herramienta de diseño, no en trámite.

**Contenido:**
- JUnit 5 a fondo: ciclo de vida, `@Nested`, `@DisplayName`, `@ParameterizedTest`, `@TestFactory`.
- AssertJ: aserciones fluidas, `assertThatThrownBy`, `extracting`, `satisfies`, comparaciones recursivas.
- Mockito: `given/willReturn`, `then/should`, `ArgumentCaptor`, `@MockitoBean`, cuándo NO mockear.
- Testcontainers: contenedor singleton, `@ServiceConnection`, reutilización.
- Object Mother y Test Data Builder.
- JaCoCo: informe, reglas que rompen el build, exclusiones justificadas.
- Pruebas de concurrencia.
- Refactorizar la suite de F04 aplicando todo lo anterior.

**Entregable verificable:** `mvn verify` con umbral de cobertura activo; suite completa < 2 minutos.

---

# BLOQUE III — EL NEGOCIO

## F06 — Productos y motor de amortización (TDD)
**Sesiones estimadas:** 2–3 · **Documento:** `fase-06-amortizacion.md`

**Objetivo:** el primer módulo construido **íntegramente con TDD**. Es el corazón financiero.

**Contenido:**
- Agregado `ProductoCredito` y su parametrización.
- Conversión de tasas E.A. ↔ periódica, con `BigDecimal` y potencias fraccionarias.
- Patrón **Strategy**: `SistemaAmortizacion` con implementaciones francesa y alemana.
- Generación del plan de pagos; ajuste de la última cuota; cierre exacto en cero.
- Cálculo de fechas de vencimiento; días no hábiles.
- Endpoint de simulación.
- TDD estricto: rojo → verde → refactor, con los casos de referencia de
  [`docs/01-negocio/01-dominio-credito.md`](../01-negocio/01-dominio-credito.md).

**Entregable verificable:** simular 1.000.000 a 6 meses al 24% E.A. devuelve cuota 177.375, suma de
capital exacta y saldo final cero (BR-040, BR-041).

---

## F07 — Originación y desembolso
**Sesiones estimadas:** 3–4 · **Documento:** `fase-07-originacion.md`

**Objetivo:** el flujo de negocio más complejo del sistema.

**Contenido:**
- Agregado `SolicitudCredito` con máquina de estados.
- Motor de evaluación: reglas knock-out, scoring ponderado, capacidad de pago.
- Registro auditable de la decisión (BR-022).
- Adaptador simulado de central de riesgo.
- Agregado `Credito`, desembolso transaccional y generación del plan.
- Idempotencia del desembolso (BR-035), garantizada con `UNIQUE` en base de datos.
- Eventos de dominio: publicación y consumo interno.
- El error clásico de la auto-invocación de `@Transactional`, visto con el depurador.

**Entregable verificable:** flujo completo solicitud → evaluación → aprobación → aceptación →
desembolso, con prueba de integración; el desembolso repetido no crea un segundo crédito.

---

## F08 — Pagos, mora y cierre de día
**Sesiones estimadas:** 3–4 · **Documento:** `fase-08-pagos.md`

**Objetivo:** la lógica financiera más delicada: aplicar dinero correctamente.

**Contenido:**
- Orden de imputación (BR-050) con TDD.
- Movimientos append-only y reversos.
- Pago parcial, en exceso, anticipado; saldo a favor.
- Abono extraordinario con recálculo de plan (reducir plazo / reducir cuota).
- Liquidación a una fecha.
- Concurrencia: bloqueo optimista con `@Version`, y cuándo usar pesimista.
- Proceso de cierre de día: `@Scheduled`, idempotencia, re-ejecución para fechas pasadas.
- Cálculo de DPD, interés moratorio y recalificación de cartera.

**Entregable verificable:** todos los escenarios de HU-040 a HU-043 y HU-050; la prueba de dos pagos
concurrentes no corrompe el saldo.

---

# BLOQUE IV — INTEGRACIONES

## F09 — Seguridad y auditoría
**Sesiones estimadas:** 2–3 · **Documento:** `fase-09-seguridad.md`

**Contenido:**
- Spring Security: cadena de filtros, `SecurityFilterChain`, cómo depurarla.
- Autenticación con JWT: emisión, validación, refresh token.
- Autorización: roles, `@PreAuthorize`, autorización a nivel de método.
- Segregación de funciones (BR-033) y doble aprobación (BR-025).
- Auditoría transversal con AOP o `@EntityListeners`.
- Cifrado de contraseñas, enmascaramiento de datos sensibles en logs.
- Habeas data (Ley 1581) y OWASP Top 10 aplicado.

**Entregable verificable:** toda la API protegida; prueba negativa por cada endpoint; auditoría
registrando antes/después.

---

## F10 — Pasarela de pagos
**Sesiones estimadas:** 3 · **Documento:** `fase-10-pasarela.md`

**Contenido:**
- Puerto `PasarelaPagoPort` y dos adaptadores (Wompi sandbox + fake controlable).
- Cliente HTTP moderno (`RestClient`), timeouts, serialización.
- Resiliencia con Resilience4j: retry, circuit breaker, fallback.
- Webhooks: verificación de firma HMAC, anti-replay, procesamiento idempotente.
- Idempotencia garantizada por `UNIQUE` (ADR-0009) y traducción de la violación de integridad.
- `Idempotency-Key` en llamadas salientes.
- Job de conciliación.
- Pruebas con WireMock o servidor simulado; prueba de webhook duplicado concurrente.

**Entregable verificable:** HU-041 completa; el webhook triplicado aplica un solo pago.

---

## F11 — Facturación electrónica DIAN
**Sesiones estimadas:** 3–4 · **Documento:** `fase-11-facturacion.md`

**Contenido:**
- Agregado `RangoNumeracion` y asignación segura de consecutivos ante concurrencia (BR-071).
- Construcción del XML UBL 2.1.
- Cálculo determinista del CUFE (SHA-384) y pruebas de *golden file*.
- Firma digital XAdES con `KeyStore`; el certificado como secreto.
- Puerto `ProveedorTecnologicoPort` con adaptador simulado que reproduce aceptaciones y rechazos.
- Reenvío conservando consecutivo (BR-072); notas crédito y débito.
- Generación de PDF con QR y CUFE.
- Almacenamiento de XML y PDF en S3 (LocalStack).

**Entregable verificable:** HU-060 y HU-061 completas; prueba de concurrencia sobre el consecutivo
sin saltos ni repeticiones.

---

# BLOQUE V — PRODUCCIÓN

## F12 — Observabilidad y rendimiento
**Sesiones estimadas:** 2 · **Documento:** `fase-12-observabilidad.md`

**Contenido:**
- Actuator: health con detalle, readiness/liveness, info, métricas.
- Micrometer: contadores y temporizadores de negocio; Prometheus.
- Logs estructurados JSON; MDC con correlation ID; filtro de propagación.
- Detección y corrección de N+1 (contador de consultas en pruebas).
- `EXPLAIN ANALYZE` en DBeaver; creación de índices con medición antes/después.
- Caché con `@Cacheable` y su invalidación.
- Generación de datos sintéticos masivos para medir de verdad.

**Entregable verificable:** RNF-001 a RNF-004 medidos y cumplidos, con evidencia.

---

## F13 — Asincronía, eventos y colas
**Sesiones estimadas:** 2–3 · **Documento:** `fase-13-asincronia.md`

**Contenido:**
- Eventos de dominio con `@TransactionalEventListener(AFTER_COMMIT)`.
- Patrón **Outbox**: por qué publicar tras el commit no basta.
- SQS con LocalStack: productor, consumidor, reintentos, DLQ.
- `@Async`, pools de hilos, virtual threads.
- Bloqueo distribuido para que el batch no se duplique con varias instancias.
- Idempotencia del consumidor.

**Entregable verificable:** un desembolso dispara facturación y notificación de forma asíncrona;
un fallo del consumidor termina en la DLQ tras N reintentos.

---

## F14 — Documentación de API
**Sesiones estimadas:** 1 · **Documento:** `fase-14-openapi.md`

**Contenido:** springdoc-openapi, anotaciones, ejemplos, agrupación, esquemas de seguridad,
exportación del contrato, Swagger UI por perfil.

---

## F15 — Contenedores y CI/CD
**Sesiones estimadas:** 2 · **Documento:** `fase-15-cicd.md`

**Contenido:**
- Dockerfile multi-stage, capas, usuario no root, healthcheck, JVM en contenedor.
- `compose.yaml` con Postgres + LocalStack.
- GitHub Actions: build, pruebas, cobertura, análisis estático, escaneo de secretos y dependencias.
- Caché de dependencias; matriz de trabajos; publicación de la imagen.

---

# BLOQUE VI — CIERRE

## F16 — AWS real con Terraform
**Sesiones estimadas:** 3 · **Documento:** `fase-16-aws.md`

**Contenido:** VPC, subredes, security groups, RDS PostgreSQL, ECS Fargate, ALB, S3, SQS,
Secrets Manager, CloudWatch, IAM con mínimo privilegio, Terraform con estado remoto, despliegue real.

---

## F17 — Hardening y pruebas de carga
**Sesiones estimadas:** 2 · **Documento:** `fase-17-hardening.md`

**Contenido:** pruebas de carga (k6 o Gatling), tuning de JVM y del pool, backups y restauración,
runbook operativo, revisión de seguridad final, retrospectiva del proyecto.

---

## F18 — Migración a Spring Boot 4.x
**Sesiones estimadas:** 2 · **Documento:** `fase-18-migracion.md`

**Contenido:** leer las notas de migración, actualizar el parent, resolver incompatibilidades
apoyándose en la batería de pruebas, adoptar novedades relevantes. **Es el examen final real**:
migrar un sistema completo confiando en las pruebas que tú escribiste.

---

## Ejercicios en frío (modo examen)

| Después de | Ejercicio | Tiempo |
|---|---|---|
| F02 | Crear un proyecto Spring Boot desde cero con un endpoint y una prueba, sin consultar nada | 30 min |
| F05 | Rodaja vertical completa de una entidad nueva (`Ciudad`), con migración, dominio, puerto, adaptador, caso de uso, controlador y pruebas | 2 h |
| F08 | Implementar una regla de negocio nueva con TDD, dado solo el enunciado | 1 h |
| F11 | Depurar un bug introducido a propósito, solo con el log como pista | 45 min |
| F15 | Explicar la arquitectura completa a otra persona, en voz alta, sin diapositivas | 20 min |

---

## Cómo se actualiza este roadmap

Al cerrar cada fase:
1. Marcar ✅ en la tabla de estado.
2. Anotar la etiqueta Git.
3. Registrar desviaciones del plan (aquí, no en la memoria).
