# Sistema Automatizado de Clasificación de Soporte Técnico B2B con IA y RAG

Este proyecto implementa un flujo de trabajo (Workflow) automatizado en **n8n** diseñado para recibir, analizar y clasificar tickets de soporte técnico B2B utilizando Inteligencia Artificial (LLMs) y Generación Aumentada por Recuperación (RAG).

El sistema es capaz de leer un manual técnico vectorizado para dotar a la IA de contexto específico, analizar la solicitud del cliente, estructurar la salida (Prioridad, Componente, Impacto), registrar el incidente en Google Sheets y enviar una alerta crítica vía Telegram.

## 🚀 Arquitectura del Flujo (Workflow)

El flujo de n8n está dividido en dos ramas principales: la **Ingesta de Datos (RAG)** y la **Ejecución Principal del Bot**.

### 1. Ingesta de Conocimiento (RAG Pipeline)
Esta rama se ejecuta manualmente cuando se necesita actualizar la base de conocimientos de la IA.
* **Manual Trigger:** Disparador para iniciar la ingesta.
* **Google Drive (Download file):** Descarga el manual técnico `rag-uniconnect.pdf`.
* **PDF Hub (Convert PDF to plain text):** Extrae el texto del documento.
* **OpenAI Embeddings:** Convierte el texto plano en vectores de alta dimensionalidad.
* **Supabase Vector Store (Insert):** Almacena los vectores y la metadata en la tabla `documents` de PostgreSQL para futuras consultas.

### 2. Flujo Principal (Procesamiento de Tickets)
Esta rama se ejecuta automáticamente al recibir una petición del cliente.
* **Webhook (POST):** Recibe el payload JSON con el campo `mensaje_estudiante`.
* **Edit Fields (Set):** Aísla y formatea el mensaje de entrada.
* **AI Agent (LangChain):** El orquestador principal. Utiliza el modelo `llama-3.1-8b-instant` vía **Groq** bajo una regla estricta: debe consultar la base de datos antes de clasificar.
* **Herramientas del Agente (Tools & Memory):**
  * **MongoDB Chat Memory:** Mantiene el contexto de la conversación almacenando el historial en un clúster de MongoDB Atlas (colección `memory`).
  * **Supabase Vector Store (Retrieve-as-tool):** Permite al agente hacer búsquedas de similitud semántica en la base de datos de PostgreSQL para entender cómo clasificar el error.
* **Google Sheets (Append Row):** Guarda un registro estructurado con la fecha exacta (`$now`), el mensaje original y la extracción JSON generada por la IA.
* **Telegram (Send a text message):** Envía una alerta inmediata al equipo de soporte B2B con formato HTML si se procesa el ticket.

---

## 🛠️ Tecnologías y Servicios Utilizados

* **Orquestador:** [n8n](https://n8n.io/)
* **LLM Provider:** [Groq](https://groq.com/) (`llama-3.1-8b-instant`)
* **Vector Database:** [Supabase](https://supabase.com/) (PostgreSQL + pgvector)
* **Embeddings:** OpenAI
* **Database Memory:** MongoDB Atlas
* **Integraciones externas:** Telegram API, Google Drive API, Google Sheets API, PDF API Hub.

---

## ⚠️ Troubleshooting y Errores Conocidos

Durante el desarrollo y despliegue de esta arquitectura, se resolvieron incidentes críticos de conectividad y bases de datos. A continuación, el registro de soluciones:

### 1. Error de RPC en Supabase Vector Store
**Error:** `PGRST202 Could not find the function public.match_documents(filter, match_count, query_embedding) in the schema cache`

**Causa:** El nodo de n8n intentaba ejecutar una búsqueda de similitud (cosine similarity) utilizando la función `match_documents`, pero esta función y la extensión `pgvector` no estaban creadas en la base de datos de Supabase.

**Solución:** Se ejecutó el siguiente script SQL en el editor de Supabase para inicializar la infraestructura vectorial:

```sql
create extension if not exists vector;

create table if not exists documents (
  id bigserial primary key,
  content text, 
  metadata jsonb,
  embedding vector(1536) -- Dimensión de OpenAI Embeddings
);

create or replace function match_documents (
  query_embedding vector(1536),
  match_count int DEFAULT null,
  filter jsonb DEFAULT '{}'
) returns table (
  id bigint,
  content text,
  metadata jsonb,
  similarity float
)
language plpgsql
as $$
#variable_conflict use_column
begin
  return query
  select
    id,
    content,
    metadata,
    1 - (documents.embedding <=> query_embedding) as similarity
  from documents
  where metadata @> filter
  order by documents.embedding <=> query_embedding
  limit match_count;
end;
$$;
```

### 2. Error de Conexión TLS/SSL en MongoDB Atlas (Chat Memory)

**Error:** `D08D2481F27F0000:error:0A000438:SSL routines:ssl3_read_bytes:tlsv1 alert internal error... SSL alert number 80`

**Causa:** Node.js (el entorno de n8n) fallaba al completar el *handshake* TLS con el clúster `+srv` de MongoDB Atlas. Esto fue provocado por dos factores simultáneos: caracteres especiales no parseados en la contraseña y un bloqueo activo por el firewall de Atlas (Whitelist).

**Solución:** 1. **URL Encoding:** Se aplicó URL Encoding a la contraseña del usuario de BD (ej. el símbolo `$` pasó a `%24`, el `&` a `%26`) para evitar que el conector rompiera la cadena.
2. **Parámetros TLS:** Se modificó la configuración en n8n para usar "Connection String" inyectando explícitamente los parámetros `&tls=true`.

```text
mongodb+srv://<usuario>:<password_encodeado>@cluster0.823om8g.mongodb.net/memory?retryWrites=true&w=majority&tls=true

```

3. **Network Access (Whitelist):** Se verificó y ajustó la regla de IP en el panel de *Network Access* de MongoDB Atlas, habilitando el acceso temporal a `0.0.0.0/0` para permitir la entrada del tráfico de n8n.

---

## 👨‍💻 Autor

**Daner Alejandro Salazar Colorado** Ingenieria de Sistemas y Computación | Universidad de Caldas, Manizales.