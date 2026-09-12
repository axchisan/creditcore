# Índice de la documentación

## 00 — Visión
| Documento | Contenido |
|---|---|
| [01 Visión y objetivos](00-vision/01-vision-y-objetivos.md) | Por qué existe el proyecto, objetivos por área, criterio de éxito |
| [02 Método de aprendizaje](00-vision/02-metodo-de-aprendizaje.md) | Contrato de trabajo, ciclo de sesión, protocolo de depuración, bitácora |
| [03 Glosario de negocio](00-vision/03-glosario-negocio.md) | Crédito, DIAN, pagos, control interno |
| [04 Glosario técnico](00-vision/04-glosario-tecnico.md) | Java, Spring, persistencia, arquitectura, pruebas, infraestructura |

## 01 — Negocio
| Documento | Contenido |
|---|---|
| [01 Dominio del crédito](01-negocio/01-dominio-credito.md) | Ciclo de vida, tasas, amortización, pagos, mora, scoring |
| [02 Facturación electrónica DIAN](01-negocio/02-facturacion-electronica-dian.md) | UBL, CUFE, rangos, firma, flujo completo |
| [03 Pasarelas de pago](01-negocio/03-pasarelas-de-pago.md) | Ciclo de una transacción, webhooks, idempotencia, conciliación |
| [04 Catálogo de reglas](01-negocio/04-catalogo-reglas-negocio.md) | BR-001 a BR-084 |

## 02 — Requerimientos
| Documento | Contenido |
|---|---|
| [01 Alcance](02-requerimientos/01-alcance.md) | Contexto, módulos, fuera de alcance, supuestos, restricciones |
| [02 Requerimientos funcionales](02-requerimientos/02-requerimientos-funcionales.md) | RF-001 a RF-115 |
| [03 Requerimientos no funcionales](02-requerimientos/03-requerimientos-no-funcionales.md) | RNF-001 a RNF-083 con verificación |
| [04 Historias de usuario](02-requerimientos/04-historias-de-usuario.md) | HU con criterios Gherkin |
| [05 Trazabilidad y aceptación](02-requerimientos/05-trazabilidad-y-aceptacion.md) | Matriz, DoR, DoD, criterios del proyecto |

## 03 — Arquitectura
| Documento | Contenido |
|---|---|
| [01 Panorama de arquitecturas](03-arquitectura/01-panorama-arquitecturas.md) | Capas, hexagonal, Clean, DDD, vertical slice, microservicios, EDA, CQRS |
| [02 Decisión arquitectónica](03-arquitectura/02-decision-arquitectonica.md) | Qué elegimos, módulos, capas, flujo de una petición |
| [03 Estructura del proyecto](03-arquitectura/03-estructura-del-proyecto.md) | Árbol de paquetes, convenciones de nombres, visibilidad |
| [04 Modelo de dominio](03-arquitectura/04-modelo-de-dominio.md) | Agregados, VO, máquinas de estado, eventos |
| [05 Modelo de datos](03-arquitectura/05-modelo-de-datos.md) | Esquema PostgreSQL, restricciones, índices, migraciones |
| [06 Contratos de API](03-arquitectura/06-contratos-api.md) | Rutas, códigos, RFC 7807, paginación, idempotencia |
| [ADRs](03-arquitectura/adr/) | Decisiones registradas 0001–0009 |

## 04 — Calidad
| Documento | Contenido |
|---|---|
| [01 Clean Code y principios](04-calidad/01-clean-code-y-principios.md) | Nombres, funciones, SOLID, olores de código |
| [02 Estrategia de pruebas](04-calidad/02-estrategia-de-pruebas.md) | Pirámide, TDD, dobles, ArchUnit, cobertura |
| [03 Convenciones de código](04-calidad/03-convenciones-de-codigo.md) | Formato, nomenclatura, anotaciones, transacciones, logging |
| [04 Flujo de Git](04-calidad/04-git-workflow.md) | Ramas, Conventional Commits, checklists |

## 05 — Fases
| Documento | Contenido |
|---|---|
| [00 Roadmap](05-fases/00-roadmap.md) | Las 18 fases con objetivos y entregables |
| [Fase 00 — Entorno](05-fases/fase-00-entorno.md) | Instalación, IntelliJ, proyecto Maven a mano |

> Los documentos de las fases 01 a 18 se escriben al inicio de cada fase, con el detalle de
> conceptos, sintaxis y ejercicios. El roadmap ya define objetivos, contenido y criterios de todas.

## 06 — Herramientas
| Documento | Contenido |
|---|---|
| [01 IntelliJ: atajos](06-herramientas/01-intellij-atajos.md) | Atajos por categoría, live templates, HTTP client, plugins |
| [02 Depuración](06-herramientas/02-depuracion.md) | Protocolo, stacktraces, breakpoints, errores típicos |
| [03 DBeaver y PostgreSQL](06-herramientas/03-dbeaver-y-postgres.md) | Configuración, EXPLAIN, SQL del dominio |
| [04 Maven](06-herramientas/04-maven.md) | Ciclo de vida, dependencias, scopes, plugins |

## 07 — Infraestructura
| Documento | Contenido |
|---|---|
| [01 Entornos y configuración](07-infraestructura/01-entornos-y-configuracion.md) | Perfiles, precedencia, secretos |
| [02 Contenedores y LocalStack](07-infraestructura/02-contenedores-y-localstack.md) | OrbStack, compose, LocalStack, Dockerfile |
| [03 Arquitectura AWS](07-infraestructura/03-arquitectura-aws.md) | Diagrama objetivo, servicios, seguridad, costos, Terraform |
| [04 CI/CD](07-infraestructura/04-ci-cd.md) | GitHub Actions, calidad, despliegue |
