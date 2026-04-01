# Escenarios de Validación de Arquitectura (Gateway)

Para asegurar que la implementación seleccionada pueda soportar la transición a microservicios en un entorno de alta demanda (como el evento del Mundial para Omega), deben ejecutarse y validarse los siguientes escenarios en un entorno pre-productivo (Staging/Load Test).

---

## 1. Escenarios de Carga y Concurrencia (Stress)
**Objetivo:** Asegurar que el gateway pueda soportar picos masivos de tráfico generados por eventos deportivos, sin degradar el pasamanos a Omega.

| ID | Escenario | Procedimiento de Validación | Criterio de Éxito (Pass) |
|:---|:---|:---|:---|
| **E1.1** | **Pico de Tráfico Repentino** | Inyectar 5,000 requests/sec sostenidos usando Vegeta o JMeter (simulando inicio de partido). | El Gateway no debe rechazar conexiones (`Connection Refused`). Latencia P99 del Gateway no debe exceder los 50ms por sobre la latencia natural de Omega. |
| **E1.2** | **Conexiones Longevas (Keep-Alive)** | Levantar 10,000 conexiones concurrentes y enviar data escasa a bajo Rate (Slowloris simulado benigno). | El Gateway no agota el límite de descriptores de archivos (`too many open files`) y escala horizontalmente (Fargate CPU). |

---

## 2. Escenarios de Resiliencia y Circuit Breaking
**Objetivo:** Probar qué ocurre con el cliente y el gateway cuando un backend (Omega o un microservicio) colapsa, para evitar falla en cascada.

| ID | Escenario | Procedimiento de Validación | Criterio de Éxito (Pass) |
|:---|:---|:---|:---|
| **E2.1** | **Backend Threshold (Timeout)** | Simular que Omega responde en 45 segundos utilizando el mock. Enviar tráfico moderado. | El Gateway interrumpe la conexión tras 30s devolviendo `504 Gateway Timeout` formateado en JSON. No se agotan los hilos del Gateway. |
| **E2.2** | **Backend Caído (Connection Refused)**| Apagar el mock de un microservicio destino (ej. ms-pagos) y enviar tráfico hacia él. | El Circuit Breaker (Opción B) entra en estado `OPEN` después del umbral de fallos, y retorna instantáneamente un `503 Service Unavailable` por su Fallback, sin sobrecargar la red interna. |
| **E2.3** | **Zero-Downtime Deployment** | Mientras se inyectan 500 RPS constantes, forzar un redespliegue de la tarea ECS Fargate del Gateway. | `0%` de errores 502/504 en los clientes. Las conexiones activas terminan amablemente (Graceful Shutdown en Spring o NGINX Worker drains) mientras las task nuevas asumen tráfico. |

---

## 3. Escenarios de Ruteo Condicional Lógico
**Objetivo:** Verificar la precisión de las reglas cuando se enciende paulatinamente un microservicio nuevo (Canary Release / Strangler Fig).

| ID | Escenario | Procedimiento de Validación | Criterio de Éxito (Pass) |
|:---|:---|:---|:---|
| **E3.1** | **Inexistencia de BranchId** | Enviar 100 requests sin el header `branchId`. | Todos los requests impactan y son atendidos exclusivamente por el monolito (Omega). |
| **E3.2** | **BranchId Derivado de la Sesión** | (A futuro) Configurar el Gateway para resolver el branch a través de JWT o Redis en el vuelo y comparar performance. | Ruteo efectivo sin sumar penalizaciones >20ms por evaluación de sesión. |

---

## 4. Escenarios de Seguridad
**Objetivo:** Comprobar defensas tempranas antes del backend.

| ID | Escenario | Procedimiento de Validación | Criterio de Éxito (Pass) |
|:---|:---|:---|:---|
| **E4.1** | **Ataque de Cabeceras (Header Overflow)**| Enviar requests con payloads masivos en el header (>16KB). | El gateway rechaza directamente el request con `431 Request Header Fields Too Large`. |
| **E4.2** | **CORS / Métodos No Permitidos** | Ejecutar un llamado preflight (`OPTIONS`) originado desde un dominio no configurado. | Se bloquea en el gateway (respuesta 403 o no se incluyen los headers `Access-Control-Allow-Origin`). El request nunca llega a Omega. |

---

## 5. Escenarios de Observabilidad Total (Datadog)
**Objetivo:** El equipo de operaciones debe poder rastrear el 100% de la vida del request.

| ID | Escenario | Procedimiento de Validación | Criterio de Éxito (Pass) |
|:---|:---|:---|:---|
| **E5.1** | **Trazabilidad Distribuida (Trace ID)** | Generar 10 transacciones con un `X-Correlation-ID` custom inyectado. | En Datadog APM, la traza muestra un único flujo continuo (`ALB -> Gateway -> Omega`), el tiempo relativo consumido en el Gateway, y la correlación con los AWS CloudWatch logs a través del `aws.trace_id`. |
| **E5.2** | **Logs Parametrizados** | Verificar los logs almacenados y exportados por NGINX/Spring a Datadog. | Todos los logs de entrada del gateway incluyen inyectados obligatoriamente los Tags `branchId`, `status_code` y `latency_ms` para dashboarding de métricas de negocio. |
