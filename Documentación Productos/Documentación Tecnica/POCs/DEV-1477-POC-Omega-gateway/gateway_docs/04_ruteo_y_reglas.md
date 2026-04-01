# Ruteo y Reglas de Enrutamiento

## Lógica de Ruteo

El gateway evalúa cada request entrante con el siguiente algoritmo:

```
1. Leer header: branchId
2. Si branchId vacío:
   a. Leer cookie: OMEGA_SESSION
   b. Consultar Redis: GET session:{OMEGA_SESSION}
   c. Si existe → usar branchId del caché
   d. Si no existe → branchId = "" (fallback a Omega)
3. Iterar sobre routes[] en orden:
   a. ¿branchId coincide? Y ¿path tiene el prefijo?
   b. Si ambas → enviar al destination_url de esa regla
4. Si ninguna regla coincide → enviar a fallback_url (Omega)
```

## Tabla de Rutas por Ambiente

### Fase 1 (Pasamanos total — Todo a Omega)

```yaml
routes: []  # Sin reglas específicas
fallback_url: "${OMEGA_HOST}"
fallback_name: "OMEGA"
```

### Fase 2 (Ruteo por branchId)

| branchId | Path Prefix | Destino | Servicio |
|---|---|---|---|
| `200` | `/api/pagos` | `${PAGOS_HOST}` | ms-pagos |
| `300` | `/api/usuarios` | `${USUARIOS_HOST}` | ms-usuarios |
| cualquier otro | cualquier path | `${OMEGA_HOST}` | Omega (fallback) |

### Fase 3 (Resolución de sesión — Pendiente)

> ⚠️ **ADVERTENCIA:**
> La lógica de derivación de `branchId` desde `OMEGA_SESSION` (Fase 3) está pendiente de definición de negocio. Actualmente el gateway consulta Redis pero la clave de sesión debe ser definida por el equipo de backend de Omega.

## Agregar una Nueva Ruta

1. Editar el archivo correspondiente en `configs/routes.[env].yaml`:

```yaml
routes:
  - branch_id: "400"          # branchId del proveedor
    path_prefix: "/api/apuestas"  # Prefijo de path que activa la regla
    destination_url: "${APUESTAS_HOST:-http://ms-apuestas:8080}"
    name: "MS_APUESTAS"        # Nombre para logs y trazas Datadog
```

2. Crear un PR con el cambio.
3. El pipeline CI/CD valida con `go test` y despliega automáticamente.
4. El nuevo contenedor en Fargate lee el archivo actualizado al arrancar.

> 💡 **CONSEJO:**
> No es necesario recompilar el binario Go para agregar rutas. El archivo `routes.yaml` es externo al binario y se provee en el contenedor vía `COPY configs/ /configs/` en el Dockerfile.

## Decisiones de Ruteo

| Escenario | Resultado |
|---|---|
| `branchId=200` + path `/api/pagos/estado` | → ms-pagos |
| `branchId=200` + path `/api/home` | → Omega (path no coincide) |
| `branchId=999` + cualquier path | → Omega (branchId no mapeado) |
| Sin `branchId`, sin sesión en Redis | → Omega (fallback) |
| Sin `branchId`, sesión en Redis con `branchId=300` | → ms-usuarios (si path coincide) |
