**fuentes

https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpu-abort-incomplete-mpu-lifecycle-config.html

Aquí tenés el mismo fragmento reescrito con los ajustes (precisión técnica, coherencia con el YAML, staging lógico sin “mover”, y CDN basada en keys inmutables). Mantengo tu estructura y estilo.

---

## 4. Estrategia de Almacenamiento y Persistencia de Datos

Para satisfacer los requerimientos de escalabilidad masiva (millones de activos), alta performance en la recuperación y soberanía de datos (No Vendor Lock-in), se selecciona Ceph como solución unificada de almacenamiento definido por software (SDS) operada sobre infraestructura IaaS.

La arquitectura implementa Ceph Object Gateway (RGW) para exponer una interfaz S3-compatible. A fin de balancear costo (archivo histórico de largo plazo) y baja latencia (previsualización editorial y distribución), se adopta una estrategia de Tiering Interno dentro del mismo clúster, diferenciando el hardware mediante CRUSH device classes (NVMe/SSD/HDD) y asignando cada tipo de dato al tier apropiado.

---

### 4.1. Diseño de Pools y Segregación de Hardware (Infraestructura)

Siguiendo la especificación declarativa (dam-storage-summary.yml), se definen pools separados por clase de dispositivo y estrategia de protección:

1. Pool de Índices (dam.rgw.buckets.index)
   * Medio: NVMe.
   * Protección: Replicación x3.
   * Función: aloja los índices/estructuras de metadata del RGW asociadas a operaciones S3 (p.ej. listados, consultas de metadata). Al residir en NVMe, reduce significativamente la latencia de operaciones HEAD/LIST y acceso a objetos con alto fan-out, evitando cuellos de botella cuando el sistema administra millones de objetos.
2. Pool de Masters (dam.masters.data)

* Medio: HDD.
* Protección:Erasure Coding (4+2).
* Función: almacenamiento costo-eficiente para el archivo histórico (masters/originales) con retención prolongada. El esquema 4+2 tolera la pérdida simultánea de dos fallos en el conjunto de fragmentos, reduciendo el overhead respecto de replicación x3 y optimizando el costo por terabyte para volúmenes de escala petabyte.
* Trade-off explícito: el erasure coding puede implicar mayor costo computacional en ciertas operaciones; por ello se reserva para datos de acceso menos frecuente y se mantiene un tier “hot” replicado para entrega.

1. Pool de Entrega / Hot Tier (dam.renditions.data)

* Medio: SSD.
* Protección: Replicación x3.
* Función: tier de baja latencia y alto IOPS para datos de alta demanda (renditions editoriales, thumbnails y objetos de distribución). La replicación reduce penalidades de CPU en lecturas y favorece un comportamiento estable bajo picos de concurrencia.

---

### 4.2. Configuración Lógica (Placement Targets)

Para abstraer la complejidad del hardware y permitir que la aplicación interactúe únicamente mediante S3, se definen dos placement targets en RGW:

* STANDARD: enruta al pool dam.masters.data (HDD + EC). Se utiliza para preservación de masters.
* HOT-STORAGE: enruta al pool dam.renditions.data (SSD + réplica). Se utiliza para proxies, thumbnails y distribución.

De este modo, el sistema DAM selecciona el bucket destino (p. ej., archivo histórico vs entrega), y RGW aplica internamente la política de colocación correspondiente.

---

### 4.3. Estructura de Buckets y Gestión del Ciclo de Vida (Lifecycle Management)

La organización lógica se divide en tres buckets principales, cada uno con reglas de S3 Lifecycle diseñadas para:

* mantener higiene operativa (limpieza de cargas incompletas),
* controlar costos (expirar objetos regenerables en tiers costosos),
* y sostener la performance evitando acumulación indefinida de derivados.

Nota: Las reglas de lifecycle gestionan retención y limpieza; la durabilidad se garantiza por las estrategias de protección de Ceph (EC/replicación) y sus capacidades de self-healing.

#### A. Bucket: dam-archive (Source of Truth)

* Propósito: almacenamiento de masters/originales y preservación a largo plazo.
* Placement Target:STANDARD (HDD + Erasure Coding).
* Versioning:Enabled (protección ante sobrescrituras accidentales y soporte a auditoría/rollback).
* Políticas de ciclo de vida:
  1. AbortIncompleteMultipartUpload (7 días):
     Justificación: al utilizar S3 Multipart Upload (MPU) para archivos grandes, esta regla elimina cargas incompletas y fragmentos huérfanos, evitando consumo de almacenamiento por subidas fallidas.

Aclaración de diseño (staging lógico): la “promoción” de un master se controla por el estado en la base de datos (UPLOADING → READY/PROCESSING), no por un “move” físico del objeto. Idealmente, el MPU se completa directamente en la key final del master (p. ej. masters/{assetId}/{sha256}/...) para evitar copias de objetos grandes; uploads/ se reserva para casos donde se requiera staging explícito.

---

#### B. Bucket: dam-proxies (Trabajo Editorial)

* Propósito: renditions internas (proxies de baja resolución, thumbnails) para edición y previsualización.
* Placement Target:HOT-STORAGE (SSD).
* Políticas de ciclo de vida:
  1. AbortIncompleteMultipartUpload (7 días):
     Justificación: higiene de cargas parciales en un bucket de alta rotación.
  2. Expiration – Prefix proxies/ (90 días):
     Justificación: los proxies son regenerables a partir del master. Mantenerlos indefinidamente en SSD incrementa costos sin aportar valor proporcional.
  3. Expiration – Prefix thumbs/ (180 días):
     Justificación: miniaturas de bajo peso relativo, útiles para exploración visual y UX; se retienen más tiempo para minimizar regeneración.
  4. Expiration – Prefix ondemand/ (2 días):
     Justificación: elimina rendiciones generadas on-demand.

---

#### C. Bucket: dam-delivery (CDN Origin)

* Propósito: salida para distribución (web, TV, redes) y origen para CDN.
* Placement Target:HOT-STORAGE (SSD).
* Políticas de ciclo de vida:

actualizar valores de dias

1. AbortIncompleteMultipartUpload (7 días):
   Justificación: evita acumulación de cargas incompletas en el bucket de entrega.
2. Expiration – Prefix web/ (60 días):
   Justificación: contenido de actualidad con caída rápida de demanda; puede regenerarse bajo demanda si fuera necesario.
3. Expiration – Prefix social/ (90 días):
   Justificación: los destinos (plataformas sociales) almacenan copia; el bucket funciona como buffer de entrega temporal.
4. Expiration – Prefix tv/ (365 días):
   Justificación: ventanas de disponibilidad más amplias por necesidades de broadcast/partners y potenciales acuerdos de sindicación.

---

### 4.4. Estrategia de Entrega y Distribución (CDN)

El bucket `dam-delivery` actúa como **Origin** para la CDN. Se utiliza **static packaging**: el transcodificador genera HLS/DASH (y/o archivos “flat” para redes) y los escribe en los prefijos por canal (`web/`, `tv/`, `social/`).

**Decisión**: la entrega se basa en **keys inmutables versionadas** (incluyendo `sha256`/`contentHash` y versión de perfil). No se sobrescriben objetos: ante un re-encode o cambio, se escribe una **nueva key**. Esto permite **cache-control agresivo** en CDN (alto hit ratio) sin invalidaciones frecuentes.

La “versión activa” de cada salida se publica por **punteros en la base de datos** (no por paths mutables): la aplicación guarda la `s3_key` activa por `assetId + canal + profile` y expone esa referencia a través de la API (o URL CDN derivada). Si cambia la versión, solo se actualiza el puntero en DB.

#### 4.4.1. Convención de keys (ejemplo)

- `delivery/web/{assetId}/{sha256}/{profileVersion}/hls/master.m3u8`
- `delivery/tv/{assetId}/{sha256}/{profileVersion}/dash/manifest.mpd`
- `delivery/social/{assetId}/{sha256}/{profileVersion}/{filename}.mp4`

**Trade-off**: requiere que consumidores usen la API/DB para obtener la versión vigente; a cambio simplifica cache y evita servir contenido obsoleto.

---

### 4.5. Artefacto de Configuración

A continuación, se adjunta la especificación técnica declarativa (dam-storage-summary.yml) que resume la configuración propuesta:

# dam-storage-summary.yml

storage:

  technology: "Ceph SDS + Ceph RGW (API S3 compatible)"

  goal: "Tiering interno (HDD+EC para archivo / SSD para entrega)"

ceph:

  pools:

    - name: dam.rgw.buckets.index

    media: NVMe

    protection: "replicación x3"

    description: "Índices/metadata RGW para operaciones S3 de baja latencia"

    - name: dam.masters.data

    media: HDD

    protection: "erasure coding 4+2"

    description: "Masters: costo eficiente para largo plazo"

    - name: dam.renditions.data

    media: SSD

    protection: "replicación x3"

    description: "Hot tier para delivery y previews"

rgw:

  placement_targets:

    - name: STANDARD

    data_pool: dam.masters.data

    - name: HOT-STORAGE

    data_pool: dam.renditions.data

s3:

  buckets:

    - name: dam-archive

    purpose: "masters (source of truth)"

    placement: STANDARD

    versioning: Enabled

    lifecycle:

    abort_incomplete_mpu: 7 days

    expire_prefixes: { "uploads/": 14 days }

    - name: dam-proxies

    purpose: "renditions editoriales"

    placement: HOT-STORAGE

    lifecycle:

    abort_incomplete_mpu: 7 days

    expire_prefixes: { "proxies/": 90 days, "thumbs/": 180 days, "ondemand/": 2 days }

    - name: dam-delivery

    purpose: "CDN origin"

    placement: HOT-STORAGE

    lifecycle:

    abort_incomplete_mpu: 7 days

    expire_prefixes: { "web/": 60 days, "social/": 90 days, "tv/": 365 days }

---

Si querés, te agrego un mini-subapartado (2–3 párrafos) con convención de keys recomendada (masters/{assetId}/{sha256}/..., delivery/web/{assetId}/{sha256}/...) porque eso conecta muy bien con lifecycle + CDN + idempotencia del pipeline.

**
