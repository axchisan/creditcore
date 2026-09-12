# 04 — Catálogo de reglas de negocio

Cada regla tiene identificador estable `BR-xxx`. Se referencian desde los requerimientos, desde el
código (en el Javadoc de la clase que la implementa) y desde las pruebas (nombre del test).

**Convención**: el test que verifica `BR-012` se llama, por ejemplo,
`rechazaSolicitudCuandoLaCuotaSuperaLaCapacidadDePago_BR012()`.

---

## A. Clientes

| ID | Regla | Severidad |
|---|---|---|
| BR-001 | Un cliente se identifica unívocamente por (tipo de documento, número de documento). | Bloqueante |
| BR-002 | El solicitante debe ser mayor de edad (≥ 18 años) a la fecha de solicitud. | Knock-out |
| BR-003 | Un cliente debe tener al menos un medio de contacto verificado (email o celular). | Bloqueante |
| BR-004 | Un cliente marcado en lista restrictiva (SARLAFT) no puede recibir crédito. | Knock-out |
| BR-005 | Los datos personales de un cliente son modificables, pero todo cambio queda auditado. | Auditoría |

## B. Productos de crédito

| ID | Regla | Severidad |
|---|---|---|
| BR-010 | Un producto define monto mínimo/máximo, plazo mínimo/máximo, tasa y sistema de amortización. | Bloqueante |
| BR-011 | El monto solicitado debe estar dentro del rango del producto. | Knock-out |
| BR-012 | El plazo solicitado debe estar dentro del rango del producto. | Knock-out |
| BR-013 | La tasa efectiva anual del producto no puede exceder la tasa de usura vigente. | Knock-out |
| BR-014 | Un producto desactivado no admite nuevas solicitudes, pero los créditos vigentes continúan. | Bloqueante |
| BR-015 | Cambiar la tasa de un producto **no** afecta créditos ya desembolsados (la tasa se congela al desembolso). | Bloqueante |

## C. Solicitud y evaluación

| ID | Regla | Severidad |
|---|---|---|
| BR-020 | Un cliente no puede tener más de una solicitud en estudio para el mismo producto. | Bloqueante |
| BR-021 | La cuota estimada no puede superar el % máximo de capacidad de pago configurado (por defecto 40% del ingreso disponible). | Knock-out |
| BR-022 | Toda decisión (aprobación/rechazo) debe registrar: puntaje, factores evaluados, versión de la política y responsable. | Auditoría |
| BR-023 | Puntaje ≥ 750 ⇒ aprobación automática; 600–749 ⇒ análisis manual; < 600 ⇒ rechazo. | Bloqueante |
| BR-024 | Cualquier regla knock-out rechaza la solicitud sin importar el puntaje. | Bloqueante |
| BR-025 | Montos superiores al umbral de autonomía requieren aprobación de un segundo usuario con rol aprobador (four-eyes). | Bloqueante |
| BR-026 | Una solicitud aprobada caduca si no se acepta dentro de la vigencia de la oferta (por defecto 15 días calendario). | Bloqueante |
| BR-027 | Una solicitud rechazada no se reabre: se crea una nueva solicitud. | Bloqueante |

## D. Desembolso

| ID | Regla | Severidad |
|---|---|---|
| BR-030 | Solo se desembolsan solicitudes en estado ACEPTADA. | Bloqueante |
| BR-031 | El desembolso es irreversible: no existe "des-desembolsar". Un error se corrige con la anulación del crédito y movimientos de reverso. | Bloqueante |
| BR-032 | El desembolso genera el plan de pagos completo en la misma transacción. | Bloqueante |
| BR-033 | Quien aprueba no puede ser quien desembolsa (segregación de funciones). | Bloqueante |
| BR-034 | La fecha de la primera cuota se calcula desde la fecha de desembolso según la periodicidad del producto. | Bloqueante |
| BR-035 | El desembolso debe ser idempotente respecto a la solicitud: una solicitud produce a lo sumo un crédito. | Bloqueante |

## E. Plan de pagos y amortización

| ID | Regla | Severidad |
|---|---|---|
| BR-040 | La suma de los abonos a capital del plan debe ser exactamente igual al capital desembolsado. | Invariante |
| BR-041 | El saldo después de la última cuota debe ser exactamente cero. | Invariante |
| BR-042 | El ajuste por redondeo se aplica en la última cuota. | Bloqueante |
| BR-043 | Todo importe monetario se maneja con escala 2 y redondeo HALF_UP; las tasas con escala ≥ 6. | Invariante |
| BR-044 | El interés de cada período se calcula sobre el saldo insoluto al inicio del período. | Bloqueante |
| BR-045 | Si la fecha de vencimiento cae en día no hábil, se traslada según la política del producto (por defecto: día hábil siguiente). | Bloqueante |

## F. Pagos

| ID | Regla | Severidad |
|---|---|---|
| BR-050 | Orden de imputación: (1) interés de mora, (2) otros conceptos, (3) interés corriente, (4) capital. | Bloqueante |
| BR-051 | Un pago con la misma referencia externa (PSP + id de transacción) se procesa **una sola vez**. | Invariante |
| BR-052 | Un pago en exceso genera saldo a favor del cliente; no se devuelve automáticamente. | Bloqueante |
| BR-053 | El pago anticipado total no genera penalización (Ley 1555 de 2012) y liquida los intereses causados a la fecha. | Legal |
| BR-054 | Un abono extraordinario a capital recalcula el plan: se reduce plazo o cuota según elección del cliente. | Bloqueante |
| BR-055 | Los movimientos son *append-only*: un error se corrige con un movimiento de reverso, nunca con UPDATE o DELETE. | Invariante |
| BR-056 | La fecha valor del pago determina el cálculo de intereses, no la fecha de registro en el sistema. | Bloqueante |
| BR-057 | No se aceptan pagos sobre un crédito en estado PAGADO o ANULADO. | Bloqueante |
| BR-058 | La aplicación de un pago a un crédito es serializable: dos pagos simultáneos no pueden leer el mismo saldo. | Invariante |

## G. Mora

| ID | Regla | Severidad |
|---|---|---|
| BR-060 | Un crédito entra en mora al día siguiente del vencimiento de una cuota no pagada íntegramente. | Bloqueante |
| BR-061 | La altura de mora (DPD) se calcula desde la cuota vencida más antigua. | Bloqueante |
| BR-062 | El interés moratorio se calcula sobre el capital vencido, no sobre el saldo total. | Bloqueante |
| BR-063 | La tasa moratoria no puede exceder el límite legal de usura vigente. | Legal |
| BR-064 | La calificación de cartera se deriva del DPD: A(0-30) B(31-60) C(61-90) D(91-180) E(>180). | Bloqueante |
| BR-065 | El proceso de cierre de día es idempotente: ejecutarlo dos veces para la misma fecha no duplica causaciones. | Invariante |

## H. Facturación electrónica

| ID | Regla | Severidad |
|---|---|---|
| BR-070 | No se emite ningún documento sin un rango de numeración vigente y con disponibilidad. | Bloqueante |
| BR-071 | Los consecutivos se asignan sin saltos ni repeticiones. | Invariante |
| BR-072 | Un documento rechazado se corrige y se reenvía con **el mismo consecutivo**. | Bloqueante |
| BR-073 | Un documento aceptado no se modifica ni se elimina: se corrige con nota crédito o débito. | Invariante |
| BR-074 | El CUFE debe ser determinista: los mismos datos producen el mismo CUFE. | Invariante |
| BR-075 | Todo documento emitido se conserva (XML firmado y representación gráfica) por el período legal. | Legal |
| BR-076 | Los intereses corrientes de crédito se registran como concepto excluido de IVA; las comisiones y servicios, como gravados, según parametrización contable. | Bloqueante |

## I. Seguridad y auditoría

| ID | Regla | Severidad |
|---|---|---|
| BR-080 | Toda operación que modifique estado registra usuario, momento, IP y valores antes/después. | Auditoría |
| BR-081 | Los datos sensibles (documento, ingresos) solo son visibles para roles autorizados. | Seguridad |
| BR-082 | Nunca se almacena ni se registra en log un PAN completo ni un CVV. | Legal/PCI |
| BR-083 | Las credenciales y certificados viven en un gestor de secretos, jamás en el repositorio ni en el YAML. | Seguridad |
| BR-084 | Las sesiones expiran; los tokens de acceso tienen vida corta y se renuevan con refresh token. | Seguridad |

---

## Leyenda de severidad

| Severidad | Significado | Consecuencia técnica |
|---|---|---|
| **Invariante** | Nunca puede violarse, ni transitoriamente | Se protege en el dominio **y** con restricción en la base de datos |
| **Knock-out** | Rechaza la operación de forma inmediata | Se evalúa antes que cualquier otra regla |
| **Bloqueante** | Impide continuar la operación | Se valida en el caso de uso / dominio |
| **Legal** | Obligación normativa | Se documenta y se prueba explícitamente |
| **Auditoría** | Obligación de registro | Se implementa transversalmente |
| **Seguridad** | Control de acceso o protección de datos | Se prueba con casos negativos |
