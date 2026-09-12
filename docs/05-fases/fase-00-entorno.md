# Fase 00 — Entorno y herramientas

> **Objetivo:** tener un entorno profesional y **dominar las herramientas** antes de escribir la
> primera línea de negocio. El tiempo invertido aquí se recupera multiplicado.
>
> **Estado del entorno (verificado el 2026-09-12):**
> ✅ JDK 17 y 25 (Temurin) · ✅ Maven 3.9.16 · ✅ PostgreSQL 17 corriendo · ✅ IntelliJ IDEA ·
> ✅ DBeaver · ✅ Git 2.50 · ✅ gh CLI · ✅ AWS CLI · ✅ OrbStack (instalado en esta sesión)
> ✅ JDK 21 (Homebrew OpenJDK) disponible

---

## Parte 1 — Herramientas base

### Paso 1.1 — Instalar JDK 21 (Temurin)

```bash
brew install --cask temurin@21
/usr/libexec/java_home -V          # debe listar 21.x
```

**Por qué 21 y no el 25 que ya tienes:** ver [ADR-0002](../03-arquitectura/adr/0002-version-java-y-spring-boot.md).

**Gestión de varios JDK.** Tienes tres opciones; elige una y sé consistente:

```bash
# Opción A — variable de entorno por sesión
export JAVA_HOME=$(/usr/libexec/java_home -v 21)

# Opción B — SDKMAN! (recomendado si vas a alternar mucho)
curl -s "https://get.sdkman.io" | bash
sdk list java
sdk install java 21.0.x-tem
sdk default java 21.0.x-tem

# Opción C — IntelliJ gestiona el SDK del proyecto (File → Project Structure → SDK)
```

> 💡 El `pom.xml` fijará `<maven.compiler.release>21</maven.compiler.release>`, así que el build
> produce bytecode de 21 aunque compiles con otro JDK. Aun así, **usa 21 para evitar sorpresas**.

**Comprobación:**
```bash
java -version     # debe decir 21
mvn -v            # debe reportar Java 21
```

### Paso 1.2 — Verificar OrbStack

```bash
open -a OrbStack        # primera vez: completar el asistente
docker --version
docker run --rm hello-world
docker compose version
```

### Paso 1.3 — Preparar PostgreSQL

Ya tienes PostgreSQL 17 corriendo por Homebrew. Crea la base y el usuario del proyecto:

```bash
psql postgres
```

```sql
CREATE USER creditcore WITH PASSWORD 'creditcore';
CREATE DATABASE creditcore OWNER creditcore;
GRANT ALL PRIVILEGES ON DATABASE creditcore TO creditcore;
\c creditcore
GRANT ALL ON SCHEMA public TO creditcore;
\q
```

```bash
psql -U creditcore -d creditcore -h localhost -c "SELECT version();"
```

> ⚠️ Esta contraseña es solo para desarrollo local. **Nunca** llega al repositorio: se inyecta por
> variable de entorno (Fase 02).

### Paso 1.4 — Conectar DBeaver

`Database → New Connection → PostgreSQL`
- Host `localhost`, Port `5432`, Database `creditcore`, User `creditcore`.
- Probar conexión y guardar.

Configuración útil (`Preferences`):
- **Editors → SQL Editor**: activar *auto-commit* solo en desarrollo; desactivarlo cuando toques datos.
- **Editors → SQL Editor → Formatting**: palabras clave en MAYÚSCULA.
- **User Interface**: tema oscuro si prefieres.

### Paso 1.5 — Git y GitHub

```bash
git config --global user.name "axchisan"
git config --global user.email "darciniegasgerena@gmail.com"
git config --global init.defaultBranch main
git config --global pull.rebase true
git config --global core.editor "nvim"     # o el que prefieras

gh auth status      # si no estás autenticado: gh auth login
```

---

## Parte 2 — IntelliJ IDEA: la herramienta que más vas a usar

### Paso 2.1 — Configuración inicial obligatoria

| Ajuste | Ruta | Valor |
|---|---|---|
| SDK del proyecto | `File → Project Structure → Project` | Temurin 21, nivel de lenguaje 21 |
| Codificación | `Settings → Editor → File Encodings` | UTF-8 en todo; saltos LF |
| Longitud de línea | `Settings → Editor → Code Style → Java` | 120 |
| Imports sin comodín | `Code Style → Java → Imports` | *Class count to use import with '*'* = **99** |
| Formatear al guardar | `Settings → Tools → Actions on Save` | ✔ Reformat code, ✔ Optimize imports |
| Mostrar espacios | `Settings → Editor → General → Appearance` | ✔ Show whitespaces |
| Inspecciones fuertes | `Settings → Editor → Inspections` | Activar *Java → Probable bugs*, *Data flow* |
| Build automático | `Settings → Build → Compiler` | ✔ Build project automatically |
| Anotaciones nulas | `Settings → ... → Nullability annotations` | JSpecify o las de JetBrains |

### Paso 2.2 — Los atajos que vas a usar todos los días (macOS)

La lista completa está en [`../06-herramientas/01-intellij-atajos.md`](../06-herramientas/01-intellij-atajos.md).
Estos son los **diez** que debes automatizar esta semana:

| Atajo | Acción | Por qué importa |
|---|---|---|
| `⇧⇧` (doble Shift) | Buscar en todas partes | El punto de entrada a todo |
| `⌘E` | Archivos recientes | Navegación real entre 5 archivos |
| `⌘B` | Ir a la declaración | Leer código ajeno |
| `⌥⌘B` | Ir a la implementación | Clave en arquitectura hexagonal (interfaz → adaptador) |
| `⌘F12` | Estructura del archivo | Ver una clase de un vistazo |
| `⌥⏎` | Acciones de contexto | El atajo más potente del IDE |
| `⌘N` | Generar (constructor, getters, equals…) | Sin Lombok, esto es tu generador |
| `⌃T` | Refactorizar | Extraer método, renombrar, etc. |
| `⇧F10` / `⌃R` | Ejecutar | Ciclo rápido |
| `⌘⇧F` | Buscar en el proyecto | Encontrar cualquier cosa |

> 🎯 **Ejercicio de esta fase**: durante toda la sesión, **prohibido usar el ratón** para navegar
> entre archivos. Solo `⇧⇧`, `⌘E`, `⌘B`. Duele el primer día y te cambia la vida el tercero.

### Paso 2.3 — Live templates propios

`Settings → Editor → Live Templates → + → Template Group "creditcore"`

Crea al menos estos tres (los usarás cientos de veces):

**`test`** — esqueleto de prueba
```java
@Test
void $NOMBRE$() {
    // given
    $GIVEN$
    // when
    $WHEN$
    // then
    $END$
}
```

**`logg`** — logger de clase
```java
private static final Logger log = LoggerFactory.getLogger($CLASS$.class);
```

**`reqnn`** — validación de invariante
```java
this.$CAMPO$ = Objects.requireNonNull($CAMPO$, "$CAMPO$ requerido");
```

Aplícalos con `Tab` tras escribir la abreviatura.

---

## Parte 3 — Crear el proyecto con el asistente de IntelliJ

> 📌 **Decisión revisada** — ver [ADR-0010](../03-arquitectura/adr/0010-generador-de-proyecto-intellij.md).
> Se usa el generador de Spring Boot de IntelliJ Ultimate, porque es lo que se hace en el trabajo real.
> Lo que **no** es negociable es la revisión posterior: debes poder explicar cada línea del POM.

### Paso 3.1 — `New Project → Spring Boot`

⚠️ **No** elijas `New Project → Java`: eso crea un proyecto Maven plano, sin Spring Boot.
La opción correcta es **Spring Boot** en la columna izquierda (generadores).

**Pantalla 1 — metadatos:**

| Campo | Valor | Por qué |
|---|---|---|
| Name | `creditcore` | En minúsculas: es el `artifactId` y el nombre del JAR. La convención Maven es minúsculas con guiones |
| Location | `/Users/mac/Documents/Dev/Apps/LeaningJava` | IntelliJ crea la carpeta `creditcore` dentro |
| Language | Java | |
| Type | **Maven** | ADR-0003 |
| Group | `com.axchisan` | `com.` para personas y empresas; `org.` es para organizaciones sin ánimo de lucro |
| Artifact | `creditcore` | Se rellena solo desde Name |
| Package name | `com.axchisan.creditcore` | La raíz de todos los paquetes |
| JDK | **21** (cualquier distribución: Temurin, Homebrew OpenJDK…) | Si no lo tienes: desplegable → *Download JDK* → Version `21` |
| Java | **21** | ADR-0002 |
| Packaging | Jar | Contenedor, no servidor de aplicaciones |
| Configuration | **YAML** | Toda la documentación del proyecto usa YAML: más legible y jerárquico que `.properties` |

**Pantalla 2 — dependencias:**

| Marcar | No marcar | Por qué |
|---|---|---|
| Spring Boot: **4.1.1** (la estable que ofrece por defecto) | Versiones SNAPSHOT o M (milestone) | ADR-0002 |
| **Spring Web** | Lombok | ADR-0004 |
| | Spring Boot DevTools | Añade recarga automática y comportamiento implícito que estorba al aprender |
| | Spring Data JPA, PostgreSQL Driver, Flyway | Se agregan **a mano en la Fase 03**, que es donde se entienden |

> 💡 Solo **una** dependencia marcada. Cada una de las demás entra en su fase, escrita por ti,
> entendiendo qué trae y por qué.

### Paso 3.2 — Revisar lo que generó (obligatorio)

Abre el `pom.xml` y comprueba, uno por uno:

```
[ ] <parent> apunta a spring-boot-starter-parent 4.1.1
[ ] <groupId>com.axchisan</groupId>
[ ] <artifactId>creditcore</artifactId>            ← en MINÚSCULAS
[ ] <java.version>21</java.version>
[ ] Está el starter web y el de test con <scope>test</scope>
[ ] Está spring-boot-maven-plugin en <build>
[ ] Existen mvnw, mvnw.cmd y .mvn/  (Maven Wrapper)
[ ] Existen src/main/resources y src/test/java
[ ] Existe CreditCoreApplication con @SpringBootApplication
```

#### ⚠️ Hallazgo: Spring Boot 4 renombró los starters

El generador produce esto, que **no** es lo que enseña la mayoría del material de internet:

| Spring Boot 3.x (lo que verás en tutoriales) | Spring Boot 4.x (lo que genera el asistente) |
|---|---|
| `spring-boot-starter-web` | `spring-boot-starter-webmvc` |
| `spring-boot-starter-test` | `spring-boot-starter-webmvc-test` |

Ambos nombres antiguos **siguen publicados** en Maven Central para 4.1.1, así que un ejemplo de 3.x
compila. Pero los nuevos son los correctos y traen más cosas. Comprobado con `dependency:tree`:

```
spring-boot-starter-webmvc       → starter, jackson, tomcat, http-converter, webmvc
spring-boot-starter-webmvc-test  → starter-test, jackson-test, webmvc-test, resttestclient
```

`spring-boot-starter-webmvc-test` **contiene** a `spring-boot-starter-test` y le añade utilidades
específicas de web (`MockMvc` moderno y `RestTestClient`). No lo cambies por el nombre antiguo.

> 📌 Este es el primer ejemplo concreto de lo que advierte el ADR-0002: el material de 3.x no siempre
> calza. La fuente de verdad es la documentación oficial de Spring Boot 4, no los tutoriales.
> Cada diferencia que encuentres, anótala en la bitácora.

#### Limpieza del POM (lo haces tú)

El generador deja estos bloques **vacíos**, que son ruido y deben borrarse:

```xml
<url/>
<licenses><license/></licenses>
<developers><developer/></developers>
<scm><connection/><developerConnection/><tag/><url/></scm>
```

Y estos valores hay que ajustarlos:

| Elemento | Genera | Debe ser |
|---|---|---|
| `<artifactId>` | `CreditCore` | `creditcore` |
| `<name>` | `CreditCore` | `creditcore` |
| `<version>` | `0.0.1-SNAPSHOT` | `0.1.0-SNAPSHOT` |
| `<description>` | `CreditCore` | `Plataforma de originación y gestión de crédito` |

> 💡 **Lo que NO hay que añadir**: `maven.compiler.release`. El parent de Spring Boot 4 ya declara
> `<maven.compiler.release>${java.version}</maven.compiler.release>`. Compruébalo tú mismo con
> `./mvnw help:effective-pom | grep -A2 compiler` — es el primer uso útil de ese comando.

### Paso 3.3 — Dos trampas del entorno macOS

#### Trampa 1 — El sistema de archivos no distingue mayúsculas

macOS es **case-insensitive** por defecto; Git y Linux **no**. Si ya existía una carpeta `CreditCore`
y creas un proyecto llamado `creditcore` en la misma ruta, macOS **reutiliza la carpeta existente
conservando su nombre original**. El resultado: en disco la carpeta se llama `CreditCore`, aunque el
asistente creyó crear `creditcore`.

Esto rompe el build en CI (Linux sí distingue) y ensucia el repositorio. Comprobación y arreglo:

```bash
ls -d */                       # muestra el nombre REAL en disco

# Renombrar en un sistema case-insensitive requiere DOS pasos:
mv CreditCore _tmp && mv _tmp creditcore
```

Un solo `mv CreditCore creditcore` **no hace nada**: para macOS son el mismo nombre.

#### Trampa 2 — El JDK del IDE y el de la terminal no son el mismo

IntelliJ usa el JDK configurado en el proyecto (21). La terminal usa el que apunte `JAVA_HOME`, que
aquí está vacío y cae en el JDK por defecto del sistema (25). Lo ves en el log de arranque:

```
Starting CreditCoreApplicationTests using Java 25.0.4.1
```

El **bytecode** sí sale correcto —`maven.compiler.release=21` lo garantiza, y se verifica con
`javap -v target/classes/.../CreditCoreApplication.class | grep major` → debe decir **65** (Java 21)—
pero la JVM que ejecuta es otra. Conviene alinearlo:

```bash
# en ~/.zshrc
export JAVA_HOME=/opt/homebrew/opt/openjdk@21
export PATH="$JAVA_HOME/bin:$PATH"
```

> Tabla útil: `major version` del bytecode → 61 = Java 17, 65 = Java 21, 69 = Java 25.

### Paso 3.4 — Arrancar

```bash
cd creditcore
./mvnw spring-boot:run
```

Debe aparecer el banner de Spring y `Tomcat started on port 8080`.
`http://localhost:8080` devolverá un 404 en JSON: **eso está bien**, significa que responde.

## Parte 4 — Maven: entender el ciclo de vida

```bash
./mvnw clean                 # borra target/
./mvnw compile               # compila a target/classes
./mvnw test                  # compila y ejecuta pruebas
./mvnw package               # genera el JAR
./mvnw verify                # package + pruebas de integración + verificaciones
./mvnw install               # instala en el repositorio local ~/.m2
```

**Comandos de diagnóstico que vas a necesitar:**

```bash
./mvnw dependency:tree                    # ¿de dónde viene esa librería?
./mvnw dependency:tree -Dincludes=org.slf4j
./mvnw help:effective-pom                 # el pom real, con todo lo heredado
./mvnw -X compile                         # salida de depuración
./mvnw versions:display-dependency-updates
```

> 🎯 **Ejercicio:** ejecuta `./mvnw dependency:tree` y localiza de dónde sale Jackson, Tomcat y
> SLF4J. No los declaraste: ¿quién los trajo?

---

## Parte 5 — Primer commit del código

```bash
cd /Users/mac/Documents/Dev/Apps/LeaningJava
git switch -c fase/00-entorno
git add creditcore/
git commit -m "build: crear proyecto maven base con spring boot web

Proyecto creado a mano (sin Spring Initializr) para entender cada
elemento del pom.xml. Incluye Maven Wrapper y la clase principal.

Refs: F00"
```

---

## Criterios de aceptación de la fase

```
[ ] java -version reporta 21
[ ] ./mvnw -v reporta Java 21
[ ] docker run --rm hello-world funciona
[ ] psql -U creditcore -d creditcore conecta
[ ] DBeaver conecta a la base creditcore
[ ] ./mvnw clean verify pasa sin errores
[ ] ./mvnw spring-boot:run arranca y Tomcat escucha en 8080
[ ] La carpeta en disco se llama creditcore (minúsculas) — verificado con ls -d */
[ ] El POM está limpio: sin bloques vacíos, artifactId y versión corregidos
[ ] java -version en la terminal reporta 21 (JAVA_HOME alineado)
[ ] javap del .class compilado reporta major version 65
[ ] IntelliJ tiene formato al guardar y sin imports con comodín
[ ] Los 3 live templates están creados y probados
[ ] Puedo navegar entre 5 archivos sin tocar el ratón
[ ] El commit está hecho en la rama fase/00-entorno
```

---

## Preguntas de control

1. ¿Qué aporta `<parent>` de Spring Boot y qué pasaría si lo quitaras?
2. ¿Cuál es la diferencia entre `mvn package` y `mvn install`?
3. Declaraste 2 dependencias pero `dependency:tree` muestra decenas. ¿Por qué?
4. ¿Qué es `~/.m2/repository` y cuándo lo borrarías?
5. ¿Para qué sirve el Maven Wrapper si ya tienes Maven instalado?
5b. ¿Qué diferencia hay entre `spring-boot-starter-webmvc` y `spring-boot-starter-web`?
5c. Tu bytecode es Java 21 pero la JVM que lo ejecuta es la 25. ¿Por qué funciona? ¿Qué riesgo tiene?
6. ¿Qué hace `⌥⌘B` y por qué será tan importante en este proyecto?
7. ¿Por qué la contraseña de la base de datos no puede ir en el `application.yml` del repositorio?

---

## Bitácora

### Sesión 1 — <fecha>
- **Hecho:**
- **Concepto que costó:**
- **Error interesante:**
- **Atajo nuevo aprendido:**
- **Duda abierta:**
