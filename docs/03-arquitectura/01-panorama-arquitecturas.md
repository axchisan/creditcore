# 01 — Panorama de arquitecturas de software

> Objetivo: que puedas **elegir** una arquitectura y **defender** por qué, en vez de repetir la que
> viste en el último tutorial. Cada estilo se explica con su problema original, su estructura, sus
> costos y cuándo **no** usarlo.

---

## 0. Las tres preguntas que responde toda arquitectura

1. **¿Cómo divido el sistema?** (módulos, capas, servicios)
2. **¿Quién puede depender de quién?** (dirección de las dependencias)
3. **¿Qué protejo del cambio?** (lo que más cuesta cambiar debe estar más aislado)

Todo lo demás son detalles. Si entiendes estas tres preguntas, entiendes cualquier arquitectura.

---

## 1. Monolito por capas (Layered / N-tier)

**El estándar de facto en Java empresarial.** Es lo que verás en el 70% de los proyectos existentes.

```
┌──────────────────────────────────┐
│  Presentación (@RestController)  │
├──────────────────────────────────┤
│  Negocio (@Service)              │
├──────────────────────────────────┤
│  Persistencia (@Repository)      │
├──────────────────────────────────┤
│  Base de datos                   │
└──────────────────────────────────┘
         dependencias hacia abajo
```

```java
@RestController
class ClienteController {
    private final ClienteService service;          // capa de abajo
}

@Service
class ClienteService {
    private final ClienteRepository repository;    // capa de abajo
}

@Repository
interface ClienteRepository extends JpaRepository<ClienteEntity, UUID> { }
```

| A favor | En contra |
|---|---|
| Simple, conocido por todo el mundo | El negocio termina dependiendo de JPA/Spring |
| Rápido de arrancar | Los `@Service` se vuelven gigantes ("God services") |
| Perfecto para CRUD | Probar la lógica exige levantar base de datos |
| Poca ceremonia | Cambiar de tecnología de persistencia lo toca todo |

**Cuándo usarla:** CRUD, prototipos, equipos pequeños, dominios sin reglas complejas.
**Cuándo no:** cuando el negocio es el activo (créditos, seguros, contabilidad), porque la lógica se
diluye entre anotaciones de framework.

**El síntoma de que se te quedó chica**: tu `@Service` tiene 800 líneas, recibe 7 dependencias y
nadie se atreve a tocarlo.

---

## 2. Arquitectura Hexagonal (Ports & Adapters)

Propuesta por Alistair Cockburn. **Idea central: el negocio no conoce la tecnología.**

```
                 ┌──── Adaptadores de entrada (driving) ────┐
                 │  REST · CLI · Scheduler · Consumidor SQS  │
                 └──────────────────┬───────────────────────┘
                                    │ llama a
                          ╔═════════▼═════════╗
                          ║   PUERTOS DE      ║
                          ║     ENTRADA       ║   ← interfaces (casos de uso)
                          ╠═══════════════════╣
                          ║                   ║
                          ║     DOMINIO       ║   ← reglas de negocio puras
                          ║   (sin Spring,    ║      Java y nada más
                          ║    sin JPA)       ║
                          ║                   ║
                          ╠═══════════════════╣
                          ║   PUERTOS DE      ║   ← interfaces (lo que el dominio necesita)
                          ║     SALIDA        ║
                          ╚═════════▲═════════╝
                                    │ implementan
                 ┌──────────────────┴───────────────────────┐
                 │ Adaptadores de salida (driven)           │
                 │ JPA · HTTP · S3 · SQS · Email            │
                 └──────────────────────────────────────────┘
```

**La regla de oro: las dependencias apuntan hacia adentro.** El dominio no importa nada de fuera.

```java
// DOMINIO — no hay un solo import de Spring ni de JPA
public class Credito {
    private final CreditoId id;
    private Dinero saldoCapital;
    private EstadoCredito estado;

    public void aplicarPago(Dinero monto, LocalDate fechaValor) {
        if (estado != EstadoCredito.VIGENTE && estado != EstadoCredito.EN_MORA) {
            throw new CreditoNoAdmitePagosException(id, estado);   // BR-057
        }
        // ... lógica pura, testeable en microsegundos
    }
}

// PUERTO DE SALIDA — el dominio declara qué necesita
public interface CreditoRepository {
    Optional<Credito> buscarPorId(CreditoId id);
    void guardar(Credito credito);
}

// PUERTO DE ENTRADA — un caso de uso
public interface AplicarPagoUseCase {
    ResultadoAplicacion aplicar(AplicarPagoCommand command);
}

// ADAPTADOR DE SALIDA — aquí sí vive JPA
@Repository
class CreditoJpaAdapter implements CreditoRepository {
    private final CreditoJpaRepository jpa;      // Spring Data
    private final CreditoMapper mapper;

    @Override public Optional<Credito> buscarPorId(CreditoId id) {
        return jpa.findById(id.valor()).map(mapper::aDominio);
    }
}
```

| A favor | En contra |
|---|---|
| El dominio se prueba sin base de datos ni Spring | Más clases: entidad de dominio **y** entidad JPA |
| Cambiar de tecnología no toca el negocio | Necesitas mappers |
| Los límites son explícitos y verificables (ArchUnit) | Puede ser sobre-ingeniería en un CRUD |
| Obliga a pensar el modelo antes que la tabla | Curva de aprendizaje |

**Cuándo usarla:** dominios con reglas ricas, integraciones múltiples, vida larga del sistema.
**Cuándo no:** un CRUD de catálogo. Duplicar modelo para guardar un nombre y un precio es absurdo.

---

## 3. Clean Architecture (Uncle Bob)

Es hexagonal con más anillos y nombres propios.

```
        ┌────────────────────────────────────────┐
        │  Frameworks & Drivers (Spring, JPA)    │
        │  ┌──────────────────────────────────┐  │
        │  │ Interface Adapters               │  │
        │  │ (controllers, presenters, gateways)│ │
        │  │  ┌────────────────────────────┐  │  │
        │  │  │ Application Business Rules │  │  │
        │  │  │       (casos de uso)       │  │  │
        │  │  │  ┌──────────────────────┐  │  │  │
        │  │  │  │ Enterprise Business  │  │  │  │
        │  │  │  │  Rules (entidades)   │  │  │  │
        │  │  │  └──────────────────────┘  │  │  │
        │  │  └────────────────────────────┘  │  │
        │  └──────────────────────────────────┘  │
        └────────────────────────────────────────┘
             Las dependencias apuntan hacia adentro
```

**Diferencia práctica con hexagonal**: casi ninguna. Clean separa explícitamente "reglas de empresa"
(entidades, válidas para toda la organización) de "reglas de aplicación" (casos de uso, propias de
este sistema). En la práctica, la mayoría de equipos usa una mezcla de ambas y la llama indistintamente.

**Aporte real de Clean**: la **regla de dependencia** enunciada de forma tajante —
*"el código de un círculo interno no puede nombrar nada de un círculo externo"*.

---

## 4. Domain-Driven Design (DDD)

**No es una arquitectura, es una forma de modelar.** Se combina con hexagonal, no compite con ella.

### DDD estratégico
- **Lenguaje ubicuo**: el código usa las mismas palabras que el negocio. Si el analista dice
  "desembolso", la clase se llama `Desembolso`, no `LoanDisbursementProcessor`.
- **Bounded Context**: fronteras donde un término tiene un único significado. "Cliente" en
  Originación no es lo mismo que "Cliente" en Cobranza.
- **Context Map**: cómo se relacionan esos contextos.

### DDD táctico

| Patrón | Qué es | Ejemplo en CreditCore |
|---|---|---|
| **Entidad** | Identidad propia que persiste en el tiempo | `Credito`, `Cliente` |
| **Value Object** | Se define solo por su valor; inmutable | `Dinero`, `Tasa`, `NumeroDocumento` |
| **Agregado** | Grupo con una raíz que garantiza invariantes | `Credito` (raíz) + `Cuota` + `Movimiento` |
| **Raíz de agregado** | Único punto de entrada al agregado | Solo se accede a `Cuota` a través de `Credito` |
| **Repositorio** | Colección de agregados | `CreditoRepository` (uno por agregado, no por tabla) |
| **Servicio de dominio** | Lógica que no pertenece a una sola entidad | `CalculadoraAmortizacion` |
| **Evento de dominio** | Hecho ocurrido, en pasado | `CreditoDesembolsado` |
| **Factory** | Creación compleja con invariantes | `CreditoFactory.desdeDesembolso(...)` |

### La regla de oro de los agregados

> **Una transacción modifica un solo agregado.** Si necesitas modificar dos, usa un evento de dominio
> y acepta consistencia eventual.

Esto no es purismo: define los límites transaccionales y previene bloqueos. Es probablemente la
lección más valiosa de DDD.

**Cuándo usar DDD:** el negocio es complejo y hay expertos de dominio con quien hablar.
**Cuándo no:** CRUD, reportes, ETL. DDD sobre un dominio trivial es puro costo.

---

## 5. Arquitectura Vertical Slice

Organizar por **funcionalidad**, no por capa técnica.

```
  Por capas (horizontal)            Vertical slice
  ─────────────────────             ──────────────
  controller/                        clientes/
    ClienteController                  RegistrarCliente.java   (todo junto)
    CreditoController                  ConsultarCliente.java
  service/                           creditos/
    ClienteService                     DesembolsarCredito.java
    CreditoService                     ConsultarCredito.java
  repository/                        pagos/
    ClienteRepository                  AplicarPago.java
    CreditoRepository
```

| A favor | En contra |
|---|---|
| Todo lo de una funcionalidad está junto | Puede duplicar código entre slices |
| Cambios localizados; menos conflictos en Git | Menos reutilización evidente |
| Encaja con CQRS y con equipos por funcionalidad | Cuesta imponer disciplina transversal |

Se combina muy bien con hexagonal: **hexagonal define las capas, vertical slice define los paquetes.**
Es exactamente lo que haremos.

---

## 6. Microservicios

Dividir el sistema en servicios desplegables de forma independiente, cada uno con su base de datos.

```
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│Clientes  │  │Originación│ │ Cartera  │  │Facturación│
│  + BD    │  │   + BD    │ │  + BD    │  │   + BD    │
└────┬─────┘  └─────┬────┘  └────┬─────┘  └────┬─────┘
     └──────────────┴─── eventos ─┴─────────────┘
```

| A favor | En contra |
|---|---|
| Escalado y despliegue independientes | Complejidad operativa enorme |
| Equipos autónomos | Transacciones distribuidas (sagas) |
| Aislamiento de fallos | Depuración distribuida |
| Libertad tecnológica por servicio | Latencia de red y fallos parciales |
| | Consistencia eventual en todas partes |

**La verdad incómoda:** la mayoría de los microservicios existen por moda, no por necesidad. El
consejo profesional estándar (Fowler, Newman) es **empezar por un monolito modular** y extraer
servicios solo cuando haya una razón concreta: un módulo que necesita escalar distinto, o un equipo
que necesita desplegar por separado.

**Costo mínimo de entrada:** *service discovery*, API gateway, trazabilidad distribuida, gestión de
configuración, orquestación, contratos versionados, mensajería, monitoreo agregado. Si no tienes eso
resuelto, los microservicios te van a costar más de lo que resuelven.

---

## 7. Monolito modular

El punto medio y, hoy, **la recomendación por defecto** para la mayoría de sistemas.

```
┌──────────────────────────────────────────────────┐
│  Un solo despliegue, una sola base de datos      │
│                                                  │
│  ┌─────────┐ ┌───────────┐ ┌─────────┐ ┌──────┐ │
│  │clientes │ │originación│ │ cartera │ │factu │ │
│  │         │ │           │ │         │ │ración│ │
│  │ [api]   │ │  [api]    │ │  [api]  │ │[api] │ │
│  └────┬────┘ └─────┬─────┘ └────┬────┘ └───┬──┘ │
│       └────── solo por sus APIs públicas ──┘    │
└──────────────────────────────────────────────────┘
```

Reglas: cada módulo expone una API pública y **oculta** el resto; los módulos se comunican por esa
API o por eventos internos; cada módulo es dueño de sus tablas.

| A favor | En contra |
|---|---|
| Simplicidad operativa del monolito | Requiere disciplina (verificable con ArchUnit) |
| Fronteras claras, listas para extraer | Sigue siendo un solo despliegue |
| Transacciones locales, sin sagas | Un fallo grave afecta a todo |
| Refactorizar entre módulos es barato | |

---

## 8. Event-Driven Architecture

Los componentes se comunican publicando y consumiendo eventos.

```
Originación ──[CreditoDesembolsado]──► ┌──────┐ ──► Facturación
                                        │ Bus  │ ──► Notificaciones
Cartera     ──[CuotaVencida]─────────► └──────┘ ──► Cobranza
```

| A favor | En contra |
|---|---|
| Desacoplamiento fuerte | Flujo difícil de seguir |
| Extensible sin tocar al productor | Consistencia eventual |
| Absorbe picos de carga | Orden y duplicados: hay que diseñarlos |
| | Depuración compleja |

**Patrones imprescindibles cuando lo usas:** *Outbox* (publicar de forma consistente con la
transacción), idempotencia en el consumidor, *Dead Letter Queue*, versionado de eventos.

En CreditCore usaremos eventos **dentro** del monolito (eventos de dominio de Spring) y
**hacia afuera** vía SQS para tareas asíncronas (Fase 13). Es la dosis correcta.

---

## 9. CQRS

Separar el modelo de **escritura** (comandos, con reglas) del de **lectura** (consultas, optimizadas).

```
Comando ──► Modelo de escritura (agregados, invariantes) ──► BD
                                                             │
Consulta ◄── Modelo de lectura (proyecciones, SQL directo) ◄─┘
```

**Versión ligera (la que usaremos):** misma base de datos; las escrituras pasan por el dominio, las
consultas complejas usan proyecciones y SQL/JPQL directo sin cargar agregados completos. Es pragmático
y resuelve el 90% del problema.

**Versión completa:** bases separadas, *event sourcing*, sincronización asíncrona. Rara vez se
justifica.

---

## 10. Event Sourcing

En lugar de guardar el estado actual, se guarda **la secuencia de eventos** y el estado se reconstruye
reproduciéndolos.

```
Estado tradicional:   saldo = 3.500.000
Event sourcing:       Desembolsado(5.000.000) → PagoAplicado(800.000) → PagoAplicado(700.000)
```

| A favor | En contra |
|---|---|
| Auditoría perfecta por diseño | Complejidad alta |
| Se puede reconstruir el pasado | Consultas requieren proyecciones |
| Encaja perfecto en finanzas | Versionado de eventos es doloroso |

**No lo usaremos como arquitectura**, pero sí su idea clave: **los movimientos son append-only**
(BR-055). La contabilidad de doble partida es event sourcing con 500 años de antigüedad.

---

## 11. Tabla comparativa

| Criterio | Capas | Hexagonal | Monolito modular | Microservicios | Event-Driven |
|---|---|---|---|---|---|
| Curva de aprendizaje | Baja | Media | Media | Alta | Alta |
| Ceremonia / nº de clases | Baja | Alta | Media | Alta | Media |
| Testabilidad del negocio | Baja | Muy alta | Alta | Alta | Media |
| Complejidad operativa | Muy baja | Muy baja | Baja | Muy alta | Alta |
| Independencia tecnológica | Baja | Muy alta | Media | Alta | Alta |
| Escalado independiente | No | No | No | Sí | Parcial |
| Facilidad de depuración | Alta | Alta | Alta | Baja | Baja |
| Apto para dominio complejo | No | Sí | Sí | Sí | Sí |

---

## 12. Cómo elegir (el árbol de decisión honesto)

```
¿El dominio tiene reglas de negocio complejas?
├─ NO → Capas. Ya. No compliques un CRUD.
└─ SÍ
   ├─ ¿Varios equipos con despliegues independientes y necesidad real de escalar por separado?
   │  ├─ SÍ → Microservicios (y asume el costo operativo, que es real)
   │  └─ NO → Monolito modular
   └─ ¿El dominio debe sobrevivir a cambios de tecnología o tiene muchas integraciones?
      ├─ SÍ → + Hexagonal
      └─ NO → Capas con un paquete de dominio bien cuidado
```

Y la pregunta de control final, que vale más que todo lo anterior:

> **"¿Qué me va a doler dentro de dos años?"**
> Si la respuesta es "cambiar de base de datos": hexagonal.
> Si es "escalar el módulo de pagos": microservicios.
> Si es "entender qué hace este servicio de 2000 líneas": modularidad y DDD.
> Si es "nada, esto es un CRUD": no te compliques.

---

## Preguntas de control

1. ¿En qué dirección apuntan las dependencias en hexagonal y por qué importa?
2. ¿Cuál es la diferencia práctica entre hexagonal y Clean Architecture?
3. ¿Por qué una transacción debería modificar un solo agregado?
4. Da un motivo válido y un motivo inválido para adoptar microservicios.
5. ¿Qué problema resuelve el patrón Outbox y por qué no basta con publicar el evento tras el commit?
6. Tu `@Service` tiene 900 líneas. ¿Qué síntoma es y qué harías?
7. ¿Qué es un Value Object y por qué `Dinero` debería serlo?
