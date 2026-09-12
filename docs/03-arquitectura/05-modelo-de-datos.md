# 05 — Modelo de datos (PostgreSQL)

> El modelo de datos **no es** una copia del modelo de dominio. El dominio optimiza para reglas;
> la base de datos optimiza para integridad, consultas y volumen. Los mappers los reconcilian.

---

## 1. Convenciones

| Aspecto | Convención | Ejemplo |
|---|---|---|
| Nombres | `snake_case`, tablas en **plural** | `solicitudes_credito` |
| Clave primaria | `id UUID PRIMARY KEY` | generado en la aplicación, no por la BD |
| Clave foránea | `<tabla_singular>_id` | `cliente_id` |
| Importes | `NUMERIC(19,2) NOT NULL` | RNF-053 |
| Tasas | `NUMERIC(12,8)` | RNF-053 |
| Fecha sin hora | `DATE` | `fecha_vencimiento` |
| Fecha con hora | `TIMESTAMPTZ` (siempre UTC) | `creado_en` |
| Enums | `VARCHAR` + `CHECK`, **no** el tipo `ENUM` de Postgres | permite agregar valores sin migración dolorosa |
| Booleanos | `BOOLEAN NOT NULL DEFAULT` | |
| Auditoría | `creado_en`, `creado_por`, `actualizado_en`, `actualizado_por` | en toda tabla mutable |
| Optimistic locking | `version BIGINT NOT NULL DEFAULT 0` | en raíces de agregado |
| Índices | `idx_<tabla>_<columnas>` | `idx_creditos_cliente_id` |
| Restricciones | `uk_`, `fk_`, `ck_` | `uk_clientes_documento` |

**Por qué UUID generado en la aplicación**: el agregado existe y es válido antes de tocar la base de
datos; permite pruebas sin BD y evita el `flush` prematuro para obtener el ID. Se usa **UUID v7**
(ordenado por tiempo) para no destrozar la localidad de los índices.

---

## 2. Diagrama entidad-relación (conceptual)

```
┌───────────────┐          ┌──────────────────────┐          ┌──────────────────┐
│   clientes    │◄────────┤ solicitudes_credito   ├────────►│ productos_credito │
│               │ 1     n │                       │ n      1│                   │
└───────┬───────┘          └───────────┬──────────┘          └──────────────────┘
        │                              │ 1                              
        │                              │ 1                              
        │ 1                  ┌─────────▼──────────┐                     
        │                    │  evaluaciones      │                     
        │                    │  + factores (json) │                     
        │                    └────────────────────┘                     
        │ n                                                             
┌───────▼────────┐  1     n  ┌──────────────────┐                       
│   creditos     ├──────────►│     cuotas       │                       
│                │           └──────────────────┘                       
│                │  1     n  ┌──────────────────┐                       
│                ├──────────►│  movimientos     │ (append-only)         
└───────┬────────┘           └──────────────────┘                       
        │ 1                                                             
        │ n                                                             
┌───────▼────────────┐       ┌─────────────────────┐                    
│  pagos_recibidos   │       │ documentos_fiscales ├──► lineas_documento│
│  UNIQUE(psp,tx_id) │       │  + rangos_numeracion│                    
└────────────────────┘       └─────────────────────┘                    

┌────────────────┐  ┌──────────────┐  ┌────────────────┐
│   usuarios     │  │  auditoria   │  │ eventos_outbox │
└────────────────┘  └──────────────┘  └────────────────┘
```

---

## 3. Tablas principales (esquema objetivo)

> Estas definiciones son la **referencia de diseño**. Las migraciones Flyway las escribirás tú,
> fase por fase, entendiendo cada línea.

### 3.1 `clientes` (F04)

```sql
CREATE TABLE clientes (
    id                   UUID         PRIMARY KEY,
    tipo_documento       VARCHAR(10)  NOT NULL,
    numero_documento     VARCHAR(20)  NOT NULL,
    nombres              VARCHAR(100) NOT NULL,
    apellidos            VARCHAR(100) NOT NULL,
    fecha_nacimiento     DATE         NOT NULL,
    email                VARCHAR(150) NOT NULL,
    celular              VARCHAR(20)  NOT NULL,
    direccion            VARCHAR(200),
    ciudad               VARCHAR(100),
    ingresos_mensuales   NUMERIC(19,2),
    egresos_mensuales    NUMERIC(19,2),
    estado               VARCHAR(20)  NOT NULL,
    version              BIGINT       NOT NULL DEFAULT 0,
    creado_en            TIMESTAMPTZ  NOT NULL DEFAULT now(),
    creado_por           VARCHAR(100) NOT NULL,
    actualizado_en       TIMESTAMPTZ,
    actualizado_por      VARCHAR(100),

    CONSTRAINT uk_clientes_documento UNIQUE (tipo_documento, numero_documento),   -- BR-001
    CONSTRAINT ck_clientes_estado    CHECK (estado IN ('ACTIVO','INACTIVO')),
    CONSTRAINT ck_clientes_ingresos  CHECK (ingresos_mensuales IS NULL OR ingresos_mensuales >= 0)
);

CREATE INDEX idx_clientes_apellidos ON clientes (lower(apellidos));
CREATE INDEX idx_clientes_email     ON clientes (lower(email));
```

> 💡 **Por qué `lower(...)` en el índice**: si la consulta busca sin distinguir mayúsculas, un índice
> normal no se usa. Es un índice funcional. Lo comprobaremos con `EXPLAIN ANALYZE` en la Fase 12.

### 3.2 `productos_credito` (F06)

```sql
CREATE TABLE productos_credito (
    id                    UUID         PRIMARY KEY,
    codigo                VARCHAR(30)  NOT NULL UNIQUE,
    nombre                VARCHAR(100) NOT NULL,
    monto_minimo          NUMERIC(19,2) NOT NULL,
    monto_maximo          NUMERIC(19,2) NOT NULL,
    plazo_minimo          INTEGER      NOT NULL,
    plazo_maximo          INTEGER      NOT NULL,
    tasa_ea               NUMERIC(12,8) NOT NULL,
    sistema_amortizacion  VARCHAR(20)  NOT NULL,
    periodicidad          VARCHAR(20)  NOT NULL,
    activo                BOOLEAN      NOT NULL DEFAULT TRUE,
    version               BIGINT       NOT NULL DEFAULT 0,
    creado_en             TIMESTAMPTZ  NOT NULL DEFAULT now(),

    CONSTRAINT ck_productos_montos CHECK (monto_minimo > 0 AND monto_maximo >= monto_minimo),
    CONSTRAINT ck_productos_plazos CHECK (plazo_minimo > 0 AND plazo_maximo >= plazo_minimo),
    CONSTRAINT ck_productos_tasa   CHECK (tasa_ea > 0)
);
```

### 3.3 `solicitudes_credito` (F07)

```sql
CREATE TABLE solicitudes_credito (
    id                UUID          PRIMARY KEY,
    numero            VARCHAR(20)   NOT NULL UNIQUE,
    cliente_id        UUID          NOT NULL REFERENCES clientes(id),
    producto_id       UUID          NOT NULL REFERENCES productos_credito(id),
    monto_solicitado  NUMERIC(19,2) NOT NULL,
    plazo_meses       INTEGER       NOT NULL,
    estado            VARCHAR(40)   NOT NULL,
    puntaje           INTEGER,
    motivo_decision   TEXT,
    decidido_por      VARCHAR(100),
    decidido_en       TIMESTAMPTZ,
    aprobado_por      VARCHAR(100),
    vigencia_oferta   DATE,
    version           BIGINT        NOT NULL DEFAULT 0,
    creado_en         TIMESTAMPTZ   NOT NULL DEFAULT now(),
    creado_por        VARCHAR(100)  NOT NULL,

    CONSTRAINT ck_solicitudes_monto CHECK (monto_solicitado > 0),
    CONSTRAINT ck_solicitudes_plazo CHECK (plazo_meses > 0)
);

CREATE INDEX idx_solicitudes_cliente ON solicitudes_credito (cliente_id);
CREATE INDEX idx_solicitudes_estado  ON solicitudes_credito (estado);

-- BR-020: una sola solicitud EN_ESTUDIO por cliente y producto (índice único parcial)
CREATE UNIQUE INDEX uk_solicitud_en_estudio
    ON solicitudes_credito (cliente_id, producto_id)
    WHERE estado IN ('EN_ESTUDIO','ANALISIS_MANUAL','PENDIENTE_SEGUNDA_APROBACION');
```

> 💡 **Índice único parcial**: una de las mejores características de PostgreSQL. Hace cumplir una
> regla de negocio condicional **a nivel de base de datos**, donde ninguna condición de carrera puede
> saltársela. Esto es exactamente lo que un `if (yaExiste)` en Java **no** garantiza.

### 3.4 `creditos` (F07)

```sql
CREATE TABLE creditos (
    id                      UUID          PRIMARY KEY,
    numero                  VARCHAR(20)   NOT NULL UNIQUE,
    solicitud_id            UUID          NOT NULL UNIQUE REFERENCES solicitudes_credito(id),  -- BR-035
    cliente_id              UUID          NOT NULL REFERENCES clientes(id),
    producto_id             UUID          NOT NULL REFERENCES productos_credito(id),
    monto_desembolsado      NUMERIC(19,2) NOT NULL,
    tasa_ea                 NUMERIC(12,8) NOT NULL,   -- congelada (BR-015)
    plazo_meses             INTEGER       NOT NULL,
    periodicidad            VARCHAR(20)   NOT NULL,
    sistema_amortizacion    VARCHAR(20)   NOT NULL,
    fecha_desembolso        DATE          NOT NULL,
    fecha_primer_vencimiento DATE         NOT NULL,
    saldo_capital           NUMERIC(19,2) NOT NULL,
    estado                  VARCHAR(20)   NOT NULL,
    dias_mora               INTEGER       NOT NULL DEFAULT 0,
    calificacion            CHAR(1)       NOT NULL DEFAULT 'A',
    desembolsado_por        VARCHAR(100)  NOT NULL,
    version                 BIGINT        NOT NULL DEFAULT 0,
    creado_en               TIMESTAMPTZ   NOT NULL DEFAULT now(),

    CONSTRAINT ck_creditos_saldo   CHECK (saldo_capital >= 0),
    CONSTRAINT ck_creditos_mora    CHECK (dias_mora >= 0),
    CONSTRAINT ck_creditos_estado  CHECK (estado IN ('VIGENTE','EN_MORA','PAGADO','CASTIGADO','ANULADO')),
    CONSTRAINT ck_creditos_calif   CHECK (calificacion IN ('A','B','C','D','E'))
);

CREATE INDEX idx_creditos_cliente   ON creditos (cliente_id);
CREATE INDEX idx_creditos_estado    ON creditos (estado);
CREATE INDEX idx_creditos_mora      ON creditos (dias_mora) WHERE dias_mora > 0;
```

**Nota clave**: `solicitud_id UNIQUE` implementa BR-035 (una solicitud produce a lo sumo un crédito)
a nivel de base de datos. Aunque dos peticiones concurrentes pasen la validación en Java, la segunda
falla en el `INSERT`. **La base de datos es la última línea de defensa y nunca falla.**

### 3.5 `cuotas` (F07)

```sql
CREATE TABLE cuotas (
    id                  UUID          PRIMARY KEY,
    credito_id          UUID          NOT NULL REFERENCES creditos(id) ON DELETE CASCADE,
    numero              INTEGER       NOT NULL,
    fecha_vencimiento   DATE          NOT NULL,
    capital             NUMERIC(19,2) NOT NULL,
    interes             NUMERIC(19,2) NOT NULL,
    otros_conceptos     NUMERIC(19,2) NOT NULL DEFAULT 0,
    total               NUMERIC(19,2) NOT NULL,
    saldo_capital_final NUMERIC(19,2) NOT NULL,
    capital_pagado      NUMERIC(19,2) NOT NULL DEFAULT 0,
    interes_pagado      NUMERIC(19,2) NOT NULL DEFAULT 0,
    estado              VARCHAR(20)   NOT NULL,

    CONSTRAINT uk_cuotas_credito_numero UNIQUE (credito_id, numero),
    CONSTRAINT ck_cuotas_valores        CHECK (capital >= 0 AND interes >= 0),
    CONSTRAINT ck_cuotas_estado         CHECK (estado IN ('PENDIENTE','PARCIAL','PAGADA','VENCIDA'))
);

CREATE INDEX idx_cuotas_credito      ON cuotas (credito_id);
CREATE INDEX idx_cuotas_vencimiento  ON cuotas (fecha_vencimiento) WHERE estado <> 'PAGADA';
```

### 3.6 `movimientos` — append-only (F08)

```sql
CREATE TABLE movimientos (
    id                 UUID          PRIMARY KEY,
    credito_id         UUID          NOT NULL REFERENCES creditos(id),
    cuota_numero       INTEGER,
    tipo               VARCHAR(30)   NOT NULL,
    concepto           VARCHAR(30)   NOT NULL,
    monto              NUMERIC(19,2) NOT NULL,
    fecha_valor        DATE          NOT NULL,
    saldo_capital_despues NUMERIC(19,2) NOT NULL,
    pago_recibido_id   UUID          REFERENCES pagos_recibidos(id),
    reversa_de         UUID          REFERENCES movimientos(id),   -- BR-055
    motivo             TEXT,
    creado_en          TIMESTAMPTZ   NOT NULL DEFAULT now(),
    creado_por         VARCHAR(100)  NOT NULL,

    CONSTRAINT ck_movimientos_tipo CHECK (tipo IN ('DESEMBOLSO','PAGO','CAUSACION_INTERES',
                                                   'CAUSACION_MORA','ABONO_CAPITAL','REVERSO','AJUSTE')),
    CONSTRAINT ck_movimientos_concepto CHECK (concepto IN ('CAPITAL','INTERES_CORRIENTE',
                                                           'INTERES_MORA','OTROS'))
);

CREATE INDEX idx_movimientos_credito ON movimientos (credito_id, creado_en DESC);

-- Sin UPDATE ni DELETE: se refuerza con permisos y con un trigger de protección (F08)
```

> 💡 **Nota `saldo_capital_despues`**: guardar el saldo resultante en cada movimiento parece
> redundante (se podría recalcular), pero convierte la auditoría en algo trivial y permite detectar
> corrupción. Es desnormalización deliberada y justificada.

### 3.7 `pagos_recibidos` — la tabla de la idempotencia (F10)

```sql
CREATE TABLE pagos_recibidos (
    id                   UUID          PRIMARY KEY,
    credito_id           UUID          REFERENCES creditos(id),
    psp                  VARCHAR(30)   NOT NULL,
    transaccion_id       VARCHAR(100)  NOT NULL,
    referencia           VARCHAR(100)  NOT NULL,
    monto                NUMERIC(19,2) NOT NULL,
    fecha_valor          DATE          NOT NULL,
    medio_pago           VARCHAR(30)   NOT NULL,
    estado               VARCHAR(20)   NOT NULL,
    payload_crudo        JSONB         NOT NULL,
    error_procesamiento  TEXT,
    recibido_en          TIMESTAMPTZ   NOT NULL DEFAULT now(),
    procesado_en         TIMESTAMPTZ,

    CONSTRAINT uk_pagos_psp_transaccion UNIQUE (psp, transaccion_id),   -- BR-051 ← LA CLAVE
    CONSTRAINT ck_pagos_monto           CHECK (monto > 0)
);

CREATE INDEX idx_pagos_estado ON pagos_recibidos (estado) WHERE estado <> 'APLICADO';
```

**`uk_pagos_psp_transaccion` es el mecanismo de idempotencia de todo el sistema de pagos.**
No es un detalle: es *la* defensa contra aplicar dos veces un pago.

### 3.8 Facturación (F11)

```sql
CREATE TABLE rangos_numeracion (
    id                UUID         PRIMARY KEY,
    prefijo           VARCHAR(10)  NOT NULL,
    numero_desde      BIGINT       NOT NULL,
    numero_hasta      BIGINT       NOT NULL,
    siguiente_numero  BIGINT       NOT NULL,
    vigente_desde     DATE         NOT NULL,
    vigente_hasta     DATE         NOT NULL,
    resolucion        VARCHAR(50)  NOT NULL,
    clave_tecnica     VARCHAR(100),
    activo            BOOLEAN      NOT NULL DEFAULT TRUE,
    version           BIGINT       NOT NULL DEFAULT 0,

    CONSTRAINT ck_rangos_numeros  CHECK (numero_desde <= siguiente_numero AND siguiente_numero <= numero_hasta + 1),
    CONSTRAINT ck_rangos_vigencia CHECK (vigente_desde <= vigente_hasta)
);

CREATE TABLE documentos_fiscales (
    id                  UUID          PRIMARY KEY,
    tipo                VARCHAR(20)   NOT NULL,
    prefijo             VARCHAR(10)   NOT NULL,
    consecutivo         BIGINT        NOT NULL,
    rango_id            UUID          NOT NULL REFERENCES rangos_numeracion(id),
    cliente_id          UUID          NOT NULL REFERENCES clientes(id),
    credito_id          UUID          REFERENCES creditos(id),
    documento_origen_id UUID          REFERENCES documentos_fiscales(id),  -- notas crédito/débito
    fecha_emision       TIMESTAMPTZ   NOT NULL,
    subtotal            NUMERIC(19,2) NOT NULL,
    total_impuestos     NUMERIC(19,2) NOT NULL,
    total               NUMERIC(19,2) NOT NULL,
    cufe                VARCHAR(200),
    estado              VARCHAR(20)   NOT NULL,
    xml_firmado_url     VARCHAR(500),
    pdf_url             VARCHAR(500),
    respuesta_dian      JSONB,
    intentos_envio      INTEGER       NOT NULL DEFAULT 0,
    version             BIGINT        NOT NULL DEFAULT 0,
    creado_en           TIMESTAMPTZ   NOT NULL DEFAULT now(),

    CONSTRAINT uk_documentos_numero UNIQUE (prefijo, consecutivo),   -- BR-071
    CONSTRAINT ck_documentos_total  CHECK (total = subtotal + total_impuestos)
);
```

### 3.9 Transversales (F09, F13)

```sql
CREATE TABLE auditoria (
    id            UUID         PRIMARY KEY,
    entidad       VARCHAR(60)  NOT NULL,
    entidad_id    UUID         NOT NULL,
    accion        VARCHAR(30)  NOT NULL,
    usuario       VARCHAR(100) NOT NULL,
    ip            VARCHAR(45),
    valores_antes JSONB,
    valores_despues JSONB,
    ocurrido_en   TIMESTAMPTZ  NOT NULL DEFAULT now()
);
CREATE INDEX idx_auditoria_entidad ON auditoria (entidad, entidad_id, ocurrido_en DESC);

CREATE TABLE eventos_outbox (
    id             UUID         PRIMARY KEY,
    tipo_evento    VARCHAR(80)  NOT NULL,
    agregado_tipo  VARCHAR(60)  NOT NULL,
    agregado_id    UUID         NOT NULL,
    payload        JSONB        NOT NULL,
    estado         VARCHAR(20)  NOT NULL DEFAULT 'PENDIENTE',
    intentos       INTEGER      NOT NULL DEFAULT 0,
    creado_en      TIMESTAMPTZ  NOT NULL DEFAULT now(),
    publicado_en   TIMESTAMPTZ
);
CREATE INDEX idx_outbox_pendientes ON eventos_outbox (creado_en) WHERE estado = 'PENDIENTE';
```

---

## 4. Decisiones de diseño y su justificación

| Decisión | Razón |
|---|---|
| UUID en vez de `BIGSERIAL` | El agregado es válido antes de persistir; evita exponer volumen de negocio; facilita sistemas distribuidos |
| UUID v7 | Ordenado por tiempo → índices B-tree eficientes (UUID v4 fragmenta) |
| `VARCHAR` + `CHECK` en vez de `ENUM` de Postgres | Agregar un valor a un `ENUM` nativo requiere migración especial; con `CHECK` es un `ALTER` simple |
| `TIMESTAMPTZ` siempre | Evita el infierno de zonas horarias; se presenta en `America/Bogota` |
| Índices únicos parciales | Reglas condicionales garantizadas por la BD, sin condiciones de carrera |
| `JSONB` para payloads y auditoría | Datos semi-estructurados que no se consultan por campo; `JSONB` permite indexar si hace falta |
| `version` para optimistic locking | Detecta escrituras concurrentes sin bloquear |
| Sin `ON DELETE CASCADE` salvo en `cuotas` | Los datos financieros no se borran |
| Desnormalizar `saldo_capital_despues` | Auditoría trivial y detección de corrupción |

---

## 5. Estrategia de migraciones

```
src/main/resources/db/migration/
├── V1__esquema_base.sql           (F03) extensiones, tablas de plataforma
├── V2__clientes.sql               (F04)
├── V3__productos_credito.sql      (F06)
├── V4__solicitudes.sql            (F07)
├── V5__creditos_y_cuotas.sql      (F07)
├── V6__movimientos_y_pagos.sql    (F08)
├── V7__seguridad_y_auditoria.sql  (F09)
├── V8__facturacion.sql            (F11)
├── V9__outbox.sql                 (F13)
└── R__vistas_reportes.sql         (repetible)
```

**Reglas inviolables:**
1. Una migración aplicada **nunca** se modifica. Se corrige con una nueva.
2. `spring.jpa.hibernate.ddl-auto=validate` — siempre. Nunca `update` ni `create`.
3. Toda migración se prueba levantando desde cero **y** desde el estado anterior.
4. Las migraciones que cambian datos van separadas de las que cambian estructura.
5. En producción, una migración no puede bloquear una tabla grande (se usa `CREATE INDEX CONCURRENTLY`).

---

## 6. Consultas que vamos a tener que optimizar (F12)

| Consulta | Reto |
|---|---|
| Cartera por altura de mora | Índice sobre `dias_mora` parcial |
| Plan de pagos de un crédito | Evitar N+1 con `@EntityGraph` |
| Listado de clientes con filtro | Índice funcional sobre `lower()` |
| Cuotas vencidas del día (cierre) | Índice parcial sobre `fecha_vencimiento` |
| Movimientos de un crédito | Índice compuesto con orden |
| Total desembolsado por producto/mes | Agregación; posible vista materializada |

Cada una se analizará con `EXPLAIN ANALYZE` en DBeaver, comparando antes y después del índice.
**Ver el plan cambiar de `Seq Scan` a `Index Scan` es una de las lecciones más útiles del proyecto.**

---

## Preguntas de control

1. ¿Por qué `NUMERIC(19,2)` y no `FLOAT` para dinero?
2. ¿Qué garantiza `uk_pagos_psp_transaccion` que el código Java no puede garantizar por sí solo?
3. ¿Para qué sirve un índice único **parcial** y qué regla implementa aquí?
4. ¿Por qué `solicitud_id` es UNIQUE en `creditos`?
5. ¿Por qué nunca se modifica una migración ya aplicada?
6. ¿Qué problema resuelve `version` y en qué se diferencia de un `SELECT FOR UPDATE`?
