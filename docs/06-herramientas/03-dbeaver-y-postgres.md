# 03 — DBeaver y PostgreSQL

> Objetivo: usar la base de datos como herramienta de diagnóstico y diseño, no solo como un lugar
> donde "están los datos".

---

## 1. DBeaver: configuración productiva

| Ajuste | Dónde | Valor |
|---|---|---|
| Auto-commit | Barra superior de la conexión | **OFF** cuando toques datos; ON para consultar |
| Límite de filas | `Preferences → Editors → Data Editor` | 200 (evita traer millones por accidente) |
| Formato SQL | `Preferences → Editors → SQL Editor → Formatting` | Palabras clave en MAYÚSCULA |
| Confirmación de DELETE/UPDATE sin WHERE | `Preferences → Editors → SQL Processing` | Activada |
| Conexión de producción | Editar conexión → *General* → **Connection type: Production** | La pinta de rojo. **Hazlo.** |

> ⚠️ Marcar las conexiones de producción como *Production* en DBeaver las colorea de rojo y activa
> confirmaciones extra. Es la protección más barata contra el `DELETE` sin `WHERE` en la base
> equivocada.

## 2. Atajos de DBeaver

| Atajo (macOS) | Acción |
|---|---|
| `⌘⏎` | Ejecutar la sentencia bajo el cursor |
| `⌥X` | Ejecutar todo el script |
| `⌘⇧E` | Ver el plan de ejecución (`EXPLAIN`) |
| `⌃␣` | Autocompletado |
| `⌘⇧F` | Formatear SQL |
| `⌘D` | Duplicar línea |
| `F4` | Abrir el editor del objeto (tabla, vista) |
| `⌘⇧M` | Ver el diagrama ER de la tabla |
| `⌘/` | Comentar |

## 3. Funciones que usaremos en el proyecto

### 3.1 Diagrama ER
Doble clic en el esquema → pestaña **ER Diagram**. Se genera solo a partir de las claves foráneas.
Úsalo para verificar que tu modelo quedó como lo diseñaste.

### 3.2 Plan de ejecución (`⌘⇧E`)
La herramienta más importante para el rendimiento. Muestra si PostgreSQL usa un índice o recorre la
tabla entera.

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM creditos WHERE dias_mora > 30;
```

**Qué buscar:**

| Señal | Significado |
|---|---|
| `Seq Scan` en una tabla grande | No hay índice utilizable → probablemente un problema |
| `Index Scan` / `Index Only Scan` | Bien |
| `Bitmap Heap Scan` | Índice usado con muchas filas; normalmente aceptable |
| `Nested Loop` con muchas filas | Posible problema de join |
| `rows=1000 ... actual rows=1000000` | **Las estadísticas están desactualizadas** → `ANALYZE` |
| `Filter: ... Rows Removed by Filter: 999000` | Trajo casi todo para descartarlo: falta índice |

**Ejercicio de la Fase 12:** medir una consulta, crear el índice, volver a medir, y registrar el
antes y el después en la bitácora. Ver `Seq Scan` convertirse en `Index Scan` con el tiempo cayendo
de 800 ms a 3 ms es una de las lecciones que no se olvidan.

### 3.3 Generación de datos de prueba
`Clic derecho en la tabla → Generate Mock Data`. Útil para llenar 100.000 filas y medir de verdad.

### 3.4 Comparar esquemas
Permite verificar que la base de local y la de un entorno superior coinciden tras las migraciones.

---

## 4. SQL de PostgreSQL que vas a necesitar

### 4.1 Inspección

```sql
-- tamaño de las tablas
SELECT relname AS tabla,
       pg_size_pretty(pg_total_relation_size(relid)) AS total
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC;

-- índices de una tabla y si se están usando
SELECT indexrelname, idx_scan, idx_tup_read
FROM pg_stat_user_indexes
WHERE relname = 'creditos';

-- índices que NUNCA se han usado (candidatos a eliminar)
SELECT indexrelname, relname FROM pg_stat_user_indexes WHERE idx_scan = 0;

-- consultas activas
SELECT pid, now() - query_start AS duracion, state, query
FROM pg_stat_activity
WHERE state <> 'idle' AND datname = 'creditcore'
ORDER BY duracion DESC;

-- bloqueos no concedidos (alguien está esperando)
SELECT * FROM pg_locks WHERE NOT granted;
```

### 4.2 Consultas del dominio que escribirás

```sql
-- Cartera por calificación
SELECT calificacion,
       COUNT(*)                     AS cantidad,
       SUM(saldo_capital)           AS saldo_total,
       ROUND(AVG(dias_mora), 1)     AS dpd_promedio
FROM creditos
WHERE estado IN ('VIGENTE', 'EN_MORA')
GROUP BY calificacion
ORDER BY calificacion;

-- Cuotas que vencen hoy (el cierre de día las busca así)
SELECT c.numero AS credito, q.numero AS cuota, q.total, q.fecha_vencimiento
FROM cuotas q
JOIN creditos c ON c.id = q.credito_id
WHERE q.fecha_vencimiento = CURRENT_DATE
  AND q.estado <> 'PAGADA';

-- Verificar la invariante BR-040 en TODA la base (auditoría de consistencia)
SELECT c.numero,
       c.monto_desembolsado,
       SUM(q.capital) AS suma_capital_plan,
       c.monto_desembolsado - SUM(q.capital) AS diferencia
FROM creditos c
JOIN cuotas q ON q.credito_id = c.id
GROUP BY c.id, c.numero, c.monto_desembolsado
HAVING c.monto_desembolsado <> SUM(q.capital);      -- debe devolver 0 filas SIEMPRE

-- Detectar posibles pagos duplicados
SELECT psp, transaccion_id, COUNT(*)
FROM pagos_recibidos
GROUP BY psp, transaccion_id
HAVING COUNT(*) > 1;                                 -- la UNIQUE lo impide; esto lo verifica

-- Funciones de ventana: evolución del saldo de un crédito
SELECT fecha_valor, tipo, monto,
       SUM(CASE WHEN concepto = 'CAPITAL' THEN -monto ELSE 0 END)
           OVER (ORDER BY creado_en) AS variacion_acumulada
FROM movimientos
WHERE credito_id = '...'
ORDER BY creado_en;
```

> 💡 La consulta de verificación de BR-040 es un ejemplo de **auditoría de consistencia**: una
> consulta que debe devolver siempre cero filas. En la Fase 17 montaremos varias como control
> operativo periódico. Es una práctica muy valorada en sistemas financieros.

### 4.3 Características de PostgreSQL que aprovechamos

| Característica | Dónde la usamos |
|---|---|
| Índice único **parcial** | Una sola solicitud en estudio por cliente/producto (BR-020) |
| Índice **funcional** | Búsqueda por `lower(apellidos)` |
| `NUMERIC` exacto | Todos los importes |
| `TIMESTAMPTZ` | Todas las marcas de tiempo |
| `JSONB` | Payloads de webhook, auditoría, respuestas DIAN |
| `CHECK` | Invariantes de negocio |
| `SELECT ... FOR UPDATE` | Asignación de consecutivos de facturación |
| `CREATE INDEX CONCURRENTLY` | Migraciones sin bloquear en producción |
| CTE (`WITH`) | Reportes de cartera |
| Funciones de ventana | Evolución de saldos y ranking |

---

## 5. Comandos de `psql` que conviene memorizar

```
\l              listar bases de datos
\c creditcore   conectar a una base
\dt             listar tablas
\d creditos     describir una tabla (columnas, índices, restricciones)
\di             listar índices
\df             listar funciones
\timing on      mostrar el tiempo de cada consulta
\x              salida vertical (ideal para filas anchas)
\e              abrir la última consulta en el editor
\q              salir
```

---

## 6. Higiene de trabajo con la base de datos

1. **Nunca** hagas `UPDATE`/`DELETE` sin `WHERE`. Escribe primero el `SELECT` con el mismo `WHERE`.
2. Trabaja con auto-commit **desactivado** cuando modifiques datos: mira el resultado y luego commit.
3. Los cambios de estructura **siempre** por migración Flyway, jamás a mano en DBeaver.
4. Nunca copies datos reales a tu máquina. Usa datos sintéticos.
5. Antes de un `ALTER` sobre una tabla grande, piensa en el bloqueo.
