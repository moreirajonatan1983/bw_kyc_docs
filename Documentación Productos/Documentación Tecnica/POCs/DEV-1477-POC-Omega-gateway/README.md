# Ticket DEV-1477: Omega Gateway POC & Documentation

Este directorio contiene toda la documentación generada durante el diseño, pruebas y desarrollo del nuevo **bw_omega-gateway**.

## Estructura de Documentación

### 1. `poc_docs/` (Pruebas de Concepto)
Contiene la investigación original y las opciones evaluadas (A a H) antes de decidir la arquitectura final.
- `01_propuesta_gateway.md`: Opciones y métricas base.
- `02_decisiones_arquitectura.md`: ADRs iniciales.
- `03_analisis_costos.md`: Costos en AWS vs on-prem.
- `05_plan_pruebas.md` y `06_escenarios_validacion.md`: Planificación de los tests.
- `07_resultados_pruebas_locales.md`: Resultados de K6 para la Opción G (Golang).
- `04_diagramas/`: Diagramas de draw.io y mermaid originales.

### 2. `gateway_docs/` (Arquitectura Final y Deploy)
Contiene la documentación final del proyecto `bw_omega-gateway` productivo.
- `01_contexto_y_arquitectura.md`: Explicación del sistema y los diagramas de arquitectura actualizados (incluye Mermaid y PNG visual).
- `02_decisiones_arquitectura.md`: ADRs consolidados para producción.
- `03_configuracion_ambientes.md`: Variables y configuración.
- `04_ruteo_y_reglas.md`: Algoritmos de enrutamiento y GitOps.
- `05_resultados_pruebas.md`: Carga (1K req/s) reportada y exitosa.
- `06_deploy_produccion.md`: Instrucciones para ECS Fargate y CI/CD.
- `assets/architecture_diagram.png`: Exportado visual de la arquitectura AWS.
