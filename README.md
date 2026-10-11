# LiteRT — in-browser semantic search

Client-side **semantic search** demos running on WebGPU. There is no search backend: after the model is loaded, inference and ranking happen in the browser.

Two standalone pages:

| File | What it does |
|---|---|
| [`index.html`](index.html) | Semantic search over **articles** (text → text) with Embedding Gemma 2 via LiteRT |
| [`clip.html`](clip.html) | Multimodal **visual** search (text → image) with CLIP via Transformers.js |

Requirements: a **WebGPU** browser (recent Chrome). Serve the files over HTTP (do not open them as `file://`), for example:

```bash
npx serve .
```

Then open `http://localhost:3000` (`index.html`) and `http://localhost:3000/clip.html`.

The large Embedding Gemma weights are **not** in this repo. They are gitignored (`.litertlm`) and loaded from a public Cloudflare R2 URL on first visit.

---

## `index.html` — Dev.to Smart Search

Search by **meaning**, not keywords, across a fixed list of articles from [dev.to/kevin-uehara](https://dev.to/kevin-uehara).

Example: the query *"my web page is super slow and taking forever to render"* ranks the Chrome DevTools / Web Vitals article first, even without the words “devtools” or “vitals”.

### How it works

1. Loads `@litert-lm/core` from the CDN and `embeddinggemma-2-text-vision-440m.litertlm` from R2 on the **GPU** backend:
   `https://pub-6262d52f8235473686b7bd4ad060cfcd.r2.dev/embeddinggemma-2-text-vision-440m.litertlm`
2. For each article, vectorizes `title + description` with `EmbeddingEngine.computeEmbedding` (normalized).
3. On search, formats the query as `task: search query | text: …`, computes the embedding, and scores each article with **cosine similarity**.
4. Reorders the cards and highlights the top match. Inference latency is shown on the badge.

Search itself stays on-device. The network is only used to fetch the LiteRT engine and the model file.

### UI

- Status bar (model, WebGPU, indexing progress).
- Search field (enabled only after the GPU index is ready).
- Demo chips (“slow page performance”, “heavy background jobs”, etc.).
- Cards with tags, similarity score, and a link to the article.

### Dependencies

- `@litert-lm/core` via jsDelivr
- Embedding Gemma 2 (`.litertlm`, ~370 MB) hosted on Cloudflare R2

---

## `clip.html` — Visual Search (CLIP)

Search a photo gallery by **text description**. CLIP maps text and pixels into the same vector space, so “sleeping cat” finds the cat photo with no metadata.

### How it works

1. Loads `Xenova/clip-vit-base-patch32` with Transformers.js (`@huggingface/transformers@3.0.2`) on WebGPU (`fp32`): tokenizer + text model + processor + vision model.
2. On the first visit, downloads images from Unsplash and stores the blobs in **IndexedDB** (`CLIP_Demo_Store`). Later sessions read from the local cache.
3. Each image becomes an L2-normalized embedding (`CLIPVisionModelWithProjection`).
4. The query is prefixed with `a photo of a …`, run through the text model, and ranked by **dot product** (equivalent to cosine after L2).
5. The displayed score is min–max normalized against the current result set.

### UI

- Same status / WebGPU / latency pattern.
- Card grid with photo, name, and score.
- Visual highlight on the top match.

### Dependencies and network

- First load: Hugging Face models + 12 Unsplash images.
- After that: images from IndexedDB; models from the browser cache (`env.useBrowserCache = true`).

---

## Shared pipeline

Both pages follow the same flow:

```
load model (WebGPU)
        ↓
index the corpus (articles or images) → embeddings
        ↓
query embedding
        ↓
similarity → reorder the list
```

Search runs on the `input` event with debounce (~180–200 ms). Nothing is sent to a search server.

---

## Repo notes

- A local copy of the model is optional; `index.html` always points at the R2 URL.

---

## Limitations

- WebGPU is required; without a GPU, `init` fails and the status bar shows an error.
- `index.html` needs the network on the **first** visit to download the Gemma weights from R2.
- `clip.html` needs the network on the **first** visit (CLIP model + images).
- The corpora are static (8 articles / 12 photos) — this is a demo, not a live index.
