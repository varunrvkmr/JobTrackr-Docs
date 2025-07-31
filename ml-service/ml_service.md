# ML-Service

### Inside **`ml-service/app/routes.py`** – what happens to the request after the backend proxy?

---

### 1. Startup & configuration

- A Flask **blueprint** `autofill_classify_bp` is registered for all ML endpoints, keeping them namespaced and easy to mount in `main.py` .
- Two runtime knobs come from environment variables:
    - `LEX_THRESHOLD` – the cut-off for the “exact word” fallback matcher .
    - `FEEDBACK_LOG` – where every classification / fill decision is appended as a CSV row .
    
    This CSV is the raw material the **training-service** later consumes when it retrains and publishes a new model version.
    

---

### 2. `POST /ml/classify` – main inference path

1. **Input check** – explosion-safe guard that the JSON body contains a non-empty `fields` list .
2. **Choose a text representation** for each field (label ≫ placeholder ≫ name ≫ id) to embed .
3. **Vectorise in one batch** with `embed_fn` (a Sentence-Transformers model), cast to `float32`, then **query FAISS** for the nearest canonical field → score & index .
4. For each field:
    - **Lexical short-circuit** – if the raw text matches a curated string table above `LEX_THRESHOLD`, accept immediately .
    - Otherwise, run **`calculate_dynamic_threshold`**, which adjusts the base semantic threshold using heuristics such as required flag, regex patterns, or generic wording; accept only if the semantic score beats that number .
5. Every decision (selector, match, score, threshold, confidence signals, accept/reject) is **logged to the CSV** for audit and future training.

The endpoint finally returns `{"matches":[…]}` with only the accepted mappings.

---

### 3. `GET /ml/health` – smoke test

Runs a tiny embed on the word “healthcheck”, verifies the tensor shape, that the FAISS index isn’t empty, and that canonical labels are loaded . If anything is off, it surfaces the exception in the JSON response so Kubernetes / Docker health-checks catch it.

---

### 4. `POST /ml/fill` – turn matches into real values

- Takes the caller-supplied `user_profile` dict plus the previously returned `matches`.
- For each match, look up `user_profile[canonical]`; if a value exists, emit `{selector, value}`; otherwise mark as empty .
- Each attempt is again **CSV-logged** – important because low fill-rates can indicate missing data or bad matches .

---

### 5. How this fits the end-to-end flow

```
Backend  (/api/autofill/…)          ml-service                 training-service
────────┬─────────────────► classify ──► embed_fn → FAISS
        │                            │         ↘ dynamic-threshold
        │                            │          ↘ lexical fallback
        │                            │           ↘ CSV log  (feedback)
        │                            │
        └─► fill ───────────────────► │ lookup values      ↘ CSV log
                                      │
                                      └─────────────▶  retrain on CSV, export
                                                        new /model_store/…
                                                        update "latest" symlink

```

- **`ml-service`** is *stateless* except for the mounted `model_store` volume and the rolling CSV.
- **`training-service`** watches (or is triggered to ingest) that CSV, rebuilds the Sentence-Transformer, writes a new versioned folder (`embeddings_vYYYYMMDD`), then flips `/model_store/latest`. Because `ml-service` reads the model at startup, a rolling restart brings the new model into production with zero changes to the backend.

---

### 6. Why this design works

- **Fast reads, cheap writes** – inference stays quick; feedback is an append-only log.
- **Safe experimentation** – thresholds & lexical tables can be tuned independently of retraining the heavy model.
- **Clear separation of duties** – the backend owns auth, data access, and routing; `ml-service` owns real-time ML; `training-service` owns offline learning and promotion.

With this, the data’s journey from the frontend click to a freshly retrained model is fully traceable, replayable, and decoupled.
