# How Does the ML Work?

# Description of Backend

---

### 1. `/api/autofill/classify` – thin proxy to **`ml-service`**

1. **Validate payload** (`fields` array must exist).
2. **Build ML endpoint URL** from `ML_SERVICE_URL` (default `http://ml-service:8000`).
3. **Forward the original JWT** so the downstream service can authorize the request.
4. **POST → `/ml/classify`** on `ml-service`; timeout after 5 s.
5. On success, **stream results back** to the frontend and **append a CSV row** for each match (selector, score, threshold, etc.).

Key excerpt:  (request forward) and CSV write loop .

> Why it matters: The backend now does zero ML work here; it simply enforces auth, handles network faults, and normalizes logging.
> 

---

### 2. `/api/autofill/fill` – proxy to **`ml-service`** for value generation

1. **Accept `matches`** from the client (output of the classify call).
2. **Flatten profile to a dict** of `canonical → value`.
3. **POST → `/ml/fill`** on `ml-service` with the profile and matches.
4. **Return filled values** and log the action.

Core flow: profile serialization and downstream call , plus logging loop .

---

### 3. Utility & feedback endpoints

- **`/debug/threshold`** – lets you inspect how the (now mostly dormant) dynamic-threshold logic would score a single field. Handy for ML tuning experiments.
- **`/feedback`** – records user feedback (`correct` / `incorrect`) on field matches for future offline training; logs straight to the same CSV.

---

### 4. Helper functions that remain in the file

Although classification now lives in `ml-service`, the backend still carries:

- **Similarity & threshold utilities** (`cosine_similarity`, `calculate_dynamic_threshold`, etc.)
- **Field-type handlers** for fill logic (textbox trim, semantic select, date formatter, etc.)—useful if you ever bring simple transformations back to the API layer.

They can be safely moved to `ml-service` (or trimmed) during cleanup, but keeping them here is benign.

---

### 5. Data flow recap (backend ↔ services)

```
Frontend
   │
   ├── POST /api/autofill/classify (fields)
   │       │
   │       └─► Backend proxy
   │                 └─► ml-service /ml/classify
   │                        └─► Model inference → matches
   │
   ├── POST /api/autofill/fill (matches)
   │       │
   │       └─► Backend serializes user profile
   │                 └─► ml-service /ml/fill
   │                        └─► Model logic → fills
   │
   └── (Optional) /training-service endpoints are called out-of-band

```

- **`ml-service`** hosts *inference* (embed model load, FAISS lookup, thresholding).
- **`training-service`** (not shown in this file) ingests feedback CSV rows, retrains the Sentence-Transformer, exports a new versioned model folder, and flips the `/model_store/latest` symlink.
- Both services mount the same **`model_store`** volume so model promotions propagate instantly.

---

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

# Training-Service

## Overall Summary

## **Purpose of the Training Service**

The training service builds and updates a **semantic search index** that maps canonical concepts (e.g., job titles, skills, etc.) to vector embeddings. It incorporates human feedback to improve accuracy, and manages versioned deployment of updated models and indexes.

---

## **Core Workflow Overview**

### 1. **Canonical Data Ingestion**

- **File**: `canonical_loader.py`
- Loads a JSON file of canonical entries like:
    
    ```json
    { "Software Engineer": ["SWE", "Software Eng", "Developer"] }
    
    ```
    
- Converts these into a flat format: `[{id, text}]`

---

### 2. **Feedback Integration (Optional)**

- **File**: `feedback_processor.py`
- Loads human feedback from a CSV.
- Extracts confirmed and rejected `(query_key, matched_key)` pairs.
- Used to improve training by giving more weight to known-correct associations.

---

### 3. **Model Setup**

- **File**: `model_manager.py`
- Downloads a pre-trained SentenceTransformer model (e.g., `all-MiniLM-L6-v2`) and saves it to disk.
- Also loads that model later for embedding text into vectors.

---

### 4. **Index Building**

- **File**: `index_builder.py`
- Converts the canonical (and optionally feedback-enriched) text into embeddings using the model.
- Builds a FAISS index for fast vector search.
- Saves both:
    - `index.faiss` → searchable embedding index
    - `meta.json` → maps FAISS IDs back to real-world keys

---

### 5. **Threshold Calibration (Full Train Only)**

- **File**: `threshold_calibrator.py` (referenced, not shown here)
- Computes optimal semantic/lexical thresholds and weights based on feedback.
- These guide downstream decision-making (e.g., match scoring).

---

### 6. **Version Management**

- **File**: `versioning.py` (referenced)
- Creates a new model version directory (e.g., `model_store/v20250730`)
- Copies model and index files into it
- Updates the `latest` symlink to point to the new version

---

## Run Modes (via `orchestrator.py`)

You launch training via CLI with different modes:

| Mode | What it does |
| --- | --- |
| `smoke-test` | Sanity check: can data load successfully? |
| `build-index` | Build a new FAISS index from canonical data only |
| `build-index-feedback` | Same as above, but incorporates confirmed matches |
| `full-train` | Runs the full pipeline: data load → feedback → calibration → embedding → indexing → versioning → metrics |

---

## Output Artifacts

Each run generates:

- `model/` — the embedding model
- `index.faiss` — the vector index for semantic search
- `meta.json` — lookup table from FAISS ID to canonical ID
- Optional: calibrated thresholds and performance metrics

## Pipeline

### orchestrator

### How data is processed inside **`training-service`** (`orchestrator.py`)

### canonical_loader

## `canonical_loader.py` — Load Canonical Data

This module defines a single function:

### `load_canonical()`

- Loads a canonical JSON file from the path specified in `TrainingConfig().canonical_file`.
- The JSON is expected to have:
    
    ```json
    {
      "id_1": ["synonym1", "synonym2", ...],
      "id_2": ["..."],
      ...
    }
    
    ```
    
- It flattens each list of synonyms into a single string:
    
    ```python
    {"id": "id_1", "text": "synonym1 synonym2"}
    
    ```
    
- Returns a list of these `{"id", "text"}` dictionaries.
- Logs how many entries were loaded and from where.

**Purpose**: This prepares the canonical dataset that acts as your reference vocabulary or gold standard for downstream indexing and matching.

---

### feedback_processor

## `feedback_processor.py` — Load and Filter Feedback

This module provides **three** functions:

### 1. `load_feedback()`

- Loads a CSV file from `TrainingConfig().feedback_file`.
- If the file doesn't exist, logs a warning and returns an empty list.
- Each row in the CSV is parsed into a `dict` and returned as a list of rows.
- Logs the number of rows loaded.

---

### 2. `get_confirmed_matches()`

- Uses `load_feedback()` and filters rows where `correct` is one of:
    - `"yes"`, `"true"`, or `"1"` (case insensitive).
- Returns a list of tuples: `(query_key, matched_key)`.
- Logs how many confirmed matches were found.

**Example use**: These pairs are considered **true matches** and are used in training or evaluation.

---

### 3. `get_rejected_matches()`

- Same as above, but filters where `correct` is `"no"`, `"false"`, or `"0"`.
- Returns a list of rejected matches.

**Example use**: Could be used to analyze model errors or train models to avoid bad matches.

---

### Summary:

| File | Purpose |
| --- | --- |
| `canonical_loader.py` | Loads your gold-standard dictionary of term IDs and their associated text blobs |
| `feedback_processor.py` | Loads human feedback and extracts confirmed or rejected matches for training/evaluation |

## Training

### orchestrator

## Overview

The `TrainingOrchestrator` serves as the **controller** for executing a full model training cycle. It pulls together canonical data and user feedback, builds a new model version, evaluates its accuracy, and promotes it if it meets quality standards. This script automates the retraining workflow and ensures only high-performing models are deployed to production.

---

## Step-by-Step Breakdown

### 1. Initialization

```python
configure_logging()
self.cfg = TrainingConfig()
self.vm  = VersionManager(self.cfg.model_store_base)
self.threshold = float(os.getenv("MIN_ACCURACY", "0.8"))

```

- Logging is configured at the start to track progress and errors.
- The config is loaded from environment and/or config files.
- A `VersionManager` is initialized to manage model version directories.
- A minimum acceptable accuracy threshold is set (default is 80%).

---

### 2. Load Input Data

```python
canon    = load_canonical()
feedback = get_confirmed_matches()

```

- **Canonical field definitions** are loaded from a trusted source.
- **Confirmed user feedback** is extracted from the feedback log (typically a CSV generated by `ml-service`). These are examples where the system made a prediction and the user marked it as correct.

---

### 3. Create a New Model Version Folder

```python
new_dir = self.vm.create_version()

```

- A timestamped folder is created inside the model store (e.g., `model_store/embeddings_v20250730`).
- This folder will store the model and associated FAISS index for the new version.

---

### 4. Populate Base Model

```python
base_model = os.getenv("BASE_MODEL")
ModelManager.copy_base_model(base_model, new_dir)

```

- The latest base embedding model (e.g., `all-MiniLM-L6-v2`) is copied into the new version directory.
- This provides the embedding engine used in downstream classification.

---

### 5. Build Index with Feedback

```python
IndexBuilder.build_with_feedback(canon, feedback, new_dir)

```

- A FAISS index is constructed using the canonical entries and the confirmed feedback.
- This enhances the search quality by incorporating real-world examples.

---

### 6. Evaluate Accuracy

```python
acc = top1_accuracy(new_dir, feedback)

```

- The Top-1 accuracy is computed using the new index and the feedback set.
- This metric assesses how often the correct canonical label was ranked as the closest match.

---

### 7. Promote if Good Enough

```python
if acc >= self.threshold:
    self.vm.update_latest(new_dir)
    self._notify_ml_service()

```

- If the accuracy meets or exceeds the configured threshold:
    - The `latest` symlink is updated to point to the new version folder.
    - A notification is sent to the `ml-service` to trigger a model reload.

If the model underperforms, it is **not promoted**, and the system continues serving the previous version.

---

### 8. Notify ML Service

```python
url = os.getenv("ML_SERVICE_RELOAD_URL", "http://ml-service:8000/reload")
requests.post(url)

```

- An HTTP POST is sent to the ML service's `/reload` endpoint to signal that a new model version is available.
- Any errors during this step are logged but do not halt the training process.

---

## Summary

The `TrainingOrchestrator` provides a clean, controlled flow for training and promoting new models in production. It ensures:

- Models are built from real feedback.
- Only high-performing models are served.
- Model updates are versioned and auditable.
- Production models can be upgraded or rolled back with no code changes—just by shifting a symlink.

### index_builder

## `IndexBuilder` class

This class provides two static methods for building FAISS indexes:

---

### 1. `build_index(canonical_data, out_dir)`

**Purpose**: Build a FAISS index using only the canonical dataset.

**Steps**:

1. Extracts text entries and original IDs from the canonical data.
2. Loads an embedding model via `ModelManager.load_model(out_dir)`.
3. Converts text into vector embeddings.
4. Builds a FAISS `IndexFlatL2` index and wraps it with `IndexIDMap` for ID tracking.
5. Saves:
    - The FAISS index to `index.faiss`
    - A `meta.json` file mapping `int_ids` to `orig_keys` (for reverse lookup)

**Output**:

- Efficient vector index (`index.faiss`)
- Metadata (`meta.json`) to connect index IDs with real-world keys

---

### 2. `build_with_feedback(canonical_data, feedback_pairs, out_dir, ...)`

**Purpose**: Incorporate confirmed feedback before building the index.

**What it does differently**:

- Constructs a map of `id → text` from canonical data.
- For each `(query_key, matched_key)` pair in the feedback:
    - Appends the matched text to the query's canonical entry.
        
        ```python
        canon_map[query_key] += " " + canon_map[matched_key]
        
        ```
        
    - This gives more semantic weight to known correct associations.
- Rebuilds the list into the same `{"id", "text"}` format.
- Then calls `build_index()` on this enriched set.

**Note**: The extra arguments (`lex_threshold`, `sem_threshold`, `weights`) are just passed for bookkeeping—**they are not used** in this function.

---

### Technologies Used

- **FAISS** for approximate nearest neighbor search
- **Sentence embedding model** (via `ModelManager`)
- **NumPy** for embedding conversion
- **JSON** for metadata tracking

---

### Summary

| Method | Use Case | Enrichment |
| --- | --- | --- |
| `build_index` | Index canonical dataset only | ❌ No feedback used |
| `build_with_feedback` | Incorporate confirmed matches | ✅ Merges matched text |

### model_manager

The `model_manager.py` file defines the `ModelManager` class, which handles saving and loading embedding models used during training and indexing. It abstracts away model handling so other modules (like `index_builder.py`) don’t need to know the internals of the SentenceTransformer workflow.

---

## `ModelManager` Class Overview

### 1. `copy_base_model(base_name: str, target_dir: Path)`

- **Downloads a SentenceTransformer model** by name (e.g., `"all-MiniLM-L6-v2"`).
- **Saves it** locally into a `model/` subdirectory inside the given `target_dir`.
- This is used to prepare a **versioned directory** during training or indexing.
- Example:
    
    ```python
    ModelManager.copy_base_model("all-MiniLM-L6-v2", Path("model_store/v20250730"))
    
    ```
    

---

### 2. `load_model(model_path: Path)`

- Loads a previously saved SentenceTransformer model from disk.
- Assumes the model lives inside a `model/` subfolder of `model_path`.
- Returns a ready-to-use `SentenceTransformer` instance set to `device="cpu"`.

---

### Key Tech Used

- [`SentenceTransformer`](https://www.sbert.net/) from `sentence-transformers` for semantic embeddings
- `Pathlib` for robust file path handling
- `shutil` is imported but not used in this snippet—you might be planning to use it for deeper file copies later

---

### Usage Summary

| Method | Purpose |
| --- | --- |
| `copy_base_model` | Pulls a pre-trained model from Hugging Face and saves it locally |
| `load_model` | Loads that saved model for encoding text during indexing |