# 03 — Convenciones de código

Reglas concretas y verificables. En caso de duda, gana la coherencia con el código ya escrito.

## 1. Formato

| Aspecto | Regla |
|---|---|
| Indentación | 4 espacios, nunca tabuladores |
| Longitud de línea | 120 caracteres |
| Llaves | Estilo K&R: `if (x) {` en la misma línea |
| Llaves siempre | Incluso para un `if` de una línea |
| Imports | Sin comodines (`import java.util.*` prohibido); ordenados; sin imports sin usar |
| Ficheros | UTF-8, salto de línea LF, final de fichero con salto |

Configuración en IntelliJ: **Settings → Editor → Code Style → Java**. Se activa
*Reformat code* (`⌥⌘L`) y *Optimize imports* (`⌃⌥O`) al guardar (Fase 00).

## 2. Nomenclatura

| Elemento | Convención | Ejemplo |
|---|---|---|
| Paquete | minúsculas, sin guiones ni acentos | `com.axchisan.creditcore.creditos` |
| Clase / interfaz / record / enum | `PascalCase` | `SolicitudCredito` |
| Método / variable / parámetro | `camelCase` | `calcularCuotaFija` |
| Constante | `SCREAMING_SNAKE_CASE` | `PORCENTAJE_MAXIMO_ENDEUDAMIENTO` |
| Valor de enum | `SCREAMING_SNAKE_CASE` | `PENDIENTE_SEGUNDA_APROBACION` |
| Genérico | Una letra mayúscula | `T`, `ID` |
| Prueba | `<Clase>Test` / `<Clase>IT` | `CreditoTest`, `CreditoRepositoryAdapterIT` |

**Idioma:** el código del dominio va en **español**, porque el lenguaje ubicuo es español.
Las palabras clave de Java, las anotaciones y los términos técnicos universales quedan en inglés
(`Repository`, `UseCase`, `Service`, `Adapter`, `Mapper`, `Controller`, `Entity`, `Request`,
`Response`, `Command`, `Port`). **Sin acentos ni `ñ` en identificadores.**

```java
// ✅ Mezcla correcta
public interface SolicitudCreditoRepository { }
class RegistrarClienteService implements RegistrarClienteUseCase { }
public record CrearClienteRequest(String tipoDocumento, ...) { }
```

## 3. Estructura de una clase

```java
public class Credito {

    // 1. Constantes
    private static final int MAXIMO_CUOTAS = 120;

    // 2. Campos (final siempre que sea posible)
    private final CreditoId id;
    private final ClienteId clienteId;
    private Dinero saldoCapital;
    private EstadoCredito estado;

    // 3. Constructor(es) — privados si hay factory methods
    private Credito(CreditoId id, ClienteId clienteId, ...) {
        this.id = Objects.requireNonNull(id, "id requerido");
        // validación de invariantes AQUÍ: un objeto nunca existe en estado inválido
    }

    // 4. Factory methods estáticos
    public static Credito desembolsar(SolicitudCredito solicitud, LocalDate fecha, Usuario usuario) { ... }

    // 5. Métodos públicos (comportamiento del negocio)
    public ResultadoAplicacion aplicarPago(Dinero monto, LocalDate fechaValor, ReferenciaPago ref) { ... }

    // 6. Consultas (accessors con nombre de negocio, sin prefijo get)
    public Dinero saldoCapital() { return saldoCapital; }
    public boolean estaEnMora()  { return estado == EstadoCredito.EN_MORA; }

    // 7. Métodos privados, justo debajo de quien los usa
    private void validarPuedeRecibirPagos() { ... }

    // 8. equals / hashCode / toString al final
    @Override public boolean equals(Object o) { ... }   // por identidad: solo el id
    @Override public int hashCode() { ... }
    @Override public String toString() { ... }          // sin datos sensibles
}
```

**Nota sobre accessors**: en el dominio se usan nombres de negocio (`saldoCapital()`), no `getXxx()`.
En las entidades JPA sí se usa `getXxx()` porque Hibernate lo espera.

## 4. Reglas Java específicas

| Regla | Razón |
|---|---|
| `final` en campos, parámetros y variables locales siempre que sea posible | Inmutabilidad por defecto |
| `Optional` solo como **tipo de retorno** | Nunca como campo, parámetro ni en colecciones |
| Nunca devolver `null` de un método público | `Optional`, colección vacía o excepción |
| `record` para VO, DTO, comandos y eventos | Inmutabilidad y `equals` gratis |
| `var` solo cuando el tipo es evidente en la misma línea | `var cliente = new Cliente(...)` sí; `var r = obtener()` no |
| Colecciones devueltas como copia inmutable | `List.copyOf(...)` |
| `switch` con expresión y `->` (Java 21) | Más seguro, exhaustivo con enums y sealed |
| Sin `System.out.println` | Logger, siempre |
| Sin números mágicos | Constante con nombre |
| Texto de usuario fuera del código | Mensajes de error y validación centralizados |

```java
// switch moderno y exhaustivo
String descripcion = switch (estado) {
    case VIGENTE  -> "Al día";
    case EN_MORA  -> "En mora";
    case PAGADO   -> "Cancelado";
    case CASTIGADO, ANULADO -> "Inactivo";
};   // el compilador exige cubrir todos los casos: si agregas uno, falla aquí
```

## 5. Logging

```java
private static final Logger log = LoggerFactory.getLogger(AplicarPagoService.class);

log.info("Pago aplicado creditoId={} monto={} referencia={}", creditoId, monto, referencia);
log.warn("Webhook con firma inválida psp={} ip={}", psp, ip);
log.error("Fallo al consultar la pasarela transaccionId={}", transaccionId, excepcion);
```

| Regla | Detalle |
|---|---|
| Parámetros con `{}` | Nunca concatenar con `+` |
| Excepción como último argumento | Sin `{}` para ella |
| Niveles | `ERROR` = requiere acción; `WARN` = anomalía manejada; `INFO` = hito de negocio; `DEBUG` = detalle técnico; `TRACE` = casi nunca |
| **Nunca** loguear | Contraseñas, tokens, PAN/CVV, documento completo, payloads con datos personales (BR-082) |
| Incluir identificadores | `creditoId`, `clienteId`, `referencia`: lo que permite investigar |
| Sin logs en bucles sobre colecciones grandes | Agrega y loguea el resumen |

## 6. Anotaciones de Spring: dónde sí y dónde no

| Anotación | Dónde |
|---|---|
| `@RestController`, `@RequestMapping` | Solo en `infraestructura/entrada/rest` |
| `@Service` | Solo en `aplicacion/servicio` |
| `@Repository` | Solo en adaptadores de `infraestructura/salida/persistencia` |
| `@Entity`, `@Table`, `@Column` | Solo en `infraestructura/salida/persistencia` |
| `@Transactional` | Solo en `aplicacion/servicio`. **Nunca** en el controlador ni en el dominio |
| `@Component` | Evitar; preferir la anotación específica |
| `@Autowired` en campos | **Prohibido**. Inyección por constructor siempre |
| `@Value` | Evitar; usar `@ConfigurationProperties` tipado |

```java
// ✅ Inyección por constructor: inmutable, explícita, testeable sin Spring
@Service
class AplicarPagoService implements AplicarPagoUseCase {

    private final CreditoRepository creditoRepository;
    private final PublicadorEventos publicador;

    AplicarPagoService(CreditoRepository creditoRepository, PublicadorEventos publicador) {
        this.creditoRepository = creditoRepository;
        this.publicador = publicador;
    }
}
```

> Con un solo constructor, Spring inyecta sin necesidad de `@Autowired`. Y la clase se puede
> instanciar en una prueba unitaria con `new`, sin contexto.

## 7. Transacciones

| Regla | Razón |
|---|---|
| `@Transactional` en el método público del servicio de aplicación | Es el límite del caso de uso |
| `@Transactional(readOnly = true)` en consultas | Hibernate evita el dirty checking; mejora el rendimiento |
| Nunca llamadas a servicios externos (HTTP) dentro de una transacción | Bloquea conexiones de BD mientras espera la red |
| Nunca `@Transactional` en un método privado o auto-invocado | El proxy no se aplica: **no hace nada** |
| Una transacción = un agregado | Regla de DDD (ver ADR-0001) |
| Eventos se publican tras el commit | `@TransactionalEventListener(phase = AFTER_COMMIT)` |

> ⚠️ **El error clásico**: llamar a `this.metodoTransaccional()` desde otro método de la misma clase.
> El proxy de Spring no intercepta la llamada interna y **la transacción no se abre**. Lo veremos con
> el depurador en la Fase 07, porque hay que verlo para creerlo.

## 8. Validación: dónde va cada cosa

| Tipo de validación | Dónde | Herramienta |
|---|---|---|
| Formato y presencia de campos | DTO de entrada | Bean Validation (`@NotBlank`, `@Email`, `@Positive`) |
| Invariantes del objeto | Constructor del dominio | `Objects.requireNonNull`, excepciones propias |
| Reglas de negocio | Dominio o servicio de dominio | Excepciones de negocio con ID de regla |
| Integridad de datos | Base de datos | `CHECK`, `UNIQUE`, `NOT NULL` |

**Las cuatro capas son necesarias.** No son redundancia: son defensa en profundidad.

## 9. Herramientas de verificación

| Herramienta | Qué verifica | Fase |
|---|---|---|
| Inspecciones de IntelliJ | Estilo, nulabilidad, olores | F00 |
| Checkstyle / Spotless | Formato y convenciones | F15 |
| ArchUnit | Reglas de arquitectura | F03 |
| JaCoCo | Cobertura | F05 |
| OWASP Dependency-Check | Vulnerabilidades | F15 |
| gitleaks | Secretos en el repositorio | F15 |

---

## Preguntas de control

1. ¿Por qué se prohíbe `@Autowired` en campos?
2. ¿Qué pasa si pones `@Transactional` en un método privado?
3. ¿Por qué el dominio usa `saldoCapital()` y la entidad JPA `getSaldoCapital()`?
4. ¿Dónde validas que el email tiene formato correcto, y dónde que el cliente es mayor de edad?
5. ¿Por qué `Optional` no debe usarse como campo de una clase?
