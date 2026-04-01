# Instructivo para el Equipo de Infraestructura — Opción A: ALB + Listener Rules

## Descripción General
Esta opción **no despliega una aplicación nueva**. Consiste en configurar reglas de ruteo directamente en el Application Load Balancer existente de AWS. El equipo de desarrollo entrega las reglas; el equipo de infra las implementa.

---

## Tareas a Realizar

### 1. Prerequisitos (verificar que existen)
- [ ] VPC con subnets públicas y privadas en al menos 2 AZs
- [ ] ALB existente (o crear uno nuevo) en las subnets públicas
- [ ] Certificado SSL en ACM para el dominio del gateway
- [ ] Target Groups existentes para Omega y cada microservicio migrado
- [ ] Security Groups configurados para permitir tráfico ALB → ECS Tasks

### 2. Configuración del ALB
- [ ] Crear Listener HTTPS (puerto 443) con certificado SSL
- [ ] Crear Listener HTTP (puerto 80) con redirect a HTTPS
- [ ] Configurar el Security Policy TLS: `ELBSecurityPolicy-TLS13-1-2-2021-06`

### 3. Target Groups
- [ ] Crear Target Group para Omega (si no existe)
  - Tipo: IP
  - Puerto: 8080
  - Health check: `GET /health` → 200 OK
  - Deregistration delay: 30s
- [ ] Crear Target Groups para cada microservicio que se migre (bajo demanda)

### 4. Listener Rules (las provee el equipo de desarrollo)
- [ ] Regla default: Todo el tráfico → Target Group Omega
- [ ] Reglas por branchId/path: Las proporciona el equipo de desarrollo vía Terraform o ticket

> **Ejemplo de regla:**
> - Prioridad: 10
> - Condición 1: Path pattern = `/api/pagos/*`
> - Condición 2: HTTP Header `branchId` = `200`
> - Acción: Forward → Target Group `ms-pagos`

### 5. Logs y Monitoreo
- [ ] Habilitar **ALB Access Logs** hacia un bucket S3
  - Bucket: `bw_omega-gateway-{env}-alb-logs`
  - Prefijo: `alb-logs/`
- [ ] Configurar CloudWatch Alarms:
  - `HTTPCode_ELB_5XX_Count` > 10 en 5 min → Alarma
  - `TargetResponseTime` p99 > 1s → Alarma
  - `UnHealthyHostCount` > 0 → Alarma

### 6. DNS
- [ ] Crear registro CNAME/Alias en Route 53 apuntando al ALB DNS name
  - `gateway.{dominio}` → ALB DNS name

### 7. Seguridad
- [ ] Security Group del ALB: Permitir ingress 443/80 desde IPs de proveedores (o 0.0.0.0/0 si no hay whitelist)
- [ ] Revisar si Omega tiene validación de IP de origen (ver pregunta abierta en la propuesta)

---

## Limitaciones de esta Opción (informar al equipo)
- **No puede leer datos de sesión** ni el body del request
- **Máximo ~100 reglas** por listener
- Cada nueva regla requiere un cambio de infra (Terraform o consola)
- **No soporta lógica condicional compleja** (ej: "si branchId=X Y el usuario tiene rol Y")

---

## Entregables del Equipo de Desarrollo
1. Archivo Terraform con las Listener Rules: `poc/opcion-a-alb/terraform/main.tf`
2. Lista de reglas de ruteo en formato tabla (Excel/Confluence)
3. Health check paths para cada servicio

## Contacto
Para dudas sobre las reglas de ruteo, contactar al equipo de desarrollo del proyecto `bw_omega-gateway`.
