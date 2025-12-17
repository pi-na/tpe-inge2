# 8. Cambios requeridos sobre la arquitectura existente (impacto y consistencia)

> Este documento lista **modificaciones necesarias** para que lo ya definido (Ceph RGW + tiering + lifecycle, Temporal, OpenSearch, pipeline IA, distribución y CDN) quede **coherente** con la arquitectura concreta de Frontend + API y con las restricciones (IaaS / sin vendor lock‑in).

## 8.1. Storage (Ceph RGW / S3) – staging y lifecycle

### 8.1.1. Eliminar el “staging por prefijo” (`uploads/`) como mecanismo principal

- **Problema**: en S3, un “move” equivale a `CopyObject + DeleteObject` (re-escritura). Para objetos de decenas de GB es inaceptable.
- **Decisión**: el upload multipart se realiza **directo a la key final** del master.
- **Control de visibilidad**: por estado en DB (UPLOADING/PROCESSING/READY) + no emitir URLs de lectura hasta READY.

**Acción**:
- En `dam-archive`, mantener solo:
  - `AbortIncompleteMultipartUpload: 7d`.
- Quitar regla `Expiration prefix uploads/` salvo que se mantenga un bucket de staging **separado y explícito** para casos excepcionales.

### 8.1.2. Agregar prefijo `ondemand/` y expiración corta

- **Motivo**: renditions on-demand son temporales.
- **Acción**: en `dam-proxies`, agregar:
  - `Expiration – Prefix ondemand/ (2 días)`.

## 8.2. Convención de keys inmutables (CDN/cache) – actualización necesaria

### 8.2.1. Keys versionadas e inmutables

- **Decisión**: incorporar `sha256` (o `contentHash`) en la key de masters, proxies y delivery.
- **Impacto**:
  - se habilita cache-control agresivo en CDN sin invalidaciones frecuentes;
  - simplifica idempotencia del pipeline (si la misma entrada produce el mismo hash, la salida es deduplicable).

**Acción**:
- Ajustar texto de 4.4 “Estrategia de Entrega” para que:
  - el transcodificador escriba en rutas inmutables (con hash/version);
  - la aplicación publique “punteros” (DB) a la versión activa.

## 8.3. OpenSearch – remover referencias a embeddings y ajustar el modelo

### 8.3.1. No se usan embeddings

**Decisión**: eliminar de la documentación y del diseño de índices cualquier mención a:
- `text_embedding`, k-NN, búsqueda híbrida lexical+vector.

### 8.3.2. Índices recomendados (texto)

Mantener:
- `dam-assets-video-v1`, `dam-assets-audio-v1`, `dam-assets-image-v1`.
- `dam-transcript-segments-v1`.

**Acción**:
- En el mapeo, usar campos:
  - `text` para full-text (con analyzer ES),
  - `keyword` para facetas/filtrado exacto,
  - `date`, `long`, `boolean` para rangos.
- Mantener estrategia **índices versionados + alias**.

## 8.4. Pipeline IA (enriquecimiento) – decisión concreta y simplificación

### 8.4.1. Mantener IA pero solo con salidas textuales

- **Se mantiene**:
  - STT (Whisper) → `transcriptFull` + segmentos.
  - OCR (OpenCV + Tesseract) → `ocrText`.
  - Visión (YOLO) → `visualTags[]` (tags discretos).
- **Se mantiene NLP liviano** (sin embeddings):
  - spaCy NER (ES) para entidades.
  - TF‑IDF para keywords.

**Acción**:
- Ajustar la narrativa para que “indexación inteligente” se logre por:
  - transcripción + OCR + tags + NER/keywords.

## 8.5. Temporal – nuevas colas y priorización (interactivo vs batch)

### 8.5.1. Agregar cola de transcode interactivo

- **Nueva Task Queue**: `q-transcode-interactive`.
- **Uso**: renditions on-demand solicitadas por usuarios.
- **Política operativa**: límites de concurrencia para evitar canibalizar batch.

**Acción**:
- Actualizar la sección de workers:
  - Transcode Worker escucha `q-transcode` y `q-transcode-interactive` (deployments separados o mismo binario con config distinta).
- Agregar ScaledObject de KEDA por cola (targetQueueSize más bajo para transcode pesado).

## 8.6. API y modelo de datos – nuevas entidades necesarias

### 8.6.1. Persistencia de upload multipart

Agregar tablas (o equivalentes):
- `asset_upload_sessions`: `asset_id`, `upload_id`, `s3_key`, `part_size`, `created_by`, `status`, timestamps.
- `asset_upload_parts` (opcional): tracking de `part_number`, `etag` para diagnósticos/reintentos (no imprescindible si el FE lo conserva, pero útil para auditoría).

### 8.6.2. Jobs on-demand

- `asset_jobs`: `job_id`, `asset_id`, `type` (ONDEMAND_TRANSCODE), `profile`, `status`, `result_s3_key`, `expires_at`.

## 8.7. DAM Storage YAML – actualización concreta

Cambios mínimos sobre tu `dam-storage-summary.yml`:

- En `dam-archive`:
  - **remover** `expire_prefixes: { "uploads/": 14 days }` (o marcarlo como “solo si staging bucket separado”).
- En `dam-proxies`:
  - **agregar** `"ondemand/": 2 days`.

## 8.8. Seguridad (CORS y permisos) – explicitación requerida

- Configurar CORS en RGW para `PUT` (multipart) y `GET` (download/stream) desde el dominio del frontend.
- Presigned URLs con TTL corto (p.ej. 15 min) y scope por key.
- Política: buckets privados por defecto; solo acceso mediante presigned.

## 8.9. Referencias

- Ceph RGW S3 Multipart: `https://docs.ceph.com/en/latest/radosgw/s3/objectops/`
- AWS S3 Multipart/Complete: `https://docs.aws.amazon.com/AmazonS3/latest/API/API_CompleteMultipartUpload.html`
- Temporal heartbeats/retry: `https://docs.temporal.io/encyclopedia/detecting-activity-failures` y `https://docs.temporal.io/encyclopedia/retry-policies`
- OpenSearch aliases: `https://docs.opensearch.org/latest/api-reference/alias/aliases-api/`
- KEDA Temporal scaler: `https://keda.sh/docs/2.18/scalers/temporal/`
