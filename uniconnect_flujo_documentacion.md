# Documentación del Flujo n8n — Asistente Virtual Uniconnect

## Descripción general

Sistema de soporte automatizado para estudiantes de Uniconnect basado en IA. Recibe mensajes de estudiantes vía Webhook, consulta una base de conocimiento en PDF usando RAG (Retrieval-Augmented Generation), y responde por Telegram registrando todo en Google Sheets.

---

## Arquitectura final

El sistema está dividido en **dos flujos separados**:

### Flujo 1 — Carga del PDF (`flujo_1_carga_supabase.json`)
Se ejecuta **una sola vez** (o cuando se actualice el PDF).

```
Manual Trigger
    → Download file (Google Drive)
    → Extract from File (PDF → texto)
    → Supabase Vector Store Insert
        ├── Default Data Loader (sub-nodo ai_document)
        └── Embeddings HuggingFace Inference (sub-nodo ai_embedding)
```

### Flujo 2 — Atención de estudiantes (`flujo_2_atencion_supabase.json`)
Siempre activo, se ejecuta con cada mensaje entrante.

```
Webhook
    → Edit Fields
        ├── AI Agent (responde al estudiante)
        │     ├── Groq Chat Model — llama-3.3-70b-versatile (ai_languageModel)
        │     └── Answer questions with a vector store (ai_tool)
        │               ├── Groq Chat Model1 — llama-3.1-8b-instant (ai_languageModel)
        │               └── Supabase Vector Store Retrieve (ai_vectorStore)
        │                         └── Embeddings Retrieve — HuggingFace (ai_embedding)
        │     → Send a text message (Telegram)
        │     → Append row in sheet1 (Google Sheets — Hoja 2)
        │
        └── AI Agent1 (clasificador B2B)
              └── Groq Chat Model Agent1 — llama-3.1-8b-instant (ai_languageModel)
              → Append row in sheet (Google Sheets — Hoja 1)
```

---

## Servicios y credenciales utilizados

| Servicio | Uso | Credencial en n8n |
|---|---|---|
| Google Drive | Descargar el PDF RAG | Google Drive OAuth2 API |
| Google Sheets | Registrar tickets y respuestas | Google Sheets OAuth2 API |
| Telegram | Enviar respuestas al estudiante | Telegram API |
| Groq | Modelos de lenguaje (LLM) | Groq account |
| HuggingFace | Embeddings vectoriales | Hugging Face account |
| Supabase | Vector Store persistente (pgvector) | Supabase account |

---

## Base de datos Supabase

### Tabla `documents`
```sql
create table documents (
  id bigserial primary key,
  content text,
  metadata jsonb,
  embedding vector(768)
);
```

### Función `match_documents`
Necesaria para que n8n pueda hacer búsquedas vectoriales. Requiere el parámetro `filter` adicional que n8n envía automáticamente:

```sql
create or replace function match_documents (
  query_embedding vector(768),
  match_count int default 5,
  filter jsonb default '{}'
)
returns table (
  id bigint,
  content text,
  metadata jsonb,
  similarity float
)
language plpgsql
as $$
begin
  return query
  select
    documents.id,
    documents.content,
    documents.metadata,
    1 - (documents.embedding <=> query_embedding) as similarity
  from documents
  where documents.metadata @> filter
  order by documents.embedding <=> query_embedding
  limit match_count;
end;
$$;
```

> El tamaño del vector `768` corresponde al modelo de embeddings de HuggingFace por defecto.

---

## Problemas encontrados y soluciones

### 1. El agente no respondía desde el PDF
**Problema:** El flujo original descargaba el PDF pero nunca lo cargaba al Vector Store. Los nodos `Extract from File` y `Default Data Loader` estaban desconectados.

**Solución:** Conectar la cadena completa:
`Extract from File → Vector Store Insert` (main) con `Default Data Loader` y `Embeddings` como sub-nodos.

---

### 2. AI Agent1 no se ejecutaba
**Problema:** El `Groq Chat Model` estaba compartido entre los dos agentes. En n8n cada agente necesita su **propia instancia** del modelo conectada directamente.

**Solución:** Crear un nodo `Groq Chat Model Agent1` independiente y conectarlo exclusivamente al `AI Agent1`.

---

### 3. "A Vector Store sub-node must be connected and enabled"
**Problema (primera vez):** El `Default Data Loader` estaba conectado al `Vector Store` por `main` en lugar de como sub-nodo `ai_document`.

**Causa real:** En n8n el `Default Data Loader` es un sub-nodo (`ai_document`) del Vector Store. El dato principal (texto del PDF) llega al Vector Store por `main` desde `Extract from File`.

**Problema (segunda vez):** El Vector Store estaba en modo `retrieve-as-tool` en lugar de `retrieve`. El modo `retrieve-as-tool` requiere conectarse directo al agente, pero la arquitectura usa el nodo intermediario `Answer questions with a vector store`.

**Solución:** Usar modo `retrieve` y conectar como `ai_vectorStore` al nodo `Answer questions with a vector store`.

---

### 4. Rate limit de Groq
**Problema:** Con tres nodos usando `llama-3.3-70b-versatile` al mismo tiempo se alcanzaba el límite de tokens por minuto del plan gratuito (12,000 TPM).

**Solución:** Usar `llama-3.1-8b-instant` (modelo más ligero) para el clasificador (`AI Agent1`) y el tool del Vector Store (`Groq Chat Model1`). El modelo potente solo se usa para el agente principal que responde al estudiante.

| Nodo | Modelo |
|---|---|
| Groq Chat Model (AI Agent principal) | `llama-3.3-70b-versatile` |
| Groq Chat Model1 (tool Vector Store) | `llama-3.1-8b-instant` |
| Groq Chat Model Agent1 (clasificador) | `llama-3.1-8b-instant` |

---

### 5. El Vector Store en memoria no persistía entre flujos
**Problema:** Al separar el flujo de carga del flujo de atención, el Vector Store `InMemory` no compartía datos entre los dos workflows — cada workflow tiene su propio contexto de memoria.

**Solución:** Migrar a **Supabase** con `pgvector`. El PDF se carga una sola vez y los embeddings quedan guardados en la base de datos permanentemente.

---

### 6. "Could not find the function public.match_documents(filter, match_count, query_embedding)"
**Problema:** La función `match_documents` fue creada sin el parámetro `filter`, pero n8n siempre lo envía en sus consultas.

**Solución:** Recrear la función incluyendo `filter jsonb default '{}'` y recargar el schema cache de Supabase en Settings → API → Reload schema cache.

---

## Notas importantes

- El **Flujo 1** solo necesita ejecutarse de nuevo si se actualiza el PDF. Los datos persisten en Supabase indefinidamente.
- Si se borran los datos de Supabase, hay que volver a ejecutar el Flujo 1.
- El Vector Store usa embeddings de dimensión `768` — si se cambia el modelo de embeddings hay que recrear la tabla con el tamaño correcto y recargar el PDF.
- El clasificador B2B (`AI Agent1`) registra prioridad, componente afectado e impacto en **Hoja 1** del Google Sheets.
- Las respuestas al estudiante se registran en **Hoja 2** del mismo Google Sheets.
