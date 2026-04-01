# Deploy en Producción (AWS Fargate)

## Prerequisitos (a cargo de SRE/DevOps)

### 1. Configurar Secretos en GitHub

El pipeline CI/CD en `.github/workflows/` requiere los siguientes secrets en el repositorio:

| Secret | Descripción |
|---|---|
| `AWS_ROLE_ARN` | ARN del Rol IAM con permisos de push a ECR y `UpdateService` en ECS (via OIDC) |
| `DD_API_KEY` | Datadog API Key para el Agent sidecar |

### 2. Inicializar Terraform (primera vez)

```bash
cd terraform/
terraform init
terraform plan -var="environment=production" -var="datadog_api_key=$DD_API_KEY"
terraform apply -var="environment=production" -var="datadog_api_key=$DD_API_KEY"
```

Esto aprovisiona automáticamente:
- VPC y subredes
- ALB (Application Load Balancer) con SSL termination
- WAF Regional (OWASP Core Rule Set)
- ECS Cluster + Fargate Task Definition (Gateway + Datadog Sidecar)
- ElastiCache Redis (resolución de sesión)

### 3. Emitir Certificado TLS (ACM)

```bash
# Solicitar el certificado vía AWS Certificate Manager
# Luego pasar el ARN al Terraform:
terraform apply -var="certificate_arn=arn:aws:acm:..."
```

---

## Variables de Entorno en Producción (Task Definition ECS)

```json
{
  "environment": [
    { "name": "APP_ENV",       "value": "prod" },
    { "name": "REDIS_ADDR",    "value": "your-redis.cache.amazonaws.com:6379" },
    { "name": "OMEGA_HOST",    "value": "http://omega-internal-alb.vpc.local" },
    { "name": "PAGOS_HOST",    "value": "http://ms-pagos-alb.vpc.local" },
    { "name": "USUARIOS_HOST", "value": "http://ms-usuarios-alb.vpc.local" },
    { "name": "DD_ENV",        "value": "production" },
    { "name": "DD_SERVICE",    "value": "bw_omega-gateway" }
  ]
}
```

---

## Estrategia de Go-Live (DNS Cutover)

> ❗ **IMPORTANTE:**
> No se requiere ventana de mantenimiento ni coordinación con proveedores externos.

1. **Verificar** que el ALB devuelve `200 OK` en el health check del gateway.
2. **Verificar** trazas en Datadog APM (servicio `bw_omega-gateway` visible).
3. **Actualizar Route53:** Cambiar el CNAME de `api.omega.com` del ALB actual de Omega al nuevo ALB del Gateway.
4. **Monitorear** el dashboard de Datadog durante los primeros 15 minutos.

### Rollback de Emergencia (< 3 segundos)

```bash
# Revertir el registro DNS en Route53 para volver a apuntar directo a Omega
aws route53 change-resource-record-sets --hosted-zone-id ZXXXXX \
  --change-batch file://rollback-dns.json
```

---

## Agregar un Nuevo Microservicio

1. Editar `configs/routes.prod.yaml` agregando la nueva regla.
2. Abrir un Pull Request → merge a `main`.
3. El pipeline CI/CD construye la nueva imagen y actualiza el Fargate Task.
4. ECS hace rolling update sin downtime (new task → drain old task).

> ℹ️ **NOTA:**
> El binario Go no se recompila para cambios de rutas. Solo cambia el `configs/routes.prod.yaml` empaquetado en la imagen Docker.
