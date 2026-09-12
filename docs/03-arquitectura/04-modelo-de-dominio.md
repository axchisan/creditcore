# 04 — Modelo de dominio

> Este documento define **qué** existe en el dominio y **qué reglas protege cada objeto**.
> El código lo escribirás tú en las fases correspondientes; aquí está el diseño.

---

## 1. Lenguaje ubicuo

Términos del negocio → nombres en el código. **Sin traducir, sin abreviar, sin anglicismos mezclados.**

| El negocio dice | El código dice |
|---|---|
| Solicitud de crédito | `SolicitudCredito` |
| Desembolso | `Desembolso` |
| Plan de pagos | `PlanPagos` |
| Cuota | `Cuota` |
| Saldo insoluto | `saldoCapital` |
| Altura de mora | `diasMora` |
| Imputación de pago | `imputar(...)` |
| Abono a capital | `abonoCapital` |
| Rango de numeración | `RangoNumeracion` |

**Regla**: si el código dice algo que el negocio no diría, el nombre está mal.

---

## 2. Mapa de agregados

```
┌──────────────────┐        ┌─────────────────────┐        ┌──────────────────┐
│     Cliente      │        │  ProductoCredito    │        │ RangoNumeracion  │
│   «agregado»     │        │    «agregado»       │        │   «agregado»     │
└────────┬─────────┘        └──────────┬──────────┘        └────────┬─────────┘
         │ referencia por ID           │                            │
         │  ┌──────────────────────────┘                            │
         ▼  ▼                                                       │
┌─────────────────────┐                                             │
│  SolicitudCredito   │                                             │
│     «agregado»      │                                             │
│  + Evaluacion       │                                             │
│  + DecisionCredito  │                                             │
└──────────┬──────────┘                                             │
           │ evento: SolicitudDesembolsada                          │
           ▼                                                        │
┌──────────────────────────────────────┐                            │
│             Credito                  │                            │
│           «agregado raíz»            │                            │
│   + PlanPagos                        │      evento:               │
│       + Cuota (1..n)                 │   CreditoDesembolsado ─────┤
│   + Movimiento (1..n, append-only)   │   PagoAplicado             │
└──────────────────┬───────────────────┘                            ▼
                   │                                   ┌────────────────────┐
                   │ evento: PagoAplicado              │  DocumentoFiscal   │
                   ▼                                   │    «agregado»      │
         ┌────────────────────┐                        │  + LineaDocumento  │
         │    PagoRecibido    │                        │  + Impuesto        │
         │    «agregado»      │                        └────────────────────┘
         └────────────────────┘
```

### Reglas de agregados aplicadas aquí

1. **Un agregado se referencia por ID, nunca por objeto.** `SolicitudCredito` guarda un `ClienteId`,
   no un `Cliente`. Esto evita cargar medio sistema en memoria y define límites transaccionales.
2. **Una transacción modifica un solo agregado.** Aplicar un pago modifica `Credito`. Si además hay
   que facturar, se hace por evento, en otra transacción.
3. **La raíz es el único punto de entrada.** No existe `CuotaRepository`: a una `Cuota` se llega por
   su `Credito`.

---

## 3. Value Objects del kernel compartido

### `Dinero`

El VO más importante del proyecto. Encapsula la regla BR-043 y elimina toda una clase de bugs.

```java
public record Dinero(BigDecimal valor, Moneda moneda) implements Comparable<Dinero> {

    private static final int ESCALA = 2;
    private static final RoundingMode REDONDEO = RoundingMode.HALF_UP;

    public Dinero {                                    // constructor compacto de record
        Objects.requireNonNull(valor, "valor requerido");
        Objects.requireNonNull(moneda, "moneda requerida");
        valor = valor.setScale(ESCALA, REDONDEO);      // normaliza SIEMPRE
    }

    public static Dinero cop(String valor) {
        return new Dinero(new BigDecimal(valor), Moneda.COP);
    }

    public static Dinero cero() { return cop("0"); }

    public Dinero mas(Dinero otro)  { validarMoneda(otro); return new Dinero(valor.add(otro.valor), moneda); }
    public Dinero menos(Dinero otro){ validarMoneda(otro); return new Dinero(valor.subtract(otro.valor), moneda); }
    public Dinero por(BigDecimal f) { return new Dinero(valor.multiply(f), moneda); }

    public boolean esCero()      { return valor.compareTo(BigDecimal.ZERO) == 0; }
    public boolean esPositivo()  { return valor.compareTo(BigDecimal.ZERO) > 0; }
    public boolean esMayorQue(Dinero otro) { validarMoneda(otro); return valor.compareTo(otro.valor) > 0; }

    private void validarMoneda(Dinero otro) {
        if (moneda != otro.moneda) throw new MonedasIncompatiblesException(moneda, otro.moneda);
    }

    @Override public int compareTo(Dinero otro) { validarMoneda(otro); return valor.compareTo(otro.valor); }
}
```

**Lo que este VO te garantiza:**
- Imposible sumar pesos con dólares.
- Imposible olvidar la escala o el redondeo.
- Imposible comparar con `equals` y equivocarse (el record normaliza la escala en el constructor).
- El tipo documenta la intención: `Dinero cuota` dice más que `BigDecimal cuota`.

> 🧪 Este es el **primer ejercicio de TDD** del proyecto (Fase 01/06). Escribe primero
> `DineroTest`, luego la clase.

### `Tasa`

```java
public record Tasa(BigDecimal valorAnualEfectivo) {
    // conversión E.A. → periódica, con la potencia fraccionaria (BR: no dividir entre 12)
    public BigDecimal periodica(Periodicidad periodicidad) { ... }
    public boolean excedeUsura(Tasa topeUsura) { ... }
}
```

### Otros VO del kernel

| VO | Encapsula |
|---|---|
| `Periodicidad` | MENSUAL, QUINCENAL, BIMESTRAL… y su número de períodos al año |
| `NumeroDocumento` | Tipo + número, con validación de formato |
| `Plazo` | Número de cuotas, con validación de rango |
| `RangoFechas` | Inicio/fin con validación de orden |

---

## 4. Agregado `Cliente`

| Atributo | Tipo | Nota |
|---|---|---|
| `id` | `ClienteId` | UUID |
| `documento` | `NumeroDocumento` | único (BR-001) |
| `nombres`, `apellidos` | `String` | |
| `fechaNacimiento` | `LocalDate` | mayoría de edad (BR-002) |
| `contacto` | `DatosContacto` | email y celular (BR-003) |
| `informacionFinanciera` | `InformacionFinanciera` | ingresos, egresos |
| `estado` | `EstadoCliente` | ACTIVO, INACTIVO |

**Invariantes que protege:**
- No se puede construir un `Cliente` menor de edad.
- No se puede construir sin al menos un medio de contacto.
- `capacidadDePago()` = ingresos − egresos, nunca negativa.

**Comportamiento (no es un saco de setters):**
```java
public void actualizarContacto(DatosContacto nuevo)
public void inactivar(String motivo)
public Dinero capacidadDePagoMensual()
public boolean esMayorDeEdad(LocalDate aFecha)
```

---

## 5. Agregado `ProductoCredito`

| Atributo | Tipo |
|---|---|
| `id`, `codigo`, `nombre` | |
| `montoMinimo`, `montoMaximo` | `Dinero` |
| `plazoMinimo`, `plazoMaximo` | `Plazo` |
| `tasa` | `Tasa` |
| `sistemaAmortizacion` | `SistemaAmortizacion` (FRANCES, ALEMAN) |
| `periodicidad` | `Periodicidad` |
| `activo` | `boolean` |

**Comportamiento:** `validarMonto(Dinero)`, `validarPlazo(Plazo)`, `estaVigente()`.

---

## 6. Agregado `SolicitudCredito`

### Máquina de estados

```
          BORRADOR
             │ enviar()
             ▼
        EN_ESTUDIO ──evaluar()──┬──► RECHAZADA        (terminal)
             │                  ├──► ANALISIS_MANUAL
             │                  └──► APROBADA
             │                            │
   ANALISIS_MANUAL ──decidir()────────────┤
                                          │ (si monto > autonomía)
                                          ▼
                          PENDIENTE_SEGUNDA_APROBACION
                                          │ segundaAprobacion()
                                          ▼
                                      APROBADA
                                          │ aceptar()        caducar()
                                          ▼                     │
                                      ACEPTADA ─────────────► CADUCADA (terminal)
                                          │ desembolsar()
                                          ▼
                                     DESEMBOLSADA (terminal)
```

**Implementación sugerida** (se discutirá en F07): el enum conoce sus transiciones válidas.

```java
public enum EstadoSolicitud {
    BORRADOR      { public Set<EstadoSolicitud> siguientes() { return Set.of(EN_ESTUDIO); } },
    EN_ESTUDIO    { public Set<EstadoSolicitud> siguientes() { return Set.of(APROBADA, RECHAZADA, ANALISIS_MANUAL); } },
    // ...
    DESEMBOLSADA  { public Set<EstadoSolicitud> siguientes() { return Set.of(); } };

    public abstract Set<EstadoSolicitud> siguientes();

    public void validarTransicionA(EstadoSolicitud destino) {
        if (!siguientes().contains(destino)) {
            throw new TransicionInvalidaException(this, destino);
        }
    }
}
```

> 💡 Un enum con cuerpo por constante es una de esas cosas que se ven poco y son muy útiles.
> Alternativas que compararemos: `Map<Estado, Set<Estado>>`, patrón State con clases, y una tabla
> en base de datos. Cada una tiene su momento.

### Sub-objetos
- `Evaluacion`: puntaje, factores evaluados (lista de `FactorEvaluado`), versión de la política.
- `DecisionCredito`: tipo, motivo, responsable, fecha. **Inmutable y obligatoria** (BR-022).

---

## 7. Agregado `Credito` (el corazón)

```
Credito  «raíz»
 ├── id, numero, clienteId, productoId, solicitudId
 ├── condiciones: montoDesembolsado, tasaCongelada, plazo, periodicidad, sistemaAmortizacion
 ├── fechaDesembolso, fechaPrimerVencimiento
 ├── estado: EstadoCredito
 ├── saldoCapital: Dinero
 ├── diasMora: int, calificacion: CalificacionCartera
 ├── planPagos: PlanPagos
 │      └── cuotas: List<Cuota>   (número, vencimiento, capital, interés, saldo, estado)
 └── movimientos: List<Movimiento>  (APPEND-ONLY, BR-055)
```

### Estados

```
VIGENTE ⇄ EN_MORA ──► PAGADO (terminal)
   │                      ▲
   └──────────────────────┘
   │
   └──► CASTIGADO (terminal)   ANULADO (terminal)
```

### Comportamiento principal

```java
public ResultadoAplicacion aplicarPago(Dinero monto, LocalDate fechaValor, ReferenciaPago referencia)
public LiquidacionTotal liquidarA(LocalDate fecha)
public void abonarACapital(Dinero monto, ModalidadAbono modalidad)   // REDUCIR_PLAZO | REDUCIR_CUOTA
public void actualizarMoraA(LocalDate fecha)
public void reversarMovimiento(MovimientoId id, String motivo)
public Dinero saldoTotal()
```

### Invariantes del agregado

| Invariante | Regla |
|---|---|
| La suma de capital del plan = monto desembolsado | BR-040 |
| El saldo tras la última cuota es exactamente cero | BR-041 |
| El saldo de capital nunca es negativo | — |
| Los movimientos solo se agregan, nunca se modifican ni borran | BR-055 |
| No se aceptan pagos en estado PAGADO o ANULADO | BR-057 |
| La tasa es la congelada al desembolso, no la del producto hoy | BR-015 |

### Por qué `Credito` es la raíz y `Cuota` no

Porque **no tiene sentido modificar una cuota sin pasar por el crédito**: cambiar una cuota afecta
el saldo, y el saldo es una invariante del conjunto. Si `Cuota` fuera agregado propio, dos hilos
podrían modificar dos cuotas del mismo crédito simultáneamente y dejar el saldo inconsistente.

**Consecuencia técnica directa:** el bloqueo de concurrencia se hace sobre `Credito` (BR-058).

---

## 8. Agregado `PagoRecibido`

Representa un pago que **entró al sistema**, antes de aplicarse.

| Atributo | Nota |
|---|---|
| `id` | |
| `referenciaExterna` | `(psp, transaccionId)` — **UNIQUE** (BR-051) |
| `monto`, `fechaValor`, `medioPago` | |
| `estado` | RECIBIDO, APLICADO, RECHAZADO, REVERSADO |
| `payloadCrudo` | el JSON original, para auditoría y reproceso |

Separarlo de `Credito` permite recibir el pago aunque la aplicación falle, y reprocesarlo después.
**Ese desacoplamiento es lo que hace el sistema robusto.**

---

## 9. Agregados de facturación

- `RangoNumeracion`: prefijo, desde, hasta, vigencia, siguiente. Protege BR-070 y BR-071.
- `DocumentoFiscal`: tipo, consecutivo, líneas, impuestos, CUFE, estado, XML, referencias.

Estados: `BORRADOR → RESERVADO → FIRMADO → ENVIADO → ACEPTADO | RECHAZADO → (reenvío) → ...`

---

## 10. Eventos de dominio

| Evento | Lo publica | Lo consume |
|---|---|---|
| `ClienteRegistrado` | clientes | notificaciones |
| `SolicitudEvaluada` | originacion | auditoría |
| `SolicitudAprobada` | originacion | notificaciones |
| `CreditoDesembolsado` | creditos | facturacion, notificaciones, cartera |
| `PagoAplicado` | creditos | facturacion, notificaciones |
| `CreditoPagado` | creditos | notificaciones, facturacion |
| `CuotaVencida` | cartera | cobranza, notificaciones |
| `CreditoEnMora` | cartera | cobranza |
| `DocumentoFiscalAceptado` | facturacion | notificaciones |
| `DocumentoFiscalRechazado` | facturacion | alertas operativas |

**Convención**: nombre en **pasado**, inmutable (`record`), contiene identificadores y el mínimo de
datos necesarios, más `ocurridoEn` (`Instant`).

```java
public record CreditoDesembolsado(
        CreditoId creditoId,
        ClienteId clienteId,
        Dinero monto,
        LocalDate fechaDesembolso,
        Instant ocurridoEn
) implements EventoDominio { }
```

---

## 11. Servicios de dominio

Lógica que no pertenece naturalmente a una sola entidad:

| Servicio | Responsabilidad |
|---|---|
| `CalculadoraAmortizacion` | Genera el plan de pagos (Strategy: francés / alemán) |
| `MotorEvaluacionCredito` | Aplica knock-outs, calcula puntaje, decide |
| `ImputadorPagos` | Aplica el orden de imputación (BR-050) |
| `CalculadoraMora` | DPD, interés moratorio, calificación |
| `CufeCalculator` | Hash determinista del documento fiscal |

Son **Java puro y sin estado**: entrada → salida. Son el paraíso de las pruebas unitarias.

---

## 12. Diagrama de clases del núcleo (simplificado)

```
         «record» Dinero          «record» Tasa         «enum» Periodicidad
                ▲                       ▲                       ▲
                └───────────┬───────────┴───────────────────────┘
                            │ usados por todo el dominio
   ┌────────────────────────┴─────────────────────────┐
   │                                                  │
┌──┴───────────────┐                      ┌───────────┴──────────┐
│    Credito       │ 1              1..n  │       Cuota          │
│  «aggregate root»├──────────────────────┤  numero, vencimiento │
│                  │                      │  capital, interes    │
│ + aplicarPago()  │ 1              0..n  │  estado              │
│ + liquidarA()    ├──────────────────────┤       Movimiento     │
│ + actualizarMora()│                     │  tipo, monto, fecha  │
└──────────────────┘                      │  (append-only)       │
                                          └──────────────────────┘
```

---

## Preguntas de control

1. ¿Por qué `Dinero` es un `record` y no una clase con setters?
2. ¿Por qué `SolicitudCredito` guarda `ClienteId` y no `Cliente`?
3. ¿Por qué `Cuota` no tiene su propio repositorio?
4. ¿Qué invariante se rompería si permitiéramos modificar un `Movimiento`?
5. ¿Por qué la tasa se "congela" en el crédito en lugar de leerse del producto?
6. Un evento se llama `CrearFactura`. ¿Qué está mal en ese nombre?
7. ¿Dónde pondrías el cálculo del interés de mora: en `Credito`, en `Cuota` o en un servicio de
   dominio? Justifica.
