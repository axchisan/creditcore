# 01 — Alcance del sistema

## 1. Descripción general

**CreditCore** es una plataforma de originación y gestión de crédito para una entidad financiera de
tamaño medio. Permite registrar clientes, evaluar y aprobar solicitudes, desembolsar créditos,
generar y mantener planes de pago, recibir pagos por múltiples canales, gestionar mora y emitir la
facturación electrónica asociada.

## 2. Contexto del sistema (C4 nivel 1)

```
                        ┌──────────────────────────────────┐
   Asesor / Analista ──►│                                  │
   Aprobador         ──►│                                  │──► Central de riesgo (simulada)
   Operaciones       ──►│           CreditCore             │
   Cartera           ──►│   (Spring Boot + PostgreSQL)     │──► Pasarela de pago (Wompi/PayU)
   Contabilidad      ──►│                                  │
   Auditor           ──►│                                  │──► Proveedor tecnológico → DIAN
   Administrador     ──►│                                  │
                        └───────┬──────────────┬───────────┘
   Cliente final ───────────────┘              │
   (portal de consulta y pago)                 ├──► Servicio de correo (notificaciones)
                                               ├──► S3 (documentos: XML, PDF, soportes)
                                               └──► SQS (procesamiento asíncrono)
```

## 3. Dentro del alcance

### Módulo 1 — Clientes
- Registro y consulta de personas naturales y jurídicas.
- Validación de identidad y datos de contacto.
- Historial de relación con la entidad.

### Módulo 2 — Productos de crédito
- Parametrización: montos, plazos, tasas, sistema de amortización, periodicidad.
- Versionado de la parametrización (la tasa se congela al desembolso).

### Módulo 3 — Originación
- Registro de solicitudes.
- Motor de evaluación: reglas knock-out + scoring ponderado + capacidad de pago.
- Flujo de aprobación con autonomías y doble aprobación.
- Aceptación de oferta y caducidad.

### Módulo 4 — Desembolso y crédito
- Desembolso con segregación de funciones.
- Generación del plan de pagos (sistemas francés y alemán).
- Consulta del estado del crédito, saldos y proyecciones.

### Módulo 5 — Recaudo
- Registro de pagos manuales y automáticos.
- Integración con pasarela de pago (intención de pago, webhooks, consulta).
- Aplicación de pagos con orden de imputación.
- Pagos anticipados y abonos extraordinarios con recálculo de plan.
- Conciliación contra la pasarela.

### Módulo 6 — Cartera y mora
- Proceso de cierre de día: causación, DPD, recalificación.
- Consultas de cartera por altura de mora.
- Gestión de cobranza (registro de gestiones y acuerdos de pago).

### Módulo 7 — Facturación electrónica
- Gestión de rangos de numeración.
- Generación de XML UBL, CUFE, firma y envío.
- Notas crédito y débito.
- Consulta de estado y reintentos.
- Representación gráfica (PDF con QR).

### Módulo 8 — Seguridad y auditoría
- Autenticación con JWT, roles y permisos.
- Auditoría de todas las operaciones que modifican estado.
- Trazabilidad de decisiones.

### Módulo 9 — Operación
- Health checks, métricas, logs estructurados.
- Documentación de API (OpenAPI).
- Reportes operativos.

## 4. Fuera del alcance (explícitamente)

| No incluido | Por qué |
|---|---|
| Interfaz gráfica de usuario (frontend) | El objetivo es backend; se consumirá con HTTP client/Postman. Opcionalmente, una UI mínima al final. |
| Contabilidad completa (partida doble, balances) | Se generan los eventos contables, pero no un módulo contable |
| Integración real con centrales de riesgo | Requiere contrato comercial; se simula con un adaptador *fake* |
| Firma electrónica del pagaré / desmaterialización | Dominio legal aparte |
| Aplicación móvil | Fuera de objetivos de aprendizaje |
| Motor de machine learning para scoring | Se usa un motor de reglas explícito y auditable |
| Multi-moneda | Solo COP |
| Multi-tenant | Una sola entidad |

## 5. Supuestos

1. Una sola moneda (COP) y una sola zona horaria (`America/Bogota`).
2. Las tasas se parametrizan como Efectiva Anual y se convierten a periódicas.
3. Los pagos llegan por pasarela o se registran manualmente; no hay integración bancaria directa.
4. La DIAN se integra a través de un proveedor tecnológico simulado en desarrollo.
5. El volumen objetivo de diseño es de decenas de miles de créditos, no millones.

## 6. Restricciones

| Restricción | Origen |
|---|---|
| Java 21 LTS + Spring Boot 4.1.x | Ver ADR-0002 |
| PostgreSQL como única base de datos | Decisión de proyecto |
| Todo el código lo escribe el estudiante a mano | Objetivo del proyecto |
| Sin frameworks de generación de código (Lombok en discusión — ver ADR-0004) | Objetivo del proyecto |
| Infraestructura simulada con LocalStack antes de AWS real | Control de costos |
| Toda la aritmética monetaria con `BigDecimal` | BR-043 |

## 7. Glosario de identificadores

| Prefijo | Significado | Ejemplo |
|---|---|---|
| `RF-` | Requerimiento funcional | RF-012 |
| `RNF-` | Requerimiento no funcional | RNF-004 |
| `BR-` | Regla de negocio | BR-050 |
| `HU-` | Historia de usuario | HU-007 |
| `CA-` | Criterio de aceptación | HU-007/CA-02 |
| `ADR-` | Registro de decisión arquitectónica | ADR-0003 |
| `F` | Fase del proyecto | F06 |
