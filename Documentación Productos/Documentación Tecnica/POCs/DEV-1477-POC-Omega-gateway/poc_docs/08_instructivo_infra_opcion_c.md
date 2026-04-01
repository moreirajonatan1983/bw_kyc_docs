# Instructivo para el Equipo de Infraestructura — Opción C: NGINX Reverse Proxy

## Descripción General
Esta opción despliega un contenedor **NGINX** como reverse proxy en ECS Fargate. No ejecuta lógica de negocio en Java; el ruteo se configura mediante `nginx.conf` con directivas `map` para resolver el `branchId` del header HTTP.

---

## Arquitectura de Despliegue
```
Internet → ALB (HTTPS) → ECS Fargate Task [NGINX + Datadog Agent Sidecar] → Omega / Microservicios
```

---

## Tareas a Realizar

### 1. Networking (VPC)
Idéntico a Opción B:
- [ ] VPC con subnets públicas y privadas en 2+ AZs
- [ ] Internet Gateway + NAT Gateway
- [ ] Route Tables configuradas

### 2. ECR
- [ ] Crear repositorio ECR: `bw_omega-gateway-nginx`
- [ ] Image scanning on push habilitado

### 3. ECS Cluster
- [ ] Crear o reutilizar cluster: `bw_omega-gateway-{environment}`
- [ ] Container Insights habilitado

### 4. Task Definition
- [ ] **Contenedor principal: `gateway`**
  - Imagen: `{account}.dkr.ecr.{region}.amazonaws.com/bw_omega-gateway-nginx:latest`
  - **CPU: 512** (0.5 vCPU — NGINX es muy liviano)
  - **Memoria: 1024 MB**
  - Puerto: 8080
  - Health check: `curl -f http://localhost:8080/nginx-health`

  > ⚡ Diferencia clave vs Opción B: NGINX necesita **la mitad de recursos** que Spring Cloud Gateway.

- [ ] **Contenedor sidecar: `datadog-agent`** (igual que Opción B)
  - Imagen: `public.ecr.aws/datadog/agent:latest`
  - Variables de entorno: misma configuración que Opción B

### 5. ECS Service
- [ ] Desired count: **2** (mínimo HA)
- [ ] Deployment: 100% min / 200% max
- [ ] Circuit breaker habilitado
- [ ] Health check grace period: **10 segundos** (NGINX arranca casi instantáneamente)
- [ ] Subnets privadas, sin IP pública

### 6. ALB
- [ ] Crear ALB en subnets públicas (idéntico a Opción B)
- [ ] Target Group:
  - Puerto: 8080
  - Health check path: `/nginx-health`
  - Interval: 10s (NGINX responde muy rápido)
  - Timeout: 3s

### 7. Security Groups
Idéntico a Opción B:
- [ ] SG ALB: 443/80 desde internet
- [ ] SG ECS: 8080 solo desde SG del ALB

### 8. CloudWatch
- [ ] Log Group: `/ecs/bw_omega-gateway-{environment}`
- [ ] Alarmas idénticas a Opción B

### 9. Autoscaling
- [ ] Min: 2, Max: 20
- [ ] CPU target: 60%
- [ ] Request count target: 2000 req/target (NGINX maneja más requests por instancia)

### 10. DNS
- [ ] `gateway.{dominio}` → ALB DNS name

---

## Diferencias con Opción B (Spring Cloud Gateway)

| Aspecto | Opción B (Spring) | Opción C (NGINX) |
|---|---|---|
| CPU/Memoria por task | 1024 CPU / 2048 MB | **512 CPU / 1024 MB** |
| Health check grace period | 60s | **10s** |
| Health check path | `/actuator/health` | `/nginx-health` |
| Arranque | ~15-30s (JVM) | **< 1s** |
| Costo estimado por task | ~$30/mes | **~$15/mes** |

---

## Limitaciones de esta Opción (informar al equipo)
- Si se necesita leer datos de sesión para derivar `branchId`, se requiere **OpenResty + Lua**, lo cual agrega complejidad
- No tiene integración nativa con Datadog APM Java Agent (se usan logs + módulo NGINX de Datadog)
- Cambios de ruteo requieren rebuild de la imagen Docker (no hot-reload como Spring)
- Debugging más difícil que en una app Java con stack traces

---

## Entregables del Equipo de Desarrollo
1. **Imagen Docker** con NGINX configurado (la sube CI/CD)
2. **Terraform** en `poc/opcion-c-nginx/terraform/`
3. **Health check path:** `/nginx-health`
4. **Puerto:** `8080`

## Información que Necesitamos del Equipo de Infra
1. **URL interna de Omega** (para `proxy_pass` en nginx.conf)
2. **ARN del certificado SSL**
3. **Dominio DNS**
4. **DD_API_KEY** en Secrets Manager
5. **¿Omega tiene validación de origen?** (misma pregunta que Opción B)
