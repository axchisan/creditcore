# 02 — Contenedores y LocalStack

## 1. Por qué contenedores en este proyecto

| Uso | Cuándo |
|---|---|
| **Testcontainers** | Pruebas de integración con PostgreSQL real (ADR-0008) |
| **LocalStack** | Simular S3, SQS y Secrets Manager sin costo ni credenciales AWS |
| **compose.yaml** | Levantar todo el entorno de desarrollo con un comando |
| **Imagen de la app** | Despliegue en ECS Fargate (F16) |

Runtime elegido: **OrbStack** (compatible con la CLI de Docker, ligero en Apple Silicon).

## 2. `compose.yaml` del entorno de desarrollo (F15)

Estructura objetivo (lo escribirás tú):

```yaml
services:
  postgres:
    image: postgres:17-alpine
    environment:
      POSTGRES_DB: creditcore
      POSTGRES_USER: creditcore
      POSTGRES_PASSWORD: creditcore
    ports: ["5433:5432"]          # 5433 para no chocar con el Postgres nativo
    volumes: ["pgdata:/var/lib/postgresql/data"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U creditcore"]
      interval: 5s
      retries: 10

  localstack:
    image: localstack/localstack:latest
    environment:
      SERVICES: s3,sqs,secretsmanager
      DEBUG: 0
    ports: ["4566:4566"]
    volumes: ["./localstack/init:/etc/localstack/init/ready.d"]

volumes:
  pgdata:
```

```bash
docker compose up -d
docker compose ps
docker compose logs -f postgres
docker compose down          # conserva los volúmenes
docker compose down -v       # borra también los datos
```

> 💡 Tienes PostgreSQL nativo en el 5432. El contenedor usa el 5433 para que puedas comparar ambos
> y para que nunca haya duda de contra cuál estás trabajando.

## 3. LocalStack: AWS sin AWS

```bash
# Configurar la CLI para apuntar a LocalStack
export AWS_ENDPOINT_URL=http://localhost:4566
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=us-east-1

# S3
aws s3 mb s3://creditcore-documentos
aws s3 ls
aws s3 cp factura.xml s3://creditcore-documentos/

# SQS
aws sqs create-queue --queue-name creditcore-eventos
aws sqs create-queue --queue-name creditcore-eventos-dlq
aws sqs list-queues
aws sqs send-message --queue-url http://localhost:4566/000000000000/creditcore-eventos --message-body '{"test":1}'
aws sqs receive-message --queue-url http://localhost:4566/000000000000/creditcore-eventos

# Secrets Manager
aws secretsmanager create-secret --name creditcore/db --secret-string '{"password":"x"}'
aws secretsmanager get-secret-value --secret-id creditcore/db
```

En la aplicación, el único cambio es el **endpoint**:

```yaml
# application-local.yml
spring:
  cloud:
    aws:
      endpoint: http://localhost:4566
      region.static: us-east-1
      credentials:
        access-key: test
        secret-key: test
```

```yaml
# application-prod.yml — sin endpoint: usa AWS real y credenciales del rol IAM
spring:
  cloud:
    aws:
      region.static: us-east-1
```

**Ese es todo el truco**: el mismo código contra LocalStack o AWS real, cambiando configuración.
Es la demostración práctica de por qué la configuración se externaliza.

## 4. Dockerfile multi-stage (F15)

Estructura objetivo:

```dockerfile
# ---- etapa de compilación ----
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /app
COPY .mvn/ .mvn/
COPY mvnw pom.xml ./
RUN ./mvnw dependency:go-offline -B        # caché de dependencias en su propia capa
COPY src ./src
RUN ./mvnw clean package -DskipTests -B

# ---- etapa final ----
FROM eclipse-temurin:21-jre-alpine
RUN addgroup -S app && adduser -S app -G app
WORKDIR /app
COPY --from=build /app/target/creditcore-*.jar app.jar
USER app                                    # RNF-082: no ejecutar como root
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget -qO- http://localhost:8080/actuator/health/readiness || exit 1
ENTRYPOINT ["java", "-XX:MaxRAMPercentage=75", "-jar", "app.jar"]
```

**Conceptos que se practican aquí:**
- **Capas y caché**: copiar el `pom.xml` antes que el código hace que las dependencias solo se
  vuelvan a descargar si el `pom` cambia.
- **Imagen final mínima**: JRE, no JDK; sin Maven ni código fuente.
- **Usuario no root**: requisito de seguridad.
- **`MaxRAMPercentage`**: la JVM debe respetar el límite de memoria del contenedor.

## 5. Comandos de Docker que vas a usar

```bash
docker ps                              # contenedores corriendo
docker logs -f <id>                    # seguir logs
docker exec -it <id> sh                # entrar al contenedor
docker inspect <id>                    # configuración completa
docker stats                           # uso de recursos
docker image ls                        # imágenes
docker system prune -a                 # limpiar (cuidado)
docker build -t creditcore:local .
docker run --rm -p 8080:8080 -e SPRING_PROFILES_ACTIVE=local creditcore:local
```
