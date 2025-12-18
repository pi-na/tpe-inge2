# 9.X. Ejemplos de Código - Distribución Automatizada

Este documento contiene ejemplos concretos de código para la implementación de distribución automatizada.

---

## 9.X.1. Estructura del Proyecto Java

```
distribution-worker/
├── src/main/java/com/dam/distribution/
│   ├── workflow/
│   │   ├── AssetDistributionWorkflow.java
│   │   └── AssetDistributionWorkflowImpl.java
│   ├── activity/
│   │   ├── DistributionActivity.java
│   │   └── DistributionActivityImpl.java
│   ├── connector/
│   │   ├── ChannelConnector.java
│   │   ├── YouTubeConnector.java
│   │   ├── FacebookConnector.java
│   │   ├── CMSConnector.java
│   │   └── TVConnector.java
│   ├── model/
│   │   ├── DistributionChannel.java
│   │   ├── DistributionResult.java
│   │   ├── ChannelCredentials.java
│   │   └── DistributionRecord.java
│   ├── repository/
│   │   └── DistributionRepository.java
│   ├── secrets/
│   │   ├── SecretProvider.java
│   │   └── VaultSecretProvider.java
│   └── config/
│       └── DistributionConfig.java
├── src/main/resources/
│   └── application.yml
└── pom.xml
```

---

## 9.X.2. Modelos de Datos

### DistributionChannel.java

```java
package com.dam.distribution.model;

import java.util.Map;

public class DistributionChannel {
    private String type; // "youtube", "facebook", "cms", "tv"
    private String id; // identificador del canal específico
    private Map<String, String> config; // configuración específica del canal
    private Map<String, String> metadata; // metadata a publicar
    
    // Getters y setters
    public String getType() { return type; }
    public void setType(String type) { this.type = type; }
    
    public String getId() { return id; }
    public void setId(String id) { this.id = id; }
    
    public Map<String, String> getConfig() { return config; }
    public void setConfig(Map<String, String> config) { this.config = config; }
    
    public Map<String, String> getMetadata() { return metadata; }
    public void setMetadata(Map<String, String> metadata) { this.metadata = metadata; }
}
```

### DistributionResult.java

```java
package com.dam.distribution.model;

import java.util.List;
import java.util.stream.Collectors;

public class DistributionResult {
    private List<ChannelDistributionResult> channelResults;
    private int successCount;
    private int failureCount;
    
    public static DistributionResult fromChannelResults(
        List<ChannelDistributionResult> results
    ) {
        DistributionResult result = new DistributionResult();
        result.channelResults = results;
        result.successCount = (int) results.stream()
            .filter(r -> r.getStatus() == ChannelStatus.SUCCESS)
            .count();
        result.failureCount = results.size() - result.successCount;
        return result;
    }
    
    // Getters
    public List<ChannelDistributionResult> getChannelResults() {
        return channelResults;
    }
    
    public int getSuccessCount() { return successCount; }
    public int getFailureCount() { return failureCount; }
    
    public boolean isAllSuccessful() {
        return failureCount == 0;
    }
}
```

### ChannelDistributionResult.java

```java
package com.dam.distribution.model;

public class ChannelDistributionResult {
    private DistributionChannel channel;
    private ChannelStatus status;
    private String externalId; // ID en el sistema externo
    private String errorMessage;
    
    public static ChannelDistributionResult success(
        DistributionChannel channel,
        String externalId
    ) {
        ChannelDistributionResult result = new ChannelDistributionResult();
        result.channel = channel;
        result.status = ChannelStatus.SUCCESS;
        result.externalId = externalId;
        return result;
    }
    
    public static ChannelDistributionResult failed(
        DistributionChannel channel,
        String errorMessage
    ) {
        ChannelDistributionResult result = new ChannelDistributionResult();
        result.channel = channel;
        result.status = ChannelStatus.FAILED;
        result.errorMessage = errorMessage;
        return result;
    }
    
    public static ChannelDistributionResult alreadyPublished(
        DistributionChannel channel,
        String externalId
    ) {
        ChannelDistributionResult result = new ChannelDistributionResult();
        result.channel = channel;
        result.status = ChannelStatus.ALREADY_PUBLISHED;
        result.externalId = externalId;
        return result;
    }
    
    // Getters
    public DistributionChannel getChannel() { return channel; }
    public ChannelStatus getStatus() { return status; }
    public String getExternalId() { return externalId; }
    public String getErrorMessage() { return errorMessage; }
}

enum ChannelStatus {
    SUCCESS, FAILED, ALREADY_PUBLISHED
}
```

---

## 9.X.3. Interfaz del Conector

### ChannelConnector.java

```java
package com.dam.distribution.connector;

import com.dam.distribution.model.ChannelCredentials;
import java.io.InputStream;
import java.util.Map;

public interface ChannelConnector {
    /**
     * Configura el conector con las credenciales del canal
     */
    void configure(ChannelCredentials credentials);
    
    /**
     * Sube un stream al canal externo
     * @param sourceStream Stream de lectura desde S3
     * @param assetId ID del asset
     * @param metadata Metadata a publicar (título, descripción, tags, etc.)
     * @return ID del recurso creado en el sistema externo
     */
    String uploadStream(
        InputStream sourceStream,
        String assetId,
        Map<String, String> metadata
    ) throws Exception;
    
    /**
     * Verifica si el canal soporta uploads resumibles
     */
    default boolean supportsResumableUpload() {
        return false;
    }
    
    /**
     * Retoma un upload desde un token/offset (si es soportado)
     */
    default String resumeUpload(
        String resumeToken,
        InputStream sourceStream,
        long offset
    ) throws Exception {
        throw new UnsupportedOperationException(
            "Resumable upload not supported"
        );
    }
}
```

---

## 9.X.4. Implementación de Conectores

### YouTubeConnector.java

```java
package com.dam.distribution.connector;

import com.dam.distribution.model.ChannelCredentials;
import com.google.api.client.googleapis.auth.oauth2.GoogleCredential;
import com.google.api.client.googleapis.javanet.GoogleNetHttpTransport;
import com.google.api.client.http.InputStreamContent;
import com.google.api.client.json.jackson2.JacksonFactory;
import com.google.api.services.youtube.YouTube;
import com.google.api.services.youtube.model.Video;
import com.google.api.services.youtube.model.VideoSnippet;
import com.google.api.services.youtube.model.VideoStatus;

import java.io.InputStream;
import java.util.Map;

public class YouTubeConnector implements ChannelConnector {
    
    private YouTube youtube;
    private ChannelCredentials credentials;
    
    @Override
    public void configure(ChannelCredentials credentials) {
        this.credentials = credentials;
        try {
            GoogleCredential credential = new GoogleCredential()
                .setAccessToken(credentials.getAccessToken());
            
            this.youtube = new YouTube.Builder(
                GoogleNetHttpTransport.newTrustedTransport(),
                JacksonFactory.getDefaultInstance(),
                credential
            ).setApplicationName("DAM-Distribution")
             .build();
        } catch (Exception e) {
            throw new RuntimeException("Failed to configure YouTube connector", e);
        }
    }
    
    @Override
    public String uploadStream(
        InputStream sourceStream,
        String assetId,
        Map<String, String> metadata
    ) throws Exception {
        // Construir objeto Video con metadata
        Video video = new Video();
        
        VideoSnippet snippet = new VideoSnippet();
        snippet.setTitle(metadata.getOrDefault("title", "Untitled"));
        snippet.setDescription(metadata.getOrDefault("description", ""));
        
        String tagsStr = metadata.getOrDefault("tags", "");
        if (!tagsStr.isEmpty()) {
            snippet.setTags(java.util.Arrays.asList(tagsStr.split(",")));
        }
        
        video.setSnippet(snippet);
        
        VideoStatus status = new VideoStatus();
        status.setPrivacyStatus(
            metadata.getOrDefault("privacy", "unlisted")
        );
        video.setStatus(status);
        
        // Upload por streaming
        YouTube.Videos.Insert request = youtube.videos()
            .insert("snippet,status", video,
                new InputStreamContent("video/*", sourceStream));
        
        // Ejecutar upload (streaming)
        Video uploadedVideo = request.execute();
        
        return uploadedVideo.getId();
    }
    
    @Override
    public boolean supportsResumableUpload() {
        return true; // YouTube soporta uploads resumibles
    }
}
```

### FacebookConnector.java

```java
package com.dam.distribution.connector;

import com.dam.distribution.model.ChannelCredentials;
import com.fasterxml.jackson.databind.ObjectMapper;
import okhttp3.*;

import java.io.IOException;
import java.io.InputStream;
import java.util.Map;

public class FacebookConnector implements ChannelConnector {
    
    private OkHttpClient httpClient;
    private ChannelCredentials credentials;
    private ObjectMapper objectMapper;
    
    public FacebookConnector() {
        this.httpClient = new OkHttpClient();
        this.objectMapper = new ObjectMapper();
    }
    
    @Override
    public void configure(ChannelCredentials credentials) {
        this.credentials = credentials;
    }
    
    @Override
    public String uploadStream(
        InputStream sourceStream,
        String assetId,
        Map<String, String> metadata
    ) throws Exception {
        // Facebook Graph API requiere multipart form-data
        String pageId = credentials.getConfig().get("page_id");
        String uploadUrl = String.format(
            "https://graph.facebook.com/v18.0/%s/videos",
            pageId
        );
        
        // Leer stream completo a memoria (Facebook no soporta streaming directo)
        // Nota: Para archivos muy grandes, considerar chunked upload
        byte[] videoData = sourceStream.readAllBytes();
        
        RequestBody requestBody = new MultipartBody.Builder()
            .setType(MultipartBody.FORM)
            .addFormDataPart("access_token", credentials.getAccessToken())
            .addFormDataPart("title", metadata.getOrDefault("title", ""))
            .addFormDataPart("description", metadata.getOrDefault("description", ""))
            .addFormDataPart("source_video_file",
                "video.mp4",
                RequestBody.create(
                    videoData,
                    MediaType.parse("video/mp4")
                ))
            .build();
        
        Request request = new Request.Builder()
            .url(uploadUrl)
            .post(requestBody)
            .build();
        
        try (Response response = httpClient.newCall(request).execute()) {
            if (!response.isSuccessful()) {
                throw new IOException(
                    "Facebook upload failed: " + response.body().string()
                );
            }
            
            // Parsear respuesta JSON
            String responseBody = response.body().string();
            Map<String, Object> result = objectMapper.readValue(
                responseBody,
                Map.class
            );
            
            return (String) result.get("id");
        }
    }
}
```

### CMSConnector.java (REST API genérico)

```java
package com.dam.distribution.connector;

import com.dam.distribution.model.ChannelCredentials;
import okhttp3.*;

import java.io.IOException;
import java.io.InputStream;
import java.util.Map;

public class CMSConnector implements ChannelConnector {
    
    private OkHttpClient httpClient;
    private ChannelCredentials credentials;
    private String baseUrl;
    
    public CMSConnector() {
        this.httpClient = new OkHttpClient();
    }
    
    @Override
    public void configure(ChannelCredentials credentials) {
        this.credentials = credentials;
        this.baseUrl = credentials.getConfig().get("base_url");
    }
    
    @Override
    public String uploadStream(
        InputStream sourceStream,
        String assetId,
        Map<String, String> metadata
    ) throws Exception {
        // Leer stream a bytes (o usar streaming si el CMS lo soporta)
        byte[] fileData = sourceStream.readAllBytes();
        
        // Construir request multipart
        RequestBody requestBody = new MultipartBody.Builder()
            .setType(MultipartBody.FORM)
            .addFormDataPart("file", metadata.getOrDefault("filename", "asset"),
                RequestBody.create(
                    fileData,
                    MediaType.parse(metadata.getOrDefault("mime_type", "application/octet-stream"))
                ))
            .addFormDataPart("title", metadata.getOrDefault("title", ""))
            .addFormDataPart("description", metadata.getOrDefault("description", ""))
            .build();
        
        Request request = new Request.Builder()
            .url(baseUrl + "/api/v1/media/upload")
            .header("Authorization", "Bearer " + credentials.getAccessToken())
            .post(requestBody)
            .build();
        
        try (Response response = httpClient.newCall(request).execute()) {
            if (!response.isSuccessful()) {
                throw new IOException(
                    "CMS upload failed: " + response.body().string()
                );
            }
            
            // Asumir que la respuesta contiene un ID
            String responseBody = response.body().string();
            // Parsear JSON según el formato del CMS
            return extractIdFromResponse(responseBody);
        }
    }
    
    private String extractIdFromResponse(String responseBody) {
        // Implementar parsing según el formato del CMS
        // Ejemplo: {"id": "123", ...}
        return responseBody; // Simplificado
    }
}
```

---

## 9.X.5. Secret Provider (Vault)

### VaultSecretProvider.java

```java
package com.dam.distribution.secrets;

import com.dam.distribution.model.ChannelCredentials;
import com.bettercloud.vault.Vault;
import com.bettercloud.vault.VaultConfig;
import com.bettercloud.vault.response.LogicalResponse;

import java.util.Map;

public class VaultSecretProvider implements SecretProvider {
    
    private final Vault vault;
    
    public VaultSecretProvider(String vaultAddr, String vaultToken) {
        try {
            VaultConfig config = new VaultConfig()
                .address(vaultAddr)
                .token(vaultToken)
                .build();
            this.vault = new Vault(config);
        } catch (Exception e) {
            throw new RuntimeException("Failed to initialize Vault client", e);
        }
    }
    
    @Override
    public ChannelCredentials getCredentials(String channelType, String channelId) {
        try {
            String path = String.format("secret/data/channels/%s/%s", channelType, channelId);
            LogicalResponse response = vault.logical().read(path);
            
            Map<String, Object> data = response.getData();
            
            return ChannelCredentials.builder()
                .accessToken((String) data.get("access_token"))
                .apiKey((String) data.get("api_key"))
                .apiSecret((String) data.get("api_secret"))
                .refreshToken((String) data.get("refresh_token"))
                .config((Map<String, String>) data.get("config"))
                .build();
        } catch (Exception e) {
            throw new RuntimeException(
                String.format("Failed to get credentials for %s/%s", channelType, channelId),
                e
            );
        }
    }
}
```

---

## 9.X.6. Activity Implementation

### DistributionActivityImpl.java (Completo)

```java
package com.dam.distribution.activity;

import com.dam.distribution.connector.ChannelConnector;
import com.dam.distribution.connector.ChannelConnectorRegistry;
import com.dam.distribution.model.*;
import com.dam.distribution.repository.DistributionRepository;
import com.dam.distribution.secrets.SecretProvider;
import io.temporal.activity.Activity;
import io.temporal.activity.ActivityExecutionContext;
import software.amazon.awssdk.core.sync.RequestBody;
import software.amazon.awssdk.services.s3.S3Client;
import software.amazon.awssdk.services.s3.model.GetObjectRequest;
import software.amazon.awssdk.services.s3.model.GetObjectResponse;

import java.io.InputStream;
import java.util.UUID;

public class DistributionActivityImpl implements DistributionActivity {
    
    private final S3Client s3Client;
    private final ChannelConnectorRegistry connectors;
    private final SecretProvider secretProvider;
    private final DistributionRepository repository;
    private final String proxiesBucket;
    
    public DistributionActivityImpl(
        S3Client s3Client,
        ChannelConnectorRegistry connectors,
        SecretProvider secretProvider,
        DistributionRepository repository,
        String proxiesBucket
    ) {
        this.s3Client = s3Client;
        this.connectors = connectors;
        this.secretProvider = secretProvider;
        this.repository = repository;
        this.proxiesBucket = proxiesBucket;
    }
    
    @Override
    public ChannelDistributionResult publishToChannel(
        String assetId,
        String renditionKey,
        DistributionChannel channel
    ) {
        ActivityExecutionContext ctx = Activity.getExecutionContext();
        String idempotencyKey = null;
        
        try {
            // 1. Generar idempotency key
            idempotencyKey = generateIdempotencyKey(assetId, channel, renditionKey);
            
            // 2. Verificar idempotencia
            DistributionRecord existing = repository.findByKey(idempotencyKey);
            if (existing != null && existing.getStatus() == DistributionStatus.SUCCESS) {
                return ChannelDistributionResult.alreadyPublished(
                    channel,
                    existing.getExternalId()
                );
            }
            
            // 3. Obtener credenciales desde Vault
            ChannelCredentials credentials = secretProvider.getCredentials(
                channel.getType(),
                channel.getId()
            );
            
            // 4. Obtener conector
            ChannelConnector connector = connectors.get(channel.getType());
            connector.configure(credentials);
            
            // 5. Abrir stream desde S3 (streaming, sin descargar completo)
            GetObjectRequest getRequest = GetObjectRequest.builder()
                .bucket(proxiesBucket)
                .key(renditionKey)
                .build();
            
            // 6. Publicar por streaming
            String externalId;
            try (InputStream sourceStream = s3Client.getObjectAsInputStream(getRequest)) {
                // Emitir heartbeat periódico durante transferencias largas
                ctx.heartbeat(new DistributionProgress(0, "Starting upload"));
                
                externalId = connector.uploadStream(
                    sourceStream,
                    assetId,
                    channel.getMetadata()
                );
            }
            
            // 7. Registrar éxito
            DistributionRecord record = new DistributionRecord();
            record.setId(UUID.randomUUID());
            record.setAssetId(assetId);
            record.setChannelType(channel.getType());
            record.setChannelId(channel.getId());
            record.setRenditionKey(renditionKey);
            record.setExternalId(externalId);
            record.setStatus(DistributionStatus.SUCCESS);
            record.setIdempotencyKey(idempotencyKey);
            
            repository.save(record);
            
            return ChannelDistributionResult.success(channel, externalId);
            
        } catch (Exception e) {
            // Registrar fallo
            if (idempotencyKey != null) {
                DistributionRecord record = new DistributionRecord();
                record.setId(UUID.randomUUID());
                record.setAssetId(assetId);
                record.setChannelType(channel.getType());
                record.setChannelId(channel.getId());
                record.setRenditionKey(renditionKey);
                record.setStatus(DistributionStatus.FAILED);
                record.setErrorMessage(e.getMessage());
                record.setIdempotencyKey(idempotencyKey);
                
                repository.save(record);
            }
            
            // Re-lanzar para que Temporal reintente si es transitorio
            throw new RuntimeException("Distribution failed", e);
        }
    }
    
    private String generateIdempotencyKey(
        String assetId,
        DistributionChannel channel,
        String renditionKey
    ) {
        return String.format(
            "%s:%s:%s:%s",
            assetId,
            channel.getType(),
            channel.getId(),
            renditionKey
        );
    }
}

class DistributionProgress {
    private long bytesTransferred;
    private String status;
    
    public DistributionProgress(long bytesTransferred, String status) {
        this.bytesTransferred = bytesTransferred;
        this.status = status;
    }
    
    // Getters
    public long getBytesTransferred() { return bytesTransferred; }
    public String getStatus() { return status; }
}
```

---

## 9.X.7. Configuración Kubernetes

### deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: distribution-worker
  namespace: dam
spec:
  replicas: 1
  selector:
    matchLabels:
      app: distribution-worker
  template:
    metadata:
      labels:
        app: distribution-worker
    spec:
      serviceAccountName: distribution-worker
      containers:
      - name: worker
        image: dam/distribution-worker:latest
        env:
        - name: TEMPORAL_HOST
          value: "temporal-frontend.temporal.svc.cluster.local:7233"
        - name: TEMPORAL_NAMESPACE
          value: "dam"
        - name: TEMPORAL_TASK_QUEUE
          value: "q-distribution"
        - name: S3_ENDPOINT
          value: "http://ceph-rgw.dam.svc.cluster.local:80"
        - name: S3_BUCKET_PROXIES
          value: "dam-proxies"
        - name: VAULT_ADDR
          value: "http://vault.vault.svc.cluster.local:8200"
        - name: VAULT_TOKEN
          valueFrom:
            secretKeyRef:
              name: vault-token
              key: token
        - name: DB_HOST
          value: "postgres.dam.svc.cluster.local"
        - name: DB_NAME
          value: "dam"
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        resources:
          requests:
            memory: "512Mi"
            cpu: "100m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
```

### scaledobject.yaml

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: distribution-workers-scaler
  namespace: dam
spec:
  scaleTargetRef:
    name: distribution-worker
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
        targetQueueSize: "10"
        activationTargetQueueSize: "0"
```

---

## 9.X.8. Configuración Vault

### vault-policy.hcl

```hcl
# Política para Distribution Worker
path "secret/data/channels/*" {
  capabilities = ["read"]
}

path "secret/data/channels/youtube/*" {
  capabilities = ["read"]
}

path "secret/data/channels/facebook/*" {
  capabilities = ["read"]
}

path "secret/data/channels/cms/*" {
  capabilities = ["read"]
}
```

### Ejemplo de secretos en Vault

```bash
# Crear secretos para YouTube
vault kv put secret/channels/youtube/channel-1 \
  api_key="YOUR_API_KEY" \
  api_secret="YOUR_API_SECRET" \
  access_token="YOUR_ACCESS_TOKEN" \
  refresh_token="YOUR_REFRESH_TOKEN"

# Crear secretos para Facebook
vault kv put secret/channels/facebook/page-1 \
  access_token="YOUR_ACCESS_TOKEN" \
  page_id="YOUR_PAGE_ID"

# Crear secretos para CMS
vault kv put secret/channels/cms/wordpress-1 \
  access_token="YOUR_API_TOKEN" \
  base_url="https://cms.example.com"
```

---

## 9.X.9. Referencias de Dependencias (pom.xml)

```xml
<dependencies>
    <!-- Temporal -->
    <dependency>
        <groupId>io.temporal</groupId>
        <artifactId>temporal-sdk</artifactId>
        <version>1.22.0</version>
    </dependency>
    
    <!-- AWS S3 SDK -->
    <dependency>
        <groupId>software.amazon.awssdk</groupId>
        <artifactId>s3</artifactId>
        <version>2.20.0</version>
    </dependency>
    
    <!-- Vault Client -->
    <dependency>
        <groupId>com.bettercloud</groupId>
        <artifactId>vault-java-driver</artifactId>
        <version>5.1.0</version>
    </dependency>
    
    <!-- YouTube API -->
    <dependency>
        <groupId>com.google.apis</groupId>
        <artifactId>google-api-services-youtube</artifactId>
        <version>v3-rev20230816-2.0.0</version>
    </dependency>
    
    <!-- OkHttp (para APIs REST) -->
    <dependency>
        <groupId>com.squareup.okhttp3</groupId>
        <artifactId>okhttp</artifactId>
        <version>4.11.0</version>
    </dependency>
    
    <!-- Jackson (JSON) -->
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>2.15.2</version>
    </dependency>
    
    <!-- PostgreSQL Driver -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <version>42.6.0</version>
    </dependency>
</dependencies>
```

