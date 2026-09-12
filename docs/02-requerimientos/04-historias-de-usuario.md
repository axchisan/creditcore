# 04 — Historias de usuario y criterios de aceptación

## Cómo se escriben

```
HU-xxx — <Título corto>
Como <rol>
quiero <capacidad>
para <beneficio de negocio>

Criterios de aceptación (Gherkin):
  Escenario: <nombre>
    Dado <estado inicial>
    Cuando <acción>
    Entonces <resultado observable>
```

**Reglas de calidad de una historia (INVEST):**

| Letra | Significa | Pregunta de control |
|---|---|---|
| **I**ndependent | Independiente | ¿Puede construirse sin esperar a otra? |
| **N**egotiable | Negociable | ¿Describe el *qué*, no el *cómo*? |
| **V**aluable | Valiosa | ¿A quién le sirve? Si nadie la nota, sobra. |
| **E**stimable | Estimable | ¿Sé cuánto cuesta aproximadamente? |
| **S**mall | Pequeña | ¿Cabe en una o dos sesiones? |
| **T**estable | Verificable | ¿Puedo escribir la prueba antes del código? |

> Regla del proyecto: **si no puedes escribir el escenario Gherkin, no entendiste el requerimiento.**
> Los escenarios se escriben **antes** de programar y se convierten, casi literalmente, en nombres de
> métodos de prueba.

---

## Épica 1 — Clientes

### HU-001 — Registrar cliente persona natural
**Como** asesor comercial
**quiero** registrar un cliente con sus datos personales y de contacto
**para** poder asociarle solicitudes de crédito.

```gherkin
Escenario: Registro exitoso
  Dado que no existe un cliente con documento CC 1098765432
  Cuando registro un cliente con CC 1098765432, nombre "Ana Pérez",
        nacimiento 1995-04-12, email "ana@example.com" y celular "3001234567"
  Entonces el sistema responde 201 Created
    Y devuelve el identificador del cliente
    Y el cliente queda consultable por su documento

Escenario: Documento duplicado
  Dado que existe un cliente con documento CC 1098765432
  Cuando intento registrar otro cliente con CC 1098765432
  Entonces el sistema responde 409 Conflict
    Y el cuerpo es un Problem Details con type "cliente-duplicado"   # BR-001

Escenario: Menor de edad
  Dado un solicitante nacido hace 17 años
  Cuando intento registrarlo
  Entonces el sistema responde 422 Unprocessable Entity
    Y el error indica la regla BR-002

Escenario: Email inválido
  Cuando registro un cliente con email "no-es-un-email"
  Entonces el sistema responde 400 Bad Request
    Y el error señala el campo "email"
```
**Cubre:** RF-001, RF-002 · **Reglas:** BR-001, BR-002, BR-003 · **Fase:** F04

---

### HU-002 — Consultar clientes con filtros
**Como** asesor comercial
**quiero** buscar clientes por nombre o documento con resultados paginados
**para** encontrar rápidamente a quién atender.

```gherkin
Escenario: Búsqueda paginada
  Dado que existen 150 clientes
  Cuando consulto la página 0 con tamaño 20
  Entonces recibo 20 clientes
    Y la respuesta indica totalElements 150 y totalPages 8

Escenario: Filtro por documento parcial
  Dado que existe el cliente con documento 1098765432
  Cuando busco por "10987"
  Entonces el resultado incluye ese cliente

Escenario: Tamaño de página excesivo
  Cuando consulto con tamaño 5000
  Entonces el sistema limita el tamaño al máximo permitido (100)   # RNF-005
```
**Cubre:** RF-003, RF-004 · **Fase:** F04

---

## Épica 2 — Simulación y productos

### HU-010 — Simular un crédito
**Como** cliente potencial
**quiero** simular un crédito indicando monto y plazo
**para** conocer la cuota antes de solicitarlo.

```gherkin
Escenario: Simulación estándar
  Dado el producto "Libre Inversión" con tasa 24% E.A. y sistema francés
  Cuando simulo 1.000.000 a 6 meses
  Entonces la cuota es 177.375
    Y el plan tiene 6 cuotas
    Y la suma de abonos a capital es exactamente 1.000.000     # BR-040
    Y el saldo final es exactamente 0                          # BR-041

Escenario: Monto fuera de rango
  Dado el producto "Libre Inversión" con monto máximo 20.000.000
  Cuando simulo 50.000.000 a 12 meses
  Entonces el sistema responde 422
    Y el error indica la regla BR-011

Escenario: Producto inactivo
  Dado que el producto está inactivo
  Cuando intento simular
  Entonces el sistema responde 422 indicando BR-014
```
**Cubre:** RF-014, RF-047 · **Reglas:** BR-011, BR-012, BR-040..BR-045 · **Fase:** F06

---

## Épica 3 — Originación

### HU-020 — Crear solicitud de crédito
**Como** asesor comercial
**quiero** registrar la solicitud de crédito de un cliente
**para** iniciar su evaluación.

```gherkin
Escenario: Solicitud creada
  Dado un cliente activo y un producto activo
  Cuando creo una solicitud por 5.000.000 a 24 meses
  Entonces la solicitud queda en estado EN_ESTUDIO
    Y se registra quién la creó y cuándo

Escenario: Solicitud duplicada en estudio
  Dado que el cliente ya tiene una solicitud EN_ESTUDIO para ese producto
  Cuando creo otra solicitud para el mismo producto
  Entonces el sistema responde 409 indicando BR-020
```
**Cubre:** RF-020 · **Fase:** F07

---

### HU-021 — Evaluar automáticamente una solicitud
**Como** entidad
**quiero** que la solicitud se evalúe automáticamente
**para** decidir con criterios uniformes y auditables.

```gherkin
Escenario: Aprobación automática
  Dado un solicitante con puntaje calculado 780 y sin reglas knock-out
  Cuando se evalúa la solicitud
  Entonces la solicitud pasa a APROBADA
    Y se registra la decisión con puntaje, factores y versión de política   # BR-022

Escenario: Requiere análisis manual
  Dado un solicitante con puntaje 650
  Cuando se evalúa la solicitud
  Entonces la solicitud pasa a ANALISIS_MANUAL

Escenario: Rechazo por knock-out pese a buen puntaje
  Dado un solicitante con puntaje 800 pero reportado en lista restrictiva
  Cuando se evalúa la solicitud
  Entonces la solicitud pasa a RECHAZADA
    Y el motivo registrado es BR-004                                       # BR-024

Escenario: Rechazo por capacidad de pago
  Dado un solicitante con ingreso disponible 1.000.000
    Y una cuota calculada de 600.000
  Cuando se evalúa la solicitud
  Entonces la solicitud pasa a RECHAZADA por BR-021
```
**Cubre:** RF-021..RF-026 · **Fase:** F07

---

### HU-022 — Doble aprobación para montos altos
**Como** entidad
**quiero** exigir un segundo aprobador para montos por encima de la autonomía
**para** reducir el riesgo de fraude interno.

```gherkin
Escenario: Requiere segundo aprobador
  Dado un umbral de autonomía de 10.000.000
    Y una solicitud aprobada por 15.000.000
  Cuando el analista la aprueba
  Entonces la solicitud queda en PENDIENTE_SEGUNDA_APROBACION

Escenario: El mismo usuario no puede aprobar dos veces
  Dado una solicitud en PENDIENTE_SEGUNDA_APROBACION aprobada por el usuario U1
  Cuando el usuario U1 intenta la segunda aprobación
  Entonces el sistema responde 403 indicando BR-025
```
**Cubre:** RF-027 · **Fase:** F09

---

### HU-030 — Desembolsar un crédito
**Como** operador de tesorería
**quiero** desembolsar una solicitud aceptada
**para** entregar el dinero al cliente y activar la obligación.

```gherkin
Escenario: Desembolso exitoso
  Dado una solicitud en estado ACEPTADA por 5.000.000 a 24 meses
  Cuando ejecuto el desembolso
  Entonces se crea un crédito en estado VIGENTE
    Y se genera un plan de pagos con 24 cuotas                # BR-032
    Y la suma de capital del plan es 5.000.000                # BR-040
    Y se registra el movimiento de desembolso
    Y se publica el evento CreditoDesembolsado

Escenario: Solicitud no aceptada
  Dado una solicitud en estado EN_ESTUDIO
  Cuando intento desembolsarla
  Entonces el sistema responde 422 indicando BR-030

Escenario: Desembolso duplicado
  Dado una solicitud ya desembolsada
  Cuando ejecuto nuevamente el desembolso de esa solicitud
  Entonces el sistema no crea un segundo crédito              # BR-035
    Y devuelve el crédito existente

Escenario: Segregación de funciones
  Dado que el usuario U1 aprobó la solicitud
  Cuando el usuario U1 intenta desembolsarla
  Entonces el sistema responde 403 indicando BR-033
```
**Cubre:** RF-040..RF-043 · **Fase:** F07 (con la parte de seguridad en F09)

---

## Épica 4 — Recaudo

### HU-040 — Registrar un pago manual
**Como** cajero
**quiero** registrar un pago sobre un crédito
**para** reducir la deuda del cliente.

```gherkin
Escenario: Pago que cubre exactamente una cuota
  Dado un crédito con cuota 177.375 al día
  Cuando registro un pago de 177.375 con fecha valor hoy
  Entonces la cuota 1 queda PAGADA
    Y el saldo de capital se reduce en el abono correspondiente
    Y se generan los movimientos de aplicación

Escenario: Pago parcial con mora
  Dado un crédito con 8.000 de mora, 12.000 de interés corriente y 150.000 de capital vencido
  Cuando registro un pago de 50.000
  Entonces se aplican 8.000 a mora, 12.000 a interés y 30.000 a capital   # BR-050

Escenario: Pago en exceso
  Dado un crédito cuyo saldo total es 100.000
  Cuando registro un pago de 150.000
  Entonces el crédito queda PAGADO
    Y se registra un saldo a favor de 50.000                              # BR-052

Escenario: Crédito ya pagado
  Dado un crédito en estado PAGADO
  Cuando intento registrar un pago
  Entonces el sistema responde 422 indicando BR-057
```
**Cubre:** RF-050, RF-051, RF-052, RF-061 · **Fase:** F08

---

### HU-041 — Recibir pago por pasarela
**Como** cliente
**quiero** pagar mi cuota en línea
**para** no tener que ir a una oficina.

```gherkin
Escenario: Intención de pago
  Dado un crédito vigente con cuota pendiente de 177.375
  Cuando solicito pagar en línea
  Entonces el sistema crea una intención de pago en la pasarela
    Y devuelve los datos de checkout con una referencia única

Escenario: Webhook aprobado
  Dado una intención de pago con referencia REF-001
  Cuando la pasarela notifica estado APPROVED con firma válida
  Entonces el sistema reconsulta el estado en la pasarela          # RF-056
    Y aplica el pago al crédito
    Y responde 200 a la pasarela

Escenario: Webhook duplicado
  Dado que el webhook de REF-001 ya fue procesado
  Cuando la pasarela lo reenvía
  Entonces el sistema responde 200
    Y NO aplica un segundo pago                                    # BR-051

Escenario: Firma inválida
  Cuando llega un webhook con firma inválida
  Entonces el sistema responde 401
    Y no procesa nada
    Y registra el intento en auditoría de seguridad

Escenario: Pasarela caída al crear la intención
  Dado que la pasarela no responde
  Cuando solicito pagar en línea
  Entonces el sistema responde 503 con un mensaje comprensible
    Y el resto del sistema sigue operativo                         # RNF-020
```
**Cubre:** RF-054..RF-056 · **Fase:** F10

---

### HU-042 — Pago anticipado total
**Como** cliente
**quiero** pagar la totalidad de mi crédito antes de tiempo
**para** ahorrar intereses.

```gherkin
Escenario: Liquidación a la fecha
  Dado un crédito con 12 cuotas, de las cuales 4 están pagadas
  Cuando consulto la liquidación para pago total a hoy
  Entonces el valor incluye el saldo de capital
    Y los intereses causados solo hasta hoy
    Y NO incluye los intereses futuros no causados       # BR-053
    Y no se cobra penalización

Escenario: Pago total aplicado
  Cuando registro el pago por el valor de liquidación
  Entonces el crédito queda en estado PAGADO
    Y todas las cuotas pendientes quedan canceladas
```
**Cubre:** RF-046, RF-058 · **Fase:** F08

---

### HU-043 — Abono extraordinario a capital
**Como** cliente
**quiero** abonar a capital y elegir si se reduce el plazo o la cuota
**para** ajustar el crédito a mi capacidad.

```gherkin
Escenario: Reducción de plazo
  Dado un crédito de 24 cuotas con saldo 4.000.000
  Cuando abono 1.000.000 a capital eligiendo REDUCIR_PLAZO
  Entonces la cuota se mantiene igual
    Y el número de cuotas restantes disminuye
    Y el nuevo plan cierra en saldo cero                  # BR-041

Escenario: Reducción de cuota
  Cuando abono 1.000.000 a capital eligiendo REDUCIR_CUOTA
  Entonces el número de cuotas se mantiene
    Y el valor de la cuota disminuye
```
**Cubre:** RF-059 · **Fase:** F08

---

## Épica 5 — Cartera

### HU-050 — Cierre de día
**Como** entidad
**quiero** un proceso diario que actualice mora e intereses
**para** reflejar el estado real de la cartera.

```gherkin
Escenario: Crédito que entra en mora
  Dado un crédito con cuota vencida ayer y no pagada
  Cuando se ejecuta el cierre de día
  Entonces el crédito pasa a estado EN_MORA
    Y su DPD es 1
    Y se causa interés moratorio sobre el capital vencido    # BR-062

Escenario: Idempotencia del cierre
  Dado que el cierre del día 2026-03-15 ya se ejecutó
  Cuando se ejecuta nuevamente para 2026-03-15
  Entonces no se duplican causaciones ni movimientos        # BR-065

Escenario: Recalificación
  Dado un crédito con DPD 65
  Cuando se ejecuta el cierre
  Entonces su calificación es C                              # BR-064
```
**Cubre:** RF-070..RF-073 · **Fase:** F08

---

## Épica 6 — Facturación electrónica

### HU-060 — Emitir documento fiscal
**Como** contabilidad
**quiero** que el sistema emita la factura electrónica de los conceptos gravados
**para** cumplir la obligación tributaria.

```gherkin
Escenario: Emisión exitosa
  Dado un rango de numeración vigente con consecutivos disponibles
    Y un concepto facturable de comisión de estudio por 150.000
  Cuando se emite el documento
  Entonces se asigna el siguiente consecutivo sin saltos     # BR-071
    Y se genera el XML UBL
    Y se calcula el CUFE
    Y se firma digitalmente
    Y el documento queda en estado ENVIADO

Escenario: Sin rango vigente
  Dado que no hay rango de numeración vigente
  Cuando se intenta emitir
  Entonces el sistema rechaza la operación indicando BR-070
    Y no consume ningún consecutivo

Escenario: Rechazo de la DIAN y reenvío
  Dado un documento con consecutivo 990000123 rechazado por dato inválido
  Cuando se corrige y se reenvía
  Entonces se usa el mismo consecutivo 990000123             # BR-072

Escenario: CUFE determinista
  Dado los mismos datos de factura
  Cuando se calcula el CUFE dos veces
  Entonces el resultado es idéntico                          # BR-074

Escenario: Concurrencia sobre el consecutivo
  Dado dos emisiones simultáneas sobre el mismo rango
  Cuando ambas solicitan consecutivo
  Entonces reciben números distintos
    Y no se repite ni se salta ninguno                       # BR-071
```
**Cubre:** RF-080..RF-086 · **Fase:** F11

---

### HU-061 — Anular una factura con nota crédito
**Como** contabilidad
**quiero** emitir una nota crédito
**para** anular o disminuir una factura ya aceptada.

```gherkin
Escenario: Nota crédito total
  Dado una factura aceptada por 150.000
  Cuando emito una nota crédito total referenciando su CUFE
  Entonces la nota queda asociada a la factura
    Y la factura original NO se modifica ni se elimina       # BR-073
```
**Cubre:** RF-087 · **Fase:** F11

---

## Épica 7 — Seguridad

### HU-070 — Autenticación
```gherkin
Escenario: Login exitoso
  Cuando envío credenciales válidas
  Entonces recibo un access token y un refresh token
    Y el access token expira en 15 minutos o menos           # RNF-031

Escenario: Acceso sin token
  Cuando consulto un endpoint protegido sin token
  Entonces el sistema responde 401

Escenario: Rol insuficiente
  Dado un usuario con rol CONSULTA
  Cuando intenta desembolsar un crédito
  Entonces el sistema responde 403                           # BR-081
```
**Cubre:** RF-100, RF-101 · **Fase:** F09

---

## Plantilla para nuevas historias

```markdown
### HU-xxx — <Título>
**Como** <rol>
**quiero** <capacidad>
**para** <beneficio>.

```gherkin
Escenario: <camino feliz>
  Dado ...
  Cuando ...
  Entonces ...

Escenario: <caso borde>
  ...

Escenario: <caso de error>
  ...
```
**Cubre:** RF-xxx · **Reglas:** BR-xxx · **Fase:** Fxx
```

> **Regla del proyecto**: toda historia necesita al menos **un camino feliz, un caso borde y un caso
> de error**. Si solo se te ocurre el camino feliz, no has entendido el requerimiento.
