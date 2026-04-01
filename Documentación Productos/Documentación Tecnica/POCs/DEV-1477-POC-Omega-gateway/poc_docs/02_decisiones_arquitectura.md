# ADR (Architecture Decision Records) — bw_omega-gateway

## Registro de Decisiones de Arquitectura

Este documento registra las decisiones técnicas y arquitectónicas tomadas durante el desarrollo del proyecto `bw_omega-gateway`. Cada decisión incluye el contexto, las opciones evaluadas, la decisión final y sus consecuencias.

---

## ADR-001: Tecnología del Gateway

- **Estado:** Aceptado
- **Fecha:** 2026-03-31 (Actualizado 2026-04-01)
- **Contexto:** Se necesita un componente proxy/gateway que permita la migración transparente del monolito a microservicios sin impactar a proveedores externos.
- **Opciones evaluadas:** ALB Rules, Spring Cloud Gateway, NGINX/OpenResty, Node.js, AWS API Gateway, Custom Go API Gateway (ver [propuesta_gateway.md](01_propuesta_gateway.md)).
- **Decisión final re-evaluada:** **Custom Go API Gateway (Opción G)** interactuando con **Redis In-Memory** tras validación empírica en Spike testing K6.
- **Consecuencias:**
  - El Gateway demostró latencia de ruteo P99 por debajo de 1ms consumiendo <20MB de RAM.
  - Se reutiliza la trazabilidad Datadog ya implementada nativamente mediante `dd-trace-go`.
  - Despliegues ultra-rápidos en Fargate gracias a binarios estáticos sin recargo de capas JVM.

---

## ADR-002: Estrategia de Despliegue

- **Estado:** Aceptado
- **Fecha:** 2026-03-31
- **Contexto:** El gateway debe ejecutarse en AWS con alta disponibilidad.
- **Decisión:** AWS ECS Fargate con Terraform como IaC y Github Actions.
- **Consecuencias:**
  - Sin gestión de instancias EC2, permitiendo Zero-Downtime deployments (testeados via k6).
  - Patrón Sidecar para Datadog Agent (consistente con la solución de trazabilidad existente).

---

## ADR-003: Estrategia de Migración (Strangler Fig Pattern)

- **Estado:** Aceptado
- **Fecha:** 2026-03-31
- **Contexto:** La migración del monolito a microservicios será progresiva. Se necesita una estrategia que no requiera un "big bang".
- **Decisión:** Implementar el patrón **Strangler Fig**:
  1. El gateway arranca ruteando todo al monolito (Fase 1: Pasamanos).
  2. A medida que se extraen microservicios, se agregan rutas específicas usando el Header `branchId`.
  3. Cuando todo esté migrado, se retira el monolito de Omega.

---

## ADR-004: Aseguramiento de Calidad Continua (Testing)

- **Estado:** Aceptado
- **Fecha:** 2026-04-01
- **Contexto:** Prevenir degradación de performance y seguridad con código nuevo introducido en el Gateway.
- **Decisión:** 
  - Jenkins/GH Actions ejecutarán Trivy image scanning sobre cada build.
  - Pruebas Funcionales en bash (CI Smoke Testing).
  - Grafana k6 inyectará assert thresholds (rate de éxito de enrutamiento 100%, P99 Latency < 200ms) pre-producción.
- **Consecuencias:**
  - Despliegues pueden bloquearse si K6 excede thresholds de latencia.
  - Mayor madurez del SDLC del proyecto.

---

## ADR-005: Almacenamiento Dinámico vs Estático de Reglas de Ruteo (Mapping)

- **Estado:** Aceptado
- **Fecha:** 2026-04-01
- **Contexto:** Se deben almacenar las reglas direccionales de cada proveedor (`branchId 200 -> URL_MS_PAGOS`) sin hardcodearlas en el binario de Go.
- **Opciones evaluadas evaluando Latencia y Costo:**
  1. **GitOps File (`routes.yaml` en RAM):** Las rutas se parsean desde un YAML en tiempo de arranque con reloads via Pipeline. Latencia=0ms, Costo=$0.
  2. **Variables de Entorno Estáticas (`ENV ROUTES`):** Latencia=0ms, Costo=$0. Dificultad severa para mapeo complejo anidado.
  3. **Mapeo Dinámico constante en Redis:** Consultar la base Redis por *cada* request entrante buscando a qué URL derivar. Latencia=+0.5ms adicionales permanentes de red, Costo=Incremento del recargo de IO y Fargate Data Transfer-IN/OUT en alto volumen.
- **Decisión:** **Opción 1 (GitOps File via `routes.yaml`)**. 
- **Consecuencias:**
  - **Latencia Optimizada (0ms):** El enrutamiento es una operación O(n) instantánea ejecutada 100% en memoria en la Goroutine actual sin requerir Network Hops extras a sub-sistemas de AWS.
  - **Costo nulo:** Fargate no factura extra por el simple uso de la RAM reservada, pero sí habría costo computacional o de red extra si abusáramos de Redis para configuraciones estáticas.
  - **Seguridad y Auditoría:** Los Microservicios futuros se habilitarán modificando `routes.yaml` via un Pull Request aprobado y trazable, previniendo caídas totales de ruteo por purgas accidentales en Redis.

---

## ADR-006: Exclusión de Arquitectura Hexagonal (Ports & Adapters)

- **Estado:** Aceptado
- **Fecha:** 2026-04-01
- **Contexto:** Se planteó aplicar los principios de la Arquitectura Hexagonal (habitual en los microservicios corporativos creados con Go) directamente en el proyecto del `bw_omega-gateway` (Opción G).
- **Decisión:** **Se excluye rotundamente el uso de Arquitectura Hexagonal** prefiriendo un Diseño Plano (Event-Loop directo) en `main.go`.
- **Consecuencias:**
  - **Over-engineering:** Un API Gateway **no** es un conjunto de reglas de negocio intrínsecas (ej. no procesa tarjetas de crédito, no tiene dominio *Usuario*, etc). Es un adaptador de infraestructura viva de red (HTTP a HTTP).
  - **Latencia y Memory Footprint:** Abstraer el `httputil.ReverseProxy` en docenas de carpetas con Puertos y Adaptadores solo genera milisegundos de overhead al encadenar interfaces abstractas sin sumar ningún beneficio estructural real de aislamiento (ya que todo en este punto es Infra).
  - **Facilidad de Auditoría:** El diseño lineal permite a un Site Reliability Engineer (SRE) leer y parchear configuraciones de Headers sin tener que indagar el ecosistema de *UseCases*.
  - **Alineación:** La arquitectura Hexagonal quedará restringida estrictamente para los **Back-Ends y Microservicios internos** transaccionales (como `ms-pagos` / `ms-apuestas`), donde el desacople del modelo de Base de Datos es mandatario.

---

## Plantilla para Nuevas Decisiones

```markdown
## ADR-XXX: [Título]

- **Estado:** Propuesto | Aceptado | Deprecado | Reemplazado
- **Fecha:** YYYY-MM-DD
- **Contexto:** [Por qué se necesita esta decisión]
- **Opciones evaluadas:** [Lista de alternativas consideradas]
- **Decisión:** [Qué se decidió y por qué]
- **Consecuencias:** [Impacto positivo y negativo de la decisión]
```
