**# Workers / Servicios

## 4.3 Estrategia de Procesamiento Asíncrono (Worker Fleet Architecture)

Para garantizar el desacoplamiento, la escalabilidad granular y la tolerancia a fallos, todos los procesos de fondo del sistema DAM se implementan bajo el patrón Worker-Based Architecture orquestado por Temporal.

### 4.3.1 Modelo de Comunicación Unificado (Long-Polling gRPC)

Independientemente de su función (Transcodificación, IA, Distribución), todos los Workers operan bajo el mismo principio de comunicación:

1. Iniciativa del Worker (Pull Model): Los servicios no exponen puertos HTTP ni aceptan peticiones entrantes para ejecutar tareas. En su lugar, establecen una conexión gRPC persistente (Long-Polling) saliente hacia el Temporal Frontend Service.
2. Segregación por Task Queues: Cada tipo de worker escucha exclusivamente una Task Queue específica. Esto permite escalar los recursos de hardware independientemente (ej. escalar CPUs para video sin agregar memoria innecesaria para distribución).
3. Control de Flujo (Backpressure): Al utilizar un modelo de Pull, el sistema es naturalmente resistente a picos de carga. Si un worker está al 100% de capacidad, simplemente demora en solicitar la siguiente tarea. El estado pendiente persiste en el servidor de Temporal, evitando la pérdida de mensajes o la saturación de memoria en el worker.

### 4.3.2 Clasificación y Definición de Workers

Se definen cuatro tipos de workers especializados, cada uno con requisitos de infraestructura y perfiles de escalado distintos:

#### A. Ingest & Metadata Worker

* Cola:q-ingest
* Responsabilidad: Validación técnica de archivos (checksums), extracción de metadatos técnicos (mediante ffprobe, exiftool) y generación de identificadores únicos.
* Perfil de Carga:I/O Bound (Lectura) + CPU Light. Requiere alto ancho de banda de lectura contra Ceph, pero bajo consumo de CPU.
* Escalado: Basado en el número de nuevos archivos ingresados por minuto.

#### B. Transcode Worker (Ya definido)

* Cola:q-transcode
* Responsabilidad: Procesamiento pesado de imagen y video (FFmpeg/libvips).
* Perfil de Carga:CPU Bound / GPU Bound. Es el consumidor principal de recursos de cómputo.
* Escalado: Agresivo, basado en profundidad de cola. Utiliza nodos con CPUs de alta frecuencia o aceleración de hardware.

#### C. AI Enrichment Worker

* Cola:q-ai-enrich
* Responsabilidad: Ejecución de inferencia de modelos (Whisper para STT, YOLO para objetos).
* Perfil de Carga:GPU Bound / Latency Bound.
* Implementación:
  * Este worker actúa como un wrapper inteligente. Descarga el activo y ejecuta el modelo (si es local) o gestiona la llamada a un servicio de inferencia interno.
  * Nota de Diseño: Se separa de la transcodificación para no bloquear un nodo de GPU costoso mientras se espera una tarea de red.
* Escalado: Basado en la disponibilidad de GPUs y licencias de software.

#### D. Distribution Worker

* Cola:q-distribution
* Responsabilidad: Transferencia de archivos a puntos finales externos (YouTube, FTPs de TV, CDN Origins).
* Perfil de Carga:Network Bound (Salida). El cuello de botella es la latencia de red externa y el ancho de banda de salida. El consumo de CPU es despreciable.
* Concurrencia: A diferencia del Transcode Worker (que procesa 1 video a la vez por hilo), un solo Distribution Worker en Go puede gestionar cientos de subidas concurrentes (goroutines) ya que pasa la mayor parte del tiempo esperando I/O de red.
* Escalado: Moderado. Unos pocos pods pueden manejar miles de distribuciones.

### 4.3.3 Matriz de Configuración de Workers

Para formalizar la implementación en Kubernetes, se presenta la siguiente matriz de despliegue:

| Tipo de Worker | Lenguaje  | Task Queue     | Resource Requests (K8s)  | Concurrencia Típica   | Estrategia de Reintento                                   |
| -------------- | --------- | -------------- | ------------------------ | ---------------------- | --------------------------------------------------------- |
| Ingest         | Go        | q-ingest       | 0.5 CPU / 512MB RAM      | Alta (20-50 tareas)    | Limitada (Error si archivo corrupto)                      |
| Transcode      | Go + C    | q-transcode    | 2.0 CPU / 4GB RAM        | Baja (1 tarea por Pod) | Exponencial + Heartbeat                                   |
| AI Enrich      | Python/Go | q-ai-enrich    | 4.0 CPU / 16GB RAM + GPU | Baja (1 tarea por GPU) | Exponencial                                               |
| Distribution   | Go        | q-distribution | 0.2 CPU / 256MB RAM      | Muy Alta (100+ tareas) | Exponencial Largo (espera recuperación de APIs externas) |

---

### Resumen para tu comprensión (Mental Model):

Imaginá una oficina de correos (Temporal Server) con 4 ventanillas (Task Queues):

1. Ventanilla "Transcode": Hay unos fisicoculturistas (CPUs potentes) que se acercan, agarran un paquete pesado, se lo llevan a una mesa, tardan 1 hora en procesarlo y vuelven a avisar. (Polling lento, tarea larga).
2. Ventanilla "Distribución": Hay unos mensajeros en moto (Red rápida). Se acercan, agarran 50 cartas de golpe y salen a repartirlas. Vuelven rápido. (Polling rápido, mucha concurrencia).

Todos van a la ventanilla (Polling). La ventanilla nunca les tira el paquete por la cabeza (Push).

Esta sección unifica tu arquitectura y demuestra coherencia técnica en todo el informe. ¿Te parece bien así o querés profundizar en alguno en particular?

**
