# Instructivo para el Equipo de Infraestructura — Opción B: Spring Cloud Gateway

## Descripción General
Esta opción despliega una **aplicación Java (Spring Cloud Gateway)** como contenedor Docker en ECS Fargate, con un **sidecar de Datadog Agent** para observabilidad. Actúa como reverse proxy inteligente entre el ALB y Omega/microservicios.

---

## Arquitectura de Despliegue
```
Internet → ALB (HTTPS) → ECS Fargate Task [Spring Cloud Gateway + Datadog Agent Sidecar] → Omega / Microservicios
```

---

## Tareas a Realizar

### 1. Networking (VPC)
- [ ] VPC con subnets públicas (ALB) y privadas (ECS) en al menos 2 AZs
- [ ] Internet Gateway asociado a la VPC
- [ ] NAT Gateway en subnet pública (para que los tasks de Fargate en subnets privadas tengan acceso a internet — pull de imágenes Docker, envío de métricas a Datadog)
- [ ] Route Tables:
  - Pública: tráfico `0.0.0.0/0` → Internet Gateway
  - Privada: tráfico `0.0.0.0/0` → NAT Gateway

### 2. ECR (Elastic Container Registry)
- [ ] Crear repositorio ECR: `bw_omega-gateway-spring`
- [ ] Configurar image scanning on push: **habilitado**
- [ ] Política de lifecycle: Retener últimas 10 imágenes tagged, eliminar untagged > 7 días
- [ ] El equipo de desarrollo pushea las imágenes vía CI/CD (GitHub Actions)

### 3. ECS Cluster
- [ ] Crear ECS Cluster: `bw_omega-gateway-{environment}`
- [ ] Habilitar **Container Insights**: `containerInsights = enabled`
- [ ] Capacity provider: FARGATE (default) + FARGATE_SPOT (opcional para dev/staging)

### 4. Task Definition
El equipo de desarrollo provee el Terraform (`terraform/main.tf`), pero la Task Definition contiene:

- [ ] **Contenedor principal: `gateway`**
  - Imagen: `{account}.dkr.ecr.{region}.amazonaws.com/bw_omega-gateway-spring:latest`
  - CPU: 1024 (1 vCPU)
  - Memoria: 2048 MB
  - Puerto: 8080
  - Health check: `curl -f http://localhost:8080/actuator/health`
  - Variables de entorno requeridas:
    | Variable | Valor | Descripción |
    |---|---|---|
    | `OMEGA_URL` | `http://omega-internal.{dominio}:8080` | URL interna de Omega |
    | `DD_ENV` | `production` | Ambiente para Datadog |
    | `DD_SERVICE` | `bw_omega-gateway` | Nombre del servicio en Datadog |
    | `DD_LOGS_INJECTION` | `true` | Habilitar inyección de trace IDs en logs |
    | `DD_AGENT_HOST` | `127.0.0.1` | El sidecar está en localhost |
    | `DD_TRACE_HEADER_TAGS` | `X-Amzn-Trace-Id:aws.trace_id` | Correlacionar con AWS |
    | `SPRING_PROFILES_ACTIVE` | `production` | Perfil de Spring activo |

- [ ] **Contenedor sidecar: `datadog-agent`**
  - Imagen: `public.ecr.aws/datadog/agent:latest`
  - Essential: true
  - Variables de entorno:
    | Variable | Valor |
    |---|---|
    | `DD_API_KEY` | Obtener de Secrets Manager o SSM Parameter Store (**NO hardcodear**) |
    | `DD_SITE` | `datadoghq.com` |
    | `DD_APM_ENABLED` | `true` |
    | `DD_APM_NON_LOCAL_TRAFFIC` | `true` |
    | `DD_LOGS_ENABLED` | `true` |
    | `DD_LOGS_CONFIG_CONTAINER_COLLECT_ALL` | `true` |
    | `ECS_FARGATE` | `true` |

### 5. ECS Service
- [ ] Nombre: `bw_omega-gateway-{environment}-service`
- [ ] Launch type: FARGATE
- [ ] Desired count: **2** (mínimo para HA)
- [ ] Deployment:
  - Minimum healthy percent: 100%
  - Maximum percent: 200%
  - **Circuit breaker habilitado** con rollback automático
- [ ] Health check grace period: 60 segundos (Spring Boot tarda ~15-30s en arrancar)
- [ ] Network configuration:
  - Subnets: **privadas** únicamente
  - Assign public IP: **false**
  - Security Group: [ver sección 7]

### 6. ALB (Application Load Balancer)
- [ ] Crear ALB en subnets públicas
- [ ] Certificado SSL de ACM asociado al Listener HTTPS (443)
- [ ] Listener HTTP (80) → Redirect a HTTPS
- [ ] Target Group:
  - Tipo: IP
  - Puerto: 8080
  - Protocolo: HTTP
  - Health check path: `/actuator/health`
  - Healthy threshold: 2
  - Unhealthy threshold: 3
  - Interval: 15s
  - Timeout: 5s
  - Deregistration delay: 30s
- [ ] Deletion protection: **habilitado** en producción

### 7. Security Groups
- [ ] **SG del ALB:**
  - Ingress: 443/tcp desde `0.0.0.0/0` (o IPs específicas de proveedores)
  - Ingress: 80/tcp desde `0.0.0.0/0` (redirect)
  - Egress: all traffic
- [ ] **SG de ECS Tasks:**
  - Ingress: 8080/tcp **solo desde el SG del ALB**
  - Egress: all traffic (necesario para NAT → internet → Datadog, ECR)

### 8. IAM Roles
- [ ] **Execution Role** (para ECS):
  - Policy: `AmazonECSTaskExecutionRolePolicy`
  - Acceso de lectura a ECR, CloudWatch Logs, Secrets Manager (para DD_API_KEY)
- [ ] **Task Role** (para la aplicación):
  - Permisos mínimos necesarios (principio de least privilege)
  - Si la app necesita acceder a otros servicios AWS, agregar aquí

### 9. CloudWatch
- [ ] Log Group: `/ecs/bw_omega-gateway-{environment}`
- [ ] Retención: 90 días en producción, 30 días en staging/dev
- [ ] Alarmas:
  | Alarma | Métrica | Umbral | Acción |
  |---|---|---|---|
  | High CPU | ECS CPUUtilization | > 80% por 5 min | Notificación SNS |
  | High Memory | ECS MemoryUtilization | > 85% por 5 min | Notificación SNS |
  | 5xx Errors | ALB HTTPCode_ELB_5XX_Count | > 10 en 5 min | Notificación SNS |
  | Target Response Time | ALB TargetResponseTime p99 | > 500ms | Notificación SNS |
  | Unhealthy Hosts | ALB UnHealthyHostCount | > 0 | Notificación SNS |

### 10. Autoscaling
- [ ] Min capacity: **2** tasks
- [ ] Max capacity: **20** tasks (para picos de eventos deportivos)
- [ ] Políticas:
  - **CPU**: Target tracking al 60% → scale out cooldown 60s, scale in 300s
  - **Request count**: Target tracking de 1000 req/target → scale out rápido

### 11. DNS
- [ ] Registro CNAME/Alias en Route 53: `gateway.{dominio}` → ALB DNS name

### 12. Secrets Management
- [ ] Almacenar `DD_API_KEY` en **AWS Secrets Manager** o **SSM Parameter Store** (SecureString)
- [ ] Dar acceso de lectura al Execution Role de ECS

---

## Diagrama de Red
```
┌─────────────────────────────────────────────────────────────────────┐
│ VPC (10.0.0.0/16)                                                   │
│                                                                     │
│  ┌─────────────────────┐    ┌─────────────────────────────────────┐ │
│  │ Subnets Públicas     │    │ Subnets Privadas                    │ │
│  │                     │    │                                     │ │
│  │  ┌───────────────┐  │    │  ┌───────────────────────────────┐  │ │
│  │  │     ALB       │──│────│──│  ECS Fargate Task              │  │ │
│  │  │ (443 → 8080)  │  │    │  │  ┌──────────┐ ┌────────────┐  │  │ │
│  │  └───────────────┘  │    │  │  │ Gateway  │ │ DD Agent   │  │  │ │
│  │                     │    │  │  │ (8080)   │ │ (sidecar)  │  │  │ │
│  │  ┌───────────────┐  │    │  │  └──────────┘ └────────────┘  │  │ │
│  │  │  NAT Gateway  │  │    │  └───────────────────────────────┘  │ │
│  │  └───────────────┘  │    │            │                        │ │
│  └─────────────────────┘    │            ▼                        │ │
│                             │    ┌───────────────┐                │ │
│                             │    │    Omega       │                │ │
│                             │    │  (monolito)    │                │ │
│                             │    └───────────────┘                │ │
│                             └─────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Entregables del Equipo de Desarrollo
1. **Imagen Docker** en ECR (la sube el CI/CD de GitHub Actions)
2. **Terraform modules** en `poc/shared/terraform/modules/` y `poc/opcion-b-spring-gateway/terraform/`
3. **Health check path:** `/actuator/health`
4. **Puerto de la aplicación:** `8080`
5. **Variables de entorno requeridas:** tabla en sección 4

## Información que Necesitamos del Equipo de Infra
1. **URL interna de Omega** (para configurar `OMEGA_URL`)
2. **ARN del certificado SSL** en ACM
3. **Dominio DNS** a usar para el gateway
4. **DD_API_KEY** (almacenada en Secrets Manager)
5. **¿Omega tiene validación de origen (IP whitelist, mTLS)?** → Si es así, necesitamos que el SG/IP del ECS Task esté whitelisted en Omega
