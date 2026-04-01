# Análisis de Opciones: bw_omega-gateway — Proxy/Gateway para Migración a Microservicios

## 1. Contexto del Problema

### 1.1 ¿Qué es Omega?

**Omega** es una plataforma de **juegos online (betting)** que actualmente opera como un monolito. Sus servicios son consumidos por múltiples proveedores externos a través de una **URL única**. Se planifica una migración progresiva hacia una arquitectura de **microservicios**, lo cual implicaría que cada servicio tenga su propia URL.

### 1.2 Consideraciones de Tráfico

> ⚠️ **ADVERTENCIA:**
> Omega recibe **alto volumen de tráfico** en condiciones normales, y este caudal **se multiplica significativamente** durante eventos deportivos de alto perfil (Mundial de Fútbol, Finales de Champions, Super Bowl, etc.). El gateway debe estar diseñado para soportar estos picos sin degradar la experiencia.

Implicaciones para el diseño del gateway:
- **Latencia mínima:** Cada milisegundo cuenta en una plataforma de betting en tiempo real.
- **Autoescalado predecible:** Capacidad de escalar horizontalmente antes de los eventos.
- **Alta disponibilidad:** El gateway será un punto único de fallo si no se diseña correctamente.
- **Zero downtime deployments:** No se puede interrumpir el servicio para desplegar nuevas reglas de ruteo.

### 1.3 El Problema

**Problema:** Los proveedores no deben verse forzados a cambiar sus integraciones ni rutear hacia múltiples URLs nuevas.

**Solución:** Interponer un componente de tipo **Proxy/Gateway** que:
1. **Fase 1 (Pasamanos):** Reciba todo el tráfico en la URL original y lo reenvíe de forma **100% transparente** a Omega (sin cambios para nadie). El gateway debe actuar como un espejo: la única diferencia es la URL del servicio destino. Todo lo demás (headers, body, query params, cookies, método HTTP) debe pasar intacto.
2. **Fase 2 (Ruteo inteligente):** Progresivamente, identifique las peticiones por `branchId` y las dirija al microservicio correspondiente o a Omega si ese servicio aún no fue migrado.


### 1.4 Clarificación Conceptual: ¿API Gateway o Reverse Proxy?

A nivel de arquitectura de diseño empresarial, es fundamental delinear qué patrón estamos implementando con "bw_omega-gateway":

*   **¿Qué NO estamos construyendo? (Reverse Proxy Puro):** Un *Reverse Proxy* clásico (como NGINX o Envoy Server crudo) es una capa agnóstica de transporte. Recibe una conexión HTTP y ciegamente balancea o rutea hacia backends basándose solamente en la URL o el puerto (e.g., `/api/*` va al Servidor 2). **No conoce reglas de negocio ni lee el estado de la sesión de los usuarios.**
*   **¿Qué SÍ estamos construyendo? (Un API Gateway Institucional):** El sistema diseñado es el orquestador inteligente (*Smart Endpoint*) del dominio de Betting. Se denomina **API Gateway** porque asume responsabilidades transaccionales, de seguridad y telemetría complejas:
    1.  **Transformación y Resolución Transaccional:** Captura *Cookies de Sesión* pasivas, cruza información en tiempo real con una Base **In-Memory Redis** (ElastiCache) para inferir el proveedor (`branchId`), inyectando proactivamente el Header antes de enviarlo al microservicio interno.
    2.  **Lógica Condicional (Business Routing):** La derivación `/api/pagos` hacia `MS_PAGOS` no rige puramente sobre el path HTTP, sino que obliga a una condicionalidad estricta (ej. `branchId === 200`). De fallar, implementa *Fallback Routing* orgánico hacia Omega.
    3.  **Cross-Cutting Concerns:** Integra Datadog APM nativo inyectando *Spans/Traces* a cada request y orquesta el modelado de observabilidad general.

### 1.5 Lógica de Ruteo por `branchId`

El ruteo se basa en el parámetro `branchId`, que identifica la sucursal/operación del proveedor:

```
Caso 1: branchId presente en el Header HTTP
┌─────────────────────────────────────────────────┐
│ Request llega con Header: branchId=100          │
│ → Gateway lee el header directamente            │
│ → Resuelve destino según tabla de ruteo         │
│ → Forwadea al servicio correspondiente          │
└─────────────────────────────────────────────────┘

Caso 2: branchId AUSENTE en el Header
┌─────────────────────────────────────────────────┐
│ Request llega SIN header branchId               │
│ → Gateway inspecciona datos de sesión           │
│ → Deriva el branchId a partir de la sesión (*)  │
│ → Resuelve destino según tabla de ruteo         │
│ → Forwadea al servicio correspondiente          │
└─────────────────────────────────────────────────┘
(*) La lógica para derivar branchId desde la sesión aún no está definida.
```

### 1.5 URLs de Ruteo (Ejemplo)

Las URLs de destino aún no están definidas. A continuación se incluyen 10 URLs de ejemplo que representan el esquema esperado:

| # | branchId (ejemplo) | Servicio Destino | URL de Ejemplo |
|---|---|---|---|
| 1 | 100 | Omega (Monolito) | `http://omega-monolith.internal:8080` |
| 2 | 200 | ms-pagos | `http://ms-pagos.internal:8080` |
| 3 | 300 | ms-usuarios | `http://ms-usuarios.internal:8080` |
| 4 | 400 | ms-apuestas | `http://ms-apuestas.internal:8080` |
| 5 | 500 | ms-eventos | `http://ms-eventos.internal:8080` |
| 6 | 600 | ms-reportes | `http://ms-reportes.internal:8080` |
| 7 | 700 | ms-notificaciones | `http://ms-notificaciones.internal:8080` |
| 8 | 800 | ms-liquidaciones | `http://ms-liquidaciones.internal:8080` |
| 9 | 900 | ms-riesgo | `http://ms-riesgo.internal:8080` |
| 10 | 1000 | ms-backoffice | `http://ms-backoffice.internal:8080` |

> ℹ️ **NOTA:**
> En la Fase 1, **todos los branchId** se rutean a Omega (monolito). La tabla se activa progresivamente en Fase 2/3.

### 1.6 Principio Fundamental: Pasamanos Transparente

> ❗ **IMPORTANTE:**
> El gateway **NO debe modificar la invocación** en ningún aspecto excepto la URL de destino. Todo el contenido del request original (método HTTP, headers, body, query params, cookies, content-type) debe llegar **intacto** a Omega o al microservicio destino. El gateway es un espejo con ruteo.

### 1.7 Requisitos de Observabilidad

La aplicación **debe contar con Datadog** para monitoreo:
- APM (Application Performance Monitoring) para trazas de cada request.
- Métricas de latencia, throughput y errores del gateway.
- Correlación con AWS CloudWatch (solución ya implementada).
- Dashboards de tráfico por `branchId` y por proveedor.
- Alertas durante picos de eventos deportivos.

### 1.8 Restricciones Técnicas

- Despliegue en **AWS** (Fargate como opción principal).
- Infraestructura gestionada con **Terraform**.
- Compatibilidad con el ecosistema existente (Datadog, CloudWatch, ALB).
- Capacidad de soportar picos de tráfico durante eventos deportivos.

### 1.9 Estrategia de Mitigación y Despliegue en Producción

Dado que hoy Omega recibe todas las invocaciones directo, la interposición de este Gateway está diseñada para mitigar totalmente cualquier riesgo de interrupción de servicio hacia los Proveedores Externos:

1. **Pasamanos Transparente (Reverse Proxy Puro):** En la Fase 1 de despliegue, el Gateway opera en modo "Shadow". Cualquier request que recibe (incluyendo Cabeceras, Body, Parámetros y Cookies) es replicada y forwadeada de manera idéntica hacia Omega. Para el monolito actual, el tráfico lucirá exactamente igual a interactuar con los proveedores de entrada.
2. **Corte y Transición por DNS (Route53):** Para el Go-Live, no es necesario orquestar ventanas de mantenimiento con los Proveedores de Betting. La puesta en producción se realizará de manera invisible mediante un **DNS Cutover** en AWS. Se actualizará la resolución del dominio base (e.g. `api.omega.com`) para que deje de apuntar al ALB actual de Omega, y comience a apuntar al nuevo ALB del Gateway.
3. **Rollback de Emergencia Inmediato (< 3 segundos):** Si tras desviar el tráfico hacia el Gateway se detectan anomalías graves, degradación de latencia o fallos volumétricos imprevistos, el plan de contingencia es *Trivial y Seguro*: Se revierte el registro DNS en AWS para enrutar el tráfico nuevamente de forma directa a Omega, extrayendo el Gateway del medio en cuestión de segundos.

---

## 2. Opciones Evaluadas

### Opción A: AWS ALB + Listener Rules (Solo Infra, sin código)

| Aspecto | Detalle |
|---|---|
| **Descripción** | Usar el Application Load Balancer de AWS con reglas de enrutamiento nativas basadas en path, headers o query strings. No requiere escribir código de aplicación. |
| **Fase 1 (Pasamanos)** | Una regla default que manda todo al Target Group del monolito. Trivial. |
| **Fase 2 (Ruteo)** | Se agregan Listener Rules en Terraform que matchean por path (`/api/v2/pagos/*` → microservicio Pagos) o por header/query (`branchId`). |
| **Ventajas** | ✅ Cero código de aplicación. ✅ Escalado y alta disponibilidad nativa. ✅ Costo mínimo (sin contenedores extra). ✅ Latencia mínima (capa de red pura). |
| **Desventajas** | ❌ Ruteo limitado a path, header, query string y host. No puede leer el body ni datos de sesión. ❌ Máximo ~100 reglas por listener. ❌ Lógica de decisión binaria (no soporta lógica condicional compleja como "si branchId=X Y el usuario tiene rol Y"). |
| **Cuándo elegirla** | Si el routeo puede resolverse **exclusivamente** por path o headers HTTP simples sin leer el cuerpo ni la sesión. |

---

### Opción B: Spring Cloud Gateway (Aplicación Java en Fargate)

| Aspecto | Detalle |
|---|---|
| **Descripción** | Proyecto Java basado en **Spring Cloud Gateway** (reactivo, Netty) desplegado como contenedor en ECS Fargate. Framework nativo del ecosistema Spring para construir API Gateways con filtros y predicados programáticos. |
| **Fase 1 (Pasamanos)** | Una ruta `default` con un `uri: lb://monolito` que proxea todo. Se configura en YAML o en código. |
| **Fase 2 (Ruteo)** | Predicados custom en Java que leen `branchId` desde query params, headers o cookies de sesión y deciden el destino. Filtros para transformar headers, agregar autenticación, etc. |
| **Ventajas** | ✅ Ecosistema Spring nativo (mismo stack que el monolito existente). ✅ Filtros y predicados programáticos: permite lógica compleja (leer sesión, consultar DB/cache, etc). ✅ Integración directa con Datadog Java Agent (misma solución de trazabilidad que ya tenemos). ✅ Service Discovery con Spring Cloud si se necesita. ✅ Circuit breaker integrado (Resilience4j). |
| **Desventajas** | ❌ Es una aplicación Java completa: requiere mantenimiento, updates de dependencias, JVM. ❌ Cold start de la JVM (~5-15s, mitigable con warm pools). ❌ Consumo de memoria base de la JVM (~256-512MB mínimo). |
| **Cuándo elegirla** | Si el equipo ya trabaja con Spring Boot y necesita lógica de ruteo compleja que dependa de datos de sesión o lógica de negocio. |

**Ejemplo de ruta en YAML:**
```yaml
spring:
  cloud:
    gateway:
      routes:
        # Fase 1: Todo al monolito
        - id: default-monolith
          uri: http://monolito-alb.internal:8080
          predicates:
            - Path=/**
          order: 9999

        # Fase 2: Ejemplo de ruteo por branchId
        - id: payments-service
          uri: http://pagos-service.internal:8080
          predicates:
            - Path=/api/pagos/**
            - Query=branchId, 100|200|300
          order: 1
```

---

### Opción C: NGINX / OpenResty como Reverse Proxy (Contenedor en Fargate)

| Aspecto | Detalle |
|---|---|
| **Descripción** | Un contenedor NGINX (o OpenResty para scripting Lua) configurado como reverse proxy. Ruteo basado en `location` blocks y variables de request. |
| **Fase 1 (Pasamanos)** | Un bloque `location / { proxy_pass http://monolito; }`. Extremadamente sencillo. |
| **Fase 2 (Ruteo)** | Se agregan `location` blocks con matching por path. Para lógica compleja (leer branchId del query string), se puede usar `map` de NGINX o scripts Lua con OpenResty. |
| **Ventajas** | ✅ Rendimiento excepcional (bajo consumo CPU/RAM). ✅ Imagen Docker ultra liviana (~20MB). ✅ Cold start prácticamente instantáneo. ✅ Ideal como pasamanos puro. |
| **Desventajas** | ❌ Lógica compleja requiere Lua (OpenResty), lenguaje menos familiar para equipos Java. ❌ Difícil leer datos de sesión (cookies firmadas, tokens JWT) sin plugins adicionales. ❌ No tiene integración nativa con Datadog Java Agent (necesita Datadog NGINX module aparte). ❌ Debugging y testing más artesanal. |
| **Cuándo elegirla** | Si la prioridad es rendimiento extremo y el ruteo se resuelve solo por path/headers sin lógica de negocio. |

---

### Opción D: AWS API Gateway (Servicio Gestionado)

| Aspecto | Detalle |
|---|---|
| **Descripción** | Usar AWS API Gateway (HTTP API o REST API) como punto de entrada gestionado, con integraciones hacia los targets (monolito o microservicios). |
| **Fase 1 (Pasamanos)** | Una integración HTTP proxy que forwardea todo al monolito. |
| **Fase 2 (Ruteo)** | Se definen rutas por path/method que apuntan a distintas integraciones (VPC Links hacia los microservicios en Fargate). |
| **Ventajas** | ✅ Cero infraestructura propia que mantener. ✅ Throttling, API Keys, y authorizers nativos. ✅ Escalado automático ilimitado. ✅ Integración nativa con CloudWatch y IAM. |
| **Desventajas** | ❌ Costo por request (puede ser significativo con alto tráfico de proveedores). ❌ Latencia adicional (~10-30ms por hop). ❌ Lógica de ruteo limitada (no lee body, sesión, ni hace lógica condicional compleja). ❌ Posibles límites de payload (10MB). ❌ Vendor lock-in más fuerte. |
| **Cuándo elegirla** | Si el volumen de requests es bajo/medio y se quiere delegar completamente la operación del gateway a AWS. |

---

### Opción E: AWS CloudFront + Lambda@Edge

| Aspecto | Detalle |
|---|---|
| **Descripción** | CloudFront como CDN/proxy con funciones Lambda@Edge que inspeccionan la request y deciden el origin (monolito o microservicio). |
| **Fase 1 (Pasamanos)** | Default origin apuntando al monolito. |
| **Fase 2 (Ruteo)** | Lambda@Edge en el evento `origin-request` que lee `branchId` y cambia el origin dinámicamente. |
| **Ventajas** | ✅ CDN + Proxy en uno. ✅ Distribución global. ✅ Caching integrado. |
| **Desventajas** | ❌ Lambda@Edge tiene limitaciones fuertes (5s timeout, 1MB response, solo Node.js/Python). ❌ Deploy lento (~15-20min por cambio). ❌ Debugging extremadamente difícil. ❌ No puede acceder a recursos de VPC directamente. ❌ Overengineering para este caso de uso. |
| **Cuándo elegirla** | Solo si necesitás caching global agresivo y distribución geográfica. No recomendada para este caso. |

---

## 3. Matriz Comparativa

| Criterio | A: ALB Rules | B: Spring Cloud GW | C: NGINX | D: API GW | E: CloudFront |
|---|---|---|---|---|---|
| **Complejidad Fase 1** | ⭐ Trivial | ⭐⭐ Baja | ⭐ Trivial | ⭐⭐ Baja | ⭐⭐⭐ Media |
| **Lógica Ruteo Compleja** | ❌ Limitada | ✅ Total | ⚠️ Con Lua | ❌ Limitada | ⚠️ Limitada |
| **Lectura de Sesión** | ❌ No | ✅ Sí | ⚠️ Difícil | ❌ No | ❌ No |
| **Ruteo por branchId** | ⚠️ Solo query/header | ✅ Query/Header/Body/Session | ⚠️ Query/Header | ⚠️ Solo path | ⚠️ Con Lambda |
| **Costo Operativo** | ⭐ Mínimo | ⭐⭐⭐ Medio | ⭐⭐ Bajo | ⭐⭐⭐ Variable | ⭐⭐ Bajo |
| **Latencia Adicional** | ⭐ Ninguna | ⭐⭐ Baja (~2-5ms) | ⭐ Mínima | ⭐⭐⭐ Media (~15ms) | ⭐⭐ Variable |
| **Stack Familiar** | ✅ Terraform puro | ✅ Java/Spring | ❌ NGINX/Lua | ⚠️ AWS Console/TF | ❌ Node.js |
| **Trazabilidad Datadog** | ⚠️ Solo ALB logs | ✅ Java Agent nativo | ⚠️ Módulo NGINX | ⚠️ CloudWatch | ❌ Difícil |
| **Mantenimiento** | ⭐ Nulo | ⭐⭐⭐ Medio | ⭐⭐ Bajo | ⭐ Nulo | ⭐⭐ Bajo |

### 3.1 Cumplimiento de Ruteo Estático (GitOps en Memoria - $0 / 0ms)

Se evaluó la capacidad de cada alternativa para satisfacer el patrón de configuración externalizada (evitando mapeos hardcodeados en el runtime y evitando llamadas de red extras a Redis por cada request HTTP):

| Propuesta | ¿Soporta Tabla de Ruteo Estática Externa? | Mecanismo de Implementación / Modificación | Cumplimiento | Limitante Técnico |
|---|---|---|---|---|
| **A: ALB Rules** | ✅ Sí | Se configura vía **Terraform IaC**. Las reglas se inyectan en AWS. | Pleno | Es ciego, no puede leer cookies de sesión ni consultar a Redis. |
| **B: Spring Boot** | ✅ Sí | Se configuró el archivo **`application.yml`** donde Spring lee las reglas. | Pleno | Ninguna. Soporta consultar a Redis si la cookie requiere traducción a branchId. |
| **C: NGINX** | ✅ Sí | Archivo de sistema **`nginx.conf`** con directiva `map`. | Pleno | Dificultad extrema para contactar a Redis en Sub-ms dentro de NGINX crudo. |
| **D: API Gateway** | ✅ Sí | Integración de AWS configurada vía **OpenAPI/Terraform**. | Pleno | No puede extraer `branchId` incrustado en payloads complejos sin mapping templates limitados. |
| **E: CloudFront** | ✅ Sí | Requiere embeber **`routes.json`** en el bundle de la Lambda@Edge. | Parcial | El despliegue a los CDN periféricos Edge tarda de 10 a 20 minutos por cambio de ruta. |
| **F: KrakenD** | ✅ Sí | Configuración declarativa mediante **`krakend.json`**. | Pleno | No cuenta con lógica condicional "IF" basada en branchId sin plugins Go injectados. |
| **G: Go (Elegida)** | ✅ Sí | Refactorizado para parsear **`routes.yaml`** a la memoria RAM. | Pleno | Absoluto ganador considerando velocidad P99 <1ms. |
| **H: Node.js** | ✅ Sí | Refactorizado para parsear **`routes.json`** vía `fs.readFileSync()`. | Pleno | Penalización menor en el GC del V8 Engine frente a Go. |

---

## 4. Recomendación Definitiva

> ❗ **IMPORTANTE:**
> **Opción G (Custom Go Reverse Proxy en Fargate)** es el candidato ganador definitivo, elegido por sobre la Opción B (Java Spring), complementado con **Opción A (ALB Rules)** como primera capa de Firewall.

### Justificación de la Elección de Go

1. **Rendimiento y Footprint inigualables:** Al compilarse estáticamente en una imagen `scratch/alpine` (15MB), la Opción G consume apenas **~20MB de RAM**, versus los ~400MB base que exige la JVM para arrancar Spring Webflux (Opción B). En un entorno *Serverless* como Fargate, donde se paga por CPU/RAM provisionada, esto significa una fracción del costo operativo.
2. **Cero Cold Starts (Resiliencia a Spikes):** Un evento deportivo de pico máximo puede requerir escalar contenedores rápido. Go levanta en **< 5 milisegundos**.
3. **Control de Lógica Compleja (El requisito central):** A diferencia de NGINX, Go nos permite programar en código `main.go` cualquier regla de negocio (como delegar autenticación, inyectar lógicas de ruteo por fallbacks vacíos, conectarnos a Redis Elasticache para Rate Limits, etc) tal cual lo haríamos en Java, pero de forma más asíncrona usando **Goroutines**.
4. **Trazabilidad Datadog y AWS WAF:** Go tiene una librería APM de Datadog oficial fantástica que insertamos fácilmente con `httptrace.WrapHandler`. Toda la capa de seguridad perimetral sigue delegada al WAF de AWS.
5. **La arquitectura definitiva propuesta:**
   - **AWS WAF** → Filtro DDoS y XSS perimetral.
   - **ALB** → recibe tráfico externo, SSL termination, health checks.
   - **Go Gateway (Fargate)** → recibe del ALB, decodifica el `branchId` u o los fallbacks a `Omega`.
   - **Omega / Microservicios (Fargate)** → destinos finales.

### Fases de Implementación
| Fase | Alcance | Configuración |
|---|---|---|
| **Fase 1** | Pasamanos total. Todo va a Omega (monolito). | Una ruta `/**` → Omega. El gateway es transparente y no modifica nada. |
| **Fase 2** | Ruteo por `branchId` (header) para servicios ya migrados. | Rutas por branchId (`branchId=200` → ms-pagos). Default sigue a Omega. |
| **Fase 3** | Resolución de `branchId` desde sesión + ruteo inteligente. | Filtro custom que detecta ausencia de branchId, consulta sesión, resuelve y rutea. |

---

## 5. Estrategia de Testing

Para validar el gateway sin depender de Omega ni de proveedores reales, se requieren **mocks**:

### 5.1 Mock de Omega (Backend)
Una aplicación simple que simule el comportamiento de Omega:
- Recibe requests en cualquier path/método.
- Valida que el request llega **intacto** (headers, body, query params, cookies).
- Responde con un JSON que incluye: el request recibido completo, timestamp, y un flag de éxito.
- Sirve para verificar que el gateway actúa como pasamanos transparente.

```json
// Response esperada del Mock Omega
{
  "mock": "omega",
  "received": {
    "method": "POST",
    "path": "/api/apuestas/crear",
    "headers": { "branchId": "100", "Content-Type": "application/json", "..." : "..." },
    "body": { "eventoId": 12345, "monto": 500 },
    "queryParams": { "divisa": "ARS" }
  },
  "timestamp": "2026-03-31T21:00:00Z",
  "integrity": "PASS"
}
```

### 5.2 Mock de Proveedores (Clientes)
Scripts o aplicaciones que simulen el comportamiento de los proveedores:
- Envían requests variados (GET, POST, PUT, DELETE) con distintos paths, bodies y headers.
- Algunos incluyen `branchId` en el header, otros no (para probar la resolución por sesión).
- Permiten ejecutar pruebas de carga para simular picos de eventos deportivos.
- Validan que la respuesta recibida es la esperada.

### 5.3 Escenarios de Testing

| # | Escenario | Resultado Esperado |
|---|---|---|
| 1 | Request con `branchId=100` en header | Rutea a Omega. Request llega intacto. |
| 2 | Request sin `branchId` en header | Resuelve branchId desde sesión, rutea correctamente. |
| 3 | Request POST con body JSON grande | Body llega intacto a Omega sin truncamiento. |
| 4 | Request con headers custom del proveedor | Todos los headers se preservan en Omega. |
| 5 | Request con query params complejos | Query params llegan intactos a Omega. |
| 6 | Request con cookies de sesión | Cookies se forwadean sin modificación. |
| 7 | Prueba de carga (simulación evento deportivo) | Gateway autoescala y mantiene latencia < 10ms. |
| 8 | Omega no disponible (timeout/error) | Gateway responde con error apropiado y loguea. |
| 9 | branchId desconocido | Fallback: rutea a Omega (monolito). |
| 10 | Request con múltiples content-types | Cada content-type se forwadea sin alteración. |

---

## 6. Decisiones

### Resueltas
- [x] **Nombre definitivo del proyecto:** `bw_omega-gateway`.
- [x] **¿Cómo viaja el `branchId`?** — Viaja como **header HTTP** en cada request. El gateway debe leer este header para tomar decisiones de ruteo.
- [x] **¿Qué pasa si no viene el `branchId`?** — El gateway debe inspeccionar los datos de sesión del request y derivar el `branchId` a partir de ellos. La lógica exacta de esta derivación aún no está definida.
- [x] **Observabilidad:** La aplicación debe contar con Datadog (APM + métricas + alertas) integrado.
- [x] **Principio de pasamanos:** El gateway NO modifica el request. Solo cambia la URL destino.

### Pendientes de Definición
- [ ] **Lógica de derivación de `branchId` desde sesión** — ¿Cómo se obtiene el branchId cuando no viene en el header? ¿De qué campo de la sesión? ¿Requiere consultar una DB o cache?
- [ ] **URLs reales de destino** — Se configuraron placeholders (`${MS_PAGOS_URL}`) pero falta llenarlos con los dominios internos de AWS.
- [ ] **¿Qué datos de sesión se necesitan para rutear?** — Token JWT, cookie de sesión, claims específicos. Se definirá cuando se implemente la Fase 3.
- [ ] **¿Cuántos proveedores/integraciones hay?** — Para dimensionar el Task de Fargate correctamente.
- [ ] **⚠️ ¿Omega tiene alguna validación de origen?** — ¿Omega valida desde dónde es invocada (IP whitelist, certificado mTLS, API key, header de origen)? Si es así, el gateway necesitará presentar las credenciales correctas para que Omega acepte los requests. **Esto es crítico y debe validarse antes de la implementación en Producción**.

### Resueltas Recientemente (Fase Productiva Opción B)
- [x] **Rate Limiting y Throttling:** Resuelto. Se implementó un algoritmo *Token Bucket* apoyado en un cluster nativo de **Amazon ElastiCache (Redis)**. Identifica a cada proveedor por su IP emisora (`remoteAddressKeyResolver`) para prevenir abusos.
- [x] **Seguridad Perimetral y Autenticación:** Se delega la capa de autorización bruta al **AWS WAF** (Core Rule Set para frenar inyecciones). El Gateway actúa puramente como ruteador y delegará los chequeos de negocio o tokens finos al microservicio subyacente o a Omega.
