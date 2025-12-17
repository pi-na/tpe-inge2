# 8. Cambios requeridos sobre la arquitectura existente (impacto y consistencia)

> Este documento lista **modificaciones necesarias** para que lo ya definido (Ceph RGW + tiering + lifecycle, Temporal, OpenSearch, pipeline IA, distribución y CDN) quede **coherente** con la arquitectura concreta de Frontend + API y con las restricciones (IaaS / sin vendor lock‑in).

### 8.6.1. Persistencia de upload multipart

Agregar tablas (o equivalentes):

- `asset_upload_sessions`: `asset_id`, `upload_id`, `s3_key`, `part_size`, `created_by`, `status`, timestamps.
- `asset_upload_parts` (opcional): tracking de `part_number`, `etag` para diagnósticos/reintentos (no imprescindible si el FE lo conserva, pero útil para auditoría).

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
