# 03 — Pasarelas de pago: conceptos e integración

> Objetivo: entender cómo funciona realmente un cobro en línea, y por qué la integración de pagos es
> el lugar donde la **idempotencia** deja de ser teoría y se vuelve dinero.

---

## 1. Quién es quién en un pago con tarjeta

```
 Cliente ──► Tu app ──► Pasarela (PSP) ──► Adquirente ──► Redes (Visa/MC) ──► Emisor
                            │                                                    │
                            └──────────────── autorización ◄─────────────────────┘
```

| Actor | Rol |
|---|---|
| **Tarjetahabiente** | Quien paga |
| **Comercio (merchant)** | Tú |
| **Pasarela / PSP** | Capa técnica que te abstrae del resto (Wompi, PayU, Mercado Pago, ePayco, Stripe) |
| **Adquirente** | Banco que recibe el dinero por el comercio |
| **Red** | Visa, Mastercard, Amex |
| **Emisor** | Banco que emitió la tarjeta y autoriza (o no) |

En Colombia, además de tarjetas hay medios locales muy relevantes: **PSE** (débito bancario en
línea), **Nequi/Daviplata**, **corresponsales bancarios** y **efectivo** (Efecty, Baloto).

---

## 2. El ciclo de vida de una transacción

```
      CREADA
        │  el cliente inicia el pago
        ▼
    PENDIENTE ─────────────────► EXPIRADA
        │                        (nunca se completó)
        │  el emisor responde
   ┌────┴─────┐
   ▼          ▼
APROBADA   RECHAZADA
   │        (fondos, fraude, datos)
   │
   │  captura (inmediata o diferida)
   ▼
CAPTURADA ──► REEMBOLSADA (total/parcial)
   │
   └────────► CONTRACARGO (el cliente disputó)
```

**Conceptos que se confunden:**

- **Autorización** ≠ **captura**. La autorización *reserva* el monto; la captura lo *cobra*.
  Muchos PSP hacen ambas en un paso ("captura automática"), pero no todos.
- **Aprobada** ≠ **dinero en tu cuenta**. Eso ocurre en el **settlement** (liquidación), días después.
  Por eso existe la **conciliación**.

---

## 3. Tokenización y PCI DSS

**Nunca almacenes números de tarjeta.** La regla práctica:

```
Cliente → (datos de tarjeta) → SDK/iframe de la pasarela → token
Tu backend solo ve: tok_xxxxx, últimos 4 dígitos, franquicia, fecha de expiración
```

Si los datos de tarjeta nunca tocan tu servidor, tu alcance PCI DSS se reduce drásticamente. Esto
**no es opcional** y define el diseño: tu API recibe un **token**, no una tarjeta.

En este proyecto, ninguna entidad, DTO, log o tabla contendrá jamás un PAN completo ni un CVV.
Lo verificaremos con una prueba y con una regla de análisis estático.

---

## 4. Webhooks: el corazón del problema

La pasarela te notifica los cambios de estado por HTTP POST a una URL tuya.

```
POST /api/webhooks/pagos/wompi
{
  "event": "transaction.updated",
  "data": { "transaction": { "id": "...", "status": "APPROVED", "reference": "CRED-001-CUOTA-3", ... } },
  "signature": { "checksum": "...", "properties": [...] },
  "timestamp": 1699999999
}
```

### Las cinco reglas de un webhook bien implementado

| # | Regla | Por qué |
|---|---|---|
| 1 | **Verificar la firma** antes de procesar | Cualquiera puede hacer POST a esa URL. Sin firma, te acreditan pagos falsos. |
| 2 | **Responder 200 rápido**, procesar después | Si tardas, la pasarela reintenta y te duplica el evento. |
| 3 | **Ser idempotente** | Los webhooks llegan **más de una vez**. Es un hecho, no una posibilidad. |
| 4 | **Tolerar desorden** | Puede llegar `APPROVED` después de `PENDING`… o antes. Valida transiciones. |
| 5 | **No confiar en el payload**: reconsultar a la pasarela | El payload puede ser antiguo o manipulado. La verdad está en su API. |

### Patrón de idempotencia que usaremos

```
1. Llega el webhook  →  calcular clave idempotente (id de transacción del PSP)
2. INSERT en tabla `pago_recibido` con UNIQUE(psp, transaccion_id)
      ├─ si viola la restricción única → ya fue procesado → responder 200 y salir
      └─ si inserta → continuar
3. Registrar el evento crudo (auditoría / reproceso)
4. Responder 200
5. Procesar ASÍNCRONAMENTE: aplicar el pago a la obligación (dentro de su propia transacción)
```

> 💡 Fíjate en el truco: **la base de datos es el mecanismo de idempotencia**, mediante una
> restricción `UNIQUE`. No un `if (yaExiste)` en Java — eso tiene una condición de carrera entre la
> consulta y la inserción.

---

## 5. Idempotencia de salida

También aplica cuando **tú** llamas a la pasarela. Si envías "cobrar $100.000" y se te cae la red
antes de recibir la respuesta: ¿se cobró o no? Si reintentas, ¿cobras dos veces?

La solución estándar es la cabecera **`Idempotency-Key`**: envías un UUID que identifica *tu
intención*. Si reintentas con la misma clave, la pasarela devuelve el resultado original sin volver
a cobrar.

```
POST /transactions
Idempotency-Key: 6a1f9c8e-... (el mismo en cada reintento del MISMO cobro)
```

Tu sistema debe **generar y persistir esa clave antes de la primera llamada**, no en cada intento.

---

## 6. Conciliación

Al final del día (o del ciclo de liquidación) se cruzan tres fuentes:

```
   Tus pagos registrados   ⟷   Reporte de la pasarela   ⟷   Extracto bancario
```

Diferencias típicas y qué significan:

| Diferencia | Significado | Acción |
|---|---|---|
| En la pasarela pero no en tu sistema | Webhook perdido | Reproceso: consultar y aplicar |
| En tu sistema pero no en la pasarela | Registro indebido o duplicado | Investigar, posible reverso |
| Montos distintos | Comisión descontada, redondeo, cambio parcial | Registrar la comisión como gasto |
| En banco pero no en la pasarela | Pago por otro canal | Aplicar manualmente |

**Implicación técnica**: necesitas un **job de reconciliación** que consulte la API de la pasarela
por rango de fechas y compare. Es la red de seguridad frente a webhooks perdidos. Lo construiremos
en la Fase 10.

---

## 7. Diseño de la integración en CreditCore

El dominio **no** debe saber que existe Wompi. Abstracción:

```
dominio/pagos/puerto/
├── PasarelaPagoPort
│     ├── IntencionPago crearIntencion(SolicitudCobro solicitud)
│     ├── EstadoTransaccion consultar(String transaccionId)
│     └── Reembolso reembolsar(String transaccionId, Dinero monto, String idempotencyKey)
└── VerificadorFirmaWebhookPort
      └── boolean esValida(String payload, String firma, Instant timestamp)

infraestructura/pagos/
├── wompi/WompiPasarelaAdapter        (implementación real, sandbox)
├── payu/PayuPasarelaAdapter          (segunda implementación → demuestra que la abstracción sirve)
└── fake/FakePasarelaAdapter          (desarrollo y pruebas: estados controlables)
```

**Por qué dos implementaciones reales**: una abstracción con una sola implementación es una
suposición. Con dos, se vuelve una verdad verificada. Es el ejercicio que demuestra que entendiste
el patrón puerto/adaptador.

---

## 8. Seguridad específica de pagos

| Amenaza | Mitigación |
|---|---|
| Webhook falsificado | Verificación de firma HMAC + validación del timestamp (anti-replay) |
| Replay de un webhook válido | Idempotencia + ventana temporal |
| Manipulación del monto desde el cliente | El monto **lo calcula el servidor**, nunca llega del front |
| Enumeración de referencias de pago | Referencias opacas (UUID), no secuenciales |
| Fuga de datos de tarjeta | Tokenización + prohibición de loguear el payload completo |
| Doble aplicación de un pago | Restricción `UNIQUE` + transacción + bloqueo del agregado |

---

## 9. Ambientes sandbox (para el proyecto)

| Pasarela | Sandbox | Notas |
|---|---|---|
| **Wompi** (Bancolombia) | Sí, con llaves de prueba | Muy usada en Colombia, documentación clara, soporta PSE y Nequi |
| **PayU LATAM** | Sí | Más antigua, muy extendida en empresas |
| **Mercado Pago** | Sí | Buena documentación |
| **Stripe** | Sí, excelente | No opera cobros locales en Colombia, pero su sandbox es el mejor para *aprender* el modelo |

**Plan**: el adaptador principal será **Wompi sandbox** (relevancia local) y el secundario un
**FakePasarela** controlable. Si al llegar a la Fase 10 conseguir credenciales resulta lento, se
trabaja primero contra el fake y luego se conecta el real — la arquitectura lo permite sin cambios.

---

## Preguntas de control

1. ¿Por qué tu backend nunca debe recibir el número de tarjeta?
2. El webhook de un pago de $500.000 llega 3 veces. ¿Qué mecanismo impide aplicarlo 3 veces y por
   qué no basta un `if (existe)`?
3. Llamas a la pasarela, se cae la red, no sabes si cobró. ¿Qué haces?
4. ¿Qué diferencia hay entre una transacción aprobada y el dinero en tu cuenta?
5. ¿Para qué sirve el job de conciliación si ya tienes webhooks?
6. ¿Por qué el monto a cobrar no puede venir del frontend?
