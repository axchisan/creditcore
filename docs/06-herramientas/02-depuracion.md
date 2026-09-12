# 02 — Depuración: el arte perdido

> El objetivo declarado del proyecto incluye **recuperar la capacidad de depurar a mano**.
> Depurar no es poner `println` hasta que algo tenga sentido: es formular hipótesis y verificarlas
> con evidencia.

---

## 1. El protocolo (repetido, porque es lo importante)

```
1. LEE EL ERROR COMPLETO.
2. FORMULA UNA HIPÓTESIS EXPLÍCITA Y ESCRÍBELA.
3. DISEÑA LA OBSERVACIÓN que la confirma o la refuta.
4. OBSERVA. Si se refuta, vuelve al 2. NO toques el código todavía.
5. CORRIGE LA CAUSA, no el síntoma.
6. ESCRIBE UNA PRUEBA que falle sin la corrección.
7. ANOTA EN LA BITÁCORA.
```

**Prohibido:** cambiar cosas al azar, envolver en `try/catch` para que "no falle", o pedir la
respuesta antes de haber formulado una hipótesis.

---

## 2. Leer un stacktrace de Java

```
org.springframework.dao.DataIntegrityViolationException: could not execute statement
	at org.springframework.orm.jpa.vendor.HibernateJpaDialect.convertHibernateAccessException(...)
	at org.springframework.orm.jpa.JpaTransactionManager.doCommit(...)
	...
	at com.axchisan.creditcore.clientes.aplicacion.servicio.RegistrarClienteService.registrar(RegistrarClienteService.java:42)   ← ¡AQUÍ!
	at com.axchisan.creditcore.clientes.infraestructura.entrada.rest.ClienteController.crear(ClienteController.java:31)
	...
Caused by: org.postgresql.util.PSQLException: ERROR: duplicate key value violates unique constraint "uk_clientes_documento"   ← ¡Y AQUÍ!
```

**Cómo se lee, en orden:**

1. **La última sección `Caused by`** — es la causa raíz real. Todo lo de arriba son envoltorios.
2. **La línea más profunda de TU paquete** (`com.axchisan.creditcore...`) — es tu punto de entrada
   al problema. Todo lo demás es código de librerías.
3. **El mensaje de la excepción** — aquí dice literalmente qué restricción se violó.

En este ejemplo: alguien intentó registrar un documento duplicado (BR-001) y la restricción de
base de datos lo impidió. El "bug" puede ser que falta traducir esa excepción a un 409.

> 💡 **Truco en IntelliJ**: pega un stacktrace en `Analyze → Stack Trace or Thread Dump` (`⌘⇧A`)
> y convierte cada línea en un enlace clicable al código.

### Filtrar el ruido

`Settings → Build → Debugger → Stepping`: marcar *Do not step into classes* con
`org.springframework.*`, `jakarta.*`, `java.*`. Así `F7` no te mete en las tripas de Spring.

---

## 3. El depurador de IntelliJ

### Atajos

| Atajo | Acción |
|---|---|
| `⌘F8` | Poner/quitar breakpoint |
| `⌥⌘F8` | Breakpoint con condiciones (menú rápido) |
| `⌃⇧D` | Depurar lo que está bajo el cursor |
| `F8` | **Step Over** — ejecuta la línea, sin entrar |
| `F7` | **Step Into** — entra en el método |
| `⇧F7` | Smart Step Into — elige en qué llamada entrar |
| `⇧F8` | **Step Out** — sale del método actual |
| `⌥F9` | **Run to Cursor** — ejecuta hasta donde está el cursor |
| `⌥F8` | **Evaluate Expression** — ejecuta código arbitrario en el contexto |
| `F9` | Resume — continuar hasta el siguiente breakpoint |
| `⌘F2` | Detener |
| `⌘⇧F8` | Ver todos los breakpoints |

### Tipos de breakpoint

| Tipo | Cómo | Para qué |
|---|---|---|
| **De línea** | `⌘F8` | El normal |
| **Condicional** | Clic derecho en el breakpoint → *Condition* | `credito.getId().equals(esteId)` — parar solo en el caso que te interesa |
| **De excepción** | `⌘⇧F8` → `+` → *Java Exception Breakpoint* | Parar **justo donde se lanza** una excepción, aunque alguien la capture |
| **De campo (watchpoint)** | `⌘F8` sobre un campo | Parar cuando el campo **cambia de valor** — perfecto para "¿quién puso esto en null?" |
| **De método** | `⌘F8` sobre la firma | Parar al entrar/salir de cualquier implementación |
| **Sin suspender + log** | Clic derecho → desmarcar *Suspend*, marcar *Evaluate and log* | Logging temporal **sin tocar el código** |

> 🎯 **El breakpoint condicional y el de excepción son los que separan a quien depura de quien
> adivina.** Un `for` de 5000 iteraciones no se depura con `F9` cinco mil veces: se pone una
> condición.

### Herramientas de la ventana de depuración

| Herramienta | Qué hace |
|---|---|
| **Variables** | Estado local en el punto actual |
| **Watches** | Expresiones que se reevalúan en cada paso |
| **Evaluate Expression** (`⌥F8`) | Ejecutar cualquier código ahí mismo: llamar métodos, consultar |
| **Frames** | La pila de llamadas; puedes navegar hacia atrás y ver el estado de cada nivel |
| **Drop Frame** | **Deshacer** la llamada actual y volver a ejecutarla (no revierte efectos externos) |
| **Set Value** | Cambiar el valor de una variable en caliente y ver qué pasa |
| **Force Return** | Forzar un valor de retorno sin ejecutar el resto |

> 💡 *Evaluate Expression* es el más infrautilizado. Con el programa parado, puedes escribir
> `credito.saldoCapital().valor().scale()` y ver la respuesta al instante. Es un REPL con tu estado
> real cargado.

---

## 4. Los errores que vas a encontrar en este proyecto (y cómo se depuran)

### 4.1 `NullPointerException`

Java 21 trae *helpful NullPointerExceptions*, que dicen exactamente qué era null:

```
Cannot invoke "Dinero.valor()" because the return value of
"Credito.saldoCapital()" is null
```

**Depuración:** watchpoint sobre el campo → ¿quién lo dejó en null?
**Causas típicas aquí:** mapper que olvidó un campo; entidad no inicializada; `Optional.get()` mal usado.

### 4.2 `LazyInitializationException`

```
could not initialize proxy [Credito#...] - no Session
```

**Causa:** accediste a una relación `LAZY` fuera de la transacción.
**Depuración:** mira dónde termina el `@Transactional` respecto a dónde accedes.
**Solución correcta:** `JOIN FETCH` o `@EntityGraph` — **no** poner `EAGER` en todo, y **no**
`open-in-view` (que lo oculta y empeora el rendimiento).

### 4.3 `@Transactional` que no funciona

**Síntoma:** los cambios no se guardan, o el rollback no ocurre.
**Causa #1:** auto-invocación — un método de la clase llama a otro método `@Transactional` de la
**misma** clase; el proxy no intercepta.
**Causa #2:** el método no es `public`.
**Causa #3:** la excepción es *checked* y no configuraste `rollbackFor`.

**Cómo verlo con el depurador:** pon un breakpoint en el método y mira en **Frames** si aparece una
clase con `$$SpringCGLIB$$` en el nombre. Si no aparece, el proxy no se aplicó.

### 4.4 Problema N+1

**Síntoma:** el endpoint tarda mucho; los logs muestran cientos de `SELECT` iguales.
**Detección:**
```yaml
logging.level.org.hibernate.SQL: DEBUG
logging.level.org.hibernate.orm.jdbc.bind: TRACE
```
**Y en pruebas:** contador de consultas que falla si se superan N.

### 4.5 Bean no encontrado

```
Parameter 0 of constructor in ... required a bean of type 'CreditoRepository' that could not be found
```

**Causas:** la implementación no tiene `@Repository`/`@Component`; está fuera del paquete que se
escanea; hay dos candidatos y falta `@Qualifier`.
**Herramienta:** arrancar con `--debug` y leer el *condition evaluation report*.

### 4.6 Diferencia de un centavo

**Síntoma:** una prueba de amortización falla por $1.
**Causa:** orden de redondeo, escala distinta, o `equals` de `BigDecimal` comparando escalas.
**Depuración:** `Evaluate Expression` con `valor.scale()`, `valor.unscaledValue()`,
`a.compareTo(b)` vs `a.equals(b)`.
**Lección:** en dinero, un centavo **sí** importa.

---

## 5. Depurar la base de datos

```yaml
# application-local.yml (solo en desarrollo)
logging:
  level:
    org.hibernate.SQL: DEBUG                  # el SQL generado
    org.hibernate.orm.jdbc.bind: TRACE        # los parámetros reales
    org.springframework.transaction: DEBUG    # inicio/commit/rollback de transacciones
```

Y en PostgreSQL, ver qué está pasando ahora mismo:

```sql
-- consultas activas
SELECT pid, state, wait_event_type, query
FROM pg_stat_activity
WHERE datname = 'creditcore' AND state <> 'idle';

-- bloqueos
SELECT * FROM pg_locks WHERE NOT granted;

-- plan de ejecución real
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
```

---

## 6. Depurar peticiones HTTP

```bash
# Ver toda la conversación HTTP
curl -v http://localhost:8080/api/v1/clientes

# Solo el código de estado
curl -o /dev/null -s -w "%{http_code}\n" http://localhost:8080/api/v1/clientes
```

Y en la aplicación:
```yaml
logging.level.org.springframework.web: DEBUG
logging.level.org.springframework.security: DEBUG   # imprescindible en la Fase 09
```

> En la Fase 09, `DEBUG` en Spring Security es la diferencia entre entender la cadena de filtros y
> pelearse con un 403 durante tres horas.

---

## 7. Depurar pruebas

- `⌃⇧D` sobre el método de prueba.
- Breakpoint condicional dentro de un `@ParameterizedTest` para parar solo en el caso que falla.
- Si una prueba falla solo al ejecutar toda la suite: hay estado compartido. Ejecuta con orden
  aleatorio para confirmarlo.
- Si falla solo en CI: casi siempre es zona horaria, locale o dependencia del orden.

---

## 8. Cuándo `println` SÍ es la herramienta correcta

Pocas veces, pero existen:
- Depurar concurrencia (un breakpoint cambia el *timing* y el bug desaparece).
- Código que corre en un contexto sin depurador conectado.
- Bucles muy largos donde solo quieres el patrón agregado.

Aun así: usa el logger, no `System.out`, y **quítalo antes del commit**. La alternativa superior es
el **breakpoint sin suspensión con *Evaluate and log***: mismo efecto, sin tocar el código.

---

## Ejercicio de la Fase 11

Se introducirá deliberadamente un bug en el sistema y tendrás que encontrarlo **solo con el log**,
sin que nadie te diga dónde está. Tiempo: 45 minutos. Es el ejercicio que mide de verdad si
recuperaste esta habilidad.
