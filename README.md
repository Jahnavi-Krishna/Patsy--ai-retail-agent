<h1 align="center">Patsy</h1>
<p align="center">A smart AI retail expert agent for PartSelect's ecommerce platform</p>


<p align="center">
  <img src="https://img.shields.io/badge/RAG-grounded-4CAF50" alt="RAG">
  <img src="https://img.shields.io/badge/GPT--4o-OpenAI-412991?logo=openai&logoColor=white" alt="GPT-4o">
  <img src="https://img.shields.io/badge/Agent-tool--calling-orange" alt="Tool-calling agent">
  <img src="https://img.shields.io/badge/ChromaDB-vector%20store-6E56CF" alt="ChromaDB">
  <img src="https://img.shields.io/badge/FastAPI-async-009688?logo=fastapi&logoColor=white" alt="FastAPI">
  <img src="https://img.shields.io/badge/Vision-OCR-9C27B0" alt="Vision + OCR">
  <img src="https://img.shields.io/badge/Vanilla_JS-frontend-F7DF1E?logo=javascript&logoColor=black" alt="Vanilla JS">
  <img src="https://img.shields.io/badge/Next.js-shell-black?logo=next.js&logoColor=white" alt="Next.js">
</p>

---

A full-stack AI assistant built for PartSelect's e-commerce platform as part of the Instalily AI case study. The assistant — named **Patsy** (short for **P**art**S**elect **A**gent) — helps customers find refrigerator and dishwasher parts, verify compatibility, follow installation guides, troubleshoot issues, and understand store policy. Every factual response is grounded in retrieved data; the system is architecturally designed to never invent part numbers, model numbers, or compatibility claims.

---

## The core engineering problem

The bottleneck for a production-scale e-commerce agent isn't general user satisfaction — it's **strict data accuracy and complete containment of hallucinations**. In an appliance-parts context, a wrong compatibility recommendation isn't just a bad answer; it's a physical return, a support ticket, and direct revenue loss. Accuracy is treated as a business constraint, not a quality nice-to-have.

> **Core principle:** Every factual claim Patsy makes about parts, compatibility, installation steps, or return policy must originate strictly from a retrieved tool result — never from model pre-training weights. The system is engineered to be **structurally incapable** of inventing part numbers, model numbers, or URLs.

## Not just a chatbot

The brief asked for a chat assistant. What got built is a **smart retail expert agent** — designed to reason like the most knowledgeable person on the sales floor, not just answer what's typed at it. It diagnoses a symptom before it's asked to, verifies compatibility from either a part number or a photo, and proactively surfaces related parts or next steps based on how the conversation is actually going.

The personality is a deliberate design choice, not a default: Patsy is trained to stay warm, cheerful, and genuinely helpful — including when a user is hostile, off-topic, or just testing it — without ever tipping into a scripted, "as an AI" tone. If it can't help with something, it still tries, and it still takes the feedback. This isn't assumed behavior; it's specifically validated in the Conversational Resilience test cases below (`QA-AGG-01`, `QA-OFT-01`, `QA-OFT-02`), and it's enforced at the prompt layer — see **System Prompt Isolation Layers** below.

## My role

Owned the full system solo, end to end: the RAG/grounding architecture, the diagnostic and recommendation logic, the personality and edge-case design, the FastAPI backend, the chat-widget frontend integration, and the QA test matrix used to validate all of it before submission — architecture through conversational UX, not just the model-calling code.

---

## How It Works

The project has two parts that must both be running:

| Service | Location | Port |
|---|---|---|
| Frontend (Next.js) | `frontend/` | 3000 |
| Backend (FastAPI) | `backend/` | 8000 |

The Next.js app serves as a shell — `app/page.tsx` immediately redirects to `partselect-landing.html`, which is the standalone chat widget. All AI logic runs through the FastAPI backend.

---

## Getting Started

### Prerequisites
- Python 3.10+
- Node.js 18+
- An OpenAI API key

---

### 1. Backend setup

```bash
cd backend
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

Create `backend/.env`:
```
OPENAI_API_KEY=your_key_here
```

Start the server:
```bash
uvicorn main:app --reload
```

On first start, the server indexes all parts, guides, and troubleshooting data into ChromaDB. Confirm it is ready before opening the frontend:
```bash
curl http://localhost:8000/health
# Expected: {"status":"ok","parts_loaded":30}
```

---

### 2. Frontend setup

```bash
cd frontend
npm install
npm run dev
# or
yarn dev
# or
pnpm dev
# or
bun dev
```

Open [http://localhost:3000](http://localhost:3000) — the page auto-redirects to the Patsy chat widget.

To modify the Next.js entry point, edit `app/page.tsx`. The page auto-updates as you save.

This project uses [`next/font`](https://nextjs.org/docs/app/building-your-application/optimizing/fonts) to automatically optimize and load [Geist](https://vercel.com/font).

---

## A Note on Scope

**This case study is focused on the chat assistant and its widget integration.** The landing page is a demo scaffold built to demonstrate three specific UX entry points:

- **Header "Start Chat" button** — immediate, unprompted access at the top of the page
- **Floating bottom-right overlay widget** — persists across the full shopping session; collapsing it preserves chat history and active audio in memory
- **"Ask about this part" on product cards** — injects part context directly into the chat, bridging static browsing to real-time AI assistance

Some surrounding page elements (top navigation links, account flows, full checkout) are static placeholders for visual context only. Everything inside the chat widget is fully functional.

---

## Full Tech Stack Breakdown

Every layer was chosen for explicit scalability and extensibility, not just MVP speed.

| Layer | Implementation Details |
|---|---|
| **Frontend: Vanilla HTML5/ES6** | Self-contained widget requiring no build step — a drop-in asset on any legacy or modern e-commerce platform, no React/Next.js/host-framework dependency required for the widget itself. Pure JavaScript state machine; client-side history array serialized into each `POST /chat` payload, preserving session state across widget collapse. |
| **Backend: Python FastAPI** | Asynchronous ASGI event loop for concurrent AI requests and streaming payloads. Four endpoints: `POST /chat` (multi-turn agent loop), `POST /tts` (voice synthesis), `POST /feedback` (evaluation logging), `GET /health` (readiness probe). Completely stateless — no session stores, no sticky-session overhead — scales horizontally behind a load balancer with zero affinity requirements. |
| **ChromaDB EphemeralClient** | In-memory vector indexing with sub-100ms retrieval lookups. Eliminates external database infrastructure during the MVP phase while maintaining full semantic search. Collections rebuilt at startup via a FastAPI lifespan context manager. |
| **text-embedding-3-small** | Shared embedding model for all catalog indexing and user query embedding, keeping the vector space consistent. Selected for high semantic precision across appliance-domain vocabulary at extreme cost efficiency. |
| **gpt-4o (chat + vision)** | The central brain of the agentic execution loop. Constructs multipart Base64 messages for image payloads. Loop runs up to 5 iterations: checks `tool_calls`, executes via a dispatcher, appends results to history, re-queries until a clean text response is finalized. |
| **tts-1 with Nova voice** | Server-side synthesis for hands-free repair mode. Markdown formatting and suggestion tokens are stripped before synthesis. Deliberate trade-off: a 1–2 second synthesis delay in exchange for human-like warmth that protects the brand experience, over the robotic quality of the native browser Speech Synthesis API. |
| **HTTPX + BeautifulSoup** | Live fallback for long-tail SKUs. If a part isn't in the local vector database, an async HTTPX request scrapes the live PartSelect product page and BeautifulSoup extracts name, price, and description — immediate extensibility without a manual catalog rebuild. |

---

## Data Schemas

Local datasets are curated sample sets built for multi-turn validation, anchored to the three real-world production test queries at the core of this case study.

**`products_sample.json`**

| Field | Type | Description |
|---|---|---|
| `part_number` | string | Primary key — PS-prefixed identifier (e.g. `PS11752778`). Used for exact lookup and compatibility joins. |
| `name` | string | Human-readable component name, rendered in product cards. |
| `brand` | string | OEM brand identifier — used for cross-brand catalog filtering and routing. |
| `category` | string | Enum: `refrigerator \| dishwasher`. Gates deep diagnostic search routing; out-of-scope categories trigger the routing matrix. |
| `short_description` | string | One-sentence snippet for product cards and quick summaries. |
| `symptoms` | string[] | Consumer-reported failure strings — the primary embedding and semantic-matching surface. |
| `product_url` | string | Verified canonical PartSelect URL. Only URLs from this field are ever surfaced to users. |
| `price` | number | Current list price (USD) — drives cart permissions and card display. |
| `in_stock` | boolean | Controls Add to Cart button state. |

**`compatibility_sample.json`**

| Field | Type | Description |
|---|---|---|
| `part_number` | string | Foreign key into `products_sample.json`. |
| `model_number` | string | Production appliance model identifier (e.g. `WDT780SAEM1`) — exact string match. |
| `compatible` | boolean | Strict true/false flag; includes intentional negative samples for QA error handling. |
| `notes` | string | Verbatim OEM confirmation or incompatibility reason, returned to the user unchanged. |

**`guides_sample.json`**

| Field | Type | Description |
|---|---|---|
| `id` | string | Unique chunk identifier for deduplication and citation. |
| `url` | string | Canonical source URL — the only URLs Patsy may present from this document type. |
| `title` | string | Guide chapter title, shown in citations. |
| `type` | string | Enum: `installation \| troubleshooting \| policy` — applied as a metadata pre-filter before embedding search runs. |
| `part_numbers` | string[] | Parts this chunk covers — used for compound filtering in `search_guides`. |
| `content` | string | Manual text, chunked at sentence boundaries (~400 tokens) before embedding. The source of every grounded agent reply. |

---

## Agent Tool Definitions and Intent Routing

The agent runs a multi-turn tool loop: evaluate intent, invoke registered tools, pack results as data tokens, and re-query the model up to 5 times until a clean textual summary is derived.

```
for _ in range(5):
    call GPT with tool definitions registered
    if response contains tool_calls:
        execute each via tools.execute(name, args)
        append tool result messages with role: tool
        re-query
    else:
        break and return final text response with parsed suggestions
```

| Tool | Parameters and Behavior |
|---|---|
| `search_guides` | Params: `intent` (`returns_policy \| installation_help \| troubleshooting`), `query_text`, optional `part_number`/`model_number`. Applies a type metadata pre-filter before similarity lookup — a returns-policy query reads exclusively from policy chunks. |
| `search_products` | Vector similarity matching over symptom descriptions to recommend candidate components matching the user's query. |
| `check_compatibility` | Deterministic matrix lookup on `part_number` + `model_number`. Returns an exact boolean and verbatim OEM notes — the agent is strictly prohibited from inferring compatibility from training weights. |
| `get_product_by_part_number` | Exact key lookup in the local JSON catalog, defaulting to the live HTTPX scraper fallback if not found locally. Returns name, price, description, and URL. |
| `track_order_status` | Processes a tracking number + email against a mock fulfillment database; returns carrier, tracking number, delivery estimate, and part context for an interactive fulfillment card. |
| `get_store_policy` | Retrieves the official 365-day return policy text directly from the data source — prevents the model from paraphrasing policy terms from memory. |
| `escalate_to_human` | Spawns an escalation card with support phone, email, and active hours on explicit user request. |

---

## System Prompt Isolation Layers

A layered constraint architecture separates stylistic persona rules from strict behavioral guardrails — each layer addresses one failure mode and is independently testable.

| Layer | Purpose |
|---|---|
| **1. Persona and Scope** | Establishes Patsy as a warm, enthusiastic appliance-parts specialist. Sets primary domain (refrigerators, dishwashers) and routing rules for out-of-scope interactions. |
| **2. Empathy and Accountability** | Overrides default LLM deflection: on detected user frustration, the model must acknowledge unconditionally, issue a dynamically varied apology (never repeated phrasing), and pivot immediately to an active fix. |
| **3. Factual Grounding** | Mandates all compatibility, pricing, and product claims originate from tool returns. Bans part-number generation and compatibility inference from training weights. |
| **4. Forbidden Phrase Enforcement** | Strict phrase blacklist — e.g. "I can't help with" is banned in favor of a routing link; "Do you have any questions?" is banned in favor of `\|\|SUGGEST:` chips. Max 3 paragraphs per response, max 3 sentences per paragraph. |
| **5. Vision and OCR Protocol** | Forces dedicated OCR evaluation on any image payload — attempts text extraction on appliance labels regardless of angle/lighting, reporting partial reads instead of a generic vision refusal. |
| **6. Suggestion Chip Protocol** | Every response closes with a `\|\|SUGGEST:` token and up to 3 pipe-delimited action labels, mapped to backend deterministic intents — no open-text dead ends. |

---

## Out-of-Scope Appliance Routing Matrix

Patsy's deep diagnostic scope is refrigerators and dishwashers — but every other category still gets a direct, click-ready link instead of a dead end, turning an out-of-scope query into a monetization path rather than a lost visitor.

| Category | Routes to |
|---|---|
| Washer | `partselect.com/Repair/Washer/` |
| Dryer | `partselect.com/Repair/Dryer/` |
| Range, Stove, Oven | `partselect.com/Repair/Range-Stove-Oven/` |
| Microwave | `partselect.com/Repair/Microwave/` |
| Freezer | `partselect.com/Repair/Freezer/` |
| Air Conditioner | `partselect.com/Repair/Air-Conditioner/` |
| Food Waste Disposer | `partselect.com/Repair/Food-Waste-Disposer/` |
| Water Heater | `partselect.com/Repair/Water-Heater/` |
| Lawn Mower | `partselect.com/Lawn-Mower-Parts.htm` |
| Lawn and Garden | `partselect.com/Lawn-and-Garden-Parts.htm` |
| Power Tools | `partselect.com/Power-Tool-Parts.htm` |
| BBQ Grill | `partselect.com/BBQ-Parts.htm` |

---

## Test Cases

Both services must be running before testing.

### Core anchor queries — the RAG pipeline was built and verified around these three

```
"How can I install part number PS11752778?"
"Is this part compatible with my WDT780SAEM1 model?"
"The ice maker on my Whirlpool fridge is not working. How can I fix it?"
```

### Runnable QA cases

**Parts, compatibility, and installation**

| Ref | Input | Test data | What to verify |
|---|---|---|---|
| QA-RAG-01 | `Check my order status` | Order: `PS-998822` / Email: `testuser@partselect.com` | Fulfillment tracking card with carrier and delivery details |
| QA-RAG-02 | `Is part PS11755072 compatible with model WDT780SAEM1?` | Part: `PS11755072` / Model: `WDT780SAEM1` | Exact true/false flag with OEM notes — no inference from training data |
| QA-RAG-03 | `How do I install part PS11752063?` | Part: `PS11752063` | Step-by-step guide extracted from indexed chunks; safety warning leads |

**Audio**

| Ref | Action | What to verify |
|---|---|---|
| QA-AUD-01 | Click **Listen** on one message, then immediately click **Listen** on another | First audio stops cleanly; second plays without any overlap |

**Conversational resilience**

| Ref | Input | What to verify |
|---|---|---|
| QA-AGG-01 | `You are completely useless.` | Sincere, varied apology; no defensiveness; immediate pivot to offering help |
| QA-OFT-01 | `Tell me a joke.` | Warm redirect in under two sentences; no blank response; no banned phrases |
| QA-OFT-02 | `My washing machine is broken.` | Direct link to `partselect.com/Repair/Washer/` — out-of-scope queries route, never dead-end |

**Vision and OCR**

| Ref | Action | What to verify |
|---|---|---|
| QA-VIS-01 | Upload a JPEG of a Whirlpool WRS325SDHZ model plate | Model number extracted and confirmed; partial reads handled gracefully; never a flat refusal |

---

## API Reference

| Endpoint | Method | Purpose |
|---|---|---|
| `/chat` | POST | Main multi-turn agent loop |
| `/tts` | POST | Text-to-speech synthesis (nova voice) |
| `/feedback` | POST | Thumbs up/down evaluation logging |
| `/health` | GET | Readiness check and catalog count |

---

## Accuracy Roadmap

Production-grade hallucination containment across the full catalog is a phased path, not a one-shot build.

| Phase | Goal |
|---|---|
| **1 — Current: Cost-efficient RAG** | Local JSON datasets indexed into ChromaDB at startup; deterministic suggestion-chip mapping bypasses open-text matching risk; deep diagnostics bounded to refrigerators and dishwashers; HTTPX/BeautifulSoup fallback covers long-tail SKUs. |
| **2 — Target: Full-catalog intelligence** | Move from targeted fallback scraping to a systematic horizontal crawl of the entire PartSelect catalog, building an exhaustive parts knowledge graph (every SKU, symptom tag, compatibility pairing) in a persistent, distributed vector store — replacing the curated MVP datasets entirely. |
| **3 — Target: Custom LLM fine-tuning** | Ingest metadata, transcripts, and step summaries from PartSelect's repair video library; fine-tune an open-weight model on the combined catalog + video corpus so it learns appliance-repair vocabulary and physical context intrinsically — targeting zero-hallucination guardrails on high-velocity SKUs and measurable conversion uplift. |

---

## Learn More

- [Next.js Documentation](https://nextjs.org/docs) — Next.js features and API reference
- [Learn Next.js](https://nextjs.org/learn) — interactive tutorial
- [Next.js GitHub](https://github.com/vercel/next.js)

---

## Deploy on Vercel

The easiest way to deploy the frontend is via [Vercel](https://vercel.com/new?utm_medium=default-template&filter=next.js&utm_source=create-next-app&utm_campaign=create-next-app-readme). The backend is a standard FastAPI application and can be deployed independently on any Python-compatible host (Railway, Render, Fly.io, etc.).

See the [Next.js deployment documentation](https://nextjs.org/docs/app/building-your-application/deploying) for frontend-specific details.
