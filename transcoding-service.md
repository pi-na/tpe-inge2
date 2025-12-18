#### 4.2.1.8 Servicio de Transcodificación y Normalización de Medios (Transcode Workers)

Responsabilidad:

Este subsistema es el motor de cómputo intensivo de la plataforma. Su función es ejecutar la normalización de activos (ingesta) y la generación de derivados (renditions) de video, audio e imagen según los perfiles definidos por negocio y los requisitos de los canales de distribución.

Implementación Tecnológica:

Se ha diseñado una arquitectura de Workers Desacoplados implementados en Go (Golang), desplegados en contenedores Docker sobre Kubernetes y orquestados mediante Temporal Activities.

Stack de Procesamiento:

* Wrapper/Controlador: Go. Se selecciona por su excelente gestión de concurrencia y manejo eficiente de subprocesos del sistema (os/exec), vital para controlar binarios externos.
* Motor de Video/Audio:FFmpeg. Estándar de la industria para manipulación multimedia.
* Motor de Imágenes:libvips. Se prefiere sobre ImageMagick por su arquitectura de procesamiento por streaming, que reduce drásticamente el consumo de memoria RAM (evitando OOM Kills en Kubernetes al procesar imágenes de alta resolución).

Arquitectura del Worker y Flujo de Ejecución:

A diferencia de un servicio web tradicional, el Transcode Worker opera bajo un modelo de "polling" sobre la Task Queue de Temporal. El ciclo de vida de una tarea de transcodificación es el siguiente:

1. Aprovisionamiento de Datos (Input):
   El worker recibe la S3Key del Master. Para maximizar el rendimiento de I/O en archivos grandes, el worker descarga el archivo fuente desde Ceph a un volumen efímero de alta velocidad (EmptyDir/SSD) en el Pod, asegurando acceso aleatorio rápido (seekable) necesario para el análisis de metadatos y codificación multipasada.
2. Ejecución Controlada (Process):
   El código en Go construye el comando FFmpeg/libvips con los parámetros del perfil solicitado.
   * Heartbeating: Durante la ejecución (que puede durar horas para un largometraje), el wrapper en Go lee la salida estándar de FFmpeg (progreso de frames), calcula el porcentaje completado y envía un Heartbeat a Temporal. Esto garantiza que si el Pod muere, Temporal detecte la falla rápidamente y reprograma la tarea.
   * Cancelación: El worker escucha el contexto de cancelación de Temporal (ctx.Done()). Si el usuario cancela la tarea desde la UI, el wrapper recibe la señal y mata inmediatamente el proceso PID de FFmpeg para liberar recursos.
3. Persistencia y Limpieza (Output):
   El archivo resultante se sube a Ceph (bucket dam-proxies o dam-delivery) mediante Multipart Upload. Al finalizar, se eliminan los archivos temporales locales para mantener la higiene del nodo ("stateless execution").

Matriz de Renditions (Perfiles Predeterminados):

| Tipo de Asset | Perfil de Rendition | Especificaciones Técnicas (Container/Codec) | Propósito                                      |
| ------------- | ------------------- | -------------------------------------------- | ----------------------------------------------- |
| Video         | Preview / Proxy     | MP4 / H.264 (High Profile) @ 1080p, 5Mbps    | Edición offline y revisión de calidad.        |
| ``     | Web Low-Res         | MP4 / H.264 (Main Profile) @ 720p, 1.5Mbps   | Visualización rápida en conexiones lentas.    |
| ``     | Poster Frame        | JPG @ 1920x1080 (Q:85)                       | Imagen de portada para reproductores.           |
| ``     | Audio Waveform      | JSON / PNG                                   | Representación visual para el editor de audio. |
| Audio         | Web Preview         | MP3 @ 128kbps (CBR)                          | Preescucha rápida en navegador.                |
| ``     | Transcode Broadcast | WAV / PCM 24-bit 48kHz                       | Estandarización para edición profesional.     |
| Imagen        | Thumbnails          | WebP @ 300px, 800px, 1200px                  | Miniaturas responsivas para la UI.              |
| ``     | Display             | JPG Optimizado (MozJPEG)                     | Visualización a pantalla completa.             |

Integración con Kubernetes y Estrategia de Recursos:

1. Aislamiento de Recursos (Resource Quotas):
   Dado que la transcodificación es agresiva en consumo de CPU, estos workers se despliegan en un NodePool dedicado (o con Taints/Tolerations) para evitar que saturen los nodos donde corren la API o la Base de Datos.
   * Requests: Garantizan CPU mínima para avanzar.
   * Limits: Previenen que un proceso FFmpeg fuera de control inestabilice el nodo.
2. Escalabilidad Horizontal (Auto-Scaling):
   Se implementa KEDA (Kubernetes Event-driven Autoscaling) con el Temporal Scaler.

* Métrica: Profundidad de la cola q-transcode.
* Comportamiento: Si llegan 50 videos simultáneos, KEDA incrementa las réplicas de los Pods de workers (hasta un límite definido, ej. 100 pods) para procesar en paralelo. Al vaciarse la cola, reduce la flota para optimizar costos de infraestructura.

Justificación de Diseño (Trade-offs):

* Go vs Python/Bash: Se elige Go para el wrapper por su tipado estático y robustez. Python consume más memoria por proceso y Bash es inmanejable para lógica compleja de reintentos y parsing de errores.
* Stateless: Aunque descargar el archivo localmente consume ancho de banda interno, garantiza compatibilidad total con cualquier filtro de FFmpeg y evita errores de red a mitad de codificación, aumentando la tasa de éxito (fiabilidad) sobre el rendimiento puro de streaming.

**
