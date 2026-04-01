# Configuración de Ambientes

El gateway lee la variable `APP_ENV` al iniciar y carga el archivo de rutas correspondiente desde `configs/`. Si no se define, usa `local` por defecto.

## Ambientes disponibles

| `APP_ENV` | Archivo de rutas | Descripción |
|---|---|---|
| `local` | `configs/routes.local.yaml` | Desarrollo local con mocks Docker |
| `dev` | `configs/routes.dev.yaml` | Instancias Alpha/Dev en AWS |
| `pre-prod` | `configs/routes.pre-prod.yaml` | Homologación con proveedores reales |
| `prod` | `configs/routes.prod.yaml` | Producción en AWS Fargate |

## Variables de Entorno

| Variable | Descripción | Default |
|---|---|---|
| `APP_ENV` | Ambiente activo | `local` |
| `REDIS_ADDR` | Dirección del cluster Redis | `localhost:6379` |
| `DD_AGENT_HOST` | Host del Datadog Agent (sidecar) | — |
| `DD_ENV` | Ambiente reportado en Datadog APM | Igual a `APP_ENV` |
| `DD_SERVICE` | Nombre del servicio en Datadog | `bw_omega-gateway` |
| `OMEGA_HOST` | URL base del monolito Omega | Variable en routes.yaml |
| `PAGOS_HOST` | URL del microservicio ms-pagos | Variable en routes.yaml |
| `USUARIOS_HOST` | URL del microservicio ms-usuarios | Variable en routes.yaml |

## Cómo levantar por ambiente

### Local (con mocks Docker)
```bash
# macOS / Linux
bash scripts/sh/start_all.sh
# Windows
.\scripts\windows\start_all.ps1
```

### Dev / Pre-prod / Prod (Fargate)
```bash
# El APP_ENV es inyectado por la Task Definition de ECS
# No se levanta manualmente — ver 06_deploy_produccion.md
```

## Estructura de `routes.yaml`

```yaml
routes:
  - branch_id: "200"
    path_prefix: "/api/pagos"
    destination_url: "${PAGOS_HOST:-http://mock-ms-pagos:8080}"
    name: "MS_PAGOS"

  - branch_id: "300"
    path_prefix: "/api/usuarios"
    destination_url: "${USUARIOS_HOST:-http://mock-ms-usuarios:8080}"
    name: "MS_USUARIOS"

fallback_url: "${OMEGA_HOST:-http://mock-omega:8080}"
fallback_name: "OMEGA"
```

> ℹ️ **NOTA:**
> Los valores `${VAR:-default}` son resueltos en runtime. En prod se inyectan via Task Definition de ECS.
