# 02 — Requerimientos funcionales

Formato: `RF-xxx | Descripción | Reglas aplicables | Fase en que se construye`

---

## M1 — Clientes

| ID | Requerimiento | Reglas | Fase |
|---|---|---|---|
| RF-001 | El sistema debe permitir registrar un cliente persona natural con tipo y número de documento, nombres, apellidos, fecha de nacimiento, email, celular y dirección. | BR-001, BR-002, BR-003 | F04 |
| RF-002 | El sistema debe impedir el registro de dos clientes con el mismo tipo y número de documento. | BR-001 | F04 |
| RF-003 | El sistema debe permitir consultar un cliente por identificador interno y por documento. | — | F04 |
| RF-004 | El sistema debe permitir listar clientes con paginación, ordenamiento y filtro por nombre o documento. | — | F04 |
| RF-005 | El sistema debe permitir actualizar los datos de contacto de un cliente, registrando el cambio en auditoría. | BR-005, BR-080 | F04 |
| RF-006 | El sistema debe permitir inactivar un cliente sin eliminarlo. | BR-055 | F04 |
| RF-007 | El sistema debe registrar la información financiera del cliente (ingresos, egresos, actividad económica) usada para evaluar capacidad de pago. | BR-021 | F06 |

## M2 — Productos de crédito

| ID | Requerimiento | Reglas | Fase |
|---|---|---|---|
| RF-010 | El sistema debe permitir crear productos de crédito con monto mín/máx, plazo mín/máx, tasa E.A., sistema de amortización y periodicidad. | BR-010, BR-013 | F06 |
| RF-011 | El sistema debe rechazar productos cuya tasa exceda la usura vigente parametrizada. | BR-013 | F06 |
| RF-012 | El sistema debe permitir activar y desactivar productos. | BR-014 | F06 |
| RF-013 | El sistema debe versionar los cambios de parametrización para que los créditos vigentes conserven sus condiciones originales. | BR-015 | F06 |
| RF-014 | El sistema debe permitir simular un crédito (monto, plazo, producto) devolviendo la cuota y el plan proyectado, sin crear solicitud. | BR-040..BR-045 | F06 |

## M3 — Originación

| ID | Requerimiento | Reglas | Fase |
|---|---|---|---|
| RF-020 | El sistema debe permitir crear una solicitud de crédito para un cliente y un producto, con monto y plazo. | BR-011, BR-012, BR-020 | F07 |
| RF-021 | El sistema debe validar las reglas knock-out antes de cualquier evaluación. | BR-002, BR-004, BR-011, BR-012, BR-024 | F07 |
| RF-022 | El sistema debe calcular un puntaje de scoring a partir de factores ponderados y parametrizables. | BR-023 | F07 |
| RF-023 | El sistema debe calcular la capacidad de pago y rechazar si la cuota supera el porcentaje máximo configurado. | BR-021 | F07 |
| RF-024 | El sistema debe consultar la central de riesgo (adaptador simulado) e incorporar el resultado a la evaluación. | BR-023 | F07 |
| RF-025 | El sistema debe decidir automáticamente: aprobar, rechazar o enviar a análisis manual, según los umbrales. | BR-023 | F07 |
| RF-026 | El sistema debe registrar la decisión con puntaje, factores evaluados, versión de la política, responsable y fecha. | BR-022, BR-080 | F07 |
| RF-027 | El sistema debe exigir aprobación de un segundo usuario con rol aprobador cuando el monto supera el umbral de autonomía. | BR-025, BR-033 | F09 |
| RF-028 | El sistema debe permitir al analista aprobar o rechazar manualmente, con justificación obligatoria. | BR-022 | F07 |
| RF-029 | El sistema debe caducar automáticamente las solicitudes aprobadas no aceptadas dentro de la vigencia. | BR-026 | F08 |
| RF-030 | El sistema debe permitir al cliente aceptar la oferta, registrando fecha y condiciones aceptadas. | BR-026 | F07 |
| RF-031 | El sistema debe exponer la máquina de estados de la solicitud y rechazar transiciones inválidas. | BR-027 | F07 |

## M4 — Desembolso y crédito

| ID | Requerimiento | Reglas | Fase |
|---|---|---|---|
| RF-040 | El sistema debe permitir desembolsar una solicitud en estado ACEPTADA, creando el crédito. | BR-030, BR-035 | F07 |
| RF-041 | El sistema debe generar el plan de pagos completo en la misma transacción del desembolso. | BR-032, BR-040, BR-041 | F07 |
| RF-042 | El sistema debe impedir que el mismo usuario apruebe y desembolse. | BR-033 | F09 |
| RF-043 | El sistema debe garantizar que una solicitud produzca a lo sumo un crédito, aun ante peticiones concurrentes o repetidas. | BR-035 | F07 |
| RF-044 | El sistema debe calcular las fechas de vencimiento según periodicidad y política de días no hábiles. | BR-045 | F06 |
| RF-045 | El sistema debe permitir consultar el crédito con su saldo, plan, movimientos y estado actual. | — | F07 |
| RF-046 | El sistema debe permitir consultar la proyección de pago total a una fecha dada (liquidación). | BR-053 | F08 |
| RF-047 | El sistema debe soportar los sistemas de amortización francés y alemán mediante estrategias intercambiables. | BR-040..BR-044 | F06 |

## M5 — Recaudo

| ID | Requerimiento | Reglas | Fase |
|---|---|---|---|
| RF-050 | El sistema debe permitir registrar un pago manual sobre un crédito, con fecha valor, monto y medio. | BR-050, BR-056 | F08 |
| RF-051 | El sistema debe aplicar los pagos según el orden de imputación configurado. | BR-050 | F08 |
| RF-052 | El sistema debe generar movimientos *append-only* por cada aplicación de pago. | BR-055 | F08 |
| RF-053 | El sistema debe impedir aplicar dos veces un pago con la misma referencia externa. | BR-051 | F10 |
| RF-054 | El sistema debe generar una intención de pago en la pasarela y devolver la URL/datos de checkout. | — | F10 |
| RF-055 | El sistema debe recibir webhooks de la pasarela, verificar su firma y procesarlos de forma idempotente. | BR-051 | F10 |
| RF-056 | El sistema debe reconsultar el estado en la pasarela antes de aplicar el pago, sin confiar solo en el payload. | — | F10 |
| RF-057 | El sistema debe ejecutar un proceso de conciliación que detecte pagos registrados en la pasarela y no aplicados, y viceversa. | — | F10 |
| RF-058 | El sistema debe permitir el pago anticipado total, liquidando intereses a la fecha, sin penalización. | BR-053 | F08 |
| RF-059 | El sistema debe permitir abonos extraordinarios a capital con recálculo de plan (reducir plazo o reducir cuota). | BR-054 | F08 |
| RF-060 | El sistema debe permitir reversar un pago mediante movimiento de reverso, conservando el original. | BR-055 | F08 |
| RF-061 | El sistema debe manejar el excedente de un pago como saldo a favor del cliente. | BR-052 | F08 |
| RF-062 | El sistema debe garantizar que dos pagos concurrentes sobre el mismo crédito no corrompan el saldo. | BR-058 | F08 |

## M6 — Cartera y mora

| ID | Requerimiento | Reglas | Fase |
|---|---|---|---|
| RF-070 | El sistema debe ejecutar un proceso de cierre de día que actualice DPD, cause intereses de mora y recalifique la cartera. | BR-060..BR-065 | F08 |
| RF-071 | El proceso de cierre de día debe ser idempotente y re-ejecutable para una fecha pasada. | BR-065 | F08 |
| RF-072 | El sistema debe calcular el interés moratorio sobre el capital vencido, respetando el tope legal. | BR-062, BR-063 | F08 |
| RF-073 | El sistema debe calificar cada crédito (A–E) según su altura de mora. | BR-064 | F08 |
| RF-074 | El sistema debe permitir consultar la cartera filtrando por calificación, rango de DPD, producto y asesor. | — | F12 |
| RF-075 | El sistema debe permitir registrar gestiones de cobranza y acuerdos de pago. | BR-080 | F08 |
| RF-076 | El sistema debe marcar como PAGADO un crédito cuando su saldo llega a cero. | BR-041 | F08 |

## M7 — Facturación electrónica

| ID | Requerimiento | Reglas | Fase |
|---|---|---|---|
| RF-080 | El sistema debe permitir registrar y administrar rangos de numeración autorizados. | BR-070 | F11 |
| RF-081 | El sistema debe asignar consecutivos de forma segura ante concurrencia, sin saltos ni repeticiones. | BR-071 | F11 |
| RF-082 | El sistema debe generar el XML UBL 2.1 del documento a partir de los conceptos facturables. | BR-076 | F11 |
| RF-083 | El sistema debe calcular el CUFE/CUDE de forma determinista. | BR-074 | F11 |
| RF-084 | El sistema debe firmar digitalmente el XML con el certificado configurado. | BR-083 | F11 |
| RF-085 | El sistema debe enviar el documento al proveedor tecnológico y registrar la respuesta. | — | F11 |
| RF-086 | El sistema debe permitir reintentar el envío de un documento rechazado conservando el mismo consecutivo. | BR-072 | F11 |
| RF-087 | El sistema debe generar notas crédito y débito referenciando el documento original. | BR-073 | F11 |
| RF-088 | El sistema debe generar la representación gráfica (PDF) con QR y CUFE. | — | F11 |
| RF-089 | El sistema debe almacenar y permitir descargar el XML firmado y el PDF de cada documento. | BR-075 | F11 |
| RF-090 | El sistema debe generar automáticamente los documentos fiscales a partir de eventos del dominio de crédito. | BR-076 | F11 |

## M8 — Seguridad y auditoría

| ID | Requerimiento | Reglas | Fase |
|---|---|---|---|
| RF-100 | El sistema debe autenticar usuarios y emitir un token de acceso de vida corta y un refresh token. | BR-084 | F09 |
| RF-101 | El sistema debe autorizar cada operación según el rol del usuario. | BR-081 | F09 |
| RF-102 | El sistema debe registrar en auditoría toda operación que modifique estado, con usuario, momento, IP y valores antes/después. | BR-080 | F09 |
| RF-103 | El sistema debe impedir que un usuario vea datos sensibles sin el rol correspondiente. | BR-081 | F09 |
| RF-104 | El sistema debe permitir consultar el historial de auditoría filtrando por entidad, usuario y rango de fechas. | — | F09 |
| RF-105 | El sistema no debe almacenar ni registrar en logs datos completos de tarjeta. | BR-082 | F10 |

## M9 — Operación

| ID | Requerimiento | Reglas | Fase |
|---|---|---|---|
| RF-110 | El sistema debe exponer endpoints de salud (liveness/readiness) y métricas. | — | F12 |
| RF-111 | El sistema debe emitir logs estructurados con identificador de correlación por petición. | — | F12 |
| RF-112 | El sistema debe exponer la documentación OpenAPI de su API. | — | F14 |
| RF-113 | El sistema debe responder los errores en formato Problem Details (RFC 7807). | — | F04 |
| RF-114 | El sistema debe procesar de forma asíncrona las tareas que no requieren respuesta inmediata (notificaciones, facturación, generación de PDF). | — | F13 |
| RF-115 | El sistema debe garantizar la publicación consistente de eventos mediante el patrón Outbox. | — | F13 |
