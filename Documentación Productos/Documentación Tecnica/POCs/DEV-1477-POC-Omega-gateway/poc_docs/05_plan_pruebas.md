# Plan de Pruebas — bw_omega-gateway

## 1. Tipos de Pruebas Requeridas

### 1.1 Pruebas Funcionales
Validan que el gateway cumple su función de pasamanos transparente.

| # | Escenario | Qué se valida | Herramienta |
|---|---|---|---|
| F01 | GET con branchId en header | Request llega intacto a Omega | curl / test_functional.sh |
| F02 | POST con body JSON | Body no se modifica ni trunca | curl |
| F03 | PUT con headers custom | Headers del proveedor se preservan | curl |
| F04 | DELETE | Método HTTP se preserva | curl |
| F05 | Query params complejos | Params llegan intactos | curl |
| F06 | Request sin branchId | Fallback a Omega funciona | curl |
| F07 | Body grande (~10KB+) | No hay truncamiento | curl |
| F08 | Content-Type variados | form-urlencoded, multipart, text/xml | curl |
| F09 | Headers de trazabilidad | X-Amzn-Trace-Id se preserva | curl |
| F10 | Todos los branchIds existentes | Cada branch rutea correctamente | curl loop |

**Script:** `poc/shared/mock-providers/test_functional.sh`

---

### 1.2 Pruebas de Estrés / Carga

Validan el comportamiento bajo alto tráfico (simulando eventos deportivos como el Mundial).

| # | Escenario | Concurrencia | Duración | Qué se mide |
|---|---|---|---|---|
| S01 | Tráfico normal (GET) | 50 conexiones | 30s | Requests/sec, latencia p50/p95/p99 |
| S02 | Tráfico normal (POST con body) | 50 conexiones | 30s | Throughput, errores |
| S03 | Pico de evento deportivo | 100+ conexiones | 30s | Saturación, recovery time |
| S04 | Branches simultáneos | 10 branches x 5 conn c/u | 30s | Distribución correcta |
| S05 | Rampa progresiva | 10→100→200 | 60s | Punto de quiebre |
| S06 | Sostenido (soak test) | 50 conexiones | 300s (5min) | Memory leaks, degradación |

**Script:** `poc/shared/mock-providers/test_stress.sh`
**Herramientas:** `wrk` (recomendado), `ab` (alternativa), `k6` (avanzado)

**Instalación:**
```bash
# macOS
brew install wrk
# o
brew install httpd  # incluye ab
```

---

### 1.3 Pruebas de Seguridad (Pentest)

| # | Escenario | Qué se valida | Herramienta |
|---|---|---|---|
| P01 | Inyección de headers maliciosos | Gateway no ejecuta ni interpreta headers peligrosos | curl + payloads |
| P02 | Request con body oversized (>50MB) | Gateway rechaza o maneja correctamente | curl |
| P03 | Slowloris (conexiones lentas) | Gateway no se bloquea ante conexiones deliberadamente lentas | slowloris.py |
| P04 | Header injection | No se puede inyectar headers HTTP extra via branchId | curl |
| P05 | Path traversal | `/../../../etc/passwd` no expone archivos del gateway | curl |
| P06 | HTTP smuggling | Request no se puede "partir" para engañar al backend | curl |
| P07 | TLS/SSL scan (producción) | Verificar cifrados y protocolos seguros | sslyze / testssl.sh |
| P08 | Rate limiting bypass | Si hay rate limiting, validar que no se puede saltar | curl loop |
| P09 | Validación de IP de origen | ¿El gateway expone la IP del proveedor original a Omega? | curl + logs |
| P10 | Cookies/sesión manipulation | Cookies no se pueden modificar en tránsito | curl |

**Script sugerido (básico):**
```bash
# P01: Header injection
curl -s -X GET "http://localhost:8080/api/test" \
  -H "branchId: 100\r\nX-Injected: malicious"

# P02: Body oversized
dd if=/dev/zero bs=1M count=100 | curl -s -X POST "http://localhost:8080/api/test" \
  -H "Content-Type: application/octet-stream" --data-binary @-

# P05: Path traversal
curl -s "http://localhost:8080/../../../etc/passwd"
curl -s "http://localhost:8080/%2e%2e/%2e%2e/etc/passwd"
```

---

### 1.4 Pruebas de Resiliencia

| # | Escenario | Qué se valida | Cómo ejecutar |
|---|---|---|---|
| R01 | Omega no disponible | Gateway responde 502/503 con error claro | `docker stop mock-omega` |
| R02 | Omega lento (timeout) | Gateway timeout configurable, no se bloquea | Agregar delay al mock |
| R03 | Reinicio del gateway | Requests no se pierden (si hay buffer) | `docker restart gateway` |
| R04 | Omega se recupera | Gateway retoma ruteo automáticamente | `docker start mock-omega` |
| R05 | DNS failure | ¿Qué pasa si el hostname de Omega no se resuelve? | Cambiar URI a inexistente |

---

### 1.5 Pruebas de Observabilidad

| # | Escenario | Qué se valida |
|---|---|---|
| O01 | Logs contienen branchId | Cada request logueado incluye el branchId |
| O02 | Logs contienen dd.trace_id | Trazabilidad Datadog funciona |
| O03 | Métricas de latencia | Se pueden consultar métricas del gateway (Actuator para opción B) |
| O04 | Errores logueados | Los errores 5xx se loguean con detalle suficiente |
| O05 | Dashboard local | Se puede armar un dashboard básico con los logs |

---

## 2. Matriz: Pruebas por Opción

| Prueba | A: ALB Sim. | B: Spring GW | C: NGINX | D: API GW | E: CloudFront |
|---|---|---|---|---|---|
| **Funcionales** | ✅ Local | ✅ Local | ✅ Local | ⚠️ Solo AWS | ❌ Descartada |
| **Estrés** | ✅ Local | ✅ Local | ✅ Local | ⚠️ Solo AWS | ❌ |
| **Pentest** | ✅ Local | ✅ Local | ✅ Local | ⚠️ Solo AWS | ❌ |
| **Resiliencia** | ✅ Local | ✅ Local | ✅ Local | ⚠️ Solo AWS | ❌ |
| **Observabilidad** | ⚠️ Logs NGINX | ✅ Actuator + DD | ⚠️ Logs NGINX | ⚠️ CloudWatch | ❌ |

---

## 3. Cómo Ejecutar las Pruebas

### Paso 1: Elegir una opción y levantarla
```bash
cd poc/opcion-b-spring-gateway
./scripts/start_all.sh
```

### Paso 2: Ejecutar pruebas funcionales
```bash
cd poc/shared/mock-providers
chmod +x test_functional.sh
./test_functional.sh http://localhost:8080
```

### Paso 3: Ejecutar pruebas de estrés
```bash
chmod +x test_stress.sh
./test_stress.sh http://localhost:8080 30 50
# Parámetros: URL, duración en segundos, concurrencia
```

### Paso 4: Detener servicios
```bash
cd poc/opcion-b-spring-gateway
./scripts/stop_all.sh
```

### Paso 5: Repetir con otra opción
```bash
cd poc/opcion-c-nginx
./scripts/start_all.sh
# Ejecutar las mismas pruebas y comparar resultados
```

---

## 4. Métricas a Comparar entre Opciones

| Métrica | Cómo medirla |
|---|---|
| **Requests/segundo** | Output de wrk/ab |
| **Latencia p50** | Output de wrk/ab |
| **Latencia p99** | Output de wrk/ab |
| **Uso de memoria** | `docker stats` durante la prueba |
| **Uso de CPU** | `docker stats` durante la prueba |
| **Tiempo de arranque** | Tiempo desde `start_all.sh` hasta health OK |
| **Tamaño de imagen Docker** | `docker images` |
| **Errores bajo carga** | Output de wrk/ab (non-2xx responses) |
| **Recovery tras caída de Omega** | Tiempo medido en prueba R04 |
