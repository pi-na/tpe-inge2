# 5. Supuestos

Los siguientes supuestos fueron considerados durante el diseño de la arquitectura. Cualquier desviación de estos supuestos podría requerir ajustes en la solución propuesta.

## 5.1 Supuestos de Negocio

| ID | Supuesto | Impacto si no se cumple |
|----|----------|------------------------|
| SN-01 | El grupo de medios cuenta con presupuesto para infraestructura on-premise o cloud IaaS significativa | Reducir escala inicial, priorizar componentes críticos |
| SN-02 | Existe un equipo técnico interno o contratado capaz de operar la infraestructura propuesta | Simplificar arquitectura o considerar servicios managed |
| SN-03 | El volumen inicial es de aproximadamente 5 millones de assets, con crecimiento de 500K assets/año | Ajustar sizing de infraestructura |
| SN-04 | Los usuarios concurrentes típicos no superan los 500 editores simultáneos | Redimensionar capacidad de servicios y base de datos |
| SN-05 | El horario de mayor uso es de 8:00 a 20:00, permitiendo ventanas de mantenimiento nocturnas | Modificar estrategia de actualizaciones y mantenimiento |

## 5.2 Supuestos Técnicos

| ID | Supuesto | Impacto si no se cumple |
|----|----------|------------------------|
| ST-01 | Los archivos master no requieren edición in-place; siempre se trabaja sobre renditions o se re-ingesta | Agregar capacidad de edición de masters |
| ST-02 | La latencia de red entre centros de datos es menor a 50ms | Revisar estrategia de replicación y consistencia |
| ST-03 | Los canales de distribución externos proveen APIs estables y documentadas | Desarrollo adicional de conectores, mayor manejo de errores |
| ST-04 | El contenido multimedia sigue formatos estándar (H.264/H.265, AAC, JPEG, PNG) | Extender workers de transcodificación para formatos legacy |
| ST-05 | Se dispone de hardware con GPU para procesamiento de IA en volumen | Procesar con CPU (mayor latencia) o reducir enriquecimiento automático |

## 5.3 Supuestos de Integración

| ID | Supuesto | Impacto si no se cumple |
|----|----------|------------------------|
| SI-01 | El directorio corporativo (LDAP/AD) está disponible para federación de identidades | Implementar gestión de usuarios local en Keycloak |
| SI-02 | Los sistemas de playout de TV soportan ingesta vía MOS Protocol o FTP estándar | Desarrollar conectores específicos para cada sistema |
| SI-03 | Las redes sociales target (YouTube, Facebook, Twitter) mantienen sus APIs públicas | Actualizar conectores ante cambios de API |

---

# 6. Riesgos

Los riesgos identificados representan eventos o condiciones que, de materializarse, podrían afectar negativamente el éxito del proyecto o la operación del sistema.

## 6.1 Riesgos Técnicos

### R-01: Complejidad Operacional de Ceph

**Descripción**: Ceph es una tecnología poderosa pero compleja. La falta de experiencia del equipo puede resultar en configuraciones subóptimas, degradación de performance o pérdida de datos.

**Probabilidad**: Media | **Impacto**: Alto

**Indicadores tempranos**:
- Tiempos de recuperación de OSDs superiores a lo esperado.
- Alertas frecuentes de cluster health.
- Performance de escritura inferior a benchmarks.

**Estrategia de mitigación**:
1. Capacitar al equipo en administración de Ceph antes del go-live.
2. Contratar soporte comercial (Red Hat Ceph) para el primer año.
3. Implementar monitoreo exhaustivo con alertas proactivas.
4. Documentar runbooks para operaciones comunes y troubleshooting.

**Plan de contingencia**: Si Ceph resulta inmanejable, migrar a MinIO que es más simple de operar, aceptando algunas limitaciones en escala y features.

---

### R-02: Degradación de Performance del Índice de Búsqueda

**Descripción**: A medida que el volumen de assets crece, el índice de OpenSearch podría degradar su performance de búsqueda, superando los SLAs de latencia requeridos.

**Probabilidad**: Media | **Impacto**: Alto

**Indicadores tempranos**:
- Latencia P95 de búsqueda superior a 500ms.
- Uso de heap de OpenSearch superior al 70%.
- Tiempos de indexación crecientes.

**Estrategia de mitigación**:
1. Diseñar índices con sharding apropiado desde el inicio (1 shard por 30GB).
2. Implementar índices time-based con rollover automático.
3. Monitorear métricas de performance continuamente.
4. Planificar capacidad con 6 meses de anticipación.

**Plan de contingencia**: Escalar horizontalmente agregando nodos de datos. En casos extremos, particionar índices por proyecto o tipo de asset.

---

### R-03: Fallas en Integraciones con Canales Externos

**Descripción**: Las APIs de redes sociales y sistemas externos pueden cambiar sin previo aviso, sufrir outages, o implementar rate limits más restrictivos, afectando la distribución automatizada.

**Probabilidad**: Alta | **Impacto**: Medio

**Indicadores tempranos**:
- Incremento en tasas de error de conectores.
- Notificaciones de deprecación de APIs.
- Nuevos rate limits anunciados por plataformas.

**Estrategia de mitigación**:
1. Implementar circuit breakers por canal para aislar fallas.
2. Mantener colas de retry independientes por destino.
3. Suscribirse a changelogs y blogs de desarrolladores de cada plataforma.
4. Diseñar conectores como plugins para facilitar actualización.

**Plan de contingencia**: En caso de falla persistente en un canal, notificar a editores y permitir distribución manual mientras se resuelve.

---

### R-04: Bottleneck en Procesamiento de IA

**Descripción**: El enriquecimiento con IA (especialmente ASR para video largo y análisis de video frame-by-frame) puede generar cuellos de botella que atrasen significativamente el tiempo de procesamiento de assets.

**Probabilidad**: Media | **Impacto**: Medio

**Indicadores tempranos**:
- Cola de jobs de IA creciendo sostenidamente.
- Tiempos de procesamiento de video superiores a 3x la duración del video.
- Utilización de GPU cercana al 100% constantemente.

**Estrategia de mitigación**:
1. Priorizar procesamiento de IA por tipo de asset y urgencia editorial.
2. Implementar procesamiento "progressive" (thumbnails y preview primero, IA después).
3. Permitir a editores marcar assets como "urgentes" para priorización.
4. Escalar workers de IA horizontalmente según demanda.

**Plan de contingencia**: Permitir que assets pasen a estado "Ready" sin enriquecimiento completo de IA, completándolo posteriormente en background.

---

### R-05: Corrupción Silenciosa de Datos (Bit Rot)

**Descripción**: Los archivos almacenados a largo plazo pueden sufrir corrupción de bits no detectada, comprometiendo la integridad del archivo original.

**Probabilidad**: Baja | **Impacto**: Muy Alto

**Indicadores tempranos**:
- Discrepancias en verificaciones periódicas de checksums.
- Errores de lectura reportados por el storage.

**Estrategia de mitigación**:
1. Almacenar checksums SHA-256 de cada asset en la base de datos.
2. Implementar verificación periódica de integridad (scrubbing) en Ceph.
3. Utilizar erasure coding que permite reconstrucción de datos corruptos.
4. Mantener al menos una réplica en medio de almacenamiento diferente.

**Plan de contingencia**: Si se detecta corrupción, restaurar desde réplicas o backups. Notificar a responsables del asset para evaluar re-ingesta si no hay copia válida.

---

## 6.2 Riesgos de Proyecto

### R-06: Subestimación del Esfuerzo de Migración

**Descripción**: Si existe un sistema DAM legacy, la migración de datos podría ser significativamente más compleja y costosa de lo estimado.

**Probabilidad**: Media | **Impacto**: Alto

**Estrategia de mitigación**:
1. Realizar análisis detallado del sistema legacy antes de planificar migración.
2. Priorizar migración por proyectos/colecciones, no big-bang.
3. Mantener sistema legacy en paralelo durante período de transición.

---

### R-07: Curva de Aprendizaje del Equipo

**Descripción**: La arquitectura propuesta utiliza tecnologías (Temporal, Ceph, Kafka) que pueden no ser familiares para el equipo, generando retrasos y errores.

**Probabilidad**: Media | **Impacto**: Medio

**Estrategia de mitigación**:
1. Invertir en capacitación antes de iniciar desarrollo.
2. Comenzar con PoC en ambiente de desarrollo.
3. Considerar consultoría especializada para setup inicial.
4. Documentar decisiones y configuraciones exhaustivamente.

---

## 6.3 Matriz de Riesgos

```
                    IMPACTO
              Bajo    Medio    Alto    Muy Alto
         ┌────────┬────────┬────────┬────────┐
   Alta  │        │  R-03  │        │        │
         ├────────┼────────┼────────┼────────┤
P  Media │        │  R-04  │ R-01   │        │
R        │        │  R-07  │ R-02   │        │
O        │        │        │ R-06   │        │
B  ├─────┼────────┼────────┼────────┼────────┤
.  Baja  │        │        │        │  R-05  │
         └────────┴────────┴────────┴────────┘
```

---

# 7. No Riesgos

Los siguientes aspectos fueron evaluados y se consideran controlados o de bajo riesgo dado el diseño propuesto.

## NR-01: Disponibilidad del Sistema Core

**Justificación**: La arquitectura con componentes replicados (PostgreSQL con Patroni, Kafka con replicación, servicios stateless en Kubernetes con múltiples réplicas) garantiza alta disponibilidad. La falla de instancias individuales no afecta el servicio.

**Evidencia de control**:
- PostgreSQL: Failover automático en <30 segundos con Patroni.
- Kafka: Mensajes replicados en 3 brokers, tolerancia a falla de 1 broker.
- Servicios: Health checks y restart automático en Kubernetes.

---

## NR-02: Pérdida de Datos de Assets

**Justificación**: Ceph con erasure coding 8+3 provee durabilidad de 11 nueves (99.999999999%). Los checksums almacenados permiten detectar cualquier corrupción. La probabilidad de pérdida irrecuperable es extremadamente baja.

**Evidencia de control**:
- Erasure coding permite reconstruir datos con hasta 3 OSDs fallidos.
- Scrubbing periódico detecta y repara inconsistencias.
- Checksums SHA-256 verifican integridad end-to-end.

---

## NR-03: Escalabilidad de Almacenamiento

**Justificación**: Ceph escala horizontalmente agregando OSDs sin interrupción de servicio. La arquitectura de object storage no tiene límites prácticos de capacidad.

**Evidencia de control**:
- Ceph utilizado en producción en deployments de exabytes.
- Agregar capacidad es operación online sin downtime.
- Tiering automático optimiza costos hot/cold.

---

## NR-04: Vendor Lock-in en Storage

**Justificación**: El uso de API S3-compatible (Ceph RGW o MinIO) permite cambiar de implementación de storage sin modificar código de aplicación. No hay dependencia de servicios propietarios de cloud.

**Evidencia de control**:
- Servicios acceden a storage exclusivamente vía API S3 estándar.
- Migración entre implementaciones S3-compatible es transparente.
- No se utilizan features propietarios de ningún vendor.

---

## NR-05: Seguridad de Acceso a Assets

**Justificación**: La combinación de Keycloak (autenticación) + OPA (autorización granular) + URLs prefirmadas con expiración + TLS en tránsito + cifrado en reposo provee un modelo de seguridad robusto.

**Evidencia de control**:
- Ningún acceso a assets sin autenticación válida.
- Políticas ABAC permiten control fino por asset/proyecto/rol.
- URLs prefirmadas expiran, previniendo acceso persistente no autorizado.
- Auditoría de todos los accesos.

---

## NR-06: Performance de Búsqueda Básica

**Justificación**: OpenSearch con índices optimizados y caching en Redis garantiza tiempos de respuesta sub-segundo para búsquedas típicas.

**Evidencia de control**:
- Benchmarks de OpenSearch muestran <100ms para queries simples en índices de millones de documentos.
- Cache de queries frecuentes reduce latencia a <10ms.
- Búsqueda híbrida mantiene latencia aceptable incluso con componente vectorial.

---

# 8. Trade-offs

Las siguientes decisiones arquitectónicas implican compromisos conscientes entre atributos de calidad o características del sistema.

## T-01: Consistencia Eventual en Búsqueda vs. Simplicidad

**Decisión**: El índice de búsqueda (OpenSearch) se actualiza de forma asíncrona mediante eventos, lo que implica que un asset recién creado puede no aparecer inmediatamente en resultados de búsqueda.

**Trade-off**:
- **Se sacrifica**: Consistencia inmediata entre base de datos e índice de búsqueda.
- **Se gana**: Desacoplamiento entre operaciones de escritura y búsqueda, mejor performance de ingesta, simplicidad del modelo.

**Justificación de negocio**: El flujo editorial típico no requiere que un asset sea buscable inmediatamente después de subirse. El procesamiento de transcodificación y enriquecimiento toma varios minutos, durante los cuales el asset se indexa. La ventana de inconsistencia (segundos) es imperceptible para el usuario.

**Impacto**: Latencia de <30 segundos entre commit en DB y aparición en búsqueda. Aceptable dado el flujo de trabajo editorial.

---

## T-02: Complejidad Operacional vs. Funcionalidad y Escala

**Decisión**: Adoptar tecnologías como Ceph, Kafka, Temporal y OpenSearch que son potentes pero requieren expertise para operar.

**Trade-off**:
- **Se sacrifica**: Simplicidad operacional; se requiere equipo más especializado.
- **Se gana**: Capacidades enterprise de escalabilidad, durabilidad, confiabilidad y funcionalidad avanzada.

**Justificación de negocio**: Un grupo de medios que maneja millones de assets justifica la inversión en infraestructura robusta. Los costos de downtime o pérdida de datos superan ampliamente el costo de operar sistemas más complejos. Alternativas más simples no cumplirían los requerimientos de escala a mediano plazo.

**Alternativas descartadas**:
- Soluciones SaaS: Vendor lock-in prohibido por la consigna.
- Stack más simple (filesystem + SQLite + cron jobs): No escala, no garantiza durabilidad.

---

## T-03: Latencia de Upload vs. Integridad de Datos

**Decisión**: Implementar verificación de checksum end-to-end para cada upload, lo que agrega overhead al proceso de ingesta.

**Trade-off**:
- **Se sacrifica**: Velocidad máxima de ingesta (overhead de cálculo de checksum ~5%).
- **Se gana**: Garantía de integridad; certeza de que el archivo almacenado es idéntico al subido.

**Justificación de negocio**: En un sistema DAM para medios, los archivos originales son patrimonio digital de la organización. Un archivo corrupto silenciosamente puede descubrirse años después cuando se necesita, generando pérdida irrecuperable. El pequeño overhead de verificación es insignificante comparado con el valor de la garantía de integridad.

---

## T-04: Costo de Almacenamiento vs. Velocidad de Acceso

**Decisión**: Implementar tiering de storage con hot tier (SSD) y cold tier (HDD), moviendo assets según frecuencia de acceso.

**Trade-off**:
- **Se sacrifica**: Velocidad uniforme de acceso a todos los assets.
- **Se gana**: Reducción significativa de costos de storage (HDD ~5x más barato que SSD por TB).

**Justificación de negocio**: El patrón de acceso en DAM sigue una distribución de cola larga: ~20% de assets se acceden frecuentemente, ~80% raramente. Mantener todo en SSD sería costoso e innecesario. El tiempo de "restore" de cold a hot (minutos) es aceptable para assets archivados que no se acceden hace meses.

**Configuración elegida**:
- Hot tier: Assets accedidos en últimos 90 días.
- Cold tier: Assets sin acceso en 90+ días.
- Restore automático a hot ante acceso (latencia adicional primera vez).

---

## T-05: Procesamiento "Eager" vs. "Lazy" de IA

**Decisión**: Procesar todos los assets con IA de forma eager (inmediatamente después del upload) en lugar de lazy (bajo demanda).

**Trade-off**:
- **Se sacrifica**: Recursos de cómputo significativos para procesar assets que quizás nunca se busquen por contenido semántico.
- **Se gana**: Todos los assets son igualmente buscables desde el momento en que completan procesamiento; mejor UX de búsqueda.

**Justificación de negocio**: El valor del DAM está en hacer descubrible el contenido. Si el enriquecimiento de IA fuera lazy, los editores no podrían encontrar assets por contenido semántico hasta que alguien los procesara explícitamente, perdiendo el principal beneficio de la indexación inteligente.

**Mitigación del costo**: 
- Priorización de cola por tipo de asset (video prioritario sobre imagen).
- Escalamiento elástico de workers según demanda.
- Procesamiento durante horarios de menor carga cuando es posible.

---

## T-06: Autonomía de Servicios vs. Consistencia Transaccional

**Decisión**: Cada servicio (Asset, Metadata, Distribution) tiene su propia responsabilidad y no participa en transacciones distribuidas. La consistencia se logra mediante eventos y sagas.

**Trade-off**:
- **Se sacrifica**: Consistencia transaccional estricta (ACID) entre servicios.
- **Se gana**: Servicios desacoplados que pueden escalar, fallar y evolucionar independientemente.

**Justificación de negocio**: Las operaciones del DAM son inherentemente de larga duración (minutos a horas). Mantener transacciones abiertas sería impráctico. El modelo eventual (asset creado → procesamiento → indexación → distribución) refleja mejor la naturaleza del dominio que un modelo transaccional.

**Garantías mantenidas**:
- Consistencia dentro de cada servicio (PostgreSQL ACID).
- Eventual consistency entre servicios con garantía de entrega (Kafka).
- Idempotencia de operaciones para manejar reintentos seguros.

---

## T-07: Flexibilidad de Esquema vs. Validación Estricta

**Decisión**: Metadatos almacenados como JSONB en PostgreSQL con esquemas JSON Schema validados pero extensibles, en lugar de tablas rígidamente tipadas.

**Trade-off**:
- **Se sacrifica**: Validación en tiempo de compilación; esquema puede divergir entre assets.
- **Se gana**: Flexibilidad para diferentes tipos de assets y evolución del esquema sin migraciones costosas.

**Justificación de negocio**: Diferentes tipos de contenido (video de noticias, foto de archivo, audio de podcast) tienen metadatos muy diferentes. Un esquema rígido forzaría o un diseño muy genérico o migraciones frecuentes. JSON Schema permite definir y validar esquemas por tipo de asset manteniendo flexibilidad.

---

## T-08: URLs Prefirmadas vs. Proxy de Descarga

**Decisión**: Las descargas de assets grandes usan URLs prefirmadas directamente al Object Storage en lugar de pasar por un servicio proxy.

**Trade-off**:
- **Se sacrifica**: Control granular durante la descarga (no se puede revocar un download en progreso).
- **Se gana**: Eliminar el servicio de aplicación como cuello de botella en transferencias de datos; escala ilimitada de descargas concurrentes.

**Justificación de negocio**: Los archivos de video pueden ser de varios GB. Si el tráfico pasara por servicios de aplicación, se necesitaría infraestructura costosa y se introduciría latencia. Las URLs prefirmadas delegan el trabajo pesado al Object Storage (diseñado para esto) manteniendo control de autorización (el servicio valida permisos antes de generar la URL).

**Controles de seguridad mantenidos**:
- URLs expiran en 15 minutos.
- Se registra auditoría de quién solicitó la URL.
- Permisos validados en tiempo de generación de URL.

---

## Resumen de Trade-offs

| ID | Atributo Sacrificado | Atributo Ganado | Justificación Principal |
|----|---------------------|-----------------|------------------------|
| T-01 | Consistencia inmediata | Performance, simplicidad | Flujo editorial tolera segundos de latencia |
| T-02 | Simplicidad operacional | Escala, funcionalidad | Requerimientos de escala lo justifican |
| T-03 | Velocidad máxima ingesta | Integridad garantizada | Assets son patrimonio de alto valor |
| T-04 | Velocidad uniforme | Costo de storage | Patrón de acceso cola larga |
| T-05 | Recursos de cómputo | Descubribilidad total | Valor del DAM es hacer contenido buscable |
| T-06 | Consistencia ACID distribuida | Autonomía de servicios | Operaciones son inherentemente de larga duración |
| T-07 | Validación estricta | Flexibilidad de esquema | Tipos de contenido muy diversos |
| T-08 | Control de descarga activa | Escala de transferencias | Volumen de datos lo requiere |

