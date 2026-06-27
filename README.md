# Biomedical RAG — Alzheimer's Disease Therapeutic Targets

A retrieval-augmented generation (RAG) pipeline that answers questions about
**Alzheimer's disease drug targets**, grounded in real PubMed literature and
running fully **locally** (open embeddings + a local LLM via Ollama — no paid APIs).

<p>
  <img alt="Python" src="https://img.shields.io/badge/Python-3.11-3776AB?logo=python&logoColor=white">
  <img alt="RAG" src="https://img.shields.io/badge/RAG-hybrid%20(dense%20%2B%20BM25)-7952B3">
  <img alt="Vector DB" src="https://img.shields.io/badge/Vector%20DB-ChromaDB-FF6F61">
  <img alt="Embeddings" src="https://img.shields.io/badge/Embeddings-PubMedBERT-0A7E8C">
  <img alt="LLM" src="https://img.shields.io/badge/LLM-Ollama%20(llama3.2)-000000">
  <img alt="License" src="https://img.shields.io/badge/License-MIT-green">
</p>

---

## What it does

Ask a biomedical question — e.g. *"What are potential targets for Alzheimer's
disease treatment?"* — and the system retrieves the most relevant passages from a
corpus of PubMed abstracts, then has a local LLM compose an answer **with inline
citations to the source papers**. Every claimed molecular target is validated
against the retrieved context to suppress hallucinations.

## Architecture

```
                ┌─────────────────────────────────────────────────────────┐
                │  1. COLLECT      PubMed E-utilities (Biopython / Entrez)  │
                │                  13 query sets → 100 abstracts (2010–2026)│
                └──────────────────────────┬──────────────────────────────┘
                                           ▼
                ┌─────────────────────────────────────────────────────────┐
                │  2. PROCESS      clean (HTML strip, NFC, whitespace)      │
                │                  → 99 docs → 564 overlapping chunks       │
                └──────────────────────────┬──────────────────────────────┘
                                           ▼
                ┌─────────────────────────────────────────────────────────┐
                │  3. INDEX        PubMedBERT embeddings → ChromaDB (cosine)│
                │                  + BM25 lexical index (hybrid)           │
                └──────────────────────────┬──────────────────────────────┘
                                           ▼
                ┌─────────────────────────────────────────────────────────┐
                │  4. RETRIEVE     dense + BM25 fusion, query expansion     │
                └──────────────────────────┬──────────────────────────────┘
                                           ▼
                ┌─────────────────────────────────────────────────────────┐
                │  5. GENERATE     local LLM (Ollama) → cited answer        │
                │                  + post-validation & retry on bad output  │
                └──────────────────────────┬──────────────────────────────┘
                                           ▼
                ┌─────────────────────────────────────────────────────────┐
                │  6. EVALUATE     citation accuracy · target coverage ·    │
                │                  retrieval quality   + interactive UI     │
                └─────────────────────────────────────────────────────────┘
```

## Key features

- **Real biomedical corpus** — pulls live data from PubMed via the NCBI Entrez API
  (rate-limit aware, deduplicated, year-filtered).
- **Domain embeddings** — `pritamdeka/S-PubMedBert-MS-MARCO`, a PubMedBERT model
  fine-tuned for biomedical retrieval.
- **Hybrid retrieval** — dense semantic search (ChromaDB, cosine) fused with a
  BM25 lexical index, plus lightweight query expansion toward known AD targets.
- **Grounded generation** — local LLM via Ollama; answers must cite `[Source N]`.
- **Anti-hallucination** — every listed molecular target is checked for presence
  in the retrieved context (synonym-aware); malformed answers trigger a retry.
- **Evaluation harness** — citation accuracy, target coverage, retrieval quality.
- **Interactive UI** — `ipywidgets` panel to query the system inside the notebook.

## Results (100-paper corpus)

| Metric              | Score      | Meaning                                            |
| ------------------- | ---------- | -------------------------------------------------- |
| Citation Accuracy   | **0.80**   | answers that correctly cite their sources          |
| Target Coverage     | **4/5**    | known AD targets recovered (tau, Aβ, BACE1, APOE…) |
| Retrieval Quality   | **0.94**   | semantic relevance of top-k retrieved chunks       |

Corpus snapshot: **100 papers** (2010–2026) → **99 docs** → **564 chunks**
(mean 346 chars). Top MeSH terms: *Alzheimer Disease, Amyloid beta-Peptides,
Amyloid Precursor Protein Secretases, tau Proteins, Blood-Brain Barrier*.

## Quickstart

```bash
# 1. Install dependencies
pip install -r requirements.txt

# 2. Set your NCBI contact email (required by PubMed) — optionally an API key
export ENTREZ_EMAIL="you@example.com"
export NCBI_API_KEY="..."          # optional, raises rate limit to 10 req/s

# 3. Start a local LLM for the generation step
#    install Ollama from https://ollama.com, then:
ollama pull llama3.2:3b

# 4. Run the pipeline
jupyter notebook notebooks/01_pubmed_rag_pipeline.ipynb
```

Run the cells top to bottom: collection → processing → chunking → indexing →
retrieval → generation → evaluation → UI.

## Project structure

```
pubmed-rag-alzheimer/
├── notebooks/
│   └── 01_pubmed_rag_pipeline.ipynb   end-to-end pipeline
├── data/                              generated locally (git-ignored)
├── requirements.txt
├── LICENSE
└── README.md
```

> `data/` (collected abstracts, processed docs, chunks, ChromaDB) is generated by
> running the notebook and is intentionally git-ignored.

## Tech stack

`Biopython (Entrez)` · `sentence-transformers / PubMedBERT` · `ChromaDB` ·
`rank-bm25-style lexical scoring` · `Ollama` · `pandas` · `ipywidgets`

## Limitations & next steps

- **Single modality** — currently title + abstract only. Natural extensions:
  full-text (PMC), structured target/drug knowledge bases (UniProt, ChEMBL,
  Open Targets), and clinical-trial metadata.
- **Small corpus** — ~100 papers; the architecture scales to thousands of docs.
- **Extractive scope** — the system is strongest at listing/explaining targets
  *explicitly present* in the retrieved context, and deliberately conservative
  about anything beyond it.

## Disclaimer

Research/educational project. **Not** medical advice and not a substitute for
professional clinical or scientific judgement.

## License

[MIT](LICENSE)
