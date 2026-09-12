# 01 — Entornos y configuración

## 1. Los entornos

| Entorno | Perfil Spring | Base de datos | AWS | Uso |
|---|---|---|---|---|
| **Local** | `local` | PostgreSQL 17 nativo (Homebrew) | LocalStack | Desarrollo diario |
| **Test** | `test` | PostgreSQL en Testcontainers | LocalStack en contenedor | Pruebas automatizadas |
| **Dev (AWS)** | `dev` | RDS PostgreSQL | AWS real | Integración, F16 |
| **Prod (AWS)** | `prod` | RDS PostgreSQL Multi-AZ | AWS real | Fase final |

**Principio Twelve-Factor:** el artefacto es **el mismo** en todos los entornos; solo cambia la
configuración. Nunca se compila "para producción".

## 2. Jerarquía de configuración

```
application.yml              ← común a todos los entornos, sin secretos
  ├── application-local.yml  ← desarrollo
  ├── application-test.yml   ← pruebas
  ├── application-dev.yml
  └── application-prod.yml   ← solo referencias a variables de entorno
```

Precedencia en Spring Boot (de mayor a menor):
1. Argumentos de la línea de comandos
2. Variables de entorno
3. `application-{perfil}.yml`
4. `application.yml`
5. Valores por defecto en `@ConfigurationProperties`

## 3. Regla de secretos

| Dato | Local | AWS |
|---|---|---|
| Contraseña de base de datos | Variable de entorno | Secrets Manager |
| Llaves de la pasarela | `.env` **no versionado** | Secrets Manager |
| Certificado digital DIAN | Archivo local fuera del repo | Secrets Manager / S3 cifrado |
| Secreto de firma JWT | Variable de entorno | Secrets Manager |

```yaml
# application-local.yml — así SÍ
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/creditcore
    username: ${DB_USER:creditcore}
    password: ${DB_PASSWORD:creditcore}     # valor por defecto solo para desarrollo
```

```yaml
# application-prod.yml — así SÍ
spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USER}
    password: ${DB_PASSWORD}                # SIN valor por defecto: debe fallar si falta
```

> ⚠️ En producción, un valor por defecto para un secreto es un agujero de seguridad. La aplicación
> **debe negarse a arrancar** si falta (RNF-024).

## 4. `.env.example`

Se versiona un `.env.example` con las claves pero **sin valores**, para que cualquiera sepa qué
necesita configurar:

```bash
DB_URL=jdbc:postgresql://localhost:5432/creditcore
DB_USER=
DB_PASSWORD=
JWT_SECRET=
WOMPI_PUBLIC_KEY=
WOMPI_PRIVATE_KEY=
WOMPI_EVENTS_SECRET=
AWS_ENDPOINT_URL=http://localhost:4566
```

## 5. Configuración tipada

Nada de `@Value("${...}")` disperso. Todo va a clases de configuración validadas:

```java
@ConfigurationProperties(prefix = "creditcore.credito")
@Validated
public record ConfiguracionCredito(
        @NotNull @DecimalMin("0.0") @DecimalMax("1.0") BigDecimal porcentajeMaximoEndeudamiento,
        @NotNull @Positive Integer diasVigenciaOferta,
        @NotNull Dinero umbralAutonomiaAprobacion
) { }
```

Ventajas: falla al arrancar si algo falta o es inválido (fail fast), autocompletado en el YAML,
y la configuración es un objeto que se puede inyectar y probar.
