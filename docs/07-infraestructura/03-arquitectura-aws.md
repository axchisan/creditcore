# 03 — Arquitectura objetivo en AWS

> Se implementa en la **Fase 16**. Aquí queda el diseño, para que las decisiones anteriores lo
> tengan en cuenta (por ejemplo: la app es stateless, los logs van a stdout, los secretos no están
> en el código).

---

## 1. Diagrama objetivo

```
                                 Internet
                                     │
                            ┌────────▼────────┐
                            │   Route 53      │  (DNS)
                            └────────┬────────┘
                                     │
                            ┌────────▼────────┐
                            │  ALB (HTTPS)    │  (ACM: certificado TLS)
                            │  + WAF          │
                            └────────┬────────┘
    ┌────────────────────────────────┼─────────────────────────────────┐
    │  VPC 10.0.0.0/16               │                                 │
    │                                │                                 │
    │  ┌── Subred pública AZ-a ──────┴──── Subred pública AZ-b ──────┐  │
    │  │              (ALB, NAT Gateway)                            │  │
    │  └────────────────────────────┬───────────────────────────────┘  │
    │                               │                                  │
    │  ┌── Subred privada AZ-a ─────┴──── Subred privada AZ-b ───────┐  │
    │  │   ┌──────────────────┐        ┌──────────────────┐         │  │
    │  │   │  ECS Fargate     │        │  ECS Fargate     │         │  │
    │  │   │  creditcore      │        │  creditcore      │         │  │
    │  │   └────────┬─────────┘        └────────┬─────────┘         │  │
    │  └────────────┼───────────────────────────┼───────────────────┘  │
    │               │                           │                      │
    │  ┌── Subred de datos AZ-a ────────── Subred de datos AZ-b ────┐  │
    │  │   ┌────────▼─────────────────────────────▼──────────┐      │  │
    │  │   │        RDS PostgreSQL 17 (Multi-AZ)             │      │  │
    │  │   │        primaria  +  réplica en espera           │      │  │
    │  │   └────────────────────────────────────────────────┘      │  │
    │  └──────────────────────────────────────────────────────────┘  │
    └──────────────────────────────────────────────────────────────────┘
                │                │                │              │
        ┌───────▼──────┐ ┌───────▼──────┐ ┌───────▼──────┐ ┌────▼──────┐
        │      S3      │ │     SQS      │ │   Secrets    │ │CloudWatch │
        │  documentos  │ │ eventos+DLQ  │ │   Manager    │ │logs+alarmas│
        └──────────────┘ └──────────────┘ └──────────────┘ └───────────┘
```

## 2. Servicios y su papel

| Servicio | Para qué | Alternativa considerada |
|---|---|---|
| **ECS Fargate** | Ejecutar los contenedores sin gestionar servidores | EKS (excesivo), EC2 (más operación), App Runner (menos control) |
| **ALB** | Balanceo, TLS, health checks, enrutamiento | API Gateway (mejor para serverless) |
| **RDS PostgreSQL** | Base de datos gestionada, backups, Multi-AZ | Aurora (más caro), EC2 con Postgres (más operación) |
| **S3** | XML firmados, PDFs, soportes documentales | EFS (innecesario) |
| **SQS** | Cola de eventos con DLQ | SNS+SQS, EventBridge, Kafka (excesivo) |
| **Secrets Manager** | Credenciales, certificado DIAN, llaves de pasarela | Parameter Store (más barato, menos rotación) |
| **CloudWatch** | Logs, métricas, alarmas | Grafana/Prometheus autogestionado |
| **ECR** | Registro de imágenes | Docker Hub |
| **Route 53 + ACM** | DNS y certificado | — |
| **WAF** | Protección básica en la capa 7 | — |

## 3. Principios de red

1. **Solo el ALB es público.** Las tareas de ECS y RDS viven en subredes privadas.
2. **RDS no es accesible desde Internet.** Nunca. Acceso solo desde el security group de ECS.
3. Salida a Internet desde subredes privadas vía **NAT Gateway** (es el recurso más caro: en
   entorno de aprendizaje, uno solo).
4. **Security groups por capa**: `sg-alb` → `sg-app` → `sg-db`, cada uno permitiendo solo el puerto
   necesario desde el grupo anterior.

## 4. Seguridad

| Control | Implementación |
|---|---|
| Cifrado en reposo | RDS con KMS, S3 con SSE, secretos en Secrets Manager |
| Cifrado en tránsito | TLS en el ALB; SSL obligatorio en la conexión a RDS |
| Credenciales de la app | **Rol IAM de la tarea ECS** — nunca llaves en variables de entorno |
| Mínimo privilegio | Una política por recurso: solo el bucket y la cola que usa |
| Rotación | Rotación automática de la contraseña de RDS en Secrets Manager |
| Auditoría | CloudTrail activo |
| Red | RDS sin IP pública; security groups restrictivos |

> 💡 **El rol IAM de la tarea es el concepto clave**: la aplicación no tiene credenciales de AWS en
> ningún sitio. El SDK las obtiene del metadata del contenedor automáticamente. Por eso el código
> nunca configura llaves: solo el entorno local (LocalStack) usa credenciales falsas.

## 5. Costos (control para un proyecto de aprendizaje)

| Recurso | Costo aproximado | Mitigación |
|---|---|---|
| NAT Gateway | El más caro del conjunto | Uno solo; **destruir cuando no se use** |
| RDS `db.t4g.micro` | Bajo; puede caer en free tier | Single-AZ en aprendizaje |
| ECS Fargate | Por vCPU/memoria y tiempo | Tareas mínimas; escalar a 0 fuera de uso |
| ALB | Costo fijo por hora | Destruir cuando no se use |
| S3 / SQS / Secrets | Marginal a este volumen | — |

**Reglas del proyecto:**
1. `terraform destroy` al terminar cada sesión de la Fase 16.
2. Presupuesto en AWS Budgets con alerta por correo **antes** del primer `apply`.
3. Etiquetar todos los recursos con `Proyecto=creditcore` para poder rastrear el gasto.

## 6. Terraform: estructura

```
infra/terraform/
├── main.tf
├── variables.tf
├── outputs.tf
├── providers.tf
├── backend.tf                 (estado remoto en S3 + bloqueo)
├── modules/
│   ├── red/                   VPC, subredes, NAT, security groups
│   ├── base-datos/            RDS, subnet group, parámetros
│   ├── computo/               ECS cluster, servicio, task definition, ALB
│   ├── almacenamiento/        S3, políticas de ciclo de vida
│   ├── mensajeria/            SQS + DLQ
│   └── observabilidad/        CloudWatch log groups, alarmas
└── entornos/
    ├── dev/terraform.tfvars
    └── prod/terraform.tfvars
```

```bash
terraform init
terraform fmt -recursive
terraform validate
terraform plan  -var-file=entornos/dev/terraform.tfvars
terraform apply -var-file=entornos/dev/terraform.tfvars
terraform destroy -var-file=entornos/dev/terraform.tfvars   # ← al terminar la sesión
```

**Reglas:** el estado nunca se versiona; `*.tfvars` con datos sensibles tampoco; todo cambio pasa
por `plan` revisado antes del `apply`.

## 7. Qué debe cumplir la aplicación para desplegarse aquí

Estas exigencias explican decisiones que tomamos mucho antes:

| Requisito de AWS | Decisión previa que lo permite |
|---|---|
| Varias réplicas detrás del ALB | La app es **stateless** (RNF-010) |
| Logs centralizados | Logs a **stdout** en JSON (RNF-062) |
| Health checks del ALB y de ECS | Actuator con readiness/liveness (RNF-063) |
| Sin credenciales en el artefacto | Configuración externalizada (RNF-081) |
| Arranque rápido y apagado limpio | Graceful shutdown configurado |
| Migraciones al desplegar | Flyway al arrancar, idempotente |
| Batch que no se duplique con N réplicas | Bloqueo distribuido (RNF-011) |

> Esta tabla es el mejor ejemplo de por qué la arquitectura se piensa desde el principio: cada
> requisito operativo del final tiene su origen en una decisión de las primeras fases.
