# Flujo

1. **Ingesta** → carga/ingreso + validación técnica (codec, duración, audio tracks)
2. **Enriquecimiento** → extracción metadata técnica + **IA** (labels, speech-to-text, OCR, escenas)
3. **Indexación** → motor de búsqueda con filtros/facetas
4. **Descubrimiento** → búsqueda, preview instantáneo, “similar assets”, etc.
5. **Entrega/Distribución** → empaquetado/transcoding + publicación a destinos (web/TV/redes)
6. **Archivo y preservación** → políticas de lifecycle, checksums/fixity, DR

## Definición simple (para examen)

Un **asset** (imagen, video, audio) tiene **un original (master)**

y  **múltiples renditions** , cada una con:

* distinto **formato**
* distinta **resolución / tamaño**
* distinta **calidad / bitrate**
* distinto **propósito**

El  **master nunca se toca** ; las renditions se generan y descartan cuando sea necesario.

## Enriquecimiento

### Busqueda / vectorizacion

 **Elastic** : BM25 + vectores + híbrido → estándar industrial

* **OpenSearch** : BM25 + vectores + HNSW → estándar open source
* **Weaviate / Pinecone** : alternativa SaaS (no típica en MAM “serios”)
* **FAISS** : pipelines offline, pero no como motor principal de búsqueda

En 2024–2025, la mayoría de vendors grandes (Dalet, Vidispine, AEM, Iconik, Cloudinary) ya usan **búsqueda vectorial** pero  *dentro de motores híbridos* , no FAISS standalone.

 **No necesitás combinar OpenSearch con FAISS.**

> **OpenSearch ya incluye vector search moderno y escalable, soportado oficialmente.**
>
> **La arquitectura recomendada** :
> → OpenSearch para full‑text + filtros + facetas
> → OpenSearch kNN para embeddings
> → Reranking opcional con IA
