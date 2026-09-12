# 04 — Maven

## 1. El ciclo de vida (lo que realmente pasa)

```
validate → compile → test → package → verify → install → deploy
```

Ejecutar una fase ejecuta **todas las anteriores**. `mvn package` compila y prueba antes de empaquetar.

| Fase | Qué hace | Plugin responsable |
|---|---|---|
| `validate` | Verifica que el proyecto es correcto | — |
| `compile` | Compila a `target/classes` | maven-compiler-plugin |
| `test` | Ejecuta las pruebas unitarias (`*Test`) | maven-surefire-plugin |
| `package` | Genera el JAR en `target/` | maven-jar-plugin + spring-boot-maven-plugin |
| `verify` | Pruebas de integración (`*IT`) y verificaciones de calidad | maven-failsafe-plugin, jacoco |
| `install` | Copia el artefacto a `~/.m2/repository` | maven-install-plugin |
| `deploy` | Publica en un repositorio remoto | maven-deploy-plugin |

Y aparte: `clean` (borra `target/`) y `site` (documentación).

## 2. Comandos del día a día

```bash
./mvnw clean verify                     # el comando completo, el que debe pasar antes de cada commit
./mvnw test                             # ciclo rápido
./mvnw test -Dtest=DineroTest           # una clase
./mvnw test -Dtest=DineroTest#sumaDosImportes
./mvnw spring-boot:run                  # arrancar
./mvnw spring-boot:run -Dspring-boot.run.profiles=local
./mvnw -o clean verify                  # modo offline
./mvnw -T 1C clean verify               # paralelo (1 hilo por núcleo)
./mvnw clean verify -DskipTests         # saltar pruebas (solo para diagnósticos)
```

## 3. Diagnóstico de dependencias

```bash
./mvnw dependency:tree                          # árbol completo
./mvnw dependency:tree -Dincludes=org.slf4j     # de dónde viene una librería concreta
./mvnw dependency:analyze                       # declaradas y no usadas / usadas y no declaradas
./mvnw help:effective-pom                       # el POM real tras heredar del parent
./mvnw versions:display-dependency-updates      # actualizaciones disponibles
./mvnw versions:display-plugin-updates
```

> 💡 Cuando dos librerías traen versiones distintas de una tercera, Maven aplica **"el más cercano
> gana"** (menor profundidad en el árbol). `dependency:tree` te muestra qué se descartó con
> `(omitted for conflict with X)`. Esta es la causa de la mayoría de los `NoSuchMethodError`.

## 4. Scopes

| Scope | Disponible en | Ejemplo |
|---|---|---|
| `compile` (por defecto) | Todo | spring-boot-starter-web |
| `provided` | Compilación y pruebas, no se empaqueta | API de servlet en un WAR |
| `runtime` | Ejecución y pruebas, no compilación | driver de PostgreSQL |
| `test` | Solo pruebas | junit, testcontainers |
| `import` | Solo en `dependencyManagement` | BOMs |

> El driver de PostgreSQL va en `runtime` precisamente porque tu código **no debe** importar nada
> de él: eso lo hace el DriverManager. Si tuvieras que ponerlo en `compile`, sería señal de que
> estás acoplado al driver.

## 5. Perfiles

```xml
<profiles>
    <profile>
        <id>integration-tests</id>
        <build>
            <plugins>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-failsafe-plugin</artifactId>
                </plugin>
            </plugins>
        </build>
    </profile>
</profiles>
```

```bash
./mvnw verify -Pintegration-tests
```

## 6. El repositorio local

```
~/.m2/repository        artefactos descargados
~/.m2/settings.xml      configuración global (espejos, credenciales, proxies)
```

Cuándo borrarlo: casi nunca. Si sospechas de un artefacto corrupto, borra solo esa carpeta.
`./mvnw -U` fuerza la actualización de SNAPSHOTs.

**Nunca** pongas credenciales en el `pom.xml`: van en `settings.xml`, fuera del repositorio.

## 7. Plugins que usaremos y en qué fase

| Plugin | Para qué | Fase |
|---|---|---|
| spring-boot-maven-plugin | JAR ejecutable, `spring-boot:run`, imagen OCI | F00 |
| maven-compiler-plugin | Nivel de lenguaje | F00 |
| maven-surefire-plugin | Pruebas `*Test` | F00 |
| maven-failsafe-plugin | Pruebas `*IT` | F05 |
| jacoco-maven-plugin | Cobertura con umbrales | F05 |
| spotless / checkstyle | Formato y estilo | F15 |
| dependency-check-maven | Vulnerabilidades (OWASP) | F15 |
| flyway-maven-plugin | Migraciones desde la línea de comandos | F03 |

## 8. Errores comunes

| Error | Causa | Solución |
|---|---|---|
| `package X does not exist` | Falta la dependencia o el scope es incorrecto | `dependency:tree` |
| `NoSuchMethodError` en runtime | Conflicto de versiones transitivas | `dependency:tree -Dverbose` |
| Las pruebas no se ejecutan | Nombre que no cumple `*Test` | Renombrar |
| `*IT` no se ejecuta con `mvn test` | Es correcto: van en `verify` | Usar `mvn verify` |
| `release version 21 not supported` | JDK equivocado | `mvn -v` y `JAVA_HOME` |
| Build lento la primera vez | Descarga de dependencias | Normal; luego usa caché |
