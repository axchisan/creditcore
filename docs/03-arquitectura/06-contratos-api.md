# 06 — Contratos y convenciones de la API REST

## 1. Principios

1. La API expone **recursos**, no procedimientos. `/creditos/{id}/pagos`, no `/aplicarPago`.
2. Los verbos HTTP tienen semántica: `GET` no modifica, `PUT` es idempotente, `POST` no lo es.
3. Los códigos de estado se usan con precisión (ver tabla más abajo).
4. Los errores siempre en **Problem Details (RFC 7807)**.
5. La API se versiona en la ruta: `/api/v1/...`.
6. Los DTO de la API **no** son entidades de dominio ni entidades JPA. Nunca.
7. Toda colección se pagina. Sin excepciones (RNF-005).

## 2. Estructura de rutas

```
/api/v1
├── /clientes
│   ├── GET    /                          listar (paginado, filtros)
│   ├── POST   /                          registrar
│   ├── GET    /{id}                      consultar
│   ├── PUT    /{id}/contacto             actualizar contacto
│   └── POST   /{id}/inactivacion         inactivar
│
├── /productos
│   ├── GET    /                          listar
│   ├── POST   /                          crear
│   └── POST   /{id}/simulaciones         simular crédito
│
├── /solicitudes
│   ├── POST   /                          crear solicitud
│   ├── GET    /{id}                      consultar
│   ├── POST   /{id}/evaluacion           evaluar
│   ├── POST   /{id}/decision             decidir manualmente
│   ├── POST   /{id}/aprobacion-adicional  segunda aprobación
│   ├── POST   /{id}/aceptacion           aceptar oferta
│   └── POST   /{id}/desembolso           desembolsar
│
├── /creditos
│   ├── GET    /                          listar (filtros de cartera)
│   ├── GET    /{id}                      consultar
│   ├── GET    /{id}/plan-pagos           plan de pagos
│   ├── GET    /{id}/movimientos          movimientos (paginado)
│   ├── GET    /{id}/liquidacion          liquidación a una fecha
│   ├── POST   /{id}/pagos                registrar pago manual
│   ├── POST   /{id}/abonos-capital       abono extraordinario
│   └── POST   /{id}/intenciones-pago     iniciar pago en línea
│
├── /documentos-fiscales
│   ├── GET    /{id}
│   ├── POST   /{id}/reenvio
│   └── POST   /{id}/notas-credito
│
├── /webhooks
│   └── POST   /pagos/{psp}               recepción de webhooks
│
└── /auth
    ├── POST   /login
    └── POST   /refresh
```

> 💡 **Truco de diseño REST**: cuando una acción no encaja en CRUD, modélala como la **creación de un
> recurso que representa esa acción**. "Desembolsar" → `POST /solicitudes/{id}/desembolso`.
> Esto mantiene la semántica REST y te da un lugar natural donde devolver el resultado.

## 3. Códigos de estado

| Código | Cuándo se usa aquí |
|---|---|
| `200 OK` | Consulta exitosa, o acción que devuelve resultado |
| `201 Created` | Recurso creado. **Siempre** con cabecera `Location` |
| `202 Accepted` | Aceptado para procesamiento asíncrono |
| `204 No Content` | Acción exitosa sin cuerpo |
| `400 Bad Request` | JSON mal formado, tipo incorrecto, campo faltante |
| `401 Unauthorized` | Sin token o token inválido |
| `403 Forbidden` | Autenticado pero sin permiso (incluye segregación de funciones) |
| `404 Not Found` | El recurso no existe |
| `409 Conflict` | Conflicto de estado: duplicado, versión desactualizada |
| `422 Unprocessable Entity` | **Regla de negocio violada** (sintaxis correcta, semántica no) |
| `429 Too Many Requests` | Rate limiting |
| `500 Internal Server Error` | Error no controlado. Nunca expone detalles |
| `503 Service Unavailable` | Dependencia externa caída (pasarela, DIAN) |

**La distinción 400 vs 422 es la que más se equivoca**: `400` = "no entiendo tu petición";
`422` = "te entiendo perfectamente, pero el negocio no lo permite".

## 4. Formato de error (RFC 7807)

```json
{
  "type": "https://creditcore.com/problemas/regla-negocio/BR-021",
  "title": "Capacidad de pago insuficiente",
  "status": 422,
  "detail": "La cuota calculada (600000.00) supera el 40% del ingreso disponible (1000000.00)",
  "instance": "/api/v1/solicitudes/7f3a.../evaluacion",
  "codigo": "BR-021",
  "traceId": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01",
  "timestamp": "2026-09-12T14:32:10Z"
}
```

Errores de validación de campos:

```json
{
  "type": "https://creditcore.com/problemas/validacion",
  "title": "Error de validación",
  "status": 400,
  "detail": "La petición contiene 2 campos inválidos",
  "instance": "/api/v1/clientes",
  "errores": [
    { "campo": "email",  "mensaje": "debe ser una dirección de correo válida", "valorRecibido": "no-es-email" },
    { "campo": "celular","mensaje": "no puede estar vacío" }
  ],
  "traceId": "...",
  "timestamp": "2026-09-12T14:32:10Z"
}
```

**Regla de seguridad (RNF-037)**: el `detail` nunca contiene stacktraces, SQL, nombres de tabla ni
versiones de librerías. En error 500 el `detail` es genérico y el `traceId` es lo que permite al
equipo encontrar el log real.

Esto se implementa una sola vez con `@RestControllerAdvice` (Fase 04).

## 5. Convenciones de payload

| Aspecto | Convención |
|---|---|
| Nombres de campo | `camelCase` |
| Fechas sin hora | ISO-8601 `"2026-09-12"` |
| Fechas con hora | ISO-8601 UTC `"2026-09-12T14:32:10Z"` |
| Importes | número JSON con 2 decimales: `177375.00` (no string, no centavos enteros) |
| Moneda | campo aparte: `"moneda": "COP"` |
| Enums | `SCREAMING_SNAKE_CASE`: `"EN_MORA"` |
| Identificadores | UUID en string |
| Nulos | se omiten los campos nulos en la respuesta |
| Booleanos | `esActivo`, `tieneMora` (prefijo `es`/`tiene`) |

### Ejemplo — crear cliente

```http
POST /api/v1/clientes
Content-Type: application/json
Authorization: Bearer <token>

{
  "tipoDocumento": "CC",
  "numeroDocumento": "1098765432",
  "nombres": "Ana María",
  "apellidos": "Pérez Gómez",
  "fechaNacimiento": "1995-04-12",
  "email": "ana@example.com",
  "celular": "3001234567",
  "direccion": "Calle 10 #5-20",
  "ciudad": "Bucaramanga",
  "ingresosMensuales": 4500000.00,
  "egresosMensuales": 1200000.00
}
```

```http
HTTP/1.1 201 Created
Location: /api/v1/clientes/7f3a9b2e-1c4d-4e8f-9a7b-3d5e6f8a9b0c

{
  "id": "7f3a9b2e-1c4d-4e8f-9a7b-3d5e6f8a9b0c",
  "tipoDocumento": "CC",
  "numeroDocumento": "1098765432",
  "nombreCompleto": "Ana María Pérez Gómez",
  "email": "ana@example.com",
  "celular": "3001234567",
  "estado": "ACTIVO",
  "capacidadPagoMensual": 3300000.00,
  "creadoEn": "2026-09-12T14:32:10Z"
}
```

### Ejemplo — paginación

```http
GET /api/v1/clientes?page=0&size=20&sort=apellidos,asc&q=perez
```

```json
{
  "contenido": [ { "...": "..." } ],
  "pagina": 0,
  "tamano": 20,
  "totalElementos": 150,
  "totalPaginas": 8,
  "esPrimera": true,
  "esUltima": false
}
```

> ⚠️ **No se expone `Page` de Spring Data directamente**. Su serialización cambia entre versiones y
> filtra detalles del framework en tu contrato público. Se mapea a un DTO propio.

## 6. Idempotencia en la API

Para operaciones no idempotentes por naturaleza pero críticas (desembolso, pago, emisión de
documento), se acepta la cabecera:

```http
Idempotency-Key: 6a1f9c8e-3b2d-4c5a-8e7f-1a2b3c4d5e6f
```

Comportamiento: si llega una petición con una clave ya procesada, se devuelve **la misma respuesta
original** sin re-ejecutar la operación.

## 7. Contrato de webhooks (entrada)

```http
POST /api/v1/webhooks/pagos/wompi
Content-Type: application/json
X-Signature: <hmac>
X-Timestamp: 1757692330
```

Reglas de implementación:
1. Verificar firma **antes** de deserializar el negocio.
2. Rechazar si el timestamp tiene más de N minutos (anti-replay).
3. Persistir el payload crudo.
4. Responder `200` aunque el procesamiento posterior falle (el reproceso es asunto interno).
5. Nunca devolver `500` a una pasarela salvo que quieras que reintente.

## 8. Cabeceras estándar

| Cabecera | Uso |
|---|---|
| `Authorization: Bearer` | Autenticación |
| `X-Correlation-Id` | Trazabilidad; si no llega, se genera (RNF-060) |
| `Idempotency-Key` | Operaciones críticas |
| `Location` | En toda respuesta 201 |
| `Retry-After` | En 429 y 503 |

## 9. Versionado

- Versión en la ruta: `/api/v1`.
- Cambios **compatibles** (agregar campo opcional, nuevo endpoint): misma versión.
- Cambios **incompatibles** (quitar campo, cambiar tipo o semántica): nueva versión.
- Una versión antigua se mantiene al menos un ciclo de deprecación anunciado.

---

## Preguntas de control

1. ¿Cuándo devuelves 400 y cuándo 422? Da un ejemplo de cada uno en este dominio.
2. ¿Por qué no se expone directamente el `Page` de Spring Data?
3. Un pago se envía dos veces por un doble clic del usuario. ¿Qué mecanismo lo evita?
4. ¿Por qué un webhook debe responder 200 aunque el procesamiento interno falle?
5. ¿Qué información NO puede aparecer nunca en un mensaje de error y por qué?
