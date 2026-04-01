# Diagramas de Arquitectura — bw_omega-gateway

> **Contexto:** bw_omega-gateway es el proxy/gateway que se interpone entre los proveedores externos y **Omega** (plataforma de juegos online). Omega recibe alto tráfico, especialmente durante eventos deportivos (Mundial, Champions, etc.).

Cada opción se presenta con un diagrama de arquitectura y un diagrama de secuencia que muestra el flujo de una petición típica.

---

## Opción A: AWS ALB + Listener Rules (Solo Infra)

### Diagrama de Arquitectura

```mermaid
graph LR
    subgraph Proveedores
        P1["Proveedor 1"]
        P2["Proveedor 2"]
        P3["Proveedor N"]
    end

    subgraph AWS Cloud
        subgraph VPC
            ALB["ALB<br/>Listener Rules<br/>SSL Termination"]
            
            subgraph ECS Cluster
                subgraph TG_Omega["Target Group: Omega"]
                    OMEGA["Omega<br/>(Fargate Task)"]
                end
                subgraph TG_MS1["Target Group: ms-pagos"]
                    MS1["ms-pagos<br/>(Fargate Task)"]
                end
                subgraph TG_MS2["Target Group: ms-usuarios"]
                    MS2["ms-usuarios<br/>(Fargate Task)"]
                end
            end
        end
        CW["CloudWatch Logs"]
    end

    P1 -->|HTTPS| ALB
    P2 -->|HTTPS| ALB
    P3 -->|HTTPS| ALB
    ALB -->|"Rule: /api/pagos/** <br/> Header: branchId"| MS1
    ALB -->|"Rule: /api/usuarios/**"| MS2
    ALB -->|"Default Rule: /**"| OMEGA
    OMEGA --> CW
    MS1 --> CW
    MS2 --> CW

    style ALB fill:#8C4FFF,stroke:#fff,color:#fff
    style OMEGA fill:#ED7100,stroke:#fff,color:#fff
    style MS1 fill:#ED7100,stroke:#fff,color:#fff
    style MS2 fill:#ED7100,stroke:#fff,color:#fff
    style CW fill:#E7157B,stroke:#fff,color:#fff
```

### Diagrama de Secuencia

```mermaid
sequenceDiagram
    participant P as Proveedor
    participant ALB as AWS ALB
    participant OMEGA as Omega
    participant MS as Microservicio

    P->>ALB: POST /api/pagos (Header: branchId=100)
    
    alt Path matchea Listener Rule
        ALB->>MS: Forward al Target Group ms-pagos
        MS-->>ALB: 200 OK (Response)
        ALB-->>P: 200 OK
    else No matchea ninguna regla
        ALB->>OMEGA: Forward al Target Group Default (Omega)
        OMEGA-->>ALB: 200 OK (Response)
        ALB-->>P: 200 OK
    end

    Note over ALB: Limitación: No puede leer<br/>body ni datos de sesión<br/>No puede derivar branchId desde sesión
```

---

## Opción B: Spring Cloud Gateway (Fargate) — RECOMENDADA

### Diagrama de Arquitectura

```mermaid
graph LR
    subgraph Proveedores
        P1["Proveedor 1"]
        P2["Proveedor 2"]
        P3["Proveedor N"]
    end

    subgraph AWS Cloud
        subgraph VPC
            ALB["ALB<br/>SSL Termination<br/>Health Checks"]
            
            subgraph ECS Cluster
                subgraph Task_GW["Task Definition: bw_omega-gateway"]
                    CONF["application.yml<br/>(RAM)"]
                    GW["Spring Cloud<br/>Gateway<br/>(Netty)"]
                    DD_GW["Datadog Agent<br/>(Sidecar)"]
                end
                subgraph Task_Omega["Task: Omega"]
                    OMEGA["Omega<br/>(Monolito)"]
                end
                subgraph Task_MS1["Task: ms-pagos"]
                    MS1["ms-pagos"]
                end
                subgraph Task_MS2["Task: ms-usuarios"]
                    MS2["ms-usuarios"]
                end
            end
        end
        CW["CloudWatch Logs"]
        DDAPM["Datadog APM"]
    end

    P1 -->|HTTPS| ALB
    P2 -->|HTTPS| ALB
    P3 -->|HTTPS| ALB
    ALB -->|HTTP| GW
    CONF -.->|"Reglas"| GW
    GW -->|"Default / branchId<br/>desconocido"| OMEGA
    GW -->|"branchId=200"| MS1
    GW -->|"branchId=300"| MS2
    GW -.->|"127.0.0.1:8126"| DD_GW
    DD_GW -.-> DDAPM
    GW --> CW

    style ALB fill:#8C4FFF,stroke:#fff,color:#fff
    style GW fill:#1A73E8,stroke:#fff,color:#fff
    style DD_GW fill:#632CA6,stroke:#fff,color:#fff
    style OMEGA fill:#ED7100,stroke:#fff,color:#fff
    style MS1 fill:#ED7100,stroke:#fff,color:#fff
    style MS2 fill:#ED7100,stroke:#fff,color:#fff
    style CW fill:#E7157B,stroke:#fff,color:#fff
    style CONF fill:#F0E68C,stroke:#333,color:#333
    style DDAPM fill:#632CA6,stroke:#fff,color:#fff
```

### Diagrama de Secuencia — Fase 1 (Pasamanos Transparente)

```mermaid
sequenceDiagram
    participant P as Proveedor
    participant ALB as AWS ALB
    participant GW as Spring Cloud Gateway
    participant DD as Datadog Agent
    participant OMEGA as Omega

    P->>ALB: POST /api/apuestas/crear (Header: branchId=100)
    Note right of P: Body, headers, cookies,<br/>query params viajan intactos
    ALB->>GW: Forward (agrega X-Amzn-Trace-Id)
    
    Note over GW: Fase 1: Ruta default /**<br/>Forwadea TODO a Omega<br/>Solo cambia la URL destino<br/>El request pasa INTACTO

    GW->>OMEGA: Proxy transparente (método, headers, body, cookies, query params intactos)
    OMEGA-->>GW: 200 OK (Response)
    GW-->>ALB: 200 OK
    ALB-->>P: 200 OK

    GW-->>DD: Envía trace (dd.trace_id + aws.trace_id)
    DD-->>DD: Push a Datadog Cloud
```

### Diagrama de Secuencia — Fase 2 (Ruteo por branchId en Header)

```mermaid
sequenceDiagram
    participant P as Proveedor
    participant ALB as AWS ALB
    participant GW as Spring Cloud Gateway
    participant OMEGA as Omega
    participant MS as ms-pagos

    P->>ALB: POST /api/pagos (Header: branchId=200)
    ALB->>GW: Forward request

    Note over GW: Lee Header: branchId=200<br/>Consulta tabla de ruteo<br/>branchId=200 → ms-pagos

    alt branchId tiene destino asignado
        GW->>MS: Rutea a ms-pagos (request intacto, solo cambia URL)
        MS-->>GW: 200 OK
    else branchId sin destino o desconocido
        GW->>OMEGA: Fallback a Omega (request intacto)
        OMEGA-->>GW: 200 OK
    end

    GW-->>ALB: 200 OK
    ALB-->>P: 200 OK
```

### Diagrama de Secuencia — Fase 3 (Resolución de branchId desde Sesión)

```mermaid
sequenceDiagram
    participant P as Proveedor
    participant ALB as AWS ALB
    participant GW as Spring Cloud Gateway
    participant SESSION as Datos de Sesión
    participant OMEGA as Omega
    participant MS as ms-pagos

    P->>ALB: POST /api/pagos (SIN header branchId)
    ALB->>GW: Forward request

    Note over GW: Header branchId AUSENTE<br/>Debe resolver desde sesión

    GW->>SESSION: Consultar datos de sesión del request
    SESSION-->>GW: Sesión contiene branchId=200
    
    Note over GW: branchId resuelto: 200<br/>Consulta tabla de ruteo<br/>branchId=200 → ms-pagos

    alt branchId resuelto tiene destino
        GW->>MS: Rutea a ms-pagos (request intacto)
        MS-->>GW: 200 OK
    else No se pudo resolver branchId
        GW->>OMEGA: Fallback a Omega
        OMEGA-->>GW: 200 OK
    end

    GW-->>ALB: 200 OK
    ALB-->>P: 200 OK

    Note over SESSION: ⚠️ Lógica de derivación<br/>aún NO definida
```

---

## Opción C: NGINX / OpenResty (Fargate)

### Diagrama de Arquitectura

```mermaid
graph LR
    subgraph Proveedores
        P1["Proveedor 1"]
        P2["Proveedor 2"]
        P3["Proveedor N"]
    end

    subgraph AWS Cloud
        subgraph VPC
            ALB["ALB<br/>SSL Termination"]
            
            subgraph ECS Cluster
                subgraph Task_NGX["Task: bw_omega-gateway"]
                    CONF["nginx.conf<br/>(map directive)"]
                    NGX["NGINX / OpenResty<br/>(Reverse Proxy)"]
                end
                subgraph Task_Omega["Task: Omega"]
                    OMEGA["Omega"]
                end
                subgraph Task_MS1["Task: ms-pagos"]
                    MS1["ms-pagos"]
                end
            end
        end
        CW["CloudWatch Logs"]
    end

    P1 -->|HTTPS| ALB
    P2 -->|HTTPS| ALB
    P3 -->|HTTPS| ALB
    ALB -->|HTTP| NGX
    CONF -.->|"Reglas"| NGX
    NGX -->|"location /api/pagos"| MS1
    NGX -->|"location / (default)"| OMEGA
    NGX --> CW

    style ALB fill:#8C4FFF,stroke:#fff,color:#fff
    style NGX fill:#009639,stroke:#fff,color:#fff
    style OMEGA fill:#ED7100,stroke:#fff,color:#fff
    style MS1 fill:#ED7100,stroke:#fff,color:#fff
    style CW fill:#E7157B,stroke:#fff,color:#fff
    style CONF fill:#F0E68C,stroke:#333,color:#333
```

### Diagrama de Secuencia

```mermaid
sequenceDiagram
    participant P as Proveedor
    participant ALB as AWS ALB
    participant NGX as NGINX / OpenResty
    participant OMEGA as Omega
    participant MS as Microservicio

    P->>ALB: POST /api/pagos (Header: branchId=100)
    ALB->>NGX: Forward request

    Note over NGX: Evalúa location blocks<br/>y map $http_branchid

    alt location matchea /api/pagos + branchId en map
        Note over NGX: Requiere script Lua<br/>para lógica compleja
        NGX->>MS: proxy_pass http://ms-pagos
        MS-->>NGX: 200 OK
    else location default /
        NGX->>OMEGA: proxy_pass http://omega
        OMEGA-->>NGX: 200 OK
    end

    NGX-->>ALB: 200 OK
    ALB-->>P: 200 OK
```

---

## Opción D: AWS API Gateway (Servicio Gestionado)

### Diagrama de Arquitectura

```mermaid
graph LR
    subgraph Proveedores
        P1["Proveedor 1"]
        P2["Proveedor 2"]
        P3["Proveedor N"]
    end

    subgraph AWS Cloud
        CONF["Terraform IaC"]
        APIGW["AWS API Gateway<br/>(HTTP API)"]
        
        subgraph VPC
            VPCLink["VPC Link"]
            subgraph ECS Cluster
                subgraph Task_Omega["Task: Omega"]
                    OMEGA["Omega"]
                end
                subgraph Task_MS1["Task: ms-pagos"]
                    MS1["ms-pagos"]
                end
                subgraph Task_MS2["Task: ms-usuarios"]
                    MS2["ms-usuarios"]
                end
            end
        end
        CW["CloudWatch Logs"]
    end

    P1 -->|HTTPS| APIGW
    P2 -->|HTTPS| APIGW
    P3 -->|HTTPS| APIGW
    CONF -.->|"Rutas Desplegadas"| APIGW
    APIGW -->|"Route: /api/pagos"| VPCLink
    APIGW -->|"Route: $default"| VPCLink
    VPCLink --> MS1
    VPCLink --> MS2
    VPCLink --> OMEGA
    APIGW --> CW

    style APIGW fill:#E7157B,stroke:#fff,color:#fff
    style VPCLink fill:#8C4FFF,stroke:#fff,color:#fff
    style OMEGA fill:#ED7100,stroke:#fff,color:#fff
    style MS1 fill:#ED7100,stroke:#fff,color:#fff
    style MS2 fill:#ED7100,stroke:#fff,color:#fff
    style CW fill:#E7157B,stroke:#fff,color:#fff
    style CONF fill:#F0E68C,stroke:#333,color:#333
```

### Diagrama de Secuencia

```mermaid
sequenceDiagram
    participant P as Proveedor
    participant APIGW as AWS API Gateway
    participant VPC as VPC Link
    participant OMEGA as Omega
    participant MS as Microservicio

    P->>APIGW: POST /api/pagos (Header: branchId=100)
    
    Note over APIGW: Evalúa Routes por path/method<br/>No puede leer branchId del header<br/>para decisiones de ruteo complejas

    alt Route matchea /api/pagos
        APIGW->>VPC: Integration → VPC Link
        VPC->>MS: Forward a ms-pagos (NLB)
        MS-->>VPC: 200 OK
        VPC-->>APIGW: 200 OK
    else Route default $default
        APIGW->>VPC: Integration → VPC Link
        VPC->>OMEGA: Forward a Omega (NLB)
        OMEGA-->>VPC: 200 OK
        VPC-->>APIGW: 200 OK
    end

    APIGW-->>P: 200 OK
    
    Note over APIGW: Latencia adicional: ~10-30ms<br/>Costo por request
```

---

## Opción E: CloudFront + Lambda@Edge

### Diagrama de Arquitectura

```mermaid
graph LR
    subgraph Proveedores
        P1["Proveedor 1"]
        P2["Proveedor 2"]
        P3["Proveedor N"]
    end

    subgraph AWS Cloud
        CONF["routes.json<br/>(Lambda Bundle)"]
        CF["CloudFront<br/>Distribution"]
        LE["Lambda@Edge<br/>(origin-request)"]
        
        subgraph VPC
            ALB_OMEGA["ALB Omega"]
            ALB_MS["ALB Microservicios"]
            subgraph ECS Cluster
                OMEGA["Omega"]
                MS1["ms-pagos"]
                MS2["ms-usuarios"]
            end
        end
    end

    P1 -->|HTTPS| CF
    P2 -->|HTTPS| CF
    P3 -->|HTTPS| CF
    CF -->|"origin-request"| LE
    CONF -.->|"Rutas"| LE
    LE -->|"Origin: omega"| ALB_OMEGA
    LE -->|"Origin: microservicios"| ALB_MS
    ALB_OMEGA --> OMEGA
    ALB_MS --> MS1
    ALB_MS --> MS2

    style CF fill:#8C4FFF,stroke:#fff,color:#fff
    style LE fill:#ED7100,stroke:#fff,color:#fff
    style ALB_OMEGA fill:#8C4FFF,stroke:#fff,color:#fff
    style ALB_MS fill:#8C4FFF,stroke:#fff,color:#fff
    style OMEGA fill:#ED7100,stroke:#fff,color:#fff
    style MS1 fill:#ED7100,stroke:#fff,color:#fff
    style MS2 fill:#ED7100,stroke:#fff,color:#fff
    style CONF fill:#F0E68C,stroke:#333,color:#333
```

### Diagrama de Secuencia

```mermaid
sequenceDiagram
    participant P as Proveedor
    participant CF as CloudFront
    participant LE as Lambda@Edge
    participant ALB as ALB
    participant OMEGA as Omega
    participant MS as Microservicio

    P->>CF: POST /api/pagos (Header: branchId=100)
    CF->>LE: Trigger origin-request event

    Note over LE: Node.js/Python function<br/>Lee branchId del header<br/>Decide origin dinámicamente<br/>⚠️ Timeout: 5s máx

    alt branchId indica microservicio
        LE->>CF: Cambia origin → ALB Microservicios
        CF->>ALB: Forward a ALB ms
        ALB->>MS: Forward a ms-pagos
        MS-->>ALB: 200 OK
        ALB-->>CF: 200 OK
    else Default
        LE->>CF: Mantiene origin → ALB Omega
        CF->>ALB: Forward a ALB Omega
        ALB->>OMEGA: Forward a Omega
        OMEGA-->>ALB: 200 OK
        ALB-->>CF: 200 OK
    end

    CF-->>P: 200 OK

    Note over LE: ⚠️ Deploy: ~15-20min<br/>⚠️ No accede a VPC<br/>⚠️ Response máx: 1MB
```

---

---

## Opción G: Custom Go API Gateway + Redis (Fargate) — 👑 ELEGIDA OFICIAL

### Diagrama de Arquitectura
```mermaid
graph LR
    subgraph Proveedores
        P1["Proveedor 1"]
        P2["Proveedor 2"]
        P3["Proveedor N"]
    end

    subgraph AWS Cloud
        subgraph VPC
            ALB["ALB<br/>SSL Termination<br/>Health Checks"]
            
            subgraph ECS Cluster
                subgraph Task_GW["Task Definition: bw_omega-gateway"]
                    CONF["routes.yaml<br/>(GitOps RAM)"]
                    GW["Golang API Gateway<br/>(Goroutines)"]
                end
                subgraph Task_Redis["Task: ElastiCache"]
                    REDIS["Redis In-Memory<br/>(Cache Mapeo Sesión)"]
                end
                subgraph Task_Omega["Task: Omega"]
                    OMEGA["Omega<br/>(Monolito)"]
                end
                subgraph Task_MS1["Task: ms-pagos"]
                    MS1["ms-pagos"]
                end
                subgraph Task_MS2["Task: ms-usuarios"]
                    MS2["ms-usuarios"]
                end
            end
        end
        DDAPM["Datadog APM<br/>(Trace Nativo Go)"]
    end

    P1 -->|HTTPS| ALB
    P2 -->|HTTPS| ALB
    P3 -->|HTTPS| ALB
    ALB -->|HTTP| GW
    CONF -.->|"Map"| GW
    GW <-->|"1. Busca OMEGA_SESSION<br/>[< 1ms]"| REDIS
    GW -->|"2. Rutea si branchId=null"| OMEGA
    GW -->|"2. branchId=200"| MS1
    GW -->|"2. branchId=300"| MS2
    GW -.->|"Emite Spans Asíncronos"| DDAPM

    style ALB fill:#8C4FFF,stroke:#fff,color:#fff
    style GW fill:#00ADD8,stroke:#fff,color:#fff
    style REDIS fill:#DC382D,stroke:#fff,color:#fff
    style OMEGA fill:#ED7100,stroke:#fff,color:#fff
    style MS1 fill:#ED7100,stroke:#fff,color:#fff
    style MS2 fill:#ED7100,stroke:#fff,color:#fff
    style CONF fill:#F0E68C,stroke:#333,color:#333
    style DDAPM fill:#632CA6,stroke:#fff,color:#fff
```

### Diagrama de Secuencia — Resolución de Sesión In-Memory (Redis)

```mermaid
sequenceDiagram
    participant P as Proveedor
    participant ALB as AWS ALB
    participant GW as Go API Gateway
    participant REDIS as Redis (ElastiCache)
    participant MS as ms-pagos

    P->>ALB: GET /api/pagos (Cookie: OMEGA_SESSION=x8y9z)
    Note right of P: No envía branchId explícito
    ALB->>GW: Forward request
    
    Note over GW: Lee Cookie: OMEGA_SESSION<br/>Necesita convertir sesión a branchId

    GW->>REDIS: GET session:x8y9z
    REDIS-->>GW: Result: "200" (Sub-milisegundo)
    
    Note over GW: branchId resuelto: 200<br/>Inyecta Header "branchId: 200"<br/>Rutea a MS_PAGOS

    GW->>MS: Forward modificado a ms-pagos
    MS-->>GW: 200 OK
    GW-->>ALB: 200 OK
    ALB-->>P: 200 OK
```

---

## Diagrama Comparativo Revisado — Flujo General Completo

```mermaid
graph TB
    PRV["Proveedores Externos<br/>(URL única)"]
    
    subgraph "Opción A"
        A_ALB["ALB + Listener Rules"]
    end
    
    subgraph "Opción B 🥈"
        B_ALB["ALB"] --> B_GW["Spring Cloud Gateway<br/>(Fargate)"]
    end
    
    subgraph "Opción G 👑"
        G_ALB["ALB"] --> G_GW["Go Gateway + Redis<br/>(Fargate)"]
    end
    
    subgraph "Opción D"
        D_APIGW["API Gateway"] --> D_VPC["VPC Link"]
    end
    
    DEST["Omega / Microservicios"]
    
    PRV --> A_ALB --> DEST
    PRV --> B_ALB
    B_GW --> DEST
    PRV --> G_ALB
    G_GW --> DEST
    PRV --> D_APIGW
    D_VPC --> DEST

    style G_GW fill:#00ADD8,stroke:#fff,color:#fff
    style B_GW fill:#1A73E8,stroke:#fff,color:#fff
    style PRV fill:#232F3E,stroke:#fff,color:#fff
    style DEST fill:#ED7100,stroke:#fff,color:#fff
```

---

## Diagrama de Testing: Mocks

```mermaid
graph LR
    subgraph "Mocks Proveedores - Clientes"
        MP1["Mock Proveedor 1<br/>GET, POST, PUT, DELETE<br/>Con branchId"]
        MP2["Mock Proveedor 2<br/>Sesión de Redis"]
        MP3["Mock K6<br/>(Spike Test: 1000 VU)"]
    end

    subgraph "Sistema bajo Test"
        ALB["ALB/Localhost"] --> GW["bw_omega-gateway<br/>(Go 1.24)"]
        GW <--> RD["Mock Redis"]
    end

    subgraph "Mocks Backend"
        MO["Mock Omega"]
        MMS["Mock Microservicio"]
    end

    MP1 -->|HTTP| ALB
    MP2 -->|HTTP| ALB
    MP3 -->|"Alto volumen"| ALB
    GW -->|"branchId conocido"| MMS
    GW -->|"Default / Fallback"| MO

    MO -.->|"Responde con request"| GW
    MMS -.->|"Responde con request"| GW

    style GW fill:#00ADD8,stroke:#fff,color:#fff
    style RD fill:#DC382D,stroke:#fff,color:#fff
    style MO fill:#ED7100,stroke:#fff,color:#fff
    style MMS fill:#ED7100,stroke:#fff,color:#fff
```
