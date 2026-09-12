# CreditCore — Plataforma de Originación y Gestión de Crédito

> Proyecto de aprendizaje intensivo de **Java + Spring Boot escrito a mano**, con arquitectura,
> pruebas, base de datos PostgreSQL, integración de pasarelas de pago, facturación electrónica DIAN
> e infraestructura en AWS.

---

## 1. Qué es esto

Esto **no** es un tutorial de "CRUD de frutas". Es la construcción, de cero y en orden real de
ingeniería, de una plataforma financiera con la complejidad que encontrarías en un entorno laboral:

- Originación de crédito (solicitud → scoring → aprobación → desembolso).
- Gestión de cartera (plan de amortización, pagos, mora, intereses).
- Integración con **pasarelas de pago** (Wompi / PayU sandbox) con webhooks, idempotencia y conciliación.
- **Facturación electrónica DIAN** (Colombia): UBL 2.1, CUFE, rangos de numeración, notas crédito.
- Infraestructura: PostgreSQL, LocalStack (S3, SQS, Secrets Manager) y despliegue final en AWS.

## 2. La regla de oro

**Todo el código de producción lo escribe el estudiante, a mano, en IntelliJ IDEA.**

Claude actúa como **profesor**: explica conceptos, muestra ejemplos funcionales en el chat o en la
documentación de cada fase, diseña, revisa, corrige y hace preguntas de comprobación. No escribe
por ti los archivos del proyecto. Las reglas completas del método están en
[`docs/00-vision/02-metodo-de-aprendizaje.md`](docs/00-vision/02-metodo-de-aprendizaje.md) y las
reglas operativas para el asistente en [`CLAUDE.md`](CLAUDE.md).

## 3. Stack

| Capa | Tecnología | Versión objetivo |
|---|---|---|
| Lenguaje | Java (Temurin) | 21 LTS |
| Framework | Spring Boot | 3.x |
| Build | Maven | 3.9.x |
| Base de datos | PostgreSQL | 17 |
| Migraciones | Flyway | — |
| ORM | Spring Data JPA / Hibernate | — |
| Pruebas | JUnit 5, AssertJ, Mockito, Testcontainers | — |
| Seguridad | Spring Security + JWT | — |
| Observabilidad | Actuator, Micrometer, Logback JSON | — |
| Contenedores | OrbStack (Docker CLI) | — |
| Cloud (simulado) | LocalStack | — |
| Cloud (real) | AWS: RDS, S3, SQS, Secrets Manager, ECS Fargate | — |
| IaC | Terraform | — |
| CI/CD | GitHub Actions | — |
| IDE | IntelliJ IDEA | — |
| Cliente SQL | DBeaver | — |

## 4. Mapa de la documentación

```
docs/
├── 00-vision/            Por qué existe el proyecto y cómo vamos a trabajar
├── 01-negocio/           El negocio explicado desde cero: crédito, pagos, DIAN
├── 02-requerimientos/    RF, RNF, historias de usuario, criterios de aceptación
├── 03-arquitectura/      Arquitecturas, decisión, modelo de dominio, datos, API, ADRs
├── 04-calidad/           Clean code, pruebas, convenciones, Git, Definition of Done
├── 05-fases/             El plan de construcción fase por fase (el corazón del proyecto)
├── 06-herramientas/      IntelliJ, DBeaver, Maven, Docker, HTTP client
└── 07-infraestructura/   Entornos, LocalStack, AWS, Terraform, CI/CD
```

### Orden de lectura recomendado (antes de escribir una sola línea de código)

1. [`00-vision/01-vision-y-objetivos.md`](docs/00-vision/01-vision-y-objetivos.md)
2. [`00-vision/02-metodo-de-aprendizaje.md`](docs/00-vision/02-metodo-de-aprendizaje.md)
3. [`01-negocio/01-dominio-credito.md`](docs/01-negocio/01-dominio-credito.md)
4. [`02-requerimientos/01-requerimientos-funcionales.md`](docs/02-requerimientos/01-requerimientos-funcionales.md)
5. [`03-arquitectura/01-panorama-arquitecturas.md`](docs/03-arquitectura/01-panorama-arquitecturas.md)
6. [`03-arquitectura/02-decision-arquitectonica.md`](docs/03-arquitectura/02-decision-arquitectonica.md)
7. [`05-fases/00-roadmap.md`](docs/05-fases/00-roadmap.md)

## 5. Estado del proyecto

| | |
|---|---|
| Fase actual | **Fase 0 — Cimientos del entorno** (no iniciada) |
| Documentación base | ✅ Completa |
| Código | ⬜ Sin iniciar |

El avance detallado vive en [`docs/05-fases/00-roadmap.md`](docs/05-fases/00-roadmap.md).

## 6. Cómo se trabaja cada sesión

1. Abrir el documento de la fase actual en `docs/05-fases/`.
2. Leer la sección de **conceptos** y responder las preguntas de comprobación.
3. Escribir el código a mano siguiendo los **entregables** de la fase.
4. Ejecutar las **pruebas de aceptación** de la fase.
5. Hacer commit siguiendo [`04-calidad/04-git-workflow.md`](docs/04-calidad/04-git-workflow.md).
6. Marcar la fase en el roadmap y anotar aprendizajes en la bitácora de la fase.
