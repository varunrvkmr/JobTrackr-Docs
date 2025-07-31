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