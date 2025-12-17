# 72.40 Ingeniería de Software

## Plataforma de Gestión de Activos Digitales (DAM) para Medios de Comunicación

---

### Alumnos
- Roman Berruti - 63533
- Tomás Pinausig - [Completar padrón]
- Agostina Squillari - 64047

### Docentes
- Sotuyo Dodero, Juan Martín
- Mogni, Guido Matías

### Fecha de entrega
Diciembre 2024

---

# Índice

1. [Introducción](#1-introducción)
2. [Funcionalidad Requerida](#2-funcionalidad-requerida)
   - 2.1 Requerimientos Funcionales
   - 2.2 Requerimientos No Funcionales
3. [Atributos de Calidad](#3-atributos-de-calidad)
   - 3.1 Disponibilidad
   - 3.2 Escalabilidad
   - 3.3 Performance
   - 3.4 Confiabilidad
   - 3.5 Tolerancia a Fallos
   - 3.6 Seguridad
   - 3.7 Interoperabilidad
4. [Arquitectura](#4-arquitectura)
   - 4.1 Estilo Arquitectónico
   - 4.2 Vista Lógica: Componentes del Sistema
   - 4.3 Resolución de Atributos de Calidad
   - 4.4 Puntos Críticos del Sistema
   - 4.5 Vista Física del Sistema
   - 4.6 Vista de Procesos: Flujos Principales
5. [Supuestos](#5-supuestos)
6. [Riesgos](#6-riesgos)
7. [No Riesgos](#7-no-riesgos)
8. [Trade-offs](#8-trade-offs)
9. [Referencias](#9-referencias)

---

# 1. Introducción

Este trabajo busca definir una arquitectura candidata para una **Plataforma de Gestión de Activos Digitales (DAM)** destinada a un grupo de medios de comunicación que administra millones de archivos multimedia, incluyendo video, audio e imágenes.

## 1.1 Objetivos del Sistema

El sistema tiene como objetivos principales:

1. **Almacenamiento a largo plazo** de activos originales (masters), garantizando su preservación y durabilidad.
2. **Generación y gestión de versiones derivadas** (renditions) optimizadas para distintos canales y casos de uso.
3. **Indexación inteligente** mediante metadatos descriptivos, técnicos y análisis de contenido con inteligencia artificial.
4. **Búsqueda rápida y semántica** que permita a los editores localizar material relevante eficientemente.
5. **Distribución multicanal automatizada** a plataformas web, televisión y redes sociales.
6. **Recuperación eficiente** de archivos grandes, manteniendo tiempos de respuesta adecuados para flujos de trabajo editoriales.
7. **Escalabilidad** para acompañar el crecimiento continuo del volumen de datos.

## 1.2 Contexto del Problema

El dominio de gestión de activos digitales para medios de comunicación presenta desafíos arquitectónicos significativos:

- **Volumen masivo**: Millones de archivos que ocupan petabytes de almacenamiento.
- **Archivos de gran tamaño**: Videos en alta resolución que pueden alcanzar varios gigabytes cada uno.
- **Diversidad de formatos**: Múltiples códecs de video, audio e imagen con diferentes características técnicas.
- **Operaciones de larga duración**: Transcodificación, análisis con IA y distribución pueden tomar minutos u horas.
- **Usuarios concurrentes**: Decenas o cientos de editores trabajando simultáneamente.
- **Integración múltiple**: Conexión con sistemas de emisión, CMS web y APIs de redes sociales.
- **Preservación**: Los activos representan patrimonio digital que debe conservarse indefinidamente.

## 1.3 Restricciones de Diseño

La solución debe cumplir con las siguientes restricciones:

- **IaaS únicamente**: En caso de utilizar nube pública, se debe limitar a servicios de infraestructura para evitar vendor lock-in.
- **Tecnologías open source**: Preferencia por soluciones de código abierto que permitan auditoría, personalización y eviten dependencias propietarias.
- **Operación interna**: El sistema debe poder ser operado por el equipo técnico del grupo de medios.

## 1.4 Metodología de Documentación

La arquitectura se documenta siguiendo el modelo de vistas 4+1:

- **Vista Lógica**: Descomposición del sistema en componentes y sus responsabilidades.
- **Vista de Procesos**: Flujos de trabajo y comunicación entre componentes.
- **Vista Física**: Despliegue en infraestructura, servidores y redes.
- **Vista de Desarrollo**: Organización del código y tecnologías utilizadas.
- **Escenarios**: Casos de uso que validan la arquitectura (documentados a lo largo del informe).

---

# 2. Funcionalidad Requerida

## 2.1 Requerimientos Funcionales

Los requerimientos funcionales se presentan ordenados por prioridad según su impacto en la operación del negocio:

| Prioridad | ID | Requerimiento | Justificación |
|-----------|-----|---------------|---------------|
| 1 | RF-01 | **Almacenamiento de archivos digitales** de gran tamaño (video, audio, imagen) con garantía de conservación a largo plazo | Core del sistema; sin almacenamiento confiable no hay DAM |
| 2 | RF-02 | **Recuperación y descarga de activos**, incluyendo archivos grandes, facilitando acceso para uso editorial | Operación diaria de editores depende de esto |
| 3 | RF-03 | **Búsqueda rápida y eficiente** sobre el conjunto de activos utilizando metadatos e información derivada de análisis de contenido | Principal forma de interacción; editores buscan más de lo que suben |
| 4 | RF-04 | **Asociación de metadatos descriptivos** a cada activo y enriquecimiento mediante análisis automático con IA | Habilita la búsqueda inteligente y el descubrimiento de contenido |
| 5 | RF-05 | **Distribución automatizada** a múltiples canales de salida (web, TV, redes sociales) | Reduce trabajo manual y acelera publicación |

### Detalle de Funcionalidades

#### RF-01: Almacenamiento a Largo Plazo

- Ingesta de archivos mediante upload resumable para archivos grandes.
- Validación de formato y tipo de archivo al momento de ingesta.
- Verificación de integridad mediante checksums (SHA-256).
- Almacenamiento inmutable del archivo original (master).
- Generación automática de renditions derivadas (previews, thumbnails, proxies).

#### RF-02: Recuperación y Descarga

- Descarga de archivos master previa autorización.
- Streaming de previews para visualización rápida.
- Soporte para descargas parciales y resumibles (HTTP Range).
- Generación de renditions bajo demanda para formatos específicos.

#### RF-03: Búsqueda

- Búsqueda por texto libre sobre título, descripción, tags y transcripciones.
- Filtrado por facetas: tipo de archivo, fecha, proyecto, estado, duración.
- Búsqueda semántica utilizando embeddings de contenido.
- Previsualización de resultados con thumbnails.
- Paginación eficiente para grandes conjuntos de resultados.

#### RF-04: Metadatos e IA

- Metadatos descriptivos editables por usuarios (título, descripción, tags, categoría).
- Extracción automática de metadatos técnicos (resolución, codec, duración, EXIF).
- Análisis de contenido con IA:
  - Detección de objetos y escenas (labels).
  - Reconocimiento óptico de caracteres (OCR).
  - Transcripción de audio (ASR/speech-to-text).
  - Generación de embeddings para búsqueda semántica.

#### RF-05: Distribución Multicanal

- Configuración de canales de distribución (CMS web, sistemas de playout TV, redes sociales).
- Reglas de publicación basadas en metadata, estado editorial o triggers manuales.
- Adaptación automática de formato según requisitos del canal destino.
- Tracking de estado de publicación por canal.
- Reintentos automáticos ante fallos de publicación.

---

## 2.2 Requerimientos No Funcionales

| ID | Requerimiento | Métrica / Criterio de Aceptación |
|----|---------------|----------------------------------|
| RNF-01 | Soportar millones de activos y crecimiento continuo | >5M activos iniciales, crecimiento >500K/año sin degradación |
| RNF-02 | Tiempos de respuesta adecuados para uso editorial | Búsqueda <500ms P95, preview en <2s |
| RNF-03 | Recuperación eficiente de archivos grandes | Throughput de descarga >100MB/s por stream |
| RNF-04 | Escalabilidad de almacenamiento y procesamiento | Escalar horizontalmente sin cambios de arquitectura |
| RNF-05 | Conservación a largo plazo sin pérdida de datos | Durabilidad >99.999999999% (11 nueves) |
| RNF-06 | Alta disponibilidad del servicio | Uptime >99.9% (8.7h downtime/año máximo) |
| RNF-07 | Seguridad acorde a entorno corporativo | AuthN/AuthZ, cifrado, auditoría |
| RNF-08 | Sin vendor lock-in en nube | Uso exclusivo de IaaS, APIs estándar |

---

# 3. Atributos de Calidad

Los atributos de calidad se presentan ordenados por prioridad según su criticidad para el éxito del sistema:

## 3.1 Disponibilidad (Prioridad: Crítica)

La disponibilidad constituye uno de los atributos más críticos. La indisponibilidad del sistema impacta directamente en la capacidad de producir, editar y publicar contenidos, afectando los tiempos de respuesta del negocio.

**Justificación de negocio:**
- La plataforma DAM es utilizada intensivamente por editores, productores y sistemas automatizados de publicación.
- El contenido puede ser requerido en distintos momentos del día y para múltiples canales.
- El sistema es origen del contenido distribuido automáticamente; una indisponibilidad interrumpe flujos de publicación.

**Escenario de calidad:**

| Elemento | Valor |
|----------|-------|
| Fuente | Falla de hardware o software |
| Estímulo | Caída de un componente del sistema |
| Artefacto | Sistema DAM |
| Ambiente | Operación normal, horario laboral |
| Respuesta | Failover automático, servicio continúa |
| Medida | Tiempo de recuperación <30s, disponibilidad >99.9% |

## 3.2 Escalabilidad (Prioridad: Crítica)

El sistema debe manejar crecimiento continuo tanto en volumen de datos como en usuarios y procesos concurrentes, sin requerir rediseño de arquitectura.

**Justificación de negocio:**
- Se administran millones de archivos con incremento constante.
- Los archivos no se eliminan; forman parte del patrimonio digital.
- Eventos de alta relevancia generan picos de demanda abruptos.

**Escenario de calidad:**

| Elemento | Valor |
|----------|-------|
| Fuente | Crecimiento orgánico del negocio |
| Estímulo | Duplicación del volumen de assets en 3 años |
| Artefacto | Sistema completo |
| Ambiente | Operación normal |
| Respuesta | Agregar capacidad sin downtime |
| Medida | Escalamiento lineal de recursos vs. capacidad |

## 3.3 Performance (Prioridad: Alta)

El rendimiento condiciona directamente la operación editorial. Se exige búsqueda rápida y velocidad de recuperación de archivos grandes.

**Justificación de negocio:**
- El editor necesita localizar material y obtener previsualización rápidamente.
- Búsquedas lentas interrumpen el flujo de trabajo.
- El sistema debe sostener alto throughput en operaciones pesadas (ingesta, transcodificación, distribución).

**Escenario de calidad:**

| Elemento | Valor |
|----------|-------|
| Fuente | Editor |
| Estímulo | Búsqueda de contenido |
| Artefacto | Search Service |
| Ambiente | Carga normal (100 usuarios concurrentes) |
| Respuesta | Resultados relevantes con previews |
| Medida | Latencia <500ms P95 |

## 3.4 Confiabilidad (Prioridad: Alta)

La plataforma debe operar consistentemente, minimizando fallas en flujos críticos como ingesta, generación de renditions, análisis con IA e indexación.

**Justificación de negocio:**
- Procesos asíncronos de larga duración no deben perderse.
- Los resultados deben ser consistentes y los trabajos reintentables de forma segura.
- Baja confiabilidad genera retrabajo manual y pérdida de confianza.

**Escenario de calidad:**

| Elemento | Valor |
|----------|-------|
| Fuente | Sistema de workflow |
| Estímulo | Falla durante procesamiento de asset |
| Artefacto | Workflow Orchestrator |
| Ambiente | Procesamiento asíncrono |
| Respuesta | Reintento automático, recuperación del estado |
| Medida | <0.1% de jobs en Dead Letter Queue |

## 3.5 Tolerancia a Fallos (Prioridad: Alta)

La falla de un componente individual no debe implicar caída total del sistema. El servicio debe continuar operando, aunque sea de forma degradada.

**Justificación de negocio:**
- Sistema complejo con muchos componentes.
- Fallas parciales de infraestructura o software son inevitables.
- Combina operaciones interactivas con procesos de larga duración.

**Escenario de calidad:**

| Elemento | Valor |
|----------|-------|
| Fuente | Falla de infraestructura |
| Estímulo | Caída del servicio de IA |
| Artefacto | Sistema completo |
| Ambiente | Operación normal |
| Respuesta | Assets se procesan sin enriquecimiento de IA; cola pendiente |
| Medida | Funciones core (búsqueda, descarga) no afectadas |

## 3.6 Seguridad (Prioridad: Alta)

El sistema administra activos de alto valor que representan propiedad intelectual, incluyendo material inédito o sensible.

**Justificación de negocio:**
- Control estricto de acceso, descarga, modificación y distribución.
- Prevención de fugas de información.
- Cumplimiento de requisitos legales y derechos sobre material.

**Escenario de calidad:**

| Elemento | Valor |
|----------|-------|
| Fuente | Usuario no autorizado |
| Estímulo | Intento de acceso a asset restringido |
| Artefacto | Asset Service |
| Ambiente | Operación normal |
| Respuesta | Acceso denegado, evento registrado en auditoría |
| Medida | 0 accesos no autorizados exitosos |

## 3.7 Interoperabilidad (Prioridad: Media)

La función principal del DAM es integrarse con distintos sistemas del ecosistema del grupo de medios.

**Justificación de negocio:**
- Comunicación con CMS web, sistemas de emisión TV, redes sociales.
- Distribución automatizada requiere interfaces basadas en estándares.
- Reducción de intervenciones manuales y facilidad para incorporar nuevos canales.

**Escenario de calidad:**

| Elemento | Valor |
|----------|-------|
| Fuente | Sistema externo (CMS) |
| Estímulo | Solicitud de asset para publicación |
| Artefacto | Distribution Service |
| Ambiente | Integración automatizada |
| Respuesta | Asset entregado en formato requerido |
| Medida | APIs REST/S3 estándar, <1 día de integración para nuevos canales |

---

# 4. Arquitectura

*[Sección completa en documento Arquitectura.md]*

## 4.1 Estilo Arquitectónico

La arquitectura propuesta combina **microservicios moderados**, **arquitectura orientada a eventos** (Event-Driven) y **CQRS parcial**.

### Justificación

- **Microservicios moderados**: Servicios de tamaño moderado que agrupan funcionalidades cohesivas, reduciendo complejidad operacional sin sacrificar desacoplamiento.
- **Event-Driven Architecture**: Desacopla temporalmente productores y consumidores, permitiendo que procesos pesados escalen independientemente.
- **CQRS parcial**: Separa modelo de escritura (PostgreSQL) del modelo de lectura optimizado (OpenSearch) para búsquedas.

### Patrones Arquitectónicos

| Patrón | Aplicación |
|--------|------------|
| API Gateway | Punto de entrada único, autenticación |
| Saga / Orquestación | Workflows de procesamiento |
| Outbox Pattern | Publicación confiable de eventos |
| Circuit Breaker | Conectores de distribución |
| Retry con Backoff | Workers de procesamiento |
| Dead Letter Queue | Manejo de jobs fallidos |

## 4.2 Vista Lógica: Componentes

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              PRESENTACIÓN                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    Editor UI (React + TypeScript)                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                                 ACCESO                                      │
│  ┌────────────────────┐      ┌────────────────────┐                        │
│  │  API Gateway/BFF   │      │   AuthN/AuthZ      │                        │
│  │     (Kong)         │◄────►│ (Keycloak + OPA)   │                        │
│  └────────────────────┘      └────────────────────┘                        │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           SERVICIOS DE DOMINIO                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │Asset Service │  │Metadata Svc  │  │Search Service│  │Distribution  │   │
│  │    (Go)      │  │    (Go)      │  │    (Go)      │  │Service (Go)  │   │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘   │
│                            │                                                │
│  ┌─────────────────────────┴─────────────────────────┐                     │
│  │              Workflow Orchestrator (Temporal)      │                     │
│  └────────────────────────────────────────────────────┘                     │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            PROCESAMIENTO                                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                      │
│  │  Transcode   │  │AI Enrichment │  │   Indexer    │                      │
│  │Workers(FFmpeg)│  │  (Python)    │  │   Workers    │                      │
│  └──────────────┘  └──────────────┘  └──────────────┘                      │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                                DATOS                                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │ PostgreSQL   │  │    Kafka     │  │  OpenSearch  │  │    Ceph      │   │
│  │ (Transacc.)  │  │  (Eventos)   │  │  (Búsqueda)  │  │  (Storage)   │   │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘   │
│  ┌──────────────┐                                                          │
│  │    Redis     │                                                          │
│  │   (Cache)    │                                                          │
│  └──────────────┘                                                          │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Componentes Principales

| Componente | Tecnología | Responsabilidad |
|------------|------------|-----------------|
| API Gateway/BFF | Kong + Node.js | Routing, auth, rate limiting, agregación |
| AuthN/AuthZ | Keycloak + OPA | Identidades, permisos RBAC/ABAC |
| Asset Service | Go | Ciclo de vida de assets, coordinación storage |
| Metadata Service | Go | Metadatos descriptivos, técnicos, validación |
| Search Service | Go | Búsqueda full-text + semántica |
| Workflow Orchestrator | Temporal | Orquestación de procesos de larga duración |
| Distribution Service | Go | Publicación multicanal |
| Transcode Workers | Go + FFmpeg | Generación de renditions |
| AI Enrichment | Python + FastAPI | Análisis de contenido con IA |
| Indexer Workers | Go | Actualización de índices de búsqueda |

### Opciones Analizadas por Componente Crítico

**Object Storage:**
- ✓ **Ceph RGW** (elegido para escala PB): Erasure coding, durabilidad 11 nueves, S3-compatible
- MinIO (alternativa): Más simple, ideal para escala inicial o PoC

**Message Broker:**
- ✓ **Apache Kafka** (elegido): Log durable, replay, múltiples consumidores
- RabbitMQ (alternativa): Más simple para job dispatch

**Base de Datos:**
- ✓ **PostgreSQL** (elegido): Robusto, JSONB, replicación madura
- CockroachDB (alternativa futura): Escala horizontal nativa

**Orquestación:**
- ✓ **Temporal** (elegido): Workflows durables, reintentos, visibilidad
- Jobs custom (alternativa simple): Menor complejidad pero más riesgoso

## 4.3 Resolución de Atributos de Calidad

| Atributo | Mecanismos | Componentes |
|----------|------------|-------------|
| Disponibilidad | Clusters replicados, failover automático, stateless | PostgreSQL+Patroni, Kafka, K8s |
| Escalabilidad | Escala horizontal, sharding, partitioning | Workers, Ceph, Kafka |
| Performance | Índices, caching, procesamiento async | OpenSearch, Redis, Kafka |
| Confiabilidad | Workflows durables, reintentos, DLQ | Temporal, checksums |
| Tolerancia a fallos | Circuit breakers, graceful degradation | Distribution, Workers |
| Seguridad | RBAC/ABAC, TLS, cifrado, URLs prefirmadas | Keycloak, OPA, Ceph |
| Interoperabilidad | APIs REST, S3-compatible, conectores | Gateway, Distribution |

## 4.4 Puntos Críticos del Sistema

### 4.4.1 Ingesta de Archivos Grandes

**Solución:** Protocolo tus para upload resumable por chunks.
- Chunks de 5MB
- Verificación de checksum SHA-256 end-to-end
- Limpieza automática de uploads abandonados

### 4.4.2 Recuperación Eficiente

**Solución:** URLs prefirmadas + HTTP Range + tiering.
- Descargas directas desde Object Storage
- Hot tier (SSD) para assets recientes
- Cold tier (HDD) para archivados

### 4.4.3 Búsqueda Rápida con Millones de Assets

**Solución:** OpenSearch con búsqueda híbrida.
- BM25 para texto
- k-NN para vectores semánticos
- Cache de queries en Redis

### 4.4.4 Procesamiento Asíncrono Confiable

**Solución:** Temporal con patrones de confiabilidad.
- Idempotencia por job
- Retry con backoff exponencial
- Dead Letter Queue para análisis

## 4.5 Vista Física

```
┌────────────────────────────────────────────────────────────────────────────┐
│                                INTERNET                                     │
│                 Editores ─── Sistemas Externos (CMS, TV, Social)           │
└────────────────────────────────────────────────────────────────────────────┘
                                     │
                              HTTPS (TLS 1.3)
                                     │
┌────────────────────────────────────────────────────────────────────────────┐
│                           DMZ                                               │
│              Load Balancer (HAProxy) ── WAF                                │
└────────────────────────────────────────────────────────────────────────────┘
                                     │
┌────────────────────────────────────────────────────────────────────────────┐
│                    KUBERNETES CLUSTER                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Control Plane (3 nodos HA)                                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐                  │
│  │Worker General │  │Worker General │  │ Worker GPU    │                  │
│  │8CPU,32GB RAM  │  │8CPU,32GB RAM  │  │ 8CPU,64GB,T4  │                  │
│  │(Services)     │  │(Services)     │  │ (AI Workers)  │                  │
│  └───────────────┘  └───────────────┘  └───────────────┘                  │
└────────────────────────────────────────────────────────────────────────────┘
                                     │
┌────────────────────────────────────────────────────────────────────────────┐
│                        CAPA DE DATOS                                        │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐               │
│  │  PostgreSQL    │  │     Kafka      │  │   OpenSearch   │               │
│  │  Primary + 2R  │  │   3 Brokers    │  │   3 Data Nodes │               │
│  │  + PgBouncer   │  │   + ZooKeeper  │  │   + Coord      │               │
│  └────────────────┘  └────────────────┘  └────────────────┘               │
│                                                                             │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐               │
│  │     Redis      │  │    Temporal    │  │      Ceph      │               │
│  │    Cluster     │  │  Frontend +    │  │  3 MON + 6 OSD │               │
│  │   6 nodos      │  │  History +     │  │  + 2 RGW       │               │
│  │                │  │  Matching      │  │  (Hot + Cold)  │               │
│  └────────────────┘  └────────────────┘  └────────────────┘               │
└────────────────────────────────────────────────────────────────────────────┘
```

### Especificación de Hardware

| Componente | Cantidad | CPU | RAM | Storage |
|------------|----------|-----|-----|---------|
| K8s Control | 3 | 4 | 8GB | 100GB SSD |
| K8s Worker General | 6+ | 8 | 32GB | 200GB SSD |
| K8s Worker GPU | 2-4 | 8 | 64GB | 200GB SSD + T4 |
| PostgreSQL | 3 | 16 | 64GB | 2TB NVMe |
| Kafka | 3 | 8 | 32GB | 1TB SSD |
| OpenSearch | 3 | 16 | 64GB | 2TB NVMe |
| Redis | 6 | 4 | 16GB | 100GB SSD |
| Ceph OSD | 6+ | 8 | 32GB | 4x10TB HDD + 400GB NVMe |

---

# 5. Supuestos

*[Sección completa en documento RiesgosTradeoffs.md]*

## Supuestos Principales

| Área | Supuesto |
|------|----------|
| Negocio | Presupuesto disponible para infraestructura significativa |
| Negocio | Equipo técnico capaz de operar la solución |
| Negocio | ~5M assets iniciales, crecimiento ~500K/año |
| Técnico | Archivos master no requieren edición in-place |
| Técnico | Formatos multimedia estándar (H.264, AAC, JPEG) |
| Integración | Directorio corporativo disponible para federación |

---

# 6. Riesgos

*[Sección completa en documento RiesgosTradeoffs.md]*

## Riesgos Principales

| ID | Riesgo | Prob. | Impacto | Mitigación |
|----|--------|-------|---------|------------|
| R-01 | Complejidad operacional de Ceph | Media | Alto | Capacitación, soporte comercial |
| R-02 | Degradación de performance de búsqueda | Media | Alto | Sharding apropiado, monitoreo |
| R-03 | Fallas en integraciones externas | Alta | Medio | Circuit breakers, retry independiente |
| R-04 | Bottleneck en procesamiento de IA | Media | Medio | Priorización, escalamiento elástico |
| R-05 | Corrupción silenciosa de datos | Baja | Muy Alto | Checksums, scrubbing, erasure coding |

---

# 7. No Riesgos

*[Sección completa en documento RiesgosTradeoffs.md]*

Los siguientes aspectos se consideran controlados:

- **Disponibilidad del sistema core**: Componentes replicados, failover automático.
- **Pérdida de datos de assets**: Erasure coding, durabilidad 11 nueves.
- **Escalabilidad de almacenamiento**: Ceph escala horizontalmente sin límites prácticos.
- **Vendor lock-in**: APIs S3-compatible, tecnologías open source.
- **Seguridad de acceso**: RBAC/ABAC, TLS, cifrado, URLs prefirmadas.

---

# 8. Trade-offs

*[Sección completa en documento RiesgosTradeoffs.md]*

## Trade-offs Principales

| Decisión | Se Sacrifica | Se Gana |
|----------|--------------|---------|
| Consistencia eventual en búsqueda | Consistencia inmediata | Performance, simplicidad |
| Stack complejo (Ceph, Kafka, Temporal) | Simplicidad operacional | Escala, funcionalidad enterprise |
| Verificación de checksum en upload | Velocidad máxima de ingesta | Garantía de integridad |
| Tiering hot/cold | Velocidad uniforme de acceso | Reducción de costos (~5x) |
| Procesamiento eager de IA | Recursos de cómputo | Descubribilidad total desde el inicio |
| URLs prefirmadas vs. proxy | Control durante descarga | Escala ilimitada de transferencias |

---

# 9. Referencias

1. **Temporal.io Documentation**. https://docs.temporal.io/
2. **Apache Kafka Documentation**. https://kafka.apache.org/documentation/
3. **Ceph Documentation**. https://docs.ceph.com/
4. **OpenSearch Documentation**. https://opensearch.org/docs/latest/
5. **Keycloak Documentation**. https://www.keycloak.org/documentation
6. **Open Policy Agent Documentation**. https://www.openpolicyagent.org/docs/latest/
7. **tus - Resumable File Uploads**. https://tus.io/
8. **FFmpeg Documentation**. https://ffmpeg.org/documentation.html
9. **Kong Gateway Documentation**. https://docs.konghq.com/
10. **PostgreSQL Documentation**. https://www.postgresql.org/docs/
11. **Kubernetes Documentation**. https://kubernetes.io/docs/
12. **Redis Documentation**. https://redis.io/documentation
13. **Bass, L., Clements, P., & Kazman, R. (2012)**. *Software Architecture in Practice* (3rd ed.). Addison-Wesley.
14. **Richards, M., & Ford, N. (2020)**. *Fundamentals of Software Architecture*. O'Reilly Media.
15. **Kleppmann, M. (2017)**. *Designing Data-Intensive Applications*. O'Reilly Media.

---

*Documento generado para la materia 72.40 Ingeniería de Software - ITBA*

