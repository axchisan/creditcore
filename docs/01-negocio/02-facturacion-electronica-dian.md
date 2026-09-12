# 02 — Facturación electrónica en Colombia (DIAN)

> ⚠️ **Aviso importante**: la normativa DIAN cambia con frecuencia (resoluciones, versiones del
> anexo técnico, esquemas XSD, URLs de los web services). Este documento explica **el mecanismo y la
> arquitectura de integración**, que son estables. Números de resolución, versiones y URLs concretas
> **deben verificarse siempre en la fuente oficial de la DIAN** antes de implementarse.
> Este material es de aprendizaje, no asesoría tributaria.

---

## 1. Por qué esto importa

En Colombia, emitir factura electrónica de venta es obligatorio para prácticamente todo el que
factura. Y, a diferencia de otros países, aquí hay **validación previa**: la DIAN valida el documento
*antes* de que tenga validez legal. Esto cambia por completo la arquitectura: no basta con generar un
PDF; hay que generar XML, firmarlo digitalmente, enviarlo, esperar respuesta y manejar los rechazos.

Es una de las integraciones **mejor pagadas y más demandadas** del mercado colombiano, precisamente
porque es tediosa, tiene mucha norma y un error tiene consecuencias fiscales.

---

## 2. Los actores

```
┌────────────┐   XML firmado   ┌──────────────────────┐   XML   ┌──────────┐
│  Tu ERP    │ ───────────────►│ Proveedor Tecnológico│────────►│   DIAN   │
│ (CreditCore)│◄─────────────── │        (PT)         │◄────────│          │
└────────────┘  CUFE + estado  └──────────────────────┘ Respuesta└──────────┘
       │
       │ PDF (representación gráfica) + XML
       ▼
   ┌────────┐
   │ Cliente│
   └────────┘
```

| Actor | Rol |
|---|---|
| **Facturador** | Quien vende y debe emitir. Puede emitir directo (software propio habilitado) o por medio de un PT. |
| **Proveedor Tecnológico (PT)** | Empresa autorizada por la DIAN que se encarga de firmar, transmitir y conservar. Ej.: Facture, Carvajal, Siigo, Alegra, Factus, Cadena. |
| **DIAN** | Valida y asigna validez legal. |
| **Adquiriente** | El cliente que recibe la factura (XML + representación gráfica). |

**Decisión de arquitectura de este proyecto**: integraremos contra un **PT** (patrón más realista
para una entidad de crédito) pero con una **abstracción propia** (`FacturacionElectronicaPort`), de
forma que se pueda cambiar de proveedor o pasar a emisión directa sin tocar el dominio. En desarrollo
usaremos un **adaptador simulado** (`FakeProveedorTecnologico`) que reproduce el protocolo completo
incluyendo rechazos.

---

## 3. Anatomía de una factura electrónica

El documento es un **XML UBL 2.1** (Universal Business Language) con extensiones propias de la DIAN.
Estructura conceptual:

```xml
<Invoice>
  <ext:UBLExtensions>          <!-- extensiones DIAN: CUFE, software, firma XAdES -->
  <cbc:UBLVersionID>UBL 2.1</cbc:UBLVersionID>
  <cbc:ID>SETP990000001</cbc:ID>            <!-- prefijo + consecutivo autorizado -->
  <cbc:UUID schemeName="CUFE-SHA384">...</cbc:UUID>
  <cbc:IssueDate/> <cbc:IssueTime/>
  <cbc:InvoiceTypeCode/>                     <!-- 01 factura de venta, etc. -->
  <cac:AccountingSupplierParty>  ...</cac:AccountingSupplierParty>  <!-- emisor -->
  <cac:AccountingCustomerParty>  ...</cac:AccountingCustomerParty>  <!-- adquiriente -->
  <cac:PaymentMeans/>                        <!-- contado/crédito, medio de pago -->
  <cac:TaxTotal/>                            <!-- IVA y otros impuestos -->
  <cac:LegalMonetaryTotal/>                  <!-- totales -->
  <cac:InvoiceLine>  ...</cac:InvoiceLine>   <!-- cada ítem -->
</Invoice>
```

### 3.1 El CUFE

El **CUFE** (Código Único de Factura Electrónica) es un **hash SHA-384** calculado concatenando
campos específicos del documento en un orden exacto definido por el anexo técnico. Conceptualmente:

```
CUFE = SHA384( NumFac + FecFac + HorFac + ValFac + CodImp1 + ValImp1 +
               CodImp2 + ValImp2 + CodImp3 + ValImp3 + ValTot +
               NitOFE + NumAdq + ClTec + TipoAmbiente )
```

**Por qué importa para tu código**: el CUFE es la huella digital del documento. Si un solo importe
cambia, el CUFE cambia. Esto obliga a que tu generación sea **determinista**: mismos datos ⇒ mismo
CUFE. Ese determinismo es perfectamente testeable y será uno de los ejercicios de la Fase 11.

> El orden exacto de los campos, su formato numérico y el separador los define el anexo técnico
> vigente. **Verificar siempre contra la versión oficial.**

### 3.2 La firma digital

El XML se firma con **XAdES** usando un certificado digital (archivo `.p12`) emitido por una entidad
de certificación autorizada. Sin firma válida, la DIAN rechaza.

**Implicación arquitectónica**: el certificado es un **secreto**. Nunca va en el repositorio. En
desarrollo se usa uno de prueba autofirmado; en producción vive en **AWS Secrets Manager** (Fase 16).

---

## 4. El flujo completo

```
 1. Se produce el hecho económico (ej.: se causó una comisión de estudio de crédito)
                    │
 2. Se reserva un CONSECUTIVO del rango de numeración vigente  ◄── ¡punto crítico!
                    │
 3. Se construye el XML UBL con todos los datos
                    │
 4. Se calcula el CUFE
                    │
 5. Se firma digitalmente (XAdES)
                    │
 6. Se envía al PT / DIAN  ──► respuesta síncrona o asíncrona
                    │
        ┌───────────┴────────────┐
        ▼                        ▼
   ACEPTADA                  RECHAZADA
        │                        │
 7a. Guardar CUFE,        7b. Guardar errores,
     XML y estado             NO consumir otro
        │                     consecutivo al
 8a. Generar PDF con         reintentar el mismo
     QR y CUFE                documento corregido
        │
 9a. Enviar al cliente (email)
        │
10a. Conservar 5 años (obligación legal)
```

### 4.1 El problema del consecutivo (el más interesante técnicamente)

La DIAN autoriza un **rango**: prefijo `SETP`, desde `990000001` hasta `995000000`, vigente hasta
cierta fecha. Reglas:

- Los números deben ser **consecutivos, sin saltos y sin repetirse**.
- No se puede facturar fuera del rango ni fuera de la vigencia.
- Un número consumido en un documento rechazado es un problema: hay que corregir y reenviar
  **ese mismo número**, no tomar el siguiente.

**Esto es un ejercicio perfecto de concurrencia y transaccionalidad.** ¿Qué pasa si dos hilos piden
un consecutivo al mismo tiempo? Opciones que analizaremos en la Fase 11:

| Opción | Pros | Contras |
|---|---|---|
| `SELECT ... FOR UPDATE` sobre la fila del rango | Simple, garantiza unicidad | Serializa la emisión |
| Secuencia de PostgreSQL | Rápida, sin bloqueos | Deja huecos si hay rollback (**inaceptable aquí**) |
| Bloqueo optimista con `@Version` + reintento | Sin bloqueo largo | Requiere manejar reintentos |
| Tabla de asignación con estado (`RESERVADO`/`USADO`/`ANULADO`) | Auditable, permite reintentar el mismo número | Más complejo |

*(Spoiler: la última es la que se usa en producción, y es la que implementaremos.)*

---

## 5. Tipos de documento

| Documento | Código | Cuándo |
|---|---|---|
| Factura electrónica de venta | 01 | Venta de bien o servicio |
| Nota crédito | 91 | Anular o disminuir una factura emitida |
| Nota débito | 92 | Aumentar el valor de una factura emitida |
| Documento soporte | 05 | Compra a no obligado a facturar |
| Nota de ajuste al documento soporte | 95 | Corregir un documento soporte |

**Regla clave**: una factura emitida y aceptada **no se modifica ni se borra**. Se corrige con una
nota crédito. Esto refuerza el mismo principio que vimos en el dominio de crédito: los documentos
contables son *append-only*.

---

## 6. Qué factura una entidad de crédito

Este es un punto de negocio que hay que entender bien:

| Concepto | ¿Se factura? | Nota |
|---|---|---|
| **Desembolso del capital** | No | Es un préstamo, no una venta |
| **Intereses corrientes** | Documento de cobro; los servicios financieros de intereses suelen ser **excluidos** de IVA | La clasificación exacta la define contabilidad |
| **Comisión de estudio / administración** | Sí, normalmente gravada | Es un servicio |
| **Seguros asociados** | Depende del esquema (intermediación) | Verificar |
| **Gastos de cobranza** | Sí, normalmente gravados | Es un servicio |

**Por eso el módulo de facturación se alimenta de eventos del dominio de crédito**, no de todos los
movimientos. En la Fase 11 definiremos exactamente qué eventos generan documento fiscal.

---

## 7. Ambientes

| Ambiente | Código típico | Uso |
|---|---|---|
| **Habilitación** | 2 | Pruebas obligatorias para obtener autorización. Los documentos **no** tienen efecto fiscal. |
| **Producción** | 1 | Documentos con validez legal. |

Antes de facturar en producción hay que **superar el set de pruebas de habilitación** de la DIAN: un
conjunto de documentos que debes emitir correctamente. En el proyecto simularemos este flujo.

---

## 8. Cómo lo vamos a construir (adelanto de la Fase 11)

```
dominio/facturacion/
├── DocumentoFiscal            (agregado: tipo, consecutivo, líneas, impuestos, estado, CUFE)
├── LineaDocumento             (VO)
├── Impuesto                   (VO: código, tarifa, base, valor)
├── RangoNumeracion            (agregado: prefijo, desde, hasta, vigencia, actual)
├── EstadoDocumentoFiscal      (enum: BORRADOR, RESERVADO, FIRMADO, ENVIADO, ACEPTADO, RECHAZADO, ANULADO)
└── puerto/
    ├── ProveedorTecnologicoPort   (enviar, consultarEstado)
    ├── FirmadorPort               (firmar XML)
    └── GeneradorRepresentacionGraficaPort  (PDF)

infraestructura/facturacion/
├── UblInvoiceBuilder          (construye el XML)
├── CufeCalculator             (hash determinista)
├── XadesFirmador              (implementación real)
├── FakeProveedorTecnologico   (desarrollo: simula aceptación/rechazo)
└── HttpProveedorTecnologico   (integración real contra el PT)
```

**Conceptos técnicos que practicarás aquí:**

- Generación y validación de XML (JAXB o construcción manual con DOM/StAX).
- Firma digital y criptografía (`KeyStore`, `Signature`, SHA-384).
- Determinismo y pruebas de *golden file* (comparar contra un XML de referencia).
- Concurrencia sobre un recurso escaso (el consecutivo).
- Máquina de estados con transiciones irreversibles.
- Integración HTTP con reintentos y manejo de errores de negocio vs técnicos.
- Almacenamiento de artefactos (XML/PDF) en S3.

---

## 9. Fuentes oficiales (verificar siempre)

- Portal DIAN — Factura electrónica: https://www.dian.gov.co/impuestos/factura-electronica/
- Anexo técnico de factura electrónica de venta (DIAN) — **la fuente de verdad del XML y del CUFE**.
- Documentación del proveedor tecnológico que se elija.
- Estándar UBL 2.1 (OASIS).

---

## Preguntas de control

1. ¿Qué diferencia hay entre "emitir" y "que la DIAN valide" una factura?
2. ¿Por qué el CUFE tiene que ser determinista? ¿Qué prueba escribirías para verificarlo?
3. La factura 990000123 fue rechazada por un error en el NIT del cliente. ¿Qué número usa el reenvío?
4. Dos procesos piden consecutivo simultáneamente. ¿Qué falla puede ocurrir y cómo lo evitas?
5. Un cliente pide anular una factura aceptada de hace dos meses. ¿Qué documento emites?
6. ¿Por qué el certificado digital no puede estar en el repositorio?
