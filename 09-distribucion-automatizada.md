# 9. Distribución Automatizada de Contenido

## 9.1. Objetivo

La distribución automatizada tiene como objetivo publicar activos (video/audio/imagen) hacia múltiples canales externos (web/CMS, TV/Playout, redes sociales, etc.) de manera confiable, escalable y trazable, sin exigir intervención manual repetitiva de los editores.

En este DAM, la distribución es especialmente crítica porque:
- Los archivos son grandes (millones de assets): la publicación debe ser eficiente y evitar I/O innecesario.
- Los canales externos son inestables (APIs con rate limits, timeouts, caídas): se requiere reintento controlado, idempotencia y registro de estado.
- La plataforma opera bajo picos (breaking news): el sistema debe autoescalar sin sobredimensionar.

---

## 9.2. Aclaración Conceptual: Distribución vs. Streaming

### 9.2.1. Distribución (Publication to External Channels)

**Distribución** se refiere a la **publicación activa** de contenido hacia canales externos donde el DAM **no controla el servidor de entrega**:

- **Redes sociales**: YouTube, Facebook, Twitter/X, Instagram, TikTok
- **CMS externos**: WordPress, Drupal, sistemas de gestión de contenido de terceros
- **Playout/TV**: sistemas de emisión broadcast (MOS, FTP/SFTP)
- **CDNs externos**: publicación de archivos a CDNs de terceros (no el CDN interno del DAM)

En este caso, el DAM **envía/empuja** el contenido hacia el canal externo mediante:
- APIs REST (YouTube Data API, Facebook Graph API)
- Protocolos de transferencia (FTP/SFTP para playout)
- Uploads HTTP multipart

**El contenido queda almacenado en el sistema externo**, y los usuarios finales lo consumen desde ese sistema.

### 9.2.2. Streaming (Serving Content to Clients via CDN)

**Streaming** se refiere a la **entrega de contenido** desde el DAM hacia clientes finales (navegadores, apps) a través de un **CDN controlado por el DAM**:

- El DAM prepara proxies en formatos de streaming (HLS/DASH) y los almacena en `dam-proxies` (hot storage).
- El CDN interno (o edge) sirve estos proxies a los clientes bajo demanda.
- Los clientes **solicitan** el contenido mediante HTTP Range Requests.

**El contenido permanece en el DAM**, y el CDN actúa como caché/acelerador.

### 9.2.3. Separación de Responsabilidades

| Aspecto | Distribución | Streaming |
|---------|--------------|-----------|
| **Dirección** | Push (DAM → Canal externo) | Pull (Cliente → DAM/CDN) |
| **Control** | DAM no controla el destino | DAM controla el CDN |
| **Almacenamiento** | Contenido queda en sistema externo | Contenido queda en DAM |
| **Ejemplo** | Publicar video a YouTube | Servir HLS desde CDN interno |
| **Workflow** | Distribution Worker (q-distribution) | No requiere workflow (servicio HTTP) |

**Conclusión**: La distribución automatizada (este documento) se enfoca en **publicar contenido a canales externos**. El streaming interno (HLS/DASH) es responsabilidad del servicio de entrega (CDN) y no requiere workflows de distribución.

---

## 9.3. Rol del Distribution Service dentro de Temporal

La distribución se modela como una etapa del workflow gestionado por Temporal:

- La lógica **"cuándo distribuir"** (aprobación editorial, embargo/ventana temporal, renditions disponibles) vive en el workflow.
- La ejecución concreta **"cómo distribuir"** se implementa como Activities en Temporal, ejecutadas por un grupo de workers específico:

**Dist Worker Group (Java)**
- Task Queue: `q-distribution`
- Perfil: High Network, Low CPU (I/O bound)
- Responsabilidad: transferir/emitir contenido a canales externos y registrar resultados.

Esto permite que la etapa de distribución:
- Sea durable y reanudable (si un worker o pod cae, Temporal reintenta/reagenda).
- Escale de forma independiente del transcoding o IA (cada cola escala por demanda real).

---

## 9.4. Diseño Interno del Distribution Worker (Java)

### 9.4.1. Núcleo + Conectores por Canal

El worker implementa un núcleo de publicación y un conjunto de conectores (plugins/módulos) para cada destino.

**Core (Java):**
- Validación de requerimientos por canal
- Selección de rendition (o verificación de disponibilidad)
- Orquestación de publicación por canal
- Normalización de errores
- Auditoría y métricas
- Políticas de reintento / idempotencia

**Connector por canal (ejemplos):**
- `CMSConnector` (REST)
- `TVConnector` (MOS/FTP/SFTP según integración)
- `YouTubeConnector` (API)
- `MetaConnector` (Graph API)
- `XConnector` (API)

Cada conector encapsula autenticación, formato de request y protocolo de subida. El core no conoce detalles específicos del canal, solo invoca una interfaz común.

### 9.4.2. "Distribution Plan"

Antes de publicar, el worker construye un plan de distribución por canal, que incluye:
- Canal destino
- Rendition requerida (tipo/codec/resolución)
- Metadata a publicar
- Políticas (embargo ya resuelto por Temporal, tags, etc.)
- Clave de idempotencia

Este plan evita decisiones improvisadas durante el upload y hace que la ejecución sea repetible/auditable.

---

## 9.5. Flujo Paso a Paso (desde el Workflow hasta el Canal Externo)

### Paso 1 — Activación de Distribución en el Workflow

Cuando el workflow llega al punto de distribución (ya se cumplieron dependencias como transcode/IA y condiciones temporales), Temporal agenda la ejecución en `q-distribution`.

### Paso 2 — Inicio de Activity y Validaciones

El Dist Worker toma la Activity y realiza:
1. Carga de metadata del asset y configuración del destino
2. Chequeo de precondiciones por canal:
   - ¿Existe rendition compatible?
   - ¿Metadata obligatoria completa?
   - ¿Credenciales disponibles?
3. Genera idempotency key por (asset, canal, versión/rendition)

### Paso 3 — Publicación por Canal (Modelo Recomendado)

Para evitar que un fallo en un canal bloquee todos, se recomienda **fan-out controlado**:

**Opción A (más robusta)**: una Activity por canal (`PublishToYouTube`, `PublishToCMS`, etc.), ejecutables en paralelo (cada una con su retry policy).

**Opción B (más simple)**: una Activity única que itera canales, guardando estado por canal y tolerando fallos parciales.

En ambos casos, el resultado por canal incluye `externalId` (id del post/video remoto) y estado.

### Paso 4 — Registro de Resultado y Trazabilidad

Al finalizar (por canal o global):
- Se registra el resultado (éxito/falla) con timestamp, canal, rendition usada, externalId y error normalizado
- Se emite un estado final para que el resto del sistema (UI/editor) muestre "Publicado en X / Falló en Y"

---

## 9.6. Distribución en Streaming (sin Descarga Completa)

### 9.6.1. Principio

El Dist Worker **no descarga el archivo completo** ni genera temporales locales. La publicación se realiza por **streaming**, leyendo la rendition desde el storage y escribiéndola directamente al canal externo.

### 9.6.2. Implementación Conceptual

**Source stream**: abrir un stream de lectura hacia la rendition en hot storage:
- Si está en FS compartido: lectura por `InputStream`/canal
- Si está en object storage: stream HTTP GET (idealmente con URL firmada)

**Sink stream**: el conector abre el canal de subida:
- HTTP streaming/chunked (APIs)
- SFTP/FTP stream (playout legacy)
- Uploads resumibles cuando el canal lo permite

**Pipe**: se transfieren bytes en chunks, sin persistirlos localmente.

### 9.6.3. Beneficio

- Evita I/O doble y almacenamiento temporal
- Reduce tiempo de publicación para assets grandes
- Disminuye presión sobre disco local (pods efímeros)
- Permite mayor throughput en picos

### 9.6.4. Resolución del Problema de Archivos Temporales Locales

**Problema**: Los pods de Kubernetes son efímeros; cualquier archivo escrito localmente se pierde si el pod muere.

**Solución**: **Streaming directo desde S3** sin escribir archivos intermedios.

**Implementación en Java**:

```java
// Ejemplo: Streaming desde S3 hacia canal externo
public class DistributionActivityImpl implements DistributionActivity {
    
    private final S3Client s3Client;
    private final ChannelConnectorRegistry connectors;
    
    @Override
    public DistributionResult publishToChannel(
        String assetId, 
        String renditionKey, 
        DistributionChannel channel
    ) {
        // 1. Obtener conector del canal
        ChannelConnector connector = connectors.get(channel.getType());
        
        // 2. Abrir stream de lectura desde S3 (sin descargar)
        S3Object s3Object = s3Client.getObject(
            GetObjectRequest.builder()
                .bucket("dam-proxies")
                .key(renditionKey)
                .build()
        );
        
        InputStream sourceStream = s3Object;
        
        // 3. Abrir stream de escritura al canal externo
        OutputStream sinkStream = connector.openUploadStream(
            channel.getConfig(),
            assetId
        );
        
        // 4. Transferir bytes en chunks (pipe)
        try {
            byte[] buffer = new byte[8192]; // 8KB chunks
            int bytesRead;
            long totalBytes = 0;
            
            while ((bytesRead = sourceStream.read(buffer)) != -1) {
                sinkStream.write(buffer, 0, bytesRead);
                totalBytes += bytesRead;
                
                // Emitir heartbeat periódico para transferencias largas
                if (totalBytes % (10 * 1024 * 1024) == 0) { // cada 10MB
                    Activity.getExecutionContext().heartbeat(
                        new DistributionProgress(totalBytes)
                    );
                }
            }
            
            sinkStream.flush();
            
            // 5. Obtener externalId del conector
            String externalId = connector.completeUpload();
            
            return DistributionResult.success(channel, externalId);
            
        } finally {
            // Cerrar streams (no hay archivos temporales que limpiar)
            sourceStream.close();
            sinkStream.close();
        }
    }
}
```

**Ventajas de este enfoque**:
- **Sin archivos temporales**: todo se maneja en memoria (buffers pequeños)
- **Resiliente**: si el pod muere, Temporal reintenta y el stream se reinicia desde S3
- **Eficiente**: no hay I/O doble (lectura + escritura a disco local)
- **Escalable**: múltiples pods pueden hacer streaming simultáneo sin competir por disco

---

## 9.7. Reintentos, Idempotencia y Reanudación (con Temporal)

### 9.7.1. Políticas de Reintento

Cada Activity de distribución se configura con exponential backoff, límite de intentos y clasificación de errores:

- **Transitorios** (timeouts, 5xx, rate limit): reintentar automáticamente
- **Permanentes** (credenciales inválidas, formato no soportado): fallar rápido y marcar el canal como error "no recuperable"

Esto evita saturar canales externos con reintentos infinitos y mantiene estabilidad.

**Configuración en Temporal Workflow**:

```java
@WorkflowInterface
public interface DistributionWorkflow {
    @WorkflowMethod
    DistributionResult distribute(AssetDistributionRequest request);
}

public class DistributionWorkflowImpl implements DistributionWorkflow {
    
    @Override
    public DistributionResult distribute(AssetDistributionRequest request) {
        // Configurar retry policy por canal
        RetryOptions retryOptions = RetryOptions.newBuilder()
            .setInitialInterval(Duration.ofSeconds(1))
            .setBackoffCoefficient(2.0)
            .setMaximumInterval(Duration.ofMinutes(5))
            .setMaximumAttempts(5)
            .setDoNotRetry(IllegalArgumentException.class) // errores permanentes
            .build();
        
        ActivityOptions activityOptions = ActivityOptions.newBuilder()
            .setStartToCloseTimeout(Duration.ofHours(2)) // para archivos grandes
            .setHeartbeatTimeout(Duration.ofMinutes(5))
            .setRetryOptions(retryOptions)
            .build();
        
        DistributionActivity activity = 
            Workflow.newActivityStub(DistributionActivity.class, activityOptions);
        
        return activity.publishToChannel(
            request.getAssetId(),
            request.getRenditionKey(),
            request.getChannel()
        );
    }
}
```

### 9.7.2. Idempotencia (Evitar Duplicados)

Como Temporal puede reintentar y además puede haber re-ejecuciones por fallos de infraestructura, se garantiza idempotencia:

- **Idempotency key estable** por asset+canal+rendition
- **Registro de externalId** por canal: si ya existe, se evita re-publicar (o se ejecuta "update" según el canal)

**Implementación**:

```java
public class DistributionActivityImpl implements DistributionActivity {
    
    private final DistributionRepository repository;
    
    @Override
    public DistributionResult publishToChannel(
        String assetId, 
        String renditionKey, 
        DistributionChannel channel
    ) {
        // Generar idempotency key
        String idempotencyKey = String.format(
            "%s:%s:%s:%s",
            assetId,
            channel.getType(),
            channel.getId(),
            renditionKey
        );
        
        // Verificar si ya existe publicación exitosa
        DistributionRecord existing = repository.findByKey(idempotencyKey);
        if (existing != null && existing.getStatus() == DistributionStatus.SUCCESS) {
            return DistributionResult.alreadyPublished(
                channel,
                existing.getExternalId()
            );
        }
        
        // Proceder con publicación...
        // Al finalizar, registrar externalId
        repository.save(new DistributionRecord(
            idempotencyKey,
            assetId,
            channel,
            externalId,
            DistributionStatus.SUCCESS
        ));
    }
}
```

### 9.7.3. Reanudación en Transfers Largos (Heartbeats)

Para subidas largas, el worker emite heartbeats. Si el pod muere:
- Temporal reprograma la Activity
- Si el canal soporta subida resumible, se retoma desde `resumeToken`/offset
- Si no soporta, se reintenta desde cero, manteniendo idempotencia para no duplicar publicaciones

---

## 9.8. Escalabilidad Automática (KEDA + Temporal Scaler aplicado a Distribución)

La distribución usa `q-distribution`, por lo que el escalado se hace por backlog real de esa task queue, no por CPU/RAM.

**Ventaja**: al ser I/O bound, la carga se refleja mejor en "cantidad de tasks pendientes" que en CPU.

**Parámetros típicos**:
- `targetQueueSize` más alto que transcoding (porque cada task suele ser más corta)
- `cooldownPeriod` para evitar "thrashing" cuando baja el pico

Esto permite responder a eventos de alta demanda (picos de publicación) sin sobredimensionar permanentemente.

**Configuración KEDA**:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: distribution-workers-scaler
spec:
  scaleTargetRef:
    name: distribution-worker-deployment
  pollingInterval: 5
  cooldownPeriod: 60
  minReplicaCount: 1
  maxReplicaCount: 50
  advanced:
    horizontalPodAutoscalerConfig:
      behavior:
        scaleDown:
          stabilizationWindowSeconds: 120
  triggers:
    - type: temporal
      metadata:
        endpoint: temporal-frontend.temporal.svc.cluster.local:7233
        namespace: dam
        taskQueue: q-distribution
        queueTypes: activity
        targetQueueSize: "10"  # Más alto que transcoding (tareas más cortas)
        activationTargetQueueSize: "0"
```

---

## 9.9. Seguridad, Secretos y Auditoría

### 9.9.1. Gestor de Secretos: HashiCorp Vault

**Decisión**: Se utiliza **HashiCorp Vault** como gestor de secretos centralizado, desplegado en Kubernetes.

**Justificación**:
- **Self-hosted**: cumple con el requerimiento de evitar vendor lock-in
- **Integración con Kubernetes**: soporta inyección de secretos vía Service Accounts y CSI Driver
- **Rotación automática**: permite rotar credenciales sin downtime
- **Auditoría**: registra todos los accesos a secretos
- **Múltiples backends**: soporta secretos estáticos, dinámicos (credenciales generadas bajo demanda) y certificados

**Alternativa (más simple)**: Kubernetes Secrets + Sealed Secrets (para secretos en Git)

**Arquitectura de Vault**:

```yaml
# Ejemplo: Despliegue de Vault en Kubernetes
apiVersion: v1
kind: ServiceAccount
metadata:
  name: distribution-worker
  namespace: dam
---
apiVersion: v1
kind: Secret
metadata:
  name: vault-token
  namespace: dam
type: Opaque
stringData:
  token: <vault-token>
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: distribution-worker
  namespace: dam
spec:
  template:
    spec:
      serviceAccountName: distribution-worker
      containers:
      - name: worker
        image: dam/distribution-worker:latest
        env:
        - name: VAULT_ADDR
          value: "http://vault.vault.svc.cluster.local:8200"
        - name: VAULT_TOKEN
          valueFrom:
            secretKeyRef:
              name: vault-token
              key: token
```

**Uso en el Worker (Java)**:

```java
public class VaultSecretProvider implements SecretProvider {
    
    private final VaultClient vaultClient;
    
    @Override
    public ChannelCredentials getCredentials(String channelType, String channelId) {
        // Leer secretos desde Vault
        String path = String.format("secret/data/channels/%s/%s", channelType, channelId);
        
        VaultResponse response = vaultClient.read(path);
        
        return ChannelCredentials.builder()
            .apiKey(response.getData().get("api_key"))
            .apiSecret(response.getData().get("api_secret"))
            .accessToken(response.getData().get("access_token"))
            .build();
    }
}
```

**Estructura de Secretos en Vault**:

```
secret/
  channels/
    youtube/
      channel-1/
        api_key: <key>
        api_secret: <secret>
        refresh_token: <token>
    facebook/
      page-1/
        access_token: <token>
    cms/
      wordpress-1/
        username: <user>
        password: <pass>
```

### 9.9.2. Mínimo Privilegio

Cada conector usa credenciales acotadas al canal correspondiente. Vault permite políticas de acceso granulares:

```hcl
# Ejemplo: Política Vault para Distribution Worker
path "secret/data/channels/*" {
  capabilities = ["read"]
}

path "secret/data/channels/youtube/*" {
  capabilities = ["read"]
}

path "secret/data/channels/facebook/*" {
  capabilities = ["read"]
}
```

### 9.9.3. Auditoría

Registrar quién/qué/cuándo/dónde:
- `assetId`, usuario/campaña, canal, rendition, `externalId`, estado, error, timestamps

Esto permite trazabilidad operativa y post-mortem de incidentes.

**Modelo de Datos**:

```sql
CREATE TABLE asset_distributions (
    id UUID PRIMARY KEY,
    asset_id UUID NOT NULL REFERENCES assets(id),
    channel_type VARCHAR(50) NOT NULL,
    channel_id VARCHAR(100) NOT NULL,
    rendition_key VARCHAR(500) NOT NULL,
    external_id VARCHAR(500), -- ID en el sistema externo
    status VARCHAR(20) NOT NULL, -- PENDING, SUCCESS, FAILED
    error_message TEXT,
    started_at TIMESTAMP NOT NULL,
    completed_at TIMESTAMP,
    created_by UUID REFERENCES users(id),
    idempotency_key VARCHAR(500) UNIQUE NOT NULL
);

CREATE INDEX idx_distributions_asset ON asset_distributions(asset_id);
CREATE INDEX idx_distributions_channel ON asset_distributions(channel_type, channel_id);
CREATE INDEX idx_distributions_status ON asset_distributions(status);
```

---

## 9.10. Restricciones por Statelessness y Adaptación

Dado que los pods/containers son efímeros:

**Prohibido**: depender de archivos temporales locales para sostener el proceso.

**Solución**: toda evidencia de progreso debe estar:
- En Temporal (estado del workflow + heartbeats), y/o
- En almacenamiento externo (object storage / volúmenes persistentes si hiciera falta excepcionalmente)

**La estrategia de streaming directo es consistente con esta restricción**: evita escribir archivos intermedios locales y reduce riesgo de pérdida ante reinicios.

---

## 9.11. Ejemplo Completo: Publicación a YouTube

### 9.11.1. Workflow (Temporal)

```java
@WorkflowInterface
public interface AssetDistributionWorkflow {
    @WorkflowMethod
    DistributionResult distributeAsset(AssetDistributionRequest request);
}

public class AssetDistributionWorkflowImpl implements AssetDistributionWorkflow {
    
    @Override
    public DistributionResult distributeAsset(AssetDistributionRequest request) {
        // 1. Esperar a que las renditions estén listas
        Workflow.await(() -> areRenditionsReady(request.getAssetId()));
        
        // 2. Verificar embargo/ventana temporal (si aplica)
        if (request.hasEmbargo()) {
            Workflow.sleep(request.getEmbargoUntil());
        }
        
        // 3. Publicar a cada canal en paralelo
        List<CompletablePromise<ChannelDistributionResult>> promises = 
            request.getChannels().stream()
                .map(channel -> Async.function(
                    this::publishToChannel,
                    request.getAssetId(),
                    request.getRenditionKey(channel),
                    channel
                ))
                .collect(Collectors.toList());
        
        // 4. Recopilar resultados
        List<ChannelDistributionResult> results = new ArrayList<>();
        for (CompletablePromise<ChannelDistributionResult> promise : promises) {
            try {
                results.add(promise.get());
            } catch (Exception e) {
                results.add(ChannelDistributionResult.failed(e));
            }
        }
        
        return DistributionResult.fromChannelResults(results);
    }
    
    private ChannelDistributionResult publishToChannel(
        String assetId,
        String renditionKey,
        DistributionChannel channel
    ) {
        ActivityOptions options = ActivityOptions.newBuilder()
            .setStartToCloseTimeout(Duration.ofHours(2))
            .setHeartbeatTimeout(Duration.ofMinutes(5))
            .setRetryOptions(RetryOptions.newBuilder()
                .setInitialInterval(Duration.ofSeconds(1))
                .setBackoffCoefficient(2.0)
                .setMaximumAttempts(5)
                .build())
            .build();
        
        DistributionActivity activity = 
            Workflow.newActivityStub(DistributionActivity.class, options);
        
        return activity.publishToChannel(assetId, renditionKey, channel);
    }
}
```

### 9.11.2. Activity (Distribution Worker)

```java
@ActivityInterface
public interface DistributionActivity {
    ChannelDistributionResult publishToChannel(
        String assetId,
        String renditionKey,
        DistributionChannel channel
    );
}

public class DistributionActivityImpl implements DistributionActivity {
    
    private final S3Client s3Client;
    private final ChannelConnectorRegistry connectors;
    private final SecretProvider secretProvider;
    private final DistributionRepository repository;
    
    @Override
    public ChannelDistributionResult publishToChannel(
        String assetId,
        String renditionKey,
        DistributionChannel channel
    ) {
        ActivityExecutionContext ctx = Activity.getExecutionContext();
        
        try {
            // 1. Verificar idempotencia
            String idempotencyKey = generateIdempotencyKey(assetId, channel, renditionKey);
            DistributionRecord existing = repository.findByKey(idempotencyKey);
            if (existing != null && existing.getStatus() == DistributionStatus.SUCCESS) {
                return ChannelDistributionResult.alreadyPublished(
                    channel,
                    existing.getExternalId()
                );
            }
            
            // 2. Obtener credenciales desde Vault
            ChannelCredentials credentials = secretProvider.getCredentials(
                channel.getType(),
                channel.getId()
            );
            
            // 3. Obtener conector
            ChannelConnector connector = connectors.get(channel.getType());
            connector.configure(credentials);
            
            // 4. Abrir stream desde S3
            S3Object s3Object = s3Client.getObject(
                GetObjectRequest.builder()
                    .bucket("dam-proxies")
                    .key(renditionKey)
                    .build()
            );
            
            // 5. Publicar por streaming
            String externalId = connector.uploadStream(
                s3Object,
                assetId,
                channel.getMetadata()
            );
            
            // 6. Registrar éxito
            repository.save(new DistributionRecord(
                idempotencyKey,
                assetId,
                channel,
                externalId,
                DistributionStatus.SUCCESS
            ));
            
            return ChannelDistributionResult.success(channel, externalId);
            
        } catch (Exception e) {
            // Registrar fallo
            repository.save(new DistributionRecord(
                idempotencyKey,
                assetId,
                channel,
                null,
                DistributionStatus.FAILED,
                e.getMessage()
            ));
            
            throw e; // Temporal reintentará si es transitorio
        }
    }
}
```

### 9.11.3. Conector YouTube (Ejemplo)

```java
public class YouTubeConnector implements ChannelConnector {
    
    private YouTube youtube;
    private ChannelCredentials credentials;
    
    @Override
    public void configure(ChannelCredentials credentials) {
        this.credentials = credentials;
        this.youtube = new YouTube.Builder(
            GoogleNetHttpTransport.newTrustedTransport(),
            JacksonFactory.getDefaultInstance(),
            new GoogleCredential()
                .setAccessToken(credentials.getAccessToken())
        ).build();
    }
    
    @Override
    public String uploadStream(
        InputStream sourceStream,
        String assetId,
        Map<String, String> metadata
    ) throws IOException {
        // YouTube requiere multipart upload con metadata
        Video video = new Video();
        VideoSnippet snippet = new VideoSnippet();
        snippet.setTitle(metadata.get("title"));
        snippet.setDescription(metadata.get("description"));
        snippet.setTags(metadata.getOrDefault("tags", "").split(","));
        video.setSnippet(snippet);
        
        VideoStatus status = new VideoStatus();
        status.setPrivacyStatus("unlisted"); // o "public", "private"
        video.setStatus(status);
        
        // Upload por streaming
        YouTube.Videos.Insert request = youtube.videos()
            .insert("snippet,status", video, 
                new InputStreamContent("video/*", sourceStream));
        
        Video uploadedVideo = request.execute();
        
        return uploadedVideo.getId();
    }
}
```

---

## 9.12. Referencias

- Temporal Retry Policies: https://docs.temporal.io/encyclopedia/retry-policies
- Temporal Heartbeats: https://docs.temporal.io/encyclopedia/detecting-activity-failures
- HashiCorp Vault Kubernetes Integration: https://www.vaultproject.io/docs/platform/k8s
- AWS S3 Streaming Upload: https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/examples-s3-transfermanager.html
- YouTube Data API v3: https://developers.google.com/youtube/v3/docs
- KEDA Temporal Scaler: https://keda.sh/docs/2.18/scalers/temporal/

