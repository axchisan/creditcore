# 04 — Glosario técnico

Definiciones cortas y operativas. Se irán ampliando en la fase donde cada concepto aparece.

## A. Java

| Término | Definición |
|---|---|
| **JDK / JRE / JVM** | JDK = kit de desarrollo (compilador + herramientas). JRE = entorno de ejecución. JVM = la máquina que ejecuta el bytecode. |
| **Bytecode** | Código intermedio (`.class`) que la JVM ejecuta; es lo que produce `javac`. |
| **Classpath** | Lista de rutas donde la JVM busca clases y recursos en tiempo de ejecución. |
| **POJO** | *Plain Old Java Object*: clase simple sin dependencias del framework. |
| **Bean** | Objeto gestionado por el contenedor de Spring. |
| **`record`** | Clase inmutable con constructor canónico, `equals`, `hashCode` y `toString` generados. |
| **`sealed`** | Clase/interfaz que restringe explícitamente quién puede implementarla o extenderla. |
| **Pattern matching** | `instanceof` y `switch` que además extraen datos del objeto. |
| **`Optional`** | Contenedor que representa "puede no haber valor". Para retornos, no para campos ni parámetros. |
| **Stream** | Secuencia perezosa de elementos con operaciones intermedias y terminales. |
| **Checked vs unchecked** | *Checked* obliga a `try/catch` o `throws`; *unchecked* (`RuntimeException`) no. |
| **Autoboxing** | Conversión automática `int` ↔ `Integer`. Causa `NullPointerException` sorpresa. |
| **`equals`/`hashCode`** | Contrato de igualdad. Romperlo rompe `HashMap`, `Set` y JPA. |
| **Inmutabilidad** | Objeto cuyo estado no cambia tras construirse. Base de código predecible y thread-safe. |
| **Virtual threads** | Hilos ligeros de la JVM (Project Loom) para concurrencia de alto volumen con bloqueo. |

## B. Spring / Spring Boot

| Término | Definición |
|---|---|
| **IoC (Inversión de Control)** | El framework crea y conecta los objetos, no tú con `new`. |
| **DI (Inyección de Dependencias)** | Mecanismo por el cual un bean recibe sus colaboradores. Preferir **por constructor**. |
| **`ApplicationContext`** | El contenedor de Spring: registro de beans y su ciclo de vida. |
| **Autoconfiguración** | Configuración automática condicional según lo que haya en el classpath. |
| **Starter** | Dependencia agregadora (`spring-boot-starter-web`) que trae un conjunto coherente de librerías. |
| **Perfil (`@Profile`)** | Activación condicional de beans/configuración por entorno (`dev`, `test`, `prod`). |
| **`@ConfigurationProperties`** | Enlace tipado entre el YAML y una clase de configuración. |
| **Proxy** | Objeto intermedio que Spring genera para aplicar comportamiento (transacciones, caché, seguridad). Explica por qué la auto-invocación de un método `@Transactional` **no** funciona. |
| **AOP** | Programación orientada a aspectos: comportamiento transversal aplicado por interceptación. |
| **`@Transactional`** | Marca un límite transaccional. Su comportamiento depende de la **propagación** y del proxy. |
| **Slice test** | Prueba que carga solo una porción del contexto (`@WebMvcTest`, `@DataJpaTest`). |
| **Actuator** | Módulo de endpoints operativos: salud, métricas, info, entorno. |

## C. Persistencia

| Término | Definición |
|---|---|
| **ORM** | Mapeo objeto-relacional: traduce entre clases y tablas. |
| **JPA** | Especificación de persistencia en Java. Hibernate es la implementación. |
| **`EntityManager`** | API central de JPA: `persist`, `merge`, `find`, `flush`. |
| **Persistence context** | Caché de primer nivel: entidades gestionadas dentro de una transacción. |
| **Entidad gestionada / detached** | Gestionada = dentro del contexto, sus cambios se sincronizan solos. Detached = fuera, no. |
| **Dirty checking** | Hibernate detecta cambios en entidades gestionadas y emite el `UPDATE` sin que lo pidas. |
| **Lazy / Eager** | Carga diferida vs inmediata de asociaciones. |
| **`LazyInitializationException`** | Acceder a una relación lazy fuera de la transacción. Error clásico. |
| **Problema N+1** | 1 consulta para la lista + N consultas para cada relación. Mata el rendimiento. |
| **`JOIN FETCH` / `@EntityGraph`** | Soluciones al N+1. |
| **Bloqueo optimista** | Control de concurrencia con `@Version`: falla al guardar si otro modificó el dato. |
| **Bloqueo pesimista** | `SELECT ... FOR UPDATE`: bloquea la fila en la base de datos. |
| **Nivel de aislamiento** | Qué anomalías de concurrencia permite una transacción (READ COMMITTED, REPEATABLE READ, …). |
| **Migración** | Script versionado que evoluciona el esquema (Flyway `V1__descripcion.sql`). |
| **DDL / DML** | DDL = estructura (`CREATE`, `ALTER`). DML = datos (`INSERT`, `UPDATE`). |
| **Índice** | Estructura que acelera búsquedas a costa de escrituras y espacio. |
| **`EXPLAIN ANALYZE`** | Muestra el plan real de ejecución de una consulta en PostgreSQL. |

## D. Arquitectura

| Término | Definición |
|---|---|
| **Acoplamiento / cohesión** | Cuánto depende un módulo de otro / qué tan relacionado está lo que contiene. |
| **Capa** | Agrupación horizontal por responsabilidad técnica (controlador, servicio, repositorio). |
| **Puerto** | Interfaz que define lo que el dominio necesita o ofrece. |
| **Adaptador** | Implementación concreta de un puerto contra una tecnología (JPA, HTTP, SQS). |
| **Caso de uso** | Una operación completa del negocio, expresada en código de aplicación. |
| **Agregado** | Grupo de objetos de dominio con una raíz que garantiza sus invariantes. |
| **Invariante** | Regla que debe cumplirse siempre (ej.: "un crédito desembolsado no puede tener saldo negativo"). |
| **Entidad vs Value Object** | Entidad tiene identidad propia; VO se define solo por su valor (`Dinero`, `Tasa`). |
| **Modelo anémico** | Objetos con solo getters/setters y toda la lógica en servicios. Anti-patrón. |
| **DTO** | Objeto de transporte entre capas/sistemas. No es una entidad. |
| **Evento de dominio** | Hecho relevante del negocio, en pasado (`CreditoDesembolsado`). |
| **Outbox** | Patrón para publicar eventos de forma consistente con la transacción de base de datos. |
| **Idempotencia** | Repetir la operación no cambia el resultado. |
| **CQRS** | Separar el modelo de lectura del de escritura. |
| **Saga** | Coordinación de una transacción distribuida mediante pasos compensables. |

## E. Web y APIs

| Término | Definición |
|---|---|
| **REST** | Estilo arquitectónico basado en recursos, verbos HTTP y representaciones. |
| **Idempotente (HTTP)** | `GET`, `PUT`, `DELETE` lo son; `POST` no. |
| **RFC 7807 / Problem Details** | Formato estándar de respuesta de error (`application/problem+json`). |
| **Content negotiation** | Selección de representación vía `Accept`/`Content-Type`. |
| **CORS** | Política del navegador para peticiones entre orígenes distintos. |
| **JWT** | Token firmado que transporta claims de autenticación. |
| **OAuth2 / OIDC** | Protocolos de autorización y de identidad. |
| **Rate limiting** | Límite de peticiones por cliente/período. |
| **OpenAPI** | Especificación del contrato de la API; `springdoc` la genera desde el código. |

## F. Pruebas

| Término | Definición |
|---|---|
| **Prueba unitaria** | Prueba una unidad aislada, sin IO ni contexto de Spring. Milisegundos. |
| **Prueba de integración** | Prueba varios componentes reales juntos (incluida la base de datos). |
| **Prueba end-to-end** | Recorre el sistema completo por su interfaz externa. |
| **Doble de prueba** | Término general: *dummy*, *stub*, *spy*, *mock*, *fake*. |
| **Mock vs Stub** | Mock verifica interacciones; stub solo devuelve datos preparados. |
| **Testcontainers** | Librería que levanta contenedores reales (PostgreSQL, LocalStack) durante las pruebas. |
| **Fixture** | Estado conocido previo a la prueba. |
| **AAA / Given-When-Then** | Estructura de una prueba: preparar, ejecutar, verificar. |
| **TDD** | Escribir la prueba que falla antes del código que la hace pasar. |
| **Cobertura** | % de código ejecutado por las pruebas. Métrica útil, objetivo peligroso. |
| **Flaky test** | Prueba que a veces pasa y a veces falla. Se corrige o se elimina; nunca se ignora. |

## G. Infraestructura y operación

| Término | Definición |
|---|---|
| **Imagen / contenedor** | Plantilla inmutable / instancia en ejecución de esa plantilla. |
| **Multi-stage build** | Dockerfile con etapa de compilación y etapa final mínima. |
| **IaC** | Infraestructura como código (Terraform). |
| **Estado de Terraform** | Archivo que mapea recursos reales con la configuración. Nunca se versiona con secretos. |
| **CI / CD** | Integración continua / entrega continua. |
| **Twelve-Factor App** | Guía de buenas prácticas para aplicaciones cloud (config por entorno, logs a stdout, etc.). |
| **Health check** | Endpoint que indica si la aplicación puede recibir tráfico (`liveness`/`readiness`). |
| **Observabilidad** | Capacidad de entender el estado interno desde fuera: logs, métricas, trazas. |
| **Log estructurado** | Log en JSON con campos, no texto plano. |
| **Correlation ID / trace ID** | Identificador que permite seguir una petición a través de todo el sistema. |
| **SLA / SLO / SLI** | Compromiso / objetivo interno / métrica concreta de nivel de servicio. |
| **RTO / RPO** | Tiempo máximo de recuperación / pérdida máxima de datos aceptable. |
