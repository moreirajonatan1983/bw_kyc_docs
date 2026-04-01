# Estándares de Repositorios y Seguridad en GitHub

Este documento establece la normativa corporativa para la creación, estructuración y aseguramiento de repositorios en GitHub para cualquier nuevo producto o servicio, tomando como modelo de éxito el proyecto `bw_omega-gateway`.

---

## 1. Estructura de Repositorios por Producto

Para mantener la separación de responsabilidades, aislar el código productivo de las pruebas exploratorias y mantener la documentación auditable; todo producto nuevo debe dividirse de forma modular y asilada orientada a dominios:

### 📦 Estructura Modelo (Ejemplo: `bw_omega-gateway`)

1. **`[nombre_producto]` (Repositorio Core / Productivo)**
   - **Ejemplo:** `bw_omega-gateway`
   - **Propósito:** Contiene únicamente el código fuente productivo (ej. Go, Java), los Dockerfiles, scripts de CI/CD para producción (`.github/workflows`) y la infraestructura IaC (Terraform) estrictamente necesaria para su despliegue.
   - **Regla de Oro:** Cero código "basura", mocks o pruebas de concepto.

2. **`[nombre_producto]-docs` (Documentación Oficial)**
   - **Ejemplo:** `bw_omega-gateway-docs`
   - **Propósito:** Almacena la arquitectura final, ADRs (Decision Records), configuración de ambientes, diagramas Mermaid/Draw.png y manuales operativos organizados en un formato listo para Confluence.
   - **Regla de Oro:** Documentación viva y versionada. Cualquier cambio en la arquitectura requiere un Pull Request (PR) y aprobación aquí.

3. **`[nombre_producto]-qa` (Calidad y Testing Avanzado)**
   - **Ejemplo:** `bw_omega-gateway-qa`
   - **Propósito:** Centraliza los *Mocks* de servicios dependientes, scripts de estrés / performance (Grafana K6), simuladores en bash/node y colecciones corporativas de Postman.
   - **Regla de Oro:** Se aísla por completo del repositorio Core para evitar inflar las imágenes Docker de producción o mezclar dependencias de node u otras librerías inseguras en el runtime productivo.

4. **`[nombre_producto]-poc` (Investigación y Pruebas de Concepto)**
   - **Ejemplo:** `bw_omega-gateway-poc`
   - **Propósito:** Sandbox temporal donde la Gerencia de Ingeniería evalúa múltiples alternativas tecnológicas (ej. AWS ALB vs Spring Cloud vs Go Gateway) contrastando latencia y costos.
   - **Regla de Oro:** Una vez que la validación finaliza, este repositorio queda archivado (*Archived*) como historial auditivo. Por política nunca se despliega en ambientes estables.

---

## 2. Configuración de Seguridad en GitHub

Para cumplir con las políticas institucionales de cumplimiento (Auditoría, SOC2, FinOps, Zero Trust), todos los repositorios Core deben configurarse bajo el siguiente estándar estricto desde la pestaña **Settings** (Ajustes):

### 🛡️ 2.1 Branch Protection Rules (Aseguramiento de Ramas)

Nadie aprueba o pushea código directamente a Producción. Debes ir a `Settings > Branches` y crear una regla para `main` (y `develop` si hubiese):

- **Require a pull request before merging:** ✅ Activado.
  - *Require approvals:* Al menos `1` o más aprobaciones requeridas.
  - *Dismiss stale pull request approvals when new commits are pushed:* ✅ Activado. (Evita que el código modificado de último momento evada la revisión anterior).
- **Require status checks to pass before merging:** ✅ Activado.
  - *Require branches to be up to date before merging:* ✅ Activado.
  - **Checks Mandatorios a Listar:** Añadir al listado de chequeos vitales los workflows correspondientes (ej. `go test`, `build-image`, `trivy-scan`, `k6-smoke`). Si fallan, el código no avanza.
- **Do not allow bypassing the above settings:** ✅ Activado, para forzar que ni siquiera los Administradores de Infraestructura puedan puentear este escudo.

### 🔐 2.2 Security & Code Analysis (Escaneo Automático)

Desde `Settings > Code security and analysis`, la organización exige los siguientes motores de escaneo:

- **Dependabot Alerts:** ✅ Activado. Asegura el monitoreo activo de vulnerabilidades CVSS en `go.mod`, `pom.xml` o `package.json`.
- **Dependabot Security Updates:** ✅ Activado. Ante un bug de día cero, Dependabot generará de oficio un Pull Request actualizando automáticamente la versión de la gema/librería parcheada.
- **Secret Scanning:** ✅ Activado. Previene que credenciales maestras formen parte del historial permanente de Git.
  - *Push protection:* ✅ Activado. Es esencial habilitar el bloqueo en Push. Si el desarrollador hace `git push` con una llave de acceso de AWS dentro del código, GitHub rechazará localmente el comando en la terminal.

### ☁️ 2.3 Políticas de Integración con Nube (Defensa OIDC)

Está **terminantemente prohibido** el uso de *IAM Users* de larga duración (evitar configurar en los Secretos de repositorio pares como `AWS_ACCESS_KEY_ID` y `AWS_SECRET_ACCESS_KEY`).

- **El Standard Obligatorio:** Implementar y utilizar **OpenID Connect (OIDC)** entre GitHub Actions y los Proveedores de Nube.
- **Método:** GitHub se comunicará asumiendo un **IAM Role** de corta vida tras canjear temporalmente el token criptográfico (thumbprint) del repositorio.
- **Beneficio Principal:** En caso de hackeo global sobre GitHub o cuentas robadas, no hay tokens maestras en duro para exfiltrar. Se asegura vida máxima de 1 hora por ejecución.

### 🐳 2.4 Auditoría de Contenedores en CI/CD

El pipeline (`.github/workflows/deploy.yml`) de todo producto contenerizado debe incluir una fase de Control de Acceso (Quality Gate) para la imagen Docker finalizada antes del push al Registry:

- **Motor Recomendado (Trivy o Snyk):** Se debe detener la construcción con estado de fracaso (`exit code 1`) si los escáneres detectan anomalías de severidad **CRITICAL** o **HIGH** del lado del sistema operativo Linux nativo (por ejemplo vulnerabilidades no arregladas en base-images de Alpine/Debian).