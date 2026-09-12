# Fase 00 — Entorno y herramientas

> **Objetivo:** tener un entorno profesional y **dominar las herramientas** antes de escribir la
> primera línea de negocio. El tiempo invertido aquí se recupera multiplicado.
>
> **Estado del entorno (verificado el 2026-09-12):**
> ✅ JDK 17 y 25 (Temurin) · ✅ Maven 3.9.16 · ✅ PostgreSQL 17 corriendo · ✅ IntelliJ IDEA ·
> ✅ DBeaver · ✅ Git 2.50 · ✅ gh CLI · ✅ AWS CLI · ✅ OrbStack (instalado en esta sesión)
> ⬜ **Falta: JDK 21** (ver paso 1)

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

## Parte 3 — Crear el proyecto Maven **a mano**

> ❗ **No uses Spring Initializr.** Todo el sentido de esta fase es que escribas el `pom.xml` y
> entiendas cada línea. Un `pom.xml` que no entiendes es deuda técnica desde el día uno.

### Paso 3.1 — Estructura mínima

```bash
cd /Users/mac/Documents/Dev/Apps/LeaningJava
mkdir -p creditcore/src/main/java/com/axchisan/creditcore
mkdir -p creditcore/src/main/resources
mkdir -p creditcore/src/test/java/com/axchisan/creditcore
mkdir -p creditcore/src/test/resources
```

### Paso 3.2 — El `pom.xml` (lo escribes tú)

Este es el contenido de referencia. **Escríbelo, no lo copies**, y pregunta por cualquier línea que
no entiendas del todo:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                             https://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <!-- Hereda gestión de dependencias, plugins y propiedades de Spring Boot.
         Es lo que permite declarar dependencias SIN versión más abajo. -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.5.16</version>
        <relativePath/>
    </parent>

    <groupId>com.axchisan</groupId>
    <artifactId>creditcore</artifactId>
    <version>0.1.0-SNAPSHOT</version>
    <name>CreditCore</name>
    <description>Plataforma de originación y gestión de crédito</description>

    <properties>
        <java.version>21</java.version>
        <maven.compiler.release>21</maven.compiler.release>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <!-- Web: Spring MVC + Tomcat embebido + Jackson -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- Pruebas: JUnit 5, AssertJ, Mockito, MockMvc, JsonPath -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <!-- Empaqueta un JAR ejecutable y permite mvn spring-boot:run -->
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

**Preguntas que debes poder responder antes de seguir:**
1. ¿Qué hace exactamente `<parent>` y por qué las dependencias no llevan `<version>`?
2. ¿Qué diferencia hay entre `spring-boot-starter-web` y `spring-web`?
3. ¿Qué significa el `<scope>test</scope>`?
4. ¿Qué es un `SNAPSHOT`?

### Paso 3.3 — El Maven Wrapper

```bash
cd creditcore
mvn wrapper:wrapper
./mvnw -v
```

Esto fija la versión de Maven para cualquiera que clone el repositorio. `mvnw` **sí** se versiona.

### Paso 3.4 — La clase principal (la escribes tú)

`src/main/java/com/axchisan/creditcore/CreditCoreApplication.java`

```java
package com.axchisan.creditcore;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class CreditCoreApplication {

    public static void main(String[] args) {
        SpringApplication.run(CreditCoreApplication.class, args);
    }
}
```

### Paso 3.5 — Arrancar

```bash
./mvnw spring-boot:run
```

Debe aparecer el banner de Spring y `Tomcat started on port 8080`.
`http://localhost:8080` devolverá un error 404 en formato JSON: **eso está bien**, significa que la
aplicación responde.

---

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
