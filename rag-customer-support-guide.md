# RAG Customer Support Assistant — Complete Build Guide

A step-by-step blueprint for building a production-grade, portfolio-ready RAG (Retrieval-Augmented Generation) chatbot that answers a company's customer questions from its own documents — with source citations, Arabic + English support, and zero hallucination tolerance.

**Target outcome:** a live demo + polished GitHub repo you can showcase on Upwork, Mostaql, and Khamsat, and a reusable codebase you can re-deploy for each new client in days, not weeks.

---

## 1. Project Definition

**Name:** AI Customer Support Assistant (RAG)

**What it does:** A chatbot trained on any company's knowledge base (PDFs, Word docs, website pages, FAQ files) that:

- Answers customer questions accurately, **citing the exact source file and page** for every claim.
- Says "I don't have enough information" instead of guessing when the answer isn't in the documents.
- Escalates to a human agent (email/webhook handoff) when needed.
- Works in **both Arabic and English**, replying in the customer's language.
- Embeds into any website with a single `<script>` tag.

**Why demand is high:** every SMB with documentation wants to cut support costs. Search volume on freelance platforms for "RAG chatbot", "AI chatbot trained on my data", and "custom knowledge base GPT" is consistently strong, and Arabic/RTL support is a genuine gap in the Gulf and Egyptian markets.

### Differentiating features (what puts you above 90% of competitors)

| Feature | Why it sells |
|---|---|
| Hybrid search (semantic + BM25) | Catches product names, SKUs, and model numbers that pure vector search misses — critical for Arabic |
| Source citations | Every answer shows file name + page number; builds client trust instantly |
| "I don't know" guardrail | Prevents hallucination — the #1 client fear about AI bots |
| Embeddable widget with RTL | One-line install, matches client branding, rare in the market |
| Analytics dashboard | Shows unanswered questions — a goldmine for the client's content team |
| Documented evaluation (RAGAS) | Real metrics in your README; almost no freelancer offers this |
| Human handoff | Sends full conversation transcript to a human via email/webhook |

---

## 2. Tech Stack (with rationale)

| Layer | Choice | Why |
|---|---|---|
| Language / API | Python 3.11 + FastAPI | Async, easy SSE streaming, industry standard |
| Orchestration | Hand-rolled core (optionally LlamaIndex) | Full control and understanding; you can defend every decision to clients |
| Embeddings | OpenAI `text-embedding-3-small` **or** `BAAI/bge-m3` (open source) | 3-small is cheap and multilingual; bge-m3 is excellent for Arabic and free to self-host |
| Vector DB | Qdrant | Runs locally via Docker, has a free cloud tier, native hybrid-search support |
| LLM | `gpt-4o-mini` or Claude Haiku | Cheap enough for production; wrap in an abstraction layer so models are swappable |
| Reranker | Cohere Rerank (multilingual, free tier) or `bge-reranker-v2-m3` | Big quality jump for ~20 lines of code |
| Relational DB | PostgreSQL | Conversation logs + analytics |
| Widget | React + Vite, built to a single JS bundle | `<script src="..." data-bot-id="...">` install |
| Ops | Docker + docker-compose | One-command spin-up; deploy on Railway / Render / cheap VPS |

---

## 3. Architecture

Two pipelines meeting at the vector store:

```
INGESTION (runs once per upload)          QUERY (runs on every message)
─────────────────────────────────         ─────────────────────────────────
Client documents                          Customer question (widget/WhatsApp)
  (PDF · Word · web pages)                        │
        │                                 Query rewriting
Cleaning + chunking                        (condense w/ chat history)
  (+ metadata)                                    │
        │                                 Hybrid search (dense + BM25, RRF)
Embeddings                                        │
  (bge-m3 / OpenAI)                       Reranking → top 4–6 chunks
        │                                         │
        ▼                                 LLM + strict grounded prompt
     Qdrant  ──── top chunks ────▶                │
  (shared vector store)                   Streamed answer + citations
```

---

## 4. Repository Structure

```
rag-support/
├── app/
│   ├── ingestion/        # loaders, cleaning, chunking, indexing
│   │   ├── loaders.py
│   │   ├── chunking.py
│   │   └── indexer.py
│   ├── retrieval/        # hybrid search, fusion, reranking
│   │   ├── hybrid.py
│   │   └── rerank.py
│   ├── generation/       # prompts, LLM abstraction, streaming
│   │   ├── prompts.py
│   │   └── llm.py
│   ├── api/              # FastAPI routes, sessions, rate limiting
│   │   ├── main.py
│   │   └── routes/
│   ├── eval/             # test set + RAGAS runner
│   └── config.py         # single source of truth for all settings
├── widget/               # React chat widget (RTL-aware)
├── data/                 # demo company documents
├── docker-compose.yml
├── Dockerfile
├── .env.example
└── README.md
```

---

## 5. Build Phases — Detailed Steps

### Phase 0 — Setup (Day 1)

1. Create the GitHub repo on day one. A steady commit history is part of your marketing.
2. Set up the environment with `uv` or `poetry`; pin dependency versions.
3. Create `.env` (git-ignored) + `.env.example` for API keys. Never hardcode a key.
4. Centralize every tunable (company name, model names, chunk size, top-k, thresholds, brand colors) in `config.py` — this is what lets you redeploy per client in days.

### Phase 1 — Ingestion Pipeline (Days 2–4)

**Loaders**
- PDF: `pymupdf` (better Arabic text extraction than pypdf).
- Word: `python-docx`.
- Web pages: `trafilatura` for clean article extraction.
- FAQs: simple CSV/JSON reader (question, answer, category).

**Cleaning**
- Strip repeated headers/footers (detect lines that recur on every page).
- Normalize whitespace and Unicode (NFC normalization matters for Arabic).
- Test on a *real Arabic PDF* from day one — broken/reversed extraction is the most common Arabic-pipeline failure and you want to catch it early.

**Chunking**
- Baseline: recursive splitting at 800–1,000 tokens with ~15% overlap.
- Better: structure-aware chunking — split on headings so each section is one chunk; each FAQ Q&A pair is its own chunk.
- Attach metadata to every chunk: `source_file`, `page`, `section_title`, `updated_at`. Citations are built entirely on this metadata.

> Chunking quality affects final answer quality more than the choice of LLM. Do not rush this phase.

### Phase 2 — Indexing (Day 5)

1. Run Qdrant locally: `docker run -p 6333:6333 qdrant/qdrant`.
2. Create a collection with the chunk vector + full metadata as payload. Enable sparse vectors if using Qdrant's native hybrid mode with bge-m3.
3. Embed in batches (e.g., 64 chunks per request), not one-by-one.
4. Incremental re-indexing: store a content hash per file; skip unchanged files. Sell it as "update the bot's knowledge with one click."

### Phase 3 — Retrieval (Days 6–9, the heart of the project)

1. **Query rewriting (condensing).** Before searching, have the LLM rewrite the user's message into a standalone question using chat history. Without this, "and how much does it cost?" retrieves nothing; multi-turn conversations fail.
2. **Hybrid search.** Run dense (embedding) search and BM25 keyword search in parallel, then merge with Reciprocal Rank Fusion (RRF):

```python
def rrf(result_lists, k=60):
    scores = {}
    for results in result_lists:
        for rank, doc_id in enumerate(results):
            scores[doc_id] = scores.get(doc_id, 0) + 1 / (k + rank + 1)
    return sorted(scores, key=scores.get, reverse=True)
```

   Hybrid search is what catches product names, SKUs, and model numbers — and it is disproportionately important for Arabic.
3. **Rerank.** Retrieve the top ~20 fused results, pass them through the reranker, keep the best 4–6 for the LLM context.
4. **Confidence threshold.** If the top reranked score is below your threshold, skip generation and return: "I don't have enough information about that — would you like to talk to a team member?" This anti-hallucination guardrail is the single feature clients care about most.

### Phase 4 — Generation (Days 10–11)

**System prompt (core of it):**

```
You are a customer support assistant for {company_name}.

Rules:
- Answer ONLY from the context passages provided below.
- After every factual claim, cite the source number, e.g. [1].
- If the answer is not in the context, say so explicitly and offer
  to connect the customer with a human agent. Never guess.
- Reply in the same language the customer used (Arabic or English).
- Keep answers concise and friendly. Use the company's tone.

Context passages:
{retrieved_chunks_with_numbered_sources}
```

- **Streaming:** use Server-Sent Events so the answer appears token-by-token. The perceived speed difference is enormous.
- **Citations payload:** alongside the streamed text, return the source list (file name + page) so the widget can render clickable source chips.
- **Model abstraction:** one thin client class where switching between OpenAI and Anthropic is a single config value. Clients love hearing "you're not locked into one provider."
- **Prompt-injection hygiene:** instructions live only in the system prompt; retrieved chunks and user messages are treated strictly as data, never as instructions.

### Phase 5 — API Layer (Days 12–13)

| Endpoint | Purpose |
|---|---|
| `POST /chat` | `{session_id, message}` → SSE streamed answer + citations |
| `POST /ingest` | Upload/re-index documents (admin-key protected) |
| `GET /analytics` | Conversation counts, top questions, **unanswered questions** |
| `GET /health` | For deployment monitoring |

- Persist every conversation turn in PostgreSQL (session id, question, rewritten question, retrieved sources, answer, latency, confidence).
- Rate limiting with `slowapi` (e.g., 20 messages/min per session).
- CORS locked to the client's domain in production.

### Phase 6 — Embeddable Widget (Days 14–16)

- React + Vite, built to a single JS bundle. Install: `<script src="https://cdn.../widget.js" data-bot-id="acme"></script>`.
- Full RTL support and Arabic typography — a rare, high-value differentiator.
- Renders streamed text, source chips, and a "Talk to a human" button that fires an email/webhook with the full transcript.
- Theming (colors, logo, welcome message) driven entirely by config so each client deployment is a config change, not a code change.

### Phase 7 — Evaluation (Days 17–18, your unfair advantage)

1. Build a test set of 30–50 questions with reference answers from your demo documents. Include: easy lookups, multi-hop questions, questions with NO answer in the docs (the bot must decline), and Arabic + English variants.
2. Run RAGAS and record: **faithfulness**, **answer relevancy**, **context precision**, **context recall**.
3. Publish the numbers in your README, e.g. "Faithfulness 0.93 across a 50-question bilingual test set." Concrete metrics convert skeptical clients faster than any sales copy.
4. Keep the eval script in `app/eval/` and re-run it after any retrieval change — it doubles as your regression suite.

### Phase 8 — Docker & Deployment (Days 19–20)

- Multi-stage `Dockerfile` (slim final image), `docker-compose.yml` running app + Qdrant + Postgres with one command.
- Deploy a live demo on Railway or Render (near-zero cost at demo scale); add a custom subdomain.
- **Demo company:** invent one (e.g., an electronics store) and write its return policy, shipping FAQ, warranty terms, and a small product catalog in both Arabic and English. Prospects must be able to try the bot within seconds of landing on your demo page.

### Phase 9 — Packaging & Documentation (Days 21–22)

**README checklist:**
- Animated GIF of the bot answering (with citations visible).
- Architecture diagram.
- Feature table.
- RAGAS metrics table.
- "Tech decisions & why" section — technical clients actually read this.
- Quickstart: `docker compose up` → working bot in 2 minutes.

**Demo video (Loom, ~2 minutes):** the problem (support team drowning in repeat questions) → live demo → the metrics. Pin it as your first portfolio item.

---

## 6. Selling It on Freelance Platforms

### Upwork
- Title: *"Custom AI Chatbot (RAG) trained on your company data — with source citations"*.
- First line of every proposal: the live demo link. A bot that answers in seconds beats any pitch text.
- Portfolio item = demo video + GitHub link + metrics screenshot.

### Mostaql / Khamsat (Arabic market)
- Same offer in Arabic, emphasizing native Arabic answers, RTL widget, and WhatsApp integration — a real gap in the Gulf/Egyptian market.

### Pricing tiers

| Tier | Includes | Price range |
|---|---|---|
| Basic | Bot trained on client docs + hosted demo | $300–500 |
| Standard | + branded widget, analytics dashboard, human handoff | $700–1,200 |
| Premium | + WhatsApp (Meta Cloud API) or CRM integration | $1,500+ |
| Retainer | Hosting, monitoring, monthly knowledge updates | $50–150/month |

The monthly retainer is the real recurring revenue — always offer it.

### Handling the eternal client question: "Why RAG and not fine-tuning?"
- RAG is cheaper (no training runs), updates instantly (re-index a file vs. retrain a model), and prevents hallucination by grounding every answer in citable sources. Fine-tuning teaches style, not facts — and can't cite sources.

---

## 7. High-Value Extensions (build after v1)

1. **WhatsApp integration** via Meta Cloud API — enormous demand in the Arabic market; roughly doubles your deal size.
2. **Multi-tenant mode** — one deployment serving several clients (per-client collections + config), turning the project into a productized service.
3. **Feedback loop** — thumbs up/down on answers stored in Postgres, surfaced in analytics.
4. **Slack/CRM handoff** — escalations create a ticket instead of an email.

---

## 8. Timeline Summary

| Phase | Duration |
|---|---|
| 0. Setup | 1 day |
| 1. Ingestion | 2–3 days |
| 2. Indexing | 1 day |
| 3. Retrieval | 3–4 days |
| 4. Generation | 2 days |
| 5. API | 2 days |
| 6. Widget | 3 days |
| 7. Evaluation | 2 days |
| 8. Docker & deploy | 2 days |
| 9. Packaging | 2 days |
| **Total** | **~3–4 weeks part-time** |

Spend the extra time on Phases 3 and 7 — retrieval quality and documented evaluation are what make this project genuinely professional rather than another tutorial clone.
