# 01 — El negocio del crédito, explicado desde cero

> Objetivo: que entiendas el negocio **antes** de modelarlo. No se puede diseñar bien un dominio
> que no se entiende. Todas las cifras son ilustrativas.

---

## 1. La idea central

Una entidad de crédito hace una sola cosa: **entrega dinero hoy y lo recupera en el futuro, con un
precio por el tiempo y por el riesgo**. Ese precio es el **interés**.

El negocio tiene tres riesgos:

| Riesgo | Pregunta | Cómo se gestiona |
|---|---|---|
| **Crédito** | ¿Me van a pagar? | Scoring, capacidad de pago, garantías, centrales de riesgo |
| **Liquidez** | ¿Tengo plata para desembolsar? | Tesorería, cupos |
| **Operativo** | ¿Mi sistema me está haciendo perder plata? | Controles, auditoría, conciliación, idempotencia |

**Tu software gestiona el riesgo operativo.** Un pago aplicado dos veces, un interés mal calculado o
una factura mal emitida son pérdidas reales y, a veces, sanciones.

---

## 2. El ciclo de vida de un crédito

```
   ┌──────────────┐
   │  SOLICITUD   │  El cliente pide un monto y un plazo
   └──────┬───────┘
          │ validación de datos, KYC, documentos
          ▼
   ┌──────────────┐
   │  EN ESTUDIO  │  Scoring, capacidad de pago, centrales de riesgo
   └──────┬───────┘
          │
   ┌──────┴────────────┬──────────────────┐
   ▼                   ▼                  ▼
┌─────────┐      ┌──────────┐      ┌────────────┐
│RECHAZADA│      │ APROBADA │      │ DESISTIDA  │
└─────────┘      └────┬─────┘      └────────────┘
                      │ el cliente acepta condiciones + firma
                      ▼
                 ┌──────────┐
                 │ACEPTADA  │
                 └────┬─────┘
                      │ DESEMBOLSO (sale el dinero)
                      ▼
              ╔═══════════════╗
              ║ CRÉDITO VIGENTE║  ← aquí nace la obligación y el plan de pagos
              ╚═══┬═══════╤═══╝
                  │       │
         pagos al │       │ incumplimiento
            día   │       ▼
                  │   ┌────────┐
                  │   │ EN MORA│──── pagos ───► vuelve a VIGENTE
                  │   └───┬────┘
                  │       │ mora prolongada
                  ▼       ▼
            ┌──────────┐ ┌──────────┐
            │ PAGADO   │ │ CASTIGADO│
            └──────────┘ └──────────┘
```

**Puntos clave para el modelado:**

- La **solicitud** y el **crédito** son dos cosas distintas. La solicitud puede morir sin crédito.
- El **desembolso** es el evento que transforma una solicitud aprobada en una obligación real.
  Es irreversible y es el punto más delicado del sistema (dinero saliendo).
- Un crédito **no cambia de estado por sí solo**: cambia por un pago, por el paso del tiempo
  (proceso de fin de día) o por una decisión administrativa.

---

## 3. Dinero, tasas e intereses

### 3.1 Regla número uno: nunca `double` ni `float`

```java
// ❌ MAL — produce errores de redondeo inaceptables en dinero
double saldo = 0.1 + 0.2;          // 0.30000000000000004

// ✅ BIEN
BigDecimal saldo = new BigDecimal("0.1").add(new BigDecimal("0.2"));  // 0.3
```

Reglas que usaremos en todo el proyecto:

1. Todo importe es `BigDecimal`, construido **desde String**, nunca desde `double`.
2. Toda división especifica escala y modo de redondeo: `.divide(x, 2, RoundingMode.HALF_UP)`.
3. Los importes en pesos colombianos se manejan con escala 2 y se redondean al final, no en cada paso.
4. Las tasas se manejan con escala alta (6 a 10 decimales) para no acumular error.
5. `equals` de `BigDecimal` compara escala: `2.0 != 2.00`. Para comparar valor se usa `compareTo`.

### 3.2 Tasas: E.A. vs M.V.

En Colombia la tasa se publica normalmente como **Efectiva Anual (E.A.)**, pero los créditos se
liquidan mensualmente. La conversión **no** es dividir entre 12:

```
i_mv = (1 + i_ea)^(1/12) − 1
```

Ejemplo: 24% E.A. →  `(1.24)^(1/12) − 1 = 0.018087...` ≈ **1.8087% mensual**
(dividir entre 12 daría 2% mensual: estarías cobrando de más, y eso es un problema legal).

### 3.3 Amortización en cuota fija (sistema francés)

Es el más común. Todas las cuotas valen lo mismo; lo que cambia es cuánto de la cuota va a interés
y cuánto a capital.

**Fórmula de la cuota:**

```
         P · i
C = ─────────────────
     1 − (1 + i)^(−n)
```

Donde `P` = capital, `i` = tasa periódica (M.V.), `n` = número de cuotas.

**Ejemplo completo** (verificado): préstamo de $1.000.000 a 6 meses, 24% E.A.

- `i_mv = (1.24)^(1/12) − 1 = 0.01808758248351...`
- Cuota = `(1.000.000 × 0.01808758) / (1 − (1.01808758)^(−6))` = **$177.375**

| # | Saldo inicial | Interés | Abono a capital | Cuota | Saldo final |
|---:|---:|---:|---:|---:|---:|
| 1 | 1.000.000 | 18.088 | 159.287 | 177.375 | 840.713 |
| 2 | 840.713 | 15.206 | 162.169 | 177.375 | 678.544 |
| 3 | 678.544 | 12.273 | 165.102 | 177.375 | 513.442 |
| 4 | 513.442 | 9.287 | 168.088 | 177.375 | 345.354 |
| 5 | 345.354 | 6.247 | 171.128 | 177.375 | 174.226 |
| 6 | 174.226 | 3.151 | 174.226 | **177.377** | **0** |

Observa el patrón: **al principio pagas casi todo interés, al final casi todo capital**.
Y observa la última cuota: vale $2 más. Ese es el **ajuste de cierre** del que hablamos abajo.

> Convención usada en la tabla: interés redondeado a peso (`HALF_UP`), tasa con precisión completa,
> y la última cuota amortiza el saldo restante exacto. Estos números son los **casos de prueba
> oficiales** del motor de amortización (Fase 6).

> 🧪 Este cálculo será el primer módulo que construyas con **TDD** (Fase 6). Es ideal: entrada clara,
> salida verificable, cero dependencias, y un error de un peso se detecta de inmediato.

**Detalle crítico — el ajuste de la última cuota**: por redondeo, la suma de los abonos a capital
casi nunca da exactamente el capital prestado. La convención es **ajustar la última cuota** para que
el saldo cierre en cero exacto. Si tu tabla no cierra en cero, está mal.

### 3.4 Otros sistemas de amortización

| Sistema | Característica | Uso típico |
|---|---|---|
| **Francés (cuota fija)** | Cuota constante | Consumo, libranza, vehículo |
| **Alemán (abono fijo a capital)** | Cuota decreciente, capital constante | Crédito empresarial |
| **Americano (bullet)** | Solo intereses; capital al final | Puente, tesorería |
| **Cuota variable / gradiente** | Cuota crece o decrece | Agropecuario, estacional |

En este proyecto implementaremos **francés** y **alemán**, precisamente para ejercitar el patrón
**Strategy** con un caso real.

---

## 4. Pagos: el punto donde más se pierde dinero

Cuando llega un pago hay que decidir **a qué se aplica**. El orden de imputación es una regla de
negocio y suele ser:

```
1. Intereses de mora
2. Otros conceptos (seguros, cobranza)
3. Intereses corrientes causados
4. Capital
```

Ejemplo: cuota de $176.720 con 10 días de mora e interés moratorio de $4.500. Si el cliente paga
$100.000, **no** se abona nada a capital hasta cubrir la mora y los intereses corrientes.

### Escenarios que tu sistema debe resolver

| Escenario | Problema |
|---|---|
| Pago parcial | ¿Qué se cubre y qué queda pendiente? |
| Pago en exceso | ¿Sobrante como saldo a favor, o abono a capital? |
| Pago anticipado | Ley 1555/2012: derecho a prepagar sin sanción; recalcular plan |
| Pago duplicado (el webhook llegó dos veces) | **Idempotencia obligatoria** |
| Pago reversado (chargeback) | Deshacer la aplicación sin corromper el histórico |
| Pago de un tercero | Trazabilidad de quién pagó |
| Pago en fecha distinta a la de registro | La fecha valor manda sobre la fecha de sistema |

> 💡 **Regla de oro contable**: los movimientos **no se borran ni se editan**. Un error se corrige
> con un movimiento de **reverso**, dejando ambos en el histórico. Esto se traduce directamente a
> tu diseño: la tabla de movimientos es *append-only*.

---

## 5. Mora

- Un crédito entra en mora cuando pasa la fecha de vencimiento de una cuota sin pago completo.
- La **altura de mora (DPD)** se calcula desde la cuota impagada **más antigua**.
- El interés moratorio se calcula sobre el **capital de la cuota vencida**, no sobre todo el saldo.
- Existe un tope legal: la tasa moratoria no puede exceder el límite de usura vigente.

Clasificación típica de cartera por altura de mora:

| Calificación | DPD | Lectura |
|---|---|---|
| A | 0–30 | Normal |
| B | 31–60 | Aceptable |
| C | 61–90 | Apreciable |
| D | 91–180 | Significativo |
| E | > 180 | Incobrable |

**Implicación técnica**: alguien tiene que "mover el reloj". Habrá un **proceso de cierre de día**
(Fase 8) que recorre la cartera, actualiza DPD, causa intereses de mora y recalifica. Es un
`@Scheduled`, debe ser idempotente (si corre dos veces el mismo día no debe duplicar nada) y debe
poder re-ejecutarse para una fecha pasada.

---

## 6. Scoring y decisión

El scoring estima la probabilidad de incumplimiento. En producción se usan modelos estadísticos;
aquí implementaremos un **motor de reglas explícito y auditable**, que es lo más común en entidades
pequeñas y lo más interesante de modelar:

```
Puntaje = Σ (peso_i × puntaje_factor_i)

Factores:
  - Edad
  - Ingresos declarados vs cuota propuesta  (capacidad de pago)
  - Nivel de endeudamiento
  - Antigüedad laboral
  - Historial interno (créditos previos y su comportamiento)
  - Consulta a central de riesgo (simulada)

Decisión:
  puntaje >= 750            → APROBADO automático
  600 <= puntaje < 750      → REQUIERE ANÁLISIS MANUAL
  puntaje < 600             → RECHAZADO
  + reglas de corte duro (knock-out) que rechazan sin importar el puntaje
```

**Reglas de corte duro (knock-out)**: reportado negativo vigente, menor de edad, sin capacidad de
pago, monto fuera del producto, lista restrictiva (SARLAFT).

**Requisito clave**: toda decisión debe quedar **explicada y almacenada**. "Rechazado" no basta; hay
que poder decir *por qué*, con los valores que se usaron y la versión de la política aplicada. Esto
es exigencia regulatoria y, para ti, un ejercicio precioso de modelado.

---

## 7. Actores del sistema

| Actor | Qué hace |
|---|---|
| **Cliente / solicitante** | Solicita crédito, consulta su plan, paga |
| **Asesor comercial** | Registra solicitudes, acompaña al cliente |
| **Analista de crédito** | Estudia solicitudes que requieren análisis manual |
| **Aprobador** | Autoriza montos por encima de cierto umbral |
| **Operaciones / tesorería** | Ejecuta desembolsos |
| **Cartera / cobranza** | Gestiona mora |
| **Contabilidad** | Concilia, emite facturación, cierra períodos |
| **Auditor** | Solo lectura, sobre todo |
| **Administrador** | Parametriza productos, tasas, usuarios |
| **Sistema externo: pasarela** | Notifica pagos vía webhook |
| **Sistema externo: DIAN/PT** | Valida facturación electrónica |
| **Proceso batch** | Cierre de día, causación, recalificación |

Esta tabla es directamente la base del modelo de roles y permisos (Fase 9).

---

## 8. Por qué este dominio es excelente para aprender

| Concepto técnico | Dónde aparece de forma natural |
|---|---|
| Máquina de estados | Ciclo de vida de solicitud y crédito |
| Value Objects | `Dinero`, `Tasa`, `Plazo`, `NumeroDocumento` |
| Agregados e invariantes | Crédito + cuotas + movimientos |
| Strategy | Sistemas de amortización |
| Aritmética exacta | `BigDecimal`, redondeo, cierre en cero |
| Transacciones y bloqueos | Aplicación concurrente de pagos |
| Idempotencia | Webhooks de pasarela |
| Eventos de dominio | `CréditoDesembolsado` → generar plan, notificar, facturar |
| Procesos batch | Cierre de día |
| Integración HTTP + resiliencia | Pasarela de pago, centrales de riesgo |
| Generación de documentos | Facturación electrónica (XML firmado + PDF) |
| Seguridad y auditoría | Roles, segregación de funciones, trazabilidad |
| Reportes y consultas complejas | Cartera, mora, proyecciones |

Prácticamente **todo** lo que se aprende en Spring Boot aparece aquí de forma justificada, sin
inventar excusas.

---

## Preguntas de control

1. ¿Por qué no se puede usar `double` para dinero? Da un ejemplo concreto.
2. Convierte 30% E.A. a tasa mensual vencida. (Resultado: 2.21045%) ¿Por qué no es 2.5%?
3. En un crédito a cuota fija, ¿por qué la primera cuota paga más interés que la última?
4. Llega un pago de $50.000 a un crédito con $8.000 de mora, $12.000 de interés corriente y
   $150.000 de capital vencido. ¿Cómo se aplica?
5. El webhook de la pasarela llega tres veces con el mismo pago. ¿Qué debe pasar y por qué?
6. ¿Cuál es la diferencia entre una solicitud aprobada y un crédito vigente?
7. ¿Por qué los movimientos no se borran nunca?
