# 03 — Estructura del proyecto

## 1. Coordenadas Maven

```
groupId:     com.axchisan
artifactId:  creditcore
version:     0.1.0-SNAPSHOT
package:     com.axchisan.creditcore
Java:        21
```

## 2. Árbol de directorios (estado objetivo al final del proyecto)

```
LeaningJava/                             ← raíz del repositorio
├── README.md
├── CLAUDE.md
├── .gitignore
├── docs/                                ← toda la documentación
├── infra/                               ← Terraform y scripts de infraestructura (F16)
│   ├── terraform/
│   └── localstack/
├── .github/workflows/                   ← CI/CD (F15)
└── creditcore/                          ← el proyecto Maven
    ├── pom.xml
    ├── Dockerfile
    ├── compose.yaml                     ← Postgres + LocalStack para desarrollo
    ├── src/
    │   ├── main/
    │   │   ├── java/com/axchisan/creditcore/
    │   │   │   ├── CreditCoreApplication.java
    │   │   │   │
    │   │   │   ├── compartido/                    ← kernel compartido
    │   │   │   │   ├── dominio/
    │   │   │   │   │   ├── Dinero.java
    │   │   │   │   │   ├── Tasa.java
    │   │   │   │   │   ├── Periodicidad.java
    │   │   │   │   │   ├── NumeroDocumento.java
    │   │   │   │   │   ├── TipoDocumento.java
    │   │   │   │   │   └── EventoDominio.java
    │   │   │   │   └── excepcion/
    │   │   │   │       ├── ReglaNegocioViolada.java
    │   │   │   │       └── RecursoNoEncontrado.java
    │   │   │   │
    │   │   │   ├── plataforma/                    ← transversal
    │   │   │   │   ├── config/
    │   │   │   │   │   ├── JacksonConfig.java
    │   │   │   │   │   ├── OpenApiConfig.java
    │   │   │   │   │   └── AsyncConfig.java
    │   │   │   │   ├── error/
    │   │   │   │   │   ├── ManejadorGlobalExcepciones.java   ← @RestControllerAdvice
    │   │   │   │   │   └── TipoProblema.java
    │   │   │   │   ├── seguridad/                            ← F09
    │   │   │   │   ├── auditoria/                            ← F09
    │   │   │   │   ├── observabilidad/                       ← F12
    │   │   │   │   └── notificacion/
    │   │   │   │
    │   │   │   ├── clientes/                      ← MÓDULO
    │   │   │   │   ├── api/                       ← lo único público para otros módulos
    │   │   │   │   │   ├── ClienteApi.java
    │   │   │   │   │   └── ClienteResumen.java
    │   │   │   │   ├── dominio/
    │   │   │   │   │   ├── modelo/
    │   │   │   │   │   │   ├── Cliente.java
    │   │   │   │   │   │   ├── ClienteId.java
    │   │   │   │   │   │   ├── DatosContacto.java
    │   │   │   │   │   │   ├── InformacionFinanciera.java
    │   │   │   │   │   │   └── EstadoCliente.java
    │   │   │   │   │   ├── excepcion/
    │   │   │   │   │   │   ├── ClienteDuplicadoException.java
    │   │   │   │   │   │   └── ClienteMenorDeEdadException.java
    │   │   │   │   │   └── puerto/
    │   │   │   │   │       ├── entrada/
    │   │   │   │   │       │   ├── RegistrarClienteUseCase.java
    │   │   │   │   │       │   ├── ConsultarClienteUseCase.java
    │   │   │   │   │       │   └── ActualizarContactoUseCase.java
    │   │   │   │   │       └── salida/
    │   │   │   │   │           └── ClienteRepository.java
    │   │   │   │   ├── aplicacion/
    │   │   │   │   │   ├── comando/
    │   │   │   │   │   │   └── RegistrarClienteCommand.java
    │   │   │   │   │   └── servicio/
    │   │   │   │   │       ├── RegistrarClienteService.java
    │   │   │   │   │       └── ConsultarClienteService.java
    │   │   │   │   └── infraestructura/
    │   │   │   │       ├── entrada/rest/
    │   │   │   │       │   ├── ClienteController.java
    │   │   │   │       │   ├── dto/
    │   │   │   │       │   │   ├── CrearClienteRequest.java
    │   │   │   │       │   │   └── ClienteResponse.java
    │   │   │   │       │   └── ClienteRestMapper.java
    │   │   │   │       └── salida/persistencia/
    │   │   │   │           ├── ClienteEntity.java
    │   │   │   │           ├── ClienteJpaRepository.java
    │   │   │   │           ├── ClienteRepositoryAdapter.java
    │   │   │   │           └── ClientePersistenciaMapper.java
    │   │   │   │
    │   │   │   ├── productos/                     ← MÓDULO (F06)
    │   │   │   ├── originacion/                   ← MÓDULO (F07)
    │   │   │   ├── creditos/                      ← MÓDULO (F07-F08)
    │   │   │   ├── pagos/                         ← MÓDULO (F08, F10)
    │   │   │   ├── cartera/                       ← MÓDULO (F08)
    │   │   │   └── facturacion/                   ← MÓDULO (F11)
    │   │   │
    │   │   └── resources/
    │   │       ├── application.yml                ← común
    │   │       ├── application-local.yml
    │   │       ├── application-test.yml
    │   │       ├── application-prod.yml
    │   │       ├── logback-spring.xml
    │   │       └── db/migration/
    │   │           ├── V1__esquema_inicial.sql
    │   │           ├── V2__clientes.sql
    │   │           └── ...
    │   └── test/
    │       ├── java/com/axchisan/creditcore/
    │       │   ├── arquitectura/
    │       │   │   └── ReglasArquitecturaTest.java     ← ArchUnit
    │       │   ├── compartido/
    │       │   │   └── dominio/DineroTest.java
    │       │   ├── clientes/
    │       │   │   ├── dominio/ClienteTest.java                  ← unitaria
    │       │   │   ├── aplicacion/RegistrarClienteServiceTest.java ← con mocks
    │       │   │   └── infraestructura/
    │       │   │       ├── ClienteControllerTest.java            ← @WebMvcTest
    │       │   │       └── ClienteRepositoryAdapterIT.java       ← @DataJpaTest + Testcontainers
    │       │   └── soporte/
    │       │       ├── PostgresTestContainer.java
    │       │       └── datos/ClienteMother.java                  ← Object Mother
    │       └── resources/
    │           ├── application-test.yml
    │           └── http/                                        ← peticiones del HTTP client
    │               ├── clientes.http
    │               └── creditos.http
    └── ...
```

## 3. Convención de nombres de paquetes y clases

| Elemento | Convención | Ejemplo |
|---|---|---|
| Módulo | sustantivo plural, minúscula | `clientes`, `pagos` |
| Entidad de dominio | sustantivo singular | `Credito`, `Cliente` |
| Identificador | `<Entidad>Id` | `CreditoId` |
| Value Object | sustantivo del concepto | `Dinero`, `Tasa` |
| Enum de estado | `Estado<Entidad>` | `EstadoCredito` |
| Puerto de entrada | `<Verbo><Sustantivo>UseCase` | `AplicarPagoUseCase` |
| Puerto de salida | `<Entidad>Repository` / `<Servicio>Port` | `CreditoRepository`, `PasarelaPagoPort` |
| Implementación de caso de uso | `<Verbo><Sustantivo>Service` | `AplicarPagoService` |
| Comando | `<Verbo><Sustantivo>Command` | `AplicarPagoCommand` |
| Entidad JPA | `<Entidad>Entity` | `CreditoEntity` |
| Repositorio Spring Data | `<Entidad>JpaRepository` | `CreditoJpaRepository` |
| Adaptador de persistencia | `<Entidad>RepositoryAdapter` | `CreditoRepositoryAdapter` |
| Adaptador externo | `<Proveedor><Concepto>Adapter` | `WompiPasarelaAdapter` |
| Controlador | `<Recurso>Controller` | `CreditoController` |
| DTO de entrada | `<Accion>Request` | `CrearClienteRequest` |
| DTO de salida | `<Recurso>Response` | `ClienteResponse` |
| Evento de dominio | participio pasado | `CreditoDesembolsado` |
| Excepción de negocio | `<Problema>Exception` | `SaldoInsuficienteException` |
| Prueba unitaria | `<Clase>Test` | `CreditoTest` |
| Prueba de integración | `<Clase>IT` | `CreditoRepositoryAdapterIT` |

## 4. Por qué esta estructura y no la clásica

La estructura clásica de tutorial:

```
com.ejemplo.app
├── controller/
├── service/
├── repository/
├── model/
└── dto/
```

Problemas reales:
1. Para entender "pagos" tienes que abrir 5 paquetes distintos.
2. No hay fronteras: cualquier `@Service` puede llamar a cualquier `@Repository`.
3. A los 50 archivos, cada carpeta es un vertedero.
4. Un cambio de funcionalidad toca 5 carpetas → conflictos de merge constantes.
5. La estructura te dice **con qué está hecho** el sistema, no **qué hace**.

Con la estructura por módulos, abrir `src/main/java/.../` responde a *"¿qué hace este sistema?"*:
clientes, productos, originación, créditos, pagos, cartera, facturación. **La arquitectura grita el
dominio, no el framework.** Esa frase es de Uncle Bob y es literalmente el criterio.

## 5. Visibilidad: el detalle que hace que funcione

Java tiene un modificador infrautilizado: **package-private** (sin modificador). Es la herramienta
para que la modularidad sea real y no decorativa.

```java
// clientes/api/ClienteApi.java          ← PÚBLICO: otros módulos pueden usarlo
public interface ClienteApi {
    Optional<ClienteResumen> buscarResumen(ClienteId id);
}

// clientes/aplicacion/servicio/RegistrarClienteService.java
@Service
class RegistrarClienteService implements RegistrarClienteUseCase {   // ← sin 'public'
    // ...                                                            solo visible en su paquete
}

// clientes/infraestructura/salida/persistencia/ClienteEntity.java
@Entity
class ClienteEntity {                                                // ← sin 'public'
    // nadie fuera del paquete de persistencia puede siquiera nombrarla
}
```

> 💡 **Regla del proyecto**: una clase es `public` solo si **tiene** que serlo. Si dudas, no lo es.
> El compilador es mejor guardián de fronteras que cualquier documento.

## 6. El `pom.xml`: qué llevará y en qué fase

| Dependencia | Para qué | Fase |
|---|---|---|
| `spring-boot-starter-web` | REST, Tomcat, Jackson | F02 |
| `spring-boot-starter-validation` | Bean Validation (`@NotNull`, `@Positive`) | F04 |
| `spring-boot-starter-data-jpa` | JPA + Hibernate + HikariCP | F03 |
| `postgresql` (runtime) | Driver | F03 |
| `flyway-core`, `flyway-database-postgresql` | Migraciones | F03 |
| `spring-boot-starter-actuator` | Salud y métricas | F12 |
| `spring-boot-starter-security` | Seguridad | F09 |
| `jjwt` o `spring-security-oauth2-jose` | JWT | F09 |
| `spring-boot-starter-test` | JUnit 5, AssertJ, Mockito, MockMvc | F02 |
| `testcontainers` + `junit-jupiter` + `postgresql` | Pruebas con BD real | F05 |
| `archunit-junit5` | Reglas de arquitectura | F03 |
| `springdoc-openapi-starter-webmvc-ui` | OpenAPI | F14 |
| `resilience4j-spring-boot3` | Circuit breaker, retry | F10 |
| `micrometer-registry-prometheus` | Métricas | F12 |
| `logstash-logback-encoder` | Logs JSON | F12 |
| `spring-cloud-aws` o SDK v2 | S3, SQS, Secrets Manager | F13/F16 |
| `jacoco-maven-plugin` | Cobertura | F05 |

**Cada dependencia se agrega en la fase donde se necesita, nunca antes**, y el estudiante debe poder
explicar por qué está ahí. Un `pom.xml` que nadie entiende es deuda técnica desde el día uno.

---

## Preguntas de control

1. ¿Por qué `ClienteEntity` no debería ser `public`?
2. ¿Dónde pondrías una clase que calcula el plan de amortización y por qué?
3. Un compañero pone un `@Autowired ClienteJpaRepository` dentro de un `@RestController`.
   ¿Qué regla rompe y qué consecuencia tiene?
4. ¿Cuál es la diferencia entre `clientes/api/` y `clientes/infraestructura/entrada/rest/`?
5. ¿Por qué las pruebas de integración se llaman `*IT` y no `*Test`?
