# Hito Alcanzado y Handover a Operaciones (Cloud/SRE)

El código fuente, arquitectura (Gateway Reactivo), pruebas de validación automatizadas y abstracciones conceptuales (Módulos Terraform compartidos) del API **bw_omega-gateway** están formalmente validadas y documentadas en este repositorio. Llegado este punto, el SDLC a nivel desarrollo culmina.

Para efectuar la puesta en marcha de esta solución en producción, el equipo de Infraestructura y Operaciones (SRE/DevOps) debe llevar a cabo las siguientes 4 tareas puramente nativas de Cloud relacionadas con facturación y segregación de cuentas AWS:

## 1. Inyectar Secretos de Integración CI/CD (GitHub)
El Pipeline de despliegue automatizado `.github/workflows/cicd.yml` ya está operativo. No obstante, para permitirle subir la imagen de Docker a AWS y renovar el Fargate, SRE debe configurar a nivel Repositorio (o nivel Organización) en GitHub los siguientes Secretos/Variables:
*   `secrets.AWS_ROLE_ARN`: El ID del Rol IAM (configurado mediante un Identity Provider OIDC de Github a AWS) que posea permisos de push sobre Amazon ECR y `UpdateService` sobre Amazon ECS.

## 2. Iniciar Infraestructura Base por primera vez
Navegar hacia el directorio del Gateway elegido (`poc/opcion-b-spring-gateway/terraform`) e inicializar localmente (de forma validada) el backend del state de Terraform que construirá los clústeres reales.
```bash
terraform init
terraform plan
terraform apply -var="datadog_api_key=TU_ACCOUNT_REAL" -var="environment=production"
```
**Este paso aprovisionará automatizadamente:**
- La VPC y las subredes.
- El ALB (Balanceador Público).
- El WAF Regional (Firewall).
- El Clúster Fargate de Servicios de Spring Webflux (Sidecars de Datadog incluidos).
- El clúster ElastiCache de Redis (Requerido para el Rate Limiting distribuido de la Opción B).

## 3. Emisión y Binding de Certificado TLS (HTTPS)
Previo al Go-Live, SRE es responsable de asociar un Certificado SSL/TLS emitido por *AWS Certificate Manager (ACM)* directamente dentro del Target HTTPS del ALB, a modo de cubrir bajo un CNAME canónico el punto de entrada para los proveedores (e.g. `gateway.miplataforma.com`).
Esto debe matchearse pasando el flag `-var="certificate_arn=..."` en la inicialización de Terraform descrita en el paso 2.

## 4. Estrategia de Rollout y Cambio de Ruteo
Una vez comprobado que el ALB devuelve `200 OK` (Heartbeat) de los contenedores Spring Netty y que los *Dashboards as Code* de Datadog APM reflejan métricas en cero, SRE debe enviar un Release Note a los proveedores indicando que cambien sus llamadas a Omega Monolito hardcodeadas por la nueva URL base (CNAME del paso 3). 
El Gateway (gracias al Patrón *Strangler Fig*) absorberá transparentemente todo el tráfico e ingresará a la validación empírica.
