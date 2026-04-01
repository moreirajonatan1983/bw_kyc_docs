# Análisis Analítico de Costos vs Arquitecturas (AWS)

Para elegir el API Gateway correcto, la técnica y la performance deben cruzarse con los costos reales de Infraestructura en Nube. 
En una plataforma de apuestas (*Betting*), el tráfico se caracteriza por ser masivo (decenas de millones de requests en tiempo real por el *Polling* constante de *Odds*/Cuotas) y extremadamente picudo (ej: cuando hay un penalti o un gol).

El siguiente es un estimado de despliegue en la región AWS `us-east-1` (Alta Disponibilidad en 3 zonas, bajo una simulación de **100 Millones de Requests al mes**).

---

## 0. Prerrequisitos de Capacity Planning (Pregunta Abierta)

Antes de definir la cantidad final de réplicas base de contenedores o la agresividad de las reglas de **Autoescalado** en AWS (Target Tracking Scaling Policy) y los tamañon de In-Memory DBs, debemos contestar al problema de **Escalabilidad** midiendo la línea de tráfico orgánico y picos. 

> ❗ **IMPORTANTE:**
> **Definición de Tráfico Pendiente (Acción Requerida del Equipo Core):**
> ¿Cuántas invocaciones de los proveedores integrados y usuarios estamos estimando procesar habitualmente y durante eventos top? Necesitamos rellenar o mapear esta proyección volumétrica para dimensionar la subred VPC:

*   **RPS (Requests Per Second) Esperados:** ¿Tráfico Valle? ¿Tráfico Pico (ej. partido Boca-River o Final del Mundo)? (Ej: *200 / 5,000 RPS*).
*   **RPM (Requests Per Minute):** Crucial para setear las directivas del Troughput y Alarmas de CloudWatch.
*   **RPH / RPD (Requests Per Hour / Day):** Crucial para el dimensionamiento del Data Transfer de salida del AWS ALB perimetral y el Billing recurrente.
*   **Concurrencia Pura TCP:** ¿Cuántas conexiones simultáneas mantendremos abiertas si los endpoints hacen long-polling sobre los tickets de apuestas?

*Solo respondiendo a estos volúmenes empíricos garantizaremos que las métricas de Autoescalado disparen el aprovisionamiento de memoria RAM en el contenedor correcto sin incurrir en cuellos de botella.*

---

## 1. AWS API Gateway (SaaS / Opción D)
**Modelo de cobro:** Pay-per-Request (Serverless Puro).

AWS API Gateway es asombroso para startups, pero catastrófico para plataformas transaccionales de *polling/streaming* crudo debido a su *Billing Model*. Su precio base es de **$3.50 USD por cada Millón de invocaciones** (más transferencia de datos saliente).

*   **100.000.000 Requests:** ~ $350.00 USD/mes.
*   **Si el evento escala a 500M de Requests:** ~ $1,750.00 USD/mes.
*   **Contras Adicionales:** Cada invocación tiene "Cold Starts" o latencias intermitentes. Requeriría conectar a AWS ElastiCache a través de Lambdas que agregan $0.20 extra por millón de ejecuciones.
*   **Veredicto de Costo:** 🔴 **Inviable / Prohibitivo**. 

---

## 2. Java Spring Cloud Gateway en Fargate (Opción B)
**Modelo de cobro:** Pago por Capacidad Computacional Asignada (vCPU + RAM per Hour).

Al ser contenedores, no importa si procesan 10 o 100 millones de solicitudes al mes, se cobra el tiempo de encendido del hardware. Sin embargo, la JVM (Java Virtual Machine) es pesada. Para soportar picos y encendidos sin congelarse frente a la red *Netty*, requiere contenedores nutridos.

*   **Configuración mínima (por contenedor):** 1 vCPU / 2GB RAM = **$0.0407 / hora**.
*   **Alta Disponibilidad (3 contenedores estables):**
    *   3 x $0.0407 x 730 horas = **~$89.13 USD/mes**.
*   **Costo de Over-Provisioning:** Debido a que un contenedor Java WebFlux tarda entre 10 y 15 segundos en arrancar, para soportar un "Spike" explosivo de un gol de Messi, los DevOps deberán aprovisionar de más (*Stand by* pasivo). Un pull de 10 contenedores en reserva sumará **$300 USD/mes**.
*   **Veredicto de Costo:** 🟡 **Costoso, aunque lineal**. Aceptable, pero con un "Idle Cost" (costo desperdiciado en espera) alto.

---

## 3. Custom Go API Gateway en Fargate (Opción G) 👑
**Modelo de cobro:** Pago por Capacidad Computacional Asignada (vCPU + RAM per Hour).

Go compila su código a lenguaje de máquina nativo e incluye su propio *Event Loop* (Goroutines) hyper liviano. Por ende, puede soportar cientos de peticiones web usando una fracción de los recursos de Java.

*   **Configuración mínima sobrada (por contenedor):** 0.25 vCPU / 512MB RAM = **$0.01018 / hora**.
*   **Alta Disponibilidad (3 contenedores estables):**
    *   3 x $0.01018 x 730 horas = **~$22.29 USD/mes**.
*   **Costo de Autoescalado RealTime:** Como la imagen Docker en Go pesa solo `15MB` (Alpine/Scratch) y arranca en **< 5 milisegundos**, la orquestación ECS de AWS reacciona inflando y matando contenedores a demanda a la perfección sin necesidad de tener contenedores caros parados (*Overprovisioning nulo*). Autoescala tan perfecto como una Lambda, pero con el precio de un contenedor EC2.
*   **Conexión Redis:** A esto sumaríamos una instancia pequeña de AWS ElastiCache (`cache.t4g.micro`) de **~$16 USD/mes**. Total estimado: **$38 USD/mes estables.**
*   **Veredicto de Costo:** 🟢 **La Opción Más Barata y Performante**. Costos fijos ínfimos, escalado instantáneo y previsibilidad financiera abasoluta frente a un tráfico demencial.

---

### Cuadro Comparativo de Gastos Teóricos (Arquitectura Target Fargate)
| Arquitectura | Tipo de Costo | Escalamiento | Costo Base (Mes) | Costo a 500M Reqs/Mes |
| :--- | :--- | :--- | :--- | :--- |
| **Opción D (AWS API GW)** | **Por Invocación** | Difícil control | `$ ~350` | `$ 1,750.00+` |
| **Opción B (Java WebFlux)** | **Computacional Alto** | Lento (Over-Provisioning) | `$ ~100` | `$ ~150.00` |
| **Opción G (Golang Nativo)** | **Computacional Bajo**| Ultrarápido y Limpio | **`$ 38.00`** | **`$ 45.00`** |

> *Nota: Los costos descritos asumen la capa de procesamiento/computación intermedia. Ambos (Java y Go) requerirían el mismo ALB perimetral.*
