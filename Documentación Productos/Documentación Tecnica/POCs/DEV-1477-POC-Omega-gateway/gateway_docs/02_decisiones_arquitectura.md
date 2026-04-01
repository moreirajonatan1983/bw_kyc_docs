# ADR — Architecture Decision Records

## ADR-001: Tecnología del Gateway

- **Estado:** Aceptado  
- **Fecha:** 2026-04-01  
- **Contexto:** Se necesita un componente proxy/gateway para migración transparente del monolito a microservicios, sin impactar proveedores externos.  
- **Opciones evaluadas:** ALB Rules (A), Spring Cloud Gateway (B), NGINX (C), AWS API Gateway (D), CloudFront+Lambda@Edge (E), KrakenD (F), **Custom Go Gateway (G)**, Node.js (H).  
- **Decisión:** **Custom Go API Gateway (Opción G)** — validado empíricamente con K6.  
- **Consecuencias:**
  - P99 < 1ms, RAM < 20MB, cold start < 5ms.
  - Trazabilidad Datadog nativa con `dd-trace-go`.
  - Deploy en Fargate con binario estático en Alpine (~20MB imagen).

---

## ADR-002: Estrategia de Despliegue

- **Estado:** Aceptado  
- **Fecha:** 2026-04-01  
- **Decisión:** AWS ECS Fargate con Terraform (IaC) y GitHub Actions (CI/CD).  
- **Consecuencias:**
  - Sin gestión de EC2, zero-downtime deployments.
  - Patrón Sidecar para Datadog Agent.
  - Go-Live vía DNS Cutover en Route53 (rollback < 3 segundos).

---

## ADR-003: Patrón Strangler Fig (Migración Progresiva)

- **Estado:** Aceptado  
- **Fecha:** 2026-04-01  
- **Decisión:** Migración progresiva con 3 fases:
  1. **Fase 1:** Todo el tráfico → Omega. Gateway completamente transparente.
  2. **Fase 2:** Ruteo por `branchId` para microservicios migrados.
  3. **Fase 3:** Resolución de `branchId` desde sesión Redis.

---

## ADR-004: Almacenamiento de Reglas de Ruteo (GitOps)

- **Estado:** Aceptado  
- **Fecha:** 2026-04-01  
- **Opciones evaluadas:**
  1. **GitOps File (`routes.yaml` en RAM)** — Latencia: 0ms, Costo: $0 ✅
  2. Variables de entorno — Difícil para mapeos complejos ❌
  3. Redis por cada request — +0.5ms latencia permanente ❌
- **Decisión:** `routes.yaml` parseado en RAM al arrancar. Cambios vía Pull Request auditado.  
- **Consecuencias:** Ruteo O(n) en memoria pura. Auditoría trazable en Git.

---

## ADR-005: Diseño Plano vs Arquitectura Hexagonal

- **Estado:** Aceptado  
- **Fecha:** 2026-04-01  
- **Decisión:** **Se excluye Arquitectura Hexagonal.** El gateway usa diseño plano (`main.go`).  
- **Justificación:** El gateway es un adaptador de infraestructura (HTTP→HTTP), no tiene dominio de negocio propio. La hexagonal es mandatoria para microservicios transaccionales (`ms-pagos`, `ms-apuestas`), no aquí.

---

## ADR-006: Testing Continuo (QA Gate)

- **Estado:** Aceptado  
- **Fecha:** 2026-04-01  
- **Decisión:**
  - `go test` → Regresión en cada commit.
  - Smoke tests (bash/PowerShell) → Post-deploy.
  - Grafana K6 → Thresholds: P99 < 200ms, error rate < 1%, routing success > 99%.
  - Trivy → Image scanning en cada build.

---

## Plantilla para Nuevas Decisiones

```markdown
## ADR-XXX: [Título]

- **Estado:** Propuesto | Aceptado | Deprecado | Reemplazado
- **Fecha:** YYYY-MM-DD
- **Contexto:** [Por qué se necesita esta decisión]
- **Opciones evaluadas:** [Lista de alternativas]
- **Decisión:** [Qué se decidió y por qué]
- **Consecuencias:** [Impacto positivo y negativo]
```
