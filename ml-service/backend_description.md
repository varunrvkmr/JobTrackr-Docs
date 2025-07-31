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