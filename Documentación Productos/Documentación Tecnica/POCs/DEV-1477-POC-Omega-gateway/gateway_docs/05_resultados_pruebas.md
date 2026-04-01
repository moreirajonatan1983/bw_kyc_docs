# Resultados de Pruebas

## Entorno de Pruebas

- **Gateway:** bw_omega-gateway (Go 1.24, Alpine ~20MB)
- **Host:** Local (Docker Desktop / ARM64)
- **Mocks:** `mock-omega`, `mock-ms-pagos`, `mock-ms-usuarios` (Node.js Express)
- **Herramienta de Carga:** Grafana K6 (via Docker fallback)
- **Fecha:** Abril 2026

---

## 1. Unit Tests (go test)

```text
=== RUN   TestGatewayRouterRouting
--- PASS: TestGatewayRouterRouting (0.00s)

=== RUN   TestRegressionRules
    --- PASS: Regla_Pagos_Valida (0.00s)
    --- PASS: Path_Pagos_pero_Branch_Diferente (0.00s)
    --- PASS: Branch_Pagos_pero_Path_Diferente (0.00s)
    --- PASS: Regla_Usuarios_Valida (0.00s)
    --- PASS: Fallback_a_Omega_(Branch_Vacio) (0.00s)
    --- PASS: Fallback_a_Omega_(Null) (0.00s)
    --- PASS: Preservacion_de_QueryParams (0.00s)

PASS  ok  github.com/bet/bw_omega-gateway/cmd/server  0.75s
```

**Resultado: 9/9 PASS ✅**

---

## 2. Smoke Tests

```text
============================================
  SMOKE TESTS — bw_omega-gateway
  Target: http://localhost:8085
============================================

  ✅ Health check responde (200)
  ✅ Actuator health (200)
  ✅ NGINX health (200)
  ✅ GET /api/ping con branchId (200)
  ✅ POST con body (200)
  ✅ Latencia < 500ms (2ms)

  ✅ SMOKE TESTS PASSED (6/6)
============================================
```

**Resultado: 6/6 PASS ✅ — Latencia: 2ms**

---

## 3. Tests Funcionales

32 casos probados cubriendo:
- GET, POST, PUT, DELETE con `branchId`
- Headers custom preservados (`X-Provider-Id`, `X-Correlation-Id`, `X-Amzn-Trace-Id`)
- Query params complejos (múltiples parámetros)
- Body grande (~10KB) sin truncamiento
- Content-Type `application/x-www-form-urlencoded`
- Requests sin `branchId` (fallback a Omega)
- 10 `branchId` distintos simultáneos

```text
  RESULTADOS: 32 passed / 0 failed / 32 total
```

**Resultado: 32/32 PASS ✅**

---

## 4. Estrés K6 (Spike Test — Evento Deportivo)

### Escenarios ejecutados

| Escenario | Configuración |
|---|---|
| `base_load` | 50 VUs constantes por 30s |
| `spike_test` | Rampa hasta 500 req/s en 40s |

### Resultados

```text
█ THRESHOLDS

  error_rate
  ✓ 'rate<0.01' rate=0.00%

  http_req_duration
  ✓ 'p(99)<200' p(99)=5.1ms

  routing_success_rate
  ✓ 'rate>0.99' rate=100.00%

█ TOTAL RESULTS

  checks_total.......: 90540   2257/s
  checks_succeeded...: 100.00% (90540/90540)
  checks_failed......: 0.00%   (0/90540)

  HTTP
  http_req_duration..: avg=1.21ms  min=305µs  med=828µs  p(99)=5.1ms  max=54ms
  http_req_failed....: 0.00%  (0/30180)
  http_reqs..........: 30180   752/s
```

### Análisis

| Métrica | Resultado | Threshold | Estado |
|---|---|---|---|
| P99 Latencia | **5.1ms** | < 200ms | ✅ |
| Error Rate | **0.00%** | < 1% | ✅ |
| Routing Success | **100%** | > 99% | ✅ |
| Throughput | **752 req/s** | — | ✅ |
| Total Requests | **30.180** | — | — |

> 💡 **CONSEJO:**
> El P99 de **5.1ms** con 50 VUs constantes + spike a 500 req/s confirma que el gateway no introduce cuellos de botella visibles. La latencia percibida por los proveedores es prácticamente la del backend destino.
