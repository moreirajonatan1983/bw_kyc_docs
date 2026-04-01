# Próximos Pasos y Pendientes

## Pendientes Técnicos

### Fase 3: Resolución de branchId desde Sesión

- [ ] Definir qué campo del token de sesión (`OMEGA_SESSION` / JWT) contiene el `branchId`.
- [ ] Confirmar si la clave Redis es `session:{token}` o una derivación del mismo.
- [ ] Implementar el parser en `cmd/server/main.go` dentro de `gatewayRouter`.

### Validación de Origen (Crítico antes de Prod)

> 🛑 **PRECAUCIÓN:**
> **¿Omega valida de dónde proviene el tráfico?** Si Omega tiene IP Whitelist, header de origen secreto, API Key o mTLS, el gateway debe presentar esas credenciales. Esto debe validarse con el equipo de Omega antes del Go-Live.

- [ ] Confirmar si Omega tiene IP Whitelist (agregar IPs de las Tasks de Fargate).
- [ ] Confirmar si requiere mTLS / API Key / header especial de origen.
- [ ] Confirmar qué pasa si Omega recibe un request sin `branchId` en el header ¿lo acepta?

### URLs de Destino en Producción

- [ ] Definir los endpoints internos reales de AWS para cada microservicio:
  - `OMEGA_HOST` — ALB interno del monolito
  - `PAGOS_HOST` — Service Discovery o ALB de ms-pagos
  - `USUARIOS_HOST` — Service Discovery o ALB de ms-usuarios

### Dimensionamiento de Fargate

- [ ] Confirmar cuántos proveedores/integraciones simultáneas se esperan.
- [ ] Definir el Task size (vCPU / RAM) y el Auto Scaling policy (target tracking en ALB RequestCountPerTarget).

---

## Pendientes de Infraestructura (SRE)

- [ ] Crear repositorio de Terraform centralizado y conectarlo con `terraform/` del gateway.
- [ ] Configurar secrets `AWS_ROLE_ARN` y `DD_API_KEY` en GitHub → Settings → Secrets.
- [ ] Emitir certificado TLS en ACM para el dominio del gateway.
- [ ] Configurar alarmas en Datadog: latencia P99 > 50ms, error rate > 0.5%.
- [ ] Definir Retention Policy de logs en CloudWatch.

---

## Roadmap de Fases

| Fase | Estado | Descripción |
|---|---|---|
| Fase 1 — Pasamanos | ✅ Implementado | Todo el tráfico → Omega. Gateway transparente. |
| Fase 2 — Ruteo por branchId | ✅ Implementado | `routes.yaml` con reglas por branchId + path. |
| Fase 3 — Resolución de sesión | 🔴 Pendiente | Derivar branchId desde `OMEGA_SESSION` en Redis. |
| CI/CD (GitHub Actions) | 🟡 Pendiente | Pipeline de build, test y deploy a ECR/Fargate. |
| Terraform Production | 🟡 Pendiente | Provisionar infraestructura real en AWS. |
| Go-Live | 🔴 Pendiente | DNS Cutover en Route53. |
