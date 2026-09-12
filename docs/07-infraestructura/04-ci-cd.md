# 04 — CI/CD con GitHub Actions

> Se implementa en la **Fase 15**.

## 1. Qué debe hacer el pipeline

```
Push / Pull Request
       │
       ▼
┌──────────────────┐
│  1. Build        │  compilar con JDK 21
├──────────────────┤
│  2. Pruebas      │  unitarias + slices  (mvn test)
├──────────────────┤
│  3. Integración  │  mvn verify con Testcontainers
├──────────────────┤
│  4. Cobertura    │  JaCoCo; falla si baja del umbral
├──────────────────┤
│  5. Calidad      │  Checkstyle/Spotless, ArchUnit
├──────────────────┤
│  6. Seguridad    │  OWASP Dependency-Check, gitleaks
├──────────────────┤
│  7. Imagen       │  build + push a ECR (solo en main)
├──────────────────┤
│  8. Despliegue   │  ECS (solo en main, con aprobación)
└──────────────────┘
```

## 2. Estructura de workflows

```
.github/workflows/
├── ci.yml            en cada push y PR: build, pruebas, calidad, seguridad
├── cd-dev.yml        en main: imagen + despliegue a dev
└── cd-prod.yml       manual, con aprobación
```

## 3. Puntos clave del `ci.yml`

| Aspecto | Detalle |
|---|---|
| Runner | `ubuntu-latest` (trae Docker, necesario para Testcontainers) |
| JDK | `actions/setup-java` con Temurin 21 |
| Caché | `cache: maven` en `setup-java`, para no redescargar dependencias |
| Comando | `./mvnw -B clean verify` |
| Informes | Publicar el resultado de las pruebas y la cobertura como artefactos |
| Fallo rápido | Si las unitarias fallan, no se ejecutan las de integración |
| Concurrencia | Cancelar ejecuciones anteriores de la misma rama |

## 4. Secretos

Se configuran en `Settings → Secrets and variables → Actions`:

| Secreto | Para qué |
|---|---|
| `AWS_ROLE_ARN` | Autenticación por **OIDC** (sin llaves estáticas) |
| `ECR_REPOSITORY` | Destino de la imagen |

> 💡 **OIDC en vez de llaves de acceso**: GitHub obtiene credenciales temporales de AWS asumiendo un
> rol. Es la práctica recomendada actual; evita tener llaves permanentes guardadas en GitHub.

## 5. Protección de la rama `main`

- Requiere que el pipeline pase antes de fusionar.
- Requiere que la rama esté actualizada.
- Prohíbe force-push.

Aunque trabajes solo: **practicar el flujo profesional es parte del objetivo**.

## 6. Estrategia de despliegue

| Entorno | Disparador | Aprobación |
|---|---|---|
| dev | push a `main` | Automático |
| prod | manual (`workflow_dispatch`) o etiqueta | Manual |

Despliegue en ECS con **rolling update** y health checks: si las tareas nuevas no pasan el health
check, ECS revierte automáticamente.

## 7. Qué debe hacer fallar el build

```
[ ] Cualquier prueba que falle
[ ] Cobertura por debajo del umbral (RNF-040)
[ ] Violación de reglas de ArchUnit (RNF-041, RNF-042)
[ ] Violación de estilo (RNF-043)
[ ] Vulnerabilidad crítica en dependencias (RNF-036)
[ ] Secreto detectado en el repositorio (RNF-033)
```

Un build que no puede fallar no sirve para nada. **La disciplina que no se automatiza, se pierde.**
