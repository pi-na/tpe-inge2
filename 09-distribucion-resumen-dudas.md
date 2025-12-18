# 9.X. Resumen: Respuestas a Dudas sobre Distribución Automatizada

Este documento responde directamente a las dudas planteadas sobre la distribución automatizada.

---

## Duda 1: "Para streaming, el flujo como queda? No sería que una vez que 'preparas' el archivo para streaming, tienes clientes que te lo piden, a través del CDN?"

### Respuesta

Hay una **confusión conceptual** entre dos conceptos diferentes:

#### 1. **Streaming (Serving Content)**
- **Qué es**: Servir contenido a clientes finales (navegadores, apps) desde el DAM a través de un CDN
- **Dirección**: Pull (Cliente → DAM/CDN)
- **Cuándo ocurre**: Cuando un usuario solicita ver un video/imagen en el frontend
- **Cómo funciona**:
  1. El DAM prepara proxies en formatos de streaming (HLS/DASH) durante el transcoding
  2. Los proxies se almacenan en `dam-proxies` (hot storage)
  3. El CDN interno sirve estos proxies bajo demanda mediante HTTP Range Requests
  4. Los clientes solicitan el contenido cuando lo necesitan
- **No requiere workflow de distribución**: Es un servicio HTTP estándar

#### 2. **Distribución (Publication to External Channels)**
- **Qué es**: Publicar contenido activamente hacia canales externos donde el DAM no controla el servidor
- **Dirección**: Push (DAM → Canal externo)
- **Cuándo ocurre**: Cuando un editor decide publicar un asset a YouTube, Facebook, CMS externo, etc.
- **Cómo funciona**:
  1. El workflow de distribución se activa (después de transcoding/IA)
  2. El Distribution Worker lee el archivo desde S3 y lo envía al canal externo
  3. El contenido queda almacenado en el sistema externo
  4. Los usuarios finales lo consumen desde ese sistema externo
- **Requiere workflow de distribución**: Es una actividad asíncrona gestionada por Temporal

### Conclusión

**Streaming** y **Distribución** son procesos **independientes**:

- **Streaming**: El DAM sirve contenido a sus propios usuarios (no requiere distribución)
- **Distribución**: El DAM publica contenido a sistemas externos (YouTube, Facebook, CMS, etc.)

La distribución automatizada (este documento) se enfoca en **publicar a canales externos**, no en servir streaming interno.

---

## Duda 2: "¿Cuál gestor de secretos?"

### Respuesta

Se propone **HashiCorp Vault** como gestor de secretos centralizado.

### Justificación

1. **Self-hosted**: Cumple con el requerimiento de evitar vendor lock-in
2. **Integración con Kubernetes**: Soporta inyección de secretos vía Service Accounts y CSI Driver
3. **Rotación automática**: Permite rotar credenciales sin downtime
4. **Auditoría**: Registra todos los accesos a secretos
5. **Múltiples backends**: Soporta secretos estáticos, dinámicos y certificados

### Alternativa (más simple)

Si Vault es demasiado complejo para el proyecto inicial, se puede usar:
- **Kubernetes Secrets** + **Sealed Secrets** (para secretos en Git)
- **Limitación**: No soporta rotación automática ni auditoría avanzada

### Estructura de Secretos en Vault

```
secret/
  channels/
    youtube/
      channel-1/
        api_key: <key>
        api_secret: <secret>
        access_token: <token>
        refresh_token: <token>
    facebook/
      page-1/
        access_token: <token>
        page_id: <id>
    cms/
      wordpress-1/
        access_token: <token>
        base_url: <url>
```

### Uso en el Worker

```java
// El worker obtiene credenciales desde Vault
ChannelCredentials credentials = secretProvider.getCredentials(
    channel.getType(),
    channel.getId()
);
```

**Ver sección 9.9.1 del documento principal para detalles completos.**

---

## Duda 3: "¿Cómo resolvemos lo de depender de archivos locales temporales para sostener el proceso?"

### Respuesta

**Solución: Streaming directo desde S3 sin escribir archivos intermedios.**

### Problema

Los pods de Kubernetes son efímeros; cualquier archivo escrito localmente se pierde si el pod muere.

### Solución Implementada

1. **Streaming directo desde S3**: El worker abre un stream de lectura desde S3 (sin descargar el archivo completo)
2. **Streaming directo al canal externo**: El conector abre un stream de escritura al canal externo
3. **Pipe en memoria**: Se transfieren bytes en chunks pequeños (8KB) sin persistirlos localmente
4. **Heartbeats periódicos**: Durante transferencias largas, el worker emite heartbeats a Temporal para mantener el estado

### Código de Ejemplo

```java
// 1. Abrir stream desde S3 (sin descargar)
S3Object s3Object = s3Client.getObject(
    GetObjectRequest.builder()
        .bucket("dam-proxies")
        .key(renditionKey)
        .build()
);

InputStream sourceStream = s3Object;

// 2. Abrir stream al canal externo
OutputStream sinkStream = connector.openUploadStream(...);

// 3. Transferir bytes en chunks (pipe en memoria)
byte[] buffer = new byte[8192]; // 8KB chunks
int bytesRead;
while ((bytesRead = sourceStream.read(buffer)) != -1) {
    sinkStream.write(buffer, 0, bytesRead);
    
    // Emitir heartbeat periódico
    if (totalBytes % (10 * 1024 * 1024) == 0) { // cada 10MB
        Activity.getExecutionContext().heartbeat(...);
    }
}

// 4. Cerrar streams (no hay archivos temporales que limpiar)
sourceStream.close();
sinkStream.close();
```

### Ventajas

- ✅ **Sin archivos temporales**: Todo se maneja en memoria (buffers pequeños)
- ✅ **Resiliente**: Si el pod muere, Temporal reintenta y el stream se reinicia desde S3
- ✅ **Eficiente**: No hay I/O doble (lectura + escritura a disco local)
- ✅ **Escalable**: Múltiples pods pueden hacer streaming simultáneo sin competir por disco

### Resiliencia ante Fallos

Si el pod muere durante una transferencia:
1. Temporal detecta la falta de heartbeat
2. Temporal reprograma la Activity en otro worker
3. El nuevo worker reinicia el stream desde S3
4. Si el canal soporta uploads resumibles, se retoma desde el offset guardado
5. Si no soporta, se reintenta desde cero (con idempotencia para evitar duplicados)

**Ver sección 9.6 del documento principal para detalles completos.**

---

## Resumen Ejecutivo

| Duda | Respuesta | Sección |
|------|-----------|----------|
| **Streaming vs Distribución** | Son procesos independientes. Streaming = servir a usuarios propios. Distribución = publicar a canales externos. | 9.2 |
| **Gestor de secretos** | HashiCorp Vault (self-hosted) con alternativa de Kubernetes Secrets + Sealed Secrets | 9.9.1 |
| **Archivos temporales** | Streaming directo desde S3 sin escribir archivos intermedios. Todo en memoria con buffers pequeños. | 9.6 |

---

## Referencias

- Documento principal: `09-distribucion-automatizada.md`
- Ejemplos de código: `09-distribucion-ejemplos-codigo.md`
- HashiCorp Vault: https://www.vaultproject.io/
- Temporal Heartbeats: https://docs.temporal.io/encyclopedia/detecting-activity-failures

