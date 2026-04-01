# Contexto y Arquitectura — bw_omega-gateway

## 1. El Problema

**Omega** es una plataforma de **juegos online (betting)** que opera como monolito. Sus servicios son consumidos por múltiples proveedores externos a través de una **URL única**. La migración progresiva hacia microservicios implica nuevas URLs internas, sin que los proveedores deban cambiar sus integraciones.

> ⚠️ **ADVERTENCIA:**
> Omega recibe **alto volumen de tráfico** en condiciones normales, multiplicado durante eventos deportivos de alto perfil (Mundial, Champions, Super Bowl). El gateway debe soportar estos picos sin degradar latencia.

## 2. La Solución

Interponer el **bw_omega-gateway** como componente proxy/gateway que:

1. **Fase 1 — Pasamanos 100% transparente:** Recibe todo el tráfico y lo reenvía a Omega sin ninguna modificación (headers, body, query params, cookies, método HTTP viajan intactos).
2. **Fase 2 — Ruteo por `branchId`:** Identifica al proveedor por el header `branchId` y lo dirige al microservicio correspondiente o hace fallback a Omega.
3. **Fase 3 — Resolución de sesión:** Si `branchId` no viene en el header, lo deduce del token de sesión `OMEGA_SESSION` vía Redis.

### Principio Fundamental: Pasamanos Transparente

> ❗ **IMPORTANTE:**
> El gateway **NO modifica la invocación** en ningún aspecto excepto la URL de destino. Todo el contenido del request original (método HTTP, headers, body, query params, cookies) llega **intacto** al destino.

## 3. Arquitectura Elegida

### Diagrama Visual

![Diagrama de Arquitectura — bw_omega-gateway](assets/architecture_diagram.png)

### Diagrama Técnico (Mermaid)

```mermaid
flowchart TD
    subgraph Internet
        PROV["🌐 Internet / Proveedores Externos"]
    end

    subgraph AWS["☁️ AWS Cloud"]
        WAF["🛡️ AWS WAF\nDDoS · XSS · OWASP Rules"]
        R53["Route53 DNS\napi.omega.com CNAME"]
        ALB["ALB — Application Load Balancer\nSSL Termination · Health Checks"]

        subgraph ECS["ECS Fargate Cluster"]
            GW["🚀 bw_omega-gateway\nGo · Port 8080\nroutes.yaml en RAM — 0ms overhead"]
            DD["📊 Datadog Agent\nSidecar APM"]
        end

        REDIS["🔴 ElastiCache Redis\nResolución de sesión:\nsession:{token} → branchId"]
        CW["📝 Amazon CloudWatch\nApplication Logs (awslogs)"]
    end

    subgraph Backends["Destinos"]
        OMEGA["⬜ Omega Monolith\nFallback — tráfico no mapeado"]
        PAGOS["🔵 ms-pagos\nbranchId=200 · /api/pagos/*"]
        USERS["🟣 ms-usuarios\nbranchId=300 · /api/usuarios/*"]
    end

    subgraph CICD["CI/CD"]
        GHA["⚙️ GitHub Actions\nBuild · Test · Deploy\nroutes.yaml GitOps"]
    end

    PROV -->|HTTPS| WAF
    R53 -->|CNAME| ALB
    WAF --> ALB
    ALB --> GW
    GW <-->|GET session:token| REDIS
    GW --> DD
    GW -->|Logs| CW

    GW -->|"branchId no match\no sin branchId"| OMEGA
    GW -->|"branchId=200\n+ /api/pagos/*"| PAGOS
    GW -->|"branchId=300\n+ /api/usuarios/*"| USERS

    GHA -->|"docker push ECR\n+ ECS update"| GW

    style GW fill:#d4edda,stroke:#28a745,stroke-width:3px
    style WAF fill:#fff3cd,stroke:#ffc107,stroke-width:2px
    style ALB fill:#cce5ff,stroke:#007bff,stroke-width:2px
    style REDIS fill:#f8d7da,stroke:#dc3545,stroke-width:2px
    style OMEGA fill:#e2e3e5,stroke:#6c757d,stroke-width:2px
    style PAGOS fill:#cce5ff,stroke:#007bff,stroke-width:2px
    style USERS fill:#e8d5f5,stroke:#6f42c1,stroke-width:2px
    style DD fill:#fff3cd,stroke:#ffc107,stroke-width:1px
    style GHA fill:#f0f0f0,stroke:#6c757d,stroke-width:1px
```

## 4. Lógica de Resolución de branchId

```
Request entrante
  │
  ├─ ¿Header branchId presente? ──YES──► Usarlo directamente
  │
  └─ NO ──► Leer cookie OMEGA_SESSION
                │
                └─ Consultar Redis: GET session:{cookie}
                       │
                       ├─ Encontrado ──► Usar branchId de caché
                       └─ No encontrado ──► Fallback a Omega
```

## 5. Por qué Go

| Criterio | Go (Elegido) | Spring Boot (B) | NGINX (C) |
|---|---|---|---|
| Latencia P99 | **< 1ms** | ~12ms | ~2ms |
| RAM en idle | **~20MB** | ~400MB | ~30MB |
| Cold start | **< 5ms** | ~15-30s | ~1s |
| Lógica condicional | ✅ Total | ✅ Total | ⚠️ Solo con Lua |
| Redis integration | ✅ Nativa | ✅ Nativa | ❌ Difícil |
| Datadog APM | ✅ `dd-trace-go` | ✅ Java Agent | ⚠️ Módulo separado |
| Costo Fargate | **Mínimo** | 6x más RAM | Bajo |
