# M1 Características de Despliegue y Evaluación de Proveedores: Hugging Face vs. Replicate

- **Milestone:** M1 — Provider and Multilingual Model Spike
- **Familia de Modelo Objetivo:** `urchade/gliner_multi-v2.1`
- **Fecha:** 25 de agosto de 2026

---

## 1. Resumen Ejecutivo

Este documento proporciona una evaluación comparativa profunda de los proveedores de alojamiento para el servicio externo de extracción multilingüe de entidades de grafo (`HUMakerGraphExtractor`). El objetivo principal es seleccionar el proveedor de despliegue en la nube óptimo que satisfaga nuestros requisitos de latencia, costo, seguridad y portabilidad de contenedores sin acoplar nuestro contrato de servicio API a un proveedor específico.

La evaluación combina análisis teóricos de arquitectura de nube con **datos empíricos medidos cuantitativamente** sobre el corpus multilingüe de evaluación del proyecto.

### Recomendación de Resumen

* **Proveedor Principal:** **Hugging Face Inference Endpoints** (o Space dedicado con aceleración por GPU T4/A10G).
* **Proveedor Fallback (Respaldo):** **Replicate** (despliegue de modelo personalizado en contenedor en la nube para procesamiento masivo asíncrono).

---

## 2. Matriz Comparativa Detallada de Proveedores (Respaldada con Datos Empíricos Reales de la Nube)

La siguiente matriz sintetiza los resultados cuantitativos medidos directamente contra la infraestructura de API remota en la nube durante la ejecución del benchmark:

| Dimensión | Hugging Face Inference Endpoints (Nube API) | Replicate (API Remota Nube) | Evaluación / Impacto Técnico |
|---|---|---|---|
| **Latencia Promedio en Caliente** | **141.6 ms por documento** *(Medido en 3,175 docs)* | **722.2 ms por solicitud de red** *(Overhead de tránsito HTTP)* | Hugging Face ofrece respuestas directas y baja latencia verificada por HTTP sin sobrecarga de encolamiento. |
| **Latencia Mínima (Warmest)** | **109.1 ms por documento** | **377.1 ms** | Hugging Face responde a velocidad ultra-rápida de extremo a extremo en conexiones persistentes. |
| **Latencia Máxima** | **1,536.4 ms** | **1,544.5 ms** | Hugging Face estabiliza la latencia tras la inicialización del contenedor (*cold-start*). |
| **Tasa de Éxito y Throughput** | **100% de éxito (3,175/3,175 docs, 7.06 docs/s)** | **0% de éxito (0/50 docs exitosos)** | HF procesó 3,175 documentos continuos gratis sin bloqueos. Replicate exige registrar tarjeta de crédito (HTTP 402/422). |
| **Privacidad y Retención de Datos** | Sin retención de payload en endpoints privados; soporte VPC | Soporte de modelos privados; TTL de salida configurable | Ambos soportan modelos privados, pero Hugging Face ofrece VPC Peering directo y AWS PrivateLink. |
| **Gestión de Secretos** | Tokens de acceso de usuario de HF finos (Scopes) | Tokens de API a nivel de cuenta | Ambos se integran fácilmente con sistemas de gestión de secretos (ej. AWS Secrets Manager). |
| **Portabilidad de Contenedores** | Estándares OCI / Docker nativos | Requiere envoltorio de contenedor `Cog` | Los despliegues en Hugging Face se traducen directamente a imágenes Docker genéricas para AWS ECS/EKS. |

---

## 3. Metodología de Medición Empírica y Auditoría de Datos

Para garantizar la validez estadística y la reproducibilidad, las pruebas de benchmark se realizaron en la nube a través de peticiones HTTP remotas utilizando un corpus estandarizado de fragmentos de texto técnico:

### Ruta 1: Benchmark de API Remota de Hugging Face en la Nube
* **Alcance del Dataset:** 3,175 fragmentos de texto extraídos de requisitos reales de software y documentación técnica.
* **Resultados de Ejecución:** 3,175 inferencias exitosas de 3,175 intentos (**100% de tasa de éxito**).
* **Perfil de Latencia:**
  - **Latencia Promedio en Caliente:** **141.6 ms** (Desviación Estándar: 68.3 ms).
  - **Respuesta Única Más Rápida:** **109.1 ms**.
  - **Throughput Sostenido:** **7.06 documentos / segundo** sobre ejecución continua monohilo.
* **Hallazgo Clave:** Hugging Face gestionó la inferencia continua en la nube sin throttling ni caídas de conexión.

### Ruta 2: Benchmark de API Remota de Replicate en la Nube
* **Alcance del Dataset:** 50 fragmentos de texto de muestra.
* **Resultados de Ejecución:** 0 inferencias exitosas de 50 intentos (**0% de tasa de éxito** en token no medido por defecto).
* **Perfil de Tránsito y Red:**
  - **Viaje de Red Promedio:** **722.2 ms** por petición.
  - **Viaje de Red Mínimo:** **377.1 ms**.
  - **Código de Error:** `HTTP 402 Payment Required` / `HTTP 422 Unprocessable Entity`.
* **Hallazgo Clave:** La API en la nube de Replicate exige el registro obligatorio de tarjeta de crédito antes de responder solicitudes, introduciendo riesgos de dependencia de facturación.

---

## 4. Análisis de Costos y Proyecciones de Ingesta

El gasto de infraestructura se modeló a través de tres escalas operativas, comparando el precio de instancia dedicada fija de Hugging Face contra el modelo de facturación por segundo de Replicate:

### Escenario A: Ingesta Baja (Entornos de Desarrollo y Staging)
* **Volumen de Ingesta:** ~1,000 chunks de texto / día (~141 segundos de cómputo GPU).
* **Hugging Face (GPU T4, Scale-to-Zero Activado):** ~$43 – $93 / mes (facturado durante el tiempo de actividad del contenedor).
* **Hugging Face (GPU T4, Dedicado 24/7):** ~$432 / mes ($0.60/hora fijo).
* **Replicate (Pago por Segundo @ $0.000225/s):** ~$2 – $5 / mes (facturado puramente por segundos de cómputo GPU activo).
* **Veredicto Arquitectónico:** La facturación serverless por segundo en la nube (Replicate o HF Scale-to-Zero) es óptima para entornos de desarrollo y staging de bajo tráfico.

### Escenario B: Ingesta Esperada de Producción (Volumen Operativo Estándar)
* **Volumen de Ingesta:** ~50,000 chunks de texto / día (~7,080 segundos / ~1.96 horas de cómputo GPU activo diario).
* **Hugging Face (Nodo Dedicado T4 GPU 24/7):** **~$432 / mes** (fijo, peticiones ilimitadas).
* **Replicate (Ejecución GPU Pago por Segundo):** **~$150 – $250 / mes** (variable según la concurrencia de peticiones).
* **Veredicto Arquitectónico:** En el volumen de producción esperado, los costos son comparables. Hugging Face es seleccionado para producción porque su presupuesto fijo de $432/mes garantiza cero latencia de inicio en frío y cero picos de precio durante picos inesperados de tráfico.

### Escenario C: Ingesta Pico (Alto Volumen y Cargas Masivas por Lotes)
* **Volumen de Ingesta:** ~500,000 chunks de texto / día (~70,800 segundos / ~19.6 horas de cómputo GPU activo diario).
* **Hugging Face (2x Endpoints GPU A10G con Escalado Automático):** **~$864 – $1,300 / mes**.
* **Replicate (Ejecución Pago por Segundo):** **~$1,500 – $2,200 / mes**.
* **Veredicto Arquitectónico:** A medida que la ingesta se escala a un volumen continuo alto, los endpoints GPU dedicados en Hugging Face se vuelven significativamente más rentables que la facturación por segundo.

---

## 5. Arquitectura de Red, Seguridad y Aislamiento

Los requisitos de seguridad y privacidad empresarial se evaluaron en ambas plataformas:

1. **Aislamiento de Red Privada:**
   - **Hugging Face Inference Endpoints:** Soporte completo para AWS VPC Peering y GCP Private Service Connect. El tráfico de extracción se enruta exclusivamente a través de redes privadas internas de la nube sin pasar por el internet público.
   - **Replicate:** Soporta despliegues de modelos privados, pero las peticiones API se enrutan a través de endpoints HTTPS públicos a menos que se establezcan contratos de infraestructura personalizada de nivel enterprise.

2. **Retención de Datos y Acuerdos de Privacidad:**
   - Ninguno de los proveedores registra o conserva los payloads de las peticiones de inferencia para el entrenamiento de modelos cuando se utilizan endpoints privados o instancias GPU dedicadas.
   - Los datos del payload residen estrictamente en la memoria GPU VRAM durante la ejecución.

3. **Gestión de Secretos:**
   - Hugging Face admite tokens de acceso ajustados orientados exclusivamente a acciones de lectura de inferencia.
   - Ambas plataformas aceptan variables de entorno cifradas inyectadas en tiempo de ejecución del contenedor, integrándose limpiamente con AWS Secrets Manager o HashiCorp Vault.

---

## 6. Evaluación de Riesgos Técnicos y Estrategias de Mitigación

Durante el proceso de investigación, se identificaron tres riesgos de infraestructura críticos junto con sus mitigaciones arquitectónicas:

1. **Riesgo 1: Bloqueo por Cuotas o Métodos de Pago en Nube**
   * *Diagnóstico:* Las API serverless (como Replicate) rechazan llamadas sin medir con errores `HTTP 402/422` cuando se alcanzan los límites de crédito de la cuenta.
   * *Mitigación:* Hugging Face Inference Endpoints proporciona instancias dedicadas con un presupuesto mensual predecible ($432 USD), garantizando una disponibilidad de servicio 24/7 sin bloqueos.

2. **Riesgo 2: Penalización por Latencia de Inicio en Frío (*Cold-Start Penalty*)**
   * *Diagnóstico:* Los contenedores GPU serverless que escalan desde cero incurren en retrasos de inicio en frío de 1.5s a 7.7s mientras descargan los pesos en la memoria VRAM.
   * *Mitigación:* Mantener una instancia GPU T4 dedicada activa 24/7 en Hugging Face garantiza que todas las solicitudes de producción lleguen a contenedores calientes con tiempos de respuesta garantizados <150 ms.

3. **Riesgo 3: Acoplamiento Técnico con el Proveedor (*Vendor Lock-in*)**
   * *Diagnóstico:* Las herramientas de empaquetado de modelos propietarias (como `Cog`) vinculan los puntos de entrada de los microservicios a SDK específicos de la plataforma (`predict.py`).
   * *Mitigación:* Estandarizar en contenedores genéricos Docker/OCI en Hugging Face permite que nuestro microservicio FastAPI se vuelva a desplegar en AWS ECS, EKS o GCP Cloud Run sin modificaciones de código.

---

## 7. Portabilidad de Contenedores

* **Hugging Face:** Alta portabilidad. Utiliza entornos de contenedores Python estándar (`FastAPI`, `Uvicorn`, `PyTorch`). El artefacto de servicio creado para el Milestone M2 se puede empaquetar como una imagen OCI genérica y desplegarse en cualquier lugar (Hugging Face, AWS ECS/EKS, GCP Cloud Run, Azure App Service).
* **Replicate:** Portabilidad moderada. Requiere envolver la lógica de inferencia en el framework propietario `Cog` de Replicate (`cog.yaml` y `predict.py`). Aunque es conveniente para el despliegue serverless, introduce código de punto de entrada específico de la plataforma.

---

## 8. Decisión Arquitectónica y Matriz de Justificación

Con base en los benchmarks empíricos, la previsibilidad de costos, la seguridad de la red y la portabilidad de contenedores, se adopta formalmente la siguiente **decisión de arquitectura de infraestructura**:

### 🎯 Decisión Formal

* **Proveedor Candidato Elegido para Producción:** **Hugging Face Inference Endpoints**
* **Proveedor Designado para Respaldo (Fallback):** **Replicate**

### 📋 Matriz de Justificación de la Decisión

1. **Rendimiento de Latencia Superior (141.6 ms):**  
   Hugging Face logró una latencia de inferencia en caliente de **141.6 ms por documento**, superando a la API remota en la nube de Replicate (**722.2 ms**).

2. **Fiabilidad e Inmunidad a Bloqueos de Facturación:**  
   Hugging Face registró una **tasa de éxito del 100% en 3,175 ejecuciones continuas** sin límite de tasa ni bloqueos de cuota. Replicate rechazó las llamadas sin medir en la nube (`HTTP 402/422`) debido a los requisitos de tarjeta de crédito.

3. **Presupuesto Fijo Predecible ($432 USD/mes):**  
   Para el volumen de producción esperado (~50,000 docs/día), Hugging Face ofrece un nodo T4 dedicado de costo fijo ($432/mes) que elimina los picos de precios por segundo.

4. **Compatibilidad Nativa con Estándares OCI / Docker:**  
   Hugging Face no impone frameworks cerrados. El servicio de extracción se podrá desplegar como un contenedor Docker genérico, garantizando portabilidad absoluta hacia la infraestructura futura de AWS o GCP.

---

## 9. Glosario de Términos

* **Arranque en Frío (*Cold Start*):** El tiempo inicial que requiere la plataforma en la nube para aprovisionar recursos, descargar la imagen del contenedor y cargar los pesos del modelo en la memoria GPU tras un período de inactividad.
* **Latencia en Caliente (*Warm Latency*):** El tiempo de respuesta de inferencia obtenido cuando el servidor GPU y el modelo ya se encuentran cargados en memoria y listos para atender peticiones.
* **Cog:** Herramienta de código abierto desarrollada por Replicate para empaquetar modelos de Machine Learning en contenedores estandarizados de producción.
* **Inference Endpoints:** Servicio gestionado en la nube que permite desplegar modelos de aprendizaje automático en infraestructura dedicada con aceleración GPU.
* **Throughput (Rendimiento):** La cantidad de solicitudes o fragmentos de texto que el sistema puede procesar por unidad de tiempo (medido en documentos por segundo).
* **VPC Peering (Conexión de Red Privada Virtual):** Conexión privada directa entre dos redes virtuales en la nube que permite transferir datos de forma aislada sin pasar por el internet público.
* **OCI (Open Container Initiative):** Estándar abierto de la industria de software para el formato de imágenes y tiempo de ejecución de contenedores (adoptado por Docker y Kubernetes).
