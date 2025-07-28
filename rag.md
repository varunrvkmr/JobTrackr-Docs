## 1. What Is Retrieval-Augmented Generation (RAG)?

* **Core Idea**
  RAG combines a **retriever** (which looks up relevant documents or data) with a **generator** (an LLM that crafts fluent answers).
* **Key Benefits**

  * **Up-to-date & grounded**: pulls in fresh, factual context at query time
  * **Reduced hallucinations**: the model writes against real source text
  * **Domain specialization**: you can feed in any proprietary or user-specific data

---

## 2. Why Use RAG in Chatbots?

* **Dynamic Knowledge**: Your bot can answer from evolving sources (e.g. new job postings, updated resumes) without retraining.
* **Personalization**: By retrieving from a user’s own resume and the target job description, each response is tailored.
* **Efficiency**: Encourages concise prompting—only the top-k most relevant chunks are sent to the LLM, keeping costs and token-usage in check.

---

## 3. JobTrackr's Use Case

### A. Data Preparation & Chunking

1. **Resume & Job Text**

   * On each chat start, pull the user’s resume entries (work, skills, projects, education) and the job’s description from your PostgreSQL via SQLAlchemy.
   * For initial testing, you used a “simple” dumper that concatenates every record into one blob; later you can swap back in your optimized, token-budgeted builder.
2. **Chunk Splitting**

   * Split each text blob into manageable pieces (e.g. paragraphs) and drop any blank segments.
   * This ensures retrievals stay relevant and token-efficient.

### B. Vector Store & Embeddings

1. **Embeddings**

   * Use OpenAI’s embedding endpoint (`text-embedding-3-small`) via the `langchain-openai` client to turn each chunk into a vector.
2. **FAISS Index**

   * Build (or load) a FAISS index on disk under `indexes/{profile_id}_{job_id}`.
   * Pass `allow_dangerous_deserialization=True` when loading to accept trusted pickle files.

### C. RAG Pipeline

Encapsulated in `rag/pipeline.py` as a `generate_with_rag` function:

1. **Build or Load** the FAISS store of resume + job chunks.
2. **Retrieve** the top-k chunks most semantically similar to the user’s question.
3. **Compose** a concise prompt that injects just those retrieved snippets under a short system instruction.
4. **Call** the LLM (e.g. GPT-4) to produce a grounded, actionable answer.

### D. Integrating into Your Routes

1. **`/start`**

   * Map the JWT’s `user_auth_id` to your `UserProfileDB.id`.
   * Gather resume/job text, split into chunks, and call `generate_with_rag` to get the first response.
   * Store that response and the user question in an in-memory `ChatSession` object.
2. **`/<session_id>/message`**

   * On follow-ups, pull the same `optimized_resume` and `optimized_job_desc` from `session.initial_context`.
   * Re-retrieve top-k chunks against the new message and rerun `generate_with_rag`.
   * Append both user and assistant messages to session memory so context is always fresh.

### E. Session Persistence for Testing

* **JSON-backed sessions**: THe app writes `chat_sessions` into a local JSON file after each update.
* **Pros**: zero external dependencies, easy to inspect.
* **When to swap**: once you need concurrency safety, richer queries, or real-world scale, you can migrate to Redis or Postgres sessions.


---
<img src="../assets/rag-diagram.svg" alt="RAG Flow" width="2000">
---