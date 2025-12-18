**
Propuesta: Motor de búsqueda e indexación de metadatos enriquecidos con Elasticsearch

Para cumplir con el requerimiento de búsqueda rápida e inteligente sobre millones de activos, se propone incorporar Elasticsearch como motor de indexación y consulta de metadatos enriquecidos. La decisión se basa en separar responsabilidades:

* La base relacional se mantiene como fuente canónica (transaccional) para el ciclo de vida del activo, control de estados, auditoría y metadatos técnicos (extraídos por herramientas como ffprobe/exiftool, pero persistidos solo en la DB).
* Elasticsearch se utiliza como vista derivada y optimizada para lectura (read model) que concentra metadatos enriquecidos (IA) y metadatos editoriales necesarios para explorar, filtrar y recuperar contenido con baja latencia.

  Esta separación evita sobrecargar la DB con consultas complejas y permite que la experiencia editorial (búsquedas y facetas) escale horizontalmente. Además, Elasticsearch ofrece capacidades de búsqueda full-text y, si se decide incorporarlo, búsqueda semántica/vectorial (k-NN) y consultas híbridas (lexical + semántica). (Elasticsearch Docs: https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)

---

Justificación de Elasticsearch como motor unificado

Se selecciona Elasticsearch self-hosted como motor unificado de búsqueda e indexación, sirviendo tanto para la búsqueda inteligente de activos como para el Visibility Store de Temporal (ver sección de Orquestación). Esta decisión se fundamenta en:

1. Unificación de infraestructura: Un único cluster de Elasticsearch sirve para:

   * Indexación de metadatos del DAM (índices dam-assets-*, dam-transcript-segments-*)
   * Visibility Store de Temporal (índice temporal_visibility_v1)

     Esto reduce la complejidad operativa al mantener un único motor de búsqueda en lugar de dos tecnologías separadas.
2. Advanced Visibility en Temporal: Usar Elasticsearch como Visibility Store habilita "Advanced Visibility" en Temporal, permitiendo queries complejos sobre millones de workflows que serían imposibles con el "Standard Visibility" basado en PostgreSQL:

   * Búsqueda full-text sobre Custom Search Attributes
   * Filtros complejos: LIKE, BETWEEN, ORDER BY sobre cualquier campo
   * Aggregations para reportes y métricas
   * Escala horizontal para millones de workflows

   Esto es crítico para un DAM de esta escala, donde cada asset genera múltiples workflows (ingesta, transcoding, enriquecimiento, distribución) y se requiere capacidad de debugging, auditoría y monitoreo operativo.
3. Compatibilidad con IaaS: Elasticsearch se despliega self-hosted sobre infraestructura IaaS, cumpliendo con el requerimiento de evitar vendor lock-in. No se utilizan servicios gestionados del proveedor cloud.
4. Madurez y ecosistema: Elasticsearch es el motor de búsqueda más adoptado en la industria, con amplia documentación, comunidad activa y herramientas de monitoreo (Kibana) y operación maduras.

---

Comparativa: PostgreSQL vs Elasticsearch para Visibility Store de Temporal

| Aspecto                  | PostgreSQL (Standard Visibility)          | Elasticsearch (Advanced Visibility)               |

| ------------------------ | ----------------------------------------- | ------------------------------------------------- |

| Queries soportados       | Solo filtros básicos (=, !=, IN, rangos) | Full-text, LIKE, BETWEEN, ORDER BY, agregaciones  |

| Custom Search Attributes | Limitados, sin búsqueda compleja         | Text, Keyword, Int, Bool, Datetime con operadores |

| Escala                   | Se degrada con millones de workflows      | Diseñado para escalar horizontalmente            |

| Casos de uso             | Proyectos pequeños/medianos              | Producción a gran escala                         |

Ejemplos de queries habilitados con Elasticsearch en Temporal:

* "¿Qué workflows de transcoding fallaron para el asset X en los últimos 7 días?"
* "¿Cuántos workflows de distribución a YouTube tuvieron errores esta semana?"
* "Buscar todos los workflows que procesaron archivos mayores a 1GB"
* "Listar workflows por tiempo de ejecución descendente"

---

Estructura propuesta de índices (diseño lógico)

Se define una estrategia de índices versionados + alias, para permitir evolución de mappings sin downtime. Elasticsearch soporta aliases como "nombres virtuales" que apuntan a uno o más índices, y reindexing para migraciones. (Elasticsearch Docs: https://www.elastic.co/guide/en/elasticsearch/reference/current/aliases.html)

Índices principales:

1. dam-assets-video-v1

   * Documentos: 1 por video (por assetId).
   * Contiene metadatos enriquecidos: entidades, tags IA, OCR agregado, "caption"/resumen, y referencias a renditions (keys S3) para navegación.
2. dam-assets-audio-v1

   * Documentos: 1 por audio.
   * Enriquecimiento: transcript agregado, entidades, keywords, tópicos.
3. dam-assets-image-v1

   * Documentos: 1 por imagen.
   * Enriquecimiento: etiquetas/escena/objetos, OCR, caption.

Alias de consulta unificada:

* dam-assets → alias que apunta a los 3 índices anteriores para búsquedas transversales (por ejemplo "incendio" y traer imágenes + videos + audios). (Elasticsearch Docs: https://www.elastic.co/guide/en/elasticsearch/reference/current/aliases.html)

Índice especializado para "buscar dentro del video/audio":

4) dam-transcript-segments-v1

* Documentos: 1 por segmento (no 1 por asset).
* Campos típicos: assetId, start_ms, end_ms, text, speaker_id (si hay diarización), confidence.
* Motivo: evitar documentos gigantes (un video puede generar miles de palabras/segmentos). Esto mejora escalabilidad, permite ranking fino por instante y devuelve el timestamp exacto del match.

Índice para Temporal Visibility:

5) temporal_visibility_v1

* Gestionado por Temporal Server.
* Almacena metadata de workflows para búsqueda y auditoría.
* Custom Search Attributes configurados: AssetId, WorkflowType, FileSizeBytes, FileName, UserId, Channel, etc.

Operación y evolución (sin downtime):

* La aplicación siempre consulta el alias (dam-assets, dam-transcript-segments).
* Ante cambios de mapping, se crea *-v2, se ejecuta _reindex, se valida, y luego se "mueve" el alias al nuevo índice. (Elasticsearch Docs: https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-reindex.html)

---

Enriquecimiento con IA: herramientas, trabajos y productos de metadatos

Objetivo del enriquecimiento

Generar metadatos semánticos que mejoren la exploración editorial:

* Texto searchable (transcripción y OCR).
* Etiquetas/tópicos/entidades (para filtros y facetas).

Herramientas de IA (propuesta "self-hosted" en IaaS)

Para evitar vendor lock-in y mantener control, los modelos se ejecutan en nodos de cómputo propios (CPU/GPU):

* Speech-to-Text (audio y video): Whisper auto-hospedado; para performance, una opción práctica es faster-whisper (optimización sobre CTranslate2). (GitHub: https://github.com/SYSTRAN/faster-whisper)
* OCR (imágenes y frames de video): motores OCR open-source (p.ej. Tesseract / PaddleOCR / EasyOCR, según idioma/calidad requerida).
* Visión (imagen/video): modelos open-source para tagging/captioning (ej. YOLO/Ultralytics para detección de objetos).
* NLP (sobre transcript/OCR): NER/keywords/tópicos con spaCy según idioma (ES/PT) y dominio.

Nota de diseño: Para mantener el control del pipeline y aislar la complejidad, la generación de metadatos se realiza en workers dedicados (ver sección de Enriquecimiento), y Elasticsearch se usa exclusivamente para indexar y consultar los resultados.

---

Vista física del cluster Elasticsearch

El cluster se despliega en Kubernetes sobre IaaS con la siguiente topología recomendada:

Nodos del cluster:

* Master nodes (3): Gestión del cluster, no almacenan datos. Garantizan quorum y alta disponibilidad.
* Data nodes (N, escalable): Almacenan índices y ejecutan queries. Se escalan horizontalmente según volumen.
* Coordinating nodes (2+): Balanceo de carga y agregación de resultados de queries distribuidos.

Configuración de alta disponibilidad:

* Réplicas: Cada índice con al menos 1 réplica (número de shards primarios = número de shards réplica).
* Distribución: Shards primarios y réplicas en nodos diferentes (shard allocation awareness por rack/zona).
* Snapshots: Backups periódicos a Object Storage (Ceph S3) para disaster recovery.

---

Si querés, te lo completo con un ejemplo de "documento Elasticsearch" para cada tipo (video/audio/imagen) y un ejemplo de documento de dam-transcript-segments-v1 (incluyendo cómo devolver "el timestamp exacto" del match), ya escrito como anexo técnico del informe.

**
