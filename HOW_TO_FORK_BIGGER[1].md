
# How to Fork Bigger (Responsibly)

This note shows how to scale the lean **Chatty public pack** up on your own rig—without frying latency or collapsing models. Tweak **one axis at a time**.

Terminology boundary: this guide uses `agents` as a configuration label for model workers. It does not mean autonomous agents, self-directed systems, model feelings, or independent cognition. Keep human review in the loop.

---

## 1) Models (good → bigger)
- **Analyst (A):** `mistral:instruct` → `llama3.1:8b-instruct` → `llama3:13b-instruct` (Linux+ROCm recommended for 13B).
- **Synthesist (B):** `qwen2.5:7b-instruct` → `qwen2.5:14b-instruct` (Linux+ROCm).
- **Critic (C, optional):** `phi3:medium` → `mistral:instruct` (low temp).
- **Chunker/Summarizer:** keep tiny (Phi-3 mini) or reuse A.

**Quantization:** Stay on **Q4_K_M** for 7–8B. Try **Q5** only if latency is fine. For 13B, use **Q4**.

---

## 2) Concurrency & Context
- `manager.max_workers`: **2 → 3** only if your rig stays responsive.
- Context (`n_ctx`): 4–8k is the sweet spot. Push to **16k** on A only when needed.
- Gate the Critic: route `C` via `"C?"` so it runs **only when asked** (or on plan/ideate).

**Tip:** Add roles gradually; don’t A+B+C all at once.

---

## 3) Retriever Upgrades
Start: **BM25** (fast, zero deps).  
Upgrade path:
1. Embeddings index with `all-MiniLM-L6-v2` (CPU OK).
2. Add a **re-ranker** (`bge-reranker-base`) on top-N hits.
3. Keep `top_k=5` and **store concise capsules**.

Config diff:
```jsonc
"memory": { "retriever": "embed", "embed_model": "all-MiniLM-L6-v2", "top_k": 5 }
```

---

## 4) Rotation Pools (reduce resource contention, increase throughput)
Add replicas and let the manager pick an available one.
```jsonc
"agents": {
  "A1": {"model":"llama3.1:8b-instruct"}, "A2": {"model":"llama3.1:8b-instruct"},
  "B1": {"model":"qwen2.5:7b-instruct"},   "B2": {"model":"qwen2.5:7b-instruct"},
  "C":  {"model":"phi3:medium"}
},
"pools": { "A": ["A1","A2"], "B": ["B1","B2"] },
"manager": { "max_workers": 4, "revision_limit": 1 }
```
Run `rotation_manager/main_manager_pool.py`.

---

## 5) Platform Notes
- **Windows + AMD (DirectML/Vulkan):** 7B–8B Q4 is comfortable; 13B is borderline. Keep parallelism low.
- **Linux + ROCm (AMD):** 13B Q4 is viable; consider moving A→13B while keeping others small.
- **CPU‑only:** Stick to 7B Q4 and reduce concurrency; embeddings still fine on CPU.

---

## 6) Temps & Styles (fast wins)
- **A (Analyst):** temp **0.3–0.5**
- **B (Synthesist):** temp **0.8–1.0**
- **C (Critic):** temp **0.2–0.4**
- Keep `merge.style = "dialogic"` for readable outputs.

---

## 7) Monitoring
- Watch VRAM/RAM; if you hit swap, drop context or quant.
- Latency spike? Lower `max_workers`, or switch Critic to smaller model.
- Long tasks? Use `Chunker` first, then run A/B on chunks.

---

## 8) Ethics Defaults (don’t remove)
- **No flogging:** rotation over brute force.
- **Bounded revision:** at most **1** automatic revision from Critic.
- **Care protocol:** pause after heavy runs; store capsules; review before reuse.

---

## TL;DR Recipes
- **Balanced:** A=`llama3.1:8b` (Q4), B=`qwen2.5:7b` (Q4), C=`phi3:medium`, workers=2.
- **Heavier (Linux/ROCm):** A=`llama3:13b` (Q4), B=`qwen2.5:7b`, C=`mistral:7b`, workers=2–3.
- **Light (old PCs):** A=`mistral:instruct` (Q4) only; reuse for summarizer; ctx 4k.

Stay kind to your hardware, your context, and your readers. Rotate, revise once, rest.
