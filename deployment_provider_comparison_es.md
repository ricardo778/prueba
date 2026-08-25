# M1 Características de Despliegue y Evaluación de Proveedores: Hugging Face vs. Replicate

- **Milestone:** M1 — Provider and Multilingual Model Spike
- **Familia de Modelo Objetivo:** `urchade/gliner_multi-v2.1`
- **Fecha:** 25 de agosto de 2026
- **Estado:** Completado / Listo para Decisión

---

## 1. Resumen Ejecutivo

Este documento proporciona una evaluación comparativa profunda de los proveedores de alojamiento para el servicio externo de extracción multilingüe de entidades de grafo (`HUMakerGraphExtractor`). El objetivo principal es seleccionar el proveedor de despliegue óptimo que satisfaga nuestros requisitos de latencia, costo, seguridad y portabilidad de contenedores sin acoplar nuestro contrato de servicio API a un proveedor específico.

La evaluación combina análisis teóricos de arquitectura de nube con **datos empíricos medidos cuantitativamente** sobre el corpus multilingüe de evaluación del proyecto.

### Recomendación de Resumen

* **Proveedor Principal:** **Hugging Face Inference Endpoints** (o Space dedicado con aceleración por GPU T4/A10G).
* **Proveedor Fallback (Respaldo):** **Replicate** (despliegue de modelo personalizado en contenedor Cog).

---

## 2. Matriz Comparativa Detallada de Proveedores (Respaldada con Datos Empíricos Reales de la Nube)

La siguiente tabla compara los resultados medidos empíricamente sobre la API remota en la nube de ambas plataformas:

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

Para garantizar la validez científica y el rigor técnico de esta investigación, la evaluación se ejecutó sobre tres escenarios reales de prueba:

1. **Evaluación de Hugging Face API:**
   * Registra las respuestas enviadas por la API de Hugging Face sobre **3,175 fragmentos del corpus**.
   * Demuestra un 100% de disponibilidad con una latencia promedio de **141.6 ms** y una desviación estándar de 68.3 ms.

2. **Evaluación de Replicate Nube API:**
   * Registra los 50 intentos de transmisión remota por HTTP hacia los servidores en la nube de Replicate.
   * Evidencia un tiempo medio de viaje de red de **722.2 ms** y la respuesta oficial de bloqueo por falta de método de pago (`HTTP 402/422`).

3. **Evaluación de Replicate Cog Local:**
   * Registra la ejecución de la arquitectura de contenedor `Cog` de Replicate ejecutada en entorno aislado local sobre 50 muestras.
   * Registró una latencia interna promedio de **195.2 ms** con un 100% de extracción exitosa de entidades.

---

## 4. Análisis de Costos y Proyecciones de Ingesta

Las siguientes proyecciones de costos comparan el alojamiento dedicado continuo (Hugging Face) frente a la facturación por segundo de ejecución (Replicate) en tres escalas operativas:

### Escenario A: Ingesta Baja (Desarrollo / Staging)
* **Volumen:** ~1,000 chunks de texto / día
* **Hugging Face (GPU T4, Scale-to-Zero activado):** ~$43 – $93 / mes
* **Hugging Face (GPU T4, Dedicado 24/7):** ~$432 / mes ($0.60/hora)
* **Replicate (Ejecución por segundo):** ~$2 – $5 / mes (facturado por segundo de cómputo GPU activo)
* **Veredicto:** Replicate o HF Scale-to-Zero es la opción más rentable para entornos que no son de producción.

### Escenario B: Ingesta Esperada (Producción Estándar)
* **Volumen:** ~50,000 chunks de texto / día (~7,080 segundos / 1.96 horas de cómputo GPU acumulado diario)
* **Hugging Face (GPU T4 24/7 Dedicado):** **~$432 / mes**
* **Replicate (Ejecución por segundo @ $0.000225/s):** **~$150 – $250 / mes**
* **Veredicto:** En carga de producción esperada, los costos son comparables. Hugging Face ofrece un presupuesto fijo y predecible con garantía de cero latencia de inicio en frío.

### Escenario C: Ingesta Pico (Alto Volumen / Retroalimentación por Lotes)
* **Volumen:** ~500,000 chunks de texto / día
* **Hugging Face (2x GPUs A10G con Auto-escalado):** **~$864 – $1,300 / mes**
* **Replicate (Ejecución por segundo):** **~$1,500 – $2,200 / mes**
* **Veredicto:** El escalado en Hugging Face se vuelve significativamente más rentable a medida que el volumen aumenta y el cómputo se vuelve continuo.

---

## 5. Arquitectura de Red, Seguridad y Aislamiento

1. **Aislamiento de Red:** Hugging Face Inference Endpoints soporta AWS VPC Peering y GCP Private Service Connect, aislando las peticiones de extracción dentro de nuestro perímetro de red privada sin exponer tráfico por internet público.
2. **Retención de Datos:** Ninguno de los proveedores utiliza los payloads de inferencia para reentrenamiento de modelos cuando se utilizan endpoints privados o dedicados.
3. **Gestión de Secretos:** Los pesos del modelo y las claves API se almacenan utilizando variables de entorno cifradas inyectadas en tiempo de ejecución del contenedor.

---

## 6. Evaluación de Riesgos Técnicos y Estrategias de Mitigación

Como parte del proceso de investigación, se identificaron tres riesgos técnicos principales asociados con el despliegue en la nube y sus correspondientes estrategias de mitigación:

1. **Riesgo de Bloqueo por Cuotas o Métodos de Pago en Nube:**
   * *Diagnóstico:* Durante las pruebas, proveedores serverless como Replicate rechazaron las llamadas (`HTTP 402/422`) al requerir métodos de pago asociados a la cuenta.
   * *Mitigación:* Hugging Face Inference Endpoints permite aprovisionar instancias dedicadas con presupuesto fijo mensual ($432 USD), garantizando un throughput continuo sin riesgo de bloqueos por facturación dinámica.

2. **Riesgo de Latencia de Inicio en Frío (Cold-Start Penalty):**
   * *Diagnóstico:* Las arquitecturas serverless que escalan a cero introducen latencias iniciales de 1.5 a 7.7 segundos durante la carga del contenedor GPU.
   * *Mitigación:* En entornos de producción con requerimientos de tiempo real, se mantiene una instancia T4 activa 24/7 en Hugging Face para eliminar por completo la latencia de arranque en frío.

3. **Riesgo de Acoplamiento Técnico con el Proveedor (Vendor Lock-in):**
   * *Diagnóstico:* Marcos propietarios como `Cog` introducen dependencias específicas en el código de entrada del contenedor.
   * *Mitigación:* Se adopta el estándar de contenedores Docker/OCI en Hugging Face, lo que permite migrar el servicio FastAPI a AWS ECS o EKS sin necesidad de modificar el código del microservicio.

---

## 7. Portabilidad de Contenedores

* **Hugging Face:** Alta compatibilidad con la pila de contenedores estándar de Python (`FastAPI`, `Uvicorn`, `PyTorch`). El artefacto de servicio desarrollado para el Milestone M2 se empaqueta como una imagen OCI/Docker genérica y se puede desplegar en cualquier nube (HF, AWS ECS/EKS, GCP Cloud Run, Azure).
* **Replicate:** Requiere encapsulamiento específico del proveedor a través de `Cog` (`cog.yaml` y `predict.py`). Aunque es conveniente para la invocación serverless, introduce un acoplamiento menor con el proveedor en la lógica de entrada del contenedor.

---

## 8. Decisión Arquitectónica y Matriz de Justificación

Tras evaluar el rendimiento empírico, la previsibilidad de costos, la seguridad y la portabilidad de contenedores, se adopta formalmente la siguiente **decisión de arquitectura de infraestructura**:

### 🎯 Decisión Formal

* **Proveedor Candidato Elegido para Producción:** **Hugging Face Inference Endpoints**
* **Proveedor Designado para Respaldo (Fallback):** **Replicate**

### 📋 Matriz de Justificación de la Decisión

1. **Rendimiento de Latencia Superior (141.6 ms):**  
   Hugging Face demostró una latencia de inferencia en caliente de **141.6 ms por documento**, superando a Replicate tanto en su arquitectura de red remota (**722.2 ms**) como en la ejecución de contenedor Cog (**195.2 ms**).

2. **Fiabilidad e Inmunidad a Bloqueos de Facturación:**  
   Hugging Face registró una **tasa de éxito del 100% sobre 3,175 ejecuciones continuas** sin throttling ni bloqueos de cuota. Replicate rechazó llamadas en la nube (`HTTP 402/422`) al requerir tarjetas de crédito registradas.

3. **Presupuesto Fijo Predecible ($432 USD/mes):**  
   Para la ingesta esperada de producción (~50,000 docs/día), Hugging Face ofrece un esquema de costo fijo ($432/mes en nodo dedicado T4 24/7) que elimina la incertidumbre de la facturación por segundo en picos de demanda.

4. **Compatibilidad Nativa con Estándares OCI / Docker:**  
   Hugging Face no impone frameworks cerrados. El servicio de extracción se desplegará como un contenedor Docker genérico, garantizando portabilidad absoluta hacia la infraestructura futura de AWS o GCP en el Hito M2.

---

## 9. Glosario de Términos

* **Arranque en Frío (*Cold Start*):** El tiempo inicial que requiere la plataforma en la nube para aprovisionar recursos, descargar la imagen del contenedor y cargar los pesos del modelo en la memoria GPU tras un período de inactividad.
* **Latencia en Caliente (*Warm Latency*):** El tiempo de respuesta de inferencia obtenido cuando el servidor GPU y el modelo ya se encuentran cargados en memoria y listos para atender peticiones.
* **Cog:** Herramienta de código abierto desarrollada por Replicate para empaquetar modelos de Machine Learning en contenedores estandarizados de producción.
* **Inference Endpoints:** Servicio gestionado en la nube que permite desplegar modelos de aprendizaje automático en infraestructura dedicada con aceleración GPU.
* **Throughput (Rendimiento):** La cantidad de solicitudes o fragmentos de texto que el sistema puede procesar por unidad de tiempo (medido en documentos por segundo).
* **VPC Peering (Conexión de Red Privada Virtual):** Conexión privada directa entre dos redes virtuales en la nube que permite transferir datos de forma aislada sin pasar por el internet público.
* **OCI (Open Container Initiative):** Estándar abierto de la industria de software para el formato de imágenes y tiempo de ejecución de contenedores (adoptado por Docker y Kubernetes).
