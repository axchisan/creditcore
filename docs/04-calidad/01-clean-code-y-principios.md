# 01 — Clean Code y principios de diseño

> No es una lista para recitar en una entrevista. Cada principio se explica con **el problema que
> resuelve**, con ejemplos del dominio de este proyecto, y con la señal que indica que lo estás
> violando.

---

## Parte I — Clean Code

### 1. Nombres

**Regla:** el nombre debe responder *qué es* y *por qué existe*, sin necesidad de leer el cuerpo.

```java
// ❌
BigDecimal calc(BigDecimal m, int n, BigDecimal i) { ... }
List<Credito> getAll();
boolean flag;
var d = new Date();

// ✅
Dinero calcularCuotaFija(Dinero capital, Plazo plazo, BigDecimal tasaPeriodica) { ... }
List<Credito> buscarCreditosEnMora();
boolean estaEnMora;
LocalDate fechaVencimiento;
```

| Regla | Ejemplo |
|---|---|
| Sin abreviaturas | `numeroDocumento`, no `numDoc` |
| Sin prefijos húngaros | `cliente`, no `objCliente` ni `strNombre` |
| Booleanos con `es`/`tiene`/`puede` | `estaVigente`, `tieneMora`, `puedeDesembolsarse` |
| Métodos con verbo | `aplicarPago`, `calcularInteres` |
| Constantes en mayúsculas y con significado | `PORCENTAJE_MAXIMO_ENDEUDAMIENTO`, no `MAX` |
| Colecciones en plural | `cuotas`, no `listaCuota` |
| El nombre no miente | `obtenerCliente()` no debe además guardar nada |
| Longitud proporcional al alcance | `i` en un bucle de 3 líneas está bien; `c` como campo de clase, no |

> 💡 **Señal de alarma:** si necesitas un comentario para explicar qué hace una variable, el nombre
> está mal.

### 2. Funciones

```java
// ❌ Una función que hace de todo
public void procesarPago(UUID creditoId, BigDecimal monto, String ref) {
    // valida
    // busca el crédito
    // calcula mora
    // imputa
    // guarda
    // envía email
    // genera factura
    // ... 180 líneas
}

// ✅ Una función, un nivel de abstracción
public ResultadoAplicacion aplicar(AplicarPagoCommand comando) {
    Credito credito = buscarCredito(comando.creditoId());
    ResultadoAplicacion resultado = credito.aplicarPago(
            comando.monto(), comando.fechaValor(), comando.referencia());
    creditoRepository.guardar(credito);
    return resultado;
}
```

Reglas:

| Regla | Detalle |
|---|---|
| Hacer **una sola cosa** | Si puedes extraer otra función con un nombre con sentido, hacía dos cosas |
| Un solo nivel de abstracción | No mezcles "aplicar el pago" con "abrir una conexión" |
| Corta | Objetivo: < 20 líneas. Límite duro: 30 (RNF-044) |
| Pocos parámetros | 0–2 ideal, 3 aceptable, 4+ → agrupa en un objeto (`Command`) |
| Sin parámetros booleanos | `guardar(cliente, true)` no dice nada. Dos métodos, o un enum |
| Sin efectos secundarios ocultos | Un `validar()` que además persiste es una trampa |
| Devolver o lanzar, no códigos de error | Excepciones de dominio, no `-1` ni `null` |

### 3. Comentarios

**El mejor comentario es el que no hiciste falta escribir.** El código explica *el qué*; el
comentario explica *el por qué*, cuando no es obvio.

```java
// ❌ Ruido
// Incrementa el contador en 1
contador++;

// ❌ Comentario que miente (el código cambió, el comentario no)
// Aplica el 5% de interés
aplicarInteres(new BigDecimal("0.07"));

// ✅ Explica una decisión no obvia
// La DIAN exige que el CUFE se calcule con los importes SIN separador de miles
// y con punto decimal, independientemente del locale del servidor (anexo técnico §3.2).
String base = formatearParaCufe(total);

// ✅ Referencia a la regla de negocio
/**
 * Aplica el pago según el orden de imputación: mora → otros → interés → capital.
 * @see docs/01-negocio/04-catalogo-reglas-negocio.md (BR-050)
 */
public ResultadoAplicacion aplicarPago(...) { ... }
```

Comentarios **prohibidos** en este proyecto: código comentado (para eso está Git), `// TODO` sin
identificador, comentarios de bloque decorativos, y Javadoc que repite la firma del método.

### 4. Estructura y formato

- Un fichero, una clase pública.
- Orden dentro de la clase: constantes → campos → constructor → métodos públicos → métodos privados.
- Los métodos privados justo debajo del público que los usa (regla de "leer de arriba abajo").
- Líneas de máximo 120 caracteres.
- Sin líneas en blanco dentro de un método salvo para separar bloques con sentido.

### 5. Manejo de errores

```java
// ❌ Tragarse la excepción
try { pasarela.cobrar(...); } catch (Exception e) { }

// ❌ Log y seguir como si nada
catch (Exception e) { log.error("error"); }

// ❌ Excepción genérica sin contexto
throw new RuntimeException("error");

// ✅ Excepción de dominio con contexto suficiente para depurar
throw new CreditoNoAdmitePagosException(creditoId, estadoActual);

// ✅ Envolver conservando la causa
catch (IOException e) {
    throw new ComunicacionPasarelaException("Fallo al consultar la transacción " + transaccionId, e);
}
```

Reglas:
- Nunca capturar `Exception` para ignorarla.
- Nunca perder la causa (`e`) al relanzar.
- Las excepciones de dominio llevan los identificadores necesarios para reproducir el caso.
- No usar excepciones para control de flujo normal.
- `null` no es un valor de retorno válido: usa `Optional` o lanza.

### 6. Objetos y estructuras de datos

**Ley de Demeter** ("no hables con extraños"):

```java
// ❌ Tren de llamadas: conoces la estructura interna de tres objetos
credito.getPlanPagos().getCuotas().get(0).getSaldo();

// ✅ Pregunta al objeto, no lo destripes
credito.saldoDeLaPrimeraCuota();
```

**Tell, don't ask** — no preguntes el estado para decidir fuera; pídele al objeto que actúe:

```java
// ❌ La lógica vive fuera del objeto (modelo anémico)
if (credito.getEstado() == VIGENTE && credito.getSaldo().compareTo(ZERO) > 0) {
    credito.setSaldo(credito.getSaldo().subtract(monto));
    credito.setEstado(credito.getSaldo().compareTo(ZERO) == 0 ? PAGADO : VIGENTE);
}

// ✅ La lógica vive donde están los datos
credito.aplicarPago(monto, fechaValor, referencia);
```

> **Este es, con diferencia, el error más común en proyectos Spring**: entidades con getters y
> setters y toda la lógica en servicios. Es el "modelo anémico" y lo evitamos por diseño.

### 7. Las tres leyes del código limpio, en versión corta

1. **No te repitas (DRY)** — pero cuidado: duplicación accidental ≠ duplicación real. Dos reglas que
   hoy coinciden y mañana divergen **no** deben unificarse.
2. **Lo simple primero (KISS)** — la solución más simple que funcione y se pueda cambiar.
3. **No lo vas a necesitar (YAGNI)** — no construyas para requisitos imaginarios. Este ADR-0001 está
   lleno de ejemplos de cosas que *podríamos* hacer y no hacemos.

---

## Parte II — SOLID

### S — Single Responsibility Principle

> Una clase debe tener **una sola razón para cambiar**.

```java
// ❌ Tres razones para cambiar: reglas de pago, formato de email, esquema de BD
class PagoService {
    void aplicarPago() { ... }
    void enviarEmailConfirmacion() { ... }
    void guardarEnBaseDeDatos() { ... }
}

// ✅ Una razón cada una
class AplicarPagoService { }          // cambia si cambian las reglas de aplicación
class NotificadorPagos { }            // cambia si cambia la comunicación
class CreditoRepositoryAdapter { }    // cambia si cambia la persistencia
```

**Señal de violación:** el nombre de la clase necesita una "y" (`ClienteYCreditoService`).

### O — Open/Closed Principle

> Abierto a la extensión, cerrado a la modificación.

```java
// ❌ Cada sistema nuevo obliga a tocar esta clase
BigDecimal calcularCuota(String sistema) {
    if (sistema.equals("FRANCES")) { ... }
    else if (sistema.equals("ALEMAN")) { ... }
    // agregar AMERICANO = modificar aquí y arriesgar lo existente
}

// ✅ Strategy: agregar un sistema = agregar una clase
public interface SistemaAmortizacion {
    PlanPagos generar(Dinero capital, Tasa tasa, Plazo plazo, LocalDate primerVencimiento);
}
class AmortizacionFrancesa implements SistemaAmortizacion { ... }
class AmortizacionAlemana  implements SistemaAmortizacion { ... }
```

Este es exactamente el caso real de la Fase 06.

### L — Liskov Substitution Principle

> Un subtipo debe poder sustituir a su tipo base sin romper al que lo usa.

```java
// ❌ Viola LSP: el que usa Pago no puede confiar en que reversar() funcione
class Pago { void reversar() { ... } }
class PagoEnEfectivo extends Pago {
    @Override void reversar() { throw new UnsupportedOperationException(); }
}

// ✅ Modela la capacidad, no la fuerces en la jerarquía
interface Reversable { void reversar(); }
```

**Señal de violación:** un `@Override` que lanza `UnsupportedOperationException`, o un `instanceof`
antes de llamar a un método de la interfaz.

### I — Interface Segregation Principle

> Mejor muchas interfaces específicas que una general.

```java
// ❌ El adaptador de facturación no necesita reembolsar nada
interface ServicioExterno {
    void cobrar(); void reembolsar(); void facturar(); void notificar();
}

// ✅ Interfaces por capacidad
interface PasarelaPagoPort { IntencionPago crear(...); Reembolso reembolsar(...); }
interface ProveedorTecnologicoPort { RespuestaDian enviar(DocumentoFirmado doc); }
```

### D — Dependency Inversion Principle

> Los módulos de alto nivel no deben depender de los de bajo nivel. Ambos dependen de abstracciones.

```java
// ❌ El caso de uso depende de una tecnología concreta
class AplicarPagoService {
    private final CreditoJpaRepository repo;    // ¡JPA dentro de la aplicación!
}

// ✅ Depende de una abstracción que el propio dominio define
class AplicarPagoService implements AplicarPagoUseCase {
    private final CreditoRepository repo;       // interfaz del dominio
}
```

**DIP es el principio sobre el que se construye toda la arquitectura hexagonal.** Si entiendes este,
entiendes hexagonal.

---

## Parte III — Otros principios que aplicaremos

| Principio | Qué dice | Dónde aparece aquí |
|---|---|---|
| **Composición sobre herencia** | Prefiere tener un objeto a heredar de él | Estrategias de amortización |
| **Programar contra interfaces** | Depende del contrato, no de la implementación | Todos los puertos |
| **Fail fast** | Falla en cuanto detectes el error, no después | Validación en constructores de VO |
| **Principle of least astonishment** | El código debe hacer lo que su nombre sugiere | Naming |
| **Command-Query Separation** | Un método o cambia estado o devuelve datos, no ambos | `aplicarPago` vs `saldoTotal` |
| **Inmutabilidad por defecto** | Haz todo `final` salvo que deba cambiar | Value Objects, eventos, DTOs |
| **Encapsulación real** | No expongas colecciones internas modificables | `List.copyOf(cuotas)` |
| **Boy Scout Rule** | Deja el código mejor de como lo encontraste | En cada fase |

### Ejemplo de encapsulación real

```java
// ❌ Cualquiera puede modificar tus cuotas por fuera
public List<Cuota> getCuotas() { return cuotas; }

// ✅ Vista inmutable
public List<Cuota> cuotas() { return List.copyOf(cuotas); }
```

---

## Parte IV — Olores de código (code smells)

| Olor | Síntoma | Refactor |
|---|---|---|
| **Método largo** | > 30 líneas | Extract Method |
| **Clase grande** | > 300 líneas, muchas responsabilidades | Extract Class |
| **Lista de parámetros larga** | 4+ parámetros | Introduce Parameter Object |
| **Obsesión por primitivos** | `String documento`, `BigDecimal monto` por todas partes | Value Object |
| **Envidia de características** | Un método usa más datos de otra clase que de la suya | Move Method |
| **Cadenas de mensajes** | `a.getB().getC().getD()` | Hide Delegate / Tell-don't-ask |
| **Cirugía con escopeta** | Un cambio obliga a tocar 10 clases | Mal reparto de responsabilidades |
| **Código duplicado** | El mismo bloque en 3 sitios | Extract Method/Class (si es duplicación *real*) |
| **Comentarios excesivos** | El código necesita explicación constante | Renombrar y extraer |
| **Condicional complejo** | `if` con 5 condiciones | Extract Method con nombre explicativo |
| **Switch repetido** | El mismo switch en varios sitios | Polimorfismo / Strategy |
| **Clase de datos** | Solo getters y setters | Mover comportamiento a la clase |

> 💡 **IntelliJ te señala muchos de estos solos.** En la Fase 00 configuraremos las inspecciones
> para que aparezcan mientras escribes.

---

## Preguntas de control

1. ¿Qué está mal en `void procesar(Credito c, boolean esReverso)`?
2. Tienes una entidad con 12 getters, 12 setters y ningún otro método. ¿Qué olor es y qué implica?
3. Explica DIP con un ejemplo de este proyecto.
4. ¿Cuándo **no** deberías eliminar código duplicado?
5. ¿Por qué `credito.getPlanPagos().getCuotas().get(0)` es un problema?
6. ¿Qué diferencia hay entre encapsular una lista y devolverla tal cual?
