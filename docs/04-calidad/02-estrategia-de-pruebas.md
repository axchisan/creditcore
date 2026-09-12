# 02 — Estrategia de pruebas

> Objetivo: que sepas **qué probar, en qué nivel y con qué herramienta**, y que escribir pruebas
> deje de ser una tarea final para convertirse en la forma de diseñar.

---

## 1. La pirámide de pruebas

```
                    ▲
                   ╱ ╲          E2E / Sistema         ~5%
                  ╱   ╲         Lento, frágil, caro
                 ╱─────╲        Solo flujos críticos
                ╱       ╲
               ╱ Integra-╲      Integración            ~20%
              ╱   ción    ╲     Componentes reales juntos
             ╱             ╲    Testcontainers, @SpringBootTest
            ╱───────────────╲
           ╱                 ╲  Rodajas (slices)       ~15%
          ╱     Slice tests    ╲ @WebMvcTest, @DataJpaTest
         ╱─────────────────────╲
        ╱                       ╲ Unitarias             ~60%
       ╱      Pruebas unitarias   ╲ Milisegundos, sin IO
      ╱___________________________ ╲ El dominio entero
```

**La regla:** cuanto más abajo, más pruebas, más rápidas y más específicas. Si tu pirámide está
invertida (muchas E2E, pocas unitarias), tus builds tardan 20 minutos y nadie sabe qué se rompió.

### Por qué esta forma en CreditCore

La arquitectura hexagonal **regala** la base de la pirámide: el dominio es Java puro, así que
`CalculadoraAmortizacion`, `Dinero`, `Credito.aplicarPago()` y el `MotorEvaluacionCredito` se prueban
sin levantar nada. Eso es el 60% del valor del sistema probado en milisegundos.

---

## 2. Los niveles, uno a uno

### 2.1 Pruebas unitarias

**Qué prueban:** una clase aislada, sin IO, sin Spring, sin base de datos.
**Cuánto tardan:** microsegundos a milisegundos.
**Dónde:** todo el paquete `dominio`.

```java
class CalculadoraAmortizacionTest {

    private final CalculadoraAmortizacion calculadora =
            new CalculadoraAmortizacion(new AmortizacionFrancesa());

    @Test
    void calculaCuotaFijaParaCasoDeReferencia() {
        // given
        Dinero capital = Dinero.cop("1000000");
        Tasa tasa = Tasa.efectivaAnual("0.24");
        Plazo plazo = Plazo.deMeses(6);

        // when
        PlanPagos plan = calculadora.generar(capital, tasa, plazo, LocalDate.of(2026, 1, 15));

        // then
        assertThat(plan.cuotas()).hasSize(6);
        assertThat(plan.cuota(1).total()).isEqualTo(Dinero.cop("177375"));
    }

    @Test
    void laSumaDeAbonosACapitalEsExactamenteElCapital_BR040() {
        PlanPagos plan = calculadora.generar(Dinero.cop("1000000"),
                Tasa.efectivaAnual("0.24"), Plazo.deMeses(6), LocalDate.of(2026, 1, 15));

        Dinero sumaCapital = plan.cuotas().stream()
                .map(Cuota::capital)
                .reduce(Dinero.cero(), Dinero::mas);

        assertThat(sumaCapital).isEqualTo(Dinero.cop("1000000"));
    }

    @Test
    void elSaldoFinalEsExactamenteCero_BR041() { ... }

    @ParameterizedTest
    @CsvSource({
        "1000000, 6,  0.24, 177375",
        "5000000, 24, 0.18, 248806"
    })
    void calculaLaCuotaParaDistintosEscenarios(String capital, int meses, String tasa, String cuota) {
        ...
    }
}
```

**Características de una buena prueba unitaria:**

| Característica | Significa |
|---|---|
| **F**ast | Milisegundos. Si tarda, no es unitaria |
| **I**ndependent | No depende del orden ni de otras pruebas |
| **R**epeatable | Mismo resultado siempre, en cualquier máquina |
| **S**elf-validating | Pasa o falla; no requiere que un humano lea la salida |
| **T**imely | Escrita junto al código, no seis meses después |

### 2.2 Pruebas de rodaja (slice tests)

Cargan **solo una porción** del contexto de Spring.

#### `@WebMvcTest` — la capa web sin base de datos

```java
@WebMvcTest(ClienteController.class)
class ClienteControllerTest {

    @Autowired MockMvc mockMvc;
    @MockitoBean RegistrarClienteUseCase registrarCliente;   // el caso de uso se simula

    @Test
    void devuelve201YLocationAlRegistrarClienteValido() throws Exception {
        given(registrarCliente.registrar(any())).willReturn(new ClienteId(UUID.randomUUID()));

        mockMvc.perform(post("/api/v1/clientes")
                        .contentType(APPLICATION_JSON)
                        .content("""
                            {
                              "tipoDocumento": "CC",
                              "numeroDocumento": "1098765432",
                              "nombres": "Ana",
                              "apellidos": "Pérez",
                              "fechaNacimiento": "1995-04-12",
                              "email": "ana@example.com",
                              "celular": "3001234567"
                            }
                            """))
                .andExpect(status().isCreated())
                .andExpect(header().exists("Location"));
    }

    @Test
    void devuelve400ConDetalleDeCamposCuandoElEmailEsInvalido() throws Exception { ... }
}
```

Prueba: mapeo JSON, validación, códigos HTTP, manejo de errores. **No** prueba la base de datos.

#### `@DataJpaTest` — la persistencia sin la web

```java
@DataJpaTest
@Testcontainers
@AutoConfigureTestDatabase(replace = NONE)      // usamos PostgreSQL real, no H2
class ClienteRepositoryAdapterIT {

    @Container @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:17-alpine");

    @Autowired ClienteJpaRepository repository;

    @Test
    void impideRegistrarDosClientesConElMismoDocumento_BR001() {
        repository.saveAndFlush(unClienteCon("CC", "1098765432"));

        assertThatThrownBy(() -> repository.saveAndFlush(unClienteCon("CC", "1098765432")))
                .isInstanceOf(DataIntegrityViolationException.class);
    }
}
```

Prueba: mapeo JPA, consultas, restricciones de base de datos, migraciones Flyway.

### 2.3 Pruebas de integración

Levantan el contexto completo y ejercitan el flujo de punta a punta dentro del proceso.

```java
@SpringBootTest(webEnvironment = RANDOM_PORT)
@Testcontainers
@ActiveProfiles("test")
class DesembolsoFlujoCompletoIT {

    @Container @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:17-alpine");

    @Autowired TestRestTemplate rest;

    @Test
    void desembolsaYGeneraPlanDePagosCompleto() {
        var clienteId  = registrarCliente();
        var solicitud  = crearSolicitud(clienteId, "5000000", 24);
        aprobar(solicitud);
        aceptar(solicitud);

        var respuesta = rest.postForEntity("/api/v1/solicitudes/" + solicitud + "/desembolso",
                                            null, CreditoResponse.class);

        assertThat(respuesta.getStatusCode()).isEqualTo(CREATED);
        assertThat(respuesta.getBody().estado()).isEqualTo("VIGENTE");
        assertThat(respuesta.getBody().cuotas()).hasSize(24);
    }
}
```

### 2.4 Pruebas de arquitectura (ArchUnit)

Verifican reglas estructurales. **Son las que impiden que la arquitectura se degrade con el tiempo.**

```java
@AnalyzeClasses(packages = "com.axchisan.creditcore")
class ReglasArquitecturaTest {

    @ArchTest
    static final ArchRule elDominioNoConoceFrameworks = noClasses()
            .that().resideInAPackage("..dominio..")
            .should().dependOnClassesThat().resideInAnyPackage(
                    "org.springframework..", "jakarta.persistence..", "com.fasterxml..");

    @ArchTest
    static final ArchRule losControladoresNoUsanRepositoriosJpa = noClasses()
            .that().resideInAPackage("..infraestructura.entrada.rest..")
            .should().dependOnClassesThat().haveSimpleNameEndingWith("JpaRepository");

    @ArchTest
    static final ArchRule sinCiclosEntreModulos = slices()
            .matching("com.axchisan.creditcore.(*)..")
            .should().beFreeOfCycles();

    @ArchTest
    static final ArchRule losImportesUsanDinero = noFields()
            .that().areDeclaredInClassesThat().resideInAPackage("..dominio.modelo..")
            .and().haveNameMatching(".*(monto|saldo|valor|cuota).*")
            .should().haveRawType(BigDecimal.class);
}
```

> 💡 Esta última regla es un ejemplo de cómo una convención del proyecto (usar `Dinero`, no
> `BigDecimal` suelto) se vuelve **verificable automáticamente**. Las convenciones que no se
> verifican, se incumplen.

### 2.5 Pruebas de concurrencia

Imprescindibles en este dominio (BR-051, BR-058, BR-071):

```java
@Test
void dosPagosSimultaneosNoCorrompenElSaldo_BR058() throws Exception {
    CreditoId creditoId = crearCreditoCon(Dinero.cop("1000000"));

    try (var executor = Executors.newFixedThreadPool(2)) {
        var tarea = (Callable<Void>) () -> {
            aplicarPagoUseCase.aplicar(new AplicarPagoCommand(creditoId, Dinero.cop("100000"), hoy()));
            return null;
        };
        var futuros = executor.invokeAll(List.of(tarea, tarea));
        // se espera que uno falle con conflicto optimista, o que ambos se apliquen correctamente
    }

    Credito credito = repositorio.buscarPorId(creditoId).orElseThrow();
    assertThat(credito.saldoCapital()).isEqualTo(Dinero.cop("800000"));  // nunca 900000
}
```

---

## 3. TDD — cuándo y cómo

### El ciclo

```
   ┌──────────────────┐
   │  ROJO            │  Escribe una prueba que falla.
   │  (falla)         │  Si pasa a la primera, la prueba no prueba nada.
   └────────┬─────────┘
            ▼
   ┌──────────────────┐
   │  VERDE           │  El código MÍNIMO que la hace pasar.
   │  (pasa)          │  Está permitido que sea feo.
   └────────┬─────────┘
            ▼
   ┌──────────────────┐
   │  REFACTOR        │  Ahora sí, limpia. Las pruebas te protegen.
   │  (sigue pasando) │
   └────────┬─────────┘
            └──► repetir
```

### Dónde usaremos TDD en este proyecto (obligatorio)

| Módulo | Por qué es ideal para TDD |
|---|---|
| `Dinero` y VO del kernel | Contrato pequeño, reglas claras, casos borde evidentes |
| `CalculadoraAmortizacion` | Entrada → salida verificable con números exactos |
| `ImputadorPagos` | Reglas de imputación con muchos escenarios |
| `CalculadoraMora` | Fechas y redondeos, terreno de bugs |
| `CufeCalculator` | Determinismo verificable |

### Dónde NO forzaremos TDD

Controladores, configuración, mappers triviales, adaptadores de infraestructura. Ahí se escribe la
prueba junto al código, no necesariamente antes.

---

## 4. Dobles de prueba

| Doble | Qué hace | Cuándo usarlo |
|---|---|---|
| **Dummy** | Se pasa pero no se usa | Rellenar un parámetro |
| **Stub** | Devuelve respuestas preparadas | "Que el repositorio devuelva este crédito" |
| **Spy** | Stub que además registra llamadas | Verificar que se notificó |
| **Mock** | Verifica interacciones esperadas | "Se llamó a la pasarela exactamente una vez" |
| **Fake** | Implementación funcional simplificada | Repositorio en memoria, pasarela simulada |

```java
// Stub con Mockito
given(creditoRepository.buscarPorId(id)).willReturn(Optional.of(credito));

// Verificación de interacción
then(pasarelaPort).should(times(1)).crearIntencion(any());
then(pasarelaPort).shouldHaveNoMoreInteractions();

// Fake: muchas veces mejor que un mock
class CreditoRepositoryEnMemoria implements CreditoRepository {
    private final Map<CreditoId, Credito> datos = new ConcurrentHashMap<>();
    public Optional<Credito> buscarPorId(CreditoId id) { return Optional.ofNullable(datos.get(id)); }
    public void guardar(Credito credito) { datos.put(credito.id(), credito); }
}
```

> 💡 **Regla práctica**: prefiere un **fake** a un mock cuando el colaborador tiene estado. Un test
> lleno de `given(...)` encadenados es una señal de que estás probando la implementación, no el
> comportamiento.

### La regla de oro de los mocks

**No mockees lo que no te pertenece.** No mockees `EntityManager`, `RestTemplate` ni clases de
librerías: mockea **tus** puertos. Si necesitas mockear una clase externa, es que te falta una
abstracción.

---

## 5. Nombres y estructura de pruebas

```java
// Nombre: qué hace el sistema, en condiciones qué, con resultado cuál
@Test
void rechazaLaSolicitudCuandoLaCuotaSuperaLaCapacidadDePago_BR021() {

    // GIVEN — el estado inicial
    Cliente cliente = ClienteMother.conIngresos("1000000").build();
    SolicitudCredito solicitud = SolicitudMother.para(cliente).porMonto("20000000").build();

    // WHEN — la acción
    Evaluacion evaluacion = motor.evaluar(solicitud, cliente);

    // THEN — el resultado observable
    assertThat(evaluacion.decision()).isEqualTo(RECHAZADA);
    assertThat(evaluacion.motivo()).contains("BR-021");
}
```

Convenciones del proyecto:
- Nombre en español, describiendo comportamiento, con el ID de la regla al final cuando aplique.
- Estructura `// given / when / then` siempre visible.
- **Una aserción lógica por prueba** (pueden ser varios `assertThat` del mismo hecho).
- Datos de prueba con **Object Mother** o **Test Data Builder**, nunca constructores gigantes
  repetidos en 30 pruebas.

```java
// Object Mother
public final class ClienteMother {
    public static ClienteBuilder cualquiera() { ... }
    public static ClienteBuilder conIngresos(String ingresos) { ... }
    public static ClienteBuilder menorDeEdad() { ... }
}
```

---

## 6. Qué NO probar

- Getters y setters triviales.
- Código de terceros (Spring, Hibernate ya están probados).
- Configuración sin lógica.
- Métodos privados **directamente** (se prueban a través del público; si te cuesta, probablemente
  deberían estar en otra clase).

---

## 7. Cobertura

| Métrica | Objetivo | Herramienta |
|---|---|---|
| Cobertura global | ≥ 60% | JaCoCo |
| Cobertura del paquete `dominio` | ≥ 80% | JaCoCo con regla por paquete |
| Reglas de negocio `BR-xxx` | 100% con al menos una prueba | Matriz de trazabilidad |

> ⚠️ **La cobertura es un indicador, no un objetivo.** Es posible tener 100% de cobertura y cero
> aserciones útiles. Lo que importa: *¿si rompo esta regla a propósito, falla alguna prueba?*
> Esa pregunta se responde con **mutación mental**: cambia un `>` por `>=` y mira si algo se queja.

---

## 8. Organización y ejecución

```
mvn test              → solo pruebas unitarias y de slice  (*Test)
mvn verify            → además las de integración          (*IT)
mvn test -Dtest=DineroTest        → una clase
mvn test -Dtest=DineroTest#sumaDosImportes   → un método
```

Separación por convención de nombres: `*Test` (rápidas, en `surefire`) e `*IT` (lentas, en
`failsafe`). Así el ciclo de desarrollo es rápido y el ciclo de CI es completo.

---

## 9. Pruebas como documentación

Una suite bien escrita responde: *"¿qué hace este sistema?"*. Si alguien lee
`CreditoTest` y entiende las reglas del negocio sin abrir `Credito.java`, las pruebas están bien.

---

## Preguntas de control

1. ¿Por qué no usamos H2 para las pruebas de integración?
2. ¿Cuál es la diferencia entre un mock y un fake? ¿Cuándo prefieres cada uno?
3. Tienes 95% de cobertura y un bug en producción. ¿Qué falló en tu estrategia?
4. ¿Por qué no se prueban los métodos privados directamente?
5. ¿Qué prueba escribirías **primero** para el motor de amortización y por qué?
6. ¿Para qué sirve ArchUnit si el código ya compila?
