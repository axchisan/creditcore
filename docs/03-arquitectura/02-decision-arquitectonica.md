# 02 — Decisión arquitectónica de CreditCore

## 1. La decisión

**Monolito modular + Arquitectura hexagonal + DDD táctico + organización por vertical slices.**

Formalmente registrada en [`adr/0001-arquitectura-base.md`](adr/0001-arquitectura-base.md).

```
CreditCore  =  Monolito modular        (un despliegue, módulos con fronteras)
            +  Hexagonal               (dominio aislado de la tecnología)
            +  DDD táctico             (agregados, VO, eventos, lenguaje ubicuo)
            +  Vertical slices         (paquetes por funcionalidad, no por capa)
            +  CQRS ligero             (consultas complejas sin pasar por agregados)
```

## 2. Por qué esta combinación

| Decisión | Razón de negocio | Razón de aprendizaje |
|---|---|---|
| **Monolito modular** | El sistema es una sola entidad, un solo equipo, decenas de miles de créditos. No hay ninguna razón real para distribuir. | Aprender modularidad y fronteras sin pagar el costo operativo de microservicios. |
| **Hexagonal** | Hay integraciones que cambian (pasarelas, proveedor tecnológico DIAN, centrales de riesgo). El dominio financiero debe sobrevivirlas. | Es el patrón que más se pide en entrevistas y el que mejor enseña la inversión de dependencias. |
| **DDD táctico** | El dominio tiene reglas ricas e invariantes monetarias. Un modelo anémico las dispersaría. | Enseña a modelar, no solo a mapear tablas. |
| **Vertical slices** | Cambios localizados por funcionalidad. | Evita el infierno de `service/` con 40 clases. |
| **CQRS ligero** | Los reportes de cartera no deben cargar agregados completos. | Enseña que lectura y escritura tienen necesidades distintas. |

## 3. Lo que explícitamente NO hacemos y por qué

| Descartado | Por qué |
|---|---|
| Microservicios | Un solo equipo, un solo despliegue, transacciones que se benefician de ser locales. El costo operativo no se justifica. Ver ADR-0001. |
| Event Sourcing completo | Complejidad desproporcionada. Conservamos su idea útil (movimientos append-only). |
| CQRS con bases separadas | No hay problema de escala de lectura que lo justifique. |
| Arquitectura por capas pura | El dominio quedaría acoplado a JPA, y este dominio es el activo del sistema. |
| Lombok | Ver ADR-0004: oculta precisamente la sintaxis que este proyecto quiere enseñar. |

## 4. Los módulos

```
┌─────────────────────────────────────────────────────────────────┐
│                          CreditCore                             │
│                                                                 │
│  ┌───────────┐   ┌──────────────┐   ┌──────────┐  ┌──────────┐ │
│  │ clientes  │   │  originacion │   │ creditos │  │  pagos   │ │
│  └─────┬─────┘   └───────┬──────┘   └────┬─────┘  └────┬─────┘ │
│        │                 │               │             │       │
│        └────────┬────────┴───────┬───────┴──────┬──────┘       │
│                 │                │              │              │
│           ┌─────▼──────┐  ┌──────▼──────┐ ┌────▼─────────┐    │
│           │  cartera   │  │ facturacion │ │ compartido   │    │
│           └────────────┘  └─────────────┘ │ (kernel)     │    │
│                                            └──────────────┘    │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  plataforma: seguridad · auditoría · notificaciones ·     │ │
│  │              configuración · observabilidad               │ │
│  └──────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### Reglas de dependencia entre módulos

1. Un módulo solo puede usar la **API pública** de otro módulo (paquete `api` o eventos).
2. Ningún módulo accede a las tablas de otro módulo directamente.
3. `compartido` (kernel compartido) contiene solo Value Objects universales (`Dinero`, `Tasa`,
   `NumeroDocumento`) y no depende de nada.
4. `plataforma` es transversal: todos pueden usarlo, él no usa módulos de negocio.
5. Las dependencias entre módulos de negocio **no pueden formar ciclos**.

Estas reglas se verifican automáticamente con **ArchUnit** (Fase 03). No son un acuerdo de caballeros:
si se violan, el build falla.

### Matriz de dependencias permitidas

| Desde ↓ / Hacia → | clientes | originacion | creditos | pagos | cartera | facturacion | compartido | plataforma |
|---|---|---|---|---|---|---|---|---|
| **clientes** | — | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ |
| **originacion** | ✓ api | — | ✓ api | ✗ | ✗ | ✗ | ✓ | ✓ |
| **creditos** | ✓ api | ✗ | — | ✗ | ✗ | ✗ | ✓ | ✓ |
| **pagos** | ✗ | ✗ | ✓ api | — | ✗ | ✗ | ✓ | ✓ |
| **cartera** | ✗ | ✗ | ✓ api | ✓ api | — | ✗ | ✓ | ✓ |
| **facturacion** | ✓ api | ✗ | ✗ | ✗ | ✗ | — | ✓ | ✓ |
| **compartido** | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | — | ✗ |
| **plataforma** | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ | — |

Cuando un módulo necesita reaccionar a otro sin depender de él, usa **eventos de dominio**
(por ejemplo, `facturacion` escucha `CreditoDesembolsado` sin conocer a `creditos`).

## 5. Las capas dentro de cada módulo

```
modulo/
├── dominio/              ← corazón. Java puro. CERO dependencias externas.
│   ├── modelo/              entidades, VO, agregados, enums
│   ├── evento/              eventos de dominio
│   ├── excepcion/           excepciones de negocio
│   ├── servicio/            servicios de dominio (lógica que no cabe en una entidad)
│   └── puerto/
│       ├── entrada/         interfaces de casos de uso
│       └── salida/          interfaces de lo que el dominio necesita (repos, gateways)
│
├── aplicacion/           ← orquestación. Conoce el dominio. Conoce Spring lo mínimo.
│   ├── comando/             DTOs de entrada del caso de uso
│   ├── resultado/           DTOs de salida
│   └── servicio/            implementaciones de los puertos de entrada (@Service, @Transactional)
│
└── infraestructura/      ← detalles. Aquí vive TODA la tecnología.
    ├── entrada/
    │   ├── rest/            @RestController, DTOs HTTP, mappers
    │   ├── scheduler/       @Scheduled
    │   └── mensajeria/      consumidores SQS
    └── salida/
        ├── persistencia/    @Entity JPA, Spring Data, adaptadores, mappers
        ├── cliente/         clientes HTTP hacia sistemas externos
        └── almacenamiento/  S3, ficheros
```

### La regla que lo resume todo

```
infraestructura ──► aplicacion ──► dominio
                                      │
                                      └── no importa NADA de los otros dos
```

Verificación con ArchUnit:

```java
@ArchTest
static final ArchRule elDominioEsPuro = noClasses()
        .that().resideInAPackage("..dominio..")
        .should().dependOnClassesThat().resideInAnyPackage(
                "org.springframework..",
                "jakarta.persistence..",
                "..infraestructura..",
                "..aplicacion.."
        );
```

## 6. Flujo completo de una petición

Ejemplo: `POST /api/v1/creditos/{id}/pagos`

```
 1. HTTP  ──► PagoController                          [infraestructura/entrada/rest]
                 │ valida el DTO (@Valid)
                 │ mapea DTO → Command
                 ▼
 2.          AplicarPagoService                       [aplicacion/servicio]
                 │ @Transactional  ← aquí empieza la transacción
                 │ carga el agregado por el puerto
                 ▼
 3.          CreditoRepository (interfaz)             [dominio/puerto/salida]
                 │
                 ▼
 4.          CreditoJpaAdapter                        [infraestructura/salida/persistencia]
                 │ CreditoJpaRepository.findById()
                 │ mapea CreditoEntity → Credito (dominio)
                 ▼
 5.          Credito.aplicarPago(...)                 [dominio/modelo]
                 │ ¡AQUÍ ESTÁ EL NEGOCIO!
                 │ valida invariantes, imputa según BR-050, genera movimientos,
                 │ registra el evento PagoAplicado
                 ▼
 6.          CreditoRepository.guardar(credito)       → adaptador → UPDATE
                 │
                 ▼
 7.          Publicación de eventos de dominio        (tras el commit)
                 │
                 ▼
 8.          Mapeo Resultado → DTO de respuesta       [infraestructura/entrada/rest]
                 ▼
 9. HTTP 200 con el resultado
```

**Observa dónde está la lógica de negocio: en el paso 5, dentro de una clase de Java puro que se
prueba en microsegundos sin levantar nada.** Ese es todo el punto de la arquitectura.

## 7. El costo que aceptamos conscientemente

| Costo | Mitigación |
|---|---|
| Dos modelos: dominio y entidad JPA | Mappers explícitos y probados. Es el precio del aislamiento. |
| Más clases e interfaces | Vertical slices mantienen todo lo de una funcionalidad junto. |
| Más ceremonia para un CRUD simple | En módulos puramente CRUD (parametrización) se permite un atajo documentado. |
| Curva de aprendizaje | Es precisamente el objetivo del proyecto. |

> **Excepción pragmática declarada**: los módulos de pura parametrización (catálogos, tablas de
> configuración) pueden usar una capa simple sin duplicar modelo. La sobre-ingeniería también es un
> defecto. Cada excepción se documenta en un ADR.

## 8. Evolución prevista

```
Hoy:      Monolito modular con fronteras verificadas
          │
Si un día hiciera falta:
          ├─► Extraer `facturacion` como servicio independiente
          │   (ya se comunica solo por eventos → extracción barata)
          └─► Extraer `cartera` (proceso batch pesado que podría escalar aparte)
```

La arquitectura está diseñada para que esa extracción sea posible, **sin hacerla hoy**. Eso es
diseñar para el cambio sin especular.

---

## Preguntas de control

1. ¿Por qué el dominio no puede importar `org.springframework`?
2. ¿Dónde se coloca `@Transactional` y por qué ahí y no en el controlador ni en el dominio?
3. Si `facturacion` necesita saber que se desembolsó un crédito, ¿cómo se entera sin depender de `creditos`?
4. ¿Qué diferencia hay entre `CreditoEntity` y `Credito`? ¿Por qué existen las dos?
5. ¿Qué pasaría si un `@RestController` llamara directamente a un `JpaRepository`?
6. Nombra dos costos reales de esta arquitectura y cómo los mitigamos.
