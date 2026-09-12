# 03 — Glosario de negocio

Referencia rápida. La explicación desarrollada está en [`../01-negocio/`](../01-negocio/).

## A. Crédito y cartera

| Término | Definición |
|---|---|
| **Originación** | Todo el proceso desde que alguien solicita un crédito hasta que se desembolsa el dinero. |
| **Solicitud (application)** | Petición formal de crédito. Tiene estados y puede ser rechazada. |
| **Deudor / obligado** | Persona natural o jurídica que asume la deuda. |
| **Codeudor** | Quien responde solidariamente si el deudor no paga. |
| **Capital / principal** | Monto prestado, sin intereses. |
| **Interés corriente** | Precio del dinero en el tiempo, pactado. Se causa mientras el crédito está al día. |
| **Interés moratorio** | Interés adicional que se cobra sobre la cuota vencida no pagada. |
| **Tasa E.A.** | Efectiva Anual. Tasa de referencia legal en Colombia. |
| **Tasa M.V.** | Mes Vencido. Tasa mensual que se aplica al saldo al final de cada mes. |
| **Conversión E.A. → M.V.** | `i_mv = (1 + i_ea)^(1/12) − 1`. No se divide entre 12. |
| **Usura** | Tope legal de tasa. En Colombia lo certifica la Superintendencia Financiera; superarlo es delito. |
| **Amortización** | Forma en que el crédito se paga en el tiempo (capital + intereses). |
| **Cuota fija (francés)** | Sistema donde todas las cuotas tienen el mismo valor; cambia la composición capital/interés. |
| **Plan de pagos** | Tabla con cada cuota: número, fecha, capital, interés, seguro, saldo. |
| **Desembolso** | Entrega efectiva del dinero al cliente. Es el hecho que "activa" el crédito. |
| **Saldo insoluto** | Capital pendiente de pago en un momento dado. |
| **Abono a capital** | Pago que reduce directamente el saldo, no los intereses. |
| **Prepago / pago anticipado** | Pagar antes de la fecha. En Colombia el deudor tiene derecho a prepagar sin penalidad (Ley 1555 de 2012). |
| **Mora** | Estado de un crédito con una o más cuotas vencidas sin pagar. |
| **Altura de mora / days past due (DPD)** | Días transcurridos desde el vencimiento de la cuota más antigua impagada. |
| **Cartera** | Conjunto de créditos vigentes de la entidad. |
| **Calificación de cartera** | Clasificación por riesgo (A, B, C, D, E) según altura de mora. |
| **Provisión** | Reserva contable por el riesgo de no recuperar la cartera. |
| **Castigo (write-off)** | Retirar contablemente un crédito considerado irrecuperable. |
| **Scoring** | Puntaje que estima la probabilidad de incumplimiento de un solicitante. |
| **Capacidad de pago** | Ingreso disponible del solicitante frente a la cuota propuesta. |
| **Nivel de endeudamiento** | Relación entre deudas y ingresos. |
| **Centrales de riesgo** | Datacrédito, TransUnion: historial crediticio. |
| **KYC (Know Your Customer)** | Proceso de identificación y verificación del cliente. |
| **SARLAFT** | Sistema de administración del riesgo de lavado de activos y financiación del terrorismo. |
| **Cupo** | Monto máximo aprobado para un cliente. |
| **Producto de crédito** | Configuración comercial: plazo mínimo/máximo, monto, tasa, garantías. |

## B. Facturación electrónica (Colombia — DIAN)

| Término | Definición |
|---|---|
| **DIAN** | Dirección de Impuestos y Aduanas Nacionales. Autoridad tributaria colombiana. |
| **Factura electrónica de venta** | Documento tributario electrónico obligatorio, con validación previa de la DIAN. |
| **UBL 2.1** | *Universal Business Language*: estándar XML sobre el que la DIAN define su anexo técnico. |
| **CUFE** | Código Único de Factura Electrónica. Hash que identifica unívocamente la factura. |
| **CUDE** | Código Único de Documento Electrónico. Equivalente para notas crédito/débito y documento soporte. |
| **CUDS** | Código único del documento soporte en adquisiciones a no obligados a facturar. |
| **Rango de numeración** | Autorización de la DIAN con prefijo, número inicial/final y vigencia. Sin rango vigente no se puede facturar. |
| **Resolución de facturación** | Acto administrativo que otorga el rango de numeración. |
| **Ambiente de habilitación** | Entorno de pruebas obligatorio antes de facturar en producción. |
| **Ambiente de producción** | Entorno real, con efectos tributarios. |
| **Proveedor tecnológico (PT)** | Empresa autorizada por la DIAN para prestar servicios de facturación electrónica. |
| **Facturador electrónico** | Quien emite directamente con su propio software habilitado. |
| **Firma digital / XAdES** | Firma electrónica avanzada aplicada al XML. Requiere certificado digital emitido por una entidad de certificación. |
| **Certificado digital** | Archivo `.p12`/`.pfx` con la llave privada usada para firmar. |
| **Validación previa** | La DIAN valida el documento **antes** de que tenga validez; devuelve aceptación o rechazo. |
| **ApplicationResponse** | XML de respuesta de la DIAN (o del receptor) sobre un documento. |
| **Representación gráfica** | El PDF legible de la factura, que debe incluir el código QR y el CUFE. |
| **Nota crédito** | Documento que disminuye o anula total/parcialmente una factura emitida. |
| **Nota débito** | Documento que aumenta el valor de una factura emitida. |
| **Documento soporte** | Documento que respalda una compra a un proveedor no obligado a facturar electrónicamente. |
| **RADIAN** | Registro de facturas electrónicas de venta como título valor (facturas negociables). |
| **Evento** | Acuse de recibo, recibo de bien/servicio, aceptación expresa o reclamo sobre una factura. |
| **IVA** | Impuesto al Valor Agregado. Tarifa general 19%; hay exentos y excluidos. |
| **Retención en la fuente** | Anticipo de impuesto que practica el pagador. |
| **ReteIVA / ReteICA** | Retenciones sobre IVA y sobre industria y comercio (municipal). |
| **Servicios financieros e IVA** | Los intereses de crédito suelen ser **excluidos** de IVA; las comisiones y servicios conexos normalmente sí son gravados. La clasificación tributaria concreta debe validarse con el área contable. |

> ⚠️ **Nota de responsabilidad**: la normativa de la DIAN cambia con frecuencia (resoluciones, versión
> del anexo técnico, esquemas XSD). **Ninguna cifra, versión o número de resolución de este repositorio
> debe darse por vigente sin verificarlo en la fuente oficial de la DIAN.** El objetivo del proyecto es
> aprender el mecanismo y la arquitectura de integración, no sustituir asesoría tributaria.

## C. Pagos y pasarelas

| Término | Definición |
|---|---|
| **PSP / pasarela de pago** | Intermediario que procesa pagos (Wompi, PayU, Mercado Pago, ePayco, Stripe). |
| **Adquirente (acquirer)** | Banco que procesa el cobro por el comercio. |
| **Emisor (issuer)** | Banco que emitió la tarjeta del cliente. |
| **Tokenización** | Reemplazar el número de tarjeta por un token, para no almacenar datos sensibles. |
| **PCI DSS** | Estándar de seguridad para el manejo de datos de tarjetas. Evitarlo es la razón de tokenizar. |
| **3-D Secure (3DS)** | Autenticación adicional del tarjetahabiente; traslada la responsabilidad del fraude. |
| **Autorización** | Reserva del monto en la tarjeta, sin cobrarlo aún. |
| **Captura** | Cobro efectivo de un monto previamente autorizado. |
| **Anulación (void)** | Cancelar una autorización no capturada. |
| **Reembolso (refund)** | Devolver dinero ya capturado. |
| **Contracargo (chargeback)** | El tarjetahabiente disputa el cobro y el banco revierte el dinero. |
| **PSE** | Medio de pago por débito bancario en línea, muy usado en Colombia. |
| **Webhook** | Notificación HTTP que la pasarela envía a tu backend cuando cambia el estado de una transacción. |
| **Idempotencia** | Garantía de que repetir la misma operación no produce un efecto duplicado. Crítico en pagos. |
| **Idempotency key** | Identificador que el cliente envía para que el servidor reconozca reintentos. |
| **Conciliación** | Cruce entre lo que tu sistema registró y lo que la pasarela/banco reporta. |
| **Settlement / liquidación** | Momento en que el dinero efectivamente llega a la cuenta del comercio. |
| **Referencia de pago** | Identificador que tu sistema envía y la pasarela devuelve, para amarrar el pago a la obligación. |
| **Doble gasto / doble aplicación** | Aplicar dos veces el mismo pago a una obligación. Es el error más caro de este dominio. |

## D. Trazabilidad y control

| Término | Definición |
|---|---|
| **Auditoría** | Registro inmutable de quién hizo qué, cuándo y sobre qué dato. |
| **Trazabilidad** | Capacidad de reconstruir la historia completa de una operación. |
| **Segregación de funciones** | Quien aprueba no desembolsa; quien registra no autoriza. |
| **Four-eyes principle** | Toda operación sensible requiere un segundo aprobador. |
| **Partida doble** | Principio contable: todo movimiento tiene débito y crédito equivalentes. |
| **Asiento contable** | Registro de un hecho económico en la contabilidad. |
